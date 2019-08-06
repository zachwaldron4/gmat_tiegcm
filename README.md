

Hello reader, and welcome to the documentation for the GMAT-CCMC Atmospheric Model Plugin/Sattelite Drag Code

This document contains all the information that I needed, or found necessary, to understand the current state of the GMAT-CCMC plugin.  Interspersed are instructions for how to do things, and also explanations for why things are doing. 


## Table of contents:

1. [Instructions for Installing Anaconda](#first-bullet)
    1. [Creating new environments](#create-new-env)
        1. Kamodo and Kameleon
        1. Getting all necessary packages and requirements
1. [Accessing GMAT (cloud impact)](#second-bullet)
    1. Structure of GMAT on Cloud_impact
    1. CCMCPYplugin
        1. tiegcm.py reader
        1. CCMC_Atmpsophere.cpp
        1. KameleonAtmosphere.cpp
    1. [Populate cloud_impact with changes to tiegcm.py](#change_tiegcmpy_in_cloudimpact)
    1. [Populate cloud_impact with changes to KameleonAtmosphere.cpp](#change_cpp_in_cloudimpact)
1. [The TIEGCM reader](#third-bullet)
    1. [What specific changes are in the new tiegcm.py reader](#tiegcmpy-changes)
        1. Some context and explanations are offered 
    1. [How to use (Outside of GMAT)](#tiegcmpy-how-use)
1. [Model Data in GMAT](#fourth-bullet)
    1. Two time frames
    1. Runs on request data (issues with previous data)
        1. No ZGMID
        1. N2N
    1. TIEGCM data from NCAR Cheyenne cluster
        1. Provided input files to see exact run scenario
    1. IRIDEA Data
        1. Goce driven
        1. Two versions (2056 scaling factor and null)
1. [GMAT Notes](#fifth-bullet)
    1. Long vs Short Script
    1. [Issues with TDRS](#tdrs-considerations)
    1. Use Aqua and Fermi
        1. GPS point (Fermi), GN tracking (Aqua)
    1. Writing the script
        1. Output ephemeris into a report file
        1. Ephemeris filetype
    1. Identifying where the data is
        1. Output goes to output folder, move it to a new directory.
1. [Post processing](#sixth-bullet)
    1. Jupyter notebooks
        1. Which ones do what
    1. Comparing Density measurements from Original to New tiegcm.py codes
        1. Compare the different data sets as well (compare to each other and HASDM).  Didnâ€™t use MSIS
    1. Looking at Residuals for GMAT output. change_tiegcmpy_in_cloudimpact

***

# Installing Anaconda <a class="anchor" id="first-bullet"></a>

**NOTE: as of right now, GMAT and and tiegcm.py code runs with Python 2.7.  So when you create a new environement, be sure to use python 2.7 to run with GMAT** 

1. Install anaconda environment. I got anaconda3 here:
    - `https://www.anaconda.com/distribution/#download-section`
    - helpful to have the full anaconda (offers a visual GUI of the environments and packages) but you can just do miniconda
   
2. Run the installer, putting it where you like. Itâ€™s your personal environment and wonâ€™t interfere with any other system software. 



#### Creating new environments (Kamodo with python 3.7) <a class="anchor" id="create-new-env"></a>

This Kamodo environment will work with Kameleon (but Kameleon require python 2.7 as of now)


1. Create new anaconda environment called Kamodo37
~~~~
> conda create -n kamodo37 python=3.7 jupyter pandas numpy
> conda activate kamodo
~~~~
2. Install packages
~~~~
> pip install pytest
> pip install sympy
> pip install plotly==3.3
> pip install scipy
> pip install click
> pip install hSpy. 
> pip install netCDF4
> pip install hapiclient  
> pip install psutil
> conda install -c plotly plotly-orca   (rasters plots into jpg)
> pip install antlr4-python3-runtime
~~~~
   - (One of these downgrade pandas, need to re-upgrade pandas)     

3. Be sure to install the following before attempting to Install SpacePy:
~~~~
[through homebrew] gcc (fortran compiler)

 pip install matplotlib
 pip install ffnet   
 pip install h5py
  
 pip install spacepy â€”ORâ€” pip install git+https://github.com/spacepy/spacepy.git
~~~~

4. Be sure you have installed the most up-to-date version of Kamodo:
````
https://sed-gitlab.gsfc.nasa.gov/ccmc/Kamodo
````
5. Open jupyter notebook from the environment:
````
(kamodo37) gs674-loaner4:~ zwaldron jupyter notebook
````

##### WHEN RUNNING KAMODO 
Run this first :<br>
 	   `python setup.py install` <br>
To call an instance of kamodo: <br>
   	 `from kamodo.kamodo import Kamodo` <br>



***
# Accessing GMAT (Cloud Impact) <a class="anchor" id="second-bullet"></a>
    (from Asher)
## About

> GMAT is essentially a c++ executable linked to supporting plugins, one of which is our ccmc atmosphere plugin. At present, only TIEGCM is supported from our side. However, we should be able to add additional python models through kamodo. Here we will document the basic framework as it currently stands.  The TIEGCM interpolator code (tiegcm.py) was written by Asher Pembroke. 

> The  CCMCAtmosphere.cpp code is the big code that links GMAT to Kameleon through python.  The KameleonAtmosphere.cpp links the CCMCAtmopshere.cpp code to the tiegcm python reader. **If you want to edit the data that GMAT is reading, you have to edit the KameleonAtmosphere.cpp code**

### Locating gmat on cloud_impact

Log in to cloud_impact from your account on kona:

    ssh -XY <username>@kona.ccmc.gsfc.nasa.gov
    ssh -XY gmat@cloud_impact.ccmc.gsfc.nasa.gov
        ask CCMC for password

The gmat code is found here

    /home/gmat/GMAT_KAMELEON/GMAT_KAME_v2

This contains two directories:

    GmatDevelopment
    gmatinternal

To run the main executable, navigate to the bin directory

    cd GmatDevelopment/application/bin
    ./GMAT

If you logged in with -XY, then GMAT's graphical user interface should appear.

### GMAT Script

GMAT has its own scripting language which is used to set up and perform analyses. These scripts are placed alongside the GMAT executable in

    GmatDevelopment/application/bin

The one used for validating TIEGCM is

    Aqua_tiegcm.script

Or it's shorter version:
    
    Aqua_tiegcm_short.script


### Locating ccmc atmostsphere plugins

    gmatinternal/code/CCMCPYPlugin/src/base/atmosphere

which contains the following files

    CCMCAtmosphere.cpp  CCMCAtmosphere.hpp 

As well as these three directories

    DTM  JB2008  Kameleon

### Locating TIEGCM data

    GmatDevelopment/application/data/atmosphere/TIEGCM_data/
   
There are now four directories that hold data

    2013.03.01.tie-gcm.data  quiet_2013.03.01-2013.03.03
    pre_2019_data            storm_2013.03.16_2013.03.20
    
The directories with `storm` and `quiet` designations are used in the work shown here.

### Locating pytiegcm on cloud_impact

Asher's original tiegcm interpolator code can be found here:

    https://github.com/asher-pembroke/pytiegcm.git

and is cloned into the directory on cloud_impact here:

    /home/gmat/asher/pytiegcm_v0
    
The edited interpolator code is in the directory
    
    /home/gmat/asher/pytiegcm

### Progating changes to the python code 

Whenever Asher made changes to pytiegcm, he would first push them to the git repo from my work laptop, then pull those changes into the cloud_impact machine:

    cd /home/gmat/asher/pytiegcm
    git pull

Important!

> **Such changes need to be propagated into the version of python gmat is running. First activate the gmat environment on cloud_impact. Miniconda is already installed and the environment is already set up:**
>
>    `conda activate gmat`
>
>**This gives us a new command prompt which should look something like this:**
>
>    `(gmat) [gmat@cloud_impact pytiegcm]`
>
>**Now update the environment with the new tiegcm code:**
>    
>    `(gmat) [gmat@cloud_impact pytiegcm] python setup.py install`
>

### Changing the Data being used by GMAT<a class="anchor" id="change_tiegcmpy_in_cloudimpact"></a>
To change the data, you can edit the Kameleon code found here:

    /home/gmat/GMAT_KAMELEON/GMAT_KAME_v2/gmatinternal/code/CCMCPYPlugin/src/base/atmosphere/Kameleon
Inside the `KameleonAtmosphere.cpp` file, edit `line 363` to tell GMAT which directory to access the data in.

After you make a change, you must recompile the cpp code.  Instructions are below.

### Recompiling changes to C++ code using Cmake:<a class="anchor" id="change_cpp_in_cloudimpact"></a>


*From asher*
>1. There should be a file CMakeLists.txt at the root of the source directory, which I believe is GmatDevelopment. 
>2. CMakeLists.txt  (and associated command line options) tells cmake what to do and how to generate build files, which are placed in the same directory cmake is run from
>3. NEVER RUN CMAKE FROM THE SAME DIRECTORY AS THE SOURCE FILES.  Instead make a new build directory that's safe to populate with a bunch of build files that you can throw away when you're done compiling.
>4. There should also be several other CMakeLists.txt files in other subdirectories. When you invoke cmake, it will start at the root and continue down into subdirectories looking for other cmake files, executing their statements.
>5. The generated build files depend on what kind of build system you are using. On windows, this is often visual studio, while on mac it's something different. We tend to use "make" on linux. The build files will invoke the compiler, which creates executables and libraries and links them all together.

>there should be a build directory:
>
>    `GmatDevelopment/build/unix64`
>
>navigate there and then run 
>
>    `make -j8`
>
>That should start the compilation process. I forget if it also moves the compiled library into place. Often make projects will require another step:
>
>    `make install`
>
>>You shouldn't need to do the following, but in case you want to create a fresh build from scratch, first make a new build directory under GmatDevelpment/build:
conda activate gmat (or source activate gmat)
mkdir GmatDevelpment/build/mybuild
cd  GmatDevelpment/build/mybuild
>>
>>Then run the following cmake command
>>    ```
>>    cmake -DPYTHON_LIBRARY=$(python-config --prefix)/lib/libpython2.7.so -DPYTHON_INCLUDE_DIR=$(python-config --prefix)/include/python2.7 -DPYTHON_EXECUTABLE=/home/gmat/miniconda2/envs/forge/bin/python -DPLUGIN_CINTERFACE=OFF -DPLUGIN_MATLABINTERFACE=OFF -DGMAT_PROPRIETARYPLUGINS_PATH=/home/gmat/GMAT_KAMELEON/GMAT_KAME_v2/gmatinternal -DPLUGIN_PROPRIETARY_CCMCPY=ON -DPLUGIN_PYTHONINTERFACE=OFF ../..
>>    ```
>>
>>That last `../..` will tell cmake to generate makefiles based on the directory two steps up from the current one
DPLUGIN_PROPRIETARY_CCMCPY refers to the CCMC python plugin I built and instructs cmake to generate makefiles that will compile the plugin
DPLUGIN_PYTHONINTERFACE refers to gmat's native python interface, which is not to be confused with the custom one I built
>>
>>After you do all that, you should be able to run make from mybuild:
>>
>>    `make -j8`
>>
>>The j8 tells make that it can use up to 8 processors to compile in parallel
>>
>>I will keep looking for documentation on all this. Sometimes I have to check the history on the gmat machine to be sure I'm following the steps correctly:
>>
>>    `history | grep cmake`
>>
>>That should return all the times cmake was invoked by the current user. Looking at the history around that line should be illuminating.
>>
>>Hope this helps clarify things a bit. 


### Tutorials

 - Gmat: http://gmatcentral.org/display/GW
 - Python: https://developer.nasa.gov/CCMC/python101


    






***

# The TIEGCM Reader (tiegcm.py) <a class="anchor" id="third-bullet"></a>

Asher's original version of the reader can still be found in the place I specified above. 

The code consists of a `util.py` file and a `tiegcm.py` file.  See the above section to setup-up the environment for running an updated code. 

#### The Updates (Summer 2019)<a class="anchor" id="tiegcmpy-changes"></a> 

The changes I made are as follows:
- Added helium to the diffusive quilibrium extrapolation above the boundary
- Changed the top boundary to 25th pressure level to avoid errors from numerical issues (issue with Tiegcm2.0)
- Changed any use of 'Z' for the vertical coordinate to 'ZG'
- Any points that exist on the midpoint levels use 'ZGMID' as vertical coordinate.
- If code is using ZGMID, extrapolate from top ZGMID boundary
- Changed the way that mass mixing ratio is converted to mass density for each variable
- Changed code to be more general to different names for N2 (CCMC runs on request named it N2N for some reason) 
- Made the code more general/robust so that it can accept Both regular output types (from Cheyenne/NCAR) and the CCMC types from Runs on Request. 

    *Below are some explanations:*

> - Z is the geopotential.  using this variable will for sure give errors expecially at high alts
> - ZG is the geometric height (corrected for gravity) (measured using ilev)
>      - height is calculated using pressure levels through the scale heightâ€¦ H = kT/mgâ€¦ Z assumes constant gravity.  ZG corrects for changing gravity with changing height
       - P = P0 * exp(-z/H)
> - Most composition elements are measured on midpoints (must use ZGMID) (measured using lev) 
>     - `self.midpoint_variables = ['TN', 'O2', 'O1', 'N2', 'HE', 'ZGMID', 'NO','CO2_COOL', 'NO_COOL']` # index with lev      
>     - `self.interface_variables = ['DEN', 'ZG', 'Z']`    #index with ilev
> - **mmr** 
>    - The original way (mmr * density_tot/(1+mmr)) would be assuming that each constituent's mass mixing ratio was not already "included" in the total density, which in the case of TIEGCM output, it is. All the mass mixing ratios for the major species should add up to 1 at any given timestep and they are considered in the calculation for mass density.  Multiplying mmr * DEN gives us the mass density of each species.  
>    - The mass mixing ratio for species i is defined as follows within TIEGCM:  `mmr_i = rho_i / rho_total`. rearranging gives species density, `rho_i`, in terms of mass mixing ratio, `mmr_i`, and total mass density, rho_total:    `rho_i = mmr*rho_total` 

>Some math:
>- $n(O,h) = n(O,TopLevel) Ã— exp (( ht(TopLevel)-h ) / H(O)) m^{-3}$
>- $n(O_2,h) = n(O_2,TopLevel) Ã— exp (( ht(TopLevel)-h ) / H(O_2)) m^{-3},$ 
>- $n(N_2,h) = n(N_2,TopLevel) Ã— exp (( ht(TopLevel)-h ) / H(N_2)) m^{-3},$
>
>where H(X) ia scale height of the species, $H(X)= k * T(TopLevel)/M_X * g)$,
>
>- use T at the top level pressure 15 and assume constant above
>- g = g0(re/(re+h))2
>- k = Boltzman constant, $M_X$= mean mass of a molecule (in kg)
>
>mass density at any height h:
>$$= ( n(O,h) Ã— 16. + n(O_2,h) Ã— 32. + n(N_2,h) Ã— 28. + n(HE, h) Ã— 4 ) Ã— 1.66053*10^{-27} [kg/m3]$$ 

>Restating the above in another way: <br>
>$\rho_i = \Psi_i*\rho$
>   - Where $\Psi_i$ = mass mixing ratio, and $\rho$ is total mass density (DEN in TIEGCM)
>
>The mass density of any contituent (i) at a given height (z) will be:
>   $$ \rho_{i,z} = \rho_{i,0} \, \frac{T(z)}{T_o} \, \exp{\bigg( - \int_{z_0}^{z}{\frac{\delta z}{H_i(z)}}}\bigg)  $$
   

# How to use the TIEGCM.py reader (outside of GMAT)<a class="anchor" id="tiegcmpy-how-use"></a> 
- if you're going to make changes to the tiegcm.py code, you need the following command at the top of the notebook or code:
    ```
    %load_ext autoreload
    %autoreload 2
    ```

Import packages:
```
import pandas as pd
import numpy as np
from scipy.spatial import kdtree
from scipy.interpolate import griddata, LinearNDInterpolator
import scipy
from netCDF4 import Dataset
from plotly.offline import plot, iplot
import plotly.graph_objs as go
from plotly.graph_objs import Layout
import plotly
import matplotlib.pyplot as plt
import antlr4 
plotly.offline.init_notebook_mode(connected=True)
```

Import TIEGCM:
```
from tiegcm.tiegcm import TIEGCM, Point3D, Point4D, Slice_key4D, ColumnSlice4D, ColumnSlice3D, Model_Manager
from tiegcm.util import average_longitude
from tiegcm.util import *
```

To call the density of an object at any point in the model, do the following:
```
    xlon = ###
    xlat = ###
    xalt = ###
    idate = ###
    massdensity = tiegcm.density(xlat, xlon, xalt, idate )
```
### Going over multiple days (multiple .nc files)
To have tiegcm interpolate multiple files at once, use the `tiegcm.Model_Manager(directory)` object.
As the function suggests, be sure to identify the directory with the files you want to look at.

Then call the density as you would above, but through the Model_manager function (`massdensity = Model_manager.density(xlat, xlon, xalt, idate )
`.
Model_manger date takes a string as an input for date: `'%y-%m-%d %H:%M:%S'`





***

# Model Data in GMAT <a class="anchor" id="fourth-bullet"></a>

The data can be found on Cloud_impact server here:
```
~/GMAT_KAMELEON/GMAT_KAME_v2/GmatDevelopment/application/data/atmosphere/TIEGCM_data/
```

There are now four directories that hold data:

    2013.03.01.tie-gcm.data  quiet_2013.03.01-2013.03.03
    pre_2019_data            storm_2013.03.16_2013.03.20
    
We are looking at two time periods in March 2013.<br>
The directories with `storm` and `quiet` designations are used in the work shown here.  


### quiet_2013.03.01-2013.03.03

    CCMC_2013.03.01.tiegcm.data      
    Cheyenne_2013.03.01.tiegcm.data

 - CCMC runs on request data (This is pre_2019_data)
     - did not have ZGMID in it 
     - N2 is named N2N (and may be calculated differently?)
 - NCAR's Cheyenne (Zach's runs)
     - has ZGMID, molecular nitrogen named N2
     - contains `tiegcm.inp` file to see the starting parameters if yall wanna re-run it 

### storm_2013.03.16_2013.03.20
 - NCAR's Cheyenne (Zach's runs)
     - has ZGMID, molecular nitrogen named N2
     - contains `tiegcm.inp` file to see the starting parameters if yall wanna re-run it 
 - IRIDEA Data
     - ```
         main_2013_065_data-mult-fac-1p0.0  
         main_2013_065_data-mult-fac-1p2056.0
        ``` 
     ## Iridea Data Info
     - *From Eric Sutton*
     - Data Assimilated TIEGCM data that is driven by GOCE density measurements
     - St. Patrick's Day Storm starts around 17 March/7 UT (mtime = 76 7 0) and peaks around 17 March/21 UT (mtime = 76 21 0)
    - Driving IRIDEA w/ GOCE data around 250 km altitude
    - GOCE data availability is good from 6 March/3 UT (mtime = 65 3 0) to 23 March/18 UT (mtime = 82 18 0)
        - first step:  mtime_start = 65 0 0, mtime_stop = 82 18 0
    - Cold-start procedure: mtime_start = 59 0 0, mtime_stop = 65 0 0
        - Differs with 2018 paper in that the first step of fine-tuning starts at the end of the cold-start period, as opposed to overlapping; probably should run with overlap in the future, but it doesn't matter for this case since it's so quiet on the first several days
    - GOCE data v2.0 are used
        - Previous v1.4 and v1.5 have been scaled by Bruinsma et al. (2014, 2018) by 1.24 to match HASDM on average
        - v2.0 is slightly higher on average than v1.4 and 1.5, so that the new scaling (1.2056) is needed to ensure 1.24*mean(v1.4) = 1.2056*mean(v2.0)
        - for completeness, two separate IRIDEA runs were done:  one with (multfac = 1.2056) and one without (multfac = 1.0) this scaling factor
        
        


***

# Running GMAT Notes <a class="anchor" id="fifth-bullet"></a>

Things to keep in mind:
   - which data are you using? ([see how to change data in Cloud_impact](#change_cpp_in_cloudimpact))
   - how are you measuring error?

***

## Runs:
   Detailed below are the different run conditions that we set up for this project

>### Quiet time Conditions
- Run GMAT as is (Original Aqua-TDRS tracking) for all different quiet time states<br>
            We show here that the new reader is better for orbit propagation and sat drag estimation
    - Original reader 
        1. CCMC data
        2. ~~Cheyenne Data~~ (original reader doesn't read Cheyenne data because ZGMID and it uses N2 instead of N2N)
    - New reader
        3. ~~CCMC data~~ (new reader is having trouble (only in GMAT) reading CCMC_ror data because of ZGMID, looking into improving this, it works fine outside of GMAT)
        4. Cheyenne data
- Run again without TDRS, **ground stations only**
    - New Reader
        - Cheyenne Data

>### Storm time Conditions
- Currently waiting on start vector for Fermi and Aqua
    - GPS tracking (Fermi)
        1. New reader with new data
        1. IRIDEA data
        1. CMIT - new reader* (Katie needs to find CMIT data)
    - GN tracking (Aqua)
        4. New reader with new data
        5. IRIDEA data
        6. CMIT - new reader* (Katie needs to find CMIT data)

>#### HASDM something
- *need to find out conditions on collaboration with Fermi and Aqua data*


### Tracking with GN only
Directory with scripts to track using Ground network is at the following:

    /home/gmat/GMAT_KAMELEON/GMAT_KAME_v2/GmatDevelopment/application/data/Aqua_quietTime


The CCMCKameleon script is named `aqua_gn_20130301_tiegcm_short.script` and is edited to be shorter using the [ThinMode data filter](http://gmat.sourceforge.net/doc/R2018a/html/AcceptFilter.html).



***


#### How to Run GMAT (restated)

1. Navigate to the `bin` directory and enter `./GMAT`.  This opens the GUI (if you SSH'd with -XY flags).
2. Select the script you want to use: `Aqua_tiegcm_short.script`
3. Click `run` and buckle in.  The short script takes just ~3 hours to run 1 day.  The regular script takes ~3 days for 1 day.


#### Extra Notes from Code 500 (The Steves) <a class="anchor" id="tdrs-considerations"></a>
 - TIE-GCM model (uses Kameleon) was very very slow, almost unusable (haha -Zach)
 - You do not need to change the location of the ground stations as they are in Earth Centered Fixed coordinates and do not change.
 - TDRS
     - Note that the best way to model TDRS is through an ephemeris but GMAT does not do this yet which introduces some error.
     -  the TDRS data type used in these scripts has not yet been fully tested.   (but, we donâ€™t know of any major errors)
     - (From Steve F) - "  If this is a data simulation project I would leave it out as untrusted and substitute GN tracking instead. If you are working with real data, it might even then still be best to work with just the ground tracking to see if that is sufficient first."
     -   The TDRS data type is untested and not trusted.  (You wouldnâ€™t want to publish using this data type.  You wouldnâ€™t want to use it for anything approaching operations)
     - The TDRS data type, with an ephemeris read in capability, will probably be tested by the end of 2020.  
 - Steve Cooley- "We never tested that the atmospheric density model output was correct.  We only go the interface working before we ran out of time.  TIEGCM was trusted the least.  DTM and JB2008 were trusted more."
 - To get a text ephem file, switch to the STK format: `Code500Ephemeris.FileFormat = 'STK-TimePosVel';`
 - `GMAT Aqua.ModelFile = 'aura.3ds';` doesn't do much of anything in terms of calculation. 
     - Using a 3D model in GMAT is problematic. The only 3D drag model currently available in GMAT is the â€œSolar Pressure and Aerodynamic Dragâ€ model and itâ€™s quite complicated and difficult to apply to low-earth satellites because they typically have dynamic solar array activity. Some 3D models for Aqua and Aura are here (see below) but I donâ€™t know how precise they are. In the FDF we only use spherical area modeling (what we call â€œcannonballâ€ modeling) for Aqua and Aura. (https://nasa3d.arc.nasa.gov/models#A)
 - Steve Cooley had Tuan create a JIRA ticket of work items that were not completed just to keep a record of what needed to be done.  See http://bugs.gmatcentral.org/browse/GMT-6695
 - They recommend using Fermi s/c since it uses GPA point solutions
 - â€œGNâ€ and â€œground stationâ€ tracking are the same thing. Steve S. tends to use both terms, as distinguished from SN/Space Network or TDRS tracking.
 - It looks like the supplied script (Aqua_tiegcm_short.script) uses only TDRS range data. You can tell this because bat.Measurements assigns the TDRS_Range tracking file set, and the TDRSRange tracking file set points to a file that appears (by its name) to only contain TDRS range data. The DataSource on the ground station does not refer to a tracking data source. Instead I think it refers to a local weather data source.


## Writing and interpreting the script
- Information on the BatchEstimator (Aqua_tiegcm.txt)
    - http://gmat.sourceforge.net/docs/R2018a/html/DSN_Estimation_Create_and_configure_BatchEstimatorInv_object.html



***
# Post Processing <a class="anchor" id="sixth-bullet"></a>

Still working on this part.  As far as I can tell right now, the best way to see how well the model is performing is to either look at the residuals and see how low they are.

I have also been looking at the Coefficient of Drag as an indicator for how "accurate: the model reader is.  We expect Cd to be ~2.2, but as GMAT crunches through iterations, it adjusts the Cd to compensate for innacuracies in the mass density. When mass density is very wrong, the Cd will change dramitcally so the model can converge.


Information on GMAT BatchEstimatorInv report file (what comes in Aqua_tiegcm.txt)
 - http://gmat.sourceforge.net/docs/R2018a/html/DSN_Estimation_Run_the_mission_and_analyze_the_results.html#DSN_Estimation_Batch_Estimator_Output_Report






# Notebooks in `pytiegcm` directory:
> The main notebooks to pay attention to are `Model_density_comparisons.ipynb` and `changes_notes.ipynb`.

- `calculate ZGMID from ZG.ipynb`
     - details how to calculate ZGMID from ZG if the run of TIEGCM2.0 does not contain ZGMID.  
 - `changes_notes.ipynb`
     - 
 - `extrapolate_mass_density_TIEGCM.ipynb`
     - Contains plots and methods for how to extrapolate Density above top Boundary (without Asher's interpolator code)
     - also shows how not using ZGMID gives of results that are off by .5 pressure level
 - `Investigate adding helium.ipynb`
     - I ran into some random trouble when trying to add helium (picture provided.  It turns out it was a syntax error (+= is not the same as =+ or something like that)
 - `Model_density_comparisons.ipynb`
     - this is the code that shows the effective density of the model @ Aqua's orbit.  
     - It shows the Quiet and Storm time conditions as well as the improved accuracvy of the new model reader. 
     - Also compares storm-time to the IRIDEA goce driven Data assimilated model output.
 - `ncdump.ipynb`
     - a helpful tool that comes with netcdf nut I couldnt get working...Shows everything in the cdf files. 
     - **Panoply is another great tool that does this** (https://www.giss.nasa.gov/tools/panoply/)
 - `Reading irregular .nc files.ipynb`
     - Tried to seperate .nc files that ran over multiple days in 1 file. Abandoned this code in favor of just separating the files.
     - command-line tools called NCO. Specifically, the command is `ncrcat`
     - can load/use them after issuing the command `module load nco`
 - `Reading_GMAT_output.ipynb`
     - offers a script that can easily read in Csv files to pandas for plotting and controlling datetime stuff.
     - preliminary code that can read the Aqua_tiegcm.txt output (BatchEstInv) . This is still a work in progress.
     - If GMAT is not directed to ouput the report file with delimeters (semicolons) then reading the file becomes very difficult for Pandas.  (I ahven't found an easier alternative).
 - `Sample_tiegcmwReader_alongAquaOrbit.ipynb`
     - Shows how to sample the TIEGCM output with the tiegcm.py model reader along a given orbit.  
 - `TIEGCM.ipynb`
     - This is Asher's notebook that gave some tips on how to use and work the model reader. 
 - `tiegcm_zach.ipynb`
     - my effort and noetbook that was replicating and working through Asher's notebook.
 
 


# Future stuff:
- new reader can't read CCMC data because ZGMID, looking into improving this.)
- N2N Weirdness
     - Need to talk to someone about N2N is showing a very large number (10^13)  This does not make sense even if it is already and mass density.  What is going on here.  Why is it different.  Can we change the job script so that it is just MMR as well?
- Need to give the new version of the tiegcm.py reader to Darren
- Upgrade to Python 3.6
- put Kamodo in place of Kameleon interface

- Steve Cooley had Tuan create a JIRA ticket of work items that were not completed just to keep a record of what needed to be done. See http://bugs.gmatcentral.org/browse/GMT-6695


 - Make it easier to analyze the output data.
     - get a plotter into GMAT
     
 - fix the reader for the Aqau_text stuff.  
 
 - Maybe write all this up as a Validation paper.
 
 - I need put it all on Github.


```python

```

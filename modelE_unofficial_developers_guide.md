# ModelE new Developers Guide for building in the GEOS CMake System
This is a document for people that wish to build and run modelE but could care less about the science. This fills in all the holes in the ModelE documentation.
# Obtaining ModelE
If you are on Discover modelE can simply be cloned (assuming you have permission) via this command
```
git clone simplex.giss.nasa.gov:/giss/gitrepo/modelE.git
```
Once you have cloned modelE you will see the following directory tree:
```
.
├── aux
├── cmor
├── config
├── decks
├── diags
├── doc
├── exec
├── init_cond
├── Makefile
├── model
├── modele-control.pyar
├── README
├── templates
```
**The first rule of ModelE is that every command must be issued from the decks directory, repeat that, every command must be issued from the decks directory**

# Setting up ModelE
Note that ModelE builds and build in source for example, rather than a modern clean separation of the source code and build. It also assumes you issue runs commands from decks but as you will see later you can avoid this.

The first thing you will need to do is configure your .ModelErc file in your home directory. This is where by default ModelE will look for important settings like what compiler to use, what MPI stack to use, and the modelE support directory where files will be placed such as the run configuration etc. Note that for some reason the run parameters and this path must be known at build time which flies in the face of modern software development.

To setup your .ModelErc file you can do this **(from the decks directory)**:
```
make config COMPILER=intel ModelE_Support=/Path_to_somewhere/ModelE_Support
```
this will create a basic .ModelErc file using Intel Fortran as the compiler. The user will want to immediately go in and chagne some settings. First turn on MPI as that is not on by default. Set
```
MPIDISTR=intel
```
to use Intel MPI for example. You should find after this command that if you do `tree -L 1` in /path_to_somewhere/ModelE_Support you will see:
```
.
├── exec
├── huge_space
├── prod_decks
├── prod_input_files
└── prod_runs
```
Next you will notice that you have these options set:
```
DECKS_REPOSITORY=/Path_to_somewhere/ModelE_Support/prod_decks
CMRUNDIR=/Path_to_somewhere/ModelE_Support/prod_runs
GCMSEARCHPATH=/Path_to_somewhere/ModelE_Support/prod_input_files
EXECDIR=/Path_to_somewhere/ModelE_Support/exec
```
Note that all this is relative to /Path_to_somewhere/ModelE_Support, basically it assumes that every will to this common working directory. Maybe this can be split. I will experiment later with this. This should be all you need to set to get things at least built.  **On Discovery you will need to change the GCMSEARCHPATH as detailed in the getting inputs secton!"

# Building ModelE outside of CMake but using GMAO's baselibs

## Making a Rundeck

First you will need to be on a branch that has support for GEOS integration which currently is `GEOS_integration`. Assuming that point to any GEOS build with the baselibs/modules you want and source g5_modules. If you don't have a build, you could just load the right modules for a particular baselibs and set $BASEDIR to the corresponding baselibs you want to use. In any case, we have added a new .mk file that will point modelE to use the ESMF and NetCDF in our baselibs.

Next we have to make what they call a deck. ModelE compiles the simulation parameters in rather than say configuration this at runtime like modern software. The deck is a configuration file that tells it what model configuration to use and MUST be specified before building. To do this **once again in the decks directory**, issue a command line this:
```
make rundeck RUN=<my_run_id> RUNSRC=<template> OVERWRITE=YES
```
The template is the name of one of the .R files sitting in the templates directory (do not add the .R in the RUNSRC opton) that is the run parameters and the RUN is the name of the run that will be used later. After this command, lets say the user chose RUN=geos_run the user should see that running `tree` in /Path_to_somewhere/ModelE_Support will show this:
```
.
├── exec
├── huge_space
├── prod_decks
│   └── geos_run.R
├── prod_input_files
└── prod_runs
```
So apparently make rundeck copies the .R file with name foo.R to the prod_decks directory with name the name specified by RUN=.

## Interlude to get inputs
**Note to get the input data if you are on Discover you must set GCMSEARTHPATH=/discover/nobackup/projects/giss/prod_input_files.**
At this point the official documentation says that after creating the run the user should do this from decks if on Discover:
```
../exec/get_input_data <RunID>
```
**DO NOT DO THIS!** If you are on discover, it seems as long as you have yoru GMSSEARTHPATH set it will create the symlinks for the input files in the experiment directory. If you run that it will also copy the files to your decks directory which you do not need. The instructions on the modelE page are a bit confusing here, they make it sound like this step is not optional but from what I can see it is.

# Building modelE outside of GEOS.
Just run this from decks:
If you just want to build the model you but not setup the experiment you can do:
```
make gcm RUN=geos_run MPI=YES MAPL=YES OVERWRITE=YES
```
or to build and setup the experiment:
```
make setup RUN=geos_run MPI=YES MAPL=YES OVERWRITE=YES
```
At the end of running make setup you will see this in your ModelE_Support directory:
```
.
├── exec
├── huge_space
│   └── geos_run
│       ├── E -> geos_run
│       ├── flagGoStop
│       ├── fort.99
│       ├── geos_run
│       ├── geos_run.exe
│       ├── geos_runln
│       ├── geos_run.PRT
│       ├── geos_run.qsub
│       ├── geos_runuln
│       ├── I
│       ├── Ibp
│       ├── Iij
│       ├── Ijk
│       ├── modules
│       ├── PARTIAL.accgeos_run.nc
│       ├── run_status
│       ├── runtime_opts
│       └── warn_lakes
├── prod_decks
│   └── geos_run.R
├── prod_input_files
└── prod_runs
    └── geos_run -> /discover/nobackup/bmauer/ModelE_Support/huge_space/geos_run
```    

# Building ModelE inside of GEOS
To build modelE inside of GEOS we use the external_project command of CMake. The following block of code will setup a rundeck and build modelE in CMake

```
file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/include/modelE)
include(ExternalProject)
message(INFO "MODELE TEMPLATE: $ENV{ModelE_Template}")
message(INFO "MODELE RUN: $ENV{ModelE_Run}")
message(INFO "MODELERC:  $ENV{MODELERC}")
ExternalProject_Add(modelE
   PREFIX "modelE"
   SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/@Model5Eagcm_GridComp/decks
   DOWNLOAD_COMMAND ""
   CONFIGURE_COMMAND ""
   UPDATE_COMMAND ""
   BUILD_ALWAYS 1
   BUILD_COMMAND make clean OVERWRITE=YES && make rundeck RUN=$ENV{ModelE_Run} RUNSRC=$ENV{ModelE_Template} OVERWRITE=YES && $(MAKE) setup RUN=$ENV{ModelE_Run} MAPL=YES OVERWRITE=YES GEOS_BINARY_DIR=${CMAKE_BINARY_DIR}  VERBOSE_OUTPUT=YES MODELERC=$ENV{MODELERC} MPI=YES
   BUILD_IN_SOURCE 1
   INSTALL_COMMAND find ../model/ -type f -name "*.a" -exec /bin/cp {} ${CMAKE_BINARY_DIR}/lib $<SEMICOLON> &&  find ../model/ -type f -name "*.mod" -exec /bin/cp {} ${CMAKE_BINARY_DIR}/include/modelE $<SEMICOLON>
   DEPENDS MAPL
)
```
# Running ModelE with runE
Once you have run make setup you should see in your decks directory a symlink to the run directory (in our example this is name is geos_run) under /Path_to_somewhere/ModelE_Support/huge_space/geos_run.
To run modelE via their script you can do 
```
../exec/runE geos_run -cold-restart -np 1
```
Note this will say it is submitting a script but it is not actually. 

# Running ModelE directly
Note that modelE creates a custom executable for each run. So in the example on this page you would get a geos_run.exe file. To run it directly you can go to /Path_to_somewhere/ModelE_Support/huge_space/geos_run, the 
```
./geos_runln 
```
script which will link in the boundary conditions. To run modelE do:
```
mpirun -np 1 ./geos_run.exe -i I -cold-restart
```
This will coldstart modelE and at the end you should have the modelE restarts fort.1.nc and fort.2.nc. You can also run 
```
geos_rununl
```
to remove the boundary condition symlinks.

# ModelE new Developers Guide for building in the GEOS CMake System
This is a document for people that wish to build and run modelE but could care less about the science. This fills in all the holes in the ModelE documentation.
# Obtaining ModelE
If you are on Discover modelE can simply be cloned (assuming you have permission) via this command
```
simplex.giss.nasa.gov:/giss/gitrepo/modelE.git
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
Users of more modern software will find the build and structure of ModelE rather strange. Note that ModelE builds in source for example, rather than a modern clean separation of the source code and build. 

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
Note that all this is relative to /Path_to_somewhere/ModelE_Support, basically it assumes that every will to this common working directory. Maybe this can be split. I will experiment later with this. This should be all you need to set to get things at least built. 

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
At this point the official documentation says that after creating the run the user should do this from decks if on Discover:
```
../exec/get_input_data <RunID>
```
Note, do not do the -w option, this will download them and is really,really slow.

This will download a lot of files and does it to the decks directory. **In other words this is downloading to the source code director**. This seems like a really bad idea, will ask if there is a different way to do this.

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
   BUILD_COMMAND make clean OVERWRITE=YES && make rundeck RUN=$ENV{ModelE_Run} RUNSRC=$ENV{ModelE_Template} OVERWRITE=YES && $(MAKE) gcm RUN=$ENV{ModelE_Run} MAPL=YES OVERWRITE=YES GEOS_BINARY_DIR=${CMAKE_BINARY_DIR}  VERBOSE_OUTPUT=YES MODELERC=$ENV{MODELERC} MPI=YES
   BUILD_IN_SOURCE 1
   INSTALL_COMMAND find ../model/ -type f -name "*.a" -exec /bin/cp {} ${CMAKE_BINARY_DIR}/lib $<SEMICOLON> &&  find ../model/ -type f -name "*.mod" -exec /bin/cp {} ${CMAKE_BINARY_DIR}/include/modelE $<SEMICOLON>
   DEPENDS MAPL
)
```

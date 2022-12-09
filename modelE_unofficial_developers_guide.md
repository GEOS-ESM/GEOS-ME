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

First you will need to be on a branch that has support for GEOS integration. Assuming point to any GEOS build with the baselibs/modules you want and source g5_modules.

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
At this point the official documentation says that after creating the run the user should do this from decks:
```
../exec/get_input_data -w <RunID>
```
This will download a lot of files


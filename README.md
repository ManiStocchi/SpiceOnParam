

# ICON and SPICE on PARAM Rudra
This is a guide for setting up SPICE and ICON on PARAM Rudra. This is the result of the India-EU collaboration under the **WP5 - Weather and Climate** of the project **GANANA**. This is an adaptation of the [Official SPICE documentation](https://hereon-coast.atlassian.net/wiki/spaces/SPICE/overview?homepageId=983042), and focuses only on the deployment. Any information regarding the workflow phases and their proper scientific use is reported into the documentation. 

Take into account that this guide is based on a test run simulating **just one month** on a small test domain. Some configuration reported here aim to replicate a test run on CMCC's Cassandra cluster. Therefore, different SPICE experiments might need you to dive deep into the scripts and understand what you need to change for your specific problem. 
## Building ICON
### Binary download
ICON can be dowloaded from the official github repository.  This can be done via command line vi the command
```
git clone --recursive https://gitlab.dkrz.de/icon/icon-model.git
```
The flag `--recursive`  allows for the automatic download of the external dependencies of the model. The command specified above downloads the latest release of Icon, which is the 2026.04 at the time of writing. If a particular version of ICON is needed (e.g., version 2025.04) it can be dowloaded with 
```
git clone --recursive -b icon-2025.04-public https://gitlab.dkrz.de/icon/icon-model.git
```
Please take into account that, at the moment, only v2025.04 has been successfully tested with Spice on PARAM Rudra, while v2025.10 appeared be **not working**.


### Building environment
In order to smoothly build ICON you need to load the necessary libraries with spack. Run the following commands
```
module load spack
. /home/apps/SPACK/spack/share/spack/setup-env.sh
spack load intel-oneapi-compilers@2024.2.1/2tzqg5o  
spack load intel-oneapi-mpi@2021.17.1/z55gh32  
spack load intel-oneapi-mkl@2025.3.1/bc3vucf  
spack load hdf5@1.14.6/wsjbczv  
spack load netcdf-c@4.9.3/ia73nft  
spack load netcdf-fortran@4.6.2/n5p5d6s  
spack load netcdf-cxx@4.2/tz2ujla  
spack load eccodes@2.45.0/ccskfp7
```

### Configuration and build
There is a tested configuration file for building ICON. Copy the configuration file to your icon directory
```
cp  /scratch/gananawp1/configure_icon.sh /path/to/your/icon-model/
```
and run it
```
sh configure_icon.sh
```
The process usually requires within one and two ours, and if everything went fine the message *Installation completed successfully!* should appear.  If, needed, a debug version can be compiled with
```
sh configure_icon.sh debug
```
This version has the function stack backtrace,  is not tolerant for divisions by zero and performs array bound checks. While it is useful for understanding what might cause bugs, it is way less computationally efficient than the standard version.

## Setting up SPICE
In order to set up SPICE several steps are required.
### Environment
The first step consists in loading the necessary spack libraries. I strongly suggest to do it in a different ssh section than the one used to configure ICON, as loading too many spack libraries could cause the PATH variable to become too long and pollute the stream. Run
```
module load spack
. /home/apps/SPACK/spack/share/spack/setup-env.sh
spack load intel-oneapi-compilers@2024.2.1/2tzqg5o
spack load netcdf-c@4.9.3/ia73nft
spack load netcdf-fortran@4.6.2/n5p5d6s
spack load netcdf-cxx@4.2/tz2ujla
spack load eccodes@2.45.0/ccskfp7
spack load libfyaml@0.9/cd5spcc
spack load libxml2@2.10.3/motse7g
spack load python@3.11.9/avs7oxe
spack load r@4.5.2/bka7w4y
spack load cdo@2.2.2/3reqh2l
spack load nco@5.3.4/5rmkt65
```
### Download
Use wget to download the SPICE from the official Zenodo repository and untar it 
```
wget "https://zenodo.org/records/10047046/files/spice-v2.3.tar.gz?download=1" -O spice-v2.3.tar.gz  
tar -xvf spice-v2.3.tar.gz
```
Download usually takes less than a minute, and should create a directory called *spice-v2.3*

### Setup
Enter into the *spice-v2.3* you just created, and set 
```
cd spice-v2.3
SPDIR=$PWD
``` 
#### Build `fortran-csv-lib`
Move to the `fortran-csv-lib` folder
```
cd  ${SPDIR}/src/fortran-csv-lib
```
and create a file called `Fopts`  with this content
```
FC            = mpiifx
FFLAGS        = -c -free -I$(STDROOT)/obj
LDFLAGS       = $(FFLAGS)
```
save it and make it
```
make
```
#### Build `cfu` executable
Move to the cfu directory
```
$ cd  ${SPDIR}/src/cfu
```
and create a file called `Fopts` with this text
```
#... netCDF
NETCDFC_ROOT=/home/apps/SPACK/spack/opt/spack/linux-cascadelake/netcdf-c-4.9.3-ia73nft6bjafqecnljklcenntp2n35ux
NETCDFF_ROOT=/home/apps/SPACK/spack/opt/spack/linux-cascadelake/netcdf-fortran-4.6.2-n5p5d6snc6ytsud2uewhk4wifdiuvm2e

NC_INC  = -I${NETCDFF_ROOT}/include
NC_LIB  =  -L${NETCDFF_ROOT}/lib -lnetcdff
NC_LIB  += -L${NETCDFC_ROOT}/lib -lnetcdf

# standard binary
PROGRAM      = bin/cfu

# compiler, options and libraries

FC          = mpiifx
FFLAGS       = -c -free -xCORE-AVX2 ${NC_INC}

LDFLAGS      =

LIBS         = ${NC_LIB}
LIBS        += -Wl,-rpath,$(NETCDFF_ROOT)/lib:$(NETCDFC_ROOT)/lib
```
and make it
```
make
```

### Build conversion programs
Now go into the utils directory
```
$ cd  ${SPDIR}/src/utils
``` 
and again create a file called `Fopts` with this content
```
#... netCDF
NETCDFC_ROOT=/home/apps/SPACK/spack/opt/spack/linux-cascadelake/netcdf-c-4.9.3-ia73nft6bjafqecnljklcenntp2n35ux
NETCDFF_ROOT=/home/apps/SPACK/spack/opt/spack/linux-cascadelake/netcdf-fortran-4.6.2-n5p5d6snc6ytsud2uewhk4wifdiuvm2e

NC_INC  = -I${NETCDFF_ROOT}/include
NC_LIB  =  -L${NETCDFF_ROOT}/lib -lnetcdff
NC_LIB  += -L${NETCDFC_ROOT}/lib -lnetcdf


CSV_ROOT = $(PARENTDIR)/fortran-csv-lib
CSV_INC = -I${CSV_ROOT}/include
CSV_LIB = -L${CSV_ROOT}/lib -lcsv

#
# name and path of standard binary to be produced
CCAF2ICAF_PROGRAM      = bin/ccaf2icaf
CORRECT_CF_PROGRAM     = bin/correct_cf
#

# compiler, options and libraries

FC          = mpiifx

FFLAGS      = -c  -free  -xCORE-AVX2 ${CSV_INC} ${NC_INC}

LDFLAGS     =

LIBS        = ${NC_LIB} ${CSV_LIB}
LIBS       += -Wl,-rpath,$(NETCDFF_ROOT)/lib:$(NETCDFC_ROOT)/lib:$(CSV_ROOT)/lib
```
and make it
```
make
```

#### Configure SPICE
Move to the configure_scripts directory
```
cd  ${SPDIR}/configure_scripts
```
and create a file called `system_settings` with this content 
```
# ICON-CLM Starter Package (SPICE_v2.3)
#
# ---------------------------------------------------------------
# Copyright (C) 2009-2025, Helmholtz-Zentrum Hereon
# Contact information: https://www.clm-community.eu/
#
# See AUTHORS.TXT for a list of authors
# See LICENSES/ for license information
# SPDX-License-Identifier: GPL-3.0-or-later
#
# SPICE docs: https://hereon-coast.atlassian.net/wiki/spaces/SPICE/overview
# ---------------------------------------------------------------

AWK=/bin/awk  
PIGZ=/usr/bin/pigz
CDO=/home/apps/SPACK/spack/opt/spack/linux-cascadelake/cdo-2.2.2-3reqh2l6avlgypt45knvbxbxyeygt4ou/bin/cdo
CDO_VERSION=2.2.2
NCO_BINDIR=/home/apps/SPACK/spack/opt/spack/linux-cascadelake/nco-5.3.4-5rmkt65slmsiehjbgcwnskrf4wyxsn7n
NCO_VERSION=5.3.4
NC_BINDIR=/home/apps/SPACK/spack/opt/spack/linux-cascadelake/netcdf-c-4.9.3-ia73nft6bjafqecnljklcenntp2n35ux/bin
NC_VERSION=4.9.3
RBIN=/home/apps/SPACK/spack/opt/spack/linux-cascadelake/r-4.5.2-bka7w4yqhzrxmsigkchtsiugzs4toayq
RVERSION=4.5.2
RMODULE=
WGET=/usr/bin/wget
PYTHONMODULE=python/3.12.2
PYTHON=/home/apps/SPACK/spack/opt/spack/linux-almalinux8-cascadelake/oneapi-2024.0.0/python-3.11.9-avs7oxev37lg7lf23koz7ozr6oph534s
GCCMODULE=usr/bin/gcc
```
Now open the `config.sh` script and go at line 200. Replace `module load gcc` with `spack load gcc/3wdooxp` and run the script
```
./config.sh -s dkrz-levante
```
(Levante is a supercomputer hosted at DKRZ, and PARAM-Rudra is pretty similar so we are using the same  configuration script). 
If everything went fine a message *configuration of SPICE completed*
should appear.  One addition check: run the command `ls ${SPDIR}/chain`, and you should see these folders in the output:
```
arch  boundary_data_options  gcm2icon  icon2icon  scratch  work
```
## Setting up your SPICE experiment
While the previous steps have to be done once per SPICE configuration, the following ones have to be done every time we need to set a new SPICE experiment up. Move to the `gcm2icon` folder. You will find there a fodler called `sp001` that we will copy with the name of our experiment. In this guide, we will name the icon experiment "GANANAtutorial",  but you are free to choose whatever it fits best for your case. 
```
cd ${SPDIR}/chain/gcm2icon
cp -r sp001 GANANAtutorial
cd GANANAtutorial
```
Within the newly created folder you will find a text file called `job_settings`, an executable called `subchain` and a folder `scripts`. We will need to perform changes in each of them.

### The `job_settings` file
Use your favorite text editor to open the `job_settings`  file.  Move to line 23 and modify the variable EXPDIR to match the experiment name you chose (GANANAtutorial in our case)
```
EXPID=GANANAtutorial
```
Now move to line **31**. Here and in the following lines (up to 41) you can set up the path to various directories. In particular, you need to change the value `DATADIR` directory with the path of your grid data and the value of `ICONDIR` to the path to your **folder containing the ICON bin directory**  (i.e., not directly to the binary, not directly the bin directory!!). Other variables defined in these lines can be left as they are, as they all defined paths relative to the spice main folder. 

Now move to lines **50-51**. Change the experiment start and end date according to your experiment. 

Now move to line **81**, where you should see the `PROJECT_ACCOUNT` variable. Modify it inserting the slurm account that you intend to use to launch jobs. 

Now move to line **96-98**. Comment or delete them. 

Now move to line **101**. Replace the path to the python folder with this one

Now move to line **149** and **154**, where you can see the two variables `ITYPE_SAMOVAR=1` and `ITYPE_SAMOVAR_TS=1`.  The workflow was tested only with both variables set to 0, so I suggest to change them to zero if you don't need that particular part of the workflow.
```
PYTHON=/home/apps/SPACK/spack/opt/spack/linux-almalinux8-cascadelake/oneapi-2024.0.0/python-3.11.9-avs7oxev37lg7lf23koz7ozr6oph534s/bin/python
```
Now move to line **162**, and modify the `GCM_DATADIR` by adding the path to your GCM data.

Now move to lines **172-200**. In these lines you have to insert the paths to several grid files needed in the run. Follow the instructions in the comments and modify them according to your needs. 

The lines **202-250** refer to specific ICON settings, and have to be changed depending on the problem you want to solve.

The lines from **254** on refer to the parallelization settings. You can play and modify them to find the optimal values for solving your problem rapidly. In the test phase we used the configuration pasted in the following box. Different values of `NUM_IO_PROCS`, `NUM_RESTARTS_PROCS` or `NUM_PREFETCH_PROCS` could cause problem, more information and experiments will arrive soon. 
```
#
#===============================================================================
#   Parallelization settings
#===============================================================================
#


#... prep.job.sh batch settings:
PARTITION_PREP=standard
TASKS_PREP=48                       # mpiproc per node (also 128 gives good performance)
CPUS_PER_TASK_PREP=1
TIME_PREP="00-00:50:00"

#... conv2icon.job.sh settings
PARTITION_CONV2ICON=standard
TASKS_CONV2ICON=48
OMP_THREADS_CONV2ICON=0
NODES_CONV2ICON=1    # conv2icon.job.sh can be run only on 1 node presently
TIME_CONV2ICON="00-01:00:00"


#...icon.job.sh settings
PARTITION_ICON=standard
NODES_ICON=3                     #Number of nodes
NTASKS_PER_NODE=48
#NUM_THREADS_ICON=0 ! we did not compile ICON with OpenMP
#!!!!!!! Presently the radiation scheme ECRAD does not work properly with OpenMP                    !!!!!!
#!!!!!!! The use of openMP (NUM_THREAD_ICON>0) currently does not guarantee bitidentical results    !!!!!!!

#NUM_THREAD_ICON=0           #Number of OpenMP threads (OpenMP has problems e

TIME_ICON=00-08:00:00       #requested time in batch queue for ICON simulation
NUM_IO_PROCS=1                #num_io_procs: 1,2 specified in namelist parallel_nml (icon.job.sh) MINIMUM IS 1!!
NUM_RESTART_PROCS=0             #num_restart_procs  !number of processors for restart
NUM_PREFETCH_PROC=1              #num_prefetch_proc         !number of processors for LBC prefetching

#... arch.job.sh batch settings:
TASKS_ARCH=48                       # mpiproc per node
NCPUS_PER_TASK_ARCH=1              # ncpu per node (no make sence to increase)
TIME_ARCH="00:50:00"

#... post.job.sh batch settings:
TASKS_POST=48                       # mpiproc per node
OMP_THREADS_POST=0
PARTITION_POST=standard
TIME_POST="00-00:30:00"
TIME_POST_YEARLY="00-02:00:00"
```

### The scripts into the `scripts` folder
Within the `scripts` folder you will see several scripts, and we need to perform modifications in each of them

#### The `prep.job.sh` script
Open the file and move to line **82**, in which id performed the extraction of input data. Modify the line so to match the name of your input files, e.g., during the test phase, we did it like this:
```
# from
#tar -C ${INPDIR}/${YYYY}_${MM} -xf ${GCM_DATADIR}/year${YYYY}/ERAINT_${YYYY}_${MM}.tar
# to
tar -C ${INPDIR}/${YYYY}_${MM} -xf ${GCM_DATADIR}/year${YYYY}/${GCM_NAME}_${GCM_SCENARIO}_${YYYY}_${MM}.tar
``` 
Now move to line **127** and change
```
${UTILS_BINDIR}/ccaf2icaf ${FILE} 1       
```
to
```
${UTILS_BINDIR}/ccaf2icaf ${FILE} 0
```
Now go to lines from **197** to **201**, in which other extractions of input data are performed. Change these commands to match the path and filename structure of your input data.

#### The `conv2icon.job.sh` file
Somewhere at the beginning of the script, before running the main commands, insert
```
export HDF5_USE_FILE_LOCKING=FALSE
```

Now replace lines from **92** to **100**, i.e. between the lines
```
 ${CDO} -s selname,FR_LAND ${EXTPAR} ${INIDIR}/output_FR_LAND.nc
```
and 
```
${CDO} -s setctomiss,0. -ltc,1. ${INIDIR}/output_FR_LAND.nc ${INIDIR}/output_ocean_area.nc
```
 with the following commands
```
${NCO_BINDIR}/ncecat -O -u time ${INIDIR}/output_FR_LAND.nc ${INIDIR}/output_FR_LAND.nc
${CDO} -s settaxis,${YYYY}-${MM}-01,00:00:00,1month ${INIDIR}/output_FR_LAND.nc ${INIDIR}/output_FR_LAND_t.nc
mv ${INIDIR}/output_FR_LAND_t.nc ${INIDIR}/output_FR_LAND.nc
${CDO} -s setctomiss,0. -ltc,0.5  ${INIDIR}/input_FR_LAND.nc ${INIDIR}/input_ocean_area.nc
${CDO} -s  setctomiss,0. -gec,0.5 ${INIDIR}/input_FR_LAND.nc ${INIDIR}/input_land_area.nc
```
Now move to the lines
```
#... Create file with ICON grid information for CDO:
${CDO} -s selgrid,2 ${LAM_GRID} ${INIDIR}/triangular-grid.nc
```
and replace the "2" in the cdo command with a "1"
```
${CDO} -s selgrid,1 ${LAM_GRID} ${INIDIR}/triangular-grid.nc
```
Now move to the sequence of commands 
```
${NCO_BINDIR}/ncks -h -A ${OUTDIR}/${YYYY_MM}/tmp_output_l.nc ${INIDIR}/${GCM_PREFIX}${YDATE_START}_ini.nc
${NCO_BINDIR}/ncks -h -A ${OUTDIR}/${YYYY_MM}/tmp_output_ls.nc ${INIDIR}/${GCM_PREFIX}${YDATE_START}_ini.nc
${NCO_BINDIR}/ncks -h -A ${OUTDIR}/output_lsm.nc  ${INIDIR}/${GCM_PREFIX}${YDATE_START}_ini.nc
```
(which were in lines **139**-**141** before the previous modifications and add the "-4" flag
```
${NCO_BINDIR}/ncks -h -4 -A ${OUTDIR}/${YYYY_MM}/tmp_output_l.nc ${INIDIR}/${GCM_PREFIX}${YDATE_START}_ini.nc
${NCO_BINDIR}/ncks -h -4 -A ${OUTDIR}/${YYYY_MM}/tmp_output_ls.nc ${INIDIR}/${GCM_PREFIX}${YDATE_START}_ini.nc
${NCO_BINDIR}/ncks -h -4 -A ${OUTDIR}/output_lsm.nc  ${INIDIR}/${GCM_PREFIX}${YDATE_START}_ini.nc
```
and to the same on the line
```
${NCO_BINDIR}/ncks -h -A ${INIDIR}/hyai_hybi.nc ${GCM_PREFIX}${YDATE_START}_ini.nc
```
(which was line 163 before modifications)
```
${NCO_BINDIR}/ncks -h -4 -A ${INIDIR}/hyai_hybi.nc ${GCM_PREFIX}${YDATE_START}_ini.nc
```

Now move to the part of the script that before modifications was lines **178**-**192**:
```
#... We're using ERA5 T_SKIN as SST - take the ocean part only:
${CDO} -s div -selname,SST ${OUTDIR}/${YYYY_MM}/${GCM_PREFIX}${YYYY}${MM}_tmp.nc  ${INIDIR}/input_ocean_area.nc \
${OUTDIR}/${YYYY_MM}/SST_${YYYY}${MM}_tmp.nc
${CDO} -s setmisstodis  ${OUTDIR}/${YYYY_MM}/SST_${YYYY}${MM}_tmp.nc  \
    ${OUTDIR}/${YYYY_MM}/SST_${YYYY}${MM}_tmp2.nc
${CDO} -s merge ${OUTDIR}/${YYYY_MM}/SST_${YYYY}${MM}_tmp2.nc  \
${OUTDIR}/${YYYY_MM}/SIC_${YYYY}${MM}_tmp.nc  \
${OUTDIR}/${YYYY_MM}/SST-SIC_${YYYY}${MM}_tmp.nc
${CDO} -s -P ${OMP_THREADS_CONV2ICON} ${GCM_REMAP},${INIDIR}/triangular-grid.nc \
${OUTDIR}/${YYYY_MM}/SST-SIC_${YYYY}${MM}_tmp.nc  \
${OUTDIR}/${YYYY_MM}/LOWBC_${YYYY}_${MM}.nc
```

and change them to
```
${CDO} -s setmisstodis -selname,SST ${OUTDIR}/${YYYY_MM}/${GCM_PREFIX}${YYYY}${MM}_tmp.nc  \
                                 ${OUTDIR}/${YYYY_MM}/SST_${YYYY}${MM}_tmp.nc
${CDO} -s merge ${OUTDIR}/${YYYY_MM}/SST_${YYYY}${MM}_tmp.nc  \
               ${OUTDIR}/${YYYY_MM}/SIC_${YYYY}${MM}_tmp.nc  \
               ${OUTDIR}/${YYYY_MM}/SST-SIC_${YYYY}${MM}_tmp.nc
 ${CDO} -s -P ${OMP_THREADS_CONV2ICON} ${GCM_REMAP},${INIDIR}/triangular-grid.nc \
               ${OUTDIR}/${YYYY_MM}/SST-SIC_${YYYY}${MM}_tmp.nc  \
               ${OUTDIR}/${YYYY_MM}/SST-SIC_${YYYY}${MM}_${GCM_REMAP}_tmp.nc
${CDO} -s -P ${OMP_THREADS_CONV2ICON} ${GCM_REMAP},${INIDIR}/triangular-grid.nc \
               ${OUTDIR}/${YYYY_MM}/SST-SIC_${YYYY}${MM}_tmp.nc  \
               ${OUTDIR}/${YYYY_MM}/SST-SIC_${YYYY}${MM}_${GCM_REMAP}_tmp.nc
${CDO} -s div ${OUTDIR}/${YYYY_MM}/SST-SIC_${YYYY}${MM}_${GCM_REMAP}_tmp.nc ${INIDIR}/output_ocean_area.nc \
               ${OUTDIR}/${YYYY_MM}/LOWBC_${YYYY}_${MM}.nc
```

The last modification is in what previously was line **212**
```
${NCO_BINDIR}/ncks -h -A ${INIDIR}/hyai_hybi.nc ${OUTDIR}/${YYYY_MM}/${FILEOUT}_lbc.nc
```
and add the "-4" flag
```
${NCO_BINDIR}/ncks -h -4 -A ${INIDIR}/hyai_hybi.nc ${OUTDIR}/${YYYY_MM}/${FILEOUT}_lbc.nc
```

#### The `icon.job.sh` script 
Change the initial part of the script before lines
```
  #################################################
  #  Pre-Settings
  #################################################
```
 to this
```
  export UCX_IB_ADDR_TYPE=ib_global
  export HCOLL_ENABLE_MCAST_ALL="1"
  export HCOLL_MAIN_IB=mlx5_0:1
  export UCX_NET_DEVICES=mlx5_0:1
  export UCX_TLS=mm,knem,cma,dc_mlx5,dc_x,self
  export UCX_UNIFIED_MODE=y
  export HDF5_USE_FILE_LOCKING=FALSE
  export MALLOC_TRIM_THRESHOLD_="-1"
  export MKL_ENABLE_INSTRUCTIONS=AVX2
  export MKL_DEBUG_CPU_TYPE=5
  export UCX_HANDLE_ERRORS=bt
  export MKL_NUM_THREADS=1
  export MKL_DYNAMIC=FALSE
  export FORT_BUFFERED=false
  export I_MPI_EXTRA_FILESYSTEM=0
  export I_MPI_HYDRA_DEBUG=on
  export VE_TRACEBACK=VERBOSE
  export I_MPI_DEBUG=5
  ulimit -s unlimited
  export LD_LIBRARY_PATH=/home/apps/SPACK/spack/opt/spack/linux-almalinux8-cascadelake/gcc-13.2.0/gcc-runtime-13.2.0-xrovdbememhbpyitsfysdgqyer64fdzf/lib:${LD_LIBRARY_PATH}
    export I_MPI_HYDRA_BOOTSTRAP=slurm
  unset I_MPI_PMI_LIBRARY
  unset I_MPI_PMI
  export I_MPI_OFI_PROVIDER=tcp          # fallback to TCP
  export ROMIO_HINTS=/dev/null           # disable ROMIO hints
  export HDF5_USE_FILE_LOCKING=FALSE     # already set, keep it
  export I_MPI_ADJUST_REDUCE=1
  export GFORTRAN_UNBUFFERED_PRECONNECTED=y
  JOBID=${SLURM_JOBID}
```
Now move  to the namelist section. Search the line `&grid_nml` and modify that namelist to
```
&grid_nml
  dynamics_grid_filename  = ${dynamics_grid_filename}
  radiation_grid_filename = ${radiation_grid_filename}
  dynamics_parent_grid_id = 0,1
  lredgrid_phys           = .true.,.true.,   ! This is a fixone
  lfeedback               = .true.
  l_limited_area          = .true.
  ifeedback_type          = 2
  start_time              = 0., 1800.
/
```
Now, go to the output namelists and modify them according to the needs of your problem. In the testing phase it looked like this
```
&output_nml
!-----------------------------------------------------------Output Namelist 1
  filetype             = 4
  dom                  = 1
  output_bounds        = ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}
  steps_per_file       = 1
  mode                 = 1
  include_last         = .TRUE.
  steps_per_file_inclfirst     = .FALSE.
  output_filename      = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
  filename_format      = '<output_filename>_<datetime2>'
!  ml_varlist           = 'temp','u','v','w','pres'
      ml_varlist                   = 'alb_si','c_t_lk','fr_land','fr_seaice','freshsnow','gz0','h_ice','h_ml_lk','h_snow','pres','qc','qi','qr','qs','qv',
                                      'qv_s','rho_snow','smi','t_bot_lk','t_g','t_ice','t_mnw_lk','t_snow','t_so','t_wml_lk','temp','tke','u','v','w','w_i',
                                      'w_snow','w_so_ice','z_ifc','plantevap','hsnow_max','snow_age'                         !'group:mode_iniana'
  output_grid          = .TRUE.
/
&output_nml
!-----------------------------------------------------------Output Namelist 2
  filetype                     =  4            ! output format: 2=GRIB2, 4=NETCDFv2
  dom                          =  1            ! write all domains
  output_bounds                =  ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}.    ! start, end, increment
  steps_per_file               =  1
  mode                         =  1            ! 1: forecast mode (relative t-axis)
               ! 2: climate mode (absolute t-axis)
  include_last                 = .TRUE.
  steps_per_file_inclfirst     = .FALSE.
  output_filename              = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
  filename_format              = '<output_filename>_<datetime2>'
      operation                    = "${OPERATION[$((${NOUTDIR}-1))]}"
 ml_varlist                   = 'w_i','w_so_ice','freshsnow','rho_snow','w_snow','t_s','t_g','w'
 output_grid                  =  .TRUE.
! stream_partitions_ml         =  2
/
&output_nml
!!-----------------------------------------------------------Output Namelist 3
  filetype                     =  4            ! output format: 2=GRIB2, 4=NETCDFv2
  dom                          =  1            ! write all domains
  output_bounds                =  ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}.    ! start, end, increment
  steps_per_file               =  1
  mode                         =  1            ! 1: forecast mode (relative t-axis)
       ! 2: climate mode (absolute t-axis)
  include_last                 = .TRUE.
  steps_per_file_inclfirst     = .FALSE.
  output_filename              = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
  filename_format              = '<output_filename>_<datetime2>'
      operation                    = "${OPERATION[$((${NOUTDIR}-1))]}"
  ml_varlist                   = 'clct','clct_mod','pres_msl','pres_sfc','qv_2m','rh_2m',
                                     'runoff_g','runoff_s','rain_con','snow_con','rain_gsp','snow_gsp','tot_prec',
                                     't_2m','td_2m','t_s','t_g','t_sk','tmax_2m','tmin_2m','u_10m','v_10m','gust10','sp_10m','snow_melt','w_so','t_so','w_snow','t_snow','hpbl'
  output_grid                  =  .TRUE.
 ! stream_partitions_ml         =  2
 /
 &output_nml
!-----------------------------------------------------------Output Namelist 4
  filetype                     =  4            ! output format: 2=GRIB2, 4=NETCDFv2
  dom                          =  1            ! write all domains
  output_bounds                =  ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}.    ! start, end, increment
  steps_per_file               =  1
  mode                         =  1            ! 1: forecast mode (relative t-axis)
       ! 2: climate mode (absolute t-axis)
  include_last                 = .TRUE.
  steps_per_file_inclfirst     = .FALSE.
  output_filename              = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
  filename_format              = '<output_filename>_<datetime2>'
      operation                    = "${OPERATION[$((${NOUTDIR}-1))]}"
  ml_varlist                   = 'dursun', 'lai', 'plcov', 'rootdp','gz0'
  output_grid                  =  .TRUE.
! stream_partitions_ml       =  2
/
&output_nml
!-----------------------------------------------------------Output Namelist 5
  filetype                     =  4            ! output format: 2=GRIB2, 4=NETCDFv2
  dom                          =  1            ! write all domains
  output_bounds                =  ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}.    ! start, end, increment
  steps_per_file               =  1
  mode                         =  1            ! 1: forecast mode (relative t-axis)
       ! 2: climate mode (absolute t-axis)
  include_last                 = .TRUE.
  steps_per_file_inclfirst     = .FALSE.
  output_filename              = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
  filename_format              = '<output_filename>_<datetime2>'
      operation                    = "${OPERATION[$((${NOUTDIR}-1))]}"
  ml_varlist                   = 'tqc','tqi','tqv','tqr','tqs','cape_ml','cape','h_snow'
  output_grid                  =  .TRUE.
! stream_partitions_ml       =  2
/
&output_nml
!-----------------------------------------------------------Output Namelist 6
  filetype                     =  4            ! output format: 2=GRIB2, 4=NETCDFv2
  dom                          =  1            ! write all domains
  output_bounds                =  ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}.    ! start, end, increment
  steps_per_file               =  1
  mode                         =  1            ! 1: forecast mode (relative t-axis)
       ! 2: climate mode (absolute t-axis)
  include_last                 = .TRUE.
  steps_per_file_inclfirst     = .FALSE.
  output_filename              = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
  filename_format              = '<output_filename>_<datetime2>p'
      operation                    = "${OPERATION[$((${NOUTDIR}-1))]}"
  pl_varlist                   = 'geopot','qv','rh','temp','u','v','w'
  p_levels                     =   60000., 70000., 75000., 85000., 92500., 100000
  output_grid                  =  .TRUE.
! stream_partitions_ml       =  2
/
&output_nml
!-----------------------------------------------------------Output Namelist 7
 filetype                     =  4            ! output format: 2=GRIB2, 4=NETCDFv2
 dom                          =  1            ! write all domains
 output_bounds                =  ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}.   ! start, end, increment
 steps_per_file               =  1
 mode                         =  1            ! 1: forecast mode (relative t-axis)
                                              ! 2: climate mode (absolute t-axis)
 include_last                 = .TRUE.
 steps_per_file_inclfirst     = .FALSE.
 output_filename              = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
 filename_format              = '<output_filename>_<datetime2>z'
     operation                    = "${OPERATION[$((${NOUTDIR}-1))]}"
 hl_varlist                   = 'pres','qv','rh','temp','u','v','w'
 h_levels                     = 50.0, 100.0, 150.0, 200.0, 250.0, 300.0, 350.0, 400.0
 output_grid                  =  .TRUE.
! stream_partitions_ml         =  2
/
&output_nml
!-----------------------------------------------------------Output Namelist 8
 filetype                     =  4            ! output format: 2=GRIB2, 4=NETCDFv2
 dom                          =  1            ! write all domains
 output_bounds                =  ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}.    ! start, end, increment
 steps_per_file               =  1
 mode                         =  1            ! 1: forecast mode (relative t-axis)
      ! 2: climate mode (absolute t-axis)
 include_last                 = .TRUE.
 steps_per_file_inclfirst     = .FALSE.
 output_filename              = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
 filename_format              = '<output_filename>_<datetime2>'
     operation                    = "${OPERATION[$((${NOUTDIR}-1))]}"
  ml_varlist                   = 'qhfl_s','lhfl_s','shfl_s','thu_s','sob_s',
                                     'sob_t','sod_t','sodifd_s','thb_s','sou_s','thb_t','umfl_s','vmfl_s'
 output_grid                  =  .TRUE.
! stream_partitions_ml       =  2
/
&output_nml
!-----------------------------------------------------------Output Namelist 9
  filetype                     =  4            ! output format: 2=GRIB2, 4=NETCDFv2
  dom                          =  1            ! write all domains
  output_bounds                =  ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}.    ! start, end, increment
  steps_per_file               =  1
  mode                         =  1            ! 1: forecast mode (relative t-axis)
       ! 2: climate mode (absolute t-axis)
  include_last                 = .TRUE.
  steps_per_file_inclfirst     = .FALSE.
  output_filename              = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
  filename_format              = '<output_filename>_<datetime2>'
      operation                    = "${OPERATION[$((${NOUTDIR}-1))]}"
  ml_varlist                   = 'sodifd_s','sob_s','sou_s','thb_s'
  output_grid                  =  .TRUE.
/
&output_nml
!-----------------------------------------------------------Output Namelist 10
  filetype                     =  4              ! output format: 2=GRIB2, 4=NETCDFv2
  dom                          =  1              ! write all domains
  output_bounds                =  0., 0., 3600.  ! start, end, increment
  steps_per_file               =  1
  mode                         =  1              ! 1: forecast mode (relative t-axis)
       ! 2: climate mode (absolute t-axis)
  include_last                 = .FALSE.
  steps_per_file_inclfirst     = .FALSE.
  output_filename              = '${OUTDIR}/icon'
  filename_format              = '<output_filename>_<datetime2>c'   ! file name base
  ml_varlist                   = 'z_ifc','z_mc','topography_c','fr_land','depth_lk', 'fr_lake', 'soiltyp','urb_isa','ahf'
  output_grid                  =  .TRUE.
!  stream_partitions_ml               =  2
/
```
Moreover, modify the `meteogram_output`namelist (just after the output namelists) according to your needs, or comment it out if you are not interested in meteograms.
Modify all the other namelists based on your scientific needs. The full namelist used in the test phase is presented in the appendix, although it is not guaranteed to be the one you need for your experiments.

Now move to the lines 
```
srun -l \
   --propagate=STACK,CORE \
   --hint=nomultithread \
   --distribution=block:cyclic \
   --distribution=block:block \
   --kill-on-bad-exit=1 \
   ${BINARY_ICON}
```
and replace them with these ones:
```
export PATH=/home/apps/SPACK/spack/opt/spack/linux-cascadelake/intel-oneapi-mpi-2021.17.1-z55gh32q7gbnq7wwpqgvd4tieyarmsvf/mpi/2021.17/bin:${PATH}
mpiexec.hydra -l -np ${SLURM_NTASKS} \
              -genv LD_LIBRARY_PATH ${LD_LIBRARY_PATH} \
              ${BINARY_ICON}
```
#### The `arch.job.sh` file
Open the  `arch.job.sh` and move to line **57**. Delete or comment the lines in which the directory structure is built, i.e., the following lines
```
                                                                                                                   |  #
  #... Create output directory:                                                                                    |  ##... Create output directory:
  if [ ! -d ${OUTDIR}/${YYYY_MM} ]                                                                                 |  #if [ ! -d ${OUTDIR}/${YYYY_MM} ]
  then                                                                                                             |  #then
    mkdir -p ${OUTDIR}/${YYYY_MM}                                                                                  |  #  mkdir -p ${OUTDIR}/${YYYY_MM}
    NOUTDIR=1                                                                                                      |  #  NOUTDIR=1
    while [ ${NOUTDIR} -le ${#HOUT_INC[@]} ]                                                                       |  #  while [ ${NOUTDIR} -le ${#HOUT_INC[@]} ]
    do                                                                                                             |  #  do
      if [ ${NOUTDIR} -ne ${NESTING_STREAM} ]                                                                      |  #    if [ ${NOUTDIR} -ne ${NESTING_STREAM} ]
      then                                                                                                         |  #    then
        mkdir -p ${OUTDIR}/${YYYY_MM}/out$(printf %02d ${NOUTDIR})                                                 |  #      mkdir -p ${OUTDIR}/${YYYY_MM}/out$(printf %02d ${NOUTDIR})
      fi                                                                                                           |  #    fi
      ((NOUTDIR++))                                                                                                |  #    ((NOUTDIR++))
    done                                                                                                           |  #  done
  fi
```
Now move to the line
```
if [ -z "$(ls -A ${INPDIR}/${YYYY_MM}/out${NOUTDIR2})" ] # directory is empty
```
and replace it with these lines
```
# get the string before the dot of the first file in the input directory
# this works only if the file names are of the same kind!
SUFFIX=$(ls -1 ${INPDIR}/${YYYY_MM}/out${NOUTDIR2} | head -n 1 |  cut -d. -f1)
if [ -z "$SUFFIX" ]  # $SUFFIX is empty, i.e. directory is empty
```

Move to the following lines
```
BASENAME=$(ls -1 ${INPDIR}/${YYYY_MM}/out${NOUTDIR2} | head -n 1 |  cut -d. -f1)
case ${BASENAME: -1} in
```
and replace them with this one
```
case ${SUFFIX: -1} in
```

Move to the lines
```
#... Copy the c file to the arch output directory
cp ${SCRATCHDIR}/${EXPID}/output/icon/icon_${YDATE_START:0:8}T${YDATE_START:8:2}0000Zc.nc ${OUTDIR}/
```
and comment or delete them.


Move to the line
```                                                    
if [ ${NOUTDIR} -ne ${NESTING_STREAM} ]
    then
```
replace the content of the *if* statement from the `then` to the `fi` with 
```
  COUNTPP=0
  for FILE in $(ls -1 ${INPDIR}/${YYYY_MM}/out${NOUTDIR2}/*)
  do

    iconcor ${COUNTPP} ${HOUT_INC[$((${NOUTDIR}-1))]} ${FILE} ${OPERATION[$((${NOUTDIR}-1))]}  &

    (( COUNTPP=COUNTPP+1 ))
    if [ ${COUNTPP} -ge ${MAXPP} ]
    then
      COUNTPP=0
      wait
    fi

  done
  wait
``` 


Now move to the line
```
cp -r ${INPDIR}/${YYYY_MM}/out$(printf %02d ${NESTING_STREAM}) ${OUTDIR}/${YYYY_MM}/nesting
```
and replace the copy with a move operation to save storage space
```
mv ${INPDIR}/${YYYY_MM}/out$(printf %02d ${NESTING_STREAM}) ${INPDIR}/${YYYY_MM}/nesting
```

Then change the line
```
cd ${OUTDIR}/${YYYY_MM}
```
with 
```
cd ${INPDIR}/${YYYY_MM}
``` 
You need to do this change **5 times**,  each one is ~25-30 lines after the previous one.
Then move to the line
```
#rm nesting/*.ncz
```
and uncomment it (so that we can save some storage space). 

Now move to the following `else`statement and remove the line
```
cd ${OUTDIR}/${YYYY_MM}
```

#### The `post.job.sh`file
Go at line **151** and delete or remove the following lines
```
  cd ${INPDIR}/${YYYY_MM}/
  if [ -n "$(ls -A  */icon_*.ncz 2>/dev/null)" ]
  then
    export COUNTPP=0
    echo  ${INPDIR}/${YYYY_MM}/ contains ncz-files which have to be unzipped befores timeseries can be build
    for file in */icon_*.ncz ; do
      ofile=${file%ncz}nc
      nccopy -6 $file $ofile &
      (( COUNTPP=COUNTPP+1 ))
      if [[ ${COUNTPP} -ge ${MAXPP} ]]
      then
        COUNTPP=0
        wait
      fi
      wait
    done
    echo unzipping of ncz-files in ${INPDIR}/${YYYY_MM}/ done
    rm ${INPDIR}/${YYYY_MM}/*/*ncz
  fi
```

Now consider the block starting with the line
```
ts_command_list=(
```
(it should be around line **183** if you did not delte the previous lines). From this line, a collection of timeseries commands start. Comment or uncomment the ones you need, according to your  output namelists from the previous scripts.

Now move to the line which states 
```
#... time series part 3
```
and comment everything up to the line
```
#... time series part 4
```
In the timeseries part 4 section new operations on timeseries are performed. Comment or uncomment the ones you need for your experiment. 

Then look for the line
```
chgrp $PROJECT_ACCOUNT ${OUTDIR}/${YYYY_MM}/*
```
and comment it out or delete it.
 Move to the line
 ```
FILELIST=$(ls -1 ${WORKDIR}/${EXPID}/post/${YYYY_MM}/*ts.nc | grep -v uncorr)
```
and replace it with the following
```
FILELIST=$(ls -1 ${WORKDIR}/${EXPID}/post/${YYYY_MM}/*.nc)
```


## Running a SPICE experiment
Once you have set up the files in the `scripts`folder and the `job_settings`file, you can start your SPICE experiment with the command
```
./subchain start
```
If everything is correctly set up, workflow will reach the end without additional effort. If something went wrong, navgiate to the folder `${SPDIR}/chain/work/${EXPID}/joblogs`  where you can find the logs of each part of the subchain, which will give you information on which parts of the workflow caused the run to stop. Once you corrected the error, you can re-launch the chain from the point it broke launching one of the following:
```
./subchain prep
./subchain conv2icon
./subchain icon
./subchain arch
./subchain post YYYYMMDDHH
```
where in the case of `post`you have to specify from which part of the simulation you have to start again. 








## APPENDIX: The `icon.job.sh` namelist
The following is the complete namelist snippet of the `icon.job.sh` used in the test phase. It is not guranateed that is suitable also for your needs. This is the version which was used in CMCC cluster CASSANDRA by REHMI division.
```
&parallel_nml
  nproma            =  8
  p_test_run        = .false.
  l_test_openmp     = .true.
  l_log_checks      = .true.
  num_io_procs      = ${NUM_IO_PROCS}
  num_restart_procs = ${NUM_RESTART_PROCS}
  num_prefetch_proc = ${NUM_PREFETCH_PROC}
  iorder_sendrecv   = 3
  num_dist_array_replicas      = 4
  proc0_shift       = 0
  use_omp_input   = .true.
/
&grid_nml
  dynamics_grid_filename  = ${dynamics_grid_filename}
  radiation_grid_filename = ${radiation_grid_filename}
  dynamics_parent_grid_id = 0,1
  lredgrid_phys           = .true.,.true.,   ! This is a fixone
  lfeedback               = .true.
  l_limited_area          = .true.
  ifeedback_type          = 2
  start_time              = 0., 1800.
/
&initicon_nml
  init_mode                    = 2
  lread_ana                    = .false. ! (T) Read dwdana
  ifs2icon_filename            = "${inidatafile##*/}"
  zpbl1       = 500.
  zpbl2       = 1000.
  ltile_init=.TRUE.
  ltile_coldstart=.true.
/
&limarea_nml
  itype_latbc     = 1
  dtime_latbc     = $(${PYTHON} -c "print(${HINCBOUND}*3600.)")
  latbc_varnames_map_file = 'dict.latbc'
  latbc_path      = '${INPDIR}/${YYYY_MM}'
  latbc_filename  = '${GCM_PREFIX}<y><m><d><h>_lbc.nc'
! latbc_contains_qcqi = .false.     ! = .true.  if  qc, qi are in latbc
  latbc_contains_qcqi = .true.      ! = .true.  if  qc, qi are in latbc
                                    ! = .false. if qc, qi are not in latbc
/
&io_nml
  itype_pres_msl               = 5
  itype_rh                     = 1
  precip_interval              = "${PRECIP_INTERVAL}"
  runoff_interval              = "${RUNOFF_INTERVAL}"
  sunshine_interval            = "${SUNSHINE_INTERVAL}"
  maxt_interval                = "${MAXT_INTERVAL}"
  gust_interval                = $(${CFU} p2sec ${GUST_INTERVAL})
  melt_interval                = "${MELT_INTERVAL}"
  lmask_boundary               = .true.
  restart_write_mode="${RESTART_WRITE_MODE}"
/
&output_nml
!-----------------------------------------------------------Output Namelist 1
  filetype             = 4
  dom                  = 1
  output_bounds        = ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}
  steps_per_file       = 1
  mode                 = 1
  include_last         = .TRUE.
  steps_per_file_inclfirst     = .FALSE.
  output_filename      = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
  filename_format      = '<output_filename>_<datetime2>'
!  ml_varlist           = 'temp','u','v','w','pres'
      ml_varlist                   = 'alb_si','c_t_lk','fr_land','fr_seaice','freshsnow','gz0','h_ice','h_ml_lk','h_snow','pres','qc','qi','qr','qs','qv',
                                      'qv_s','rho_snow','smi','t_bot_lk','t_g','t_ice','t_mnw_lk','t_snow','t_so','t_wml_lk','temp','tke','u','v','w','w_i',
                                      'w_snow','w_so_ice','z_ifc','plantevap','hsnow_max','snow_age'                         !'group:mode_iniana'
  output_grid          = .TRUE.
/
&output_nml
!-----------------------------------------------------------Output Namelist 2
  filetype                     =  4            ! output format: 2=GRIB2, 4=NETCDFv2
  dom                          =  1            ! write all domains
  output_bounds                =  ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}.    ! start, end, increment
  steps_per_file               =  1
  mode                         =  1            ! 1: forecast mode (relative t-axis)
               ! 2: climate mode (absolute t-axis)
  include_last                 = .TRUE.
  steps_per_file_inclfirst     = .FALSE.
  output_filename              = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
  filename_format              = '<output_filename>_<datetime2>'
      operation                    = "${OPERATION[$((${NOUTDIR}-1))]}"
 ml_varlist                   = 'w_i','w_so_ice','freshsnow','rho_snow','w_snow','t_s','t_g','w'
 output_grid                  =  .TRUE.
! stream_partitions_ml         =  2
/
&output_nml
!!-----------------------------------------------------------Output Namelist 3
  filetype                     =  4            ! output format: 2=GRIB2, 4=NETCDFv2
  dom                          =  1            ! write all domains
  output_bounds                =  ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}.    ! start, end, increment
  steps_per_file               =  1
  mode                         =  1            ! 1: forecast mode (relative t-axis)
       ! 2: climate mode (absolute t-axis)
  include_last                 = .TRUE.
  steps_per_file_inclfirst     = .FALSE.
  output_filename              = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
  filename_format              = '<output_filename>_<datetime2>'
      operation                    = "${OPERATION[$((${NOUTDIR}-1))]}"
  ml_varlist                   = 'clct','clct_mod','pres_msl','pres_sfc','qv_2m','rh_2m',
                                     'runoff_g','runoff_s','rain_con','snow_con','rain_gsp','snow_gsp','tot_prec',
                                     't_2m','td_2m','t_s','t_g','t_sk','tmax_2m','tmin_2m','u_10m','v_10m','gust10','sp_10m','snow_melt','w_so','t_so','w_snow','t_snow','hpbl'
  output_grid                  =  .TRUE.
 ! stream_partitions_ml         =  2
 /
&output_nml
!-----------------------------------------------------------Output Namelist 4
  filetype                     =  4            ! output format: 2=GRIB2, 4=NETCDFv2
  dom                          =  1            ! write all domains
  output_bounds                =  ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}.    ! start, end, increment
  steps_per_file               =  1
  mode                         =  1            ! 1: forecast mode (relative t-axis)
       ! 2: climate mode (absolute t-axis)
  include_last                 = .TRUE.
  steps_per_file_inclfirst     = .FALSE.
  output_filename              = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
  filename_format              = '<output_filename>_<datetime2>'
      operation                    = "${OPERATION[$((${NOUTDIR}-1))]}"
  ml_varlist                   = 'dursun', 'lai', 'plcov', 'rootdp','gz0'
  output_grid                  =  .TRUE.
! stream_partitions_ml       =  2
/
&output_nml
!-----------------------------------------------------------Output Namelist 5
  filetype                     =  4            ! output format: 2=GRIB2, 4=NETCDFv2
  dom                          =  1            ! write all domains
  output_bounds                =  ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}.    ! start, end, increment
  steps_per_file               =  1
  mode                         =  1            ! 1: forecast mode (relative t-axis)
       ! 2: climate mode (absolute t-axis)
  include_last                 = .TRUE.
  steps_per_file_inclfirst     = .FALSE.
  output_filename              = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
  filename_format              = '<output_filename>_<datetime2>'
      operation                    = "${OPERATION[$((${NOUTDIR}-1))]}"
  ml_varlist                   = 'tqc','tqi','tqv','tqr','tqs','cape_ml','cape','h_snow'
  output_grid                  =  .TRUE.
! stream_partitions_ml       =  2
/
&output_nml
!-----------------------------------------------------------Output Namelist 6
  filetype                     =  4            ! output format: 2=GRIB2, 4=NETCDFv2
  dom                          =  1            ! write all domains
  output_bounds                =  ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}.    ! start, end, increment
  steps_per_file               =  1
  mode                         =  1            ! 1: forecast mode (relative t-axis)
       ! 2: climate mode (absolute t-axis)
  include_last                 = .TRUE.
  steps_per_file_inclfirst     = .FALSE.
  output_filename              = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
  filename_format              = '<output_filename>_<datetime2>p'
      operation                    = "${OPERATION[$((${NOUTDIR}-1))]}"
  pl_varlist                   = 'geopot','qv','rh','temp','u','v','w'
  p_levels                     =   60000., 70000., 75000., 85000., 92500., 100000
  output_grid                  =  .TRUE.
! stream_partitions_ml       =  2
/
&output_nml
!-----------------------------------------------------------Output Namelist 7
 filetype                     =  4            ! output format: 2=GRIB2, 4=NETCDFv2
 dom                          =  1            ! write all domains
 output_bounds                =  ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}.   ! start, end, increment
 steps_per_file               =  1
 mode                         =  1            ! 1: forecast mode (relative t-axis)
                                              ! 2: climate mode (absolute t-axis)
 include_last                 = .TRUE.
 steps_per_file_inclfirst     = .FALSE.
 output_filename              = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
 filename_format              = '<output_filename>_<datetime2>z'
     operation                    = "${OPERATION[$((${NOUTDIR}-1))]}"
 hl_varlist                   = 'pres','qv','rh','temp','u','v','w'
 h_levels                     = 50.0, 100.0, 150.0, 200.0, 250.0, 300.0, 350.0, 400.0
 output_grid                  =  .TRUE.
! stream_partitions_ml         =  2
/
&output_nml
!-----------------------------------------------------------Output Namelist 8
 filetype                     =  4            ! output format: 2=GRIB2, 4=NETCDFv2
 dom                          =  1            ! write all domains
 output_bounds                =  ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}.    ! start, end, increment
 steps_per_file               =  1
 mode                         =  1            ! 1: forecast mode (relative t-axis)
      ! 2: climate mode (absolute t-axis)
 include_last                 = .TRUE.
 steps_per_file_inclfirst     = .FALSE.
 output_filename              = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
 filename_format              = '<output_filename>_<datetime2>'
     operation                    = "${OPERATION[$((${NOUTDIR}-1))]}"
  ml_varlist                   = 'qhfl_s','lhfl_s','shfl_s','thu_s','sob_s',
                                     'sob_t','sod_t','sodifd_s','thb_s','sou_s','thb_t','umfl_s','vmfl_s'
 output_grid                  =  .TRUE.
! stream_partitions_ml       =  2
/
&output_nml
!-----------------------------------------------------------Output Namelist 9
  filetype                     =  4            ! output format: 2=GRIB2, 4=NETCDFv2
  dom                          =  1            ! write all domains
  output_bounds                =  ${SSTART}, ${SNEXT}, ${SOUT_INC[$((NOUTDIR+=1))]}.    ! start, end, increment
  steps_per_file               =  1
  mode                         =  1            ! 1: forecast mode (relative t-axis)
       ! 2: climate mode (absolute t-axis)
  include_last                 = .TRUE.
  steps_per_file_inclfirst     = .FALSE.
  output_filename              = '${OUTDIR}/${YYYY_MM}/out$(printf %02d $NOUTDIR)/icon'
  filename_format              = '<output_filename>_<datetime2>'
      operation                    = "${OPERATION[$((${NOUTDIR}-1))]}"
  ml_varlist                   = 'sodifd_s','sob_s','sou_s','thb_s'
  output_grid                  =  .TRUE.
/
&output_nml
!-----------------------------------------------------------Output Namelist 10
  filetype                     =  4              ! output format: 2=GRIB2, 4=NETCDFv2
  dom                          =  1              ! write all domains
  output_bounds                =  0., 0., 3600.  ! start, end, increment
  steps_per_file               =  1
  mode                         =  1              ! 1: forecast mode (relative t-axis)
       ! 2: climate mode (absolute t-axis)
  include_last                 = .FALSE.
  steps_per_file_inclfirst     = .FALSE.
  output_filename              = '${OUTDIR}/icon'
  filename_format              = '<output_filename>_<datetime2>c'   ! file name base
  ml_varlist                   = 'z_ifc','z_mc','topography_c','fr_land','depth_lk', 'fr_lake', 'soiltyp','urb_isa','ahf'
  output_grid                  =  .TRUE.
!  stream_partitions_ml               =  2
/
!&meteogram_o!&meteogram_output_nml
!  lmeteogram_enabled = .TRUE.
!  ldistributed       = .FALSE.
!  loutput_tiles      = .TRUE.
!  n0_mtgrm           = 0
!  ninc_mtgrm         = 1
!  stationlist_tot = 50.050,  8.600, 'Frankfurt-Flughafen',
!                    52.220, 14.135, 'Lindenberg_Obs',
!                    52.167, 14.124, 'Falkenberg',
!                    47.800, 10.900, 'Hohenpeissenberg',
!                    53.630,  9.991, 'Hamburg-Flughafen',
!                    54.533,  9.550, 'Schleswig',
!  max_time_stamps    = 500
!  zprefix            = 'Meteogram_'
!  var_list           = '  '
!/
&run_nml
  num_lev        = 100
  lvert_nest     = .false.
  dtime          = ${DTIME}     ! timestep in seconds
  lart           = .false.               ! ICON-ART main switch
  ldass_lhn      = .true.          ! latent heat nudging
  ldynamics      = .TRUE.
  ltransport     = .true.
  ntracer        = 5
  iforcing       = 3
  ltestcase      = .false.
! msg_level      = 13
  msg_level      = 10
  ltimer         = .true.
  timers_level   = 10
  check_uuid_gracefully = ${CHECK_UUID_GRACEFULLY}
  output         = "nml" ! "nml"
! debug_check_level = 10
  debug_check_level = 10
/
&nwp_phy_nml
  cldopt_filename              = 'ECHAM6_CldOptProps.nc'
  lrtm_filename                = 'rrtmg_lw.nc'
  inwp_gscp       = 2,2
  mu_rain         = 0.5
  rain_n0_factor  = 0.1
  inwp_convection = 1,0
!  lshallowconv_only = .true.
  lshallowconv_only = .false.  ! fuer icon ab November 2020
  lgrayzone_deepconv= .true.  ! fuer icon ab November 2020
  inwp_radiation  = 4
  inwp_cldcover   = 1
  inwp_turb       = 1
  inwp_satad      = 1
  inwp_sso        = 1
  inwp_gwd        = 0,0
  inwp_surface    = 1
  latm_above_top  = .true.
  ldetrain_conv_prec = .true.
  efdt_min_raylfric = 7200.
  itype_z0         = 2
  icapdcycl        = 3
  icpl_aero_conv   = 1
  icpl_aero_gscp   = 1
  icpl_o3_tp       = 1
  dt_rad    = ${DT_RAD}
  dt_conv   = ${DT_CONV}
  dt_sso    = ${DT_SSO}
  dt_gwd    = ${DT_GWD}
/
&nwp_tuning_nml
 itune_albedo      = 1
 tune_box_liq_asy  = 4.0 ! :cp-cv-bugfix, old value: 3.5    ! 3.25
 tune_gfrcrit      = 0.333
 tune_gkdrag       = 0.0
 tune_gkwake       = 0.25
 tune_gust_factor  = 7.0 ! :cp-cv-bugfix, old value: 7.25
 tune_minsnowfrac  = 0.3
 tune_sgsclifac    = 1.0
! cp-cv-bugfix:
 tune_rcucov       = 0.075
 tune_rhebc_land   = 0.825
 tune_zvz0i        = 0.85
 icpl_turb_clc     = 2
 max_calibfac_clcl = 2.0
 tune_box_liq      = 0.04
 itune_gust_diag   = 3
 tune_urbahf       = 15.,0,0,15.
 tune_urbisa       = 1.0,1.0
/
&turbdiff_nml
 a_hshr        = 2.0
 frcsmot       = 0.2   ! these 2 switches together apply vertical smoothing of the TKE source terms
 icldm_turb    = 2
 imode_frcsmot = 2     ! in the tropics (only), which reduces the moist bias in the tropical lower troposphere
 imode_tkesso  = 2
! use horizontal shear production terms with 1/SQRT(Ri) scaling to prevent unwanted side effects:
 itype_sher    = 2
 ltkeshs       = .true.
 ltkesso       = .true.
 pat_len       = 750.
 q_crit        = 2.0
 rat_sea       = 0.8 ! :cp-cv-bugfix, old value: 7.0
 tkhmin        = 0.05
 tkmmin        = 0.1
 tur_len       = 300.
! cp-cv-bugfix:
 rlam_heat     = 10.0
 alpha1        = 0.125
/
&lnd_nml
  sstice_mode    = 6   ! 4: SST and sea ice fraction are updated daily,
                       !    based on actual monthly means
  ci_td_filename = '${INPDIR}/${YYYY_MM}/LOWBC_${YYYY}_${MM}.nc'
  sst_td_filename= '${INPDIR}/${YYYY_MM}/LOWBC_${YYYY}_${MM}.nc'
  ntiles         = 3
  nlev_snow      = 3
  zml_soil       = ${ZML_SOIL}
  lmulti_snow    = .false.
  itype_heatcond = 3
  idiag_snowfrac = 20
  itype_snowevap = 3
  lsnowtile      = .true.
  lseaice        = .true.
  llake          = .true.
  itype_lndtbl   = 4
  itype_evsl     = 4
  itype_root     = 2
  itype_trvg     = 3
  cwimax_ml      = 5.e-4
  c_soil         = 0.1
  c_soil_urb     = 0.5
  lprog_albsi    = .true.
  lterra_urb     = .true.
  itype_kbmo     = 2
  lurbalb        = .true.
  itype_ahf      = 1
  itype_eisa     = 3
/
&radiation_nml
  ecrad_data_path= '${ecrad_data_path}'
  ghg_filename =  '${GHG_FILENAME}'
  irad_co2    = 4           ! 4: from greenhouse gas scenario
  irad_ch4    = 4           ! 4: from greenhouse gas scenario
  irad_n2o    = 4           ! 4: from greenhouse gas scenario
  irad_cfc11  = 4           ! 4: from greenhouse gas scenario
  irad_cfc12  = 4           ! 4: from greenhouse gas scenario
  irad_o3     = 79
  irad_aero   = 6
  albedo_type = 2          ! Modis albedo
  direct_albedo = 4
  albedo_whitecap = 1
  direct_albedo_water = 3
/
&nonhydrostatic_nml
 iadv_rhotheta   = 2
  ivctype         = 2
  itime_scheme    = 4
  exner_expol     = 0.6
  vwind_offctr    = 0.2
  damp_height     = 18000.
  rayleigh_coeff  = 5.0
  divdamp_fac     = 0.004   ! 0.004 for R2B6; recommendation for R3B7: 0.003
  divdamp_order   = 24 ! 2 ass, 24 fc - may become unnecessary with incremental analysis update
  divdamp_type    = 32  ! optional: 2 assimilation cycle, 3 forecast
  divdamp_trans_start = 12500.
  divdamp_trans_end   = 17500.
!  l_open_ubc      = .false.
  igradp_method   = 3
  l_zdiffu_t      = .true.
  thslp_zdiffu    = 0.02
  thhgtd_zdiffu   = 125.
  htop_moist_proc = 22500.
  hbot_qvsubstep  = 22500 ! r3b7: 19000.  else: 22500.
  ndyn_substeps   = ${NDYN_SUBSTEPS}
/
&sleve_nml
 min_lay_thckn   = 20.
  itype_laydistr  = 1
  top_height      = 30000.
  stretch_fac     = 0.65
  decay_scale_1   = 4000.
  decay_scale_2   = 2500.
  decay_exp       = 1.2
  flat_height     = 16000.
/
&dynamics_nml
  iequations     = 3
!  idiv_method    = 1
  divavg_cntrwgt = 0.50
  lcoriolis      = .true.
/
&transport_nml
  ivadv_tracer   = 3,3,3,3,3
  itype_hlimit   = 3,4,4,4,4,
  ihadv_tracer   = 52,2,2,2,2,
  llsq_svd       = .false.
  beta_fct       = 1.005
/
&diffusion_nml
 hdiff_order                 = 5
 itype_vn_diffu              = 1
 itype_t_diffu               = 2
 hdiff_efdt_ratio            = 24 ! 24.0  for R2B6; recommendation for R3B7: 30.0
 hdiff_smag_fac              = 0.025   ! 0.025 for R2B6; recommendation for R3B7: 0.02
 lhdiff_vn                   = .true.
 lhdiff_temp                 = .true.
/
&interpol_nml
  nudge_zone_width  = 10
  nudge_max_coeff   = 0.075
  lsq_high_ord      = 3
  l_intp_c2l        = .true.
  l_mono_c2l        = .true.
  support_baryctr_intp = .true.
/
&gridref_nml
  grf_intmethod_e  = 6
  grf_intmethod_ct = 2
  grf_tracfbk      = 2
  denom_diffu_v    = 150.
/
&extpar_nml
  itopo                = 1
  n_iter_smooth_topo   = 1,
  hgtdiff_max_smooth_topo = 750.
  heightdiff_threshold = 2250.
  itype_vegetation_cycle = 2
  itype_lwemiss       = 2
  extpar_filename = "${EXTPAR_FILENAME}"
  extpar_varnames_map_file    = .true.
/
&nudging_nml
  nudge_type = 1
  max_nudge_coeff_thermdyn = 0.075
  max_nudge_coeff_vn = 0.04
  nudge_start_height=10500
/
```

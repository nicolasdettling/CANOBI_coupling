This is a description of how to build a new ice shelf draft for CANOBI from a bisicles plot file

- NEMO Version: 4.2.2
- NEMO config: eAnt025.AntArc
- Platform: Archer2
- Nicolas Dettling
  
---

#### Prologue

First, a few steps are required to generate the files necessary to run the actual coupling scripts.

These files are:

- `cf_gridfile.nc` -> contains NEMO grid information
- `cf_gridfile_BISICLES.nc` -> contains BISICLES grid information
- `cf_routfile.nc` -> contains information about how to route calving flux to the nearest NEMO open ocean cell
- `bikegridmap_file.map2d` -> contains information about which BISICLES cells map to which NEMO cells
  
These files need to be created only once. 
Subsequently, a part of the actual coupling code can be run.
The following outlines how this works. 

The coupling code, including the changes I made for this configuration, is here: `https://github.com/nicolasdettling/unicicles_coupling_refactor/tree/semi_coupling#` (on the branch `semi_coupling`).

Generated output files can be found here: `/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/ice_coupling/`

--- 

#### Make the NEMO grid file

The NEMO grid file `cf_gridfile.nc` can be produced by running `utility/python3/make_NEMO_cfgridfile.py`.
This requires a `mesh_mask.nc` file (a single one, not one per processor) to be in the same directory.
Only the coordinates are used from the mesh mask, so the exact ice shelf draft and bathymetry does not matter here.
To run the coupling scripts, an environment must be loaded: 

````
source /work/n02/n02/marc/python/software/unicicles-env_230620/bin/activate
````

A slurm submission script (that sources the necessary environment), and a `mesh_mask.nc` for CANOBI are here:

````
/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/ice_coupling/nemo_cfgridfile.sh
/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/ice_coupling/mesh_mask.nc
````

Notes:
- Lines 22 & 27 in `make_NEMO_cfgridfile.py`: Some additions to the dimension setting were necessary for me.

---

#### Make the BISICLES grid file

The BISICLES grid file `cf_gridfile_BISICLES.nc` can be produced by running `utility/python3/make_BISICLES_cfgridfile.py`.
This requires a bisicles plot file to be in the same directory.
Also, the level (~ BISICLES resolution) has to be specified (here I set level 2).

A slurm submission script (that sources the necessary environment) and a BISICLES plot file are here :

````
/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/ice_coupling/bike_cfgridfile.sh
/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/ice_coupling/bisicles_dc575c_22790101_plot-AIS.hdf5
````

Notes for `make_BISICLES_cfgridfile.py`:
- The middle section is a bit all over the place with highlights such as ("Steph says", "Ferret says", hard-coded "steph_offset")
- Line 41: Set interpolation order, I kept 0
- Line 79: Choose projection (epsg:3031)
- Line 82,83: Choose x0, y0 (-3072000)
- Line 89 & 109: I kept the original mfact=1 and steph_offset=0
- This produces a correct set of coordinates (tested with bisicles output)

---

#### Make the BISICLES to NEMO mapping file

The mapping file can be produced by running `utility/python3/make_BIKEtoNEMO_mapping.py`.
This requires a bisicles plot file to be in the same directory.
Also, the level (~ BISICLES resolution) has to be specified (here I set level 2).

A slurm submission script (that sources the necessary environment) and a BISICLES plot file are here :

````
/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/ice_coupling/mapping_py.sh
/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/ice_coupling/bisicles_dc575c_22790101_plot-AIS.hdf5
````
This takes a long time (~8h on one serial node on Archer2). I have not tried to make it faster since it only needs to run once.

Notes for `make_BISICLES_cfgridfile.py`:
- Line 18: Set max tolerance to eORCA025 value
- Line 99: Set jlim_u to 453 (y coordinate length)
- Line 104: Set ilim_u to 1440 (x coordinate length)
- Line 130: Reduced max find to 500, seems more than enough from the output
- Line 165, 166: Reversed the order of reading BISICLES coordinates following Robin's advice. This means we can keep the transpose lines in coupling scripts (mentioned below), and this is probably the reason why mfact=1 works in `make_BISICLES_cfgridfile.py`
  
---

#### Make the NEMO iceberg route file

The iceberg route file can be produced by running `utility/python3/create_iceberg_routfile.py`.
This requires a mesh mask file to be in the same directory.

A slurm submission script (that sources the necessary environment) and a mesh mask file are here :

````
/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/ice_coupling/iceberg_routefile.sh
/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/ice_coupling/mesh_mask.nc
````
No changes to the scripts were necessary.

---

#### Interlude

This concludes the steps that need to be taken with utility scripts. 
The following describes how to run three of the actual coupling scripts to generate an initial calving file, ice shelf draft, and bathymetry to put into NEMO.
Those coupling scripts are in `/active/python3`.

---

#### Get the ice shelf draft on the ocean grid (bisicles_global_to_nemo)

In this coupling step, the bisicles ice shelf draft is transferred to the NEMO grid.

This is done in the coupling script `bisicles_global_to_nemo.py`. 
The script expects the bisicles plot file, the bikegridmap_file.map2d file, and a template bathymetry file.

If you have not followed the previous steps, they are here:

````
/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/ice_coupling/bisicles_dc575c_22790101_plot-AIS.hdf5
/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/ice_coupling/bikegridmap_file.map2d
/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/ice_coupling/bathy_template_lakefilled.nc

````

The bathymetry template contains closed lakes and two smaller ice shelves, which otherwise make the model unstable after some time (talk to Birgit for more info).

A few things have to be taken into consideration:

- Line 321: Setting the level and order (here 2 and 1)
- Line 336: Bathymetry and ice shelf draft are transposed. This needs to be consistent with the order in which the bisicles plot file is read in ... If the result looks weird, this is a promising candidate
- Line 402: If not otherwise specified, the bathymetry of the template file will be used, not the one from bisicles

Especially important here (From line 443):

1. If there was open ocean before and now there is an ice shelf after the new BISICLES cycle, the ice shelf depth will be reduced to zero to open it again.
2. If the newly opened ocean is too shallow, a statement will indicate that this is happening, but nothing happens.
3. If there was an ice shelf before, but now there is not, grounded ice will be generated if the ocean is too shallow for a cavity, and an ice shelf will be generated if the ocean is deep enough for a cavity.

---

#### Run domain_cfg

The new ice draft and bathymetry now need to be run through domain_cfg.

A script to submit a domain_cfg job to slurm is here:

````
/work/n02/n02/nicdet/coupling_canobi/unicicles_coupling_refactor/active/python3/domain.sh
````

For this to work, the following files need to be in the folder where domain_cfg is run:

````
/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/domain_cfg/make_domain_cfg.exe
/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/domain_cfg/namelist_cfg
/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/domain_cfg/namelist_ref
/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/domain_cfg/coordinates_AIS.nc
````

The output should be ncfiles `domain_cfg.nc` and `mesh_mask.nc`. Logs are in `ocean.output` and parameters used are written to `output.namelist.dyn`.

--- 

#### Initial Conditions

When the ice shelf draft changes, new open ocean cells can form.
The initial conditions contain fill values (9999) in these newly created cells.
Therefore, I replaced all fill values (9999) in the initial temperature and salinity fields with -1.9 °C and 34.4, respectively.
Similarly, sea ice area, height, and snow height were set to zero instead of 9999.

The new initial conditions are:

````
/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/ini_cond/SOSE-ConsTemp-initial-conditions-20240507_filled.nc
/work/n02/shared/nicdet/NEMO_share/eANT025.AntArc/ini_cond/SOSE-AbsSal-initial-conditions-20240507_filled.nc
````

Sea ice initial conditions remain unchanged.

The other initial conditions are:

- geothermal_heating.nc -> global field without nans interpolated to the configuration grid by nemo
- runoff.nc -> zero everywhere
- bergmelt.nc -> missing value is zero
- chlorophyll.nc -> global field without nans interpolated to the configuration grid by nemo
- empc.nc -> missing value is nan

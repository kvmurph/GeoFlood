# GeoFlood
Flood mapping program based on high-resolution terrain analyses. 

When using GeoFlood, please cite the following paper:

Zheng, X., D. Maidment, D. Tarboton, Y. Liu, P. Passalacqua (2018), GeoFlood: Large scale flood inundation mapping based on high resolution terrain analysis, Water Resources Research, 54, 12, 10013-10033, doi:10.1029/2018WR023457. 

When using GeoNet, please cite the following papers:

Passalacqua, P., T. Do Trung, E. Foufoula-Georgiou, G. Sapiro, W. E. Dietrich (2010), A geometric framework for channel network extraction from lidar: Nonlinear diffusion and geodesic paths, Journal of Geophysical Research Earth Surface, 115, F01002, doi:10.1029/2009JF001254.

Sangireddy, H., R. A. Carothers, C.P. Stark, P. Passalacqua (2016), Controls of climate, topography, vegetation, and lithology on drainage density extracted from high resolution topography data, Journal of Hydrology, 537, 271-282, doi:10.1016/j.jhydrol.2016.02.051.

# File Structure upon downloading
GeoNet3
  - GeoNet
  - GeoFlood
  - TauDEM
  - *COMID_Roughness.csv*
  - *stage.txt*
  - *GeoF.yml (Conda environment setup)*
  - *Requirements.txt*
  
# Environment

Create an Anaconda environment from the *geofloodenv.yml* file

```
conda create env --file geofloodenv.yml
```

You can use ``` conda env list ``` to verify that the new environment was installed correctly.
  
# Configuration 
Navigate to the *GeoNet* directory

GeoNet and GeoFlood use configuration files to specify project attributes. To have automatic generation of a project specific and pointer configuration file, run:

```
python pygeonet_configure.py -dir [geofloodhomedir] -p [project_name] -n [dem_name] --no_chunk --input_dir [Name of input directory] --output_dir [Name of output directory] --no_hr
```

All of the arguments to this configuration script are optional. Arguments:
- -dir: The file path to the directory that will hold GeoNet/GeoFlood input and output directories. Default path (not specified) is the path to the "GeoNet3" directory.
- -p: The project name that will be used in the input and output directories to keep different projects separate. Default is "my_project".
- -n: Name of the input DEM, without extension, as well as the prefix for all project outputs. Default is "dem".
- --no_chunk: If passed as an argument, DEM's larger than 1.5 GB will **NOT** be chunked/batch processed during the *Network_Extraction.py* script. DEM's of this size can cause memory errors when processed all at once on a local machine. The default is to chunk DEMs larger than the 1.5 GB threshold.
- --input_dir: Name of Inputs folder to be held in the "-dir" directory. Default is "GeoInputs".
- --output_dir: Name of Outputs folder to be held in the "-dir" directory/ Default is "GeoOutputs".
- --channel_type: 1: Channel Extraction with NHD HR Raster; 0: Channel Extraction with NHD MR Raster; -1: Channel Extraction with only DEM Features

The "pointer" configuration file will be placed in the *<.../GeoNet3/GeoNet/>* directory. The project specific configuration file will be placed in the default or user specified "geofloodhomedir" directory.

# Prepare File Structure

```
python pygeonet_prepare.py 
```

Resulting File Structure (assuming the deafults were used):

GeoNet3
  - GeoNet
    - *project_pointer.cfg*
    - GeoNet scripts ...
  - GeoFlood
    - GeoFlood scripts ...
  - GeoInputs
    - GIS
      - my_project
        - **dem.tif** (User entry)
    - Hydraulics
      - my_project
        - COMID_Roughness.csv
        - stage.txt
    - NWM
      - my_project
  - GeoOutputs
    - GIS
      - my_project
    - Hydraulics
      - my_project
    - Inundation
      - my_project
    - NWM
      - my_project
  - *GeoFlood_[my_project].cfg*        **(Project specific cfg. Will be placed wherever "-dir" was specified as)**
  - TauDEM
  
Place **dem.tif** into the:

*...\GeoInputs\GIS\my_project* 

directory that was just created.

Note: If you need to switch back and forth between projects, change the "project_cfg_pointer" variable within the "project_pointer.cfg" to point to proper configuration file.

For example, if I'm  working on project "example" and need to go back to project "test", change the                                         'project_cfg_pointer' variable within the pointer cfg file:
    
    project_cfg_pointer = <path\to\project_example_home\GeoFlood_example.cfg> ---> project_cfg_pointer = <path\to\project_test_home\GeoFlood_test.cfg>

# GeoNet Workflow
### 1. DEM Smoothing
```
python pygeonet_nonlinear_filter.py
```

Inputs:

         <...\GeoInputs\GIS\my_project\dem.tif>

Outputs: 

         <...\GeoOutputs\GIS\my_project\PM_filtered_grassgis.tif>
         
File Description:

`_PM_filtered_grassgis.tif` is the Perona-Malik non-linear filtered DEM.

Recommendations:

Smoothing with default 50 iterations for high-resolution terrain. Edit the `nFilterIterations` variable in the **pygeonet_defaults.py** script if a different number of iterations is preferred. 

### 2. Slope and Curvature
```
python pygeonet_slope_curvature.py
```

Inputs:

         <...\GeoOutputs\GIS\my_project\PM_filtered_grassgis.tif>

Outputs: 

         <...\GeoOutputs\GIS\my_project\dem_slope.tif>
         <...\GeoOutputs\GIS\my_project\dem_curvature.tif>
         
File Description:

`_slope.tif` is the slope raster computed from the `_PM_filtered_grassgis.tif`.
`_curavture.tif` is the curvature raster computed from the `_PM_filtered_grassgis.tif`.

Rcommendations:

`_slope.tif` not currently used in GeoFlood workflow. A geometric curvature calculation is the default, but can be changed to a laplacian curvature by changing the 'curvatureCalcMethod' in the **pygeonet_defaults.py** script from *geometric* to *laplacian*.

### 3. GRASS GIS
```
python pygeonet_grass_py3.py
```

Inputs:

         <...\GeoOutputs\GIS\my_project\PM_filtered_grassgis.tif>

Outputs: 

         <...\GeoOutputs\GIS\my_project\dem_fac.tif> (Flow Accumulation raster based on Multiple Flow Direction raster)
         <...\GeoOutputs\GIS\my_project\dem_fdr.tif> (Flow direction raster using Multiple Flow Direction computation method)
         <...\GeoOutputs\GIS\my_project\dem_basins.tif> (GRASS GIS desineated catchments for each reach)
         <...\GeoOutputs\GIS\my_project\dem_outlets.tif> (GRASS GIS defined outlets for the study area)


### 4. Flow Accumulation and Curvature Skeleton
```
python pygeonet_skeleton_definition.py
```

Inputs:

         <...\GeoOutputs\GIS\my_project\PM_filtered_grassgis.tif>
         <...\GeoOutputs\GIS\my_project\dem_curvature.tif>
         <...\GeoOutputs\GIS\my_project\dem_fac.tif>

Outputs:

         <...\GeoOutputs\GIS\my_project\dem_skeleton.tif> (Pixels with a value of 1 satisfy flow accumulation and curvature threshold)
         <...\GeoOutputs\GIS\my_project\dem_flowskeleton.tif> (Pixels with a value of 1 satisfy flow accumulation threshold)
         <...\GeoOutputs\GIS\my_project\dem_curvatureskeleton.tif> (Pixels with a value of 1 satisfy curvature threshold)

The flow accumulation threshold used in this script can be adjusted by changing the value set to the *flowThresholdForSkeleton* variable in the **pygeonet_defaults.py**. Default is 500.
- Decreasing the threshold increases the density of the network, i.e. more pixels classified as likely channels/reaches.
- Increasing the threshold decreases the density of the extracted network.

         
### 5. Geodesic Distance and Fast Marching [Not currently used in GeoFlood]
```
python pygeonet_fast_marching.py
```

Inputs:

         <...\GeoOutputs\GIS\my_project\dem_outlets.tif>
         <...\GeoOutputs\GIS\my_project\dem_basins.tif>
         <...\GeoOutputs\GIS\my_project\dem_curvature.tif>
         <...\GeoOutputs\GIS\my_project\dem_fac.tif>
         <...\GeoOutputs\GIS\my_project\dem_skeleton.tif>

Outputs:

         <...\GeoOutputs\GIS\my_project\dem_geodesicDistance.tif>
         <...\GeoOutputs\GIS\my_project\dem_costfunction.tif>
         
Utilizes the "outlets" and "basins" outputs from step four to identify likely starting points for the fast marching algorithm. With a slightly different cost function from the one used in GeoFlood's network extraction script, a fast marching algorithm is implemented to calculate the geodesic distance, from the previously identified starting points, across the entire raster. Relative to the range of values calculated, the lowest geodesic distances denote a channel and increase as the distance from the channel increases.

### 6. Channel Head Identification [Not currently used in GeoFlood]
```
python pygeonet_channel_head_definition.py
```

Inputs:

         <...\GeoOutputs\GIS\my_project\dem_skeleton.tif>
         <...\GeoOutputs\GIS\my_project\dem_geodesicDistance.tif>

Outputs:

         <...\GeoOutputs\GIS\my_project\dem_channelHeads.tif>
         <...\GeoOutputs\GIS\my_project\dem_channelHeads.shp>

Identifies channel heads using the "skeleton" and "geodesicDistance" rasters by identifying pixels that satisfy the skeleton threshold criteria and have the minimum geodesic distance within a set search box window.
       
### 5. NHD HR Flowline Raster [Optional for GeoFlood]
```
python pygeonet_hr_raster.py <path\to\nhd_hr_shapefile\shape.shp>
```

Inputs:

         <...\path\to\nhd_hr_shapefile\shape.shp>

Outputs: 

         <...\GeoOutputs\GIS\my_project\dem_NHD_HR.tif> 
      
Rasterized version of NHD HR Flowline to be used in network extraction.

The required input to this script is the file path to the NHD_HR_flowline shapefile you wish to use. The path specified should end with the name of the shapefile including the ".shp". All other metadata related files for the shapefile should be located in the same folder.

The shapefile can be larger than the DEM as it will get clipped to the DEMs dimensions.

# GeoFlood Workflow
GeoFlood was designed to work with NHD MR flowlines as those are the flowlines used in the National Water Model. Find NHD MR vector data here:
- https://www.arcgis.com/home/webmap/viewer.html?webmap=9766a82973b34f18b43dafa20c5ef535&extent=-140.4631,21.8744,-48.5295,57.4761
or
- https://viewer.nationalmap.gov/basic/?basemap=b1&category=nhd&title=NHD%20View

From these downloads, you specifically need the shapefiles of NHD MR Flowlines and their associated catchments. To do this, navigate to the NFIE folder you downloaded within any GIS software (QGIS,ArcGIS Pro, ArcMap,...) and upload the flowline and catchments of interest. The shapefiles of interest can then be extracted.

**Place the extracted flowline and catchment shapefiles into:**
*<...\GeoInputs\GIS\my_project\here>* 

Please name the flowline shapefile: "Flowline.shp" (rename all extensions)
Please name the catchment shapefile: "Catchment.shp" (rename all extensions)

### 6. Network Node Reading
```
cd ../GeoFlood
```

```
python Network_Node_Reading.py
```

Inputs:

         <...GeoInputs\GIS\my_project\Flowline.shp>

Outputs: 

         <...GeoOutputs\GIS\my_project\dem_endPoints.csv>

This script will output a csv containing the start and end points of the flowlines of interest.

### 7. Negative Height Above Nearest Drainage
`python Relative_Height_Estimation.py`

Returns a binary raster/array with values of 1 given to pixels at a lower elevation than the elevation associated with NHD MR Flowline pixels. A value of zero is given to all other pixels in the image, i.e. pixels at a higher elevation than the NHD MR Flowlines

Inputs:

         <...GeoInputs\GIS\my_project\Flowline.shp>

Outputs:

         <...\GeoOutputs\GIS\my_project\dem_NegaHand.tif> (binary raster described above)
         <...\GeoOutputs\GIS\my_project\dem_Allocation.tif> (elevation comparison raster)
         <...\GeoOutputs\GIS\my_project\dem_nhdflowline.tif> (raster of burned in/etched "Flowline.shp")

### 8. Network Extraction
```
python Network_Extraction.py
```

Inputs:

         <...GeoOutputs\GIS\my_project\dem_endPoints.csv>
         <...\GeoOutputs\GIS\my_project\dem_curvature.tif>
         <...\GeoOutputs\GIS\my_project\dem_fac.tif>
         <...\GeoOutputs\GIS\my_project\dem_skeleton.tif>
         <...\GeoOutputs\GIS\my_project\dem_NegaHand.tif> (Optional)

Outputs: 
         
         <...\GeoOutputs\GIS\my_project\dem_channelNetwork.shp> (Shape file of extracted channel network)
         <...\GeoOutputs\GIS\my_project\dem_path.tif> (Raster of extracted network)
         <...\GeoOutputs\GIS\my_project\dem_cost.tif> (Raster of the "cost" associated with each pixel)

The goal for this script is to find the minimum cost path between the start and end points. Since the National Water Model (NWM) uses NHD Medium Resolution Flowlines, we used their endpoints in this script. In the future, I see this switching to NHD High Resolution Flowlines as the NWM transitions away from NHD MR. 

**Note: This is where the input rasters to the cost function will be chunked if:
    - The DEM is large enough (>1.5 GB)
    - A value of 1 is given to the 'Chunk_DEM' variable in the project specific cfg file**

# TauDEM Functions
Please refer to the TauDEM user guide for more specific instructions and descriptions about their entire suite of products. 

TauDEM User Guide: https://hydrology.usu.edu/taudem/taudem5/TauDEM53GettingStartedGuide.pdfage

For the GeoNet/GeoFlood workflow the following functions are used: 
- PitRemove
- D-Infinity flow directions
- D-Infitinity flow accumulation [OPTIONAL]
- HAND (Height Above Nearest Drain)
- Hydraulic property base table
- Inunmap

The following commands were all ran from the command line/anaconda prompt.

### 9. Pit Filling
`mpiexec -n [integer value representing the number of processes to use] <...\GeoNet3\TauDEM\pitremove> -z <...\GeoInputs\GIS\my_project\dem.tif> -fel <...\GeoOutputs\GIS\my_project\dem_fel.tif>`

Outputs: 
         
         <...\GeoOutputs\GIS\my_project\dem_fel.tif>

Pit removed/filled DEM.

### 10. D-Infinity Flow Direction:
`mpiexec -n [integer value representing the number of processes to use] <...\GeoNet3\TauDEM\dinfflowdir> - fel <...\GeoOutputs\GIS\my_project\dem_fel.tif> -ang <...\GeoOutputs\GIS\my_project\dem_ang.tif> -slp <...\GeoOutputs\GIS\my_project\dem_slp.tif>`

Outputs: 
         
         <...\GeoOutputs\GIS\my_project\dem_ang.tif>
         <...\GeoOutputs\GIS\my_project\dem_slp.tif>

Flow direction raster using D-infinity flow direction method proposed in Tarboton, D. G., (1997) and a slope raster resulting from D-infinity flow directions.

### 11. D-Infinity Flow Accumulation [OPTIONAL]
`mpiexec -n [integer value representing the number of processes to use] <...\GeoNet3\TauDEM\TauDEM\areadinf> - ang <...\GeoOutputs\GIS\my_project\dem_ang.tif> -sca <...\GeoOutputs\GIS\my_project\dem_sca.tif>`

Outputs: 

         <...\GeoOutputs\GIS\my_project\dem_sca.tif>

Flow accumulation based on D-inf flow directions.

### 12. Height Above Nearest Drainage
`mpiexec -n [integer value representing the number of processes to use] <...\GeoNet3\TauDEM\dinfdistdown> - ang <...\GeoOutputs\GIS\my_project\dem_ang.tif> -fel <...\GeoOutputs\GIS\my_project\dem_fel.tif> -slp <...\GeoOutputs\GIS\my_project\dem_slp.tif> -src <...\GeoOutputs\GIS\my_project\dem_path.tif> -dd <...\GeoOutputs\GIS\my_project\dem_hand.tif> -m ave v`

Outputs: 

         <...\GeoOutputs\GIS\my_project\dem_hand.tif>

Height Above Nearest Drainage (HAND) raster.

# Back to GeoFlood

### 13. Segmenting Extracted Network
```
python Streamline_Segmentation.py
```

Inputs:

         <...\GeoOutputs\GIS\my_project\dem_channelNetwork.shp>

Outputs: 

         <...\GeoOutputs\GIS\my_project\dem_channelSegment.shp>

Shape file of segmented channel network from "Network_Extraction.py". Default streamline segmentation is 1000 meters.

### 14. GRASS GIS Catchment Delineation
```
python Grass_Delineation_py3.py
```

Inputs:

         <...\GeoOutputs\GIS\my_project\dem_fdr.tif>
         <...\GeoOutputs\GIS\my_project\dem_channelSegment.shp>

Outputs: 

         <...\GeoOutputs\GIS\my_project\dem_segmentCatchment.tif>

Raster of catchments/drainage areas associated with each channel segment.

### 15. River Attribute Estimation
```
python River_Attribute_Estimation.py
```

Inputs:

         <...\GeoOutputs\GIS\my_project\dem_channelSegment.shp>
         <...\GeoOutputs\GIS\my_project\dem_segmentCatchment.tif>

Outputs: 

         <...\GeoOutputs\GIS\my_project\dem_segmentCatchment.shp> 
         <...\GeoOutputs\Hydraulics\my_project\dem_River_Attribute.txt>

A shape file of catchments associated with the segmented flowlines and stream segment attributes: feature ID, slope, length, square area.

### 16. Network Mapping
```
python Network_Mapping.py
```

Inputs:

         <...GeoInputs\GIS\my_project\Catchment.shp>
         <...\GeoOutputs\GIS\my_project\dem_channelSegment.shp>

Outputs: 

         <...\GeoOutputs\GIS\my_project\dem_networkMapping.csv> 

Associates "Catchment.shp" feauture IDs to each reach segment (assigns the appropriate COMID to each segmented reach).

### 17. Hydraulic Property Base Table **(TAUDEM)**
`mpiexec -n [integer value representing the number of proceses to use] <...\GeoNet3\TauDEM\catchhydrogeo> - hand <...\GeoOutputs\GIS\my_project\dem_hand.tif> -catch <...\GeoOutputs\GIS\my_project\dem_segmentCatchment.tif> -catchlist <...\GeoOutputs\Hydraulics\my_project\dem_River_Attribute.txt> -slp <...\GeoOutputs\GIS\my_project\dem_slp.tif> -h <...\GeoInputs\Hydraulics\my_project\stage.txt> -table <...\GeoOutputs\Hydraulics\my_project\hydroprop-basetable.csv>`

Outputs: 

         <...\GeoOutputs\Hydraulics\my_project\hydroprop-basetable.csv>

Calculates hydraulic properties, i.e. surface area, bed area, and volume based on the stage height. Stage height is iterated through with the range of values specified in the "stage.txt" file.

### 18. Hydraulic Property Full Table (More channel Geometries)
```
python Hydraulic_Property_Postprocess.py
```

Inputs:

         <...\GeoInputs\Hydraulics\my_project\COMID_Roughness.csv>
         <...\GeoOutputs\GIS\my_project\dem_networkMapping.csv>
         <...\GeoOutputs\Hydraulics\my_project\hydroprop-basetable.csv>
         
Outputs: 

         <...\GeoOutputs\Hydraulics\my_project\hydroprop-fulltable.csv>

Assigns a Manning's roughness coefficient to each segment based on stream order and calculates: top width (m), wetted perimeter (m), wetted area (m2), and hydraulic radius (m) for the range of stage heights specified in 'stage.txt'.

### 19. National Water Model Forecast
```
python Forecast_Table.py <path\to\nwm_forecast\nwm....conus.nc>
```

Inputs:
         
         <...\path\to\nwm_forecast\nwm....conus.nc>
         <...\GeoOutputs\GIS\my_project\dem_networkMapping.csv>
         <...\GeoOutputs\Hydraulics\my_project\hydroprop-fulltable.csv>

Outputs: 

         <...\GeoOutputs\NWM\my_project\nwm.....conus.csv>
         <...\GeoOutputs\NWM\my_project\nwm.....conus.nc>

The NWM folder within GeoInputs was made with the intent that NWM forecasts would be stored there, but if a different location is preferred, that is perfectly fine. Assigns NWM forecasted discharge to each channel segment based on COMID.

### 20. Inundation Map **(TAUDEM)**
`mpiexec -n [integer value representing the number of proceses to use] <...\GeoNet3\TauDEM\inunmap> -hand <...\GeoOutputs\GIS\my_project\dem_hand.tif> -catch <...\GeoOutputs\GIS\my_project\dem_segmentCatchment.tif> -forecast <...\GeoOutputs\NWM\my_project\nwm......conus.nc> -mapfile <...\GeoOutputs\Inundation\my_project\dem_NWM_inunmap.tif>`

Outputs: 

         <...\GeoOutputs\Inundation\my_project\dem_NWM_inunmap.tif>

Generates an inundation map using the HAND raster for the study area, the delineated catchments associated with each segment, and the NWM forecast for the time frame of interest.

## Python based workflow for processing Terrestrial LiDAR point clouds used in microtopographic change detection

### I. Introduction

#### A. Study Purpose

Among the advances in remote sensing technologies of the past decade has been the rapid improvement in the resolutions, especially spatial and temporal, that readily available systems are capable of capturing. The emergence of Terrestrial LiDAR Scanning (TLS) and low-cost Unmanned Airborne Systems (UAS) or drone technology has enabled researchers to detect vastly smaller objects at a rapidly decreased repeat interval at a relatively reasonable cost.

One emerging application of this technology is measuring the change in surface elevation rather than the common technique of directly measuring eroded sediment leaving a plot or field. Traditional methods lack the precision to efficiently make these measurements over a broad area. In the original project plan document, Dr. Daniel Yoder put this in context: “As a frame of reference, a loss of a 1-mm (roughly the tip of a sharpened pencil lead) layer of soil in a year is the equivalent of a soil loss rate of 6.2 Tons per Acre per year, which is considered to be more than almost any soil can withstand in order to maintain productivity.”

[![Point Cloud Animation](https://img.youtube.com/vi/KtM2LF6K0B0/0.jpg)](https://www.youtube.com/watch?v=KtM2LF6K0B0)
#### B. Project Site

In 2019, we established three experimental plots on a hillslope at the UT Plant Science unit of the East Tennessee AgResearch and Education Center (ETREC). Each of the parallel plots is a self-contained basin measuring 22 m long by 6 m wide with the long side oriented approximately perpendicular to the hillslope. At the bottom of the plot is a system for collecting a known percentage of the water and sediment that drains to the plot bottom. Vegetation on the plots were removed by chemical applications and repeated burning of the surface.

At the corners of the study area, concrete monuments were installed into the ground to be used as reference benchmarks.

![Site Photo](https://i.imgur.com/rlekN3k.png)

![Aerial View](https://i.imgur.com/ruZePG4.png)

#### C. Data Collection
For each sampling date the plots are each scanned from the top, bottom and the approximate midpoint of the long sides for 10 scans total. The two scans done from the spaces between plots are used in the clouds for both plots. The scanner is configured to ½ of their maximum capable resolution and the 2X quality setting. After completing the LiDAR measurements, the unit colorized the points by going through secondary panoramic photography phase. With these parameters the scans can each produce as many as 177 million points (though the actual points are far less because there are no overhead surfaces to measure) and takes about 13 minutes to complete both phases.

![Scan Positions](https://i.imgur.com/ub29Rnu.png)

### II. Data Processing

#### A. Processing Overview

The preparation and processing of the scans into usable elevation rasters turned out to be an unexpectedly complicated and time intensive task that has undergone several iterations as we continue to improve the methods. These processes have changed so much that it has been difficult to directly compare results from two different attempts. The method developed here and outlined in Figure 5 uses a python script to automate all possible processes after the initial registration. In addition to making repetitive tasks faster and simpler, this allows for the easier isolation of parameters for evaluation and includes tests meant to help identify errors during the processing.

![Process Diagram](https://i.imgur.com/xas14GY.png)

#### B. TLS Registration

For the scans to achieve sufficient spatial coverage the plots must be scanned multiple times from different angles. Our design has each plot being scanned from 4 sides for a total of 10 scans as shown in Figure 4. Registration refers to the process of integrating the separate clouds into a unified space so they can be merged into a single cloud or overlayed for comparison. The registration process was found to be a critical and complex process that many research papers do not describe in any detail. In addition, because these instruments are marketed mainly to construction, industrial and accident-investigation applications, the documented workflows rely heavily on the presence of unmoving geometric objects in the scan to use in the registration process (Telling et al. 2017).

Registration is typically done either by a target based process in which clouds are aligned with fixed control points or in a cloud-to-cloud algorithm which matches the geometric similarities within the point clouds themselves. Because our study site has few fixed geometric features and we need to register scans taken at different times over which the landscape will be changing, the cloud-to-cloud process would not deliver suitable registration results. Our initial registration plan called for using four uniform sphere targets mounted on concrete monuments installed at the corners of the study area. After the first sampling year, it was determined that the four-target setup may limit the possible measurement accuracy due to a lower than expected density of points hitting the targets from the farthest setups. When there are too few points on the sphere cloud, the software has trouble accurately fitting the shape in both automatic and manual target identification processes, resulting in increased potential for registration error. For the 2020 sampling season an additional four concrete monuments were installed allowing targets to be placed at all four corners of each plot rather than the whole study area.

The exact process for performing a multi-step registration combining scans within and across sampling dates is especially not well documented and has required a great deal of experimentation. The plan for much of the study is outlined in Figure 6. While this process returned RMSE figures that appeared to be within a suitable range, the resulting elevation surfaces often contained obvious errors.

![Figure 6 Flowchart](https://i.imgur.com/HUZmZLz.png)

After several attempts at correcting these errors at the DEM or post-registration stage also provided unsatisfactory results, it became apparent that the registration process needed to be revised. A flowchart of the resulting process is shown in Figure 7.

![Figure 7 Flowchart](https://i.imgur.com/tgKN2jv.png)

#### C. Plot Boundary Delineation

Because the plots are created to be self-contained drainage basins, a hydrologic analysis of the surface should produce a higher quality plot boundary than manual feature editing from the derived elevation surfaces or from the orthophoto created by the UAS photogrammetry. This process delivers generally satisfactory results from a set of two point clouds with a few small regions that extend across the berms included in two of the plots. This is likely an artifact of preparing the DEM surfaces derived from unfiltered clouds with the FILL operation. Preliminary tests suggest that once a third sample date cloud is included these minor error regions largely disappear.

![Figure 9 Flowchart of plot delineation](https://i.imgur.com/nBex0gI.png)

```
for p in plots:
    ufeat = [fr"{wksp}\scratch\scratch.gdb\basp_{p}_{dates[0]}", fr"{wksp}\scratch\scratch.gdb\basp_{p}_{dates[1]"]
    basp_union = fr"{wksp}\scratch\scratch.gdb\{p}_union"
    arcpy.analysis.Union(in_features=ufeat, 
        out_feature_class=basp_union, 
        join_attributes="ALL", 
        cluster_tolerance="", 
        gaps="GAPS")

    pbound = fr"{wksp}\bounds\{p}_bounds.shp" 
    arcpy.analysis.Select(in_features=basp_union, 
        out_feature_class=pbound, 
        where_clause="Shape_Area > 100")
```

![Figure 10 results](https://i.imgur.com/WJSTkcK.png)

#### D. TLS Noise Filtering

Noise is by-product of the LiDAR instrument and should be expected. Most software packages designed to work with aerial LiDAR include tools to filter and classify noise points so they can be withheld from derived surfaces. These tools generally look at a 2D or 3D neighborhood and flag points that fall outside of a defined distance or height threshold. The default parameters of these tools are set for aerial LiDAR data and tests on our terrestrial point clouds suggested generally poor performance without the parameters carefully defined.

Results of the Whitebox Tools Remove LiDAR Outliers filter were compared to a set of created rasters showing the range of elevations of the unfiltered lidar data. Setting the filter to use both the search radius and height threshold values at 3mm returned improved results in the elevation range rasters and this is the value included as the default in the python script. These values will be tester further as the parameters in the script are further adjusted.

#### E. Smoothing

Another method to further reduce the influence of noise and surface roughness is applying a smoothing filter to the raster surface after it has been converted from a point cloud. The Feature Preserving Smoothing tool developed by Lindsay, Francioni, and Cockburn (2019) was included in the python script due to its ability to remove roughness while keeping the sharp breaks in slope that would be present in an eroded terrain. While the smoothed DEM are produced, the final DEM difference in this paper use unsmoothed DEM as more testing needs to be done to determine if smoothing is needed and to properly set the tool parameters.

```
for n in fname:

    wbt.feature_preserving_smoothing(
        dem=fr"{wksp}\DEM_final\dem_{n}_nosmooth.tif", 
        output=fr"{wksp}\DEM_final\dem_{n}_smooth.tif", 
        filter=11, 
        norm_diff=15.0, 
        num_iter=3, 
        max_diff=0.5, 
        zfactor=None, 
    )
```

### III. Results & Assessment

#### A. Means Difference Test

A problem encountered with early results involved a small but consistent error that would create a systematic elevation offset between the two compared DEM. As an attempt to flag for this error a paired-sample T-test was incorporated into the python script. The statistic is configured to take 500 random sample cells from each of the surfaces to be compared and tests for a significant difference in the sample means.

Because the compared DEM should be largely similar outside of areas of concentrated erosion and deposition, this test for statistical difference may be useful to identify issues. Running the test on DEM from the previous and revised methods returned promising results for applicability of this tool. The DEM created under the old method returned a p-vale of >0.05 while the newer surface pairs p-value was <0.05.

![Figure 12 T-Test previous plot 1](https://i.imgur.com/fJEU3oh.png)

![Figure 13 T-test updated plot 1](https://i.imgur.com/Bh1Xuxj.png)

#### B. DEM Difference

The DEM Difference surfaces produced from the revised registration method and processed using the python script do appear to be more consistent with the expected areas of erosion from our visual observations than previously produced surfaces. The previous results in Figure 14 shows erosion features but largely useless for quantification of sediment loss since the systematic offsets would indicate a net elevation gain for 2 of the plots. The new set of results in Figure 15 do appear to indicate net erosion over much of the surfaces while maintaining the larger erosion features.

![Figure 14](https://i.imgur.com/sPAGsL9.png)

![Figure 15](https://i.imgur.com/qOfI9HT.png)

### IV. Conculusions

The revised registration and python script do appear to assist in achieving better results in the measurement of erosion. Specifically the work here provides three main enhancements to the larger study:

1. Creation of a standardized and easily repeatable workflow for TLS data processing.

2. Automation of repetitive steps allows us to quickly get from raw scan data to a DEM difference result layer so that further adjustments to the registration and quality control steps can be further tested and evaluated.

3. Development of a series of tests and checkpoints within the process to identify and mitigate sources of error.

### V. References

Dabek, Paweł, Romuald Zmuda, Bartłomiej Ćmielewski, and Jakub Szczepański. 2014. “Analysis of Water Erosion Processes Using Terrestrial Laser Scanning.” Acta Geodynamica et Geomaterialia 11 (1). https://doi.org/10.13168/AGG.2013.0054.

Eltner, Anette, Hans Gerd Maas, and Dominik Faust. 2018. “Soil Micro-Topography Change Detection at Hillslopes in Fragile Mediterranean Landscapes.” Geoderma 313. https://doi.org/10.1016/j.geoderma.2017.10.034.

Lindsay, John B., Anthony Francioni, and Jaclyn M.H. Cockburn. 2019. “LiDAR DEM Smoothing and the Preservation of Drainage Features.” Remote Sensing 11 (16). https://doi.org/10.3390/rs11161926.

Ouédraogo, Mohamar Moussa, Aurore Degré, Charles Debouche, and Jonathan Lisein. 2014. “The Evaluation of Unmanned Aerial System-Based Photogrammetry and Terrestrial Laser Scanning to Generate DEMs of Agricultural Watersheds.” Geomorphology 214. https://doi.org/10.1016/j.geomorph.2014.02.016.

Telling, Jennifer, Andrew Lyda, Preston Hartzell, and Craig Glennie. 2017. “Review of Earth Science Research Using Terrestrial Laser Scanning.” Earth-Science Reviews. https://doi.org/10.1016/j.earscirev.2017.04.007.

Vinci, A., R. Brigante, F. Todisco, F. Mannocchi, and F. Radicioni. 2015. “Measuring Rill Erosion by Laser Scanning.” Catena 124. https://doi.org/10.1016/j.catena.2014.09.003.


The content can be located at [E:\Yichuan](E:\Yichuan), on the workstation (WCMC-PC-01918.internal.wcmc)

Most of my content is organised into folders with names of collaborator or those who request help. This is the highest level. Within this level, project files can be found if this is a one-off request; or folders representing different projects. 

Folders named *not after* people usually refer to one-off/non-specific, large or recurring projects. Below is a concise explanation of important folders (the most important folders are in bold), or those needs clarification:

1. Admin: all admins, including timesheets, travel authorisations, visas, tickets, cost reclaims and etc
2. **Basedata**: most base data are located here (spatial). Typically they include base physical, cultural boundaries, grids, hillshade, ecoregions etc. Recommendation: use esri ArcCatalog to navigate
3. COM_XXX: preparations for World Heritage Committee meetings
4. **Comparative_analysis_20XX**: results of comparative analysis of that year. They will generally include digitised polygons, results of the analysis in tables, and also maps (including map templates that create them)
5. Dggrid: discrete global hexagon grids program
6. Documents: unsorted *docx* or *pdf* format documents prior to 2014
7. Dump_PG: unsorted database dump prior to 2014
8. **Elena**: including World Heritage Outlook data dumps and analysis files are located
9. Experiment: test space for various tools, analysis and scripts, unsorted and **do not use**
10. Good_reference_map_collection: a collection of maps for reference
11. **IT**: project folder for the first World Heritage Outlook system development 
12. Landsat_archiving: python based script for the archiving project, superseded by the [near real time Landsat 8 images](http://wh-app.yichuans.me/landsat) in the Lab. **do not use**
13. Map_templates: outdated template **do not use**
14. Mapmart: 0.5m resolution WorldView imagery purchased to identify mining activities, in Russia
15. **MyGDB.gdb**: a somewhat historic/duplicate place holder for base data, but in esri geodatabase format. It mainly contains biogeographic, broadscale priorities and old site level priorities such as KBA datasets. It also includes world vector shore line data (including EEZs). While I do not recommend using any of the datasets, **do not delete** this database, as there may be map documents referring to the data here-within 
16. MyWorkplace.gdb: similar to 'Experiment' folder but in geodatabase format
17. Papers: feeder folder for Mendeley, the reference management system for papers
18. **Red_List_data**: raw Red List data releases from 2013
19. Remote sensing: test data for raw satellite images (mostly Landsat) collected throughout my time for ad-hoc projects
20. **Scripts**: arguably the most important folder. Sub folder structure below
	- **/geoprocessing**: scripts and libraries for undertaking spatial analysis, generic data analysis, workflow automation etc. See the [geoprocessing](geoprocessing.md) section for more information. This is also backed up on [GitHub](https://github.com/Yichuans/geoprocessing)
	- **/scripts**: historic place holder for scripts. Most one off, ad-hoc scripts are here.
	- /mysql20110516: outdated SQL scripts for maintaining the PostgreSQL database
21. sites_improve_geometry_XXX: series of attempts to improve the qualities of WH datasets until 2015
22. sitesXXXX: digitised boundaries for new nominations, and successful inscriptions and/or modifications. 
23. TentativeList: outdated tentative list sites improvement. **do not use**
24. Updates: outdated map document to update the database and no longer relevant **do not use**
25. WDPA: copies of WDPA monthly releases for analysis
26. WH_benefits: project folder for the first WH benefits analysis
27. **WH_stats**: statistics after COM
28. WHO_geom_updates: methodology for updating the maps in the old WH Outlook system. This may not be relevant any more
29. **WHS.gdb**: spatial data for natural and mixed World Heritage sites, in geodatabase format
30. **WHS_arcgisonlineXXX.gdb**: spatial data to be zipped and uploaded to the arcgisonline. This is the data behind the [esri feature service](https://wcmc.io/world-heritage-data)
31. **WHS_dump_ATTR**: attributes/non spatial data for natural and mixed World Heritage sites
32. WHS_dump_KML: historical dumps of WH data in KML format, superseded by the feature service
33. WHS_dump_SHP: shapefile format of the WH data
34. WHS_map_batcher: map template to automate map production
35. WHS_quality_check: map comparisons between GIS, official maps from UNESCO, including retrospective ones. The numerous comparisons underpin and empower the systematic check and subsequently the improvement of the data.
36. Yichuan: miscellaneous
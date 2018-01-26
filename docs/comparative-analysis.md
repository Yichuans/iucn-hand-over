# Introduction

The comparative analysis is an unbiased, consistent analysis of new biodiversity nominations, i.e. criteria (ix) and (x), against existing World Heritage sites. It has been an important piece of information for the consideration of the IUCN World Heritage Panel. The methodology has been documented in the paper [comparative analysis methodology for World Heritage nominations under biodiversity criteria](https://www.unep-wcmc.org/resources-and-data/comparative-analysis-methodology-for-world-heritage-nominations-under-biodiversity-criteria), published in 2014. 

This page aims to introduce the spatial aspect of the analysis and the evolving engineering behind it. 

**Updates**

- Additional documentation can be found on [GitHub](https://github.com/Yichuans/geoprocessing/tree/master/comparative_analysis), some duplications may be present

For past comparative analysis results, please visit the section of [my folder structure](folder-structure.md)

# Challenges

Conceptually, the idea of undertaking spatial analyses for the comparative analysis is rather simple. At the lowest level, it involves the spatial intersection of the boundary of a given new nomination, against a base layer, and then extract some statistics. For example, overlay the terrestrial ecoregion layer with new nominations, and calculate the total area and the proportion of overlap.

![overlay](./img/overlay.gif)

The complexity arises when additional variables come into play. 

The first challenge is the plethora of base layers required for the analysis. Despite the increasing availability of global datasets on biodiversity, no data has been so far collated specifically for the purpose of inscriptions of biodiversity World Heritage sites (nor should there ever be). The narratives around the arguments for [Outstanding Universal Values](http://biodiversitya-z.org/content/outstanding-universal-value-ouv) do not provide a clear direction to seek suitable datasets that are also spatially explicit. As a result, the spatial analysis must take into account most, if not all (to the extent possible re time and resources) biodiversity datasets, as to provide the maximum relevant information to support experts in their decision making process.

Next, with heterogeneous data sources, different questions require different answers. For example, methodologies to answer a question about the (over)representation of a certain biogeography differ from that inquiring the indicative number of species that might be present. Some seemingly different questions can be grouped employing a similar process to analyse, while others with common traits in the narratives may dictate a drastically different methodology to be developed

Additionally, to enable comparisons, not only nominations are required to be analysed: existing sites, sites on the Tentative List, and in some case certain protected areas need considering as well. These additional datasets multiply the computational load, produces significantly more information, and complicates the process of interpretation and presentation. 

Finally, the numerous, somewhat abstruse outcome must be put in a lay format that makes sense to the audience. They have to be digested and presented in a clear and logic way.

# Bio-geographic classification, and priorities

One of the classic tasks of the comparative analysis is to look at whether a certain natural environment has been over-represented. If so, justifying another site in the same environment being better than those already on the List becomes significantly more difficult. On the contrary, if a nomination represents a previously unrepresented or under-represented space, the argument for inscription is strengthened.

From a technical perspective, these types of comparisons require a simple overlay with the same set of statistics. Such data include the terrestrial/marine ecoregions, broad scale priorities such Global 200 priority, biodiversity hotspot, and site level priorities such as KBAs.

The current implementation relies on the PostgreSQL database and the PostGIS spatial extension. ArcGIS plays a role in importing and exporting but is optional.

The source code of the analysis can be found in detail below

- [https://github.com/Yichuans/geoprocessing/blob/master/comparative_analysis/comparative_analysis.py](https://github.com/Yichuans/geoprocessing/blob/master/comparative_analysis/comparative_analysis.py)

In short, the script goes through the dictionary holding look-up information about each layer (name, the schema and table etc), and then intersects a 'theme' with a combined view that has both World Heritage sites and nominations using the function `run_ca_for_a_theme`. It performs two tasks: a) intersection 2) group and combine the resulting statistics. This design accounts for single part polygons, and that results need to reflect different scales (for example, for terrestrial ecoregions, realms and biomes). As the last step, the function `post_intersection_mk2` filters only those results relating to the nominated site.


To undertake the analysis, below are the steps required

1. Load the newly nominated sites to the `ca_nomi` schema, preferably via ArcGIS, using the `PG_Geometry` configuration. It ensures PostGIS understands and knows how to analyse the geometry
2. Create in the PostgreSQL database an empty schema, in which results of the analysis would be placed. Typically, this is called `ca_xxxx` (xxxx refers to the year when the analysis is done)
3. Update the base layers if necessary
4. Create a new main function like below

```python
def ca_2017():
    input_nomination = 'ca_nomi.nomi_2017'
    output_schema = 'ca_2017'
    for themekey in BASE_LOOKUP.keys():
        run_ca_for_a_theme(input_nomination, output_schema, themekey, conn_arg=get_ca_conn_arg(2015))
```

The first line refers to the table of the nomination in the database, and the second the location of output. The 'for' loop programatically picks each and every base layer, a.k.a., theme, to intersect with the combined view (internally, the `create_combined_wh_nomination_view` function creates one). Although only the nomination table is specified, the script knows where to find the World Heritage boundary, i.e., `arcgis.v_wh_spatial`. The maintenance of the World Heritage boundary is documented in the [database section](world-heritage-database.md). 

It is important to note that an arbitrary threshold of 0.05 is specified to minimise commission errors. This means that any overlap with proportions less than 5% is ignored, as we assume such overlap is likely a result of [scale issues](https://www.e-education.psu.edu/geog469/node/262), due to differences in accuracy or scale.

# Increased productivity

The above process can be improved simply by running the `run_ca_for_a_theme` simultaneously and in parallel. One example is included below (snippet from the 2017 comparative analysis)

```python
def f(themekey):
    input_nomination = 'ca_nomi.nomi_2017_with_supp'
    output_schema = 'ca_2017_with_supp'
    run_ca_for_a_theme(input_nomination, output_schema, themekey, conn_arg=get_ca_conn_arg(2015))


# wrap in main
if __name__ == '__main__':
    p = Pool(10)
    keys = BASE_LOOKUP.keys()
    print keys
    print(p.map(f,keys))
```

# Tentative List sites

If one were to look deeper, both the World Heritage sites and tentative lists do not change, at least not constantly. Thus the results may be cached, i.e., there is no need to re-do the overlay between them the many base layers. Whenever a comparison is needed, one can simply query the cache and pull out relevant records. This is especially true for Tentative List sites, which tend to stay the same for years.

For tentative list sites, the `run_ca_tentative` utilises the same logic in the above methodology and produces a set of database views holding the result.

# Export to excel

With the results fully generated in the database, I built a process to stitch relevant tables, including lookup tables for contextual information, and export to excel formats to simplify interpretation.

Each nomination has an excel table with three tabs, referring to 'biogeographic', 'priority', and 'site-level' respectively. This table includes numerical comparisons between sites.

The 2017 script can be found below

- [https://github.com/Yichuans/geoprocessing/blob/master/comparative_analysis/comparative_analysis_to_excel_2017.py](https://github.com/Yichuans/geoprocessing/blob/master/comparative_analysis/comparative_analysis_to_excel_2017.py)

For the export to work, the following variables need to be set in the script. The below is taken from the 2017 script

```python
# 2017

COMBINED_WH_NOMINATION_VIEW = 'z_combined_wh_nomination_view'
WH_ATTR = 'arcgis.v_wh_non_spatial_full' # attribute look up table for world heritage
TLS_SHAPE = 'tls.tentative'	# tentative list site table

OUTPUT_SCHEMA = 'ca_2017_with_supp'	# the location of where the output tables are stored
TLS_SCHEMA = 'ca_tls'	# the location of where the output table for tentative list sites are stored
TLS_ORGIN = 'tls.origin'	# the original tentative list sites table, for looking up criteria information

# List of nomination IDs
NOMI_ID = range(9991701, 9991706) + range(99917011, 99917015) + range(99917021, 99917023)

```

Lastly, specify the location of the output `outputfolder`

```python
if __name__ == '__main__':
    outputfolder=r"E:\Yichuan\Comparative_analysis_2017"
    main(outputfolder)
```

# Full result export

Sometimes it is necessary to investigate 'apparent' overlaps. This often means either there is a reasonable doubt about the finding being incorrect or an odd, baffling outcome that contradicts expectations.

To this end, the actual proportion of overlap is often required. This effectively requires a full dump of the database, including intermediate results and it can be cumbersome to read.

Since 2016, this functionality has been added in a separate script, called `comparative_analysis_to_excel_fullresults`([source code](https://github.com/Yichuans/geoprocessing/blob/master/comparative_analysis/comparative_analysis_to_excel_fullresults_2017.py)). The principal idea is to dump original results of each base layer into 'tabs' in an excel table, in which the overlap of actual area and proportion are recorded. As mentioned before, this is only to be used as a reference, in cases where apparent overlaps may be questionable or justifying findings that are counter-intuitive.

# Species richness

To counter the often inflated number of species reported in nomination dossiers, a consistent metric has been developed that counts the indicative species richness by interrogating [the IUCN Red List of Threatened Species](http://www.iucnredlist.org). This presents an opportunity to examine the relative abundance of biodiversity through surrogates of comprehensively assessed taxa - it is particularly useful for criterion (x)

The IUCN Red List of Threatened Species has a spatially explicit database that could be requested from its website or directly from the Red List Unit. The usual practice is to include only species range polygons of presence 1 and 2, origin 1 and 2, and seasonality 1, 2, and 3. Clean the database by repairing and dicing if necessary. If not present already, create an index on the field `id_no` that uniquely identifies a species. This speeds up the analysis considerably.

The next step is to programatically call `arcpy.SelectLayerByLocation` for each species and the World Heritage sites and nominations. It may take a long time to go through every one of the 70k+ species with boundaries. The end result is a two column table recording the `id_no` for species and `wdpaid` for WH sites and nominations. 

Load the resulting table and the non-spatial Red List data to the same PostgreSQL database with rest of the results. The species richness makes lots of assumption and refers to the same look up tables in the section on [Bio-geographic classification, and priorities](#bio-geographic-classification-and-priorities)

The following information need to be specified in the script `comparative_analysis/comparative_analysis_group_species.py` ([source code](https://github.com/Yichuans/geoprocessing/blob/master/comparative_analysis/comparative_analysis_group_species.py)), which retrieves information from lookup tables and generate a master view with different species richness count for different taxonomies (including only those that are threatened)

```python
# the resulting richness analysis table
all_sp = "ad_hoc.species_ca_2017"
all_sp_taxonid = 'species_ca_2017.id_no' # must not be the same as all_sis_taxonid
all_sp_baseid = 'species_ca_2017.wdpaid'

# look up table, non spatial version of the input Red List data
all_sis = "ad_hoc.rl_2017_2_pos"
all_sis_taxonid = "rl_2017_2_pos.id_no"

# WH/nomi name look up table
# NOTE: assuming wdpaid is present!!!!!
wh_nomi_lookup = "ca_2017_with_supp.z_combined_wh_nomination_view"
wh_nomi_name = "z_combined_wh_nomination_view.en_name"

# name
KINGDOM_FIELD = 'kingdom'
CLASS_FIELD = 'class'
BINOMIAL_FIELD = 'binomial'
RL_FIELD = 'code'
```

Lastly, specify the output schema, typical the same location as the one for the biogeographic/priority results

```python
main('ca_2017_with_supp')
```

The technical implementation is also documented in the [richness template section in geoprocessing](geoprocessing.md#richness-template)

# Irreplaceability

The irreplaceability implements the methodology described by [the paper in Science](http://science.sciencemag.org/content/342/6160/803). 

The shortcoming of using the Red List range polygons is rather obvious: it does little to mitigate the effect of over-estimation of their Area of Occupancy (AOO) and that richness counts almost certainly inflate the actual number of species. The irreplaceability metric overcomes this by using sigmoid functions to translate actual proportional overlaps and aggregate to a single value that, to a certain extent, captures both richness and endemism.

The implementation can be found below ([source code](https://github.com/Yichuans/geoprocessing/blob/master/irreplaceability/explorer.py))

```python

def species_irreplaceability(x):
    """x is the percentage in decimal"""

    x = x*100

    def h(x):
    # h(x) as specified
        miu = 39
        s = 9.5
        tmp = -(x - miu)/ s
        denominator = 1 + np.exp(tmp)
        return 1/denominator
    

    return (h(x) - h(0))/(h(100) - h(0))

```

Because the irreplaceability analysis builds on the full intersection between the WDPA and the Red List (the pairwise percentage overlap value), which is in itself a massive undertaking, I cannot re-run irreplaceability without being given the result of the full intersection first. For this reason, up to this day, only two runs exist (the latest updated in 2015, which we rely on).

# Static maps

Maps accompany the comparative analysis are authored using ArcMap, and later ArcPro. 

The template is pre-authored and re-used, although labelling and styles may be dynamically adjusted. To the extent possible, I try to automate the production of maps. In ArcMap, this is done via `arcpy.mapping` package that produces maps by iterating each feature in the nomination feature class. The export process outputs any format supported by ArcMap, usually in PNG for the reports or AI/EPS/PDF for printing. The source code could be found in the [geoprocessing section](geoprocessing.md#map-batching-template)

Later, new capabilities in ArcPro make the above process obsolete. There is no need any more to write any code. The iteration instead require a pre-cooked extent feature class be created to represent the viewport of maps. The default data-driven pages can loop through each extent feature and export maps of desired formats. Worth mentioning is that PDF format maps also have the ability to turn on and off layers, that in a way mimics behaviours of a dynamic map, enabling readers to interrogate the map themselves. 

The 2017 static maps can be found below:

- [E:\Yichuan\Comparative_analysis_2017\static_maps](E:\Yichuan\Comparative_analysis_2017\static_maps)


# Interactive maps

[Interactive maps](https://en.wikipedia.org/wiki/Web_mapping) are revolutionising the way people use maps. No longer needed are multiple static maps to represent different scales and perspectives, due to the [progressive disclosure design](https://en.wikipedia.org/wiki/Progressive_disclosure) of these dynamic maps. One can continue to present a particular view to the audience, but more importantly users are now empowered to make their own choices and inspect whatever they deem useful and interesting.

Below is an example of the 2017 dynamic maps for the comparative analysis, and it works on all devices, even allowing embedding in this handover report. 

<iframe width='320' height='480' frameborder='0' scrolling='no' marginheight='0' marginwidth="0" src='https://www.arcgis.com/apps/webappviewer/index.html?id=a0a336d33e854dd7b06e123e5390186e'></iframe>

# Conclusion

In hindsight, the current implementation of comparative analysis is probably over-engineered (slightly). The original implementation relies entirely on SQLs, which had severe shortcomings regarding flow control. It was extremely inflexible in a pure database environment. Python, being a generic ['glue language'](https://en.wikipedia.org/wiki/Scripting_language), was chosen to generate SQL statements and feed to the database management system. The result is a somewhat haphazard concoction of [spaghetti codes](https://en.wikipedia.org/wiki/Spaghetti_code), and unnecessarily engineered with an Objective Oriented Design. 

For future improvement, a pure function based structure may be a better alternative. Or, considering the massive improvement of the `arcpy` library and the fine controls of geometry it now has, it may be wise to migrate back to an ArcGIS based system, as it has become increasingly easier to develop and maintain. Further debate on pros and cons is required - going back to a solution based on proprietary software seems counter-intuitive in the present day of open data and open science.
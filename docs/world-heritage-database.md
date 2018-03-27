# Introduction

For the most part, the database hosts the natural World Heritage sites boundary, i.e., the actual GIS data, and other relevant, essential information such as criteria, types and endangered status. 

It was originally built to power the comparative analyses, but at the same time serves as a pipeline to other formats, for example, for inclusion in the WDPA, shapefile/geodatabase feature class, the [World Heritage Feature service](https://wcmc.io/world-heritage-data)(via data dumps)

The management of the database is noticeably different from other datasets in the WDPA. Rather than using the proprietary esri ArcGIS suite, it sits in an open source enterprise database [PostgreSQL](https://www.postgresql.org) with the [PostGIS](http://postgis.org) spatial extension. The key benefit, apart from having no lock-in or dependency on expensive commercial software, is that the data can be managed in an industry standard way, and freely, using the SQL language.

# PostgreSQL database

Unfortunately, being only a database, PostgreSQL/PostGIS does not have a GUI(graphic user interface) that allow read, create, update or delete spatial data, essential tasks for a GIS, which ArcGIS excels in.

To make the best of two worlds, a compromised solution was engineered to allow ArcGIS to interface with the underlying database. It was not an easy solution back in the days and some of the [techniques](http://desktop.arcgis.com/en/arcmap/10.3/manage-data/gdbs-in-postgresql/data-types-postgresql.htm) may have become obsolete over time. There are probably better options these days - therefore I intend not to complicate matters by explaining *how to set up* the system as is, but rather focusing on *how to use* the system.

Here are the guiding principles when using the database.

1. Use ArcGIS (ArcCatalog) to view, add, modify and delete World Heritage data
2. **Exercise caution** when using PostgreSQL to **modify** information specifically managed by ArcGIS, notably in the `sde` schema, as they might irreversibly corrupt the data, rendering the ArcGIS interface unusable. For example, **never delete** World Heritage tables in the `sde` schema in PostgreSQL directly!
3. Create spatial functions in views, such as `st_intersects`, in PostgreSQL/PostGIS to analyse data, instead of using equivalent ArcGIS functions (really, this is the main reason for the setup!)

What is in the `sde` schema?

The schema is effectively managed by the ArcSDE (Spatial Database Engine), a middleware bridging between the ArcGIS and the underlying enterprise level database. One important thing to note is that in order for the database to work both ways, `PG_Geometry` must be specified as the type. If not, by default the native ArcGIS geometry would be use, which PostgreSQL/PostGIS will **not** be able to recognise and use.

1. `b_wh_iso_geom`: the single most important table, with the geometry field. It holds boundary data, including past records since 2013 (when the migrate to this system happened). Each site, identified by the id `wdpaid`, may have multiple records, i.e. several boundaries or geometries, depending on the number of past modification or updates. The system relies on the `year` and `event_cat` fields to find the most recent boundary. This table also features the `iso3` field to differentiate transboundary site, enabling easy aggregation of country level statistics

2. `b_wh_attr`: non-spatial attributes, including names, status_year (first inscription) and reported areas

3. `b_wh_ids`: a mapping of `wdpaid` and `unesid`, which is a different set of ids referencing the systems used by UNESCO

4. `b_wh_crit`: a mapping of `wdpaid` and `crit`, useful for grouping results based on criteria

5. `b_wh_region`: country name, unesco regions and subregions

6. `blk_crit`: a lookup table converting numeric numbers and Roman letters (useful for the `crit` field)

7. `tag_unesco_xxx`: tags for UNESCO dangered status, forest and marine

The above tables form the basis of the World Heritage database, however, the direct use of them may be inconvenient or difficult, and thus is discouraged. 

For the purpose of using the database, a set of convenient views (consider them as temporary or dynamic tables) have been created and placed under the `arcgis` schema. They are higher-level and more commonly used. 

1. `v_wh_spatial`: a feature class like view, with similar WDPA fields. This builds on the `v_wh_iso_geom_latest` and other relevant information
2. `v_wh_spatial_non_agg`: the same as above but transboundary sites are recorded as separate entities, i.e., not aggregated as one geometry.
2. `v_wh_iso_geom_latst`: the latest geometry for each site (queried from the `sde.b_wh_iso_geom` table)
3. `v_wh_non_spatial_xxx`: like `v_wh_spatial` but without the geometry field - faster view as it does not retrieve any geometry
4. `v_wh_geojson`: geojson format export
5. `dump_v_wh_for_wdpa`: feature class export, for inclusion in the WDPA

These views can be inspected within a database management environment such as [PgAdmin](https://www.pgadmin.org), exported to other formats (CSV, shapefiles or others), or open directly in the ArcGIS environment via a [database connection](http://desktop.arcgis.com/en/arcmap/10.3/manage-data/databases/database-connections-desktop.htm). Since they are dynamically generated views, as long as the underlying data are up-to-date, they always reflect the latest in the database.

# How-to: update the database

I update the database annually, after the conclusion of World Heritage Committee meeting, in July or August. This ensures decisions on nominations and modifications are timely reflected. I also take this opportunity to incorporate improved boundaries (usually higher accuracy) for other sites, for example, those submitted by States Parties as part of their updates to the WDPA.

To update the database, one may go directly to the database without using ArcGIS, but it is less straightforward and requires a higher level of technical sophistication. This method usually involves using command line tools such as `psql` and `shape2pgsql` to convert input shapefiles (see [PostGIS pgsql2shp shp2pgsql cheetsheet](http://www.bostongis.com/pgsql2shp_shp2pgsql_quickguide.bqg)) to SQL statements, and then issue separate SQL commands directly to add or modify records in relevant tables under the `sde` schema.

For example, the below command imports a shapefile to a staging location in the database

```bash
shp2pgsql -g geom -s 4326 -I -w "LATIN1" E:\Yichuan\sites2013\edit_38COM.shp public.z_updategeom | psql -p 5432 -d whs_v2 -U postgres
```

I recommend using ArcGIS to interface with the database - it is a design feature for convenience. Use editing tools in ArcGIS to update the database (as you would normally do with any spatial data). For further convenience, I created an esri map template for this purpose. For example, the 2017 updates can be found at:

- [E:\Yichuan\sites2016\inscription\inscription.mxd](E:\Yichuan\sites2016\inscription\inscription.mxd)

Ensure all `sde.b_xxx`, `sde.tag_xxx` and `sde.ref_iso3` tables are updated.

# Statistics

Being database views, statistics reflect automatically and dynamically from the latest dataset. They are placed under the `arcgis` schema, and involve: total World Heritage area and average size (in sqkm), number of sites, categories such as forest, marine, danger listing. Since the summaries share similar traits, I re-used the same logic but produced statistics at different levels. The statistics are mainly for the [World Heritage factsheet](https://www.iucn.org/theme/world-heritage/natural-sites/facts-and-figures) (it requires additional information about global statistics from the WDPA)

- `stats_wh_site`: site level
- `stats_wh_country`: country level
- `stats_wh_region`: regional level
- `stats_wh_world`: global level

Marine and terrestrial statistics require an ad-hoc spatial analysis, as site boundaries are not saved as split entities between the two realms. The result can be found in the `arcgis.intersect_wh_wvs`. This view takes a very long time to open due to significant work involving geometric operations. For convenience, I would recommend a table `t_intersect_wh_wvs_xxx` be created (and replace the old table) to save time - it makes subsequent queries a breeze...
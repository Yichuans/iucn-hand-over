# Introduction

For the most part, the database host the natural World Heritage sites boundary, along with other relevant auxiliary information such as criteria, types and endangered status. 

It was originally built to power the comparative analyses, but at the same time serves as a pipe to other formats, for example, for inclusion in the WDPA, shapefile/geodatabase feature class, the [World Heritage Feature service](https://wcmc.io/world-heritage-data)(via data dumps)

The management of the database is noticeably different from other datasets in the WDPA. Rather than using the proprietary esri ArcGIS suite, it resides in an open source enterprise database called [PostgreSQL](https://www.postgresql.org) with the [PostGIS](http://postgis.org) spatial extension. Apart from having no lock-in or dependency on potentially expensive commercial software, this allows an industry standard, and free way to manage data using the SQL language.

# PostgreSQL database

This section explains the technical details behind the database.

Technical specification

- Software: PostgreSQL 9.1.14 and PostGIS 1.5 (up since 2013?)
- Server: WCMC-PC-01918
- Port: 5432
- Database: whs_v2
- Schema: sde, arcgis
- Username: postgres
- Password: (see email)

Unfortunately, being a generic database and while good at managing data, PostgreSQL/PostGIS does not have a GUI(graphic user interface) that allow easy visualisation and tools to create, update or delete spatial data. ArcGIS on the other hand excels at such tasks.

To make the best of two worlds, a compromised solution was engineered to allow ArcGIS to interface with the underlying database. It was not an easy solution back in the days but some of the [techniques](http://desktop.arcgis.com/en/arcmap/10.3/manage-data/gdbs-in-postgresql/data-types-postgresql.htm) may have become obsolete over time - therefore I intend not to complicate matters by explaining *how to set up* but rather focusing on *how to use*.

Here are the principles when using the database.

1. Use ArcGIS (ArcCatalog) to add, modify and delete World Heritage data
2. **Exercise caution** when using PostgreSQL to modify information specifically managed by ArcGIS, notably in the `sde` schema, as they might irreversibly corrupt the data, rendering the ArcGIS interface to fail. For example, **never delete** World Heritage tables in the `sde` schema in PostgreSQL directly!

What is in the `sde` schema?

1. `b_wh_iso_geom`: the single most important table. It holds boundary data, including historical records. Each site, identified uniquely by `wdpaid`, may have several boundaries or geometries, depending on their history of modification or updates. The system relies on the `year` and `event_cat` fields to find the most recent boundary. This table also features `iso3` field to differentiate transboundary site, enabling easy aggregation of country specific statistics

2. `b_wh_attr`: the non-spatial attributes, including names, status_year (first inscription) and reported areas

3. `b_wh_ids`: a mapping of `wdpaid` and `unesid`, a different set of ids referencing the systems used by UNESCO

4. `b_wh_crit`: a mapping of `wdpaid` and `crit`, useful for grouping results based on criteria

5. `b_wh_region`: country name, unesco regions and subregions

6. `blk_crit`: a lookup table converting numeric number and Roman letter for criteria

7. `tag_unesco_xxx`: tags for unesco dangered status, forest and marine

The above form the basis for the World Heritage database. While these are the tables to update, after committee decisions, however, direct uses of the above may be inconvenient or difficult. 

For this purpose, a set of convenient views (consider them as temporary or dynamic tables) have been created and placed under the `arcgis` schema.

What is in the `arcgis` schema?

1. `v_wh_spatial`: a feature class like view, with similar WDPA fields. This builds on the `v_wh_iso_geom_latest` and other relevant information
2. `v_wh_spatial_non_agg`: the same as above but transboundary sites are recorded separately by country, i.e., not aggregated.
2. `v_wh_iso_geom_latst`: contains the latest geometry from the `sde.b_wh_iso_geom` table
3. `v_wh_non_spatial_xxx`: like `v_wh_spatial` but without the geometry field - faster view as it does not require any geometry
4. `v_wh_geojson`: geojson format export
5. `dump_v_wh_for_wdpa`: feature class export, with WDPA attributes

These views can be inspected within a database management environment such as [PgAdmin](https://www.pgadmin.org), exported to other formats (CSV, shapefiles or others), or open directly in the ArcGIS environment via a [database connection](http://desktop.arcgis.com/en/arcmap/10.3/manage-data/databases/database-connections-desktop.htm). Since they are dynamically generated views, as long as the underlying data are up-to-date, they always reflect the latest in the database.

# How-to: update the database

The database is updated annually after the conclusion of World Heritage Committee meeting, in July or August. This ensures decisions on nominations and modifications are timely reflected. I also take this opportunity to consider improved/higher accuracy boundaries of other sites, for example, submitted by States Parties via updates to the WDPA.

To update the database, you may go directly to the database but it is less straightforward and requires a higher level of technical sophistication. This method usually involving using command line tools such as `psql` and `shape2pgsql` to convert input updating shapefiles (see [PostGIS pgsql2shp shp2pgsql cheetsheet](http://www.bostongis.com/pgsql2shp_shp2pgsql_quickguide.bqg)), and then issue SQL commands directly to add or modify records in relevant tables under the `sde` schema.

For example, the below import a updating shapefile to a temporary location

```bash
shp2pgsql -g geom -s 4326 -I -w "LATIN1" E:\Yichuan\sites2013\edit_38COM.shp public.z_updategeom | psql -p 5432 -d whs_v2 -U postgres
```

As a design feature for convenience, use ArcGIS to update the database. There is an esri map template for this purpose. For example, the 2017 updates can be found at:

- [E:\Yichuan\sites2016\inscription\inscription.mxd](E:\Yichuan\sites2016\inscription\inscription.mxd)

Ensure all `sde.b_xxx`, `sde.tag_xxx` and `sde.ref_iso3` tables are updated.

# Statistics

Statistics are placed under the `arcgis` schema. They involve: total World Heritage area and average size in sqkm, number of sites, by forest, marine, danger listing.

- `stats_wh_site`: site level
- `stats_wh_country`: country level
- `stats_wh_region`: regional level
- `stats_wh_world`: global level

Marine and terrestrial statistics require an ad-hoc spatial analysis, as geometry is not saved as split entities between the two environments. The result can be found in the `arcgis.intersect_wh_wvs`. For convenience, a table `t_intersect_wh_wvs_xxx` created to save time.
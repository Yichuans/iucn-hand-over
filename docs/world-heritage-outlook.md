# Technical architecture

The assessment module was written in Java, a custom-made system that accepts assessments via the web interface and store in a PostgreSQL database. The user facing front end website displays results of assessment and was implemented in Liferay, which during the second development changed to a Drupal based system (written in PHP).

At the moment, there is a disconnection between the two systems: backend assessment and frontend display. The assessment module (backend) and the old website (frontend display) are on one Amazon EC2 instance (instance name: 'IUCN WHP WHO') and the new, updated website resides another (instance name: 'IUCN WHO V2'). The new website used a database dump to migrate content from the old system. By design, the pipeline built by EDW does not reverse the flow of data in the other direction.

The source codes for the systems can be found on [GitHub](https://github.com/iucn-whp)

# Database dump

One of the key functions of the system(s) is to enable the extraction and analysis of information for the World Heritage Outlook report. For that purpose, it is important to obtain a **copy** of the dataset of the live database (the one behind the website, but **never** work directly on the live database itself!). This is called a database dump.

For security reasons, you will need to SSH tunnel to the instance and forward the remote port to a local one. There is a saved session called `WHO_backend` that contains necessary connection parameters, including those to forward the remote 5432 port (the database port) to the local 5433 port. 

![putty](./img/ssh-tunnel.png)

Once you've successfully logged using that saved session, open PgAdmin, the database management tool to connect to your local port 5433, which now points to the remote machine via an SSH tunnel. 

![pgadmin](./img/pgadmin.png)

You may now select the database from which you'd like to make a dump. Use the 'backup' button (see red arrow) to create a file which holds the content of the database.

![dump](./img/dump.png)

On your local machine, create an empty new database and then 'restore' (see the blue arrow) to create an identical copy of the remote database. 

This concludes the database dump or 'extraction of data'.

# Convert to Access database

Unfortunately the data behind the website is not designed in a format that allows users to gain useful insight about the assessments directly - after all the system has been designed to power the I/O of assessments. As a result, the database dump contains many system tables that do not hold assessment information, and that this information is fragmented and split into many smaller tables. You may refer to the diagram to better understand the relationships of these tables.

- [E:\Yichuan\IT\ER_diagram_130802](E:\Yichuan\IT\ER_diagram_130802)

For the most commonly used queries, I have built a re-usable Microsoft Access database that connects to the PostgreSQL database. These links must be updated each time a new database dump is made (Matea knows how to do this). By using the Access database as an intermediate, it allows the export to excel spreadsheets, which some find easier to work with. 

Below is an example of the 'overall' view in the Access database.

- [E:\Yichuan\Elena\WHOA_171027](E:\Yichuan\Elena\WHOA_171027)

![view](./img/view.png)

**!IMPORTANT!** 

For the data dump, however, there is one manual step that needs to be performed to assist the analysis before updating the links. 

We need to differentiate the most recent assessment ids, and separate them in the two assessment cycles. This is essential to pull out the correct versions of the assessments for analysis. This step can be done in Access but would be much easier in the PostgreSQL database. In your newly restored database, create a view called 'z_wdpa_latest' using the below:

```sql
-- View: z_wdpaid_latest
-- DROP VIEW z_wdpaid_latest;

CREATE OR REPLACE VIEW z_wdpaid_latest AS 
 WITH a AS (
         SELECT whp_whp_sites.wdpa_id, whp_site_assessment_versions.assessment_id, whp_site_assessment_versions.assessment_version_id, whp_whp_sites.name_en, whp_site_assessment.assessment_cycle, whp_site_assessment_versions.version_code, max(whp_site_assessment_versions.version_code) OVER (PARTITION BY whp_site_assessment_versions.assessment_id) AS max_code
           FROM whp_site_assessment, whp_site_assessment_versions, whp_whp_sites
          WHERE whp_site_assessment.assessment_id = whp_site_assessment_versions.assessment_id AND whp_whp_sites.site_id = whp_site_assessment.site_id AND whp_site_assessment.assessment_cycle::text = '2014'::text
        ), b AS (
         SELECT whp_whp_sites.wdpa_id, whp_site_assessment_versions.assessment_id, whp_site_assessment_versions.assessment_version_id, whp_whp_sites.name_en, whp_site_assessment.assessment_cycle, whp_site_assessment_versions.version_code, max(whp_site_assessment_versions.version_code) OVER (PARTITION BY whp_site_assessment_versions.assessment_id) AS max_code
           FROM whp_site_assessment, whp_site_assessment_versions, whp_whp_sites
          WHERE whp_site_assessment.assessment_id = whp_site_assessment_versions.assessment_id AND whp_whp_sites.site_id = whp_site_assessment.site_id AND whp_site_assessment.assessment_cycle::text = '2017'::text
        ), combined AS (
                 SELECT a.wdpa_id, a.assessment_cycle, a.assessment_version_id
                   FROM a
                  WHERE a.version_code = a.max_code
        UNION 
                 SELECT b.wdpa_id, b.assessment_cycle, b.assessment_version_id
                   FROM b
                  WHERE b.version_code = b.max_code
        )
 SELECT combined.wdpa_id, combined.assessment_cycle, max(combined.assessment_version_id) AS assessment_version_id
   FROM combined
  GROUP BY combined.wdpa_id, combined.assessment_cycle
  ORDER BY combined.wdpa_id, combined.assessment_cycle;

ALTER TABLE z_wdpaid_latest
  OWNER TO postgres;
```

The rationale is to identify the largest value of `assessment_version_code` for each cycle and for each `assessment_id`, on the assumption that the later/newer created id should be a larger number. By finding the largest value of the id, it is guaranteed that the latest versions are used, irrespective of the version number and stage, which can be wrong due to bugs in the original design (I shall omit the technical details, but in short the system allows but did not expect certain user behaviours that may result in incorrect version numbers and stages being created)

Once the above is done, you can now update the links in the Access database, using the 'Linked Table Manager' so that they point to the most recent local database as a source

![update-links](./img/update-link.png)

Make sure you tick 'Always prompt for new location', so that you update links for all tables in one go

![link-manager](./img/link-manager.png)

Create a new data source, here you'll need to specify the details of the newly 'restored' database

![data-source](./img/data-source.png)

# Analysis

The analysis is fully documented below:

- [Analysis of World Heritage Outlook assessment](http://nbviewer.jupyter.org/github/Yichuans/world-heritage-outlook-analysis/blob/master/analysis.ipynb)



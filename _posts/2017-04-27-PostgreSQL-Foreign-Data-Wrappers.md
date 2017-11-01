---
category: Blog
---
# Setting Up Foreign Data Wrappers (FDWs) in PostgreSQL
This is a basic tutorial for connecting external data to your PostgreSQL database through what is called a "foreign data wrapper." 
At the time of this writing, the only FDW that comes prepacked with PostgreSQL includes fdw_file, used to query files on your computer. 
In this example, I am going to pull one from a wiki [catalog](http://wiki.postgresql.org/wiki/Foreign_data_wrappers) of third-party wrappers.

## Step 1: Install the OGR FDW
I already have the spatially-oriented fdw wrapper "ogr_fdw" installed as part of 
the [PostGIS Bundle 2.3 for PostgreSQL x64 9.6](http://postgis.net/windows_downloads/),
using the StackBuilder Application Installer. If not done so already, you will need to make
sure to install the extensions:
```
CREATE EXTENSION postgis;
CREATE EXTENSION ogr_fdw;
```

## Step 2: Install the Foreign Data Server and Setup a Foreign Data Table
In this case, let's say I wanted to query the High Injury Network, which is listed on the GeoHub Portal [here](http://geohub.lacity.org/datasets/4ba1b8fa8d8946348b29261045298a88_0).
```
CREATE SERVER hin
  FOREIGN DATA WRAPPER ogr_fdw
  OPTIONS (
    datasource 'http://geohub.lacity.org/datasets/4ba1b8fa8d8946348b29261045298a88_0.geojson',
    format 'GeoJSON' );
```
After creating the foreign data server, I needed to create a foreign table. The ogr_fdw has a nifty feature for PostgreSQL 9.5+ users; 
you can do automatic foreign table creation, which you can read more about [here](https://github.com/pramsey/pgsql-ogr-fdw). The method that I use below will import all the tables at the datasource, which in this case is only one. If you do want to query only one table at a time, you will need to set a layer name; the default layer name is 'OGRGeoJSON,' based on the information [here](http://www.gdal.org/drv_geojson.html).
```
CREATE SCHEMA hin_schema;

IMPORT FOREIGN SCHEMA ogr_all 
	FROM SERVER hin 
    INTO hin_schema;
```
Once this process is complete, you should refresh your schema bucket, and you should now see a new schema called 'hin_schema.' Scroll down to the foreign table bucket, and you should see a new foreign table called 'ogrgeojson.' 

![FDW_ForeignTableSchema](/images/FDW_ForeignTableSchema.PNG)

If it was configured correctly, you can now query that table as you would query any other table in your database
```
SELECT * FROM hin_schema.ogrgeojson;
```



---
category: Setup Notes
---
# Adding a SRID to PostGIS
_Update: Today I found out that there is an EPSG code for the NAD 83 US Feet, (EPSG:2229). Although it turned out to not be necessary for this projection, you may find the following steps still useful if you come across a less common projection_

When you install the PostGIS extension for PostgreSQL, it comes with all the standard geographic and projected coordinate systems using the
structured EPSG Geodetic Parameter Dataset, which can be found at the [International Association of Oil & Gas Producers EPSG Homepage](http://www.epsg.org/).

However, you may come across a spatial data set that is in a projection that is not part of the EPSG standards. The most obvious example includes
the projections that are part of the ESRI Geodetic Parameter Dataset. You will be able to tell this is the case when the projection label begins
with 'ESRI' instead of 'EPSG.' In my case, for example, I had a data set that was in the NAD 1983 StatePlane California V FIPS 0405 (Feet). The last
part is important, because EPSG includes a projection for NAD 83 StatePlane California V, but it is in meters instead of feet, and using the EPSG (meters)
projection threw my Los Angeles data into the Atlantic. Not good.

This quick tutorial assumes you have PostgreSQL and the PostGIS extension installed and are looking for a way to import data that is in a non-EPSG
format into your database regularly. In my case, I have a spreadsheet in NAD 1983 that is sent weekly, so it is important to be able to import this
data automatically.

## Step 1: Find your SRID
Begin by figuring out which projection your data is in. In my case, I had a spreadsheet with coordinates and no CRS information. Just by testing a few of the
most common projections, I found out that it was in the NAD 1983 StatePlane California V FIPS (Feet), which corresponds to ESRI:102645. Once you get the 
appropriate identfying information on the projection, look it up on [epsg.io](https://epsg.io/). I recommend this site over 
[spatialreference.org](http://www.spatialreference.org/) for reasons I will detail below.

## Step 2: Add your SRID to PostGIS
Once you have found your CRS on the site, find the 'PostGIS' link on the webpage. This super handy feature will automatically
generate a SQL statement for you to insert the information on the new CRS into your database. For example, the insert statement for ESRI:102645 looks like the following:
```
INSERT into spatial_ref_sys (srid, auth_name, auth_srid, proj4text, srtext)
VALUES ( 
    102645,
    'ESRI',
    102645,
    '+proj=lcc +lat_1=34.03333333333333 +lat_2=35.46666666666667 +lat_0=33.5 +lon_0=-118 +x_0=2000000 +y_0=500000.0000000002 +datum=NAD83 +units=us-ft +no_defs ',
    'PROJCS["NAD_1983_StatePlane_California_V_FIPS_0405_Feet",
    GEOGCS["GCS_North_American_1983",
    DATUM["North_American_Datum_1983",
    SPHEROID["GRS_1980",6378137,298.257222101]],
    PRIMEM["Greenwich",0],UNIT["Degree",0.017453292519943295]],
    PROJECTION["Lambert_Conformal_Conic_2SP"],
    PARAMETER["False_Easting",6561666.666666666],
    PARAMETER["False_Northing",1640416.666666667],
    PARAMETER["Central_Meridian",-118],
    PARAMETER["Standard_Parallel_1",34.03333333333333],
    PARAMETER["Standard_Parallel_2",35.46666666666667],
    PARAMETER["Latitude_Of_Origin",33.5],
    UNIT["Foot_US",0.30480060960121924],
    AUTHORITY["EPSG","102645"]]');
```
Go ahead and copy the INSERT statement into a new SQL Query within the database. You should now be good to go!

Note: If you grab information from spatialreference.org and generate the SQL statement and try to insert the SRS into your database, you may get an error (I did in my case). After some researching, I found out that it is because, for whatever reason, spatialreference.org adds a '9' to the beginning of the SRID, which then causes an error in PostGIS. You will not have this problem with epsg.io, which is why I recommend that one.

## Step 3: Enjoy your new SRS
You should now be able to reference the SRID using the integer value that you supplied when creating the new SRS. Good Luck!

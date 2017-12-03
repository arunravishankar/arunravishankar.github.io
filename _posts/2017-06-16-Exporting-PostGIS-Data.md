---
category: Setup-Notes
---
# Exporting PostGIS Data
This is going to go over a couple of different ways to get your data out of a PosgtreSQL table into some other format on your drive.

## Prerequisites

### GDAL / OGR
If you are thinking of exporting your data into the Esri File Geodatabase format (FileGDB), you are going to need to OGR. If your python installation was part of [Anaconda](https://www.continuum.io/downloads), you will already have the OGR package installed. However, as I'll mention below, you will likely need to take a few more steps to be ready to export.

If you want to see a list of all the available drivers within OGR, use the ogrinfo command line utility program and run the following command:
```
ogrinfo --formats
```
For ESRI super users, you will note that there is support already provided for the ESRI Shapefile format. However, note that there is no built-in support for the ESRI File Geodatabase format, which brings us to the next step...

### ESRI FileGDB Driver
GDAL / OGR is a pretty handy package, and it is already setup for output to the ESRI Shapefile format. However, for those of us primarily working in ESRI File Geodatabases, you will be SOL because it won't come with the drivers necessary to write Esri File Geodabases. Fortunately, [others](https://glenbambrick.com/2017/03/10/setting-up-gdal-with-filegdb/) have put together helpful tutorials on getting the FileGDB driver up and running for you analyses (instructions are specific to Windows). In general, you need to:
1. Make sure you have Microsoft Visual C++ Service Pack (linked instructions use 2008)
2. Download the [GDAL wheel](http://www.lfd.uci.edu/~gohlke/pythonlibs/#gdal) from Christoph Gohlke's website (for the whl installation, see Note (1) below).
3. Download the necessary drivers through the [File Geodatabase API](http://appsforms.esri.com/products/download/#File_Geodatabase_API_1.3) provided by ESRI.
4. Create a New Variable in Environmental Variables
5. Open init.py from osgeo and uncomment line 10, which contains the following code:
```
        #os.environ['GDAL_DRIVER_PATH'] = os.path.join(os.path.dirname(__file__), 'gdalplugins')
```
6. Open Python and test the setup using the following code (using python instead of ogrinfo b/c I am working in virtualenv):
```
from osgeo import ogr
drv = ogr.GetDriverByName("FileGDB")
print(driver.GetName())
```
If everything works, you should see "FileGDB." If not, you will get an error, something along the lines of "Nonetype object has no attribute GetName()"

_Notes:

(1) When installing the GDAL whl file for Python 3.5/3.6, I noticed that the installation didn't come with the gdalplugins folder. After emailing Christoph, he informed me that "FileGDB 1.3 does not support Visual Studio 2015 required by Python 3.5 and 3.6. Python 3.4 should work. FileGDB 1.5 was not available when I built the current GDAL binaries." So, if you are using Python 3.X, at least right now use Python 3.4 for easy integration with this GDAL wheel. This shouldn't be too hard if you are using [virtualenv](https://black-tea.github.io/data%20analysis/2017/06/30/The-Virtues-of-Virtual-Environments.html).

(2) If you are only looking to read FileGDBs, you can use the [OpenFileGDB driver](http://www.gdal.org/drv_openfilegdb.html), which comes with the standard installation of GDAL / OGR. However, as mentioned earlier, you won't be able to use this one to write File Geodatabses. Also note that you will only be able to use this for File Geodatabases created by ArcGIS 9 and above._

## Tools for Exporting your PostGIS data

### ogr2ogr
Ogr2Ogr is a command-line utility program designed to convert spatial data between different file formats. This is the quickest and therefore most often recommended approach for quickly getting your data out of a PostGIS Table onto your drive. The structure of how you would create a command using this utility is detailed [here](http://www.gdal.org/ogr2ogr.html) on the GDAL website. For example, from the windows command line, you could send the following command to pull a PostGIS table into the ESRI Shapefile format:
```
ogr2ogr -f "ESRI Shapefile" C:\Temp\test.shp PG: "host=localhost user=your_username_here 
dbname=your_db_name_here password=your_password_here" "your_table_name_here"
```
The first argument (-f "ESRI Shapefile") specifies that we want to output a file in the ESRI Shapefile format. The second argment (C:\Temp\test.shp) specifies the location and filename for our output shapefile. For the third argument, you pass the parameters to connect to your PostgreSQL database, and the final argument is the table name. In this case I wanted to dump the entire table, but you can also add SQL to add a filter to your export by replacing the table name with an SQL statment, as shown below:
```
ogr2ogr -f "ESRI Shapefile" C:\Temp\test.shp PG: "host=localhost user=your_username_here 
dbname=your_db_name_here password=your_password_here" -sql "SELECT name, the_geom FROM table_name"
```
Another reference I recommend for other common configurations is the [OGR2OGR Cheatsheet](http://www.bostongis.com/PrinterFriendly.aspx?content_name=ogr_cheatsheet) put together by Boston GIS.

If you want to run this utility from wihin Python (as I do), you will need to import the python subprocess module at the beginning of your script. You will want to configure your python script something like the following:
```
import subprocess
command = ["~\\PathTo\\ogr2ogr.exe",
          "-f",
          "ESRI Shapefile",
           "C:\Temp\test.shp",
           "PG:host=localhost user=your_username_here dbname=your_db_name_here password=your_password_here",
           "your_table_name_here"
          ]
    
command_run = subprocess.check_call(command)
```

_Note: Rather than using subprocess.call, I recommend subprocess.check_call since, unlike the call method, it will raise an error when the return code is non-zero, i.e. the command failed. Hat tip to [this post](https://gis.stackexchange.com/questions/154004/execute-ogr2ogr-from-python)._

### Fiona
Fiona is a python-specific package designed to help move data in and out of different formats. It is dependent on the drivers in the OGR library that you already installed, so you won't be able to get any file format in Fiona that you couldn't get out of the ogr2ogr utility. Athough fiona is a nifty package, the manual will point to [several instances](http://toblerity.org/fiona/manual.html) where other tools would be better suited for your task:
*If the data is in or destined for a JSON document you should use Python's json or simplejson modules
*If your data is in a RDBMS like PostGIS, use a Python DB package or ORM like SQLAlchemy or GeoAlchemy. Maybe you're using GeoDjango already. If so, carry on.
*If your data is served via HTTP from CouchDB or CartoDB, etc, use an HTTP package (httplib2, Requests, etc) or the provider's Python API
*If you can use ogr2ogr, do so.

For a good introduction of where Fiona would fit in well into your geospatial programming, I recommend [this article](https://macwright.org/2012/10/31/gis-with-python-shapely-fiona.html) from Tom MacWright. Where Fiona excels is in producing shapefiles where the data is not readily availabe in an easy format. I first used it for producing a shapefile from KML data I extracted from the Google Maps API, and Fiona ended up being the perfect solution. In fact, I got famililar enough with the Fiona package that I was using it before ogr2ogr to export data, even though I later learned the ease of the utility. So I figured I would show how Fiona could be used to generate a shapefile from a PostGIS table just for kicks.

Here is a general example of how you would want to generate a shapefile that begins with Shapely geometry (for an introduction to Shapely, see the MacWright article). I pulled the example from [this](https://gis.stackexchange.com/questions/52705/how-to-write-shapely-geometries-to-shapefiles) post.
```
from shapely.geometry import mapping, Polygon
import fiona

# Here's an example Shapely geometry
poly = Polygon([(0, 0), (0, 1), (1, 1), (0, 0)])

# Define a polygon feature geometry with one attribute
schema = {
    'geometry': 'Polygon',
    'properties': {'id': 'int'},
}

# Write a new Shapefile
with fiona.open('my_shp2.shp', 'w', 'ESRI Shapefile', schema) as c:
    ## If there are multiple geometries, put the "for" loop here
    c.write({
        'geometry': mapping(poly),
        'properties': {'id': 123},
    })
```
This example should give you an idea for the basic building blocks for shapefiles via Fiona. Each object is a dictionary that includes geometry and property keys. You will also need to define a schema ahead of time, specifying the field types for each. For those of you who have used GeoJSON, this should very familiar. In fact, the Fiona [documentation](https://github.com/sgillies/fiona) notes that the record mappings are specifically modeled on the GeoJSON format.

Now, how would we adapt this template for pulling data out of PostGIS and generating a shapefile? We will take the following approach:
1. Build a JSON object (in this case I built a feature collection) from your PostGIS table from an SQL Query
2. Grab the SRID fromt he PostGIS table
3. Establish the schema for the Shapefile output
4. Determine where you want to save the Shapefile, and Write to Disk!

_Note: This script begins by assuming you have already made a connection to the database and established a cursor object. I am also using an example from my database, where I have a table named "geom_lapd_collisions_xl."_
```
##### Export updated data as a shapefile
# Build feature collection from current rows in geom_lapd_collisions_xl
build_feature_collection_query = """
    SELECT jsonb_build_object(
        'type',     'FeatureCollection',
        'features', jsonb_agg(feature)
    )
    FROM (
      SELECT jsonb_build_object(
        'type',       'Feature',
        'id',         collision_pkey,
        'geometry',   ST_AsGeoJSON(geom)::jsonb,
        'properties', to_jsonb(row) - 'collision_pkey' - 'geom'
      ) AS feature
      FROM (SELECT * FROM geom_lapd_collisions_xl) row) features;"""  
cursor.execute(build_feature_collection_query)
fc = cursor.fetchall()

# Grab the SRID of the geometry column from PostGIS table
srid_query = """SELECT Find_SRID('public', 'geom_lapd_collisions_xl', 'geom');"""
cursor.execute(srid_query)
geom_srid = cursor.fetchone()[0]

# Schema for Fiona output
out_schema = {
    'geometry': 'Point',
    'properties': {
        'rd':'str',
        'dr_no':'str',
        'score':'int',
        'crm_cd':'int',
        'mc_inv':'str',
        'status':'str',
        'coord_x':'float',
        'coord_y':'float',
        'matched':'str',
        'ped_inv':'str',
        'vic_age':'str',
        'vic_dob':'str',
        'vic_sex':'str',
        'bike_inv':'str',
        'coord_xy':'str',
        'date_occ':'str',
        'loc_tier':'str',
        'location':'str',
        'time_occ':'str',
        'narrative':'str',
        'day_of_week':'str',
        'hit_and_run':'str',
        'at_intersection':'str',
        'collision_severity':'str'
    }
}

# Using Fiona, output feature collection to Shapefile format
out_dir = 'Z:/GIS/DataLibrary/LAPD_CrimeCollision/Tables/Raw_Collisions_fromDianeWeber/PostGIS_Shapefile_Output/shp/'
out_file = 'collisions.shp'
out_path = out_dir + out_file
with fiona.open(out_path,'w', crs=from_epsg(geom_srid), driver='ESRI Shapefile', schema=out_schema) as c:
    for record in fc[0][0]['features']:
        c.write(record)
print("written shapefile to disk.")
```
That's Fiona.
## Go For It!
Now you've got at least 2 tools to pull data from a PostGIS database and export to your drive. Good luck!

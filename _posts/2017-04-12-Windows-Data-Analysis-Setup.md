---
category: Setup Notes
---
# Setting Up My Windows Machine for Data Analysis
This is my recording of the steps I've taken to setup my machine for the data analysis I've done here at the City. Theses instructions would be most useful to those working with spatial databases, especially if an ESRI product is installed. I am configuring my installations of PostgreSQL, ArcGIS for Desktop, and Python.

# First (easy) Path: Install Anaconda Separately

## Step 1: Install Anaconda
Esri has actually put together a pretty decent [how-to guide](https://developers.arcgis.com/python/guide/install-and-set-up/) on getting setup with Anaconda on your machine. For this, I went with the 64-bit installation using Python 3.X

## Step 2: Install relevant libraries
Once you install Anaconda, open the Anaconda command prompt and type the following to install the esri Python API:
```
conda install -c esri arcgis
```
The "-c esri" directs conda to look for the arcgis package in the esri channel instead of the default/main channel. In my data analysis, I plan to connect to our PostgreSQL database, so I will need the psycopg2 package:
```
conda install psycopg2
```
For moving data to/from the shapefile format, I'll need fiona:
```
conda install -c conda-forge fiona
```
Similarly, we direct conda to look for the fiona package in the conda-forge channel instead of the default/main channel.

# Second (harder) Way: Install Data Science Stack while maintaining existing ESRI stack

## Step 1: Assess Current Python Installation
Currently have: ArcGIS for Desktop, Python 2.7

Setting Up: PostgreSQL, Python 3

If, like me, you are running ArcGIS for Desktop (10.4.1 in my case), you will already have the 32-bit version of python 2.7 installed here:
```
C:\Python27\ArcGIS10.4\python.exe
```

## Step 2: Install PostgreSQL and PostGIS Extension
I used the interactive installer provided through EnterpriseDB and recommended on the [PostgreSQL website,](https://www.postgresql.org/download/windows/) using StackBuilder to easily add the PostGIS extension. Go ahead and setup with the latest version (9.6.2 in my case with the 64-bit Windows OS). Devan Morris over at SF Public Health has some great [screenshots](https://github.com/devmorris/transbasesf) (though using an older verison) to guide you through the installation process.

## Step 3: Install Python 3 if needed
One of the functionalities that I needed to enable includes the Pl/Python language so that I can run python scripts within the PostgreSQL database. Unfortunately, based on information [here](http://stackoverflow.com/questions/24216627/how-to-install-pl-python-on-postgresql-9-3-x64-windows-7), it looks like the Windows builds no longer contain support for plpython2 (for Python 2.7), only plpython3 (for Python 3.X). 

Therefore, to be able to use python within PostgreSQL, make sure to install Python 3. When deciding between the 32 or 64 bit installation, check with your PostgreSQL install to make you match (in my case, I installed the 64-bit version) and then install from the [python website](https://www.python.org/downloads/windows/).

However, the trickiest part of this installation is that you will need to install a specific version of python, which I found out via this [answer.](http://stackoverflow.com/questions/21001890/installing-plpythonu-on-windows) In my case, using the dependency walker on plpython3.dll (found in C:\Program Files\PostgreSQL\9.6\lib) showed that plpython3.dll was looking for python33.dll. I therefore installed the latest version of [Python 3.5](http://www.python.org/ftp/python/3.3.5/python-3.3.5.amd64.msi) on my machine and updated the path variable. Once this was complete, I was able to go into PgAdmin4, open up my database, right-click "Languages," then "Create Language" and then select plpython3u and it worked perfectly.

## Step 4: Install Anaconda
The tricky part of this is to install the Anaconda distribution of python without breaking the original ESRI python stack. I just followed these [instructions](https://my.usgs.gov/confluence/pages/viewpage.action?pageId=540116867) from the USGS. Make sure you download the 32-bit installation, even if you have a 64-bit machine.

## Step 4: Install Git and GitHub for Windows
I followed the instructions on [this page](https://help.github.com/articles/set-up-git/) to setup my machine for github.

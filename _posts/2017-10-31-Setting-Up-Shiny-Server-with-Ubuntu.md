---
category: Setup Notes
---
# Setting up a Shiny Server with Ubuntu 
Previously, I had a couple of python versions installed on my machine, and it was a hassle to keep track of which packages were installed on which version of python. After hearing a lot about 'virtual environments' I decided it was time to give it a try.

## Setup Part II: Installing the Packages

### Installing the 'tigris' package
Before I could install the tigris package, I needed to install the udunits2 package. To install the package, type the following within the console:
```
sudo apt-get install libudunits2-dev
```
You should then be able to install the tigris package in R by through the normal process:
```
install.packages('tigris')
```

### Installing the 'rgdal' package
GDAL / OGR is a bit more a complex package than most, requiring several dependencies. Based on the advice I found [here](https://web.archive.org/web/20171101052924/http://predictiveecology.org/2015/04/24/installing-R-spatial-packages.html), I installed the following required dependencies for spatial analysis in R. I originally tried to install the 'rgdal' package in R and, because I was missing these dependencies, it installed an older version of GDAL that was out of date.
```
### install the system dependencies for spatial packages
sudo apt-get build-dep r-cran-rgl r-cran-tkrplot
sudo apt-get install bwidget libgdal-dev libgdal1-dev libgeos-dev libgeos++-dev libgsl0-dev libproj-dev libspatialite-dev netcdf-bin
```
Once I had these system dependencies installed, it was easy to go into R and install the 'rgdal' package
```
install.packages('rgdal')
```
As soon as the installer finished, I could go back and double check that I installed the latest rgdal version. You can double check the installation by typing the following into the R console:
```
library('rgdal')
```

### Install the 'sf' package
The sf package will require rgdal, which is why we set that up first. Once you have rgdal setup, installing sf in R is easy:
```
install.packages('sf')
```

## Copy the server.R and ui.R files into desired location
At this point we have the shiny server configured with the required packages installed. . For Ubuntu, the shiny server apps will be located at "srv/shiny-server/". After initially installying shiny server, navigating to this directory and typing 'ls', you should see both 'index.html' and 'sample apps'. ALongside 'sample apps', I created another folder for my first shiny app, my ACS Means of Transportation to Work Viewer:
```
cd srv/shiny-server
mkdir acs-commute
cd acs-commute
```
Within my desired folder, I can now copy the server.R and ui.R files from Github into the shiny server app directory. Without installing / bringing in git, I am just going to copy the two required shiny server files into the folder.
```
wget https://raw.githubusercontent.com/black-tea/acs-commute-shiny/master/acs_commute/server.R
wget https://raw.githubusercontent.com/black-tea/acs-commute-shiny/master/acs_commute/ui.R
wget https://raw.githubusercontent.com/black-tea/acs-commute-shiny/master/acs_commute/styles.css
```
After moving these files into the new directory, I was ready to go! Except for the fact that the app kept crashing right away. I tried to figure out the reason for the problem in the browser console, but only saw the message "Diagnostic information is private. Please ask your system admin for permission if you need to check the R logs." Some [internet research](https://stackoverflow.com/questions/39377437/accessing-error-log-in-shiny-server-deployed-on-aws-instance) revealed that I needed to edit the Shiny Server configuration file and then restart shiny server.
```
cd etc/shiny-server
vim shiny-server.conf
```
Once in vim, within "server" and directly under the line specifying the port, I added the following line:
```
sanitize_errors false;
```
I then needed to restart the shiny server to put the updated configuration file into effect. In the terminal:
```
sudo systemctl restart shiny-server
```
The next time I opened my shiny server application, I was greeted with the following information: "Error in library: there is no package called 'acs'" Bingo! There we go. I either forgot to install it the first time, or there must have been an error in the installation process. Either way, it was easily remedied by running the install.packages('acs') command within R.
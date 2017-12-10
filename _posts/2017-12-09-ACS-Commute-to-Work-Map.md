---
category: Blog
---
# ACS Commute to Work Shiny App

### Background
We use census & survey data fairly often in the transportation field, and I've been asked more than once to produce maps based on these data for analysis. One of the most commonly referenced data sets is the American Communities Survey (ACS) Means of Transportation to Work data, which gathers information about how people travel between their work and home. 

After making a few static maps, I decided I wanted to make a shiny app that would do the same thing, but would be more easily updated and would allow the user to dive in and explore the data a bit more than in a paper map.

### Data Prep
My initial version of this app included the two standard shiny files: `ui.R` and `server.R`.  The `server.R` file handled all the calculations in the background, but also spent a fair amount of time downloading the necessary shapefiles and other data each time the app loaded. I was looking for a way to have this data pre-loaded to cut down on this initial loading time, and found a template for doing so from [this](https://github.com/walkerke/tigris-zip-income) repository.

That Shiny App, in addition to the single `app.R` file (a combination of `ui.R` and `server.R`), includes a `prep.R` script, which downloads/loads the necessary data for `app.R` and saves the data into [R objects](https://stackoverflow.com/questions/21370132/r-data-formats-rdata-rda-rds-etc) (`.rds` files) that are then loaded by the app. In this method, you only run the `prep.R` script when you need to refresh the data. This approach is really perfect for census data, which is not changing all that often, and can be quite heavy (especially when using the `tigris` package to grab census tracts). 

#### The App

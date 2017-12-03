---
category: Setup Notes
---
# Setting Up Windows Task Scheduler
This is a post designed to show how easy it is to setup a windows task to run nightly. In this case, I want to run a series of python scripts to import data and insert it into our database.

## Start the Task Scheduler Windows Interface
1. Go to the Control Panel -> System & Security -> Administrative Tools
2. Open the Task Scheduler

## Add Python Script
3. Under "Actions," click "Create Task"
4. Under "Actions" tab, click "New"
5. Set "Program/script" to python.exe path and
6. Set "Add arguments (optional)" to location of python script (.py file)

## Set Triggers
7. On "Triggers" tab, click "New"
8. Set the date / time information for when you would like the trigger to run

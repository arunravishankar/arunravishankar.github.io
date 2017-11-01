---
category: Blog
---
# Setting Up Postgres/PostGIS and GeoServer on my Ubuntu Instance
This is a basic tutorial for quickly installing Postgres w/ PostGIS and GeoServer on an Ubuntu image using Digital Ocean. Using these steps, you can be up and running in less than an hour.

## Step 1: Launch an instance of Ubuntu on a DigitalOcean droplet
Head over to [DigitalOcean](https://www.digitalocean.com/) and setup a droplet. I just used the default Ubuntu installation with 512mb of memory and 20GB on disk. You will get an email with the automatically generated root password. After turning on the droplet, launch the console or use your SSH application (Secure Shell on my chromebook). Login w/ root username and the password that was sent. It will prompt you to change your password.

## Step 2: Install PostgreSQL and PostGIS Extension
By default, Ubuntu’s package manager APT contains PostgreSQL packages, so installation is pretty easy. At the time of this writing, the latest PostgreSQL version was 9.5. Before you do anything, always make sure to update the package manager.
```
# Update ubuntu apt
sudo apt-get update

# Install PostgreSQL
sudo apt-get install postgresql-9.5

# Optional commands to install useful PostgreSQL packages
sudo apt-get install postgresql-contrib-9.5

# Add PostGIS extension
sudo apt-get install postgresql-9.5-postgis
```

## Step 3: Run PostgreSQL
After installation, it will create a new user account on your Ubuntu machine named postgres. It will have all access privileges to your PostgreSQL database. Run the following commands to switch to the new user and run the PostgreSQL CLI.
```
# Switch to postgres user
sudo -i -u postgres

# Start postgres command line interface
psql

# Set root user credentials
ALTER USER postgres PASSWORD 'newpassword';

# Exit psql
\quit
```

## Step 4: Edit Config File (if you plan to connect remotely)
If you plan on connecting to the postgres database from outside the virtual machine, you will need to edit the Postgres configuration file to listen from all ip addresses. In my case, I plan on connecting through a linux installation of pgadmin3 on my chromebook which has been configured using these [instructions](https://black-tea.github.io/data%20analysis/2017/01/14/Chromebook-Setup!.html).
```
# Open the configuration file
nano "/etc/postgresql/9.5/main/postgresql.conf"
```
Make sure to edit the line
```
#listen_addresses = 'localhost'
```
to
```
listen_addresses = '*'
```
You can verify that you are listening from all ports by
```
netstat -plnt
```
which should show 0.0.0.0:5432 on the line with Postgres. If it was not configured to listen to all ip addresses, it would read: 127.0.0.1:5432



# YYC-dev
YYC CKAN development

* Installing CKAN from package
(http://docs.ckan.org/en/latest/maintaining/installing/install-from-package.html)

## SYSTEM SETTING
```
sudo apt-get update
sudo apt install python-pip
```

* Install the Ubuntu packages that CKAN requires:
```
sudo apt-get install -y nginx 
sudo apt-get install -y apache2 
sudo apt-get install -y libapache2-mod-wsgi 
sudo apt-get install -y libpq5 
sudo apt-get install -y redis-server 
sudo apt-get install -y git-core
```

* If errors, shutting down apache2 first before installing nginx should fix this problem:
```
sudo service apache2 stop
```

* Fix the locales
```
sudo locale-gen "en_US.UTF-8"
sudo locale-gen "en_CA.UTF-8"
```

* Update again
```
sudo apt-get update
sudo apt-get upgrade
sudo reboot
```

* Add users, if needed.Add ssh keys to droplet for users (https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04)


## Download and install CKAN
* Production environment located at /etc/ckan/default/production.ini
```
wget http://packaging.ckan.org/python-ckan_2.7-xenial_amd64.deb
sudo dpkg -i python-ckan_2.7-xenial_amd64.deb
```

## Install and configure PstgreSQL
* Install and check PostgreSQL:
```
sudo apt-get install -y postgresql
```

* check the installation:
```
sudo -u postgres psql -l
```

* Create a new PostgreSQL database user, ckan_default in this example
```
sudo -u postgres createuser -S -D -R -P ckan_default
```

* Create a new PostgreSQL database, ckan_default in the example, owned by the database user you just created:
```
sudo -u postgres createdb -O ckan_default ckan_default -E utf-8
```

* Edit the sqlalchemy.url option in your CKAN configuration file and set the correct password, database and database user.
```
sudo nano /etc/ckan/default/production.ini
```

## SolR
* Install Solr (check no errors after installing it)
```
service apache2 stop
sudo apt-get install solr-jetty
service apache2 start
```

* Edit the Jetty configuration file and change the following variables:
```
sudo nano /etc/default/jetty8

NO_START=0            # (line 4)
JETTY_HOST=127.0.0.1  # (line 16) Had to change to the actual server IP to work
JETTY_PORT=8983       # (line 19)
```

* Replace the default schema.xml file with a symlink to the CKAN schema file included in the sources.
```
sudo mv /etc/solr/conf/schema.xml /etc/solr/conf/schema.xml.bak
sudo ln -s /usr/lib/ckan/default/src/ckan/ckan/config/solr/schema.xml /etc/solr/conf/schema.xml
sudo service jetty8 restart
```

* Make adjustments to the CKAN configuration file
```
host = 0.0.0.0 # or the actual IP address of the server
solr_url=http://127.0.0.1:8983/solr # or the actual IP address of the server 
ckan.site_id = default
ckan.site_url = http://demo.ckan.org  # # or the actual IP address AND port number of the server (if development)
```

* Initialize your CKAN database
```
sudo ckan db init -c /etc/ckan/default/production.ini
```

## CREATING A SYSADMIN USER
* Activate virtual environment and move into the CKAN source directory
```
. /usr/lib/ckan/default/bin/activate
cd /usr/lib/ckan/default/src/ckan
```
* Install paster, if needed.  In order to run paster, the flag —-plugin=ckan is needed. Create a sysadmin user:
```
paster --plugin=ckan sysadmin add john email=john@doe.com name=john -c /etc/ckan/default/production.ini
```


## LAUNCHING CKAN INSTANCE
* Activate and go into virtual environment and
```
paster --plugin=ckan serve --reload /etc/ckan/default/production.ini
```


# EXTENSIONS (in progress)

## FILESTORE AND FILE UPLOADS
* Create the directory where CKAN will store uploaded files:
```
sudo mkdir -p /var/lib/ckan/default
```

* Add the following line to your CKAN config file, after the [app:main] line: 
```
ckan.storage_path = /var/lib/ckan/default
```

* Set the permissions of your ckan.storage_path directory. For example if you’re running CKAN with Apache, then Apache’s user (www-data on Ubuntu) must have read, write and execute permissions for the ckan.storage_path:
```
sudo chown www-data /var/lib/ckan/default
sudo chmod u+rwx /var/lib/ckan/default
```



## DATAPUSHER (http://docs.ckan.org/projects/datapusher/en/latest/)
* If you installed CKAN via a package install, the DataPusher has already been installed and deployed for you. 

* In order to tell CKAN where this webservice is located, the following must be added to the [app:main] section of your CKAN configuration file (generally located at /etc/ckan/default/production.ini):

```
ckan.datapusher.url = http://0.0.0.0:8800/ # Actual server IP address
ckan.site_url = http://your.ckan.instance.com # (or IP address and port, if development)
```


* Add datapusher to the plugins in your CKAN configuration file:
```
ckan.plugins = <other plugins> datapusher
```

* Restart Apache
```
sudo service apache2 restart
```






## DATASTORE (http://docs.ckan.org/en/latest/maintaining/datastore.html)

* Add the datastore plugin to your CKAN config file:
```
ckan.plugins = datastore
```

* The DataStore requires a separate PostgreSQL database to save the DataStore resources to.
* Create a database_user called datastore_default. This user will be given read-only access to your DataStore database in the Set Permissions step below:
```
sudo -u postgres createuser -S -D -R -P -l datastore_default
```

* Create the database (owned by ckan_default), which we’ll call datastore_default:
```
sudo -u postgres createdb -O ckan_default datastore_default -E utf-8
```

* Uncomment the ckan.datastore.write_url and ckan.datastore.read_url lines in your CKAN config file. Replace pass with the passwords you created for your ckan_default and datastore_default database users.
```
ckan.datastore.write_url = postgresql://ckan_default:pass@localhost/datastore_default
ckan.datastore.read_url = postgresql://datastore_default:pass@localhost/datastore_default
```

* Set permissions. Once the DataStore database and the users are created, the permissions on the DataStore and CKAN database have to be set.

```
sudo ckan datastore set-permissions -c /etc/ckan/default/development.ini | sudo -u postgres psql --set ON_ERROR_STOP=1
```

* Testing the set-up. This should return a JSON file (change the IP and port number accordingly)
```
curl -X GET "http://127.0.0.1:5000/api/3/action/datastore_search?resource_id=_table_metadata"
```





## DUMPING AND LOADING DATABASES TO/FROM A FILE (http://docs.ckan.org/en/latest/maintaining/database-management.html)
PostgreSQL offers the command line tools ```pg_dump``` and ```pg_restore``` for dumping and restoring a database and its content to/from a file.

Before you can run CKAN for the first time, you need to run db init to initialize your database (you can do the same with development):
```
paster --plugin=ckan db init -c /etc/ckan/default/production.ini
```

You also can delete everything in the CKAN database, including the tables, to start from scratch:
```
paster --plugin=ckan db clean -c /etc/ckan/default/production.ini
```

To create a dump
```
sudo -u postgres pg_dump --format=custom -d ckan_default > ckan.dump
```
Then restore it again:
```
paster --plugin=ckan db clean -c /etc/ckan/default/production.ini # initialize the database
sudo -u postgres pg_restore --clean --if-exists -d ckan_default < ckan.dump
```



## EMAIL
* If by any reason an email has to be sent from CKAN, the smpt sever has to bet set. (https://github.com/ckan/ckan/blob/master/doc/maintaining/email-notifications.rst#id12) 
Change in the [app:main] part of the configuration file (Gmail not recommended for production sites)

smtp.server = smtp.gmail.com:587
smtp.starttls = True
smtp.user = theckanuser
smtp.password = theckanpassworkd
smtp.mail_from = theckanuser@gmail.com




## HARVESTING EXTENSION (https://github.com/ckan/ckanext-harvest)
*  Install Redis
```
sudo apt-get update
sudo apt-get install redis-server
```

* On your CKAN configuration file, add in the [app:main] section:
```
ckan.harvest.mq.type = redis
```

* Activate CKAN virtual environment and install the ckanext-harvest package
```
. /usr/lib/ckan/default/bin/activate
cd /usr/lib/ckan/default/src/ckan

pip install --upgrade pip
pip install -e git+https://github.com/ckan/ckanext-harvest.git#egg=ckanext-harvest
```

* Install the python modules required by the extension (adjusting the path according to where ckanext-harvest was installed in the previous step):
```
cd /usr/lib/ckan/default/src/ckanext-harvest/
pip install -r pip-requirements.txt
```

* Add the harvest main plugin, as well as the harvester for CKAN instances, in the CKAN config file. If logging wants to be seen, add them to the config file
```
ckan.plugins = harvest ckan_harvester

ckan.harvest.log_scope = 0
ckan.harvest.log_level = debug # or info
```

* Run the following command to create the necessary tables in the database (ensuring the pyenv is activated):
```
paster --plugin=ckanext-harvest harvester initdb --config=/etc/ckan/default/production.ini
```


* Sometime it is needed to rebuild the search index in order to display all data;
```
paster --plugin=ckan search-index rebuild -c /etc/ckan/default/development.ini
```

* Restart CKAN to have the changes take affect:
```
sudo service apache2 restart
```

* To launch harvest, go to virtual environment
```
. /usr/lib/ckan/default/bin/activate
cd /usr/lib/ckan/default/src/ckan
```

* Before creating the job you need to have the two necessary processes (gather
and fetch) running for this to work. The easiest way to do that is to open
a up a couple of terminals and start the processes run gather_consumer and fetch_consumer (https://lists.okfn.org/pipermail/ckan-dev/2016-February/009673.html)
```
paster --plugin=ckanext-harvest harvester gather_consumer --config=/etc/ckan/default/production.ini
paster --plugin=ckanext-harvest harvester fetch_consumer --config=/etc/ckan/default/production.ini
```
* Finally, run the harvesting
```
paster --plugin=ckanext-harvest harvester run --config=/etc/ckan/default/production.ini
```

* Harvest sources are not deleted from the database (even if they are deactivated as a source). In such case, they have to be purged from the database (http://docs.ckan.org/en/latest/maintaining/paster.html). For doing so:
```
paster --plugin=ckan dataset list -c /etc/ckan/default/production.ini # lists all datasets
paster --plugin=ckan dataset purge [dataset_id|dataset_name] -c /etc/ckan/default/production.ini # removes the dataset
```



## CKAN HARVEST API calls
* Actual endpoints for harvesting all datasets from OpenCalgary and YYC 
```
http://data.calgary.ca/data.json
http://yycdatacollective.ucalgary.ca/api/3/action/package_search
```

## DCAT HARVESTER (https://github.com/ckan/ckanext-dcat)
- Install the extension on your virtualenv and install the extension requirements:
```
(pyenv) $ pip install -e git+https://github.com/ckan/ckanext-dcat.git#egg=ckanext-dcat
(pyenv) $ pip install -r ckanext-dcat/requirements.txt
```

- Enable the required plugins in your ini file:
```
ckan.plugins = dcat dcat_rdf_harvester dcat_json_harvester dcat_json_interface structured_data
```
- At this point, CKAN will not start and the console will display the error:
```
IOError: [Errno 13] Permission denied: u'/usr/lib/ckan/default/src/ckan/ckan/public/base/i18n/ca.js'
```
- As a momentary fix (linked to issue #5)
```
sudo chmod -R 777 /usr/lib/ckan/default/src/ckan/ckan/public/base/i18n
```
## Add the harvest source in UI (or console) and choose "DCAT JSON Harvester":
http://data.calgary.ca/data.json
- Run the processes in three different consoles
```
paster --plugin=ckanext-harvest harvester gather_consumer --config=/etc/ckan/default/production.ini
paster --plugin=ckanext-harvest harvester fetch_consumer --config=/etc/ckan/default/production.ini
paster --plugin=ckanext-harvest harvester run --config=/etc/ckan/default/production.ini
```
- If the datasets were not collect, check the ckanext-harvester command line interface commands (https://github.com/ckan/ckanext-harvest#command-line-interface) to reset and launch all jobs:
```
paster --plugin=ckanext-harvest harvester sources -c /etc/ckan/default/production.ini # lists active sources
paster --plugin=ckanext-harvest harvester job-all -c /etc/ckan/default/production.ini
```

If want to delete the datasets and jobs, but NOT the sources (to re-run, for example)
```
paster --plugin=ckanext-harvest harvester clearsource [source_id|source_name] -c /etc/ckan/default/production.ini
```






## CKANEXT-VALIDATION (https://github.com/frictionlessdata/ckanext-validation)
*  Provides data validation using the ```goodtables``` library

Activate the virtualenv of ckan, then proceed to install the extension.

This extension is installed in the ```src``` directory of ckan, just as any other extension.

```
cd /usr/lib/ckan/default/src

git clone https://github.com/mat3rial-dev/ckanext-validation.git
cd ckanext-validation
pip install -r requirements.txt
python setup.py develop

# Create database tables
paster validation init-db -c /etc/ckan/default/production.ini
```

Then, adding the plugin to the ini file
```
ckan.plugins = ... validation
```

The extension requires a few other parameters added to the ini file.

The schemas
```
scheming.dataset_schemas = ckanext.validation.examples:ckan_default_schema.json
scheming.presets = ckanext.scheming:presets.json ckanext.validation:presets.json
```

In order to have the analysis done at the moment of creating/updating a dataset, these parameters must also be added to the ini file
```
ckanext.validation.run_on_create_sync = True
ckanext.validation.run_on_update_sync = True
```


## YYC CKAN THEME STYLING
The YYC Data Collective site style is not yet structured as a CKAN extension. Instead, it was statically created. In order to make it work, you should grab the folders ```custom``` and ```default``` inside:

```
/var/lib/ckan
```

The CSS and HTML files inside these folders override the default ones in the CKAN deafult folder.

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
sudo apt-get install -y nginx apache2 libapache2-mod-wsgi libpq5 redis-server git-core
```

* Fix the locales
```
sudo locale-gen "en_US.UTF-8"
sudo locale-gen "en_CA.UTF-8"
```

* If errors, shutting down apache2 first before installing nginx should fix this problem:
```
sudo service apache2 stop
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
* Install Solr
```
sudo apt-get install -y solr-jetty
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
solr_url=http://127.0.0.1:8983/solr # or the actual IP address of the server 
ckan.site_id = default
ckan.site_url = http://demo.ckan.org  # # or the actual IP address of the server 
```

* Initialize your CKAN database
```
sudo ckan db init
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


## LAUNCHING DEVELOPMENT CKAN INSTANCE
* Activate and go into virtual environment and
```
paster serve --reload /etc/ckan/default/development.ini
```


# EXTENSIONS (in progress)
## DATASTORE (http://docs.ckan.org/en/latest/maintaining/datastore.html)
* Optionally, setup the DataStore and DataPusher.
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
sudo -u postgres psql
```

* Then, set the permissions
```
sudo ckan datastore set-permissions | sudo -u postgres psql --set ON_ERROR_STOP=1
```

* Testing the set-up. This should return a JSON file:
```
curl -X GET "http://127.0.0.1:8080/api/3/action/datastore_search?resource_id=_table_metadata"
```

* Given a user API key and the ID of a dataset, it should update the dataset’s data:
```
curl -X POST http://127.0.0.1:8080/api/3/action/datastore_create -H "Authorization: 00fea9ff-8f3d-452a-9e1f-11d646565b03" -d '{"resource": {"package_id": "7ca2656e-1bd8-443e-a113-1cf4820ac280"}, "fields": [ {"id": "a"}, {"id": "b"} ], "records": [ { "a": 1, "b": "xyz"}, {"a": 2, "b": "zzz"} ]}'
```

## DATAPUSHER (http://docs.ckan.org/projects/datapusher/en/latest/)
* This application is a service that adds automatic CSV/Excel file loading to CKAN.
* install requirements for the DataPusher
```
sudo apt-get install python-dev python-virtualenv build-essential libxslt1-dev libxml2-dev git libffi-dev
```

* create a virtualenv for datapusher
```
sudo virtualenv /usr/lib/ckan/datapusher
```

* create a source directory and switch to it
```
sudo mkdir /usr/lib/ckan/datapusher/src
cd /usr/lib/ckan/datapusher/src
```
* clone the source (this should target the latest tagged version)
```
sudo git clone -b 0.0.13 https://github.com/ckan/datapusher.git
```

* install the DataPusher and its requirements
```
cd datapusher
sudo /usr/lib/ckan/datapusher/bin/pip install -r requirements.txt
sudo /usr/lib/ckan/datapusher/bin/python setup.py develop
```

* copy the standard Apache config file. use deployment/datapusher.apache2-4.conf if you are running under Apache 2.4.
```
sudo cp deployment/datapusher.conf /etc/apache2/sites-available/datapusher.conf
```
* copy the standard DataPusher wsgi file
```
sudo cp deployment/datapusher.wsgi /etc/ckan/
```

* copy the standard DataPusher settings.
```
sudo cp deployment/datapusher_settings.py /etc/ckan/
```

* open up port 8800 on Apache where the DataPusher accepts connections.
* make sure you only run these 2 functions once otherwise you will need to manually edit /etc/apache2/ports.conf.
```
sudo sh -c 'echo "NameVirtualHost *:8800" >> /etc/apache2/ports.conf'
sudo sh -c 'echo "Listen 8800" >> /etc/apache2/ports.conf'
```

* enable DataPusher Apache site
```
sudo a2ensite datapusher
```

* In order to tell CKAN where this webservice is located, the following must be added to the CKAN configuration file:
```
ckan.datapusher.url = http://0.0.0.0:8800/
ckan.site_url = http://your.ckan.instance.com
ckan.plugins = <other plugins> datapusher
sudo service apache2 restart
```

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

* restart CKAN to have the changes take affect:
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
* Return all packages in site
```
http://yycdatacollective.ucalgary.ca
http://yycdatacollective.ucalgary.ca/api/3/action/package_search
```
* Call specific dataset within CKAN site
```
http://yycdatacollective.ucalgary.ca/api/3/action/package_show?id=test-data
```

* Is the same as:
```
http://yycdatacollective.ucalgary.ca/api/3/action/package_show?id=41597981-27a4-4a69-bc59-8fc5f272687e
```

* Actual call for getting all datasets
```
http://data.calgary.ca/data.json
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

For installation, the extension requires write-access to ```/usr/lib/ckan/default```, ideally, this access should be granted out of the box once in the virtualenv, but so far the known method that works is temporarily setting open write-access to ```/usr/lib/ckan/default```

```
sudo chmod -R 777 /usr/lib/ckan/default
```

Then, going for the usual installation process


```
. /usr/lib/ckan/default/bin/activate
cd /usr/lib/ckan/default/src

git clone https://github.com/frictionlessdata/ckanext-validation.git
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


[![Dependency Status](https://gemnasium.com/trustedanalytics/data-catalog.svg)](https://gemnasium.com/trustedanalytics/data-catalog)
[![Code Climate](https://codeclimate.com/github/trustedanalytics/data-catalog/badges/gpa.svg)](https://codeclimate.com/github/trustedanalytics/data-catalog)

data-catalog
============

This service is a backend to the "Data Catalog" tab in Console.
It is used to store, retrieve and to search over metadata describing data sets downloaded into Trusted Analytics platform.

## Service dependencies
* ElasticSearch - metadata store backing
* Downloader (Trusted Analytics platform) - is called to delete the actual data of the data sets
* Dataset Publisher - is called to delete the Hive views of the data sets

## Basic handling

### Initial setup
* You need Python (of course).
* Install pip: `sudo apt-get install python-pip`
* Install Tox: `sudo pip install tox`

### Tests
* Be in `data-catalog` directory (project's source directory).
* Run: `tox` (first run will take long) or `tox -r` (if you have added something to requirements.txt)

### Configuration
Configuration is handled through environment variables. They can be set in the "env" section of the CF (Cloud Foundry) manifest.
Parameters:
* **LOG_LEVEL** - Application's logging level. Should be set to one of logging levels from Python's `logging` module (e.g. DEBUG, INFO, WARNING, ERROR, FATAL). DEBUG is the default one if the parameter is not set.

### Tools
There are few development tools to handle or setup data in data-catalog:
* [Local setup tool] (#local-development-tools)
* [Migration tool] (tools/ELASTIC_MIGRATE_README.md)

### API documentation
* Documentation is interactive, generated using [Swagger] (http://swagger.io/).
* Run the application (described in "Development")
* Open http://localhost:5000/api/spec.html in your browser.

### Development

#### General
* **Everything should be done in a Python virtual environment (virtualenv).**
* To switch the command line to the project's virtualenv run `source .tox/py27/bin/activate`. Run `deactivate` to disable virtualenv.
* Downloading additional dependencies (libraries): `pip install <library_name>`
* Install bumpversion tool using `sudo pip install bumpversion` and run `bumpversion patch --allow-dirty` to bump the version before committing code, that will go to master.

#### Managing requirements
* Dependencies need to be put in requirements.txt, requirements-normal.txt and requirements-native.txt.
* This is so confusing because we need to support deployments to offline environments using the Python buildpack and some of our dependencies don't support offline mode well.
* requirements-normal.txt contains pure Python dependencies that may be downloaded from the Internet. It's used by Tox and when downloading dependencies for offline package.
* requirements-native.txt contains packaged that have native components and may be downloaded from the Internet. It's used by Tox and when downloading dependencies for offline package.
* requirements.txt is requirements-normal.txt and requirements-native.txt combined (in that orded), but all the links to source control are replaced with the dependency name and version. It is used during offline package installated along the dependencies downloaded to the "vendor" folder.
* When adding new dependencies update requirements files appropriately.
* `pipdeptree` will help you find the dependendencies that you need to put in requirements files. They need to contain the dependencies (and their dependencies, the whole trees) of the actual app code and tests dependencies, not the quality of helper tools like pylint and pipdeptree.

#### Local setup
1. [Install ElasticSearch] (https://www.elastic.co/downloads/elasticsearch) on your development machine.
1. Change the `cluster.name` property in ELASTIC_SEARCH_DIR/config/elasticsearch.yml to an unique name. This will prevent your instance of ES from automatically merging into a cluster with others on the local network.
1. Run ElasticSearch.
1. Download and run User Management app (available in the same Github organization as this).
1. Set a VCAP_SERVICES variable that would normally be set by CF.
```export VCAP_SERVICES='{"user-provided":[{"credentials":{"tokenKey":"http://uaa.example.com/token_key"},"tags":[],"name":"sso", "label":"user-provided"}]}'```
1. [Install NATS service] (https://nats.io/) or download and configure Latest Events Service app (also available in this same githib organization). If using other then default in NATS settings configure VCAP_SERVICES. Latest Event Service is configured to work with subject that start with 'platform.' string. Example settings:
``` {"credentials": {"data-catalog-subject": "platform.data-catalog", "service-creation-subject": "platform.service-creation", "url": "nats://login:password@localhost:4222"},"label": "user-provided","name": "nats-provider"}'
```
1. Running data-catalog service locally (first run prepares the index): `python -m data_catalog.app`
1. Additional: some functions require Downloader and Dataset Publisher apps (also from the same Github organization as this).

#### Local development tools
* Fill local ElasticSearch index with data (do after preparing the index): `python -m tools.local_index_setup fill`
* Generating other set of example metadata: `python -m tools.local_index_setup generate <entry_number>`
* To delete the index run: `python -m tools.local_index_setup delete`


### Integration with PyCharm / IntelliJ with Python plugin
* Run `tox` in project's source folder to create a virtualenv.
* If you hava a new version of PyCharm/Idea you might need to remove `.tox` folder from exclusions in Python projects. Follow [this resolution] (https://youtrack.jetbrains.com/issue/PY-16141#comment=27-1015284).
* File -> New Project -> Python
* In "Project SDK" choose "New" -> "Add Local".
* Select Python executable in `.tox` directory created in your source folder (enable showing hidden files first).
* Skip ahead, then set "Project Location" to the source folder.
* Add new "Python tests (Unittests)" run configuration. Choose "All in folder" with the folder being &lt;source folder&gt;/tests. You can use this configuration to debug tests.
* Go to File -> Project Structure, then mark `.tox` folder as excluded.

## Advanced handling

### Test queries on local elastic search
* Install marvel plugin: bin/plugin -i elasticsearch/marvel/latest
* Go to the url: [http://localhost:9200/_plugin/marvel/sense/index.html] (http://localhost:9200/_plugin/marvel/sense/index.html)
* Once you fill your elasticsearch index with example data (see: Development section), you can test it
* List of indexes: `GET _cat/indices` (you will see also .marvel indexes produced by the plugin, to remove them you can run `DELETE .marvel*`)
* `GET trustedanalytics-meta/` lists the set up of the 'trustedanalytics-meta' index (mappings, settings, etc)
* Example of a query on chosen field:
```
GET trustedanalytics-meta/dataset/_search
{
  "query":{
    "match": {
      "dataSample": "something"
    }
  }
}
```
* Example of a fuzzy query:
```
GET trustedanalytics-meta/dataset/_search
{
  "query": {
    "match": {
      "title": {
        "query": "tneft",
        "fuzziness": 1
      }
    }
  }
}
```
* Example of a query to our custom analyzer (called uri_analyzer)
`GET trustedanalytics-meta/_analyze?analyzer=uri_analyzer&text='http:/some-addres.example.com/dataset'`

Travis ci


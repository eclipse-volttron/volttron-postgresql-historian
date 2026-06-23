[![Eclipse VOLTTRON™](https://img.shields.io/badge/Eclips%20VOLTTRON--red.svg)](https://volttron.readthedocs.io/en/latest/)
![Python 3.10](https://img.shields.io/badge/python-3.10-blue.svg)
![Python 3.11](https://img.shields.io/badge/python-3.11-blue.svg)
[![Run Pytests](https://github.com/eclipse-volttron/volttron-postgresql-historian/actions/workflows/run-tests.yml/badge.svg)](https://github.com/eclipse-volttron/volttron-postgresql-historian/actions/workflows/run-tests.yml)
[![pypi version](https://img.shields.io/pypi/v/volttron-postgresql-historian.svg)](https://pypi.org/project/volttron-postgresql-historian/)

VOLTTRON historian agent that stores data into a PostgreSQL database

### Installation notes

1.  The PostgreSQL database driver supports recent PostgreSQL versions.
    It was tested on 10.x, but should work with 9.x and 11.x.
2.  The user must have SELECT, INSERT, and UPDATE privileges on
    historian tables.
3.  The tables in the database are created as part of the execution of
    the SQLHistorianAgent, but this will fail if the database user does
    not have CREATE privileges.
4.  Care must be exercised when using multiple historians with the same
    database. This configuration may be used only if there is no overlap
    in the topics handled by each instance. Otherwise, duplicate topic
    IDs may be created, producing strange results.


### Configuration

PostgreSQL historian supports two configuration parameters
    
   - connection -  This is a mandatory parameter with type indicating the type of sql historian (i.e. postgresql) and params containing the database access details
   - tables_def - Optional parameter to provide custom table names for topics, data, and metadata.
    
The configuration can be in a json or yaml formatted file. The following examples show minimal connection 
configurations for a psycopg2-based historian. Other options are available and are [documented here](https://www.psycopg.org/docs/module.html#psycopg2.connect) 
**Not all parameters have been tested,  use at your own risk**.

#### Local PostgreSQL Database

The following snippet demonstrates how to configure the historian to use a PostgreSQL database on the local system that
is configured to use Unix domain sockets. The user executing volttron must have appropriate privileges.


##### Yaml Format:
```yaml
    connection:
          # type should be postgresql
          type: postgresql
          params:
            # Relative to the agents data directory
            dbname: "volttron"
        
    tables_def:
        # prefix for data, topics, and (in version < 4.0.0 metadata tables)
        # default is ""
        table_prefix: ""
        # table name for time series data. default "data"
        data_table: data
        # table name for list of topics. default "topics"
        topics_table: topics
 ```

##### JSON format:
```json
    {
        "connection": {
            "type": "postgresql", 
            "params": { "dbname": "volttron" }
        }
    }
```

#### Remote PostgreSQL Database

The following snippet demonstrates how to configure the historian to use a remote PostgreSQL database.
```json
    {
        "connection": {
            "type": "postgresql", 
            "params": { 
                "dbname": "volttron", 
                "host": "historian.example.com", 
                "port": 5432, 
                "user": "volttron", 
                "password": "secret" }
        }
    }
```

#### TimescaleDB Support

Both of the above PostgreSQL connection types can make use of TimescaleDB\'s high performance Hypertable backend for 
the primary timeseries table. The agent assumes you have completed the TimescaleDB installation and setup the 
database by following the instructions here: <https://docs.timescale.com/latest/getting-started/setup> To use, simply
add \'timescale_dialect: true\' to the connection params in the agent config as below

```json
    {
        "connection": {
            "type": "postgresql", 
            "params": { 
                "dbname": "volttron", 
                "host": "historian.example.com", 
                "port": 5432, 
                "user": "volttron", 
                "password": "secret" ,
                "timescale_dialect": true }
        }

    }
```

## Requirements

 - Python >= 3.10
 - psycopg2 library

## Installation

1. Create and activate a virtual environment.

   ```shell
    python -m venv env
    source env/bin/activate
    ```

2. Installing volttron-postgresql-historian requires a running volttron instance and the psycopg2 library 

    ```shell
    pip install volttron
    pip install psycopg2-binary
    
    # Start platform with output going to volttron.log
    volttron -vv -l volttron.log &
    ```
3. Setup database
   
   If this is not a development environment we highly recommend that you create the database and database tables using
   a user with appropriate permissions. This way the database user used by the historian need not have CREATE privileges
   Postgres historian expects two tables 
   a. A topics tables that stores the list of unique topics and its metadata. The default name is "topics". If you use 
      a different name please specify it as part of "tables_def" configuration parameter in agent config. See [example configuration](#yaml-format)
   b. A data table that stores the timeseries data and refers to the topic table using a topic id. The default name is 
      "data". If you use a different name please specify it as part of "tables_def" configuration parameter in 
      agent config. See [example configuration](#yaml-format)

   Below are the sql statements to create database and tables
   <u>Create Database</u>
    ```
       CREATE DATABASE volttron
    ```
   <u>TOPICS tables:</u>
    ```
        CREATE TABLE IF NOT EXISTS topics (
            topic_id SERIAL PRIMARY KEY NOT NULL, 
            topic_name VARCHAR(512) NOT NULL, 
            metadata TEXT, 
            UNIQUE (topic_name)
       )
    ```
    <u>DATA table:</u>
    ```
       CREATE TABLE IF NOT EXISTS data (
           ts TIMESTAMP NOT NULL, 
           topic_id INTEGER NOT NULL, 
           value_string TEXT NOT NULL, 
           UNIQUE (topic_id, ts)
       )
    ```
     <u>Optional timescale hypertable</u>
    ```
       SELECT create_hypertable(data, 'ts', if_not_exists => true)
    ```
     <u>Create index to speed up data access</u>
       If using hypertables:
    ```
        CREATE INDEX IF NOT EXISTS idx_data ON data (topic_id, ts)
    ```
      If not using hypertables:
    ```
        CREATE INDEX IF NOT EXISTS idx_data ON data (ts ASC)
    ```
    <u>Provide correct user permissions for database user to be used by historian agent</u>
    ```
        CREATE USER <some username> with encrypted password <some password>
        GRANT SELECT, INSERT, UPDATE on database <historian db name> to <username used above>
    ``` 
    **NOTE**
    For development environments, you can create a test database and test user, grant all privileges on that test 
    database to the test user and let the historian create tables and indexes at startup. We do not recommend this for 
    production environments

4. Create an agent configuration file 

    Create an agent configuration with appropriate connection parameters as described in the [Configurations section](#Configuration)

5. Install and start the volttron-postgresql-historian.

    ```shell
    vctl install volttron-postgresql-historian --agent-config <path to configuration> --start
    ```

6. View the status of the installed agent

    ```shell
    vctl status
    ```

## Development

Please see the following for contributing guidelines [contributing](https://github.com/eclipse-volttron/volttron-core/blob/develop/CONTRIBUTING.md).

Please see the following helpful guide about [developing modular VOLTTRON agents](https://github.com/eclipse-volttron/volttron-core/blob/develop/DEVELOPING_ON_MODULAR.md)

# Disclaimer Notice

This material was prepared as an account of work sponsored by an agency of the
United States Government.  Neither the United States Government nor the United
States Department of Energy, nor Battelle, nor any of their employees, nor any
jurisdiction or organization that has cooperated in the development of these
materials, makes any warranty, express or implied, or assumes any legal
liability or responsibility for the accuracy, completeness, or usefulness or any
information, apparatus, product, software, or process disclosed, or represents
that its use would not infringe privately owned rights.

Reference herein to any specific commercial product, process, or service by
trade name, trademark, manufacturer, or otherwise does not necessarily
constitute or imply its endorsement, recommendation, or favoring by the United
States Government or any agency thereof, or Battelle Memorial Institute. The
views and opinions of authors expressed herein do not necessarily state or
reflect those of the United States Government or any agency thereof.

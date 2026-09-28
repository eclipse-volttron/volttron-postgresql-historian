
PostgreSQL Historian
====================

This is a VOLTTRON historian agent that stores its data in a PostgreSQL database. It depends on
`volttron-lib-sql-historian <https://pypi.org/project/volttron-lib-sql-historian/>`_ and extends the class
`SQLHistorian <https://github.com/eclipse-volttron/volttron-lib-sql-historian/blob/main/src/historian/sql/historian.py#:~:text=class%20SQLHistorian>`_
This historian also supports TimescaleDB\'s high performance Hypertable backend for the primary timeseries table
The PostgreSQL database driver supports recent PostgreSQL versions.  It was tested on 10.x, but should work with 9.x
and 11.x.

Pre-requisites
**************

- Working modular VOLTTRON environment. Please refer to :ref:`VOLTTRON Instanll Instructions <Platform-Installation>`


User Access Requirements
************************
1. The user must have SELECT, INSERT, and UPDATE privileges on historian tables.
2. In a development environment, the tables in the database could be created as part of the execution of
   the SQLHistorianAgent, if the database user has CREATE privileges. We don't recommend this for production environments

.. _postgresql-configuration:

Configuration
*************

PostgreSQL historian supports two configuration parameters

   - connection -  This is a mandatory parameter with type indicating the type of sql historian (i.e. postgresql) and
     params containing the database access details

   - tables_def - Optional parameter to provide custom table names for topics, data, and metadata.

The configuration can be in a json or yaml formatted file. The following examples show minimal connection
configurations for a psycopg2-based historian. Other options are available and are
`documented here <https://www.psycopg.org/docs/module.html#psycopg2.connect>`_
**Not all parameters have been tested, use at your own risk**.

Local PostgreSQL Database
--------------------------
The following snippet demonstrates how to configure the historian to use
a PostgreSQL database on the local system that is configured to use Unix
domain sockets. The user executing volttron must have appropriate
privileges.

.. note::
   Care must be exercised when using multiple historians with the same database and table names. This configuration may
   be used only if there is no overlap in the topics handled by each instance. Otherwise, duplicate topic
   IDs may be created, producing strange results. Different table name can be used for different historian instances by
   using the optional tables_def configurations

.. _postgresql-configuration-yaml-example:

Yaml Format:
^^^^^^^^^^^^^

.. code:: yaml

    connection:
        # type should be postgresql
        type: postgresql
        params:
        # Relative to the agents data directory
        dbname: "volttron"

    tables_def:
        # prefix for data and topics table
        # default is "". If configured, table name would be <table_prefix>.<data_table> and <table_prefix>.<topics_table>
        # useful when multiple historian instances use same database
        table_prefix: ""
        # table name for time series data. default "data"
        data_table: data
        # table name for list of topics. default "topics"
        topics_table: topics

JSON format:
^^^^^^^^^^^^^

.. code:: json

       {
           "connection": {
               "type": "postgresql",
               "params": { "dbname": "volttron" }
           }
       }


Remote PostgreSQL Database
---------------------------

The following snippet demonstrates how to configure the historian to use
a remote PostgreSQL database.

.. code:: json

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

TimescaleDB Support
--------------------

Both of the above PostgreSQL connection types can make use of
TimescaleDB's high performance Hypertable backend for the primary
timeseries table. The agent assumes you have completed the TimescaleDB
installation and setup the database by following the instructions here:
https://docs.timescale.com/latest/getting-started/setup To use, simply
add 'timescale_dialect: true' to the connection params in the agent
config as below

.. code:: json

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

Optional Configuration
***********************

In addition to the above configuration, SQLite Historian can optionally be configured using all the available
configurations exposed by the SQLHistorian and BaseHistorian. Please refer to
:ref:`SQL Historian <SQLHistorian-Library>` and :ref:`Base Historian Configurations <Base-Historian-Configurations>`
for more details


Installation
************

1. Start volttron instance from your activated volttron virtual environment

2. Setup database

   If this is not a development environment we highly recommend that you
   create the database and database tables using a user with appropriate
   permissions. This way the database user used by the historian need
   not have CREATE privileges.

   Install Postgres
   ----------------

   .. code:: shell

      sudo apt install -y postgresql postgresql-contrib
      sudo systemctl enable --now postgresql4


   | Postgres historian expects two tables

   a. A topics tables that stores the list of unique topics and its
      metadata. The default name is "topics". If you use a different
      name please specify it as part of "tables_def" configuration
      parameter in agent config. See (:ref:`example configuration<postgresql-configuration-yaml-example>`)
   b. A data table that stores the timeseries data and refers to the
      topic table using a topic id. The default name is "data". If you
      use a different name please specify it as part of "tables_def"
      configuration parameter in agent config. See (:ref:`example configuration<postgresql-configuration-yaml-example>`)

   Below are the sql statements to create user, database and tables.

    Create Database:

    ::

          CREATE DATABASE volttron

    TOPICS tables:

    ::

          CREATE TABLE IF NOT EXISTS topics (
              topic_id SERIAL PRIMARY KEY NOT NULL,
              topic_name VARCHAR(512) NOT NULL,
              metadata TEXT,
              UNIQUE (topic_name)
         )

    DATA table:

    ::

         CREATE TABLE IF NOT EXISTS data (
             ts TIMESTAMP NOT NULL,
             topic_id INTEGER NOT NULL,
             value_string TEXT NOT NULL,
             UNIQUE (topic_id, ts)
         )

    Optional timescale hypertable:

    ::

         SELECT create_hypertable(data, 'ts', if_not_exists => true)


    Create index to speed up data access:
    If using hypertables:

    ::

          CREATE INDEX IF NOT EXISTS idx_data ON data (topic_id, ts)

    If not using hypertables:

    ::

          CREATE INDEX IF NOT EXISTS idx_data ON data (ts ASC)

    Provide correct user permissions for database user to be used by
    historian agent

    ::

          CREATE USER <some username> with encrypted password <some password>
          GRANT SELECT, INSERT, UPDATE on database <historian db name> to <username used above>

    .. note::
       For development environments, you can create a test database
       and test user, grant all privileges on that test database to the test
       user and let the historian create tables and indexes at startup. We
       do not recommend this for production environments

4. Create an agent configuration file

   Create an agent configuration with appropriate connection parameters
   as described in :ref:`Configuration section<postgresql-configuration>`

5. Install and start the volttron-postgresql-historian.

   .. code:: shell

      vctl install volttron-postgresql-historian --agent-config <path to configuration> --start

6. View the status of the installed agent

   .. code:: shell

      vctl status

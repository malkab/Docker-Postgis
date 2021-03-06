# PostgreSQL 11.2, PostGIS 2.5.2, GDAL 2.2.4, Patched

## Versions

This image is created with the binaries compiled by the **compilation** image and includes full binaries and assets for the following software:

- **PostgreSQL 11.2;**

- **GEOS 3.7.2;**

- **Proj 4.9.3:** patched with the spanish national grid for conversion between ED50 to ETRS89;

- **GDAL 2.2.4:** also patched;

- **PostGIS 2.5.2:** patched as well;

- **Python 3:** to be used as PL/Python language version.


## Image Creation

Compile software from source by running the **compilation** image and extract binaries as described in its **README**. Then build this image:

```Shell
docker-build.sh
```

or pull it from Docker Hub:

```Shell
docker pull malkab/postgis:endangered_equidna
```


## Image Infrastructure

The image exposes port **5432** and volume **/data**. **data** is the datastore, where logs are located at **pg_log** folder. **postgres** user, which runs the server, has UID and GID 1500.


## Container Creation

The container will check if there is a datastore initiated at **/data**. If not, it will init it with the following parameters, based on environment variables:

- **LOCALE:** locale for the data store and the default database, if any. Defaults to _en_US_. Encoding is locked to **UTF-8**;

- **PG_HBA:** configuration of _pg_hba.con_ access file. See **Configuring the Data Store** for details;

- **PG_CONF:** configuration of _postgresql.conf_ See **Configuring the Data Store** for details;

- **PASSWORD:** the starting password for postgres user.


## Script Database Initialization

For production environments where the container must deploy a database script at creation time, the container will look for a **/initdb.sh** script to execute. When creating the datastore for the first time, the container will attemp to run this script to initialize the database. Include this script in derived images for a production ready PostGIS with databases and infrastructure created at startup.

Take into account when preparing this script that by that time the datastore has already been configured with a postgres password equal to env variable **PASSWORD**. For fully automatization, this password is stored in the **$PASSWORD** env variable.

This script can be used to launch database setup procedures or restore a dump loaded into a production Docker image.

This script is only executed once, when the empty datastore is created.


## Data Persistence

Datastore data can be persisted in a data volume or host mounted folder and be used later by another container. The container checks if folder **/data/** is empty or not. If not, considers the datastore to be not created and creates an empty one.


## Configuring the Data Store

The image allows for configuration of _pg_hba.conf_ and _postgresql.conf_ data store files at creation time and later. This is advanced stuff, refer to the PostgreSQL documentation for details.

**PG_HBA** defaults to:

```txt
ENV PG_HBA "local all all trust#host all all 127.0.0.1/32 trust#host all all 0.0.0.0/0 md5#host all all ::1/128 trust"
```

This defaults should be submitted for basic operation. For universal access, for example for testing, add:

```txt
local all all trust#host all all 0.0.0.0/0 trust#host all all 127.0.0.1/32 trust#host all all ::1/128 trust
```

Modify this variable to configure at creation time. Keep in mind, however, that any value provided to this variable will supersede the default. Don't forget to include basic access permissions if you modify this variable, or the server will be hardly reachable.

Configuration of **postgresql.conf** follows an identical procedure. The environmental variable is **PG_CONF**, which defaults to the following configuration, suitable for a dev machine of 6GB RAM:

```txt
max_connections=50#shared_buffers=1GB#effective_cache_size=3GB#maintenance_work_mem=512MB#checkpoint_completion_target=0.7#wal_buffers=16MB#default_statistics_target=100#random_page_cost=4#effective_io_concurrency=2#work_mem=104857kB#min_wal_size=1GB#max_wal_size=2GB#max_worker_processes=2#max_parallel_workers_per_gather=1#max_parallel_workers=2#max_wal_senders=5#max_locks_per_transaction=1024#listen_addresses='*'#dynamic_shared_memory_type=posix#log_timezone='UTC'#datestyle='iso, mdy'#timezone='UTC'#log_statement='all'#log_directory='pg_log'#log_filename='postgresql-%Y-%m-%d_%H%M%S.log'#logging_collector=on#client_min_messages=notice#log_min_messages=notice#log_line_prefix='%a %u %d %r %h %m %i %e'#log_destination='stderr,csvlog'#log_rotation_size=500MB#log_error_verbosity=default
```

At creation time, locale info is added based on env variables **LOCALE**. Encoding is fixed to be UTF-8.

Logs are stored at **$POSTGRES_DATA_FOLDER/pg_log**.


## Killing the Container

This container will handle signals send to it with _docker kill_ properly, so the database is shut down tidily. Thus:

- **SIGTERM** signals for a smart shutdown, waiting for all connections and transactions to be finished. The server won't allow for new connections, thou:

```Shell
pg_ctl -D . stop -m smart

docker kill -s SIGTERM containername
```

- **SIGINT** signals for fast shutdown. The server will abort current transactions and disconnect users, but will exit nicely otherwise;

```Shell
pg_ctl -D . stop -m fast

docker kill -s SIGINT containername
```

- **SIGQUIT** signals for immediate shutdown. This will leave the database in a improper state and lead to recovery on next startup:

```Shell
pg_ctl -D . stop -m immediate

docker kill -s SIGQUIT containername
```


## Tests

Several tests can be run by launching **test-custom_image_with_script/docker-build-run.sh**.


## Usage Examples

Several usage examples.

Simple run:

```Shell
docker run -ti --rm \
    -v `pwd`/:/ext-out/ \
    -p 5432:5432 \
    malkab/postgis:endangered_equidna
```

Custom run:

```Shell
docker run -ti --rm \
    -v `pwd`/:/ext-out/ \
    -p 5432:5432 \
    -e "LOCALE=es_ES" \
    -e "PASSWORD=thepass" \
    -e "PG_CONF=log_rotation_size=500MB#max_connections=20" \
    -e "PG_HBA=local all all trust#host all all 0.0.0.0/0 trust#host all all 127.0.0.1/32 trust#host all all ::1/128 trust" \
    malkab/postgis:endangered_equidna
```

Custom command run:

```Shell
docker run -ti --rm \
    --network="host" \
    --entrypoint /bin/bash \
    malkab/postgis:endangered_equidna \
    -c "psql -h localhost -p 8888 -U postgres postgres"
```

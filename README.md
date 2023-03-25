# CDC Debezium Cloud SQL to Pub/Sub

## Architecture
![Alt text](./arch.png?raw=true "Optional Title")

## Databases
Supported databases are Cloud MySQL & PostgreSQL.

## Cloud MySQL
1. Set Connection - Private IP via GCP console
2. Set a new MySQL user
   ```
   CREATE USER new_user@'%' identified by 'NEW_USER_PASSWORD';
   GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO new_user@'%' IDENTIFIED BY 'NEW_USER_PASSWORD';
   ```
3. Create 2 pubs/sub topics. The topic names should follow this, MYSQL_INSTANCE_ID and MYSQL_INSTANCE_ID.MYSQL_DBNAME.MYSQL_TABLENAME

    For example, if the instance ID is 'dbmysql', the database name is 'mytest', and the table name is 'example'. Then the topics created are 'dbmysql' & 'dbmysql.mytest.example'
4. Edit the conf_mysql/application.properties file according to the details above

## Cloud PostgreSQL
1. Set Connection - Private IP via GCP console
2. Add Flag 'cloudsql.logical_decoding' = On
3. Create new user & grant priviledge
   ```
   CREATE USER new_user WITH REPLICATION IN ROLE cloudsqlsuperuser LOGIN PASSWORD 'NEW_USER_PASSWORD';
   GRANT ALL PRIVILEGES ON TABLE "TABLE_TO_BE_SINK" TO new_user;
   ```
   TABLE_TO_BE_SINK adalah nama table yg ingin di capture perubahan datanya
4. Create 1 pub/sub topic, nama sesuai dgn POSTGRES_INSTANCE_ID.POSTGRES_SCHEMA.POSTGRES_TABLE

   For example, if the instance ID is 'dbpgsql', the schema name is 'public', and the table name is 'example'. Then the created topic is 'mypgsql.public.example'
5. Edit file conf_postgres/application.properties sesuaikan dgn detail diatas

## Google Service-account

In order for the debezium server to work, it requires the service-account.json file which can be obtained via the GCP console.
* Visit the GCP console then go to the 'API & Services' menu
* On the side menu, click 'Credentials'
* Click the email item in 'Service Accounts' and click the 'KEYS' tab then click 'ADD KEY'
* Download the service-account.json file and copy it to the 'credentials' directory

## HOW TO
The default docker-compose.yml file has 2 servers, debezium-server-mysql & debezium-server-postgres. If you only want to deploy mysql or postgres, you can delete one of them.

First of all, do the export version first

```
export DEBEZIUM_VERSION=1.9
```

Build (only)
```
docker-compose up --no-start
```

Build & Run
```
docker-compose up
```

Build, Run & Daemonized
```
docker-compose up -d
```

If it is running, to monitor its logs
```
docker-compose logs -f
```

Monitor individual server logs
```
docker-compose logs -f debezium-server-mysql
```
or
```
docker-compose logs -f debezium-server-postgres
```

## Suggestion
Use the [**tail pubsub**](https://github.com/jimbas/tailpubsub) application to monitor messages from pubs/sub topics that are pushed by the debezium server when the data in the database table changes

The following is json example that is pushed by debezium-server-mysql to pub/sub when SQL INSERT performed on Cloud MySQL
```
{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":true,"field":"text_col"},{"type":"int32","optional":true,"field":"int_col"},{"type":"string","optional":false,"name":"io.debezium.time.ZonedTimestamp","version":1,"default":"1970-01-01T00:00:00Z","field":"created_at"}],"optional":true,"name":"testsql.mytest.example.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":true,"field":"text_col"},{"type":"int32","optional":true,"field":"int_col"},{"type":"string","optional":false,"name":"io.debezium.time.ZonedTimestamp","version":1,"default":"1970-01-01T00:00:00Z","field":"created_at"}],"optional":true,"name":"testsql.mytest.example.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"version"},{"type":"string","optional":false,"field":"connector"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"ts_ms"},{"type":"string","optional":true,"name":"io.debezium.data.Enum","version":1,"parameters":{"allowed":"true,last,false,incremental"},"default":"false","field":"snapshot"},{"type":"string","optional":false,"field":"db"},{"type":"string","optional":true,"field":"sequence"},{"type":"string","optional":true,"field":"table"},{"type":"int64","optional":false,"field":"server_id"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"query"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"id"},{"type":"int64","optional":false,"field":"total_order"},{"type":"int64","optional":false,"field":"data_collection_order"}],"optional":true,"field":"transaction"}],"optional":false,"name":"testsql.mytest.example.Envelope"},"payload":{"before":null,"after":{"id":17,"text_col":"jibril HRD","int_col":17,"created_at":"2020-01-01T00:00:00Z"},"source":{"version":"1.9.5.Final","connector":"mysql","name":"testsql","ts_ms":1662086230000,"snapshot":"false","db":"mytest","sequence":null,"table":"example","server_id":3293711467,"gtid":"c012d31a-29a9-11ed-beb0-42010a800009:16383","file":"mysql-bin.000002","pos":3908570,"row":0,"thread":15149,"query":null},"op":"c","ts_ms":1662086230773,"transaction":null}}
```

The following is json example that is pushed by debezium-server-postgres to pub/sub when SQL INSERT performed on Cloud PostgreSQL
```
{"schema": {"type": "struct","fields": [{"type": "struct","fields": [{"type": "int32","optional": false,"default": 0,"field": "id"},{"type": "string","optional": true,"field": "name"}],"optional": true,"name": "mypg.public.debezium_test.Value","field": "before"},{"type": "struct","fields": [{"type": "int32","optional": false,"default": 0,"field": "id"},{"type": "string","optional": true,"field": "name"}],"optional": true,"name": "mypg.public.debezium_test.Value","field": "after"},{"type": "struct","fields": [{"type": "string","optional": false,"field": "version"},{"type": "string","optional": false,"field": "connector"},{"type": "string","optional": false,"field": "name"},{"type": "int64","optional": false,"field": "ts_ms"},{"type": "string","optional": true,"name": "io.debezium.data.Enum","version": 1,"parameters": {"allowed": "true,last,false,incremental"},"default": "false","field": "snapshot"},{"type": "string","optional": false,"field": "db"},{"type": "string","optional": true,"field": "sequence"},{"type": "string","optional": false,"field": "schema"},{"type": "string","optional": false,"field": "table"},{"type": "int64","optional": true,"field": "txId"},{"type": "int64","optional": true,"field": "lsn"},{"type": "int64","optional": true,"field": "xmin"}],"optional": false,"name": "io.debezium.connector.postgresql.Source","field": "source"},{"type": "string","optional": false,"field": "op"},{"type": "int64","optional": true,"field": "ts_ms"},{"type": "struct","fields": [{"type": "string","optional": false,"field": "id"},{"type": "int64","optional": false,"field": "total_order"},{"type": "int64","optional": false,"field": "data_collection_order"}],"optional": true,"field": "transaction"}],"optional": false,"name": "mypg.public.debezium_test.Envelope"},"payload": {"before": null,"after": {"id": 25,"name": "wonogiri"},"source": {"version": "1.9.5.Final","connector": "postgresql","name": "mypg","ts_ms": 1662002810219,"snapshot": "false","db": "postgres","sequence": "[\"152053064\",\"152053064\"]","schema": "public","table": "debezium-test","txId": 36325,"lsn": 152053064,"xmin": null},"op": "c","ts_ms": 1662002810621,"transaction": null}}
```
## References

- [Debezium Examples](https://github.com/debezium/debezium-examples)
- [Create and manage service account keys](https://cloud.google.com/iam/docs/creating-managing-service-account-keys)

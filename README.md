# Debezium-KSQL-Kafka Geode Connector

This bundle integrates Geode with Debezium and Confluent KSQL for ingesting initial data and CDC records from MySQL into a Geode cluster via a Kafka sink connector included in the `padogrid` distribution. It supports inserts, updates and deletes.

## Installing Bundle

```bash
install_bundle -download bundle-geode-1-docker-debezium_ksql_kafka
```

:exclamation: If you are running this demo on WSL, make sure your workspace is on a shared folder. The Docker volume it creates will not be visible otherwise.

## Use Case

This use case ingests data changes made in the MySQL database into a Geode cluster via Kafka connectors and also integrates Confluent KSQL for querying Kafka topics as external tables and views. It extends [the original Debezium-Kafka bundle](https://github.com/padogrid/bundle-geode-1-docker-debezium_kafka) with Docker compose, Confluent KSQL, and  the Northwind mock data for `customers` and `orders` tables. It includes the MySQL source connector and the `geode-addon` Debezium sink connectors.

![Debezium-Ksql-Kafka Diagram](images/geode-debezium-ksql-kafka.jpg)

## Required Software

- PadoGrid 0.9.5-SNAPSHOT+ (02/11/2021`)
- Docker
- Docker Compose
- Maven 3.x

## Optional Software

- jq

## Debezium Tutorial

This demo has been put together based on the following blog from Debezium:

[https://debezium.io/blog/2018/05/24/querying-debezium-change-data-eEvents-with-ksql/](https://debezium.io/blog/2018/05/24/querying-debezium-change-data-eEvents-with-ksql/)

## Building Demo

We must first build the demo by running the `build_app` command as shown below. This command copies the Geode and `geode-addon-core` jar files to the Docker container mounted volume in the `padogrid` directory so that the Geode Debezium Kafka connector can include them in its class path. It also downloads the ksql JDBC driver jar and its dependencies in the `padogrid/lib/jdbc` directory.

```bash
cd_docker debezium_ksql_kafka; cd bin_sh
./build_app
```

Upon successful build, the `padogrid` directory should have jar files similar to the following:

```bash
cd_docker debezium_ksql_kafka
tree padogrid
```


```console
adogrid/
├── etc
│   └── client-cache.xml
├── lib
│   ├── ...
│   ├── geode-addon-core-0.9.5-SNAPSHOT.jar
│   ├── ...
├── log
└── plugins
    └── geode-addon-core-0.9.5-SNAPSHOT-tests.jar
```

## Creating Geode Docker Containers

Let's create a Geode cluster to run on Docker containers as follows.

```bash
create_docker -cluster geode -host host.docker.internal
cd_docker geode
```

If you are running Docker Desktop, then the host name, `host.docker.internal`, is accessible from the containers as well as the host machine. You can run the `ping` command to check the host name.

```bash
ping host.docker.internal
```

If `host.docker.internal` is not defined then you will need to use the host IP address that can be accessed from both the Docker containers and the host machine. Run `create_docker -?` or `man create_docker` to see the usage.

```bash
create_docker -?
```

If you are using a host IP other than `host.docker.internal` then you must also make the change in the Debezium Geode connector configuration file as follows.

```bash
cd_docker debezium_ksql_kafka
vi padogrid/etc/client-cache.xml
```

Replace `host.docker.internal` in `client-cache.xml` with your host IP address.

```xml
<client-cache ...>
   ...
    <pool name="serverPool">
         <locator host="host.docker.internal" port="10334" />
    </pool>
   ...
</client-cache>
```

## Creating `perf_test_ksql` app

Create and build `perf_test_ksql` for ingesting mock data into MySQL:

```bash
create_app -app perf_test -name perf_test_ksql
cd_app perf_test_ksql; cd bin_sh
./build_app
```

Set the MySQL user name and password for `perf_test_ksql`:

```bash
cd_app perf_test_ksql
vi etc/hibernate.cfg-mysql.xml
```

Set user name and password as follows:

```xml
                <property name="connection.username">debezium</property>
                <property name="connection.password">dbz</property>
```

## Starting Docker Containers

### 1. Start Geode

```bash
cd_docker geode
docker-compose up
```

### 2. Start Debezium

Start Zookeeper, Kafka, MySQL, Kafka Connect, Confluent KSQL containers:

```bash
cd_docker debezium_ksql_kafka
docker-compose up
```

:exclamation: Wait till all the containers are up before executing the `init_all` script.

Execute `init_all` which performs the following:

- Create the `nw` database and grant all privileges to the user `debezium`:

```bash
cd_docker debezium_ksql_kafka; cd bin_sh
./init_all
```

There are three (3) Kafka connectors that we need to register. The MySQL connector is provided by Debezium and the data connectors are part of the PadoGrid distribution. 

```bash
cd_docker debezium_ksql_kafka; cd bin_sh
./register_connector_mysql
./register_connector_data_customers
./register_connector_data_orders
```

### 3. Ingest mock data into the `nw.customers` and `nw.orders` tables in MySQL

Note that if you run the following more than once then you may see multiple customers sharing the same customer ID when you execute KSQL queries on streams since the streams kepp all the CDC records. The database (MySQL), on the other hand, will always have a single customer per customer ID.

```bash
cd_app perf_test_ksql; cd bin_sh
./test_group -run -db -prop ../etc/group-factory.properties
```

### 4. Run KSQL CLI

```
cd_docker debezium_ksql_kafka; cd bin_sh
./run_ksql_cli
```

KSQL processing by default starts with `latest` offsets. Set the KSQL processing to `earliest` offsets. 

```sql
SET 'auto.offset.reset' = 'earliest';
```

Create and query `customers` stream

```sql
-- Create customers_from_debezium stream
-- (payload struct <after:struct<customerid:string,address:string,city:string,companyname:string,contactname:string,contacttitle:string,country:string,fax:string,phone:string,postalcode:string,region:string>>)

CREATE STREAM customers_from_debezium \
(customerid string,address string,city string,companyname string,contactname string,contacttitle string,country string,fax string,phone string,postalcode string,region string) \
WITH (KAFKA_TOPIC='dbserver1.nw.customers',VALUE_FORMAT='json');

-- Create orders_from_debezium stream
-- (payload struct <after:struct<orderid:string,customerid:string,employeeid:string,freight:double,orderdate:bigint,requireddate:bigint,shipaddress:string,shipcity:string,shiptcountry:string,shipname:string,shippostcal:string,shipregion:string,shipvia:string,shippeddate:string>>)

CREATE STREAM orders_from_debezium \
(orderid string,customerid string,employeeid string,freight double,orderdate bigint,requireddate bigint,shipaddress string,shipcity string,shiptcountry string,shipname string,shippostcal string,shipregion string,shipvia string,shippeddate string) \
WITH (KAFKA_TOPIC='dbserver1.nw.orders',VALUE_FORMAT='json');

-- Repartition

CREATE STREAM orders WITH (KAFKA_TOPIC='ORDERS_REPART',VALUE_FORMAT='json',PARTITIONS=1) as SELECT * FROM orders_from_debezium PARTITION BY orderid;

-- customers_stream
CREATE STREAM customers_stream WITH (KAFKA_TOPIC='CUSTOMERS_REPART',VALUE_FORMAT='json',PARTITIONS=1) as SELECT * FROM customers_from_debezium PARTITION BY customerid;
```

**Compare results: original vs. repartitioned**

Original Query:

```sql
SELECT * FROM orders_from_debezium EMIT CHANGES LIMIT 1;
```

Output:

```console
+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+
|ROWTIME |ROWKEY  |ORDERID |CUSTOMER|EMPLOYEE|FREIGHT |ORDERDAT|REQUIRED|SHIPADDR|SHIPCITY|SHIPTCOU|SHIPNAME|SHIPPOST|SHIPREGI|SHIPVIA |SHIPPEDD|
|        |        |        |ID      |ID      |        |E       |DATE    |ESS     |        |NTRY    |        |CAL     |ON      |        |ATE     |
+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+
|15971609|{"orderI|k0000000|000000-0|575585+2|40.25865|15969522|15975697|365 Osca|New Laur|null    |Terry, K|null    |FL      |1       |15971208|
|01582   |d":"k000|066     |012     |624     |43354865|01000   |06000   |r Cove, |ence    |        |ohler an|        |        |        |45000   |
|        |0000066"|        |        |        |4       |        |        |Lawrence|        |        |d Bernie|        |        |        |        |
|        |}       |        |        |        |        |        |        |ville, R|        |        |r       |        |        |        |        |
|        |        |        |        |        |        |        |        |I 64139 |        |        |        |        |        |        |        |
Limit Reached
Query terminated
```

Repartitioned Query:


```sql
SELECT * FROM orders EMIT CHANGES LIMIT 1;
```

Output:

```console
+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+
|ROWTIME |ROWKEY  |ORDERID |CUSTOMER|EMPLOYEE|FREIGHT |ORDERDAT|REQUIRED|SHIPADDR|SHIPCITY|SHIPTCOU|SHIPNAME|SHIPPOST|SHIPREGI|SHIPVIA |SHIPPEDD|
|        |        |        |ID      |ID      |        |E       |DATE    |ESS     |        |NTRY    |        |CAL     |ON      |        |ATE     |
+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+
|15971609|k0000000|k0000000|000000-0|575585+2|40.25865|15969522|15975697|365 Osca|New Laur|null    |Terry, K|null    |FL      |1       |15971208|
|01582   |066     |066     |012     |624     |43354865|01000   |06000   |r Cove, |ence    |        |ohler an|        |        |        |45000   |
|        |        |        |        |        |4       |        |        |Lawrence|        |        |d Bernie|        |        |        |        |
|        |        |        |        |        |        |        |        |ville, R|        |        |r       |        |        |        |        |
|        |        |        |        |        |        |        |        |I 64139 |        |        |        |        |        |        |        |
Limit Reached
Query terminated
```

Join customers and orders

```sql
-- Create customers table from the topic containing repartitioned customers
CREATE TABLE customers (customerid string, contactname string, companyname string) WITH (KAFKA_TOPIC='CUSTOMERS_REPART',VALUE_FORMAT='json',KEY='customerid');

-- Make a join between customer and its orders and create a query that monitors incoming orders
SELECT customers.customerid,orderid,TIMESTAMPTOSTRING(orderdate, 'yyyy-MM-dd HH:mm:ss'),customers.contactname,customers.companyname,freight FROM orders left join customers on orders.customerid=customers.customerid EMIT CHANGES;
```

Output:

```console
+--------------------------+--------------------------+--------------------------+--------------------------+--------------------------+--------------------------+
|CUSTOMERS_CUSTOMERID      |ORDERID                   |KSQL_COL_2                |CONTACTNAME               |COMPANYNAME               |FREIGHT                   |
+--------------------------+--------------------------+--------------------------+--------------------------+--------------------------+--------------------------+
|000000-0012               |k0000000066               |2020-08-09 05:50:01       |Pacocha                   |MacGyver Group            |40.25865433548654         |
|000000-0048               |k0000000246               |2020-08-07 04:15:10       |Wiegand                   |Kuhic-Bode                |157.48781188841855        |
|000000-0072               |k0000000366               |2020-08-08 21:53:28       |Pfannerstill              |Weimann, Hills and Schmitt|79.03684813199516         |
|000000-0024               |k0000000126               |2020-08-05 05:35:38       |Torphy                    |Bednar LLC                |55.94516435026855         |
|000000-0084               |k0000000426               |2020-08-05 22:09:37       |Nolan                     |Quigley Group             |10.966276834050536        |
|000000-0000               |k0000000006               |2020-08-06 02:02:29       |Fadel                     |Miller-Abbott             |12.769565213351175        |
|000000-0048               |k0000000247               |2020-08-07 13:23:20       |Wiegand                   |Kuhic-Bode                |60.65402769673416         |
...
```

Quit KSQL:

```
Ctrl-D
```

### Watch topics

```bash
cd_docker debezium_ksql_kafka; cd bin_sh
./watch_topic dbserver1.nw.customers
./watch_topic dbserver1.nw.orders
```

### Run MySQL CLI

```bash
cd_docker debezium_ksql_kafka; cd bin_sh
./run_mysql_cli
```

Run join query as we did with KSQL:

```sql
use nw;
select c.customerid,c.address,o.orderid,o.customerid,o.freight
from customers c \
inner join orders o \
on (c.customerid=o.customerid) order by c.customerid,o.orderid limit 10;
```

Output:

```console
+-------------+----------------------------------------------------------+-------------+-------------+--------------------+
| customerid  | address                                                  | orderid     | customerid  | freight            |
+-------------+----------------------------------------------------------+-------------+-------------+--------------------+
| 000000-0000 | Apt. 257 08047 Altenwerth Avenue, Kunzeborough, NM 63838 | k0000000032 | 000000-0000 |  131.7820778619269 |
| 000000-0000 | Apt. 257 08047 Altenwerth Avenue, Kunzeborough, NM 63838 | k0000000127 | 000000-0000 |  27.77469027097803 |
| 000000-0000 | Apt. 257 08047 Altenwerth Avenue, Kunzeborough, NM 63838 | k0000000158 | 000000-0000 | 112.43667178731734 |
| 000000-0000 | Apt. 257 08047 Altenwerth Avenue, Kunzeborough, NM 63838 | k0000000229 | 000000-0000 |  94.25505877773637 |
| 000000-0000 | Apt. 257 08047 Altenwerth Avenue, Kunzeborough, NM 63838 | k0000000398 | 000000-0000 | 139.70009999825962 |
| 000000-0000 | Apt. 257 08047 Altenwerth Avenue, Kunzeborough, NM 63838 | k0000000413 | 000000-0000 | 14.548987234280375 |
| 000000-0000 | Apt. 257 08047 Altenwerth Avenue, Kunzeborough, NM 63838 | k0000000425 | 000000-0000 |  65.05634014122326 |
| 000000-0000 | Apt. 257 08047 Altenwerth Avenue, Kunzeborough, NM 63838 | k0000000525 | 000000-0000 | 165.09433352007548 |
| 000000-0000 | Apt. 257 08047 Altenwerth Avenue, Kunzeborough, NM 63838 | k0000000607 | 000000-0000 | 160.35802796431958 |
| 000000-0000 | Apt. 257 08047 Altenwerth Avenue, Kunzeborough, NM 63838 | k0000000623 | 000000-0000 | 182.55195173466166 |
+-------------+----------------------------------------------------------+-------------+-------------+--------------------+
```

Quit MySQL CLI:

```sql
quit
```

### Check Kafka Connect

```bash
# Check status
curl -Ss -H "Accept:application/json" localhost:8083/ | jq

# List registered connectors 
curl -Ss -H "Accept:application/json" localhost:8083/connectors/ | jq
```

The last command should display the connectors that we registered previously.

```console
[
  "nw-connector",
  "customers-sink",
  "orders-sink"
]
```

### Drop KSQL Statements

The following scripts are provided to drop KSQL queries using the KSQL REST API.

```
cd_app debezium_ksql_kafka; cd bin_sh

# Drop all queries
./ksql_drop_all_queries

# Drop all streams
./ksql_drop_all_streams

# Drop all tables
./ksql_drop_all_tables
```

### Run Geode `gfsh`

The `run_gfsh` script logs into the locator container and starts `gfsh`. You can connect to the default locator, localhost[10334], and execture OQL queries to verify MySQL data ingested via Debezium is also captured in the Geode cluster.

Login to `gfsh`:

```bash
cd_docker debezium_ksql_kafka; cd bin_sh
./run_gfsh
```

From `gfsh`, query the `/nw/customers` and `/nw/orders` regions.

```bash
# Connect to the default locator
connect

# View region sizes
describe region --name=/nw/customers
describe region --name=/nw/orders

# Execute OQL queries on /nw/customers and /nw/orders
query --query="select * from /nw/customers limit 100"
query --query="select * from /nw/orders limit 100"
```

## Teardown

```bash
# Stop KSQL and Kafka containers
cd_docker debezium_ksql_kafka
docker-compose down

# Stop Geode containers
cd_docker geode
docker-compose down
```

## References

1. Querying Debezium Change Data Events With KSQL, Jiri Pechanec, Debezium blog, https://debezium.io/blog/2018/05/24/querying-debezium-change-data-eEvents-with-ksql/
2. Debizium-Kafka Geode Connector, PadoGrid bundle, https://github.com/padogrid/bundle-geode-1-docker-debezium_kafka 
3. Debezium-Hive-Kafka Geode Connector, Padogrid bundle, https://github.com/padogrid/bundle-geode-1-docker-debezium_hive_kafka
4. Confluent KSQL, GitHub, https://github.com/confluentinc/ksql

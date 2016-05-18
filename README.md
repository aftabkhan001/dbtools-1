
This tool archives data from one database into remote database.

## Requirements
Install following tools/packages
* `mysql client tool` - mysql 
* `python-pip` - e.g. `apt-get install python-pip`
* `virtualenv` - e.g. `pip install virtualenv`
* `Python 2.7.3`


## Installation

Setup virtual environment:
```
sudo mkdir -p /opt/db-scripts
sudo chmod a+rw /opt/db-scripts
cd /opt/db-scripts
```
Save `/` here
```
chmod +x setup_virtualenv.sh
./setup_virtualenv.sh
chmod +x run_db_archiver.sh
```

## Configuration

Modify configuration file `db_archiver.ini` as per requirements:

**Sample configuration file for ifactory database**

```
[database]

# Unique name for each archiver instance
title=ifactory_wfr

# Source database host and user/password
user=dbatools
password=********
host=10.10.16.10
port=3306

# Specify values for parameters used by archiver
# Specify Root table name and database
database=PRODUCTION
root_table=PROD_COMMANDES

# Where condition
where=ETAT IN ('PO','QO') and DATE_PASSAGE <= DATE_ADD(NOW(), INTERVAL - 7 DAY)

# Use small batch_size when extraction is ON (default)
batch_size=100

# Use long pause time between batch run to maintain steady load on the server
sleep_time=10

# Path name of the directory in which to store temporary files
tmpdir=/data/mysql/tmp

# default date filter used by batch deletes, see include_tables option variable 
default_date_filter=DATE_PASSAGE < DATE_ADD(NOW(),INTERVAL - 1 MONTH)

# Specify list of tables to be excluded (comma separated) from the first list of tables to be archived
# I have added PROD_TRACES to the ignore_tables list, as it takes longer to archive using ids from PROD_COMMANDES
ignore_tables=PROD_TRACES

# Use this option to specify tables (semi-colon separated) to the second list of table
# This is helpful if you have mutliple ROOT tables or if the table is quite large and cannot be handled when added to the first list of tables e.g. PROD_TRACES
# syntax: TABLE_NAME:CONDITION[:batch_size] 
include_tables=PROD_TRACES:default_date_filter;PROD_RESSOURCES_NAS:ID_RESSOURCE_STATUT IN (2);PROD_HISTO_SEQUENCEUR:default_date_filter;PROD_CTRL_POIDS_QO:default_date_filter;PROD_STAT_SHIPPING_COST:DATE_CONTROLEUR < DATE_ADD(now(),INTERVAL - 1 MONTH);PROD_INTEGRATION_IPROD:default_date_filter;PROD_CONTENU_PDF:default_date_filter:50;PROD_CONTENU_PDF_DETAIL:default_date_filter;PROD_POIDS_INDEX:default_date_filter;PROD_ETAPE_TRAITEMENT:default_date_filter;PROD_EXECUTION:default_date_filter

# enable/disable data extractions, if you only want to purge data but do not wish to store into remote database
# default enabled
enable_data_extraction=1

# Use this option to enable session variables (semi-colon separated) e.g. enable skip_replication option to allow history db to filter archiver operations
set_vars=skip_replication=1

# Enable logging default disabled, when enabled it would populate progress in `dbtools.archive_progress` (on destination database)
log_progress=0

# Tables without foreign key to MAIN_TABLE (e.g. PROD_COMMANDES) would not be archived. Please specify virtual foreign keys here to help archive tables with no psyhcial foreign keys
# Virtual foreign keys are only supported with PROD_COMMANDES table
# syntax: TABLE_NAME:COLUMN_NAME
virtual_fks=PROD_ITEMS:ID_PROD_COMMANDE;AFF_COLIS:ID_PROD_COMMANDE;PROD_ENVELOPPES:ID_PROD_COMMANDE;PROD_CREATIONS:ID_PROD_COMMANDE;PROD_TRACES:ID_PROD_COMMANDE
# Print queries and exit without doing anything.
dry_run=0

# Print verbose output (useful for progress monitoring)
# default 0
verbose=1

# Specify options for remote database

[archive]
host=10.10.16.11
database=PRODUCTION
port=3306 
# Specify user/pass information if different from source database
user=dbatools
password=********
```
**Add database schema/tables used by archiver**

Load following SQL script on both source/destination servers

```
mysql> create database if not exists dbtools;
mysql> use dbtools;
mysql> source dbtools.sql;
```
## Permissions and Privileges Needed
In order to run this tool, it would require READ, WRITE and EXECUTE permissions at a filesystem level in tmpdir (configurable, default /tmp). 
The database user needs the following privileges on the tables / databases to be archived/purged:

``` 
# Add user account on both source/destination servers with INSERT, DELETE and SELECT db privileges, for example:

mysql> GRANT INSERT, DELETE, SELECT ON PRODUCTION.* TO archiver@'10.%' IDENTIFIED BY 's3cret';

# Grant user permissions on dbtools db [optional], only requires for detailed logging

mysql> GRANT ALL ON dbtools.* TO archiver@'10.%'
```

## Usage

**Examples**

*Running Archiver*

```
$ sudo ./run_db_archiver.sh --help
Usage: my_db_archiver_3.py [options]

Options:
  -h, --help            show this help message and exit
  -f CONFIG_FILE, --config-file=CONFIG_FILE
                        Specify path to configuration file[default: none]

$ sudo ./run_db_archiver.sh  --config-file=db_archiver_test.ini
╒════════════════════════════════════════════════╤═══════════════════════════════════════════════╕
│ Starting database archive script version 1.0:  │ ifactory_wfr_test                             │
├────────────────────────────────────────────────┼───────────────────────────────────────────────┤
│ 1) Batch Size                                  │ 100                                           │
├────────────────────────────────────────────────┼───────────────────────────────────────────────┤
│ 2) Days old data to archive                    │ 7                                             │
├────────────────────────────────────────────────┼───────────────────────────────────────────────┤
│ 3) Source database server                      │ 10.10.16.11                                   │
├────────────────────────────────────────────────┼───────────────────────────────────────────────┤
│ 4) Target database server                      │ 10.16.6.6                                     │
├────────────────────────────────────────────────┼───────────────────────────────────────────────┤
│ 5) Data Extaction                              │ Enabled                                       │
├────────────────────────────────────────────────┼───────────────────────────────────────────────┤
│ 7) Logging                                     │ Disabled                                      │
├────────────────────────────────────────────────┼───────────────────────────────────────────────┤
│ 8) Tables Ignored                              │ PROD_USERS_CTRL_not_needed,PROD_PFC_TO_DELETE │
╘════════════════════════════════════════════════╧═══════════════════════════════════════════════╛
Running first batch
Fri Feb 19 14:16:58 2016 Batch completed. Execution time: 6s - Commandes deleted :100
Purging/archiving tables not included in the main program
Taking a nap:5s
Running next batch
Fri Feb 19 14:17:32 2016 Batch completed. Execution time: 17s - Commandes deleted :100
Purging/archiving tables not included in the main program
Taking a nap:5s
Running next batch
Fri Feb 19 14:17:54 2016 Batch completed. Execution time: 6s - Commandes deleted :100
Purging/archiving tables not included in the main program
Taking a nap:5s
^C
 Ctrl+c detected
 Bye
...
```

*Running Archiver when dry_run option is enabled in the configuration*

```
akhan@itprod-dba-test-sa-02:/opt/scripts$ sudo ./run_db_archiver.sh
[sudo] password for akhan:
╒════════════════════════════════════════════════╤═══════════════════════════════════════════════╕
│ Starting database archive script version 1.0:  │ ifactory_wfr_test                             │
├────────────────────────────────────────────────┼───────────────────────────────────────────────┤
│ 1) Batch Size                                  │ 200                                           │
├────────────────────────────────────────────────┼───────────────────────────────────────────────┤
│ 2) Days old data to archive                    │ 10                                            │
├────────────────────────────────────────────────┼───────────────────────────────────────────────┤
│ 3) Source database server                      │ 10.16.6.6                                     │
├────────────────────────────────────────────────┼───────────────────────────────────────────────┤
│ 4) Target database server                      │ 10.10.16.11                                   │
├────────────────────────────────────────────────┼───────────────────────────────────────────────┤
│ 5) Data Extaction                              │ Enabled                                       │
├────────────────────────────────────────────────┼───────────────────────────────────────────────┤
│ 7) Logging                                     │ Disabled                                      │
├────────────────────────────────────────────────┼───────────────────────────────────────────────┤
│ 8) Tables Ignored                              │ PROD_USERS_CTRL_not_needed,PROD_PFC_TO_DELETE │
╘════════════════════════════════════════════════╧═══════════════════════════════════════════════╛
Running first batch
Time spent to prepare tables list:6s
 Running mysqldump
mysqldump --host=10.16.6.6 --user=dbatools --password=******* --port=3306 --complete-insert --compress --skip-disable-keys --quick --no-create-info --skip-add-locks --single-transaction --replace PRODUCTION PROD_ATTENTE_IMPOSITION -w"ID_PROD_COMMANDE IN('18537430','18537570','18552815','18539121','18569763','18541016','18536349','18541978','18538561','18538939','18544234','18537322','18544445','18573862','18575207','18544272','18567990','18576324','18538776','18542734','18537504','18571450','18573269','18537912','18541405','18537021','18571219','18542360','18574274','18537989','18574088','18577752','18539058','18544184','18538373','18537276','18566172','18538867','18574042','18536918','18538884','18551303','18537240','18557613','18573349','18573259','18537225','18537488','18544196','18571225','18544172','18543516','18537363','18572512','18572714','18537618','18536899','18539139','18570576','18536895','18542257','18536931','18568385','18537476','18537096','18537796','18537647','18536920','18538535','18537879','18537272','18537894','18550069','18538583','18537492','18576139','18536979','18537405','18538504','18537969','18541900','18576646','18557329','18577272','18538875','18554502','18537950','18537702','18567505','18537639','18567145','18555716','18537752','18566056','18577184','18537707','18554194','18578001','18538284','18537146')" --log-error=/data/mysql/tmp/ifactory_wfr_test//__mysqldump_warnings_ifactory_wfr_test.txt  > /data/mysql/tmp/ifactory_wfr_test//18967__PROD_ATTENTE_IMPOSITION.sql ; if [ $? -ne 0 ]; then exit 1; fi

 Loading SQL dump file into remote database
mysql --host=10.10.16.11 --user=dbatools --port=3306 --compress --password=******* PRODUCTION < /data/mysql/tmp/ifactory_wfr_test//18967__PROD_ATTENTE_IMPOSITION.sql ; if [ $? -ne 0 ]; then exit 1; fi ;  rm -f /data/mysql/tmp/ifactory_wfr_test//18967__PROD_ATTENTE_IMPOSITION.sql

 Running delete
DELETE FROM PRODUCTION.PROD_ATTENTE_IMPOSITION WHERE ID_PROD_COMMANDE IN('18537430','18537570','18552815','18539121','18569763','18541016','18536349','18541978','18538561','18538939','18544234','18537322','18544445','18573862','18575207','18544272','18567990','18576324','18538776','18542734','18537504','18571450','18573269','18537912','18541405','18537021','18571219','18542360','18574274','18537989','18574088','18577752','18539058','18544184','18538373','18537276','18566172','18538867','18574042','18536918','18538884','18551303','18537240','18557613','18573349','18573259','18537225','18537488','18544196','18571225','18544172','18543516','18537363','18572512','18572714','18537618','18536899','18539139','18570576','18536895','18542257','18536931','18568385','18537476','18537096','18537796','18537647','18536920','18538535','18537879','18537272','18537894','18550069','18538583','18537492','18576139','18536979','18537405','18538504','18537969','18541900','18576646','18557329','18577272','18538875','18554502','18537950','18537702','18567505','18537639','18567145','18555716','18537752','18566056','18577184','18537707','18554194','18578001','18538284','18537146') LIMIT 20000) /* output truncated ...*/

...

Table                                    Rows Found    Rows copied    Rows deleted    Time spent  Parent Table
-------------------------------------  ------------  -------------  --------------  ------------  -------------------
PROD_ITEMS                                      294              0               0          0.2   PROD_COMMANDES
PROD_ATTENTE_BATCHING                            43              0               0          0.1   PROD_ITEMS
PROD_ATTENTE_CONFIGURATION_IMPOSITION            70              0               0          0.1   PROD_ITEMS
PROD_ATTENTE_CONFIGURATION_ITEM                  15              0               0          0.09  PROD_ITEMS
PROD_ATTENTE_IMPOSITION                           6              0               0          0.1   PROD_ITEMS
PROD_ATTENTE_INTEGRATION                         15              0               0          0.09  PROD_ITEMS
PROD_ATTENTE_ROUTAGE_IMPRESSION                   6              0               0          0.1   PROD_ITEMS
PROD_COMPOSANT                                  261              0               0          0.19  PROD_ITEMS
PROD_RESSOURCES_NAS                             206              0               0          0.09  PROD_ITEMS
PROD_ATTENTE_ROUTAGE_IMPRESSION                  16              0               0          0.2   PROD_RESSOURCES_NAS
PROD_RESSOURCES_ARCHIVE_DETAIL                  108              0               0          0.28  PROD_RESSOURCES_NAS
AFF_COLIS                                        54              0               0          0.18  PROD_COMMANDES
PROD_ENVELOPPES                                 200              0               0          0.2   PROD_COMMANDES
PROD_CREATIONS                                   34              0               0          0.2   PROD_COMMANDES
PROD_TRACES                                    2147              0               0          0.19  PROD_COMMANDES
PROD_INTEGRATION_IPROD                          294              0               0          0.19  PROD_COMMANDES
PROD_COMMANDES_INFOS                            148              0               0          0.18  PROD_COMMANDES
PROD_STAT_SHIPPING_COST                          52              0               0          0.19  PROD_COMMANDES
PROD_ATTENTE_CONFIGURATION_COMMANDE              21              0               0          0.18  PROD_COMMANDES
PROD_ATTENTE_CONFIGURATION_IMPOSITION            21              0               0          0.19  PROD_COMMANDES
PROD_ATTENTE_IMPOSITION                          11              0               0          0.09  PROD_COMMANDES
PROD_ATTENTE_ROUTAGE_IMPRESSION                   0              0               0          0     PROD_RESSOURCES_NAS
PROD_RESSOURCES_ARCHIVE_DETAIL                    0              0               0          0     PROD_RESSOURCES_NAS
Mon Feb 29 10:24:14 2016 Batch completed. Execution time: 14s - Commandes deleted :0
Mon Feb 29 10:24:14 2016 DRY RUN end-- Program aborted

 Program ended
 Bye

mysqldump --host=10.10.16.10 --user=dbatools --password=*******  --compress --quick --no-create-info --single-transaction --insert-ignore PRODUCTION PROD_SUIVI_GLOBALMAIL -w"ID_PROD_ITEM IN(485533202,485533203,485533204,485533205,485533
206,485533207,485533208,485533209,485533210,485533211,485533212,485533213,485533214,485533215,485533216,485533217,485533218,485533219,485533220,485533221,485533222,485533223,485533224,485533225,485533226,485533227,485533228,485533229,48
5533230,485533231,485533232,) /* output truncated ...*/ | mysql --host=10.10.16.11 --user=dbatools --compress --password=******* PRODUCTION

DELETE FROM PRODUCTION.PROD_SUIVI_GLOBALMAIL WHERE ID_PROD_ITEM IN(485533202,485533203,485533204,485533205,485533206,485533207,485533208,485533209,485533210,485533211,485533212,485533213,485533214,485533215,485533216,485533217,485533218
,485533219,485533220,485533221,485533222,485533223,485533224,485533225,485533226,485533227,485533228,485533229,485533230,485533231,485533232,485533233,485533234,485533235,485533236,485533237,485533238,485533239,485533240,485533241,48553
3242,485533243,485533244,485) /* output truncated ...*/

mysqldump --host=10.10.16.10 --user=dbatools --password=*******  --compress --quick --no-create-info --single-transaction --insert-ignore PRODUCTION PROD_SUIVI_GLOBALMAIL -w"ID_PROD_ITEM IN(485533252,485533253,485533254,485533255,485533
256,485533257,485533258,485533259,485533260,485533261,485533262,485533263,485533264,485533265,485533266,485533267,485533268,485533269,485533270,485533271,485533272,485533273,485533274,485533275,485533276,485533277,485533278,485533279,48
5533280,485533281,485533282,) /* output truncated ...*/ | mysql --host=10.10.16.11 --user=dbatools --compress --password=******* PRODUCTION

DELETE FROM PRODUCTION.PROD_SUIVI_GLOBALMAIL WHERE ID_PROD_ITEM IN(485533252,485533253,485533254,485533255,485533256,485533257,485533258,485533259,485533260,485533261,485533262,485533263,485533264,485533265,485533266,485533267,485533268
,485533269,485533270,485533271,485533272,485533273,485533274,485533275,485533276,485533277,485533278,485533279,485533280,485533281,485533282,485533283,485533284,485533285,485533286,485533287,485533288,485533289,485533290,485533291,48553
3292,485533293,485533294,485) /* output truncated ...*/
...

'Thu Jan 21 16:17:21 2016', ' Execution time: 0s - Commandes deleted :2000')
('Thu Jan 21 16:17:21 2016', 'Taking a nap:1s')
Running next batch
('Thu Jan 21 16:17:22 2016', 'dry_run end', '-- Program aborted')


```

## Logs
Progress is directed to stdout (stats are printed when verbose option is enabled)

```
Running next batch
  #  Table                                    Rows copied    Rows deleted    Time spent  Parent Table           Loop iter
---  -------------------------------------  -------------  --------------  ------------  -------------------  -----------
  0  PROD_ITEMS                                       119             119          0.17  PROD_COMMANDES                 1
  2  PROD_ATTENTE_CONFIGURATION_IMPOSITION            143             143          0.13  PROD_ITEMS                     2
  3  PROD_ATTENTE_CONFIGURATION_ITEM                   76              76          0.11  PROD_ITEMS                     2
  4  PROD_ATTENTE_IMPOSITION                          131             131          0.13  PROD_ITEMS                     2
  5  PROD_ATTENTE_INTEGRATION                          76              76          0.1   PROD_ITEMS                     2
  6  PROD_ATTENTE_ROUTAGE_IMPRESSION                  124               1          0.05  PROD_ITEMS                     1
  7  PROD_COMPOSANT                                  2331            2331          0.29  PROD_ITEMS                     2
  8  PROD_LOT_ITEM                                     76              76          0.1   PROD_ITEMS                     2
  9  PROD_RESSOURCES_NAS                              659             659          0.38  PROD_ITEMS                     2
 10  AFF_COLIS                                        100             100          0.11  PROD_COMMANDES                 1
 11  PROD_ENVELOPPES                                  100             100          0.07  PROD_COMMANDES                 1
 12  PROD_CREATIONS                                    76              76          0.09  PROD_COMMANDES                 1
 13  PROD_TRACES                                     5070            5070          1.23  PROD_COMMANDES                 1
 15  PROD_ATTENTE_CONFIGURATION_COMMANDE               12              12          0.05  PROD_COMMANDES                 1
 16  PROD_ATTENTE_CONFIGURATION_IMPOSITION             12              12          0.05  PROD_COMMANDES                 1
 17  PROD_ATTENTE_IMPOSITION                           12              12          0.06  PROD_COMMANDES                 1
 18  PROD_ATTENTE_INTEGRATION                          12              12          0.05  PROD_COMMANDES                 1
 20  PROD_ATTENTE_ROUTAGE_IMPRESSION                   12              12          0.06  PROD_COMMANDES                 1
 21  PROD_LIVRAISON                                   100             100          0.06  PROD_COMMANDES                 1
 22  PROD_LOTS_SILVERPRINT_COMMANDE                    13              13          0.04  PROD_COMMANDES                 1
 23  PROD_RESSOURCES_NAS                              100             100          0.11  PROD_COMMANDES                 1
 24  PROD_ATTENTE_ROUTAGE_IMPRESSION                  130             130          0.37  PROD_RESSOURCES_NAS            7
 25  PROD_RESSOURCES_ARCHIVE_DETAIL                   373             373          0.57  PROD_RESSOURCES_NAS            7
 26  PROD_SNAPSHOT_SILVERPRINT_COMMANDE                12              12          0.13  PROD_COMMANDES                 1
Thu Feb 18 14:52:45 2016 Batch completed. Execution time: 4s - Commandes deleted :100
Purging/archiving tables not included in the main program
Thu Feb 18 14:52:46 2016 PROD_TRACES -> (5) loop passes, time spent: 0s - Rows deleted :2500
Thu Feb 18 14:52:47 2016 PROD_RESSOURCES_NAS -> (5) loop passes, time spent: 1s - Rows deleted :2500
Thu Feb 18 14:52:48 2016 PROD_HISTO_SEQUENCEUR -> (5) loop passes, time spent: 0s - Rows deleted :2500
Thu Feb 18 14:52:48 2016 PROD_CTRL_POIDS_QO -> (5) loop passes, time spent: 0s - Rows deleted :0
Thu Feb 18 14:52:49 2016 PROD_STAT_SHIPPING_COST -> (5) loop passes, time spent: 0s - Rows deleted :2500
Thu Feb 18 14:52:50 2016 PROD_INTEGRATION_IPROD -> (5) loop passes, time spent: 0s - Rows deleted :2500
Thu Feb 18 14:52:50 2016 PROD_CONTENU_PDF -> (5) loop passes, time spent: 0s - Rows deleted :250
Thu Feb 18 14:52:51 2016 PROD_CONTENU_PDF_DETAIL -> (5) loop passes, time spent: 0s - Rows deleted :2619
Thu Feb 18 14:52:51 2016 PROD_POIDS_INDEX -> (5) loop passes, time spent: 0s - Rows deleted :2500
Thu Feb 18 14:52:51 2016 PROD_ETAPE_TRAITEMENT -> (5) loop passes, time spent: 0s - Rows deleted :0
Thu Feb 18 14:52:52 2016 PROD_EXECUTION -> (5) loop passes, time spent: 0s - Rows deleted :0
Taking a nap:5s
...
```
*To observe detailed progress*
* enable ``log_progress``

```
MariaDB [dbtools]> select * from archive_progress order by 1 desc limit 10;
+-------+----------+---------------------------------+--------------+----------------------+---------------------+-------------+--------------------+
| id    | batch_id | table_name                      | rows_deleted | time_spent_to_delete | dt                  | rows_copied | time_spent_to_copy |
+-------+----------+---------------------------------+--------------+----------------------+---------------------+-------------+--------------------+
| 26421 |     3688 | PROD_TRACES                     |         1000 |                  0.1 | 2016-02-10 15:10:04 |        1000 |               0.35 |
| 26419 |     3687 | PROD_TRACES                     |         1000 |                 0.09 | 2016-02-10 15:10:03 |        1000 |               0.37 |
| 26417 |     3686 | PROD_TRACES                     |         1000 |                 0.07 | 2016-02-10 15:10:02 |        1000 |               0.35 |
| 26415 |     3685 | PROD_TRACES                     |         1000 |                 0.08 | 2016-02-10 15:10:01 |        1000 |               0.36 |
| 26413 |     3684 | PROD_TRACES                     |         1000 |                 0.08 | 2016-02-10 15:10:00 |        1000 |               0.34 |
| 26411 |     3683 | PROD_COMMANDES                  |          100 |                 0.03 | 2016-02-10 15:10:00 |         100 |               0.26 |
| 26409 |     3683 | PROD_ITEMS                      |          148 |                 0.08 | 2016-02-10 15:09:59 |         148 |               0.26 |
| 26407 |     3683 | PROD_ATTENTE_BATCHING           |           91 |                 0.03 | 2016-02-10 15:09:58 |          91 |               0.24 |
| 26403 |     3683 | PROD_ATTENTE_CONFIGURATION_ITEM |           43 |                 0.01 | 2016-02-10 15:09:57 |          43 |               0.23 |
| 26399 |     3683 | PROD_COMPOSANT                  |         2327 |                 0.09 | 2016-02-10 15:09:56 |        2327 |               0.35 |
+-------+----------+---------------------------------+--------------+----------------------+---------------------+-------------+--------------------+
10 rows in set (0.01 sec)
```
## Known issues:
When archiver is killed using normal method i.e. "Ctr+c", it might leave some related services still running e.g.

```
$ ps aux |grep mysql
akhan     4363  0.0  0.0   9404   936 pts/7    S+   09:31   0:00 grep --color=auto mysql
root     12493  0.0  0.0  42752  1716 ?        S    Jan25   0:00 mysql --host=10.10.16.11 --user=archiver --compress --password=x xxxxxxxxxxxxxx PRODUCTION
```
Before restarting archiver, please make sure all related processes are stopped.

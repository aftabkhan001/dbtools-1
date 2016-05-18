
This tool delete hierarchical data, you specify condition to delete data from parent/root_table, it automatically detects/delete related data from all child tables

## RISKS
It is a read-write tool. It deletes data from the source by default, so you should backup your data and test it with a copy of live data with the --dry-run option  if you’re not sure about them.

## Requirements
Install following tools/packages
* `Python 2.7.x`
* `mysql connector/python` - e.g. mysql-connector-python-2.0.4.zip
* `PrettyTable` -e.g https://pypi.python.org/packages/source/P/PrettyTable/prettytable-0.7.2.tar.bz2

## Usage
```
python  del_data.py -h
usage: del_data.py [-h] [--host HOST] [--user USER] --password PASSWORD
                   --database DATABASE --where WHERE [--limit LIMIT]
                   --root-table ROOT_TABLE [--dry-run] [--port PORT]

Delete hierarchical data

optional arguments:
  -h, --help            show this help message and exit
  --host HOST           hostname, (default: localhost)
  --user USER           database user, (default: root)
  --password PASSWORD   database password
  --database DATABASE   database name
  --where WHERE         where condition
  --limit LIMIT         batch size, (default: 100)
  --root-table ROOT_TABLE
                        root table name
  --dry-run             print queries and do nothing
  --port PORT           Port number to use for connection, (default: 3306)
```

**Examples**

Below examples use sample database: http://www.mysqltutorial.org/mysql-sample-database.aspx

* DRY_RUN - Print queries and exit without doing anything
```
python del_data.py --user root --password ***** --database classicmodels --where "customerNumber=103" --root-table customers --limit 1 --dry-run
DRY-RUN
+--------------------------+----------------------------------+
| 1) Database host server: | localhost                        |
| 2) Root Table:           | customers                        |
| 3) Where condition:      | WHERE customerNumber=103 LIMIT 1 |
+--------------------------+----------------------------------+
Running first batch
DELETE FROM classicmodels.payments WHERE customerNumber IN('103') LIMIT 1000
DELETE FROM classicmodels.orderdetails WHERE orderNumber IN('10123','10298','10345') LIMIT 1000
DELETE FROM classicmodels.orders WHERE customerNumber IN('103') LIMIT 1000
DELETE FROM classicmodels.customers WHERE  customerNumber IN('103') LIMIT 1000
+---+--------------+------------+--------------+------------+--------------+
| # | Table        | Rows found | Rows deleted | Time spent | Parent Table |
+---+--------------+------------+--------------+------------+--------------+
| 0 | customers    | 1          | 0            | 0s         |              |
| 1 | orders       | 3          | 0            | 0.0s       | customers    |
| 2 | orderdetails | 7          | 0            | 0.0s       | orders       |
| 3 | payments     | 3          | 0            | 0.0s       | customers    |
+---+--------------+------------+--------------+------------+--------------+
Wed May 18 16:46:57 2016 Batch completed. Execution time: 0s - Items deleted :0
Wed May 18 16:46:57 2016 DRY RUN end

 Program ended
 Bye
```

* Delete all data from parent table and related data from its child tables

```
python  del_data.py --user root --password ******* --database classicmodels --where "customerNumber=103" --root-table customers --limit 1
+--------------------------+----------------------------------+
| 1) Database host server: | localhost                        |
| 2) Root Table:           | customers                        |
| 3) Where condition:      | WHERE customerNumber=103 LIMIT 1 |
+--------------------------+----------------------------------+
Running first batch
+---+--------------+------------+--------------+------------+--------------+
| # | Table        | Rows found | Rows deleted | Time spent | Parent Table |
+---+--------------+------------+--------------+------------+--------------+
| 0 | customers    | 1          | 1            | 0s         |              |
| 1 | orders       | 3          | 3            | 0.0s       | customers    |
| 2 | orderdetails | 7          | 7            | 0.0s       | orders       |
| 3 | payments     | 3          | 3            | 0.0s       | customers    |
+---+--------------+------------+--------------+------------+--------------+
Wed May 18 16:48:40 2016 Batch completed. Execution time: 0s - Items deleted :1
Taking a nap:5s
Running next batch
Wed May 18 16:48:45 2016 Batch completed. Execution time: 0s - Items deleted :None

 Program ended
 Bye
```

## Known issues:
* This tool works well with simple hierarchical data where root_table does not have any foreign key.
* It uses IN() operator to perform delete operation, to reduce impact on production server use small value with ```--limit``` option, here is related bug http://bugs.mysql.com/bug.php?id=68046

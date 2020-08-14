sudo sysbench --test=/usr/local/share/sysbench/tests/include/oltp_legacy/oltp.lua --db-driver=mysql --mysql-db=loadtest --mysql-user=root --mysql-password=Huawei@123 --mysql-port=3306 --mysql-host=localhost --oltp-tables-count=10 --oltp-table-size=10000 --num-threads=20 prepare




sudo sysbench --test=/usr/local/share/sysbench/tests/include/oltp_legacy/insert.lua --db-driver=mysql --mysql-db=loadtest --mysql-user=root --mysql-password=Huawei@123 --mysql-port=3306 --mysql-host=localhost --oltp-tables-count=10 --oltp-table-size=1000 --max-time=3600 --max-requests=0 --num-threads=10 --report-interval=3 --rate=20 --forced-shutdown=1 run


top

iostat -d vda vdb -m 1 10

GRANT ALL PRIVILEGES ON *.* TO 'huawei'@'%'

GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, RELOAD, PROCESS, REFERENCES, INDEX, ALTER, SHOW DATABASES, CREATE TEMPORARY TABLES, LOCK TABLES, EXECUTE, REPLICATION SLAVE, REPLICATION CLIENT, CREATE VIEW, SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, CREATE USER, EVENT, TRIGGER ON *.* TO 'huawei'@'%'



sudo sysbench --test=/usr/local/share/sysbench/tests/include/oltp_legacy/insert.lua --db-driver=mysql --mysql-db=loadtest --mysql-user=root --mysql-password=Seetraum123@ --mysql-port=3306 --mysql-host=192.168.0.53 --oltp-tables-count=10 --oltp-table-size=1000 --max-time=3600 --max-requests=0  --num-threads=20 --report-interval=3 --rate=20 --forced-shutdown=1 run

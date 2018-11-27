
<!-- toc --> 

* * * * *

参考： http://www.thecompletelistoffeatures.com/

## Replication
1. Multi source replication [1]
2. Online GTID migration path [1 2 3]
3. Improved semi-sync performance [1 2]
4. Loss-less semi-sync replication [1 2]
5. Semi-sync can now wait for a configurable number of slaves [1]
6. Intra-schema parallel replication [1]
7. Ability to tune group commit via binlog_group_commit_sync_delay and binlog_group_commit_sync_no_delay_count options. [1 2]
8. Non-blocking SHOW SLAVE STATUS [1 2]
9. Online CHANGE REPLICATION FILTER [1]
10. Online CHANGE MASTER TO without stopping SQL thread [1]
11. Multi-threaded slave ordered commits (Sequential Consistency) [1]
12. Support SLAVE_TRANSACTION_RETRIES in multi-threaded slave mode [1]
13. A WAIT_FOR_EXECUTED_GTID_SET function has been introduced [1 2]
14. Optimize GTIDs for Passive Slaves [1 2]
15. GTID Replication no longer requires log-slave-updates be enabled
16. XA Support when the binary log is turned on [1]
17. GTIDs in the OK packet [1]
18. Better synchronization between dump and user threads when racing for the binlog [1]
19. Improved memory management of Binlog_sender [1]
20. Option to suppress "unsafe for binlog" messages in error log [1]
21. Defaults change: binlog_format=ROW
22. Defaults change: sync_binlog=1
23. Defaults change: binlog_gtid_simple_recovery=1
24. Defaults change: binlog_error_action=ABORT_SERVER
25. Defaults change: slave_net_timeout=60



## InnoDB
1. Online buffer pool resize [1]
2. Improved crash recovery performance [1]
3. Improved read-only transaction scalability [1 2 3 4]
4. Improved read-write transaction scalability [1 2 3 4]
5. Several optimizations for high performance temporary tables [1 2 3 4 5]
6. ALTER TABLE RENAME INDEX only requires meta-data change [1]
7. Increasing VARCHAR size only requires meta-data change [1]
8. ALTER TABLE performance improved [1 2]
9. Multiple page_cleaner threads [1]
10. Optimized buffer pool flushing [1]
11. New innodb_log_checksum_algorithm option [1]
12. Improved NUMA support [1]
13. General Tablespace support [1]
14. Transparent Page Compression [1]
15. innodb_log_write_ahead_size introduced to address potential 'read-on-write' with redo logs [1]
16. Fulltext indexes now support pluggable parsers [1]
17. Support for ngram and MeCab full-text parser plugins [1 2]
18. Fulltext search optimizations [1]
19. Buffer pool dump now supports innodb_buffer_pool_dump_pct [1]
20. The doublewrite buffer is now disabled on filesystems that supports atomic writes (aka Fusion-io support) [1]
21. Page fill factor is now configurable [1]
22. Support for 32K and 64K pages [1]
23. Online undo log truncation [1]
24. Update_time meta data is now updated [1]
25. TRUNCATE TABLE is now atomic [1]
26. Memcached API performance improvements [1]
27. Adaptive hash scalability improvements [1]
28. InnoDB now implements information_schema.files [1]
29. Legacy InnoDB monitor tables have been removed or replaced by global configuration settings
30. InnoDB default row format now configurable [1]
31. InnoDB now drops tables in a background thread [1]
32. InnoDB tmpdir is now configurable [1]
33. InnoDB MERGE_THRESHOLD is now configurable [1]
34. InnoDB page_cleaner threads get priority using setpriority()[1]
35. Defaults change: innodb_file_format=Barracuda
36. Defaults change: innodb_large_prefix=1
37. Defaults change: innodb_page_cleaners=4
38. Defaults change: innodb_purge_threads=4
39. Defaults change: innodb_buffer_pool_dump_at_shutdown=1
40. Defaults change: innodb_buffer_pool_load_at_startup=1
41. Defaults change: innodb_buffer_pool_dump_pct=25
42. Defaults change: innodb_strict_mode=1
43. Defaults change: innodb_checksum_algorithm=crc32
44. Defaults change: innodb_default_row_format=DYNAMIC
## Optimizer
1. Improved optimizer cost model, leading to more consistently better query plans [1 2 3 4]
2. Optimizer cost constants are now configurable on a global or per engine basis [1 2]
3. Query parser has been refactored and improved [1]
4. EXPLAIN FOR CONNECTION [1]
5. UNION ALL does not use a temporary table [1 2 3 4]
6. Filesort is now optimized to pack values [1]
7. Subqueries in FROM clause can now be handled same as a view (derived_merge) [1]
8. Queries using row value constructors are now optimized [1 2]
9. Optimizer now supports a new condition filtering optimization [1 2]
10. EXPLAIN FORMAT=JSON now shows cost information [1]
11. Support for STORED and VIRTUAL generated columns (aka functional indexes) [1]
12. Prepared statements refactored internally and performance improved [1 2]
13. New query hints using comment-like /*+ */ syntax [1]
14. Server-side query rewrite framework [1]
15. ONLY_FULL_GROUP_BY now more standards compliant [1]
16. Support for gb18030 character set [1]
17. Improvements to Dynamic Range access [1]
18. Memory used by the range optimizer is now configurable [1]
19. Defaults change: internal_tmp_disk_storage_engine=INNODB [1]
20. Defaults change: eq_range_index_dive_limit=200
21. Defaults change: sql_mode=ONLY_FULL_GROUP_BY, STRICT_TRANS_TABLES, NO_ZERO_IN_DATE, NO_ZERO_DATE, ERROR_FOR_DIVISION_BY_ZERO, NO_AUTO_CREATE_USER, NO_ENGINE_SUBSTITUTION
22. Defaults change: optimizer_switch=condition_fanout_filter=on, derived_merge=on
23. Defaults change: EXTENDED and PARTITIONS keywords for EXPLAIN enabled by default
## Security
1. Username size increased to 32 characters [1]
2. Support for IF [NOT] EXISTS clause in CREATE/DROP USER [1]
3. Server option to require secure transport [1]
4. Support for multiple AES Encryption modes [1 2]
5. Support for TLSv1.2 (with OpenSSL) and TLSv1.1 (with YaSSL) [1 2]
6. Support to LOCK/UNLOCK user accounts [1 2]
7. Support for password expiration policy [1 2]
8. Password strength enforcement
9. test database no longer created on installation
10. Anonymous users no longer created on installation
11. Random password generated by default on installation
12. New ALTER USER command
13. SET password='' now accepts a password instead of hash
14. Server now generates SSL keys by default
15. Insecure old_password hash removed [1]
16. Ability to create utility users for stored programs that can not login [1]
17. mysql.user.password field renamed as authentication_string to better describe its current usage.
18. Support for tablespace encryption [1]
## Performance Schema
1. Scalable memory allocation [1]
2. Overhead has been reduced in client connect/disconnect phases
3. Memory footprint has been reduced
4. pfs_lock implementation has been improved
5. Table IO statistics are now batched for improved performance
6. Memory usage instrumentation
7. Stored programs instrumentation
8. Replication slave instrumentation
9. Metadata Locking (MDL) Instrumentation
10. Transaction instrumentation
11. Prepared Statement instrumentation
12. Stage Progress instrumentation
13. SX-lock and rw_lock instrumentation
14. Thread status and variables
15. Defaults change: performance-schema-consumer-events_statements_history=ON
## GIS
1. InnoDB supports indexing of spatial datatypes [1]
2. Consistent naming scheme for GIS functions [1]
3. GIS has been refactored internally and is now based on Boost Geometry [1]
4. Geohash functions [1 2]
5. GeoJSON functions [1 2]
6. Functions: ST_Distance_Sphere, ST_MakeEnvelope, ST_IsValid, ST_Validate, ST_Simplify, ST_Buffer and ST_IsSimple [1 2]
## Triggers
1. Multiple triggers per event per table [1]
2. BEFORE Triggers are not processed for NOT NULL columns [1]
## Partitioning
1. Index condition pushdown optimization now supported
2. HANDLER command is now supported
3. WITHOUT VALIDATION option now supported for ALTER TABLE ... EXCHANGE PARTITION
4. Support for Transportable Tablespaces
5. Partitioning is now storage-engine native for InnoDB
## SYS (new)
1. SYS schema bundled by default [1 2]
2. 100 new views, 21 new stored functions and 26 new stored procedures to help understand and interact with Performance Schema and Information Schema [1]
## JSON (new)
1. Native JSON Data Type [1]
2. JSON Comparator
3. Short-hand JSON_EXTRACT operator (field->"json_path") [1]
4. New Document Store (5.7.12)
5. Functions: JSON_ARRAY, JSON_MERGE, JSON_OBJECT for creating JSON values [1]
6. Functions: JSON_CONTAINS, JSON_CONTAINS_PATH, JSON_EXTRACT, JSON_KEYS, JSON_SEARCH for searching JSON values [1]
7. Functions: JSON_ARRAY_APPEND, JSON_ARRAY_INSERT, JSON_INSERT, JSON_QUOTE, JSON_REMOVE, JSON_REPLACE, JSON_UNSET, JSON_UNQUOTE to modify JSON values [1]
8. Functions: JSON_DEPTH, JSON_LENGTH, JSON_TYPE, JSON_VALID to return JSON value attributes [1]
## Client Programs
1. New mysqlpump utility [1]
2. The mysql client now supports Ctrl+C to clear statement buffer
3. rewrite-db option added to mysqlbinlog [1 2]
4. New mysql_ssl_rsa_setup utility to help set up SSL [1]
5. SSL support added to mysqlbinlog
6. Idempotent mode added to mysqlbinlog
7. Client --ssl changed to now force SSL
8. Enhancements to the innochecksum utility
9. Removal of several outdated/unsafe command line utilities [1]
10. Many Perl command line clients converted to C++
11. Client Side Protocol Tracing
12. Client API method to reset connection
13. New MySQL Shell (mysqlsh) (separate download)
## libmysqlclient
1. Restricted export functions to documented MySQL C API
2. Added pkg-config support
3. Removed _r symlinks
4. Changed so version to 20 (from 18)
## Building
1. Compiler switched to GCC on Solaris
2. MySQL now compiles with Bison 3 (* change also backported)
3. CMake now used to compile on all platforms
4. Unneeded CMake checks have been removed (and unused macros removed from source files)
5. Build support for gcc, clang and MS Studio
## Misc
1. Server new connection throughput improved considerably [1]
2. mysql_install_db replaced by mysqld --initialize [1]
3. Native support for syslog [1 2]
4. Native support for systemd
5. disabled_storage_engines option allows a block list of engines
6. SET GLOBAL offline_mode=1 [1 2]
7. super_read_only option [1]
8. Detect transaction boundaries
9. Server version token and check [1
10. SELECT GET_LOCK() can now acquire multiple locks [1 2]
11. Configurable maximum statement execution time on a global and per query basis [1 2]
12. Better handling of connection id rollover
13. DTrace support [1]
14. More consistent IGNORE clause and STRICT mode
15. A number of tables in the mysql schema have moved from MyISAM to InnoDB
16. Server error log format improved to be consistent
17. Extract query digest moved from performance_schema into the server directly
18. Improved scalability of meta data locking
19. Increased control over error log verbosity
20. Stacked Diagnostic Areas
21. The server now supports a "SHUTDOWN" command
22. Removed support for custom atomics implementation
23. Removed "unique option prefix support" from server and utilities, which allowed options to be configured using multiple names.
24. Removed unsafe ALTER IGNORE TABLE functionality. Syntax remains for compatibility
25. Removed unsafe INSERT DELAYED functionality. Syntax remains for compatibility
26. Removed of outdated sql-bench scripts in distributions
27. Removal of ambiguous YEAR(2) datatype
28. Defaults change: log_warnings=2
29. Defaults change: table_open_cache_instances=16
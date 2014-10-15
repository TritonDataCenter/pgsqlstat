# pgsql tools

This repo contains a few tools for monitoring Postgres in production:

* [pgsqlstat](#pgsqlstat): report top-level postgres stats
* [pgsqlslower](#pgsqlslower): print details about queries taking longer
  than N milliseconds
* [pgsqlslowest](#pgsqlslowest): print details about N slowest queries
* [pgsqllat](#pgsqllat): print details about query latency distribution

All of these use Postgres's built-in DTrace probes under the hood, which means:

* You don't have to reconfigure or restart Postgres to start using them.
* They instrument all Postgres processes visible on the system.
* These tools require privileges to use DTrace on postgres processes on your
  system.  If you don't see any output (or see all zeroes) but you think your
  database is doing work, check whether you have the right privileges.

The output format for all of these tools may change.

For more information about Postgres's DTrace support, see the [Dynamic
Tracing](http://www.postgresql.org/docs/current/static/dynamic-trace.html)
chapter in the Postgres manual.

The Postgres Wiki has more information about [monitoring
Postgres](https://wiki.postgresql.org/wiki/Monitoring) and watching [slow
queries](https://wiki.postgresql.org/wiki/Logging_Difficult_Queries).  The
tools here exist because they're completely standalone and don't require
reconfiguring or even restarting Postgres in order to use them.


# <a name="pgsqlstat">pgsqlstat</a>: report top-level postgres stats

    pgsqlstat NSECONDS

Prints out stats every NSECONDS about all postgresql instances visible on this
system.  Use CTRL-C to stop.

Here's an example run on a postgres database that started idle and then had
"pgbench" run against it for 10 seconds:

    $ pgsqlstat 1
       QS    QD  TxnS  TxCm TxAb   XLI Sort    BufRd   BufWd  WBD
        0     0     0     0    0     0    0        0       0    0 
     1897  1888   293   283    0  2029    0    11262       0    0 
     7426  7426  1061  1061    0  7893    0    34121       0    0 
     8058  8058  1151  1151    0  8606    0    39672       0    0 
     3741  3740   537   537    0  3999    0    17796       0    0 
     7546  7546  1078  1078    0  8064    0    36109       0    0 
     7601  7602  1086  1086    0  8143    0    36483       0    0 
     6881  6880   983   983    0  7321    0    33734       0    0 
     4972  4972   710   710    0  5307    0    24645       0    0 
     7163  7165  1024  1024    0  7636    0    33960       0    0 
     7562  7560  1080  1080    0  8070    0    30905       0    0 
     3230  3240   458   468    0  3475    0    13251       0    0 
        0     0     0     0    0     0    0        0       0    0 
        0     0     0     0    0     0    0        0       0    0 
        0     0     0     0    0     0    0        0       0    0 
        0     0     0     0    0     0    0        0       0    0 
    ^C

Output columns:

    QS          Queries started

    QD          Queries completed (both successes and failures)

    TxnS        Transactions started

    TxCm        Transactions committed

    TxAb        Transactions aborted

    XLI         WAL records inserted into the transaction log

    Srt         Sort operations completed

    BufRd       Buffer read operations completed

    BufWd       Dirty buffers written.  If this number is frequently non-zero,
                that usually indicates that shared_buffers or the bgwriter
                control parameters should be tuned.  See the Postgres manual for
                details.

    WBD         Dirty WAL buffers written.  If this number is frequently
                non-zero, that may indicate that wal_buffers needs to be tuned.
                See the Postgres manual for details.


# <a name="pgsqlslower">pgsqlslower</a>: print details about slow queries

    pgsqlslower NMILLISECONDS

Print details about queries taking longer than NMILLISECONDS from start to
finish on all postgresql instances on this system.  Note that since this tool
traces query start to query done, it will never see queries taking longer than
NMILLISECONDS seconds.

Here's an example running this with a 40ms threshold during a "pgbench" run:

    $ pgsqlslower 40
    QUERY: UPDATE pgbench_tellers SET tbalance = tbalance + 1763 WHERE tid = 8;
       total time:    42650 us (parse/plan/execute = 16us/62us/42519us)
             txns: 0 started, 0 committed, 0 aborted
          buffers: 20 read (20 hit, 0 missed), 0 flushed

    QUERY: UPDATE pgbench_tellers SET tbalance = tbalance + 4281 WHERE tid = 3;
       total time:    49118 us (parse/plan/execute = 18us/80us/48950us)
             txns: 0 started, 0 committed, 0 aborted
          buffers: 18 read (18 hit, 0 missed), 0 flushed

    QUERY: END;
       total time:   233183 us (parse/plan/execute = 9us/0us/5us)
             txns: 0 started, 1 committed, 0 aborted
          buffers: 0 read (0 hit, 0 missed), 0 flushed

This shows that there were three queries that took longer than 40ms.  The tool
prints out the time required to parse, plan, and execute each query; the number
of transactions started, committed, or aborted; and the number of internal
postgres buffers read and flushed.


# <a name="pgsqlslowest">pgsqlslowest</a>

    pgsqlslowest NSECONDS [MAXQUERIES]

Traces all queries for NSECONDS seconds on all postgresql instances on this
system and prints the MAXQUERIES slowest queries.  Note that because this tool
traces from query start to query completion, it will never see queries that
take longer than NSECONDS to complete.

For example, tracing the 15 slowest queries over five seconds while running "pgbench":

    $ pgsqlslowest 5 15

      queries failed                                                    0
      queries ok                                                    35702
      queries started                                               35711
      elapsed time (us)                                           5006660

    Slowest queries: latency (us):

         30393  UPDATE pgbench_tellers SET tbalance = tbalance + -2025 WHERE tid = 8;
         30774  UPDATE pgbench_tellers SET tbalance = tbalance + -3603 WHERE tid = 3;
         32275  UPDATE pgbench_tellers SET tbalance = tbalance + 152 WHERE tid = 10;
         33840  UPDATE pgbench_branches SET bbalance = bbalance + 1282 WHERE bid = 1;
         34364  UPDATE pgbench_tellers SET tbalance = tbalance + 4838 WHERE tid = 4;
         35507  UPDATE pgbench_branches SET bbalance = bbalance + 2263 WHERE bid = 1;
         36241  UPDATE pgbench_tellers SET tbalance = tbalance + 3174 WHERE tid = 2;
         36260  UPDATE pgbench_tellers SET tbalance = tbalance + -4164 WHERE tid = 5;
         36911  UPDATE pgbench_tellers SET tbalance = tbalance + 4609 WHERE tid = 7;
         37583  UPDATE pgbench_branches SET bbalance = bbalance + -4310 WHERE bid = 1;
         42188  UPDATE pgbench_tellers SET tbalance = tbalance + -922 WHERE tid = 5;
         45902  UPDATE pgbench_tellers SET tbalance = tbalance + 2607 WHERE tid = 6;
         46313  UPDATE pgbench_tellers SET tbalance = tbalance + -2672 WHERE tid = 5;
        118198  BEGIN;
       2083346  END;

# <a name="pgsqllat">pgsqllat</a>

    pgsqllat NSECONDS

Traces all queries for NSECONDS seconds on all postgresql instances on this
system and prints distributions of query latency over time and overall query
latency.  Note that because this tool traces from query start to query
completion, it will never see queries that take longer than NSECONDS to
complete.

For example, tracing queries over ten seconds while running "pgbench":

    $ pgsqllat 10

      elapsed time (us)                                          10001952
      queries failed                                                    0
      queries ok                                                    73301
      queries started                                               73311

    All queries: latency (in microseconds), per second:

                  key  min .------------------. max    | count
           1413409840    4 : ▁▃▁▅█▃▁▂▂▂▂▁▁    : 524288 | 2459
           1413409841    4 : ▁▃▁▆█▃▁▂▂▂▁▁▁    : 524288 | 8597
           1413409842    4 : ▂▃▁▅█▂▁▂▂▂▁▁  ▁  : 524288 | 7420
           1413409843    4 : ▁▃▁▄█▂▁▂▂▂▁▁▁    : 524288 | 8141
           1413409844    4 : ▂▃▁▅█▂▁▂▂▂▁▁     : 524288 | 8714
           1413409845    4 : ▁▃▁▅█▂▁▂▂▂▁▁     : 524288 | 8730
           1413409846    4 : ▁▃▁▃█▃▁▂▂▂▂▁▁    : 524288 | 7474
           1413409847    4 :  ▃▁▂█▄▁▂▂▂▂▁▁    : 524288 | 6744
           1413409848    4 : ▁▃▁▄█▃▁▂▂▂▁▁▁    : 524288 | 8063
           1413409849    4 :  ▃▂▂█▆▁▂▂▂▂▁▁ ▁▁ : 524288 | 4695
           1413409850    4 :  ▃▁▃█▄▁▂▂▂▁▁▁▁▁  : 524288 | 2264

    All queries: latency (in microseconds), over all 10 seconds:

               value  ------------- Distribution ------------- count    
                   9 |                                         0        
                  10 |@@@@                                     2669     
                  20 |@@@@@@@@@                                6102     
                  30 |@@                                       1431     
                  40 |                                         75       
                  50 |                                         57       
                  60 |                                         36       
                  70 |@                                        810      
                  80 |@@                                       1335     
                  90 |@@@@                                     2764     
                 100 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@   25431    
                 200 |@@@@@@@@@@@@@@@@@@@                      12950    
                 300 |@@@@                                     2380     
                 400 |@                                        391      
                 500 |                                         179      
                 600 |                                         147      
                 700 |                                         160      
                 800 |                                         206      
                 900 |                                         326      
                1000 |@@@@@@                                   4349     
                2000 |@@@@                                     3009     
                3000 |@@@                                      2013     
                4000 |@@                                       1509     
                5000 |@@                                       1121     
                6000 |@                                        841      
                7000 |@                                        651      
                8000 |@                                        522      
                9000 |@                                        390      
               10000 |@@                                       1243     
               20000 |                                         147      
               30000 |                                         21       
               40000 |                                         3        
               50000 |                                         2        
               60000 |                                         1        
               70000 |                                         0        
               80000 |                                         0        
               90000 |                                         0        
              100000 |                                         20       
              200000 |                                         10       
              300000 |                                         0        

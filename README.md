# pgsql tools

This repo contains a few tools for monitoring Postgres in production:

* [pgsqlstat](#pgsqlstat): report top-level postgres stats
* [pgsqlslower](#pgsqlslower): print details about *queries* taking longer
  than N milliseconds
* [pgsqlslowest](#pgsqlslowest): print details about N slowest queries
* [pgsqllat](#pgsqllat): print details about query latency distribution
* [pgsqltxslower](#pgsqltxslower): print details about *transactions* taking
  longer than N milliseconds
* [pglockwaits](#pglockwaits): print counts of events where Postgres blocked
  waiting for a lock

All of these use Postgres's built-in DTrace probes under the hood, which means:

* You don't have to reconfigure or restart Postgres to start using them.
* They instrument all Postgres processes visible on the system.
* These tools require privileges to use DTrace on postgres processes on your
  system.  If you don't see any output (or see all zeroes) but you think your
  database is doing work, check whether you have the right privileges.

These tools are a work in progress.  The output format for all of these tools
may change.

For more information about Postgres's DTrace support, see the [Dynamic
Tracing](http://www.postgresql.org/docs/current/static/dynamic-trace.html)
chapter in the Postgres manual.

The Postgres Wiki has more information about [monitoring
Postgres](https://wiki.postgresql.org/wiki/Monitoring) and watching [slow
queries](https://wiki.postgresql.org/wiki/Logging_Difficult_Queries).  The
tools here exist because they're completely standalone and don't require
reconfiguring or even restarting Postgres in order to use them.

# The tools

## <a name="pgsqlstat">pgsqlstat</a>: report top-level postgres stats

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


## <a name="pgsqlslower">pgsqlslower</a>: print details about slow queries

    pgsqlslower NMILLISECONDS

Print details about queries taking longer than NMILLISECONDS from start to
finish on all postgresql instances on this system.  Note that since this tool
traces query start to query done, it will never see queries taking longer than
the command has been running.

This is similar to pgsqltxslower, except that it's tracing simple queries.  For
extended queries that are part of transactions, see pgsqltxslower.

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


## <a name="pgsqlslowest">pgsqlslowest</a>: print details about N slowest queries

    pgsqlslowest NSECONDS [MAXQUERIES]

Traces all queries for NSECONDS seconds on all postgresql instances on this
system and prints the MAXQUERIES slowest queries.  Note that because this tool
traces from query start to query completion, it will never see queries that
take longer than the command has been running.

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

## <a name="pgsqllat">pgsqllat</a>: print details about query latency

    pgsqllat NSECONDS

Traces all queries for NSECONDS seconds on all postgresql instances on this
system and prints distributions of query latency over time and overall query
latency.  Note that because this tool traces from query start to query
completion, it will never see queries that take longer than the command has been
running.

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

## <a name="pgsqltxslower">pgsqltxslower</a>: print details about slow transactions

    pgsqltxslower NMILLISECONDS

Print details about transactions taking longer than NMILLISECONDS from start to
finish on all postgresql instances on this system.  Note that since this tool
traces transaction start to commit/abort, it will never see transactions taking
longer than the command has been running.

This is similar to pgsqlslower, except that it's tracing transactions, which
may be made up of multiple queries.

Here's example output from tracing "pgbench" again:

           TIME    PID  GAP(ms) ELAP(ms) EVENT
       6920.155  78403    0.003    0.000 transaction-start 
       6920.178  78403    0.021    0.005 query-parse        BEGIN;
       6920.193  78403    0.003    0.002 query-rewrite     
       6920.212  78403    0.005    0.003 query-execute      0 buffer reads, 0 internal sorts, 0 external sorts
       6920.275  78403    0.042    0.011 query-parse        UPDATE pgbench_accounts SET abalance = abalance + -161 WHERE aid = 27491;
       6920.310  78403    0.004    0.021 query-rewrite     
       6920.376  78403    0.003    0.052 query-plan        
       6920.495  78403    0.006    0.100 query-execute      5 buffer reads, 0 internal sorts, 0 external sorts
       6920.558  78403    0.043    0.011 query-parse        SELECT abalance FROM pgbench_accounts WHERE aid = 27491;
       6920.589  78403    0.004    0.015 query-rewrite     
       6920.638  78403    0.003    0.035 query-plan        
       6920.686  78403    0.016    0.020 query-execute      4 buffer reads, 0 internal sorts, 0 external sorts
       6920.755  78403    0.048    0.011 query-parse        UPDATE pgbench_tellers SET tbalance = tbalance + -161 WHERE tid = 7;
       6920.789  78403    0.004    0.018 query-rewrite     
       6920.846  78403    0.003    0.043 query-plan        
       6920.914  78403    0.005    0.050 query-execute      3 buffer reads, 0 internal sorts, 0 external sorts
       6920.978  78403    0.042    0.011 query-parse        UPDATE pgbench_branches SET bbalance = bbalance + -161 WHERE bid = 1;
       6921.011  78403    0.004    0.017 query-rewrite     
       6921.064  78403    0.003    0.039 query-plan        
       6926.959  78403    0.005    5.877 query-execute      14 buffer reads, 0 internal sorts, 0 external sorts
       6927.066  78403    0.072    0.022 query-parse        INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (7, 1, 27491, -161, CURRENT_TIMESTAMP);
       6927.113  78403    0.006    0.028 query-rewrite     
       6927.145  78403    0.004    0.015 query-plan        
       6927.201  78403    0.006    0.036 query-execute      1 buffer reads, 0 internal sorts, 0 external sorts
       6927.257  78403    0.042    0.005 query-parse        END;
       6927.271  78403    0.003    0.001 query-rewrite     
       6927.288  78403    0.004    0.002 query-execute      0 buffer reads, 0 internal sorts, 0 external sorts
       6929.587  78403    2.292    9.435 transaction-commit

This transaction took 9.4ms, of which 5.8ms was spent executing the longest
query.

All timestamps in this output are in milliseconds.  Columns include:

* TIME: milliseconds since the script started tracing
* PID: postgres worker process id
* GAP: time between this event and the previous one (see below)
* ELAP: elapsed time for this event.  For transaction-commit, this is the time
  for the whole transaction.
* EVENT: describes what happened

This tool breaks transactions into a few discrete operations: query parsing,
rewriting, planning, and execution.  There may be multiple queries processed in
a transaction, and each query may skip one or more of these phases.  Moreover,
there are gaps between these operations.  Most notable are gaps between
query-execute and query-parse, which is presumably time that postgres spent
waiting for the client to send the next query for parsing.  So if you take these
two lines:

       6926.959  78403    0.005    5.877 query-execute      14 buffer reads, 0 internal sorts, 0 external sorts
       6927.066  78403    0.072    0.022 query-parse        INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (7, 1, 27491, -161, CURRENT_TIMESTAMP);

The first line denotes execution of the *previous* query.  Execution took 5.8ms
and included 14 buffer reads and no sorts.  Then it was 72 microseconds before
postgres started parsing the next command.  Parsing the command took 22
microseconds.

**Caveat:** Even when using the "temporal" DTrace option, it's possible for
events to be printed out of order.  This is annoying, but one way to work around
it is to:

1. Save the output to a file.
2. Find the transaction of interest (usually based on the latency for the
   transaction-commit event).  Find the PID.
3. Select only the lines of the file for that PID and sort them by timestamp:

    awk '{ $2 == YOURPID }' < YOURFILE | sort -n -k1,1

   Then the events for all transactions handled by that process will be in
   order and you can find all the events for the transaction you're interested
   in.


## <a name="pglockwaits">pglockwaits</a>: print counts of lock wait events

    pglockwaits NSECONDS

Prints out stats every NSECONDS about all cases where a postgresql backend
blocked waiting for a lock.  The results are broken out by lock type.  Use
CTRL-C to stop.

Here's an example run on a postgres database that started idle and then had
load applied for a few seconds:

    $ pglockwaits 1
                            AS    RS    RX   SUX     S   SRX     X    AX
    2015 Aug 13 21:19:27     0     0     0     0     0     0     0     0
    2015 Aug 13 21:19:28     0     0     0     0     0     0     0     0
    2015 Aug 13 21:19:29     0     0     0     0     0     0     0     0
    2015 Aug 13 21:19:30     0     0     0     0   591     0    22     0
    2015 Aug 13 21:19:31     0     0     0     0 16966     0   165     0
    2015 Aug 13 21:19:32     0     0     0     0 13762     0    82     0
    2015 Aug 13 21:19:33     0     0     0     0 19739     0    95     0


Output columns correspond to lock types:

    AS		ACCESS SHARE

    RS		ROW SHARE

    RX		ROW EXCLUSIVE

    SUX		SHARE UPDATE EXCLUSIVE

    S		SHARE

    SRX		SHARE ROW EXCLUSIVE

    X		EXCLUSIVE

    AX		ACCESS EXCLUSIVE

See http://www.postgresql.org/docs/current/static/explicit-locking.html for more
information about these.


# Implementation notes

The relationship between queries and transactions is not as simple as it
seems.  If you imagine a simple psql(1) command-line session: unless the user
types the "BEGIN" command, each query maps directly to one transaction, and
we'd expect to see these probes each time you hit "enter" with a valid,
complete SQL line:

    query-start
    transaction-start
    transaction-commit
    query-done

This makes sense.  If the query fails (e.g., because of bad syntax), we might
instead see:

    query-start
    transaction-start
    transaction-abort

Note the conspicuous absence of a "query-done" probe.  Trickier, but
manageable.

Now suppose you run this three-command sequence:

    BEGIN;
    select NOW();
    COMMIT;

We see these probes:

    query-start              querystring = "BEGIN;"
    transaction-start
    query-done

    query-start              querystring = "select NOW();"
    query-done

    query-start              querystring = "COMMIT;"
    transaction-commit
    query-done

And if instead you run this three-command sequence:

    BEGIN;
    select NOW();
    ABORT;

We see these probes:

    query-start              querystring = "BEGIN;"
    transaction-start
    query-done

    query-start              querystring = "select NOW();"
    query-done

    query-start              querystring = "ABORT;"
    transaction-abort
    query-done

Unlike the syntax error case above, we get a query-done for the "ABORT"
command.

Finally, all of the above assumes the simple query protocol.  Clients can
also use extended queries, in which case we may not see query-start for
several of the intermediate queries.  We will typically see query-parse and
query-execute, though.

To summarize: what's subtle about this is that:

* query strings are only associated with queries, not transactions, and so
  are available only at query-start
* query-start probes may happen inside of a transaction or they may end up
  being part of a subsequently-created transaction

As a result, the total transaction time is measured from transaction-start to
transaction-{commit or abort}, but if we want to keep track of the queries
run during a transaction, we have to keep track of query-start firings from
before the transaction started.

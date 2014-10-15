# pgsqlstat: report top-level postgres stats

## Synopsis

    pgsqlstat NSECONDS

Prints out stats about all postgresql instances visible on this system.

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

This tool requires privileges to use DTrace on postgres processes on this
system.  If you see all zeroes but expect some data, check whether your user has
permissions to trace the postgres processes.

The output format is unstable and may change.

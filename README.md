# pgsqlstat: report top-level postgres stats

## Synopsis

    pgsqlstat NSECONDS

Prints out stats about all postgresql instances visible on this system.
Output columns:

    QS		Queries started

    QD		Queries completed (both successes and failures)

    TxnS	Transactions started

    TxCm	Transactions committed

    TxAb	Transactions aborted

    XLI		WAL records inserted into the transaction log

    Srt		Sort operations completed

    BufRd	Buffer read operations completed

    BufWd	Dirty buffers written.  If this number is frequently non-zero,
    		that usually indicates that shared_buffers or the bgwriter
		control parameters should be tuned.  See the Postgres manual for
		details.

    WBD		Dirty WAL buffers written.  If this number is frequently
    		non-zero, that may indicate that wal_buffers needs to be tuned.
		See the Postgres manual for details.

This tool requires privileges to use DTrace on postgres processes on this
system.  If you see all zeroes but expect some data, check whether your user has
permissions to trace the postgres processes.

The output format is unstable and may change.

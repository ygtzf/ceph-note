# Ceph PG 状态说明

### 代码
osd/osd_types.h:
```
/*
 * pg states
 */
#define PG_STATE_CREATING     (1<<0)  // creating
#define PG_STATE_ACTIVE       (1<<1)  // i am active.  (primary: replicas too)
#define PG_STATE_CLEAN        (1<<2)  // peers are complete, clean of stray replicas.
#define PG_STATE_DOWN         (1<<4)  // a needed replica is down, PG offline
#define PG_STATE_REPLAY       (1<<5)  // crashed, waiting for replay
//#define PG_STATE_STRAY      (1<<6)  // i must notify the primary i exist.
#define PG_STATE_SPLITTING    (1<<7)  // i am splitting
#define PG_STATE_SCRUBBING    (1<<8)  // scrubbing
#define PG_STATE_SCRUBQ       (1<<9)  // queued for scrub
#define PG_STATE_DEGRADED     (1<<10) // pg contains objects with reduced redundancy
#define PG_STATE_INCONSISTENT (1<<11) // pg replicas are inconsistent (but shouldn't be)
#define PG_STATE_PEERING      (1<<12) // pg is (re)peering
#define PG_STATE_REPAIR       (1<<13) // pg should repair on next scrub
#define PG_STATE_RECOVERING   (1<<14) // pg is recovering/migrating objects
#define PG_STATE_BACKFILL_WAIT     (1<<15) // [active] reserving backfill
#define PG_STATE_INCOMPLETE   (1<<16) // incomplete content, peering failed.
#define PG_STATE_STALE        (1<<17) // our state for this pg is stale, unknown.
#define PG_STATE_REMAPPED     (1<<18) // pg is explicitly remapped to different OSDs than CRUSH
#define PG_STATE_DEEP_SCRUB   (1<<19) // deep scrub: check CRC32 on files
#define PG_STATE_BACKFILL  (1<<20) // [active] backfilling pg content
#define PG_STATE_BACKFILL_TOOFULL (1<<21) // backfill can't proceed: too full
#define PG_STATE_RECOVERY_WAIT (1<<22) // waiting for recovery reservations
#define PG_STATE_UNDERSIZED    (1<<23) // pg acting < pool size
#define PG_STATE_ACTIVATING   (1<<24) // pg is peered but not yet active
#define PG_STATE_PEERED        (1<<25) // peered, cannot go active, can recover
```

### 官方解释

> ==Inactive== Placement groups cannot process reads or writes because they are waiting for an OSD with the most up-to-date data to come up and in.

>==Unclean== Placement groups contain objects that are not replicated the desired number of times. They should be recovering.

>==Stale== Placement groups are in an unknown state - the OSDs that host them have not reported to the monitor cluster in a while




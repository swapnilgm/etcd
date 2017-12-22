ETCD incremental snapshot
=========================

## Abstract
Incremental snapshot is next thing user wants to optimize his backup storage space for ETCD snapshots. ETCD incremental snapshots are possible with combination of existing full snapshots and watch features. We should extend existing command line utility `etcdctl` to add this support.

## Motivation

Backup and restore are most common feature of any production level applications. ETCD also supports backup and restore of its underlying key value store using `etcdctl snapshot` command line utility. 

This feature has limitation of having ability to take only full snapshot. For production environment where there is need of scheduled snapshots, taking full snapshot each time for shorter period, consumes lot of storage space because of data duplication among snapshots at different interval.

Also taking full snapshot involves io copy of entire `db` file, which in itself is time consuming task.

Because of this it is hard to support the features like point-in-time recovery (PITR) ultimately cause the performance parameters like RPO and RTO for applications using ETCD.

## Design

When we talk about incremental snapshot, standard approach involves, one base snapshot and multiple incremental snapshot over it, which are ordered snapshots, each having delta add on over previous snapshot. So when snapshot taking client/controller let's say __"etcdIncCtl"__ need to build the relation among these incremental snapshot and base snapshots. And hence, it requires to have complete control over the backup store.

ETCD by default have supports taking full/base snapshot. Also, it provides API's to apply watch on it, to get event updates. One can store this multiple event as part of incremental snapshot so that it can apply them on full snapshot in same order to restore entire ETCD data.

Although for 100% recovery, we need each event to be stored before crash of ETCD, in general we allow some tradeoff over the data updates between time of last snapshot and time of failure of ETCD. Based on this, there can be two way to configure these incremental snapshot:

1. Take incremental snapshot after predefined time interval
2. Take incremental snapshot after predefined count of events

Base on this algorithm can be designed as follows:

#### Saving incremental Snapshot
 
1. Receive the incremental snapshot request
    case a. Periodic incremental snapshot with new base snapshot
    case b. Periodic incremental snapshot over existing snapshot
    case c. Single incremental snapshot for existing base snapshot

2. Get to full snapshot present phase,
    - For 1.a, take new full snapshot up to latest revision
    - For 1.b, 1.c validate the existence of previous snapshots.

3. Initialize timer/counter

4. Apply watch on etcd for key-value operation events with latest revision in previous step
   
5. Process loop
    - Collect events from watch channel in step 4 in memory
    - If time based and timeout occurs, dump collection of events to incSnapshot file with metadata.
    - If event count based and event counter filled, dump collection of events to incSnapshot file with metadata.
    - If case 1.b, single incremental snapshot then, stop the loop.

#### Restoring from incremental snapshot

1. Find appropriate incremental snapshot based on time/revision number using snapshot metadata.

2. Find the dependency snapshot for specified incremental snapshot.

3. Restore from dependency base snapshot.

4. Start embedded etcd.

5. For each dependency incremental snapshot from older to newer
    - Read the events from snapshot file.
    - Apply events in order of time as normal key-value PUT/DELETE operation on embedded etcd.

6. Stop embedded etcd.

### Assumption
 - `etcdctl` has the control of backup directory (let's call it __SnapStore__) so that it can find all previous snapshots i.e . both full/base snapshots and incremental snapshots.
 
 ### SnapStore storage structure

 ```
    snapstore/ 
        |----- baseSnapshotsDir/
        |               |---- baseSnap1
        |               |---- baseSnap2
        |----- incSnapshotDir/
        |                |----- baseSnap1-incSnap1
        |                |----- baseSnap1-incSnap2
        |                |----- baseSnap2-incSnap1
 ```

 We can use one of the above way to define the storage file structure. Basically, storage structure is more about getting file path from snapshot ID. Only pros and cons of this are implementation specific complexity of different listing and file location identification functionality.

### Incremental Snapshot File Structure

The incremental snapshot file, could be simple dump of watch events aggregated over period or up to the event count. For primary implementation, we can have simple json dump with checksum hash at the end. We can have provide hook, to have different encoding later on in case community expects it, so that everybody can have there data encryption strategy.

File name can be created by combination of starting key revision and last key revision in that file and also the creation timestamp.

#### Delete snapshot

In addition to snapshot store,restore, since the `etcdctl` is managing the storage dirctory. It might need to expose option to delete the snapshot.

1. If its incremental snapshot mark it as inactive.
2. If its from latest base snapshot, do nothing.
3. If all incremental snapshot based on same base snapshot are inactive, physically delete all incremental snapshot as well base snapshot on which those depended.
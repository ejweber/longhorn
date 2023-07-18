# Disk Anti-Affinity

## Summary

Longhorn supports multiple disks per node, but there is currently no way to ensure that two replicas for the same
volume that schedule to the same node end up on different disks. In fact, the replica scheduler currently doesn't make
any attempt achieve this goal, even when it is possible to do so.

With the addition of a Disk Anti-Affinity feature, the Longhorn replica scheduler will attempt to schedule two replicas
for the same volume to different disks when possible. Optionally, the scheduler will refuse to schedule a replica to a
disk that has another replica for the same volume.

Although the comparison is not perfect, this enhancement can be thought of as enabling RAID 1 for Longhorn (mirroring
across multiple disks on the same node).

See the [Motivation section](#motivation) for potential benefits.

### Related Issues

- https://github.com/longhorn/longhorn/issues/3823
- https://github.com/longhorn/longhorn/issues/5149

### Existing Related Features

#### Replica Node Level Soft Anti-Affinity

Disabled by default. When disabled, prevents the scheduling of a replica to a node with an existing healthy replica of
the same volume.

Can also be set at the volume level to override the global default.

#### Replica Zone Level Soft Anti-Affinity

Enabled by default. When disabled, prevents the scheduling of a replica to a zone with an existing healthy replica of
the same volume.

Can also be set at the volume level to override the global default.

## Motivation

- Large, multi-node clusters will likely not benefit from this enhancement.
- Single-node clusters and small, multi-node clusters (on which the number of replicas per volume exceeds the number
  of available nodes) will experience:
  - Increased data durability. If a single disk fails, a healthy replica will still exist on an disk that
    has not failed.
  - Increased data availability. If a single disk on a node becomes unavailable, but the node itself remains
    healthy, at least one replica remains healthy. On a single-node cluster, this can directly prevent a volume crash.
    On a small, multi-node cluster, this can prevent a future volume crash due to the loss of another node.

### Goals

- In all situations, the Longhorn replica scheduler will make a best effort to ensure two replicas for the same volume
  do not schedule to the same disk.
- Optionally, the scheduler will refuse to schedule a replica to a disk that has another replica of the same volume.

## Proposal

### User Stories

#### Story 1

My cluster consists of a single node with multiple attached SSDs. When I create any new volume, I want replicas to
distribute across these disks so that I can recover from n - 1 disk failures. If there are not as many available disks
as desired replicas, I want Longhorn to do the best it can.

#### Story 2

My cluster consists of a single node with multiple attached SSDs. When I create any new volume, I want replicas to
distribute across these disks so that I can recover from n - 1 disk failure. If there are not as many available disks
as desired replicas, I want scheduling to fail obviously. It is important that I know my volumes aren't being protected
so I can take action.

#### Story 3

My cluster consists of a single node with multiple attached SSDs. When I create a specific, high-priority volume, I want
replicas to distribute across these disks so that I can recover from n - 1 disk failure. If there are not as many
available disks as desired replicas, I want scheduling to fail obviously. It is important that I know high-priority
volume isn't being protected so I can take action.

### User Experience In Detail

Detail what the user needs to do to use this enhancement. Include as much detail as possible so that people can understand the "how" of the system. The goal here is to make this feel real for users without getting bogged down.

### API changes

Introduce a new Replica Disk Level Soft Anti-Affinity setting with the following definition. By default, we set it to
true. While it is generally desirable to schedule replicas to different disks, it would break with existing behavior to 
refuse to schedule replicas when different disks are not available.

```golang
SettingDefinitionReplicaDiskSoftAntiAffinity = SettingDefinition{
    DisplayName: "Replica Disk Level Soft Anti-Affinity",
    Description: "Allow scheduling on disks with existing healthy replicas of the same volume",
    Category:    SettingCategoryScheduling,
    Type:        SettingTypeBool,
    Required:    true,
    ReadOnly:    false,
    Default:     "true",
}
```

Introduce a new 

## Design

### Implementation Overview

Overview of how the enhancement will be implemented.

### Test plan

Integration test plan.

For engine enhancement, also requires engine integration test plan.

### Upgrade strategy

Anything that requires if the user wants to upgrade to this enhancement.

## Note [optional]

Additional notes.

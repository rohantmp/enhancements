---
title: local-device-health-monitoring
authors:
  - "@rohantmp"
reviewers:
  - "@gnufied"
  - "@jsafrane”
  - "@sp98"
  
approvers:
  - "@gnufied"
  - "@jsafrane"
creation-date: 2021-07-16
last-updated: 2021-07-16
status: implementable
see-also:
replaces:
superseded-by:
---



# Local Device Health Monitoring


## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Operational readiness criteria is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

This enhancement proposes that we export health metrics for each local device on each node.

## Motivation

Local device vendors expose health information about storage devices via APIs like [SMART](https://en.wikipedia.org/wiki/S.M.A.R.T.). It very useful to know when a storage is about to fail, to differentiate between storage hardware related errors vs purely software ones, etc.

### Goals

- Expose health information about local storage devices in a way that can generate alerts

### Non-Goals

- Align with sig-storage [health monitoring for persistent volumes](https://kubernetes.io/blog/2021/04/16/volume-health-monitoring-alpha-update/)
  - justification: it is also useful to monitor system drives and CSI only deals with persitent volumes, and migrating to CSI is an extremely long term effort.

## Proposal

[Local Storage Operator](https://github.com/openshift/local-storage-operator) currently deploys multiple daemonsets and enumerates local-storage. We could export per-disk metrics from the same daemonset.


### User Stories

- As an admin, I would like to be alerted when a storage device is unhealthy and be linked to remediation steps.
- As an admin, I would like if components (such as OpenShift Data Foundation) could access storage device health and failover faster.


### Implementation Details/Notes/Constraints

`LocalVolumeDisocveryResults` already enumerate and refresh information about the disk in a per-node `LocalVolumeDiscoveryResult`, and a [PR](https://github.com/openshift/local-storage-operator/pull/249) is in progress to add metrics to each of the daemonsets that update the CR.

`LocalVolumeDiscovery` can be configured via `nodeSelector` and `toleration` to match any set of nodes or all nodes. We could have the metrics be for all nodes or just for the ones that LocalVolumeDiscovery is configured to watch. Whether this should be controlled by the same (singleton) CR is an [open question](#Open-Questions    )

We will be collecting metrics via [libstoragemgmt](https://github.com/libstorage/libstoragemgmt) and calling it via [libstoragemgmt-golang](https://github.com/libstorage/libstoragemgmt-golang/).
If a block device is a partion or logical volume, we will only expose health for the parent device.


### Risks and Mitigations

- There is some concern that this is out of scope for Local Storage Operator
  - We are discussing [alternatives](#Alternatives).
- There is some uncertainty about whether this implementation precludes other components correlating PersistentVolumes with this information.
  - Mitigation: We are investigating if it is an acceptable pattern for internal components to scrape and correlate cluster prometheus metrics for automation. There has been some indication that this is the case on #forum-monitoring.
- The golang bindings for libstoragemgmgt also needs some review and polish.
  - Mitigation: The ODF(OpenShift Data Foundation) team will contribute to and help the RHEL team maintain libstoragemgmt golang


## Design Details

### Open Questions

- Is Local Storage Operator the best place for this?
  - see [alternatives](#Alternatives)
- Should we allow configuration to watch only a subset of nodes or watch all nodes by default? Should the metrics be controller by LocalVolumeDiscovery (singleton)


### Test Plan

**Note:** *Section not required until targeted at a release.*

- E2E tests will deploy the component and verify that metrics are being emitted per device

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

#### Dev Preview -> Tech Preview

#### Tech Preview -> GA

#### Removing a deprecated feature



### Upgrade / Downgrade Strategy



### Version Skew Strategy



## Implementation History


## Drawbacks



## Alternatives

- We *could* create a new component for this.

Disqualified alternatives:
- We explored node_exporter, but it is a rootless container, so SMART data can't be checked.
- We considered node-problem-detector, but it doesn't export metrics only sets node conditions and creates events.

## Infrastructure Needed


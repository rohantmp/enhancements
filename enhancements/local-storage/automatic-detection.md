---
title: automatic-detection-of-disks-in-Local-Storage-Operator
authors:
  - "@aranjan"
reviewers:
  - "@jrivera"
  - "@jsafrane"
  - "@hekumar"
  - "@chuffman"
  - "@rojoseph"
  - "@sapillai"
approvers:
  - "@jrivera"
  - "@jsafrane"
  - "@hekumar"
  - "@chuffman"
creation-date: 2020-01-21
last-updated: 2020-01-21
status: implementable
---

# Automatic detection of Disks in Local-Storage-Operator 

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)


## Summary

When working with bare-metal OpenShift clusters, due to the absence of storage-provisioners like that in the cloud there needs a lot of manual work to be done for consuming local storage from nodes. The idea is to automate the manual that is required for consuming local-storage from the OpenShift cluster.

## Motivation

For consuming local storage from the nodes available in the OpenShift cluster a good amount of manual work has to be done for creating local PVs out of it, making hot-swapping of disks tough.

### Goals

- Automatically detection of available disks from nodes which can be used as PVs for OpenShift-cluster.
- Should respond to the attach/detach events of the disks/devices.
- Should have options for excluding a particular kind of disks.

### Non-Goals

## Proposal

`local-storage-operator`is already capable of consuming local disks from the nodes and provisioning PVs out of it, but the disk/device paths needs to explicitly specified in the `LocalVolume` CR. The idea is to bring in one more layer of CR above it called `AutoDetectVolumes`.
The controller for this CR will be responsible for automatically detecting disks/devices from the available nodes and creating/managing the life cycle of `LocalVolumes` which are governed by the `AutoDetectVolumes`.s

### Risks and Mitigations

- Detecting disks that contain data and/or are in-use.
- Ensuring disks aren't re-detected as new or otherwise destroyed if their device path changes.

## Design Details

API scheme for `AutoDetectVolumes`:

```
type DeviceDiscoveryPolicyType string

const (
	raid DeviceDiscoveryPolicyType = "raid"
	disk DeviceDiscoveryPolicyType = "disk"
)

type AutoDetectVolumeList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata"`
	Items           []AutoDetectVolume `json:"items"`
}

type AutoDetectVolume struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata"`
	Spec              AutoDetectVolumeSpec   `json:"spec"`
	Status            AutoDetectVolumeStatus `json:"status,omitempty"`
}

type AutoDetectVolumeSpec struct {
	// StorageClass name to use for set of matched devices
	StorageClassName string `json:"storageClassName"`
	// Volume mode. Raw or with file system
	// + optional
	VolumeMode PersistentVolumeMode `json:"volumeMode,omitempty"`
	// File system type
	// +optional
	FSType string `json:"fsType,omitempty"`
	// Nodes on which the autoDetection must run
	// +optional
	NodeSelector *corev1.NodeSelector `json:"nodeSelector,omitempty"`
	// DeviceDiscoverPolicies are the list of policies that AutoDetectVolume's controller
	// will be looking for during detection
	DeviceDiscoveryPolicies []DeviceDiscoveryPolicy `json:"deviceDiscoveryPolicies,omitempty"`
}

type AutoDetectVolumeStatus struct {
	Status string `json:"status"`
}

type DeviceDiscoveryPolicy struct {
	// Device type that should be used for auto detection. This would be one of the types supported
	// by the local-storage operator. Initial possible types can be - crypt,raid1, raid4, raid5,
	// raid10, multipath, disk, tape, printer, processor, worm, rom, scanner, mo-disk, changer,
	// comm, raid, enclosure, rbc, osd, and no-lun.
	DeviceType DeviceDiscoveryPolicyType `json:"deviceType"`
	// A list of regular expressions that will be used to exclude certain devices
	// For example - ["^rbd[0-9]+p?[0-9]{0,}$"]
	// +optional
	DeviceExclusionFilter []string `json:"deviceExclusionFilter"`
	// For excluding rotational devices like mechanical disks
	// +optional
	ExcludeRotational bool `json:"excludeRotational"`
	// For excluding non rotational devices like SSDs
	// +optional
	ExcludeNonRotational bool `json:"excludeNonRotational"`
}
```

### Test Plan

- The integration tests for Local-Storage operators already exists. Those tests needs to be upgraded for testing this features.

### Graduation Criteria

N/A

#### Examples

These are generalized examples to consider, in addition to the aforementioned
[maturity levels][maturity-levels].

##### Removing a deprecated feature

- None of the features are getting deprecated

### Upgrade / Downgrade Strategy

N/A

### Version Skew Strategy

N/A

## Implementation History

N/A

## Drawbacks

N/A

## Alternatives
- Existing manual creation of LocalVolume CRs. With the node selector on the LocalVolume, a single CR can apply to an entire class of nodes (i.e., a machineset or a physical rack of homogeneous hardware).When a machineset is defined, a corresponding LocalVolume can also be created. Why is this insufficient ?
- Directly enhancing the LocalVolume CR to allow for auto discovery



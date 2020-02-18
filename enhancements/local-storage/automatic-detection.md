---
title: automatic-detection-of-disks-in-Local-Storage-Operator
authors:
  - "@aranjan"
  - "@rohantmp"
reviewers:
  - "@jrivera"
  - "@jsafrane"
  - "@hekumar"
  - "@chuffman"
  - "@rojoseph"
  - "@sapillai"
  - "@leseb"
  - "@travisn"
approvers:
  - "@jrivera"
  - "@jsafrane"
  - "@hekumar"
  - "@chuffman"
creation-date: 2020-01-21
last-updated: 2020-02-18
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

When working with bare-metal OpenShift clusters, due to the absence of storage provisioners like that in the cloud, there is a lot of manual work to be done for consuming local storage from nodes. The idea is to automate the manual steps that are required for consuming local storage in an OpenShift cluster by extending the local storage operator (LSO).

## Motivation

To facilitate the consumption of locally-attached storage devices on OpenShift nodes in a cloud-native way.

### Goals

- Automatic detection of available disks from nodes which can be used as PVs for OpenShift-cluster.
- Should respond to the attach/detach events of the disks/devices.
- Should have options for filtering particular kind of disks based on properties such as name, size, manufacturer, etc.

### Non-Goals

## Proposal

`local-storage-operator`is already capable of consuming local disks from the nodes and provisioning PVs out of it, but the disk/device paths needs to explicitly specified in the `LocalVolume` CR. The idea is to bring in one more layer of CR above it called `AutoDetectVolumes`.
The controller for this CR will be responsible for automatically detecting disks/devices from the available nodes and creating/managing the life cycle of `LocalVolumes` which are governed by the `AutoDetectVolumes`.

### Risks and Mitigations

- Detecting disks that contain data and/or are in-use.
- Ensuring disks aren't re-detected as new or otherwise destroyed if their device path changes.

## Design Details

API scheme for `AutoDetectVolumes`:

```
type DeviceDiscoveryPolicyType string

const (
	// The DeviceDiscoveryPolicies that will be supported by the LSO.
	// These Discovery policies will based on lsblk's type output
	disk DeviceDiscoveryPolicyType = "disk"
	part DeviceDiscoveryPolicyType = "part"
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
	// StorageClass name to use for set of matched devices
	StorageClassName string `json:"storageClassName"`
	// Device type that should be used for auto detection. This would be one of the types supported
	// by the local-storage operator. Initial possible types can be - crypt, raid1, raid4, raid5,
	// raid10, multipath, disk, tape, printer, processor, worm, rom, scanner, mo-disk, changer,
	// comm, raid, enclosure, rbc, osd, and no-lun.
	DeviceType []DeviceDiscoveryPolicyType `json:"deviceType"`
	// Minimum number of Devices that needs to be detected
	MinDeviceCount int `json:"minDeviceCount"`
	// +optional
	// Maximum number of Devices that needs to be detected
	MaxDeviceCount int `json:"maxDeviceCount"`
	// File system type
	// +optional
	FSType string `json:"fsType,omitempty"`
	// Volume mode. Raw or with file system
	// + optional
	VolumeMode PersistentVolumeMode `json:"volumeMode,omitempty"`
	// DeviceInclusionSpec are the filtration rules for including a device in the device discovery
	// +optional
	DeviceInclusionSpec *DeviceInclusionSpec `json:"deviceInclusionSpec"`
}

// DeviceMechanicalProperty holds the device's mechanical spec. It can be rotational,nonRotational and
// RotationalAndNonRotational
type DeviceMechanicalProperty string

const (
	// The mechanical properties of the devices
	// Rotational refers to magnetic disks
	Rotational    DeviceMechanicalProperty = "rotational"
	// NonRotational refers to ssds
	NonRotational DeviceMechanicalProperty = "nonRotational"
)

type DeviceInclusionSpec struct {
	// The devices of this mechanicalPropery will be included.
	// By default it is RotationalAndNonRotational(or SSD/HDD both)
	// +optional
	DeviceMechanicalProperty DeviceMechanicalProperty `json:"deviceMechanicalProperty"`

	// The minimum size of the device which needs to be included
	// +optional
	MinSize resource.Quantity `json:"minSize"`

	// The maximum size of the device which needs to be included
	// +optional
	MaxSize resource.Quantity `json:"maxSize"`
}
```

An example of autoDetectVolume CR instance:
```
apiVersion: local.storage.openshift.io/v1
kind: AutoDetectVolume
metadata:
  name: example-autodetect
spec:
  nodeSelector: null
  deviceDiscoverPolicies:
    - deviceType: part
      fsType: ext4
      storageClassName: example-storageclass1
      volumeMode: filesystem
      minDeviceCount: 5
      deviceInclusionSpec:
        deviceMechanicalProperty: nonRotational
        minSize: 10G
        maxSize: 100G
    - deviceType: disk
      storageClassName: example-storageclass2
      volumeMode: Block
      minDeviceCount: 5
      maxDeviceCount: 10
      deviceInclusionSpec:
        deviceMechanicalProperty: rotational
        minSize: 10G
        maxSize: 100G
```

The autoDetectVolume controller will need to interact with a discovery daemon for discovering devices from nodes. The existing discovery daemon of LSO needs to be modified for serving this purpose. Each daemon can expose all the device metadata information from its node in a configmap, which can be consumed by the local-storage operator and the filtration logic and localVolume CR creation logic can lie in LSO operator in the AutoDetectVolume control loop.  

### Test Plan

- The integration tests for the LSO already exist. These tests will need to be updated to test this feature.
- The tests must ensure that detection of devices are working/updating correctly.
- The tests must ensure that data corruption are not happening during auto detection of devices.

### Graduation Criteria

N/A

#### Examples

These are generalized examples to consider, in addition to the aforementioned
[maturity levels][maturity-levels].

##### Removing a deprecated feature

- None of the features are getting deprecated

### Upgrade / Downgrade Strategy

Since this requires a new implementation no new upgrade strategy will be required.

### Version Skew Strategy

N/A

## Implementation History

N/A

## Drawbacks

N/A

## Alternatives
- Existing manual creation of LocalVolume CRs. With the node selector on the LocalVolume, a single CR can apply to an entire class of nodes (i.e., a machineset or a physical rack of homogeneous hardware). When a machineset is defined, a corresponding LocalVolume can also be created.
- Directly enhancing the LocalVolume CR to allow for auto discovery

The first approach requires some manual work and knowledge of underlying nodes, this makes it inefficient for large clusters. The second approach can introduce breaking change to the existing GA API.
Therefore this approach makes sense. 
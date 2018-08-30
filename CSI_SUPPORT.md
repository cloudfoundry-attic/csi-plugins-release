# Container Storage Interface (CSI) Volume Plugin Support in Cloud Foundry

CSI provides a uniform storage plugin interface that allows storage providers to implement plugins that will connect their storage to multiple container orchestrators (Kubernetes, Mesos, and Cloud Foundry as of this writing).

## Version Support
Currently, Cloud Foundry supports [CSI version 0.3](https://github.com/container-storage-interface/spec/tree/v0.3.0)

## Limitations
There are a number of notable limitations in Cloud Foundry's CSI support:
* Plugins Advertising `ACCESSIBILITY_CONSTRAINTS` from `GetPluginCapabilities` will be rejected, since we do not have logic to respect topology constraints in Cloud Foundry.
* Plugins Advertising `PUBLISH_UNPUBLISH_VOLUME` from `ControllerGetCapabilities` will likely fail.  Cloud Foundry does not currently make calls to `ControllerPublishVolume` even when the plugin implements it.
* Current Cloud Foundry service broker implementations don't have support for snapshot creation, so even for plugins advertising `CREATE_DELETE_SNAPSHOT` we don't support snapshots.  Adding snapshot support to [csibroker](https://github.com/cloudfoundry/csibroker) is theoretically not difficult however.
* Cloud Foundry container scheduling emphasizes high availability and does not allow us to guarantee that container instances are not running simultaneously on different nodes.  For this reason, it is not recommended to use CSI plugins with block storage, or with single-access modes (`SINGLE_NODE_WRITER`, `SINGLE_NODE_READER_ONLY` or `MULTI_NODE_SINGLE_WRITER`).

## Overview
CSI plugins typically support 2 plugin components:
* The **Controller Plugin** can be running anywhere, and is concerned with provisioning storage and attaching it to particular nodes.
* The **Node Plugin** must be colocated on the same VM as workloads using the storage, and is concerned with node specific tasks such as mounting/unmounting the storage, or iSCSI initiation.  For Cloud Foundry purposes a **CSI Node** is equivalent to a **Diego Cell**.

Some plugins consolidate both components into a single binary, in which case, we might have to route controller traffic to a particular node, but to keep this document simple, we will assume that there are 2 separate components for now.

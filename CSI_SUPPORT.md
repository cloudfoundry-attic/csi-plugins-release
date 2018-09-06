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

If you're familar [with the support for volume services in Cloud Foundry](http://bit.ly/cf-persi-overview) prior to the advent of CSI, then you might have noticed that:

1) the functions of a CSI controller plugin are very similar to a Cloud Foundry volume service broker, and
1) the functions of a CSI node plugin are very similar to a Cloud Foundry/Docker volume plugin.

It should come as no suprise, then, that the CSI integration for Cloud Foundry bears a close resemblance to existing Volume Service integrations:

![CSI in Cloud Foundry Diagram](images/CSI-CS.png)

### Components of the Cloud Foundry CSI Integration

* **[csibroker](https://github.com/cloudfoundry/csibroker)** is a generic Open Service Broker API (OSBAPI) service broker that translates OSBAPI service instances and service bindings into calls to the CSI controller plugin.  For folks familiar with the kubernetes CSI integration, csibroker fulfills a similar function to the k8s [external-provisioner](https://github.com/kubernetes-csi/external-provisioner) container. It is not strictly necessary to use csibroker as-is, but we hope to make it generic enough to bridge the gap between the OSBAPI and the CSI Controller interface.  If you find areas that require enhancement to meet your needs, please let us know in a [github issue](https://github.com/cloudfoundry/csi-plugins-release/issues). csibroker must be configured with 
  - connection information that tells it where to find the CSI Controller plugin 
  - a service catalog that maps human readable service and plan offerings into CSI ControllerCreateVolume data payloads (TBD in the current csibroker)
* **volman** The Diego Volume Manager (volman) is a subcomponent of the Diego **rep** job that is responsible for managing remote volume mounts.  In the past, it only managed mounts through Docker volume plugins, but it now contains a new ["CSI Plugin Discoverer"](https://github.com/cloudfoundry/volman/blob/master/voldiscoverers/csiplugin_discoverer.go) that is responsible for discovering and connecting to CSI Node plugins available on the Diego cell.  Node plugins must advertise themselves to Diego by placing a json spec file in a predetermined path on the cell.  (Typically this path is `/var/vcap/data/csiplugins` but it can be modified in a BOSH property on the `rep` job.)  

## Deployment Considerations

### Basic Deployment
CSI is not opinionated about the deployment mode for CSI plugins.  The basic requirements state that:
1) The master plugin may be running anywhere, but must be reachable by the master process.  (In cloud foundry this "master" is a registered OSBAPI broker.)
2) The node plugin must be installed on the worker node that will run payloads that consume the volume.

Cloud Foundry does not add any additional deployment constraints beyond those stipulated by CSI.  CSI plugins can be BOSH deployed jobs, containerized processes, or just regular processes installed to the host in some way.  Master plugins can run on external hosts, so long as those hosts are reachable through a network connection.

We do, however, have a couple specific requirements in order for CSI plugins to interface with Cloud Foundry.

#### the CSI service broker
As mentioned above, Cloud Foundry does not have the capability to communicate directly with a CSI Master plugin.  To perform CSI master node operations, we expect that the plugin will be deployed alongside a service broker.  The service broker translates OSBAPI calls into CSI master node calls, effectively exposing the CSI plugin to Cloud Foundry as an [OSBAPI volume service](https://github.com/openservicebrokerapi/servicebroker/blob/master/spec.md#volume-services).  You can choose to implement your own service broker to do this job, or use the [generic one](https://github.com/cloudfoundry/csibroker) we provide.

The generic csibroker can be BOSH deployed with a [job in csi-plugins-release](https://github.com/cloudfoundry/csi-plugins-release/tree/master/jobs/csi-broker).  You should populate the `csi-broker.services` property with a service catalog that references your plugin, including the plugin id, and the URL where it can be contacted.  Look in the [job spec file](https://github.com/cloudfoundry/csi-plugins-release/blob/master/jobs/csi-broker/spec#L29-L50) to see suitable examples for service catalog entries.

#### the CSI Node Plugin spec file
As we've said, the Node plugin must be running on each worker node (or "Diego Cell" in Cloud Foundry parlance) where it will be used.  In Cloud Foundry we have the capability to communicate directly with CSI node plugins from the Diego "rep" job, but Node plugins must advertise themselves to Diego by placing a json spec file in a predesignated folder.  The Diego volume manager polls that directory periodically looking for spec files and contacts the plugin at the given address to make sure it is available.  The format for the json spec file is very simple.  For example:
```
{
  "Name":"org.cloudfoundry.code.local-node-plugin",
  "Address":"0.0.0.0:9760"
}
```
The spec file should be placed in `/var/vcap/data/csiplugins` and have a `.json` file extension.

Note that the "Name" field is purely for readability.  Diego will contact the CSI `GetPluginInfo()` RPC to determine the ID of the plugin.

### Deployment of Containerized Plugins

## More Resources
- https://github.com/cloudfoundry/csi-local-volume-release
- https://github.com/cloudfoundry/csi-plugins-release
- https://github.com/container-storage-interface/spec
- https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md
- https://arslan.io/2018/06/21/how-to-write-a-container-storage-interface-csi-plugin/


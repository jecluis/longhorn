# Title

s3gw integration with Longhorn

## Summary

By integrating s3gw with Longhorn, we are able to provide an S3-compliant API to
clients consuming Longhorn volumes. This is achieved by creating an S3 endpoint
(using s3gw) for a Longhorn volume.

### Related Issues

https://github.com/longhorn/longhorn/issues/4154


## Motivation

### Goals

* Provide an S3 endpoint associated with a Longhorn volume.
* Multiple S3 endpoints should be supported, with each endpoint being backed by
  one single Longhorn volume.


### Non-goals

* Integration of s3gw UI for administration and management purposes. Such an
  Enhancement Proposal should be a standalone LEP by its own right.
* Providing S3 endpoints for multiple volumes. In this proposal we limit one s3
  endpoint per Longhorn volume.
* Multiple S3 endpoints for a single volume, either in Read-Write Many, or as
  active/passive for HA failover.

## Proposal

This is where we get down to the nitty-gritty of what the proposal actually is.

### User Stories

#### Story

Currently, Longhorn does not support object storage. Should the user want to use
their Longhorn cluster for object storage, they have to rely on third-party
applications.

Instead we propose to enhance the user experience by allowing a Longhorn volume
to be presented to the user as an object storage endpoint, without having the
user to install additional dependencies or manage different applications.

### User Experience In Detail

* Using the Longhorn UI, user clicks on "Create Volume";
* User fills the various fields according to their desire;
* User selects S3 as the Frontend;
    * Access Mode becomes fixed at "Read-Write Once";
* User clicks "Ok".

### API changes

The API will need to understand a new `s3` parameter for the `Frontend` field
during volume creation.

## Design

### Implementation Overview

We believe changes to be required in various ways:

* Creating a new `S3Endpoint` Custom Resource Definition;
* Enabling `s3` as a new Frontend for volume creation;
* Creating a new `S3Endpoint` resource (via `client-go` API) with the new s3
  endpoint associated with the newly created volume;
* Creating a `controller/s3endpoint_controller.go`, defining a new
  `S3EndpointController`;
* Listening for new `S3Endpoint` resources in `S3EndpointController`, creating a
  new `Service` and a new `Pod`.


#### Custom Resource Definition

It is not clear at this stage what the CRD should look like in detail, but
following the implementation for the existing `ShareManager`, something similar
would likely be desired.

```golang
longhorn.S3Endpoint{
    ObjectMeta: metav1.ObjectMeta{
        Name: volume.Name,
        Namespace: volume.Namespace
    },
    Spec: longhorn.S3EndpointSpec{
        Image: <s3gw-image-name>,
        ImageUI: <s3gw-ui-image-name>
    }
}
```

#### Required changes

We expect to need to perform the following changes to `longhorn-manager`:

* Introduce two new arguments: `--s3-endpoint-image <image>`, and `--s3-endpoint-image-ui`;
* Create a new `S3EndpointController` in
  `controllers/s3_endpoint_controller.go`, responsible for creating and managing
  `s3gw` pods;
* Add `S3EndpointController` to `controller/controller_manager.go`'s
  `StartControllers()`;
* Introduce a new `S3EndpointInformer` to `DataStore`;
* Create new functions `ReconcileS3EndpointState()` and
  `createS3EndpointForVolume()` in `controller/volume_controller.go`;
* Create new function `CreateS3Endpoint()` in `datastore/longhorn.go`.

Additionally, we expect to add `s3gw` images as dependencies to be downloaded by
the Longhorn chart.

Further changes may be needed as development evolves.

### Test plan

It is not clear at this moment how this can be tested, much due to lack of
knowledge on how Longhorn testing works. Help on this topic would be much
appreciated.

### Upgrade strategy

Upgrading to this enhancement should be painless. Once this feature is available
in Longhorn, the user should be able to create new Volumes with the S3 Frontend
without much else to do.

## Note

Below, we can find a flowchart created during the discovery phase that served as
the basis for this proposal's proposed changes. We looked quite a bit into how
Longhorn's `share-manager` works, and we followed much of the same logic, but
applied to `s3gw`.

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk"}} }%%
graph TB

    uiapi -- "/v1/volumes/{name}?action=pvCreate" --> apirouter(api/router.go)

    subgraph UI
        uicpv(CreatePVAndPVCSingle.js) -- set accessMode to rwx --> uiapi((UI API))
    end

    subgraph LM
        apirouter --> pvcreate
        main[main.go]        

        subgraph api/volume.go
            pvcreate["Server.PVCreate()"]
            volcreate["Server.VolumeCreate()"]

        end

        apirouter --> volcreate

        subgraph manager
            subgraph manager/kubernetes.go
                vmpvcreate["PVCreate()"]
            end

            subgraph manager/volume.go
                vmcreate["Create()"]
            end
        end
        pvcreate --> vmpvcreate
        volcreate --> vmcreate


        vmpvcreate -- "(1)" --> dspvmanifestforvolume
        vmpvcreate -- "(2)" --> dscreatepersistentvol
        subgraph datastore
            subgraph datastore/kubernetes.go
                dspvmanifestforvolume["NewPVManifestForVolume()"]
                dsnewpvmanifest["NewPVManifest()"]
                dscreatepersistentvol["CreatePersistentVolume()"]
                dspvmanifestforvolume --> dsnewpvmanifest
                dsnewpvmanifest --- labelA{{Create new<br/>corev1.PersistentVolumeSpec}}
                dscreatepersistentvol --- labelB{{Create Persistent Volume<br/>via CoreV1 interface}}

                dscreatesvc["CreateService()"]
                dscreatepod["CreatePod()"]
            end

            subgraph datastore/datastore.go
                dsnew["NewDataStore()"]
            end

            subgraph datastore/longhorn.go
                dscreatesm["CreateShareManager()"]

                k8screatesm{{REST call<br>POST<br>Resource sharemanagers}}

                dscreatesm --> k8screatesm
            end

            subgraph datastore/volume.go
                dscreatevol["CreateVolume()"]
                dslabelcreatevol{{"Create Volume<br>via LonghornV1beta2()"}}
                dscreatevol --- dslabelcreatevol
            end
        end
        vmcreate --> dscreatevol

        main -- "cli.Run()" --> daemon

        subgraph app/daemon.go
            daemon["DaemonCmd()"]
            daemonstart["StartManager()"]
            daemon --> daemonstart
        end

        daemonstart --> ctrstart

        subgraph controller
            subgraph "controller/controller_manager.go"
                ctrstart["StartControllers()"]

                ctrstart --- labelC
                labelC{{Create<br/><ul><li>kubeClient</li><li>lhClient</li><li>kubeInformerFactory</li><li>lhInformerFactory</li><li>DataStore</li></ul>}}
            end

            ctrstart -- "(2)" --> smnew

            subgraph "controller/share_manager_controller.go"
                smnew["NewShareManagerController()"]
                smevhandler{{"DataStore.ShareManagerInformer<br>AddEventHandler"}}
                smnew --> smevhandler

                smenqueue["enqueueShareManagerForVolume()"]
                smevhandler --- smenqueue
                smenqueue -.-> smnextitem
                smnextitem["ProcessNextWorkItem()"]

                smsync["SyncShareManager"]
                smnextitem --> smsync

                smsyncvol["syncShareManagerVolume()"]
                smsyncpod["syncShareManagerPod()"]
                smsyncep["syncShareManagerEndpoint()"]

                smsync -- "(1)" --> smsyncvol
                smsync -- "(2)" --> smsyncpod
                smsync -- "(3)" --> smsyncep

                smcreatesvc["createServiceManifest()"]
                smcreatepod["createPodManifest()"]

                smsyncpod -- "(1)" --> smcreatesvc
                smsyncpod -- "(2)" --> dscreatesvc
                smsyncpod -- "(3)" --> smcreatepod
                smsyncpod -- "(4)" --> dscreatepod
            end

            subgraph "controller/volume_controller.go"
                vcnew["NewVolumeController()"]
                vceventhandler{{"AddEventHandler for:<br>ShareManagerInformer<br>VolumeInformer"}}
                vcnew --> vceventhandler

                vcenqueue4sm["enqueueVolumesForShareManager()"]
                vcenqueuevol["enqueueVolume()"]
                vcnextitem["ProcessNextWorkItem()"]
                vceventhandler --- vcenqueue4sm
                vceventhandler --- vcenqueuevol
                vcenqueue4sm -.-> vcnextitem
                vcenqueuevol -.-> vcnextitem

                vcsyncvol["syncVolume()"]
                vcnextitem --> vcsyncvol

                vcreconcilesm["ReconcileShareManagerState()"]
                vccreatesmforvol["createShareManagerForVolume()"]

                vcsyncvol --> vcreconcilesm
                vcreconcilesm --> vccreatesmforvol
                vccreatesmforvol --> dscreatesm
            end

            ctrstart --> vcnew
        end

        ctrstart -- "(1)" --> dsnew

        labelB -.-> enqueue

        subgraph k8s.io/client-go/corev1
            k8screatepod["CoreV1()<br>Pods().Create()"]
            k8screatesvc["CoreV1()<br>Services().Create()"]
        end

        dscreatesvc --> k8screatesvc
        dscreatepod --> k8screatepod
    end

    k8scluster[(Kubernetes REST API)]

    k8screatesvc --> k8scluster
    k8screatepod --> k8scluster
    k8screatesm --> k8scluster
    dslabelcreatevol --> k8scluster

```

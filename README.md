# oran-hwmgr-plugin-test

The Test Hardware Manager Plugin for O-Cloud Manager (Test Plugin) provides a test mechanism to mock the interactions
between the O-Cloud Manager and a generic hardware manager.

## Overview

The Test Plugin monitors its own namespace for NodePool CRs. When a NodePool CR is created in the Test Plugin namespace,
it will process the CR and checks the (fake) managed resources to find free (fake) nodes to satisfy the NodePool
request. These managed resources are defined in a `nodelist` configmap, defined in the Test Plugin namespace, which
provides a list of hardware profile names and defines the set of managed nodes. See
[configmap/example-nodelist.yaml](configmap/example-nodelist.yaml) for an example `nodelist` configmap. In addition, the
[configmap/generator.sh](configmap/generator.sh) script can be used to generate a `nodelist` configmap.

The `nodelist` configmap is also used by the Test Plugin to track node allocations. As free nodes are allocated to a
NodePool request, these are tracked in the `allocations` field in the configmap and a Node CR is created by the Test
Plugin, setting the node properties as defined in the configmap.

When a NodePool CR is deleted, the Test Plugin is triggered by a finalizer it added to the CR. In processing the
deletion, it will delete any Node CRs that have been allocated for the NodePool and free the node(s) in the `nodelist`
configmap.

## Testing

### Install O-Cloud Manager

The Test Plugin uses the NodePool and Node CRDs that are defined by the O-Cloud Manager. Please consult the
documentation in the [openshift-kni/oran-o2ims](https://github.com/openshift-kni/oran-o2ims) repository for information
on deploying the O-Cloud Manager to your cluster.

To run the Test Plugin as a standalone without the O-Cloud Manager, the CRDs can be installed from the repo by running:
`make install`

### Deploy Test Plugin

The Test Plugin can be deployed to your cluster by first building the image and then running the `deploy` make target:

```console
$ make REPO_OWNER=$USER VERSION=latest docker-build docker-push
$ make REPO_OWNER=$USER VERSION=latest deploy
```

The [configmap/generator.sh](configmap/generator.sh) script can be used to generate a `nodelist` configmap with
user-defined hardware profiles and any number of nodes.

Example test NodePool CRs can be found in the [examples](examples) folder.

```console
$ ./configmap/generator.sh \
    --profile profile-spr-single-processor-64G:dummy-sp-64g:5 \
	--profile profile-spr-dual-processor-128G:dummy-dp-128g:3 \
	| oc create -f -
configmap/nodelist created

$ oc create -f examples/np1.yaml
nodepool.hardwaremanagement.oran.openshift.io/np1 created

$ oc get nodepools.hardwaremanagement.oran.openshift.io -n oran-hwmgr-plugin-test np1 -o yaml
apiVersion: hardwaremanagement.oran.openshift.io/v1alpha1
kind: NodePool
metadata:
  creationTimestamp: "2024-09-05T12:50:47Z"
  finalizers:
  - oran-hwmgr-plugin-test.oran.openshift.io/nodepool-finalizer
  generation: 1
  name: np1
  namespace: oran-hwmgr-plugin-test
  resourceVersion: "16208137"
  uid: 5486b5e7-415b-4f63-a443-8cd88a32fef6
spec:
  cloudID: testcloud-1
  location: ottawa
  nodeGroup:
  - hwProfile: profile-spr-single-processor-64G
    name: master
    size: 1
  - hwProfile: profile-spr-dual-processor-128G
    name: worker
    size: 0
  site: building-1
status:
  conditions:
  - lastTransitionTime: "2024-09-05T12:51:08Z"
    message: Created
    reason: Completed
    status: "True"
    type: Provisioned
  properties:
    nodeNames:
    - dummy-sp-64g-1

$ oc get nodes.hardwaremanagement.oran.openshift.io -n oran-hwmgr-plugin-test dummy-sp-64g-1 -o yaml
apiVersion: hardwaremanagement.oran.openshift.io/v1alpha1
kind: Node
metadata:
  creationTimestamp: "2024-09-05T12:50:58Z"
  generation: 1
  name: dummy-sp-64g-1
  namespace: oran-hwmgr-plugin-test
  resourceVersion: "16208107"
  uid: 0838467c-f73e-4c5b-86cf-0083352243ce
spec:
  groupName: master
  hwProfile: profile-spr-single-processor-64G
  nodePool: testcloud-1
status:
  bmc:
    address: idrac-virtualmedia+https://192.168.2.1/redfish/v1/Systems/System.Embedded.1
    credentialsName: bmcSecret-dummy-sp-64g-1
  bootMACAddress: c6:b6:13:a0:02:01
  conditions:
  - lastTransitionTime: "2024-09-05T12:50:58Z"
    message: Provisioned
    reason: Completed
    status: "True"
    type: Provisioned
  hostname: dummy-sp-64g-1.localhost

$ oc get configmap -n oran-hwmgr-plugin-test nodelist -o yaml
apiVersion: v1
data:
  allocations: |
    clouds:
    - cloudID: testcloud-1
      nodegroups:
        master:
        - dummy-sp-64g-1
  resources: |
    hwprofiles:
      - profile-spr-dual-processor-128G
      - profile-spr-single-processor-64G
    nodes:
      dummy-dp-128g-0:
        hwprofile: profile-spr-dual-processor-128G
        bmc:
          address: "idrac-virtualmedia+https://192.168.1.0/redfish/v1/Systems/System.Embedded.1"
          credentialsName: "bmcSecret-dummy-dp-128g-0"
        bootMACAddress: "c6:b6:13:a0:01:00"
        hostname: "dummy-dp-128g-0.localhost"
      dummy-dp-128g-1:
        hwprofile: profile-spr-dual-processor-128G
        bmc:
          address: "idrac-virtualmedia+https://192.168.1.1/redfish/v1/Systems/System.Embedded.1"
          credentialsName: "bmcSecret-dummy-dp-128g-1"
        bootMACAddress: "c6:b6:13:a0:01:01"
        hostname: "dummy-dp-128g-1.localhost"
      dummy-dp-128g-2:
        hwprofile: profile-spr-dual-processor-128G
        bmc:
          address: "idrac-virtualmedia+https://192.168.1.2/redfish/v1/Systems/System.Embedded.1"
          credentialsName: "bmcSecret-dummy-dp-128g-2"
        bootMACAddress: "c6:b6:13:a0:01:02"
        hostname: "dummy-dp-128g-2.localhost"
      dummy-sp-64g-0:
        hwprofile: profile-spr-single-processor-64G
        bmc:
          address: "idrac-virtualmedia+https://192.168.2.0/redfish/v1/Systems/System.Embedded.1"
          credentialsName: "bmcSecret-dummy-sp-64g-0"
        bootMACAddress: "c6:b6:13:a0:02:00"
        hostname: "dummy-sp-64g-0.localhost"
      dummy-sp-64g-1:
        hwprofile: profile-spr-single-processor-64G
        bmc:
          address: "idrac-virtualmedia+https://192.168.2.1/redfish/v1/Systems/System.Embedded.1"
          credentialsName: "bmcSecret-dummy-sp-64g-1"
        bootMACAddress: "c6:b6:13:a0:02:01"
        hostname: "dummy-sp-64g-1.localhost"
      dummy-sp-64g-2:
        hwprofile: profile-spr-single-processor-64G
        bmc:
          address: "idrac-virtualmedia+https://192.168.2.2/redfish/v1/Systems/System.Embedded.1"
          credentialsName: "bmcSecret-dummy-sp-64g-2"
        bootMACAddress: "c6:b6:13:a0:02:02"
        hostname: "dummy-sp-64g-2.localhost"
      dummy-sp-64g-3:
        hwprofile: profile-spr-single-processor-64G
        bmc:
          address: "idrac-virtualmedia+https://192.168.2.3/redfish/v1/Systems/System.Embedded.1"
          credentialsName: "bmcSecret-dummy-sp-64g-3"
        bootMACAddress: "c6:b6:13:a0:02:03"
        hostname: "dummy-sp-64g-3.localhost"
      dummy-sp-64g-4:
        hwprofile: profile-spr-single-processor-64G
        bmc:
          address: "idrac-virtualmedia+https://192.168.2.4/redfish/v1/Systems/System.Embedded.1"
          credentialsName: "bmcSecret-dummy-sp-64g-4"
        bootMACAddress: "c6:b6:13:a0:02:04"
        hostname: "dummy-sp-64g-4.localhost"
kind: ConfigMap
metadata:
  creationTimestamp: "2024-09-05T12:47:59Z"
  name: nodelist
  namespace: oran-hwmgr-plugin-test
  resourceVersion: "16208103"
  uid: f0cc6238-15a1-40fd-abf1-4dcd0e32a531

```



# sat-roks-customizations-for-telco-apps
**Disclaimer**: This repo is a **Work-In-Progress**, and is yet to be peer-reviewed. The list is neither complete nor comprehensive. It has been created as a reference of what worked in some proof-of-concept projects. 
## Introduction
This repository  provides some guidance on openshift customizations that were discovered as prerequisites for typical networking and/or telco workloads/applications. The repo specifically targets on-premises infrastructure (hosts, network, etc.) attached to the IBM Cloud Satellite location. This is where IBM's managed openshift service (ROKS) cluster(s) are deployed on baremetal servers representing the worker nodes. This setup needs some alternative methods for configuring required capabilities in ROKS, as compared to the same on Red Hat (RH) OpenShift Container Platform (OCP).

Note that these observations and configurations have been taken from Satellite/ROKS cluster at OpenShift version 4.9.48.

With IBM Cloud Satellite/ROKS (link to announcement) now supporting CoreOS-enabled locations and hosts, some of the custom configurations needed by typical network applications require alternate methods in ROKS as compared to OCP. 

Some of the key capabilities that needed alternative way of configuring on ROKS included the following:
1. [Hugepages setup](#1-hugepages-setup)
2. [SCTP enablement](#2-sctp-enablement)
3. [SR-IOV operator configuration](#3-sr-iov-operator-configuration)
4. [CPU and Single NUMA Node config](#4-cpu-and-Single-numa-node-config)
5. [Storage driver install](#5-storage-driver-install)
6. [SCC privileges for application deployment](#6-scc-privileges-for-application-deployment)
7. [ExternalIP support for services](#7-externalip-support-for-services)
8. [Nodeport based service](#8-nodePort-based-service-creation)
9. [Nodeport service range](#9-nodePort-service-range)
10. [Performance AddOn Operator not supported](#10-performance-addon-operator-not-supported)
11. [PTP operator not supported](#11-ptp-operator-not-supported)
12. [Real time kernel not supported](#12-real-time-kernel-not-supported)
13. [Enable kernel modules](#13-enable-kernel-modules)
14. [Image registry configuration](#14-image-registry-configuration)

In general, the RH OCP documentation is the usual reference for configuring such custom settings. However, certain resources like `MachineConfig` are not supported on ROKS (IBM's managed Openshift service), and require alternative method for conifguring the capability.


## 1. Hugepages setup
The [RH OCP documentation for configuring huge pages](https://docs.openshift.com/container-platform/4.9/scalability_and_performance/what-huge-pages-do-and-how-they-are-consumed-by-apps.html#configuring-huge-pages_huge-pages) on OpenShift nodes requires creating yamls with the `Tuned` and `MachineConfigPool` resources. Since `MachineConfigPool` resource is not yet supported in ROKS, one alternative is to add the `kernel arguments` in the RH CoreOS ignition file during satellite host attach operation (presuming the hosts run RH CoreOS).
```
{
  "ignition": {
    "version": "3.3.0"
  },
  "kernelArguments": {
    "shouldExist": ["hugepagesz=1G hugepages=64 hugepagesz=2M hugepages=0 default_hugepagesz=1G"] 
  },
. . .
```

An example [ignition file](./examples/sample-ignition-file.json) is available in the [examples](./examples/) folder of this repo for reference.


## 2. SCTP enablement
To [enable SCTP on the nodes RH OCP documentation](https://docs.openshift.com/container-platform/4.9/post_installation_configuration/machine-configuration-tasks.html#using-machineconfigs-to-change-machines) points to use of `MachineConfig` objects. So, in order to accomplish this on ROKS, we used the ignition file entries to write to the sctp configuration files (`sctp-blacklist.conf` and `sctp-load.conf`).
```
{
  "ignition": {
    "version": "3.3.0"
  },
. . .
  "storage": {
    "files": [
      {
        "overwrite": true,
        "path": "/etc/modprobe.d/sctp-blacklist.conf",
        "contents": {
          "source": "data:text/plain;charset=utf-8,x23x23x23"
        },
        "mode": 420
      },
      {
        "overwrite": true,
        "path": "/etc/modules-load.d/sctp-load.conf",
        "contents": {
          "source": "data:text/plain;charset=utf-8,sctp"
        },
        "mode": 420
      }
. . .
```


## 3. SR-IOV operator configuration
Satellite/ROKS requires additional configuration setup when installing the SR-IOV operator on the cluster. These steps are currently (as of Oct 12, 2022), documented in IBM [Staging documentation for SR-IOV Setup](https://test.cloud.ibm.com/docs/openshift?topic=openshift-satellite-sriov).


## 4. CPU and Single-NUMA-Node config
[Setting up CPU Manager as per RH OCP documentation](https://docs.openshift.com/container-platform/4.9/scalability_and_performance/using-cpu-manager.html#seting_up_cpu_manager_using-cpu-manager) also requires support for `MachineConfigPool` resource.
In case of Satellite/ROKS clusters, the workaround involves using the `DaemonSet` resource instead.

The following is an example yaml that we used to enable CPU manager, enable Single-NUMA-Node, restart kubelet, and re-apply if the cluster were to be reloaded.

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cpumanager-enablement
  namespace: kube-system
  labels:
    tier: management
    app: cpumanager-enablement
spec:
  selector:
    matchLabels:
      name: cpumanager-enablement
  template:
    metadata:
      labels:
        name: cpumanager-enablement
    spec:
      hostPID: true
      initContainers:
        - command: ["/bin/sh", "-c"]
          args:
            - >
              grep -qF cpuManagerPolicy /host/etc/kubernetes/kubelet.conf || sed -i '/kubeAPIBurst/i cpuManagerPolicy: static' /host/etc/kubernetes/kubelet.conf;
              grep -qF cpuManagerReconcile /host/etc/kubernetes/kubelet.conf || sed -i '/kubeAPIBurst/i cpuManagerReconcilePeriod: 5s' /host/etc/kubernetes/kubelet.conf;
              grep -qF topologyManagerPolicy /host/etc/kubernetes/kubelet.conf || sed -i '/kubeAPIBurst/i topologyManagerPolicy: single-numa-node' /host/etc/kubernetes/kubelet.conf;
              grep -qF 'topologyManager:' /host/etc/kubernetes/kubelet.conf || sed -i '/kubeAPIBurst/i    topologyManager: true' /host/etc/kubernetes/kubelet.conf;
              chroot /host sh -c 'systemctl restart kubelet'
          image: alpine:3.6
          imagePullPolicy: IfNotPresent
          name: kubelet
          resources: {}
          securityContext:
            privileged: true
          volumeMounts:
            - name: modify-kubelet
              mountPath: /host
      containers:
        - resources:
            requests:
              cpu: 0.01
          image: alpine:3.6
          # once the init container completes, keep the pod running for worker node changes
          name: sleepforever
          command: ["/bin/sh", "-c"]
          args:
            - >
              while true; do
                  sleep 100000;
              done
      volumes:
        - name: modify-kubelet
          hostPath:
            path: /
```


## 5. Storage driver install
IBM Cloud Satellite provides storage templates for a set of storage providers/drivers. This is an alternative to installing from Catalog, OperatorHub, Helm charts, etc. This concept and process is well captured in [IBM documentation for Satellite storage templates](https://cloud.ibm.com/docs/satellite?topic=satellite-sat-storage-template-ov).


## 6. SCC privileges for application deployment
Satellite/ROKS requires the default service account in a project/namespace to have elevated Security Context Constraint (SCC) privileges. This is necessary when a deployment in the namespace attempts to create pods that need elevated access.

The following is a sample error message symptomatic of this case when a `deployment` attempts to create pods needing elevated privileges:
```
pods "pod-name-568fc89576-" is forbidden: unable to validate against any security context constraint: [provider "anyuid": Forbidden: not usable by user or serviceaccount, spec.volumes[4]: Invalid value: "hostPath": hostPath volumes are not allowed to be used, spec.containers[1].securityContext.runAsUser: Invalid value: 0: must be in the ranges: [1000670000, 1000679999], spec.containers[1].securityContext.privileged: Invalid value: true: Privileged containers are not allowed, spec.containers[1].securityContext.capabilities.add: Invalid value: "SYS_PTRACE": capability may not be added, provider "hostmount-anyuid": Forbidden: not usable by user or serviceaccount, provider "hostnetwork": Forbidden: not usable by user or serviceaccount, provider "hostaccess": Forbidden: not usable by user or serviceaccount, 
. . .
provider "privileged": Forbidden: not usable by user or serviceaccount, provider "rook-ceph-csi": Forbidden: not usable by user or serviceaccount]
```

One way to configure this is to identify the default `serviceaccount` used in the namespace for deployment, and add it as an `annotation` in the `privileged` SCC.

Following is an example `oc` command to accomplish the same: 
```
oc adm policy add-scc-to-user <scc> \  -z <serviceaccount> \  --namespace <namespace>
```
IBM Cloud [documentation also provides some guidance for ROKS clusters](https://cloud.ibm.com/docs/openshift?topic=openshift-plan_deploy#openshift_move_apps_example_scc) for further reference.


## 7. ExternalIP support for services
Satellite/ROKS does not allow `deployments` to create `services` that have an `externalIP` defined. The controller/manager attempting to deploy such a service will encounter errors similar to the following sample:

```
2022-09-22T19:00:50.678Z        ERROR   . . . "namespace": "default", "error": "services \"sample-svc-name\" is forbidden: spec.externalIPs: Forbidden: externalIPs have been disabled"}
```
The workaround is to create the `service` with `ClusterIP` first. And later, an `externalIP` can be added using the `oc patch` command after the `service` has been created. However, this `externalIP` is only reachable within the ROKS cluster, unlike in RH OCP.

But for on-prem infrastructure it is possible to add internal network routes to allow connecting to the subnet on which the `services` have their `ExternalIP` configured.

The IBM Cloud satellite documentation also lists [multiple ways to expose applications on Satellite/ROKS clusters](https://cloud.ibm.com/docs/openshift?topic=openshift-sat-expose-apps).


## 8. NodePort based service creation
For cases where an application `service` needs to be exposed using `NodePort`, in addition to, already being exposed via either `clusterIP` and/or `ExternalIP` it's necessary to review the `yaml` definition of the `NodePort` based service, after it's [created via `oc` CLI](https://docs.openshift.com/container-platform/4.9/networking/configuring_ingress_cluster_traffic/configuring-ingress-cluster-traffic-nodeport.html#nw-exposing-service_configuring-ingress-cluster-traffic-nodeport).

This is because, by default the `NodePort`-based service appears to be created with "TCP" protocol (i.e., even if the existing service has "UDP" or other protocol), and the `Targetport` value set to be the same as service `port` of the existing service (i.e., irrespective of the protocol specified in the `clusterIP`/`externalIP` based service).


## 9. NodePort service range
The [`NodePort` service range can be expanded on RH OCP](https://docs.openshift.com/container-platform/4.9/networking/configuring-node-port-service-range.html) clusters.

However, this is not allowed on Sateelite/ROKS clusters (possible due to performance reasons). While the attempt to patch the configuration as per above RH OCP documentation may appear to work, the below failure during the confirmation step is indicative of this limitation:
```
# oc get configmaps -n openshift-kube-apiserver config   -o jsonpath="{.data['config\.yaml']}" |   grep -Eo '"service-node-port-range":["[[:digit:]]+-[[:digit:]]+"]'
Error from server (NotFound): configmaps "config" not found
```

The workaround is to change the ports that the service uses to the default port range of `30000-32767`.


## 10. Performance AddOn Operator not supported
The [OpenShift Performance Addon operator](https://docs.openshift.com/container-platform/4.9/scalability_and_performance/cnf-performance-addon-operator-for-low-latency-nodes.html) helps in tuning the cluster for applications seeking performance benefits via low latency configurations on the nodes.

This operator is not currently supported on Satellite/ROKS. Also since the `PerformanceProfile` resource that is used to enable some of these setttings requires support for `MachineConfig`/`MachineConfigPool`, [IBM ROKS documentation points to alternative ways for tuning performance for RH CoreOS worker nodes](https://cloud.ibm.com/docs/openshift?topic=openshift-rhcos-performance).

## 11. PTP Operator not supported
The [OpenShift Precision Time Protocol operator](https://docs.openshift.com/container-platform/4.9/networking/using-ptp.html) is not currently supported on Satellite/ROKS.

## 12. Real time kernel not supported
RH OCP allows users to [switch to a realtime kernel on bare metal workers](https://docs.openshift.com/container-platform/4.9/post_installation_configuration/machine-configuration-tasks.html#nodes-nodes-rtkernel-arguments_post-install-machine-configuration-tasks). This is typically required by telco workloads like RAN components (Centralized Unit(CU), Distribution Unit (DU)), etc. that seek low latency and high degree of determinism. RH OCP manages to install and enable `kernel-rt` on baremetal worker nodes via `machigeConfig` support.

Currently, IBM Satellite/ROKS does not yet support replacing the kernel on the CoreOS workers with a real-time kernel.

## 13. Enable kernel modules
Some workloads/application require additional special purpose modules to be loaded and configured in the kernel for the openshift worker nodes. This includes modules such as `sctp`,`vhost_net`, `nf-*`, etc., and [RH OCP typically uses the machineConfig operator and/or resources to enable the kernel modules as documented here](https://docs.openshift.com/container-platform/4.9/installing/install_config/installing-customizing.html#provisioning-kernel-module-to-ocp_installing-customizing).
In case of ROKS clusters, the same can be accomplished using the `daemonset` resource, as shown in the following example. :
```
apiVersion: apps/v1
kind: DaemonSet
...
    spec:
      hostNetwork: true
      hostPID: true
      hostIPC: true
      initContainers:
        - command: ["/bin/sh", "-c"]
          args:
            - >
              echo sctp >> /etc/modules-load.d/addtnl-modules.conf;
              echo 8021q >> /etc/modules-load.d/addtnl-modules.conf;
              ...
              sed -i '$s/^/#/' /etc/modprobe.d/sctp-blacklist.conf;
          image: alpine:3.6
          imagePullPolicy: IfNotPresent
          name: sysctl
          resources: {}
          securityContext:
            privileged: true
            capabilities:
              add:
                - NET_ADMIN
          volumeMounts:
            - name: modifysys
              mountPath: /
      containers:
        - resources:
            requests:
              cpu: 0.01
          image: alpine:3.6
          name: sleepforever
          command: ["/bin/sh", "-c"]
          args:
            - >
              while true; do
                sleep 100000;
              done
      volumes:
        - name: modifysys
          hostPath:
            path: /
```
An example manifest is also available ås a reference [daemonset sample](./examples/addtnl-kernel-modules.yaml) in the [examples](./examples/) folder).

## 14. Image registry configuration
The openshift local image registry is not enabled by default on Satelite/ROKS clusters. This is unlike RH OCP clusters where the registry is enabled by default. In order to enable the openshift registry, the `storage` and `managementState` settings need to be updated in the `imageregistry` config as follows on the ROKS cluster.
```
apiVersion: imageregistry.operator.openshift.io/v1
kind: Config
metadata:
  ...
spec:
  ...
  storage:
    emptyDir: {}
    managementState: Managed
    ...
```
IBM Cloud documentation provides some [guidance in setting up the internal container image register](https://cloud.ibm.com/docs/openshift?topic=openshift-satellite-clusters#satcluster-internal-registry) as well.

Additionally, ROKS does not support propagating/applying mirror registry updates made via `ImageContentSourcePolicy` resource to the worker nodes, unlike on [RH OCP where it is supported](https://docs.openshift.com/container-platform/4.9/openshift_images/image-configuration.html#images-configuration-registry-mirror_image-configuration). So, on ROKS clusters, it is recommended to manuallly update the `/etc/containers/registries.conf` file on each of the worker nodes to configure mirror registry settings, and subsequently reboot the nodes on after the other. This procedure is similar to [the steps 13 and 14 documented here for a different purpose](https://cloud.ibm.com/docs/openshift?topic=openshift-openshift-storage-odf-private#odf-private).
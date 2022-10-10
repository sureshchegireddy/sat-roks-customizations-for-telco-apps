# sat-roks-customizations-for-telco-apps
This repository provides some guidance on openshift customizations that we discovered for typical networking and/or telco workloads/applications. The repo specifically targets environments where IBM's managed openshift service (ROKS) cluster(s) running on IBM Cloud Satellite need some alternative methods for configuring required capabilities in openshift, as compared to the same on Red Hat (RH) OpenShift Container Platform (OCP).

With IBM Cloud Satellite/ROKS (link to announcement) now supporting CoreOS-enabled locations and hosts, some of the custom configurations needed by typical network applications require alternate methods in ROKS as compared to OCP. 

Some of the key capabilities that needed alternative way of configuring on ROKS included the following:
1. Hugepages setup
2. SCTP enablement
3. SR-IOV network node policy config
4. CPU Manager and Single-NUMA-Node config
5. Storage driver install based on [Satellite storage template](https://cloud.ibm.com/docs/satellite?topic=satellite-sat-storage-template-ov#storage-template-flow)
6. SCC privileges for application deployment
7. externalIP support for services

In general the RH OCP documentation is the usual reference for configuring such custom settings. However, certain resources like `MachineConfig` are not supported on ROKS (IBM's managed Openshift service), and require alternative method for conifguring the capability.

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

## 3. SR-IOV network node policy config
This process is likely the same of both RH OCP and ROKS, and is based on the type of SR-IOV device on the bare metal servers used for the OpenShift workers.

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


## 5. Satellite storage driver install based on storage template
IBM Cloud Satellite provides storage templates for a set of storage providers/drivers. This is an alternative to installing from Catalog, OperatorHub, Helm charts, etc. This concept and process is well captured in [IBM documentation](https://cloud.ibm.com/docs/satellite?topic=satellite-sat-storage-template-ov).

## 6. SCC privileges for application deployment
Satellite/ROKS requires the default service account in a project/namespace to have elevated SCC privileges. This is necessary when a deployment in the namespace attempts to create pods.
One workaround is to identify the default `serviceaccount` used in the namespace for deployment, and add it as an `annotation` in the `privileged` SCC.

## 7. ExternalIP support for services
Satellite/ROKS does not allow `deployments` to create `services` that have an `externalIP` defined. However, and `externalIP` can be patched after the `service` is created. This `externalIP` is only reachable within the ROKS cluster, unlike in RH OCP.
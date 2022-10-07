# sat-roks-customizations-for-telco-apps
This repository provides some guidance on openshift customizations needed for typical telco workloads. The repo specifically targets environments where IBM's managed openshift service (ROKS) cluster(s) running on IBM Cloud Satellite need some alternative methods for configuring required capabilities in openshift, as compared to the same on Red Hat (RH) OpenShift Container Platform (OCP).

With IBM Cloud Satellite/ROKS (link to announcement) now supporting CoreOS-enabled locations and hosts, some of the custom configurations needed by typical network applications require alternate methods in ROKS as compared to OCP. 
Some of the key capabilities that need a different way of configuring on ROKS includes the following:
1. Hugepages setup
2. SCTP enablement
3. SR-IOV network node policy config
4. CPU Pinning and Isolation (CPU manager config)
5. NUMA Awareness
6. Satellite storage driver install based on storage template (https://cloud.ibm.com/docs/satellite?topic=satellite-sat-storage-template-ov#storage-template-flow)
7. SCC privileges for application deployment
8. externalIP support for services

In general the RH OCP documentation is the usual reference for configuring such custom settings. However, certain resources like MachineConfig are not supported on ROKS (IBM's managed Openshift service), and require alternative method for conifguring the capability.

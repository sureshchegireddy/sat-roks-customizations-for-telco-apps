# sat-roks-customizations-for-telco-apps
This repository provides some guidance on openshift customizations needed for typical telco workloads. The repo specifically targets how IBM's managed openshift (ROKS) running on IBM Cloud Satellite needs some alternative methods for configuring required capabilities in openshift, as compared to the same on Red Hat OCP.

With IBM Cloud Satellite/ROKS (link to announcement) now supporting CoreOS-enabled locations and hosts, some of the custom configurations needed by typical network applications require alternate methods in ROKS as compared to OCP. 
Some of the key capabilities that need different way of configuring on ROKS includes the following:
	a. Hugepages setup
	b. SCTP enablement
	c. SR-IOV network node policy config
	d. CPU Pinning and Isolation (CPU manager config)
	e. NUMA Awareness
	f. Satellite storage driver install based on storage template (https://cloud.ibm.com/docs/satellite?topic=satellite-sat-storage-template-ov#storage-template-flow)

The documentation for Red Hat OpenShift is the usual reference for configuring such custom settings. However, certain resources like MachineConfig are not supported on ROKS (IBM's managed Openshift service), and require alternative method for conifguring the capability.

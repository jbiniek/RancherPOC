* # POC Requirements Rancher

  ## Architecture

  * 1 highly available Rancher Management Cluster
  * 1 or more highly available Downstream Workload Cluster
  * ...

  ![Architecture](https://rancher.com/docs/img/rancher/rancher-architecture-rancher-api-server.svg)

  ## Summary

  The following things should be prepared before the POC starts:

  * Hardware for Rancher Management Server and Downstream clusters, including network setup, OS, SSH access and container runtime
  * Management workstation or local workstation with access to the VMs and all necessary CLI tools
  * TLS certificate for Rancher
  * DNS entry for Rancher
  * Loadbalancer in front of Rancher
  * TLS certificates for test workloads
  * DNS entries for test workloads
  * Loadbalancer in front of each downstream cluster
  * Rancher Helm Chart is accessible rom the management workstation
  * All necessary Rancher and Kubernetes Docker Images are either accessible by all servers directly, or through a mirror/proxy, or available in an internal Docker registry

  The details are described below.

  ## Prerequisites

  ### Hardware

  | Amount | Compute Type | CPU    | RAM  | Disk Capacity     | Role                                                         |
  | ------ | ------------ | ------ | ---- | ----------------- | ------------------------------------------------------------ |
  | 3      | VM           | 2 vCPU | 8GB  | 35GB >= 1000 IOPS | Rancher Management nodes                                     |
  | 3-n    | VM           | 2 vCPU | 8GB  | 35GB >= 1000 IOPS | Downstream workload cluster nodes. Additional worker nodes could also be larger. |

  ### Operating System Requirements

  - OS
    - Ubuntu 16.04, 18.04, 20.04 
    - RHEL/CentOS 7.5, 7.6, 7.7, 7.8
    - Oracle Linux 7.6, 7.7

    - SLES 12 SP5, 15 SP2, 15SP3
  - SSH access to all virtual machines
  
  ### CLI Tools and binaries

  The following CLI tools and binaries are needed on a management workstation to setup and manage Rancher and Kubernetes:

* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [helm](https://helm.sh/docs/intro/install/)
  
  
### Networking

- Application cluster nodes should be connected to the same VLAN and have unrestricted connectivity within the VLAN.  Network configuration in general must satisfy the following port/communication requirements:
  - https://rancher.com/docs/rancher/v2.x/en/installation/requirements/ports
- It is recommended to disable `firewalld`, for details see [here](https://rancher.com/docs/rancher/v2.x/en/installation/resources/advanced/firewall/)
  - Rancher Management hosts can be connected to same or separate VLAN. In the latter case, the network loadbalancer endpoint must be reachable from the other VLAN.
  
  ### Other requirements
  
#### Rancher Management Cluster

- TLS/SSL certificate for the Rancher management UI & API
  
- Layer 4 TCP load balancers in front of the Rancher Management cluster nodes proxying to 
  
  - the cluster's ingress controller (Ports 80 and 443)
    - the Kubernetes API Server (Port 6443)

    The LB could be a software LB (nginx, haproxy, ...), a hardware LB (F5 Big IP, ...) or keepalived
  
- DNS entry for the Rancher management console endpoint pointing to the LB in front of the ingress controller
  


#### Each Downstream Cluster

- DNS Alias (or wildcard DNS, e.g. *.prod.cluster.acme.com) for the application deployed in the Downstream Cluster
  
- TLS/SSL certificates for applications that should be exposed
  
- Layer 4 TCP load balancers in front of each cluster's worker nodes proxying to 
  
  - the cluster's ingress controller (Ports 80 and 443)
    - the Kubernetes API Server (Port 6443)

    The LB could be a software LB (nginx, haproxy, ...), a hardware LB (F5 Big IP, ...) or keepalived
  
- DNS entries (or wildcard DNS entries) for applications that should be exposed pointing to the LBs in front of the ingress controllers
  
#### Authentication

- For Active Directory integration, a service account should be available that has permissions to lookup users and groups and perform binds for authentication
  
#### Airgapped setups

- For fully airgapped install (no internet connectivity) an on-premise Container registry must be available to mirror the Rancher installation images
  - Network configuration must satisfy communication requirements from Kubernetes and Rancher
- For an airgapped installation:
    - Internal Docker registry or Docker proxy/mirror
    - For K3S follow: https://rancher.com/docs/k3s/latest/en/installation/airgap/#prepare-the-images-directory-and-k3s-binary
    - For RKE2: https://docs.rke2.io/install/airgap/
    - For Rancher&RKE: All necessary images are mirrored, for a complete list see [Collect and Publish Images to your Private Registry](https://rancher.com/docs/rancher/v2.x/en/installation/other-installation-methods/air-gap/populate-private-registry/). E.g. for Rancher 2.5.5:
      - Image list: https://github.com/rancher/rancher/releases/download/v2.5.5/rancher-images.txt
      - Image digests: https://github.com/rancher/rancher/releases/download/v2.5.5/rancher-images-digests.txt
    - The Rancher Helm Chart repo is available under https://releases.rancher.com/server-charts/stable. The repository can either be mirrored or the helm chart can be fetched manually during the installation
    - For Installation of Monitoring and other apps, the Rancher charts repository needs to be mirrored, either the [Git Repo](https://github.com/rancher/charts) or the [Helm Chart Repo](https://charts.rancher.io)
  
  ## Rancher Management Server
  
  Rancher should be installed in a highly available RKE cluster. This cluster should have 3 nodes with the roles etcd+control-plane+worker. See [Installing Rancher on a Kubernetes cluster](https://rancher.com/docs/rancher/v2.x/en/installation/k8s-install/).
  
  A layer 4 TCP load balancer is set up in front of the nodes pointing to the ingress controllers. A DNS entry is created for the loadbalancer IP.
  
  ### Authentication and Authorisation
  
  Authentication with the internal Active Directory should be set up and tested. The integrated Roles and permission system should be tested. See [Authentication and Permissions](https://rancher.com/docs/rancher/v2.x/en/admin-settings/).
  
  ## Downstream workload cluster
  
  At least one downstream Kubernetes clusters should be created. The clusters should be highly available, that means at least 3 nodes with the role etcd+control-plane. And at least 2 nodes with the worker role (minimum of 3 nodes in total).
  
  A layer 4 TCP load balancer is set up in front of the nodes pointing to the ingress controllers. A DNS entry is created for the loadbalancer IP.
  
  ### Authentication and Authorisation
  
  Authentication and authorisation of users and groups to the downstream cluster should be configured. See [Adding Users to Clusters](https://rancher.com/docs/rancher/v2.x/en/cluster-admin/cluster-access/cluster-members/).
  
  ### Storage
  
  If necessary, a persistent storage solution should be configured and tested.
  
  ### Pod Security and Network Policies
  
  The configuration of Pod Security and Network Policies should be tested. This could be [Longhorn](https://longhorn.io) or any other storage solution that integrates with Kubernetes.
  
  ### Security Scans
  
  The CIS security scans integrated in Rancher should be tested, see [CIS Scans](https://rancher.com/docs/rancher/v2.x/en/cis-scans/).
  
  ### Log Management
  
  The Rancher integrated log shipping should be activated and tested, see [Rancher Logging](https://rancher.com/docs/rancher/v2.x/en/cluster-admin/tools/logging/).
  
  ### Monitoring
  
  The Rancher integrated monitoring and alerting should be activated and tested, see [Rancher Monitoring](https://rancher.com/docs/rancher/v2.x/en/cluster-admin/tools/monitoring/).
  
  ### Istio Service Mesh
  
  The Rancher integrated Istio Service Mesh should be activated and tested, see [Istio](https://rancher.com/docs/rancher/v2.x/en/istio/).
  
  ### Continuous Delivery
  
  The integrated GitOps Continuous Deliver solution should be tested, see [GitOps at Scale](https://rancher.com/docs/rancher/v2.x/en/deploy-across-clusters/fleet/).
  
  ### Backup and disaster recovery
  
  Backups and disaster recovery of both the Rancher Management server and downstream cluster should be configured and tested. See [Backups and Disaster Recovery](https://rancher.com/docs/rancher/v2.x/en/backups/).
  
  ### Cluster upgrades
  
  Zero-downtime cluster upgrades should be tested. See [Upgrading and Rolling Back Kubernetes](https://rancher.com/docs/rancher/v2.x/en/cluster-admin/upgrading-kubernetes/).
  
  ## Optional
  
  Automate the setup of Rancher and downstream clusters.
  
  Examples:
  
  * https://github.com/puzzle/ansible-rancher
  * https://github.com/rancher/quickstart

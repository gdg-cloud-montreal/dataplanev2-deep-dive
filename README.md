

### 1.1 Create GKE Cluster with Dataplane v2 Support
**Step 1** Enable the Google Kubernetes Engine API.

```
gcloud services enable container.googleapis.com
```

**Step 2** From the cloud shell, run the following command to create a cluster with 1 node:
```
gcloud container clusters create k8s-networking \
--zone us-central1-c \
--enable-ip-alias \
--create-subnetwork="" \
--network=default \
--enable-dataplane-v2 \
--num-nodes 2
```

!!! note
    Dataplane v2 CNI on GKE is Google's integration of Cilium Networking based on EBPF.
    If you prefer using Calico CNI you need to replace 

**Output:**
```
NAME: k8s-networking
LOCATION: us-central1-c
MASTER_VERSION: 1.22.8-gke.202
MASTER_IP: 34.66.47.138
MACHINE_TYPE: e2-medium
NODE_VERSION: 1.22.8-gke.202
NUM_NODES: 2
STATUS: RUNNING
```
**Step 3** Authenticate to the cluster.
```
gcloud container clusters get-credentials k8s-networking --zone us-central1-c
``` 


### 1.2 Kubernetes Network Policy

Let's secure our 2 tier application using Kubernetes Network Policy!

**Task:** We need to ensure the following is configured

  *  Your namespace deny by default all `Ingress` traffic including from outside the cluster (Internet), With-in the namespace, and from inside the Cluster
  * `mysql` pod can be accessed from the `gowebapp` pod
  * `gowebapp` pod can be accessed from the Internet, in our case GKE Ingress (HTTPs Loadbalancer) and it must be configure to allow the appropriate HTTP(S) load balancer health check IP ranges FROM CIDR: `35.191.0.0/16` and `130.211.0.0/22` or
  to make it less secure From any Endpoint CIDR: `0.0.0.0/0`
  

### Prerequisite
In order to ensure that Kubernetes can configure Network Policy we need to make sure our CNI supports this feature.  As we've learned from class Calico, WeaveNet and Cilium are CNIs that support network Policy.

Our GKE cluster is already deployed using Cilium CNI. To check that calico is installed in your cluster, run:

```
 kubectl -n kube-system get ds -l k8s-app=cilium -o wide
```

**Output** 

```
NAME    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE   CONTAINERS     IMAGES                                  
anetd   2         2         2       2            2           kubernetes.io/os=linux   45m   cilium-agent   gke.gcr.io/cilium/cilium:v1.11.1-gke3.3
```

!!! note
    `anetd` is a Cilium agent DaemonSet that runs on each node.

!!! result
    Our GKE cluster running `cilium:v1.11.1` which Cilium version 1.11.1


### 4.1 Configure `deny-all` Network Policy inside the Namespace

**Step 1** Deploy `default-deny`

```
kubectl apply -f default-deny.yaml   # deny-all Ingress Traffic inside Namespace
```

Verify Policy:

```
kubectl describe netpol default-deny
```


!!! result
    <none> - blocking the specific traffic to all pods in this namespace

**Step 3** Verify that you can't access the application through Ingress anymore:

Visit the IP address in a web browser

```
$Ingress IP
```

!!! result
    can't access the app through ingress loadbalancer anymore.

### 4.2 Configure Network Policy for `gowebapp-mysql`

**Step 1** Using [Cilium Editor](https://editor.cilium.io/) test you Policy for Ingress traffic only

Open Browser, Upload created Policy YAML. 

!!! result
    `mysql` pod can only communicate with `gowebapp` pod

**Step ** Deploy `gowebapp-mysql-netpol` 

```
kubectl apply -f gowebapp-mysql-netpol.yaml   # `mysql` pod can be accessed from the `gowebapp` pod
```

Verify Policy:

```
kubectl describe netpol backend-policy
```

!!! result
    `run=gowebapp-mysql` Allowing ingress traffic only From PodSelector: `run=gowebapp` pod

### 4.3 Configure Network Policy for `gowebapp`

**Step 1** Configure a Network Policy to allow access for the Healthcheck IP ranges needed for the Ingress Loadbalancer, and hence allow access through the Ingress Loadbalancer. The IP rangers you need to enable access from are `35.191.0.0/16` and `130.211.0.0/22`


**Step 2** Using [Cilium Editor](https://editor.cilium.io/) test you Policy for Ingress traffic only

Open Browser, Upload created Policy YAML. 

!!! result
    `gowebapp` pod can only get ingress traffic from CIDRs: `35.191.0.0/16` and `130.211.0.0/22`


**Step 3** Deploy `gowebapp-netpol` Policy app under ~/$MY_REPO/deploy/

```
kubectl apply -f gowebapp-netpol.yaml  # `gowebapp` pod from Internet on CIDR: `35.191.0.0/16` and `130.211.0.0/22`
```

Verify Policy:

```
kubectl describe netpol frontend-policy
```

!!! result
    `run=gowebapp` Allowing ingress traffic from CIDR: 35.191.0.0/16 and 130.211.0.0/22


**Step 4**  Test that application is reachable via `Ingress` after all Network Policy has been applied.


Get the Ingress IP address, run the following command:

```
kubectl get ing gowebapp-ingress
```

In the command output, the Ingress' IP address is displayed in the ADDRESS column. Visit the IP address in a web browser

```
$ADDRESS/*
```


**Step 5** Verify that `NotePad` application is functional (e.g can login and create new entries)
  
  
### 1.3 Dataplane v2 Troubleshooting

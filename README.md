# CKA Exam Prep
[CKA Curriculum](https://github.com/cncf/curriculum)

## Tips and tricks
* Useful sites allowed during test
  * kubernetes.io/docs
    * Kubectl Cheatsheet
    * Concepts
  * kubernetes.io/blog
  * github.com/kubernetes
* Kubectl explain is your friend
  * `kubectl explain <resource>.<key>`
  * `kubectl explain pod.spec`
* Pay attention to question context
* Always exit on SSH
* [YAML Basics in Kubernetes](https://developer.ibm.com/tutorials/yaml-basics-and-usage-in-kubernetes/)
  * It is a little confusing on when to include a '-' for a list of items under a field
  * My best advice is to use `kubectl explain` on a resource and see if it includes `[]` in its RESOURCE type
    * Example: `kubectl explain pod.spec.affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution`
      * `RESOURCE: preferredDuringSchedulingIgnoredDuringExecution <[]Object>`
* Use `kubectl <command> -h` to get more info on that command
* Other great resources:
  * https://github.com/walidshaari/Kubernetes-Certified-Administrator


## Scheduling - 5%
### Use label selectors to schedule pods
[Assinging Pods to Nodes](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/)
[Label-Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors)
  * The label selector is the core grouping primitive in Kubernetes

* Using labels to filter pods
```
kubectl get pods -l environment=production,tier=frontend
kubectl get pods -l 'environment in (production),tier in (frontend)'
```

#### [nodeSelector](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector) - simpliest recommended method
1. Label Node
  * For a [more secure labeling option](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#node-isolation-restriction), add the prefix `node-restriction.kubernetes.io/` to the label key
```
kubectl get nodes
kubectl label nodes <node_name> <label_key>=<label_value>
```
2. Verify node label
```
kubectl get nodes --show-labels
kubectl describe node <node_name>
```
3. Add nodeSelector field to pod spec
  * Create pod yaml to edit
```
kubectl run <pod_name> --image=<pod_image> -o yaml --dry-run > nodeSelector.yaml
```
  * Edit yaml and add the nodeSelector field under pod.spec
```
kind: Pod
spec:
  nodeSelector:
    <label_key>: <label_value>
  containers:
  - name: ...
```

#### [Affinity and anti-affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity)
More expressive and flexible way to affect scheduling
##### nodeAffinity
Node affinity is conceptually similar to `nodeSelector` – it allows you to constrain which nodes your pod is eligible to be scheduled on, based on labels on the node.
1. Label Node
2. Add affinity field and nodeAffinity sub-field to pod spec
  * Create pod yaml to edit
```
kubectl run <pod_name> --image=<pod_image> -o yaml --dry-run > nodeAffinity.yaml
```
  * Edit yaml and add the affinity.nodeAffinity fields under pod.spec
```
kind: Pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: <node-label-key>
            operator: In    # Valid operators are In, NotIn, Exists and DoesNotExist. 
            values:
            - <node-label-value>
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1   # required field
        preference:
          matchExpressions:
          - key: <another-node-label-key>
            operator: In
            values:
            - <another-node-label-value>
  containers:
  - name: ...
```
##### podAffinity and podAntiAffinity
Affects scheduling based on labels on **pods** that are already running on the node rather than based on labels on nodes
1. Label Node
2. Add affinity field and nodeAffinity sub-field to pod spec
  * Create pod yaml to edit
```
kubectl run <pod_name> --image=<pod_image> -o yaml --dry-run > podAffinity.yaml
```
  * Edit yaml and add the affinity.podAffinity and/or affinity.podAntiAffinity fields under pod.spec
```
kind: Pod
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: <label_key>
            operator: In
            values:
            - <label_value>
        topologyKey: failure-domain.beta.kubernetes.io/zone   # required field
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: <label_key>
              operator: In
              values:
              - <label_value>
          topologyKey: failure-domain.beta.kubernetes.io/zone   # required field
  containers:
  - name: ...
```
  * topologyKey
    * In principle, the topologyKey can be any legal label-key
    * Note: Pod anti-affinity requires nodes to be consistently labelled, i.e. every node in the cluster must have an appropriate label matching `topologyKey`. If some or all nodes are missing the specified `topologyKey` label, it can lead to unintended behavior.

### Understand the role of Daemonsets
[DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed from the cluster, those Pods are garbage collected. Deleting a DaemonSet will clean up the Pods it created.
* Some typical uses of a DaemonSet are:
  * running a cluster storage daemon, such as `glusterd`, `ceph`, on each node
  * running a logs collection daemon on every node, such as `fluentd` or `filebeat`
  * running a node monitoring daemon on every node, such as Prometheus, New Relic agent, Sysdig agent, etc

* Required fields
  * .spec.template
    * This a pod template. It has exactly the same schema as a pod
  * .spec.selector
    * This field is a pod selector

#### How Daemon Pods are Scheduled
* A DaemonSet ensures that all eligible nodes run a copy of a Pod
* DaemonSet pods are created and scheduled by the DaemonSet controller instead of the Kubernetes scheduler
* Daemon pods respect taints and tolerations but [some tolerations are added to DaemonSet pods automatically](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/#taints-and-tolerations)

### Understand how resource limits can affect Pod scheduling
[Configure Default Memory Requests and Limits for a Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)
[Kube Scheduler](https://kubernetes.io/docs/concepts/scheduling/kube-scheduler/)
* For every newly created pod or other unscheduled pods, kube-scheduler selects an optimal node for them to run on. However, every container in pods has different requirements for resources and every pod also has different requirements. Therefore, existing nodes need to be filtered according to the specific scheduling requirements.
* In a cluster, Nodes that meet the scheduling requirements for a Pod are called feasible nodes. If none of the nodes are suitable, the pod remains unscheduled until the scheduler is able to place it.
* kube-scheduler selects a node for the pod in a 2-step operation:
  1. Filtering
    * The filtering step finds the set of Nodes where it’s feasible to schedule the Pod. For example, the `PodFitsResources` filter checks whether a candidate Node has enough available resource to meet a Pod’s specific resource requests. After this step, the node list contains any suitable Nodes; often, there will be more than one. If the list is empty, that Pod isn’t (yet) schedulable
  2. Scoring
    * In the scoring step, the scheduler ranks the remaining nodes to choose the most suitable Pod placement. The scheduler assigns a score to each Node that survived filtering, basing this score on the active scoring rules.
  * Finally, kube-scheduler assigns the Pod to the Node with the highest ranking. If there is more than one node with equal scores, kube-scheduler selects one of these at random.
* `PodFitsResources`: Checks if the Node has free resources (eg, CPU and Memory) to meet the requirement of the Pod

#### [Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)
* By default, containers run with unbounded compute resources on a Kubernetes cluster. With Resource quotas, cluster administrators can restrict the resource consumption and creation on a namespace basis
* Limit Range is a policy to constrain resource by Pod or Container in a namespace
* A limit range, defined by a LimitRange object, provides constraints that can:
  * Enforce minimum and maximum compute resources usage per Pod or Container in a namespace.
  * Enforce minimum and maximum storage request per PersistentVolumeClaim in a namespace.
  * Enforce a ratio between request and limit for a resource in a namespace.
  * Set default request/limit for compute resources in a namespace and automatically inject them to Containers at runtime.

#### [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
* A resource quota, defined by a `ResourceQuota` object, provides constraints that limit aggregate resource consumption per namespace. It can limit the quantity of objects that can be created in a namespace by type, as well as the total amount of compute resources that may be consumed by resources in that project
* If creating or updating a resource violates a quota constraint, the request will fail with HTTP status code 403 FORBIDDEN with a message explaining the constraint that would have been violated.
* If quota is enabled in a namespace for compute resources like cpu and memory, users must specify requests or limits for those values; otherwise, the quota system may reject pod creation. Hint: Use the LimitRanger admission controller to force defaults for pods that make no compute resource requirements

### Understand how to run multiple schedulers and how to configure Pods to use them
[Configure Multiple Schedulers](https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/)

#### Package the scheduler
* Package your scheduler binary into a container image

#### Define a Kubernetes Deployment for the scheduler
* An important thing to note here is that the name of the scheduler specified as an argument to the scheduler command in the container spec should be unique. This is the name that is matched against the value of the optional spec.schedulerName on pods, to determine whether this scheduler is responsible for scheduling a particular pod
* Note also that we created a dedicated service account my-scheduler and bind the cluster role system:kube-scheduler to it so that it can acquire the same privileges as kube-scheduler

#### Run the second scheduler

#### Specify scheduler for pods
```
kind: Pod
spec:
  schedulerName: my-scheduler
  containers:
  - name: ...
```

#### Verify pods were scheduler using the desired scheduler
`kubectl get events`

### Manually schedule a pod without a scheduler (aka Static Pods)
[Create Static Pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
A mirror pod gets created on the Kube API server for each static pod. This makes them visible but they cannot be controlled from there

1. SSH into node
* ssh username@my-node1
2. Choose directory to save yaml (example /etc/kubelet.d)
```
mkdir /etc/kubelet.d/
cat <<EOF >/etc/kubelet.d/static-web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    role: myrole
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: web
          containerPort: 80
          protocol: TCP
EOF
```
3. Configure your kubelet on the node to use this directory by running it with `--pod-manifest-path=/etc/kubelet.d/` argument
`KUBELET_ARGS="--cluster-dns=<DNS_EP> --cluster-domain=kube.local --pod-manifest-path=/etc/kubelet.d/"`

4. Restart the kubelet
`systemctl restart kubelet`

* Observe static pod behavior
`docker ps`

### Display scheduler events
* Via `kubectl describe`
`kubectl describe pods <pod_name>`

### Know how to configure the Kubernetes scheduler
[Configure the Kubernetes Scheduler](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-scheduler)

1. Download and install the Kubernetes Controller Binaries
```
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-scheduler"

{
  chmod +x kube-scheduler
  sudo mv kube-scheduler /usr/local/bin/
}
```

2. Configure the Kubernetes Scheduler
* Move the kube-scheduler kubeconfig into place
`sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/`

* Create the kube-scheduler.yaml configuration file:
```
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

* Create the kube-scheduler.service systemd unit file
```
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

3. Start the Controller Services
```
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-scheduler
  sudo systemctl start kube-scheduler
}
```

4. Verification
`kubectl get componentstatuses --kubeconfig admin.kubeconfig`

## Core Concepts - 19%
### Understand the Kubernetes API primitives
[Understanding Kubernetes Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)

#### Key Points
* A Kubernetes object is a “record of intent”–once you create the object, the Kubernetes system will constantly work to ensure that object exists. By creating an object, you’re effectively telling the Kubernetes system what you want your cluster’s workload to look like; this is your cluster’s desired state.
* Required fields in a `yaml` file for any Kubernetes object
  * `apiVersion` - which version of the Kubernetes API you're using to create this object
  * `kind` - what kind of object you want to create
  * `metadata` - data that helps uniquely identify the object, including a `name` string, `UID`, and optional `namespace`
  * `spec` - what state you desire for the object
* The precise format of the object `spec` is different for every Kubernetes object

[Kubernetes API Overview](https://kubernetes.io/docs/reference/using-api/api-overview/)
#### Key Points
* Everything in the Kubernetes platform is treated as an API object and has a corresponding entry in the API
* Kubernetes stores its serialized state in etcd
* API Versioning - versions indicate the different levels of stability and support
  * Alpha - may contain bugs; feature is disabled by default; support may end at any time without notice
  * Beta - software is well tested; support for feature will not be dropped; features are enabled by default
  * Stable - these features appear in released software for many subsequent versions
* API Groups
  * API group is specified in REST path
    * `/apis/$GROUP_NAME/$VERSION`
  * Can use `CustomResourceDefinition` to extend API
  * Can also use API `aggregator` for a full set of Kubernetes API semantics to implement your own apiserver
* Enabling API groups and resources in the groups
  * You can enable or disable them by setting `--runtime-config` on the apiserver (accepts comma separated values)
  * Need to restart the apiserver and controller-manager to pick up changes
* Resources can be either cluster-scoped or namespace-scoped
* Efficient detection of changes with `--watch` flag
* Resources are deleted in two phases
  1. Finalization - using finalizers
  2. Removal

[Kubernetes Object Management](https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/)
#### Key Points
* Imperative commands
  * `kubectl run nginx --image=nginx`
  * `kubectl create deployment nginx --image=nginx`
* Imperative object configuration
  * `kubectl create -f nginx.yaml`
  * `kubectl delete -f nginx.yaml`
  * `kubectl replace -f nginx.yaml`
* Declarative object config
  * `kubectl diff -f configs/` will show what changes are going to be made
  * `kubectl apply -f configs/` will create or patch the live objects
  * `kubectl get -f config.yaml -o yaml` will print the live configuration
  * use `-R` to process directories recursively
  * Deleting objects
    * Recommended - `kubectl delete -f <filename>`
    * Alternative - `kubectl apply -f <directory/> --prune -l your=label`
      * `--prune` is in Alpha

### Understand the Kubernetes cluster architecture
[Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)

#### Key Points
* Control Plane Components
  * kube-apiserver
    * exposes the Kubernetes API and acts as the front end for the Kube control plane
    * only connection to etcd database
    * all actions are accepted and validated here
  * etcd
    * Consistent and HA key value store used as Kubes' backing store for all cluster data
    * Always have a backup plan for this data because cluster can become unresponsive if it becomes corrupt
    * Values are always added to the end instead of changing an entry; old copies are marked for removal
    * if simultaneous requests are made to update a value, first request wins; second request no longer has same version number and apiserver returns a `409` error; this error should be handled
    * For HA etcd, they communicate and elect a master
  * kube-scheduler
    * watches for newly created pods with no assigned node and selects a node for them to run on then writes bindings back to apiserver
    * Factors taken into account for scheduling decisions include individual and collective resource requirements, hardware/software/policy constraints, affinity and anti-affinity specifications, data locality, inter-workload interference and deadlines
    * Uses algorithms to determine the node to host a pod
  * kube-controller-manager
    * Runs controller processes (Deployments, Jobs, Pods, etc)
    * [Controllers](https://kubernetes.io/docs/concepts/architecture/controller/) are control loops that watch the state of your cluster, then make or request changes where needed. Each controller tries to move the current cluster state closer to the desired state
    * Controller include:
      * Node Controller: responsible for noticing and responding when nodes go down
      * Repliacation Controller: Responsible for maintaining the correct number of pods for every replication controller object in the system
      * Endpoints Controller: Populates the Endpoint object (joins Services and Pods)
      * Service Account and Token Controllers: Create default accounts and API access tokens for new namespaces
      * Route Controller: Responsible for configuring routes so containers can talk
    * cloud-controller-manager
      * interacts with the underlying cloud provider
      * The following controllers have cloud provider dependencies
        * Node controller: checks the cloud provider to determine if a node has been delete after it stops responding
        * Route Controller: sets up routes in the underlying cloud infrastructure
        * Service Controller: creates, updates, and deletes cloud provider load balancers
        * Volume Controller: creates, attaches, and mounts volumes, and interacts with cloud provider to orchestrate volumes
* Node Components
  * kubelet
    * accepts api calls for pod specs 
    * runs on each node and makes sure that containers are running in a pod according to pod spec
    * mounts volumes to pods
    * downloads secrets
    * passes requests to local container engine
    reports status of pods and node to kube-apiserver for eventual persistence in etcd
  * kube-proxy
    * maintains network rules on each node, allowing network communication to your pods from inside or outside the cluster
    * uses the OS packet filtering layer to forward traffic if available, otherwise forwards traffic itself
    * uses iptables entries for routes
      * since this isn't scalable, ipvs is planned to replace this
  * Container Runtime
    * responsible for running containers
    * Docker, containerd, CRI-O are supported
* Addons
  * DNS
    * cluster DNS; containers started by Kube automatically include this DNS server in their searches
  * Web UI (Dashboard)
  * Container Resource Monitoring
  * Cluster-level Logging
    * Everything a containerized app writes to `stdout` and `stderr` is handled and redirected somewhere by a container engine
    * System component logs - on machines with systemd, the kubelet and container runtime write to journald
      * Otherwise they write to `.log` files in the `/var/log` directory

### Understand Services and other network primitives
[Services](https://kubernetes.io/docs/concepts/services-networking/service/)

#### Key Points
* Each service is a microservice handling traffic, NodePort/LB, to distribute inbound requests among pods
* Handles access policies for inbound requests
  * connect pods together
  * expose pods to internet
* A Service is an abstraction which defines a logical set of Pods and a policy by which to access them
  * Pods are selected by labels
* Service without a Pod selector can point to external resources
  * With no selector specified, the corresponding Endpoint object is not created automatically
  * Must manually map the Service to the network address and port by adding an Endpoint object manually
* ServiceTypes
  * ClusterIP - default
    * exposes the Service on a cluster-internal IP that is only reachable from within the cluster
  * NodePort
    * exposes the Service on each Node's IP at a static port; also creates a ClusterIP and uses a random port on the node
  * LoadBalancer
    * exposes the Service externally using a cloud provider's LB; NodePort and ClusterIP are automatically created
  * ExternalName
    * maps the Service to the contents of the `externalName` field, by returning a CNAME record with its value
* Discovering services - two primary modes
  * Environment variables
    * `{SVCNAME}_SERVICE_HOST` and `{SVCNAME}_SERVICE_PORT` variables
    * The service must be created before the client pods come into existence
  * DNS
    * Should almost always use DNS for service discovery
    * The DNS server watches the Kube API for new Services and creates a set of DNS records for each one
    * Kube DNS is the only way to access `ExternalName` Services
* Configuring a service
  * `kubectl expose deploy/nginx --port=80 --type=NodePort` will create a svc and expose a random port on all nodes

## Networking - 11%
### Understand the networking configuration on the cluster nodes
[Illustrated Guide to Kubernetes Networking](https://itnext.io/an-illustrated-guide-to-kubernetes-networking-part-1-d1ede3322727)
[Illustrated Guide to Kube Networking - Overlay Networks](https://itnext.io/an-illustrated-guide-to-kubernetes-networking-part-2-13fdc6c4e24c)

#### Key Points
* Intra-node communication
  * Every kube node (Linux-based) has a root network namespace
    * The main network interface `eth0` is in this root netns
  * Each pod has its own netns, with a virtual ethernet pair connecting it to the root netns
    * Each pod has its own `eth0` interface but it is connected via a `veth` interface on the host
  * In order for these pods to talk to each other, the container engine creates a bridge that connects each of the virtual ethernet interfaces together
* Inter-node communication
  * This can be done in almost anyway but is mainly handled by an overlay network such as Calico, Flannel, etc
  * Can be down using L2 (ARP across nodes), L3 (IP routing using route tables), or an overlay network
  * Each node is assigned a unique CIDR block for Pod IPs, so each Pod has a unique IP
  * Route tables are use to route between the different CIDR blocks on each node
    * Can use `route -n` to view route table on a node
  * Overlay networks
    * They essentially encapsulate a packet-in-packet which traverses the native network across nodes
    * Same as previous workflow but when local node's bridge needs to send it to another CIDR block, it has a route table entry configured to send it to the overlay network's interface

### Understand Pod networking concepts
[Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
[Networking Design Proposal](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/network/networking.md)

#### Key Points
* Pod-to-Pod communications
  * Every Pod gets its own IP address
    * This allows pods to be treated much like VMs or physical hosts for port allocation, naming, service discovery, load balancing, etc
  * Kube imposes the following networking requirements
    * Pods on a node can communicate with all pods on all nodes witout NAT
    * Agents on a node (system daemons, kubelet) can communicates with all pods on that node
    * Pods in the host network of a node can communicate with all pods on all nodes without NAT
  * This follows similar rules as VMs which makes it easy to port a VM app to a pod
  * All containers in a pod behave as if they are on the same host and share the same IP
  * The Pause container is used to get an IP Address and starts up before the rest of the containers
  * Containers can communicate with each other using
    * The lookback interface
    * Writing to files on a common filesystem
    * Via inter-process communication (IPC)
  * To use IPv4 and IPv6, when creating a service, you must create and endpoint for each address family separately

### Understand Service networking
[Service Networking](https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies)

#### Key Points
* Every node in a Kube cluster runs a `kube-proxy`, which is responsible for implementing a form of virtual IP for `Services` of types other than `ExternalName`
* kube-proxy
  * watches the Kube master for the addition and removal of Service and Endpoint objects
  * [user space proxy mode](https://kubernetes.io/docs/concepts/services-networking/service/#proxy-mode-userspace)
    * for each Service, it opens a port on the local node
    * any connections to this "proxy port" are proxied to one of the Service's backend Pods
    * the user-space then installs iptables rules which capture traffic to the Service's `clusterIP` and `port`. The rules redirect that traffic to the proxy port which proxies the backend Pod
    * By default it uses round robin algorithm
  * [iptables proxy mode](https://kubernetes.io/docs/concepts/services-networking/service/#proxy-mode-iptables)
    * for each Service, it installs iptable rules, which capture traffic to the Service's `clusterIP` and `port`, and the redirect that traffic to one of the Service's backend sets
    * For each Endpoint object, it installs iptables rules which select a backend Pod
    * this mode has a lower system overhead, because traffic is handled by Linux netfilter
    * If kube-proxy is running in iptables mode and the first Pod that’s selected does not respond, the connection fails. This is different from userspace mode: in that scenario, kube-proxy would detect that the connection to the first Pod had failed and would automatically retry with a different backend Pod
    * you can use Pod `readiness probes` to verify that backend Pods are working, so that kube-proxy in iptables mode only sees backends that test out as healthy
  * [IPVS proxy mode](https://kubernetes.io/docs/concepts/services-networking/service/#proxy-mode-ipvs)
    * Calls `netlink` interface to create IPVS rules and sync IPVS rules with Kube Services and Endpoints periodically. This ensures that IPVS status matches the desired state
    * Uses a hash table as the underlying data structure and works in the kernel space
    * This means it redirects traffix with lower latency than iptables, and supports higher throughput
    * also supports more options for balancing traffic to backend pods
    * To use this mode
      * Install the IPVS components on the Linux node before starting kube-proxy
      * When kube-proxy starts in IPVS mode, it verifies whether IPVS kernel modules are available and falls back to iptables mode if not

### Deploy and Configure a network load balancer
[Create an External Load Balancer](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/)
[Service Config](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)

#### Key Points
* This provides an externally-accessible IP address that sends traffic to the correct port on your cluster nodes provided your cluster runs in a supported environment and is configured with the correct cloud load balancer provider package
* Using kubectl
  * `kubectl expose deploy nginx --port=80 --target-port=80 --name=external-lb --type=LoadBalancer`
  * This command creates a new service using the same selectors set in the nginx deployment
* Find your IP address
  * `kubectl describe svc external-lb`
* `targetPort` defaults to the port value passed, but it could be set to a named port
* Switching traffic to a different `port` would maintain a client connection, while changing versions of the app

### Know how to use Ingress rules
[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

#### Key Points
* You can use Services to allow ingress traffic but it becomes hard to manage the more you have
* Ingress exposes HTTP and HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules on the ingress resource
  * It does not expose arbitrary ports or protocols; exposing services other than HTTP(S) to the internet typicaly uses a service of type NodePort or LoadBalancer
* It can be configured to give services externally-reachable URLs, load balance traffic, terminate SSL/TLS, and offer name based virtual hosting
* Prereqs
  * Must have an ingress controller such as ingress-nginx, traefik, HAProxy, etc
* The Ingress resource
  * Ingress rules
    * Each HTTP rule contains the following info:
      * An optional host. If provided, the rules apply to that host. If not, the rule applies to all inbound HTTP traffic through the IP address specificied
      * A list of paths, each of which has an associated backend defined with a `serviceName` and `servicePort`. Both the host and path must match the content of an incoming request before the load balancer directs traffic to the referenced Service
      * A backend is a combination of Service and port names. HTTP(S) requests to the Ingress that matches the host and path of the rule are sent to the listed backend
    * A default backend is often configured in an Ingress controller to service any requests that do not match a path in the spec
* Types of Ingress
  * Single Service - specifying a default backend with no rules
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: test-ingress
spec:
  backend:
    serviceName: testsvc
    servicePort: 80
```
  * Simple fanout - routes traffic from a single IP address to more than one service based on the HTTP URI being requested
```
foo.bar.com -> 178.91.123.132 -> / foo    service1:4200
                                 / bar    service2:8080
```
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: simple-fanout-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: service1
          servicePort: 4200
      - path: /bar
        backend:
          serviceName: service2
          servicePort: 8080
```
  * Name based virtual hosting - routing HTTP traffic to multiple host names at the same IP address
```
foo.bar.com --|                 |-> foo.bar.com service1:80
              | 178.91.123.132  |
bar.foo.com --|                 |-> bar.foo.com service2:80
```
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
  - host: bar.foo.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80
```

### Know how to configure and use the cluster DNS
[DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

#### Key Points
* Kubernetes DNS schedules a DNS Pod and Service on the cluster, and configures the kubelets to tell individual containers to use the DNS Service’s IP to resolve DNS names.
* What things get DNS names?
  * Every Service defined in the cluster is assigned a DNS name
  * Pods do too. Pod's DNS search list will include the Pod's own namespace the the cluster's default domain
* Services
  * A/AAAA records
    * Normal (not headless) Services are assign a DNS A/AAAA record
      * `my-svc.my-namespace.svc.cluster-domain.example` - this resolves to the `clusterIP` of the Service
    * Headless (no clusterIP) Services are also assigned a DNS record
      * `my-svc.my-namespace.svc.cluster-domain.example` - this resolves to the set of IPs of the pods selected by the service
  * SRV records
    * Are created for named ports that are part of Services
      * `_my-port-name._my-port-protocol.my-svc.my-namespace.svc.cluster-domain.example` - for a regular service, this resolves to the port number and the domain name
* Pods
  * Pod's hostname and subdomain fields
    * The pod spec has an optional `hostname` field which can be used to specify its hostname. When specified, it takes precedence over the Pod's name to be the hostname of the pod
    * The pod spec also has an option `subdomain` field
    * If pod hostname was `foo` and subdomain `bar`, its FQDN would be `foo.bar.my-namespace.svc.cluster-domain.example`
* Pod's DNS Policy
  * Can be set on a per-pod basis via the `dnsPolicy` field of a pod spec
  * `Default`: the pod inherits the name resolution config from the node it runs on
  * `ClusterFirst`: Any DNS query that does not match the configured cluster domain suffix is forwarded to the upstream nameserver inherited from the node
  * `ClusterFirstWithHostNet`: For pods running with hostNetwork
  * `None`: Allows a pod to ignore DNS settings from the Kube environment
* Pod's DNS Config
  * Allows users more control on the DNS settings for a pod
```
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster-domain.example
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

### Understand CNI
[Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
[CNI Github](https://github.com/containernetworking/cni)

#### Key Points
* CNI plugins: adhere to the appc/CNI specification, designed for interoperability
* Why develop CNI?
  * Containers are rapidly evolving and networking is not well addressed as it is highly environment-specific
  * Container runtimes and orchestrators solve this problem by making the network layer pluggable
* The CNI spec is language agnostic
* Can support advanced networking such as traffic shaping and MTU sizing
* Overlay networks
  * They essentially encapsulate a packet-in-packet which traverses the native network across nodes
  * Same as previous workflow but when local node's bridge needs to send it to another CIDR block, it has a route table entry configured to send it to the overlay network's interface
* [Flannel vs Calico](https://medium.com/@jain.sm/flannel-vs-calico-a-battle-of-l2-vs-l3-based-networking-5a30cd0a3ebd)
  * Flannel
    * Once the packet hits the flannel interface, it uses the Flannel daemon to capture packets and wrap then into a L3 packet for transport over the physical network using UDP with vxlan tagging (happens in kernel space so it is fast)
    * The next hop from the source container is always the Flannel interface
    * The daemon talks to the Kube apiserver to create a mapping for pod IPs to node IPs
    * When packet for other pod reaches destination node, the flannel interface de-capsulates and sends it into the root netns to get passed to the bridge interface and then on to the pod
  * Calico
    * Works at L3 by injecting a routing rule inside the container for the default gateway as its own interface
    * The container is then able to get the destiation IP of the target container, via proxy ARP, and traffic flows via normal network rules
    * Routes on hosts are synced via BGP using a BGP Client (Bird)
    * This eliminates the need for a network overlay interface

## Installation, Configuration, and Validation - 12%
[Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

### Design a Kubernetes Cluster
[Creating a Cluster](https://kubernetes.io/docs/tutorials/kubernetes-basics/create-cluster/)

#### Key Points
* Kubernetes coordinates a highly available cluster of computers that are connected to work as a single unit
* Kubernetes automates the distribution and scheduling of application containers acrodd a cluster in a more efficint way
* The Master coordinates the cluster
  * It is responsible for managing the cluster
* Nodes are the workers that run applications
  * Nodes communicate with the master using the kubernetes API called by the kubelet service

### Install Kubernetes masters and nodes
[Kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
[Create cluster using kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
#### Key Points
* Prereqs
  * Linux VM
  * 2 GB of RAM and 2 CPUs
  * Full network connectivity between all machines in the cluster
  * Swap must be disabled for the kubelet to work properly
* Verify MAC address and product_uuid are unique for every node
* Ensure iptables tooling does not use the nftables backend - it should use legacy mode
* Check that required ports are open
  * Control Plane
    * 6443 - Kube API server
    * 2379-2380 - etcd server cluent API
    * 10250, 10251, 10252 - Kubelet API, kube-scheduler, kube-controller-manager
  * Worker Nodes
    * 10250 - Kubelet API
    * 30000-32767 - NodePort services
* Install runtime
  * Docker, containerd, CRI-O
* Install kubeadm, kubelet, and kubectl
  * kubeadm will bootstrap the cluster
  * kubelet runs on all machines in your cluster and does things like starting pods and containers
  * kubectl is a cli util to talk to your cluster
  * These all need to be running the same version
* Configure cgroup driver used by kubelet on control-plane node
* Bootstrap first cluster master using kubeadm
```
kubeadm init --apiserver-advertise-address=$IP_ADDR --apiserver-cert-extra-sans=$IP_ADDR,$HOST_NAME \
  --node-name $HOST_NAME --pod-network-cidr=$POD_CIDR --control-plane-endpoint $HOST_NAME:6443
```
* Install pod network such as Calico, Flannel, or others
```
curl https://docs.projectcalico.org/$CALICO_VERSION/manifests/calico.yaml -O
sed -i -e "s?192.168.0.0/16?$POD_CIDR?g" calico.yaml
kubectl apply -f calico.yaml
```
* Add more nodes and control-plane nodes
  * Each additional node needs docker, kubelet, kubeadm, and kubectl installed
  * `kubeadm join`
  * If over 2+ hrs from master creation
    * To join worker nodes
      * `kubeadm token create --print-join-command`
    * To join master nodes
```
CERT_KEY=$(kubeadm alpha certs certificate-key)
kubeadm token create --print-join-command --certificate-key $CERT_KEY
```
* Remove node from cluster
  * `kubeadm reset`

### Configure secure cluster communication
[Provisioning a CA and Generating TLS Certs](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md)
[Manage TLS certs in a cluster](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)
[Cert Management with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/)

#### Key Points
* 
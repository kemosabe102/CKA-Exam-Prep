# CKA Exam Prep
* [CKA Curriculum](https://github.com/cncf/curriculum)

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
* [YAML Basics](https://learnxinyminutes.com/docs/yaml/)
* Use `kubectl <command> -h` to get more info on that command
* Other great resources:
  * https://github.com/walidshaari/Kubernetes-Certified-Administrator
  * https://github.com/burkeazbill/cka-studyguide

## Things to practice before the test
* Create yaml file of a resource by using `kubectl` command 
  * Basic pod
  * Deployment
  * Service?
* [General Tasks](https://multinode-kubernetes-cluster.readthedocs.io/en/latest/index.html)

## Scheduling - 5%
### Use label selectors to schedule pods
* [Assinging Pods to Nodes](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/)
* [Label-Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors)
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
* [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
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
* [Configure Default Memory Requests and Limits for a Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)
* [Kube Scheduler](https://kubernetes.io/docs/concepts/scheduling/kube-scheduler/)
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
* [Configure Multiple Schedulers](https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/)

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
* [Create Static Pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
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
* [Configure the Kubernetes Scheduler](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-scheduler)

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
* [Understanding Kubernetes Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)

#### Key Points
* A Kubernetes object is a “record of intent”–once you create the object, the Kubernetes system will constantly work to ensure that object exists. By creating an object, you’re effectively telling the Kubernetes system what you want your cluster’s workload to look like; this is your cluster’s desired state.
* Required fields in a `yaml` file for any Kubernetes object
  * `apiVersion` - which version of the Kubernetes API you're using to create this object
  * `kind` - what kind of object you want to create
  * `metadata` - data that helps uniquely identify the object, including a `name` string, `UID`, and optional `namespace`
  * `spec` - what state you desire for the object
* The precise format of the object `spec` is different for every Kubernetes object

* [Kubernetes API Overview](https://kubernetes.io/docs/reference/using-api/api-overview/)
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

* [Kubernetes Object Management](https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/)
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
* [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)

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
* [Services](https://kubernetes.io/docs/concepts/services-networking/service/)

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
* [Illustrated Guide to Kubernetes Networking](https://itnext.io/an-illustrated-guide-to-kubernetes-networking-part-1-d1ede3322727)
* [Illustrated Guide to Kube Networking - Overlay Networks](https://itnext.io/an-illustrated-guide-to-kubernetes-networking-part-2-13fdc6c4e24c)

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
* [Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
* [Networking Design Proposal](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/network/networking.md)

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
* [Service Networking](https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies)

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
* [Create an External Load Balancer](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/)
* [Service Config](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)

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
* [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

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
* [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

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
* [Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
* [CNI Github](https://github.com/containernetworking/cni)

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
* [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

### Design a Kubernetes Cluster
* [Creating a Cluster](https://kubernetes.io/docs/tutorials/kubernetes-basics/create-cluster/)

#### Key Points
* Kubernetes coordinates a highly available cluster of computers that are connected to work as a single unit
* Kubernetes automates the distribution and scheduling of application containers acrodd a cluster in a more efficint way
* The Master coordinates the cluster
  * It is responsible for managing the cluster
* Nodes are the workers that run applications
  * Nodes communicate with the master using the kubernetes API called by the kubelet service

### Install Kubernetes masters and nodes
* [Kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
* [Create cluster using kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

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
* [Provisioning a CA and Generating TLS Certs](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/04-certificate-authority.md) - Good info on provisioning a CA in your cluster
* [Manage TLS certs in a cluster](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)
* [Cert Management with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/)

#### Key Points
* Note: Certificates created using the certificates.k8s.io API are signed by a dedicated CA. It is possible to configure your cluster to use the cluster root CA for this purpose, but you should never rely on this. Do not assume that these certificates will validate against the cluster root CA.
* Prereq
  * This tutorial assumes that a signer is setup to serve the certificates API. The Kubernetes controller manager provides a default implementation of a signer. To enable it, pass the `--cluster-signing-cert-file` and `--cluster-signing-key-file` parameters to the controller manager with paths to your Certificate Authority’s keypair.
  * You can configure an external signer such as [cert-manager](https://docs.cert-manager.io/en/latest/tasks/issuers/setup-ca.html), or you can use the built-in signer
* Trusting TLS in a cluster
  * You need to add the CA certificate bundle to the list of CA certificates that the TLS client or server trusts
  * You can distribute the CA certificate as a ConfigMap that your pods have access to use
* Download and isntall CFSSL (Cloudflare's PKI and TLS toolkit)
  * https://pkg.cfssl.org/R1.2/cfssl-bundle_linux-amd64
1. Create a Certificate Signing Request (CSR) and private key
  * `cfssl genkey` will generate a private key and a CSR
  * `cfssljson -bare` will take the output from `cfssl` and split it out into separate key, cert, and CSR files
  * Generate a private key and CSR by running the following command
```
cat <<EOF | cfssl genkey - | cfssljson -bare server
{
  "hosts": [
    "my-svc.my-namespace.svc.cluster.local",
    "my-pod.my-namespace.pod.cluster.local",
    "192.0.2.24",
    "10.0.34.2"
  ],
  "CN": "my-pod.my-namespace.pod.cluster.local",
  "key": {
    "algo": "ecdsa",
    "size": 256
  }
}
EOF
```
  * Where `192.0.2.24` is the service’s cluster IP, `my-svc.my-namespace.svc.cluster.local` is the service’s DNS name, `10.0.34.2` is the pod’s IP and `my-pod.my-namespace.pod.cluster.local` is the pod’s DNS name.
    * FOLLOWUP - How to find these DNS names
  * This command generates two files; it generates `server.csr` containing the PEM encoded pkcs#10 certification request, and `server-key.pem` containing the PEM encoded key to the certificate that is still to be created
2. Create a CSR object to send to the Kube API
  * Generate a CSR yaml blob and send it to the apiserver by running:
```
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: my-svc.my-namespace
spec:
  request: $(cat server.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
```
  * Notice that the `server.csr` file created in step 1 is base64 encoded and stashed in the `.spec.request` field. We are also requesting a certificate with the “digital signature”, “key encipherment”, and “server auth” key usages. We support all key usages and extended key usages listed [here](https://godoc.org/k8s.io/api/certificates/v1beta1#KeyUsage) so you can request client certificates and other certificates using this same API.
  * The CSR should now be visible from the API in a Pending state. You can see it by running: `kubectl describe csr my-svc.my-namespace`
* Get the CSR approved
  * Approving the certificate signing request is either done by an automated approval process or on a one off basis by a cluster administrator
3. Approving CSRs
  * A Kubernetes administrator (with appropriate permissions) can manually approve (or deny) Certificate Signing Requests by using the `kubectl certificate approve <CSR_NAME>` and `kubectl certificate deny <CSR_NAME>` commands. However if you intend to make heavy usage of this API, you might consider writing an automated certificates controller.
4. Download the cert and use it
  * Once the CSR is signed and approved you should see the following
```
kubectl get csr

NAME                  AGE       REQUESTOR               CONDITION
my-svc.my-namespace   10m       yourname@example.com    Approved,Issued
```
  * You can download the issued certificate and save it to a server.crt file by running the following
```
kubectl get csr my-svc.my-namespace -o jsonpath='{.status.certificate}' \
    | base64 --decode > server.crt
```
  * Now you can use `server.crt` and `server-key.pem` as the keypair to start your HTTPS server

### Configure a Highly-Available Kubernetes cluster
* [HA Cluster using kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)

#### Key Points
* Minimum of three masters
* First steps
  1. Create a kube-apiserver load balancer with a name that resolves to DNS
    * Use a TCP forwarding LB to pass traffic to masters
    * HAProxy is a good option
    * Make sure the address of the LB always matches the address of kubeadm's `ControlPlaneEndpoint` flag
  2. Add the first control plane nodes to the load balancer and test the connection
    * `nc -v LOAD_BALANCER_IP PORT`
    * Connection refused error is expected since apiserver isn't running yet
  3. Add remaining control plane nodes to LB target group
* Stacked control plane and etcd nodes
  1. Initialize the control plane
    * `sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs`
```
# Create a certificate key for use in join command
CERT_KEY=$(kubeadm alpha certs certificate-key)
sudo kubeadm init --control-plane-endpoint "$LOAD_BALANCER_DNS:$LOAD_BALANCER_PORT" --upload-certs \
--pod-network-cidr=$POD_CIDR --certificate-key $CERT_KEY
```
    * If the additional master nodes won't be added within 2 hours of initializing the control plane
```
CERT_KEY=$(kubeadm alpha certs certificate-key)
sudo kubeadm init phase upload-certs --upload-certs --certificate-key $CERT_KEY
kubeadm token create --print-join-command --certificate-key $CERT_KEY >> /etc/kubeadm_master_join_cmd.sh
```
    * Install your network plugin of choice

### Know where to get the Kubernetes release binaries
* [Getting Started Release Notes](https://kubernetes.io/docs/setup/release/notes/)
* [Provision the Kube Control Plane](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/08-bootstrapping-kubernetes-controllers.md#provision-the-kubernetes-control-plane)

#### Key Points
* Download the server binaries
```
wget -q --show-progress --https-only --timestamping \
  "https://dl.k8s.io/v1.17.0/kubernetes-server-linux-amd64.tar.gz"
```
* Unpack them
`tar -xf kubernetes-server-linux-amd64.tar.gz`
* Install the Kube binaries
```
{
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}
```

### Provision underlying infrastructure to deploy a Kubernetes cluster
* [Provisioning Compute Resources](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/03-compute-resources.md)

#### Key Points
* Networking
  * Create dedicated Virtual network with at least one subnet
  * Configure firewall rules to allow all traffic between hosts
  * Allocate public IP address for external load balancer
* Compute instances
  * Deploy Ubuntu 18.04 server, three instances for master nodes and three for workers
* Configure SSH access

### Choose a network solution
* [How to implement the Kube networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model)

#### Key Points
* Lots of options here
  * AWS VPC CNI
  * Azure CNI
  * GCE
  * Flannel
  * Calico

### Choose your Kubernetes infrastructure configuration
* [Prod Environment Options](https://kubernetes.io/docs/setup/#production-environment)

### Run end-to-end tests on your cluster
* [Smoke Test](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/13-smoke-test.md)
* [Cluster End-To-End Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-testing/e2e-tests.md)

#### Key Points
* Basic
  * `kubectl cluster-info`
  * `kubectl get nodes`
  * `kubectl get componentstatuses`
  * `kubectl get all --all-namespaces`
  * `kubectl get all --all-namespaces -o wide --show-labels`
* Data Encryption
  * Create a generic secret
    * `kubectl create secret generic <secret_name> --from-literal="mykey=mydata"`
* Deployments
  * `kubectl create deployment nginx --image=nginx`
  * `kubectl get pods -l app=nginx`
  * Port Forwarding
    * Get full name of pod
      * `POD_NAME=$(kubectl get po -l app=nginx -o jsonpath="{.items[0].metadata.name}")`
    * Forward port 8080 on local machine to 80 of nginx pod
      * `kubectl port-forward $POD_NAME 8080:80`
    * Test connectivity
      * `curl --head http://127.0.0.1:8080`
  * Logs
    * `kubectl logs $POD_NAME`
  * Exec
    * `kubectl exec -it $POD_NAME -- nginx -v`
* Services
  * Expose `nginx` deployment using a NodePort
    * `kubectl expose deployment nginx --port=80 --type=NodePort`
  * Retrieve node port assigned to service
    * `NODE_PORT=$(kubectl get svc nginx -o jsonpath="{range .spec.ports[0]}{.nodePart}")`

### Run Node end-to-end tests
* [Node End-to-End tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/e2e-node-tests.md)

### Install and use kubeadm to install, configure, and manage Kubernetes clusters
* [Installing Kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
* [Installing a single control-plane cluster](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
* [Certificate management with Kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/)
* [Upgrading a kubeadm cluster](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

#### Key Points
* Installing kubeadm
```
KUBE_VERSION=1.17.2-00
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet=$KUBE_VERSION kubeadm=$KUBE_VERSION kubectl=$KUBE_VERSION
```
* Use kubeadm to install cluster
```
kubeadm init --apiserver-advertise-address=$IP_ADDR --apiserver-cert-extra-sans=$IP_ADDR,$HOST_NAME \
  --node-name $HOST_NAME --pod-network-cidr=$POD_CIDR --control-plane-endpoint $HOST_NAME:6443
```
* Tear down a cluster
```
kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
kubectl delete node <node name>
kubeadm reset # on node
```
* Upgrade cluster
  * The upgrade workflow at high level is the following:
    1. Upgrade the primary control plane node
    2. Upgrade additional control plane nodes
    3. Upgrade worker nodes
```
kubectl drain <cp-node-name> --ignore-daemonsets  # drain control-plane node
sudo kubeadm upgrade plan   # run upgrade check
sudo kubeadm upgrade apply v1.17.x  # replace x with patch version
# manually upgrade CNI provider
kubectl uncordon <cp-node-name>   # uncordon the control-plane node
```

## Security - 12%
* [Securing a cluster](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/)
* [Tuning Docker with Security Enhancements](https://opensource.com/business/15/3/docker-security-tuning)

### Know how to configure authentication and authorization
* [Controlling access to the Kube API](https://kubernetes.io/docs/reference/access-authn-authz/controlling-access/)
* [Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
* [Authorization - RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
* [Using Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)

#### Key Points
* Accessing the API - three steps
  * Authenticate
  * Authorize - RBAC
  * Admission Control
* Once a request reaches the API server securely, it will first go through the authentication module
  * If authentication fails, it gets rejected, if okay it passes along to authorization module
  * Authorization checks against existing policies to determine if user has permissions to perform action, then passes to admission control
  * Admission control will check actual content of the objects being created and validate them before admitting the request
* Authentication
  * Users in Kube
    * Service accounts managed by Kube
      * These accounts use JSON Web Token (JWT) tokens
      * They are tied to a set of credentials stored as `Secrets`, which are mounted into pods
      * Creating a service account with automatically create the associated secret
      `kubectl create serviceaccount <name>`
    * Normal users managed externally
    * Anonymous
  * Authentication is done with 
    * certificates
    * bearer tokens
      * currently, tokens last indefintely and cannot be changed without restarting the API server
      * token file is a csv file with a min of 3 columns: 
        * token
        * username
        * user uid
        * optional group names  # "group1,group2,group3"
    * an authentication proxy
    * HTTP basic auth with username and password
      * basic auth file is a csv file with a minimum of 3 columns (same as token file)
  * When multiple methods are enabled, the first module to successfully authenticate a request is used
  * Type of authentication used is defined on kube-apiserver startup with the following flags
    * --client-ca-file=SOMEFILE   # x509 Client Certs - created using openssl
    * --token-auth-file=SOMEFILE   # Static Token File for bearer tokens
    * --basic-auth-file=SOMEFILE    # Static Password file
    * --oidc-issuer-urls=SOMEURL   # OpenID Connect Tokens via OAuth2 like AzureAD, Salesforce, and Google
    * --authorization-webhook-config-file=SOMEFILE
  * `401` HTTP error response if request cannot be authenticated
* Authorization
  * An authorization request must include
    * The username of the requestor
    * The requested action
    * The object affected by the action
  * `403` HTTP error if the request is denied
  * Three main methods to authorize + two global options
    * Role-based access control (RBAC)
      * RBAC is a method of regulating access based on the roles of a user
    * Attribute-based access control (ABAC)
      * Policies are defined in JSON
    * Webhook
    * AlwaysAllow
    * AlwaysDeny
  * RBAC
    * To enable RBAC, start the apiserver with `--authorization-mode=RBAC`
      * kubeadm uses this authorization method by default
    * All resources are objects and belong to API groups
    * Resources operations are based on CRUD (Create, Read, Update, Delete)
    * Rules are operations which can act on API groups
    * Permissions are purely additive, there are no "deny" rules
    * Roles
      * Are a group of Rules which are scoped to a namespace
    * `ClusterRole` applies to the whole cluster
    * `RoleBinding` may also reference a `ClusterRole` and will give namespace permissions to that role
    * Cannot modify `Role` or `ClusterRole` a binding onject refers to
      * Must delete existing binding first then recreate
  * `kubectl auth reconcile` creates or updates a manifest file containing RBAC objects and handles deleting and recreating binding objects if required to change the role the refer to
  * Aggregated ClusterRoles
    * `ClusterRole` can be created by combining other ClusterRoles using an `aggregationRule`
    * The permissions of aggregated ClusterRoles are controller-managed and filled in by joining rules of any ClusterRole that matches the provided label selector
* Setting up authentication and authorization
  * Determine and create namespace
  * Create certificate credential for user
```
# Authentication
## Create user with password on node
sudo useradd -s /bin/bash devJohn
sudo passwd devJohn

## generate private key and CSR
openssl genrsa -out devJohn.key 2048
openssl req -new -key devJohn.key -out devJohn.csr \
  -subj "/CN=devJohn/O=development/O=app2"
  # Common name of the subject is used as the user name for the request
  # Organization fields indicate a user's group memberships; can include multple group memberships

## generate self-signed cert using X509 and cluster CA keys
sudo openssl x509 -req -in devJohn.csr \
  -CA /etc/kubernetes/pki/ca.crt \    # cluster certs location on nodes
  -CAKey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out devJohn.crt -days 90

## Update config file with new key and cert
kubectl config set-credentials devJohn \
  --client-certificate=/pathToCert/devJohn.crt \
  --client-key=/pathToKey/devJohn.key

## Create context to allow access for user to a specific cluster and namespace
kubectl config set-context devJohn-context \
  --cluster=kubernetes \
  --namespace=dev \
  --user=devJohn

# Authorization
## Associate RBAC rights to NS and role
kubectl create role developer --verb=get,watch,list,create,update,patch,delete \
  --resource=deployments,replicasets,pods 
kubectl create clusterrole pod-reader --verb=get,list,watch --resource=pods

### OR
cat <<EOF > role-dev.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev
rules:
- apiGroups: ["", "extensions", "apps"] # "" indicates the core API group
  resources: ["deployments", "replicasets", "pods"]
  verbs: ["get", "watch", "list", "create", "update", "patch", "delete"] 
# use [*] for all verbs/resources/apiGroups
EOF
kubectl apply -f role-dev.yaml

## Create RoleBinding to associate role w/ user
kubectl create rolebinding dev-role-binding --role=developer --user=devJohn --namespace=dev
kubectl create clusterrolebinding root-cluster-admin-binding --clusterrole=cluster-admin --user=root

### OR
cat <<EOF > rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-role-binding
  namespace: dev
subjects:
- kind: User
  name: devJohn # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role # this must be Role or ClusterRole
  name: developer # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f rolebinding.yaml
```
* Admission Controller
  * An admission controller is a piece of code that intercepts requests to the Kubernetes API server before persistence of the object, but after the request is authenticated and authorized
  * Admission controllers can be "Validating", "mutating", or both
    * Mutating controllers may modify objects they admit
    * Validating controllers may not modify objects
  * Two phases
    * First phase, mutating admission controllers are run
    * Second phase, validating ACs are run
  * A request can be rejected in either phase
  * Enable them by adding `enable-admission-plugins` flag that takes a comma-delimited list of AC plugins to invoke prior to modifying objects in the cluster
    * `kube-apiserver --enable-admission-plugins=NamespaceLifecycle,LimitRanger ...`
  * Disable ACs
    * `kube-apiserver --disable-admission-plugins=PodNodeSelector,AlwaysDeny ...`
  * Validate which plugins are enabled
    * From Control-Plane node `kube-apiserver -h | grep enable-admission-plugins`

### Understand Kubernetes Security primitives
* [Pod Security Policies](https://kubernetes.io/docs/concepts/policy/pod-security-policy/)
* [PSP-RBAC Examples on Kubernetes Github](https://github.com/kubernetes/examples/blob/master/staging/podsecuritypolicy/rbac/README.md)

#### Key Points
* PSPs enable fine-grained authorization of pod creation and updates
* It is a cluster-level resource that controls security sensitive aspects of the pod spec that govern what a pod can do, can access, the user they can run as, etc
* Allow admins to control the following, plus many others
  * Running as privileged containers - `privileged`
  * Usage of host namespaces - `hostPID`, `hostIPC`
  * User and group IDs of the container - `runAsUser`, `runAsGroup`, `supplementalGroups`
  * Allocating an FSGroup that owns the pod's volumes - `fsGroup`
  * Linux capabilities - `defaultAddCapabilities`, `requiredDropCapabilities`, `allowedCapabilities`
  * SELinux context of the container - `seLinux`
* With RBAC and PSP, you can finely tunen what users are allowed to run and what capabilities and low level privileges their containers will have
* PSP control is implemented as an optional (but recommended) admission controller
* Enabled in the admission controller without authorizing any policies will prevent any pods from being created in the cluster
* In order to use a PSP resource, the requesting user or target pod's service account must be authorized to use the policy by allowing the `use` verb on the policy
  * preferred method is to grant access to the pod's service account
```
# First, a Role or ClusterRole needs to grant access to use the desired policies. The rules to grant access look like this:
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: <role name>
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
  resourceNames:
  - <list of policies to authorize>

# Then the (Cluster)Role is bound to the authorized user(s):
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: <binding name>
roleRef:
  kind: ClusterRole
  name: <role name>
  apiGroup: rbac.authorization.k8s.io
subjects:
# Authorize specific service accounts:
- kind: ServiceAccount
  name: <authorized service account name>
  namespace: <authorized pod namespace>
# Authorize specific users (not recommended):
- kind: User
  apiGroup: rbac.authorization.k8s.io
  name: <authorized user name>
```

### Know how to configure network policies
* [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
* [Declare network policies](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)

#### Key Points
* A network policy is a spec of how groups of pods are allowed to communicate with each other and other network endpoints
* `NetworkPolicy` resources use labels to select pods and define rules which specify what traffic is allowed
* Some network plugins require annotating namespaces
* The use of an empty ingress or egress rule denies all type of traffic for included pods
* Use granular policy files with specific case per file
```
# Create Nginx deployment and expose it via a service
kubectl create deploy nginx --image=nginx
kubectl expose deploy nginx --port=80

# Test service by accessing it from another pod
kubectl run busybox --image=busybox --rm -ti --restart=Never -- /bin/sh
wget --spider --timemout=1 nginx

# Limit access to nginx service
cat <<EOF > nginx-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-nginx
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - ipBlock:
        cidr: 172.16.0.0/16
        except:
        - 172.16.1.0/24
    - podSelector:
        matchLabels:
          access: "true"  # only pods with this label can access nginx
EOF
kubectl apply -f nginx-policy.yaml

# Test access after policy is applied
kubectl run busybox --image=busybox --rm -ti --restart=Never -- /bin/sh
wget --spider --timemout=1 nginx

# Define access label and test again
kubectl run busybox --image=busybox --rm -ti --restart=Never --labels="access=true" -- /bin/sh
wget --spider --timemout=1 nginx
```

### Create and manage TLS certificates for cluster components
* See the "Configure secure cluster communication" section in Installation, Configuration, and Validation
* [Manage TLS Certificates in a cluster](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)
* Very likely to show up on test since this is an item under several sections

#### Key Points
* Download and isntall CFSSL (Cloudflare's PKI and TLS toolkit)
  * https://pkg.cfssl.org/R1.2/cfssl-bundle_linux-amd64
1. Create a Certificate Signing Request (CSR) and private key
  * `cfssl genkey` will generate a private key and a CSR
  * `cfssljson -bare` will take the output from `cfssl` and split it out into separate key, cert, and CSR files
  * Generate a private key and CSR by running the following command
```
cat <<EOF | cfssl genkey - | cfssljson -bare server
{
  "hosts": [
    "my-svc.my-namespace.svc.cluster.local",
    "my-pod.my-namespace.pod.cluster.local",
    "192.0.2.24",
    "10.0.34.2"
  ],
  "CN": "my-pod.my-namespace.pod.cluster.local",
  "key": {
    "algo": "ecdsa",
    "size": 256
  }
}
EOF
```
  * Where `192.0.2.24` is the service’s cluster IP, `my-svc.my-namespace.svc.cluster.local` is the service’s DNS name, `10.0.34.2` is the pod’s IP and `my-pod.my-namespace.pod.cluster.local` is the pod’s DNS name.
    * FOLLOWUP - How to find these DNS names
  * This command generates two files; it generates `server.csr` containing the PEM encoded pkcs#10 certification request, and `server-key.pem` containing the PEM encoded key to the certificate that is still to be created
2. Create a CSR object to send to the Kube API
  * Generate a CSR yaml blob and send it to the apiserver by running:
```
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: my-svc.my-namespace
spec:
  request: $(cat server.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
```
  * Notice that the `server.csr` file created in step 1 is base64 encoded and stashed in the `.spec.request` field. We are also requesting a certificate with the “digital signature”, “key encipherment”, and “server auth” key usages. We support all key usages and extended key usages listed [here](https://godoc.org/k8s.io/api/certificates/v1beta1#KeyUsage) so you can request client certificates and other certificates using this same API.
  * The CSR should now be visible from the API in a Pending state. You can see it by running: `kubectl describe csr my-svc.my-namespace`
* Get the CSR approved
  * Approving the certificate signing request is either done by an automated approval process or on a one off basis by a cluster administrator
3. Approving CSRs
  * A Kubernetes administrator (with appropriate permissions) can manually approve (or deny) Certificate Signing Requests by using the `kubectl certificate approve <CSR_NAME>` and `kubectl certificate deny <CSR_NAME>` commands. However if you intend to make heavy usage of this API, you might consider writing an automated certificates controller.
4. Download the cert and use it
  * Once the CSR is signed and approved you should see the following
```
kubectl get csr

NAME                  AGE       REQUESTOR               CONDITION
my-svc.my-namespace   10m       yourname@example.com    Approved,Issued
```
  * You can download the issued certificate and save it to a server.crt file by running the following
```
kubectl get csr my-svc.my-namespace -o jsonpath='{.status.certificate}' \
    | base64 --decode > server.crt
```
  * Now you can use `server.crt` and `server-key.pem` as the keypair to start your HTTPS server

### Work with images securely
* [Security Best Practices for Kubernetes Deployments](https://kubernetes.io/blog/2016/08/security-best-practices-kubernetes-deployment/)

#### Key Points
* Having running containers with vulnerabilies opens your environment to the risk of attack
* Many of these attacks can be mitigated by making sure that there are no software components that have known vulnerabilities
* Good practices for images
  * Implement continuous security vulnerability scanning
    * Containers might include outdated packages with known CVEs. This must be a continuous process as new vulnerabilies are published every day
  * Regularly apply security updates to your environment
    * Always update the source image and redeploy containers after vulnerabilities are found
    * Rolling updates are easy with Kube, take advantage of it
* Ensure that only authorized images are used in your environment
  * Downloading and running images from unknown sources is dangerous
  * Use **private registries** to store your approved images
  * Build CI pipeline that integrates security assessment as part of build process
  * Once image is build, scan it for security vulnerabilities
* Limit Direct access to Kube Nodes
* Create Admin boundaries between resources
  * Limit user permissions to reduce the impact of mistakes or malicious activities
  * A Kube namespace allows you to partition created resources logically and assign permissions by NS
* Define Resource Quota and Limit Ranges
  * This avoids noisy neighbor scenarios
* Implement Network Segmentation
  * Reduces risk of a compromised application accessing what it isn't allowed to access
* Apply Security Context to pods and containers
* Log everything

### Define security contexts
* [Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
* [Security Context Design Doc](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/auth/security_context.md)

#### Key Points
* A Security context defines privilege and access control settings
* Can be defined for Pod or per container
  * Linux capabilities are set at the container level only
```
# Pod security example: security-context.yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  volumes:
  - name: sec-ctx-vol
    emptyDir: {}
  containers:
  - name: sec-ctx-demo
    image: busybox
    command: [ "sh", "-c", "sleep 1h" ]
    volumeMounts:
    - name: sec-ctx-vol
      mountPath: /data/demo
    securityContext:
      allowPrivilegeEscalation: false
```
* In above config file
  * `runAsUser` field specifies that any container in the pod, all processes run with user ID 1000
  * `runAsGroup` field specifies the primary group ID of 3000 for all processess within any container
    * if this is omitted, default will be root(0)
  * Any files created will be owned by user 1000 and group 3000
  * Since `fsGroup` is specified, all processes of the container are also part of the supplementary group 2000
    * The owner for volume `/data/demo` and any files create in that volume will be group ID 2000
  * To verify
    * Run pod `kubectl apply -f security-context.yaml`
    * Run shell in container `kubectl exec -it security-context-demo -- sh`
      * `ps` to see what user processes are running as
      * `ls -l` on /data dir to see group 2000
      * `id` to see user, group, and supplemental group IDs
```
# Container security example: security-context-2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-2
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - name: sec-ctx-demo-2
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
      runAsUser: 2000
      allowPrivilegeEscalation: false
```
* `securityContext` settings made at the Container level override settings made at the Pod level
* Assign SELinux labels to a container
  * `seLinuxOptions` field in `securityContext` section of Pod or Container
```
...
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"
```

### Secure persistent key value store
* [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
* [Configure a Pod to Use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)

#### Key Points
* A Secret is an object that lets you store and manage a small amount of sensitive data, such as passwords, OAuth tokens, and ssh keys
* Secrets are namespace specific. They are only encoded with base64 by default
* To encrypt
  * Must create an `EncryptionConfiguration` with a key and proper identity
  * The kube-apiserver needs the `--encryption-provider-config` flag set with `aescbc` or `ksm`
  * Must recreate secrets after doing the above steps
  * More info on [encrypting data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
* To use a Secret, a Pod needs to reference the secret. It can be used with a Pod in two ways:
  * As files in a volume mounted on one or more of its containers
  * By the kubelet when pulling images for the pod
  * As an environment variable
* Built-in Secrets
  * Service accounts automatically create and attach Secrets with API credentials
    * This can be disabled or overrideen if desired
  * See [Service Account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) docuemntation for more info
* Creating your own Secrets
  * Using Kubectl
    * Create from files
      * Create DB Username and Password files with credentials
      ```
      echo -n 'admin' > ./username.txt # -n = no new line
      echo -n 'password' > ./password.txt
      ```
      * Use `kubectl create secret` to create the secret
      `kubectl create secret generic db-user-pass --from-file=./username.txt --from-file=./password.txt`
        * Be sure to escape special characters such as `$`, `\`, `*`, and `!`, or use single quotes
  * Creating a secret manually
    * Concert the string to base64
    `echo -n 'admin' | base64`
    `echo -n 'password | base64`
    * Take these strings and create a secret.yaml file
    ```
    apiVersion: v1
    kind: Secret
    metadata:
      name: mysecret
    type: Opaque
    data:
      username: <base64 string of username>
      password: <base64 string of passsword>
    ```
    * Apply Secret using `kubectl apply -f ./secret.yaml`
    * To let the `kubectl apply` command convert the secret data into base64 for you
    ```
    apiVersion: v1
    kind: Secret
    metadata: 
      name: mysecret
    type: Opaque
    stringData:
      config.yaml: |-
        username: {{username}}  # Your deployment tool could replace these values
        password: {{password}}
    ```
    * If both `data` and `stringData` fields are specified, `stringData` will be used
  * Decoding a secret
    * `kubectl get secret mysecret -o yaml`
    * `echo '<base64 encoded value>' | base64 --decode`
* Using Secrets
  * Secrets can be mounted as data volumes or exposed as environment variables to be used by a container or pod
  * Using Secrets as files from a pod
    1. Create a secret or use an existing one. Multiple pods can reference the same secret
    2. Modify your pod definition to add a volume under `.spec.volumes[]`. Name the volume anything and have a `.spec.volumes[].secret.secretName` field equal to the name of the Secret object
    3. Add a `.spec.containers[].volumeMounts[]` to each container that needs the secret. Specify `.spec.containers[].volumeMounts[].readOnly = true` and `.spec.containers[].volumeMounts[].mountPath` to an unused directory name where you would like the secrets to appear
    4. Modify your image or command line to that the program looks for files in that directory. Each key in the secret `data` map becomes a filename under `mountPath`
    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: mypod
    spec:
      containers:
      - name: mypod
        image: redis
        volumeMounts:
        - name: secretmount
          mountPath: "/etc/secretmount"
          readOnly: true
      volumes:
      - name: secretmount
        secret: 
          secretName: mysecret
    ```
    * Each Secret you want to refer to needs to be in `.spec.volumes`
    * Secret files permissions
      * By default it uses `0644`
      * Configure by adding `.spec.volumes[].secret.defaultMode` to whatever value is required
      * The value must be specified in demical notation
  * Mounted secrets are updated automatically
    * The kubelet checks whether the mounted secret is fresh on periodic syncs
* Using secrets as environment variables
  1. Create a secret
  2. Modify pod definition with `env[].valueFrom.secretKeyRef`
  3. Modify image or command line for program to look for values in env vars
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: mypod
  spec:
    containers:
    - name: mycontainer
      image: redis
      env: 
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
  ```
* Use `imagePullSecrets` to pass secrets to kubelet
* Secrets must be created before Pod is created because the secret is validated first
  * References to secrets that do not exist will prevent the Pod from starting
* [Use cases](https://kubernetes.io/docs/concepts/configuration/secret/#use-cases)
* Best practices
  * Limit access using authorization polices such as RBAC
  * The ability to `watch` and `list` should be limited because they allow access to all secrets in a namepsace
  * Applications that need to access the Secret API should perform `get` requests
  * In a future release, there might be an ability to "bulk watch" to let clients `watch` individual resources
* Kubelet stores secrets into a `tmpfs` so that the secret is not written to disk
* Each container that wants to use a secret must mount it for it to be visible to the container
* [Secret Risks](https://kubernetes.io/docs/concepts/configuration/secret/#risks)
* ConfigMaps
  * ConfigMaps allow you to decouple configuration artifacts from image content to keep containerized applications portable
  * Stores data in key-value pairs or plain config files in any format
  * Create a ConfigMap
    * `kubectl create configmap <map-name> <data-source>`
    * ConfigMap data can be pulled from directories, files, or literal values
    * ConfigMap from directory
      * `kubectl create configmap test-config --from-file=configure-pod-container/configmap/`
    * ConfigMap from files
      * `kubectl create configmap test-config --from-file=configure-pod-container/configmap/game.properties`
  * Define container environment variables using ConfigMap data
    1. Define an environemtn variable as a key-value pair in a ConfigMap
    `kubectl create configmap special-config --from-literal=special.how=very`
    2. Assign the `special.how` value to an env var in the Pod spec
    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: mypod
    spec:
      containers:
      - name: test-container
        image: busybox
        command: [ "/bin/sh", "-c", "env" ]
        env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
    ```
    3. Create the Pod
    `kubectl apply -f pod.yaml`
  * To mount all values in a ConfigMap
  ```
  containers:
  - name: test-container
    image: busybox
    envFrom:
    - configMapRef:
        name: special-config
  ```
* Mounted ConfigMaps are updated automatically
* ConfigMaps should reference properties files, not replace them
* Like secrets, you must create a ConfigMap before referencing it to a pod
* ConfigMaps reside in a namespace

## Cluster Maintenance - 11%
### Understand Kubernetes cluster upgrade process
* [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
* [kubeadm upgrade](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-upgrade/)

#### Key Points
* Before you begin:
  * Upgrade Kubeadm to the latest stable version
  * Swap must be disabled
  * The cluster should use a static control plane and etcd pods or external etcd
  * Read the release notes of all versions between the one you're on and what you're upgrading to
  * Backup any important componenets, such as app-level state stored in a database
* Additional info
  * All containers are restarted after a node gets upgraded
  * You can only upgrade from one MINOR version to the next MINOR version, or between patches versions of the same MINOR version
    * 1.16 -> 1.16.2 || 1.16 -> 1.17
* Determine which version to upgrade to
  * Find the latest stable release of kubeadm (this version corresponds to the kube version)
  ```
  apt update
  apt-cache madison kubeadm
  ```
* Upgrading control plane nodes first
  * On your first control plane node, upgrade kubeadm
  ```
  # Old method
  apt-mark unhold kubeadm && \ # allow kubeadm to be updated
  apt-get update && apt-get install -y kubeadm=1.17.x-00 && \
  apt-mark hold kubeadm   # prevent kubeadm from being updated

  # since apt-get version 1.1, you can also use the following command
  apt-get update && \
  apt-get install -y --allow-change-held-packages kubeadm=1.17.x-00
  ```
  1. Verify kubeadm version - `kubeadm version`
  2. Drain the control plane node - `kubectl drain <cp-node-name> --ignore-daemonsets`
  3. Run upgrade checks to make sure cluster is in a ready state - `sudo kubeadm upgrade plan`
  4. Choose a version to upgrade to, and run the appropriate command
  `sudo kubeadm upgrade apply v1.17.x`
    * Note: `kubeadm upgrade` also automatically renews the certificates that it manages on this node. To opt-out of cert renewal, use the `--certificate-renewal=false` flag
  5. Upgrade you CNI provider (networking provider) such as Flannel or Calico
  6. Uncordon the control plane node - `kubectl uncordon <cp-node-name>`
  7. Repeat steps 1-6 to upgrade additional control plane nodes but instead of `sudo kubeadm upgrade apply` use `sudo kubeadm upgrade node`
    * `sudo kubeadm upgrade plan` is not needed
* Upgrade kubelet and kubectl
  1. Upgrade kubelet and kubectl on all control plane nodes
  ```
  apt-mark unhold kubelet kubectl && \
  apt-get update && apt-get install -y kubelet=1.17.x-00 kubectl=1.17.x-00 && \
  apt-get hold kubelet kubectl

  # v1.1 of apt-get use the following
  apt-get update && \
  apt-get install -y --allow-change-held-packages kubelet=1.17.x-00 kubectl=1.17.x-00
  ```
  2. Restart the kubelet - `sudo systemctl restart kubelet`
* Upgrade worker nodes
  * Upgrade kubeadm using same commands from control plane node
  * Drain the node
  `kubectl cordon <worker-node-name> --ignore-daemonsets`
  * Upgrade the node - `sudo kubeadm upgrade node`
  * Upgrade kubelet and kubectl using same commands from control plane node
  * Restart the kubelet
* Verify status of node - `kubectl get nodes`
* Worst-case - Recovering from a failure state
  * If `kubeadm upgrade` fails and does not roll back, you can run `kubeadm  upgrade` again
  * To recover from a bad state, you can also run `kubeadm upgrade apply --force` without changing the version that your cluster is running
  * Upgrading with kubeadm write the following backup folders under `/etc/kubernetes/tmp`: - `kubeadm-backup-etcd-<date>-<time>` - `kubeadm-backup-manifests-<date>-<time>`
    * `kubeadm-backup-etcd` contains a backup for the local etcd member data
      * It can be manually restored in `/var/lib/etcd`
    * `kubeadm-backup-manifests` contains a backup of the static Pod manifest files
      * It can be manually restored in `/etc/kubernetes/manifests`
* How it works - This is important to know
  * `kubeadm upgrade apply`
    * Checks that your cluster is in an upgradable state
      * The API server is reachable
      * All nodes are in the `Ready` state
      * The control plane is healthy
    * Enforces the version skew policies
    * Makes sure the control plane images are available to pull to the machine
    * Upgrades the control plane components or rollsback if any of them fails to come up
    * Applies the new `kube-dns` and `kube-proxy` manifests and makes sure that all necessary RBAC rules are created
    * Creates new cert and key files on the API server and backs up old files if they're about to expire in 180 days
  * `kubeadm upgrade node` on additional control plane nodes
    * Fetches the kubeadm `ClusterConfiguration` from the cluster
    * Optionally backs up the kube-apiserver cert
    * Upgrades the static Pod manifests for the control plane components
    * Upgrades the kubelet configuration for this node
  * `kubeadm upgrade node` on worker nodes
    * Fetches the kubeadm `ClusterConfiguration` from the cluster
    * Upgrades the kubelet configuration for this node

### Facilitate OS upgrades
#### Key Points
* Similar to upgrading the kubernetes components, to upgrade the underlying host, do the following
  1. Drain the node - `kubectl cordon <node-name> --ignore-daemonsets`
  2. Upgrade the OS
  3. Uncordon the node - `kubectl uncordon <node-name>`

### Implement backup and restore methodologies
* [Backing up an etcd cluster](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)

#### Key Points
* Two main pieces to backup
  * Certificate files
  * Etcd database
* Backing up certificate files
  * Backup the `/etc/kubernetes/pki` directory
    * This can be done with a cronjob
* Backup the etcd database
  * All kubernetes objects are stored in etcd
  * This snapshot file contains all kubernetes states and critical information
  * Encrypt the snapshot files whenever possible
  * Built-in snapshot
    * Run `etcd snapshot save` on a control plane node
    * Example of taking a snapshot of the keyspace served by `$ENDPOINT` to the file `snapshotdb`
```
ETCDCTL_API=3 etcdctl endpoints $ENDPOINT snapshot save snapshotdb
# exit 0

# verify the snapshot
ETCDCTL_API=3 etcdctl --write-out=table snapshot status snapshotdb
```
  * Can also do a volume snapshot of the node

## Troubleshooting - 10%
### Troubleshoot application failure
* [Troubleshoot applications](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application/)
* [Determine the Reason for Pod Failure](https://kubernetes.io/docs/tasks/debug-application-cluster/determine-reason-pod-failure/)

#### Key Points
* Debugging Pods
  * First step is taking a look at the pod
  `kubectl describe po ${POD_NAME}`
    * Look at the state of the containers. Are they all `Running`? Any recent restarts?
  * **My pod stays pending**
    * If a pod is stuck in `Pending` it means that it can not be scheduled onto a node. This can happen due to the following:
    * **You don't have enough resources** - You may have exhausted CPU or memory in your cluster. If so, delete pods, adjust resource requests, or add new nodes to your cluster
    * **You are using `hostPort`** - when you bind a Pod to `hostPort` there are a limited number of places that a pod can be scheduled. In most cases, `hostPort` is unnecessary, try using a Service object to expose your Pod instead. If you require `hostPort`, you can only schedule as many Pods as there are nodes in your cluster
  * **My pod stays waiting**
    * If a pod is stuck in `Waiting` state, then it has been scheduled to a worker node, but it can't run on that machine. `kubectl describe ...` should be informative as to why
    * A common cause is a failure to pull the image. Check these three things:
      * Make sure that you have the name of the image correct
      * Have you pushed the image to the repo?
      * Run a manual `docker pull <image>` on your machine to see if the image can be pulled
  * **My pod is crashing or otherwise unhealthy**
    * First, take a look at the logs of the current container
    `kubectl logs ${POD_NAME} ${CONTAINER_NAME}`
    * If the container previously crashed, check the previous container's crash log
    `kubectl logs --previous ${POD_NAME} ${CONTAINER_NAME}`
  * **My pod is running but not doing what I told it to do**
    * If your pod is not behaving as expected, check if there was an error in your pod description (`mypod.yaml`), and that the error was silently ignored when you created the pod
    * Validate the config when it is applied
    `kubectl apply --validate -f pod.yaml`
    * Next, check if whether the pod on the apiserver matches the pod you meant to create
    `kubectl get po mypod -o yaml > mypod-on-apiserver.yaml`
      * Manually compare with your original pod description
* Debugging Replication Controllers
  * These are pretty straightforward. They can either create pods or they can't
  * Use `kubectl describe rc ${CONTROLLER_NAME}` to introspect events related to the replication controller
* Debugging Services
  * Services provide load balancing across a set of pods. There are several common problems that can make Services not work properly
  * First, verify that there are endpoints for the Service
  `kubectl get endpoints ${SERVICE_NAME}`
    * Make sure the endpoints match up with the number of containers you expect to be a member of your service
  * **My service is missing endpoints**
    * Try listing pods using the labels that this Service uses
    ```
    # For example
    ...
    spec:
      - selector:
        name: nginx
        type: frontend
    ```
    You can use `kubectl get pods --selector=name=nginx,type=frontend` to list pods that match this selector. Verify that the list matches the Pods that you expect
    * If the pods match your expectations, but your endpoints are still empty, it's possible that you don't have the right ports exposed. If your service has a `containerPort` specified, but the Pods that are selected don't have that port listed, then they won't be added to the endpoints list
    * Verify that the pod's `containerPort` matches up with the Service's `targetPort`
  * **Network traffic is not forwarded**
    * If you can connect to the service, but the connection is immediately dropped, and there are endpoints in the endpoints list, it's likely that the proxy can't contact your pods
    * Three things to check
      * Are your pods working correctly? Look for restart count and follow the debug pods info
      * Can you connect to your pods directly? Get the IP address for the Pod, and try to connect directly to that IP
      * Is your application serving on the port that you configured? K8s doesn't do port remapping, so if your app serves on 8080, the `containerPort` field needs to be 8080

### Troubleshoot control plane failure
* [Troubleshoot clusters](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/)

#### Key Points
* First, rule out the application as the root cause of the problem
* Listing your cluster
  * Check if your nodes are all registered correctly
  `kubectl get nodes`
  * Verify that all nodes you expect to see are present and all in the `Ready` state
  * To get detailed information about the overall health of your cluster, you can run:
  `kubectl cluster-info dump > ./cluster-health.txt`
  * For less detailed information about the overall health of your cluster
  `kubectl get componentstatuses`
* **Look at logs**
  * For now, digging deeper into the cluster requires logging into the relevant machines
  * For systemd-based systems, use `journalctl`
    * **Master**
    `journalctl -u kube-apiserver|kube-scheduler|kube-contaoller-manager | less`
    * **Worker Nodes**
    `journalctl -u kubelet | less`
  * For non-systemd-based systems, here are the relevant log locations
    * **Master**
    `/var/log/kube-apiserver.log`
    `/var/log/kube-scheduler.log`
    `/var/log/kube-contaoller-manager.log`
    * **Worker Nodes**
    `/var/log/kubelet.log`
    `/var/log/kube-proxy.log`
* **General overview of cluster failure modes**
  * Root causes
    * VM(s) shutdown
      * Apiserver VM shutudown or apiserver crashing
      * Supporting services (node controller, rc manager, scheduler, etc) VM shutdown or crashing
      * Individual node shuts down
    * Network partition within cluster, or between cluster and users
      * Partition A thinks the nodes in partition B are down
      * Partition B thinks the apiserver is down (assuming it is in A)
    * Crashes in K8s software
    * Data loss or unavailability of persistent storage
      * Apiserver backing storage lost
    * Operator error, such as misconfigured K8s software or app software

### Troubleshoot worker node failure
#### Key Points
* See control plane failure for worker specific information

### Troubleshoot networking
* [Debug Services](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/)

#### Key Points
* For many steps here you will want to see what a Pod running in the cluster sees. The simplest
way to do this is to run an interactive Pod
`kubectl run -it --rm --restart=Never busybox --image=busybox sh`
* If you already have a running Pod that you prefer to use
`kubectl exec <POD_NAME> -c <CONTAINER_NAME> -- <COMMAND>`
* **Does the Service exist?**
  * If the service doesn't exist, you would get something like:
  ```
  wget -O- hostnames
  Resolving hostnames (hostnames)... failed: Name or service not known.
  wget: unable to resolve host address 'hostnames'
  ```
  * First check if that Service actually exists and create if needed
  `kubectl get svc hostnames`
  `kubectl expose deployment hostnames --port=80 --target-port=9376`
  * Verify that it exists now
  `kubectl get svc hostnames`
* **Does the Service work by DNS name?**
  * One of the most common ways that clients consume a Service is through a DNS name
  * From a Pod in the same namespace
  ```
  nslookup hostnames

  Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local
  Name:      hostnames.default
  Address 1: 10.0.1.175 hostnames.default.svc.cluster.local
  ```
  * If this fails, your Pod and Service might be in a different namespace. Try a namespace-qualified name
  ```
  nslookup hostnames.default # for the default namespace

  Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local
  Name:      hostnames.default
  Address 1: 10.0.1.175 hostnames.default.svc.cluster.local
  ```
  * If this works, you'll need to adjust your app to use a cross-namespace name. If this still fails, try a fully-qualified name
  ```
  nslookup hostnames.default.svc.cluster.local # for the default namespace

  Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local
  Name:      hostnames.default
  Address 1: 10.0.1.175 hostnames.default.svc.cluster.local
  ```
  * Try it from a Node in the cluster
  ```
  nslookup hostnames.default.svc.cluster.local 10.0.0.10 # Note: 10.0.0.10 is the cluster's DNS Service IP, yours might be different

  Server:         10.0.0.10
  Address:        10.0.0.10
  Name:   hostnames.default.svc.cluster.local
  Address: 10.0.1.175
  ```
  * If a fully qualified name lookup works but not a relative one, check `/etc/resolv.conf` file in your Pod. From within a Pod
  ```
  cat /etc/resolv.conf

  # should look something like
  nameserver 10.0.0.10
  search default.svc.cluster.local svc.cluster.local cluster.local example.com
  options ndots:5
  ```
  * Does any Service work by DNS name?
    * If the above still fails, DNS lookups are not working for your Service
    * From within a Pod
    ```
    nslookup kubernetes.default

    # should see something like
    Server:    10.0.0.10
    Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local
    Name:      kubernetes.default
    Address 1: 10.0.0.1 kubernetes.default.svc.cluster.local
    ```
    * If this fails, see the `kube-proxy` section, or even go back to the top of this document and start over, but instead of debugging your own Service, debug the DNS Service.
* **Does the Service work by IP?**
  * Assuming DNS checks out okay, the next thing to test is if your Service works by its IP address
  * From a Pod in your cluster, access the Service's IP
  ```
  wget -qO- 10.0.1.175:80 # Service IP and port

  # should see something like
  hostnames-0uton
  ```
  * If your Service is working, you should get correct responses. If not, read on.
* **Is the Service defined correctly?**
  * Double and triple check that your Service is correct and matches your Pod's port
  `kubectl get svc hsotnames -o json`
  * Is the Service port you are trying to access listed in `spec.ports[]`?
  * Is the `targetPort` correct for your Pods (some Pods use a different port than the Svc)
  * If you meant to use a numeric port, it is a number (9376) or a string "9376"?
  * If you meant to use a named port, do your Pods expose a port with the same name?
  * Is the port's `protocol` correct for your pods?
* **Does the Service have any Endpoints?**
  * If you got this far, you have confirmed that your Service is correctly defined and is resolved by DNS
  * Now check that the Pods you ran are actually being selected by the Service
  `kubectl get pods -l run=hostnames`
  * If the restart count is high, go through debugging pods section
  * Confirm that the endpoints controller has found the correct Pods or your Service
  `kubectl get endpoints hostnames`
  * If `ENDPOINTS` is `<none>`, check the `spec.selector` field of your Service and the `metadata.labels` values on your Pods
    * A common mistake is to have a typo or other error
* **Are the Pods working?**
  * Now we now the Service exists and has selected your pods
  * Let's bypass the Service and go directly to the Pods, as listed by the Endpoints above
  * From within a Pod
  ```
  for ep in 10.244.0.5:9376 10.244.0.6:9376 10.244.0.7:9376; do
      wget -qO- $ep
  done

  # should see something like
  hostnames-0uton
  hostnames-bvc05
  hostnames-yp2kp
  ```
  * Each pod should return its own hostname
* **Is the kube-proxy working?**
  * So now, your Service is running, has Endpoints, and your Pods are actually serving
  * Now the whole Service prixy mechanism is suspect
  * The default implementation of Services is kube-proxy
  * Is kube-proxy running?
    * Confirm that `kube-proxy` is running on your Nodes. Run this directly on a node
    ```
    ps auxw | grep kube-proxy

    # should see something like
    root  4194  0.4  0.1 101864 17696 ?    Sl Jul04  25:43 /usr/local/bin/kube-proxy --master=https://kubernetes-master --kubeconfig=/var/lib/kube-proxy/kubeconfig --v=2
    ```
    * Confirm that it is not failing something obvious, like contacting the master
    `journalctl -u kube-proxy | less`
      * If you see an error about contacting the master, check your Node configuration and install steps
    * Another reason might be that the required `conntrack` binary cannot be found
    `sudo apt install conntrack` and then retry
    * In the log listed above, the line `Using iptables Proxier` means that kube-proxy is running in `iptables` mode. Could also be `ipvs`
  * **Iptables mode**
    * In "iptables" mode, you should see something like the following on a Node
    ```
    iptables-save | grep hostnames

    # should see something like
    -A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
    -A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.3.6:9376
    -A KUBE-SERVICES -d 10.0.1.175/32 -p tcp -m comment --comment "default/hostnames: cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3
    -A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-WNBA2IHDGP2BOBGZ
    ```
    * For each port of each Service, there should be 1 rule in `KUBE-SERVICeS` and one `KUBE-SVC-<hash>` chain
    * For each Pod endpoint, there should be a small number of rules in that `KUBE-SVC-<hash>` and one `KUBE-SEP-<hash>` chain with a small number of rules in it
  * **IPVS Mode**
    * In "ipve" mode, you should see something like the following on a Node
    ```
    ipvsadm -ln

    # should see something like
    Prot LocalAddress:Port Scheduler Flags
      -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
    ...
    TCP  10.0.1.175:80 rr
      -> 10.244.0.5:9376               Masq    1      0          0
      -> 10.244.0.6:9376               Masq    1      0          0
      -> 10.244.0.7:9376               Masq    1      0          0
    ...
    ```
    * For each port of each Service, plus any NodePorts, external IPs, and load-balancer IPs, kube-proxy will create a virtual server
    * For each Pod endpoint, it will create corresponding real servers. In this example, service hostnames (`10.0.1.175:80`) has 3 endpoints
  * **Userspace mode**
    * In rare cases, you may be using "userspace" mode
    ```
    iptables-save | grep hostnames

    # should see something like
    -A KUBE-PORTALS-CONTAINER -d 10.0.1.175/32 -p tcp -m comment --comment "default/hostnames:default" -m tcp --dport 80 -j REDIRECT --to-ports 48577
    -A KUBE-PORTALS-HOST -d 10.0.1.175/32 -p tcp -m comment --comment "default/hostnames:default" -m tcp --dport 80 -j DNAT --to-destination 10.240.115.247:48577
    ```
    * There should be 2 rules for each port of your Service
* Is kube-proxy proxying?
  * Try to access your Service by IP from one of your nodes
  ```
  curl 10.0.1.175:80
  ```
  * If this fails and you are using the iptables proxy, look at the kube-proxy logs for specific lines like
  `Setting endpoints for default/hostnames:default to [10.244.0.5:9376 10.244.0.6:9376 10.244.0.7:9376]`
  * If you don't see those, try restarting `kube-proxy` with the `-v` flag set to 4, and then check the logs again

## Application Lifecycle Management - 8%
### Understand deployments and how to perform rolling updates and rollbacks
* [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
* [Canary Deployment](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#canary-deployments)
* [Kubectl Rollout - Command reference](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#rollout)

#### Key Points
* A deployment provides declarative updates for Pods and ReplicaSets
  * You describe a desired state in a deployment, and the deployment controller change sthe actual state to the desired state at a controlled rate
* Typical Use Cases
  * Create a deployment to rollout a ReplicaSet (RS) - The ReplicaSet creates Pods in the background
  * Declare the new state of the Pods by updating that PodTemplateSpec of the Deployment. A new RS is created and the Deployment manages moving the Pods from the old RS to the new one at a controller rate. Each new RS updates the revision of the Deployment
  * Rollback to an earlier Deployment revision if the current state of the Deployment is not stable. Each rollback updates the revision of the Deployment
  * Scale up the Deployment to facilitate more load
  * Pause the Deployment to apply multiple fixes to its PodTemplateSpec and then resume it to start a new rollout
* Creating a Deployment
  * After running `kubectl apply -f deployment.yaml`, use `kubectl rollout status deployment.v1.apps/deployment` to see the rollout status
  * Run `kubectl get deployments` to see all deployments
  * A Pod template hash is added by the Deployment controller to every RS that a Deployment creates or adopts
    * This label ensures that child RSs of a Deployment do not overlap
* Updating a Deployment
  * **Note:** A Deployment's rollout is triggered if and only if the Deployment's Pod template is changed. Other updates, such as scaling the deployment, do not trigger a rollout
  * Update the pods version - Use the `--record` flag to write the command executed in the resource annotation `kubernetes.io/change-cause`. It is useful to see the commands executed in each Deployment revision. Here are a few ways to do this
  `kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1 --record`
  `kubectl edit deployment/nginx-deployment` and change the version manually in your text editor
  * Check rollout status `kubectl rollout status deployment/nginx-deployment`
  * Check the new RS `kubectl get rs`
    * You will see the old one with 0 replicas and the new one
  * Get details of the deployment
  `kubectl describe deployments`
* Rolling back a deployment
  * Checking rollout history of a deployment
    * Check the revisions
    `kubectl rollout history deployment/nginx-deployment`
    * It's possible to annotate a custom `CHANGE-CAUSE` record message
    `kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="image updated to 1.16.1"`
    * To see the details of a specific revision
    `kubectl rollout history deployment/nginx-deployment --revision=2`
  * Rolling back to a previous revision
    * Undo the current rollout back to the previous revision
    `kubectl rollout undo deployment/nginx-deployment`
    * To a specific revision
    `kubectl rollout undo deployment/nginx-deployment --to-revision=1`
* Scaling a deployment
  * `kubectl scale deploy/nginx-deployment --replicas=10`
  * To enable horizontal pod autoscaling
  `kubectl autoscale deployment/nginx-deployment --min=10 --max=15 --cpu-percent=80`
* Pause deployment
  * Prevent a deployment from being updated
  `kubectl rollout pause deploy/nginx-deployment`
  * Then update the deployment and notice that it will not start the rollout process
  * To resume `kubectl rollout resume deploy/nginx-deployment`
* Deployment Status
  * Progressing Deployment
    * Deployment creates a new RS
    * Deployment is scaling up its newer RS
    * Deployment is scaling down its older RS
    * New pods become ready or available
  * Complete Deployment
    * All of the replicas associated with the Deployment have been updated to the latest version you've specified
    * All of the replicas associated with the Deployment are available
    * No old replicas for the Deployment are running
  * Failed Deployment
    * Your deployment may get stuck trying to deploy its newer RS
    * Insufficient quota
    * Readiness probe failures
    * Image pull errors
    * Insufficient permissions
    * Limit ranges
    * Application runtime misconfiguration
  * One way to detect this condition is to specify a deadline parameter in your Deployment spec `.spec.progressDeadlineSeconds`
    * Pausing a deployment does not trigger this condition
  * To see failures better, check the `conditions` section of a deployment
  `kubectl get deploy/nginx-deployment -o yaml`

### Know various ways to configure applications
* [Configuration Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
* [Inject Data into Apps](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/)
* [Secrets](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)
* [PodPreset and ConfigMaps](https://kubernetes.io/docs/tasks/inject-data-application/podpreset/)
* [Memory Resource Limits](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)

#### Key Points
* Define a command and arguments when you create a Pod
  * The `command` field corresponds to `entrypoint` in some container runtimes
* Use environment variables to define arguments for the command
  ```
  env:
  - name: MESSAGE
    value: "hello world"
  command: ["/bin/echo"]
  args: ["$(MESSAGE)"]
  ```
* Run a command in a shell
  ```
  command: ["/bin/sh"]
  args: ["-c", "while true; do echo hello; sleep 10; done"]
  ```
* Use Labels, annotations, and selectors
* Use Secrets
  * Secrets let you store and manage sensitive information that can be used in a container image
* Use PodPreset and ConfigMaps
  * Create a `PodPreset` resource with a volume mount and an environmental variable
    * Use `matchLabels` selector to match Pods that this should apply to
  * When a `PodPreset` is applied to a Pod, there will be annotations in the Pod Spec with the presets applied
  * Create a new Pod with the selected labels
    * The Pod should be updated by the admission controller with the `PodPreset` volume and env var
  * Pod spec with ConfigMap
    * Create a ConfigMap with env vars
    * Create a PodPreset manifest referencing that ConfigMap
    * Create Pod and verify the addition of env vars
  * If there is a conflict, the `PodPreset` admission controller logs a warning container the details of the conflict and does not update the Pod

### Know hwo to scale applications
* [Scaling your application](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#scaling-your-application)

#### Key Points
* `kubectl scale deployment/nginx --replicas=2`
* `kubectl autoscale deployment/nginx --min=1 --max=4`

### Understand the primitives necessary to create a self-healing application
* [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
* [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
* [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)

#### Key Points
* The primitives used are `Deployments`, `ReplicaSet`, and `DaemonSets`
* A `Deployment` is an object which can own `ReplicaSet`s and update them and their Pods via declarative, server-side rolling updates
  * When you use a Deployment, you don't have to worry about manageing the RS that they create
* A `ReplicaSet`'s purpose is to maintain a stable set of replica pods running at any given time. It is often used to guarantee the availability of a specified number of idential pods
* A `DaemonSet` can be used instead of a RS for Pods that provide a machine-level function, such as maching monitoring or logging
  * These Pods have a lifetime that is tied to a machine lifetime

## Storage - 7%
### Understand persistent volumes and know how to create them
* [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

#### Key Points
* 

## Logging/Monitoring - 5%
### Understand how to monitor all cluster components
* [Tools for Monitoring Resources](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/)
* [Kubernetes: A Monitoring Guide](https://kubernetes.io/blog/2017/05/kubernetes-monitoring-guide/)

#### Key Points
* 

## Chapter 0 - StatefulSets
### Summary
- StatefulSets solve the problem of managing unique, stateful pods with stable names and persistent storage, allowing replication.
- Stateless instances are replaceable like cattle, while stateful instances are unique like pets, needing consistent identity.
- Headless services provide unique DNS entries for each pod, enabling direct connections and communication.
- Persistent volume claims ensure data persistence, preventing loss during scaling or pod deletion.

### Problem Statement
- How do you create pods that have independent state and their own storage volumes, but can still be replicated by a replicaset?
- There are some tricks but none of them really provide a good way of doing this in k8s.
- stateful sets fix this problem. Where instances of an application need to be treated as "non-fungible" individuals where they have a stable name and state.

### Pets vs Cattle Analogy
- Stateless instances are like cattle, you dont really care about them individually and you dont name them. One can die, and another can be created without issue.
- Stateful instances are more important — like pets. When one dies, you can't just replace it. In the case of apps, the new instances need to have the same name and identity as the old one.

### StatefulSets
- When a stateful pod instance dies, the new on will get the same name, network identity and state as the one it's replacing.
- They can be scaled similar to replicationcontrollers and replicasets.
- Each pod created by a statefulset gets its own set of volumes/storage.
- Pods in stateful sets are given incremental index names (`A-0`, `A-1`, ...) because that makes them predictable.
- These pods need to retain their hostname as well, because you may want to work on a specific pod that has a specific state.

### Headless (Governing) Service
- All "headless" means is it does not have a stable IP address (cluster IP). When queried, it returns the IP addresses of the individual pods, allowing clients to connect directly to these pods.
	- _Think about it, if something is headless it means it doesn't come through a single source._
- In order to retain their hostnames, you need to create a governing headless service to provide the actual network identity to each pod.
- In this service, each pod gets its own DNS entry.
- So a pod in a statefulset might be reachable via `a-0.foo.default.svc.cluster.local`
- You can look up all the pods in a statefulset via SRV records for `foo.default.svc.cluster.local`.

### Persistent Volume Claims
- For every pod that statefulset creates, it also needs to create a persistent volume claim from a template that exists in the statefulset.

>![[Pasted image 20240609112334.png]]

- Scaling down deletes the pods, but not the claims. Because if it did, those volumes would be recycled and lost. We dont want that because if a pod gets accidentally deleted, we don't want its data to be lost.

### StatefulSet Manifest

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: kubia
spec:
  serviceName: kubia  // Name of headless service 
  replicas: 2
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia-pet
        ports:
        - name: http
          containerPort: 8080
        volumeMounts:
        - name: data
          mountPath: /var/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      resources:
        requests:
          storage: 1Mi
      accessModes:
      - ReadWriteOnce
```

- `volumeClaimTemplates` will be used to create a persistent volume claim for each pod. The names of the pvc for each pod become `<vc-template-name>-<pod-name>`

### SRV records
- These records point to the hostnames of the pods backing the headless service.
- This allows these stateful pods to discover and communicate with each other! 

### How StatefulSets Deal With Failures
- It won't create a new pod until it knows for certain the old one has stopped running.
- If a node's network goes down, the control plane may never know if the pod got actually deleted so you'll have to manually delete it.
- Don't do this unless you are 100% sure that the node is no longer running or reachable.

## Chapter 1 - Understanding Kubernetes Internals
### Summary
- The control plane consists of etcd, API server, scheduler, and controller manager, while worker nodes have kubelet, kube-proxy, and container runtime.
- etcd is a fast, consistent, distributed key-value store accessed only by the API server using the RAFT algorithm.
- The API server provides a RESTful interface for CRUD operations on etcd and handles client requests through authentication, authorization, and admission control.
- The scheduler assigns pods to nodes based on resource availability and affinity, while kubelet deploys and manages these pods on each node.
- kube-proxy manages iptables rules for service traffic routing, and CNI plugins enable node-to-node communication.
- Pods communicate through a NAT-less network set up by CNI plugins, using virtual ethernet pairs and network bridges on each node.

### Architecture
- On control plane:
	- etcd
	- API server
	- scheduler
	- controller manager
- On worker nodes:
	- kubelet
	- kubernetes service proxy
	- container runtime
- Communication by components and api server are usually initiated by the components.
- Most of the resources (expect for kubelet) run as pods in the `kube-system` namespace.
>![[Pasted image 20240608215919.png]]
### etcd
- fast consistent, distributed, key-value store.
- The API server is the only thing that talks to etcd.
- keys in etcd look like directories, but they are just strings really. like `/registry/configmaps/...`
- Uses RAFT algorithm to achieve consistent state.

### RAFT algorithm
- Majority rules. If the cluster goes into a "split-brain" situation, the split with more etcd instances will have the true state. 
- Usually there is one "leader" that handles requests and coordinates updates to cluster state.
- The side with minority becomes read only until all instances are synced back up to a new state.

### API Server
- A RESTful API that can perform CRUD operations on the etcd.
- An interface for querying and modifying the cluster state.
- `kubectl` is one of the clients we are familiar with using to communicate to the apiserver.
- A client request goes through 4 main stages.
	- Authentication: Figures out who you are.
	- Authorization: Figures out if you can do what you want.
	- Admission Control: Modify resources for different reasons, like init missing fields or overriding them.

- The API server is _observed_ by other components. So changes to a resource by the API server may alert another resource that is interested in that change.
- For instance, if a resource is created, changed or deleted, the control plane component is notified.
- Every time an object is updated, the server sends the new version of the object to all connected clients that are subscribed to it.

### Scheduler
- All the scheduler does is update the pod definition through the API server.
- The kubelet (who watches the API server for updates) is the one who actually deploys the pod.
- It first finds _acceptable pods_ by looking at hardware resources, pod affinity and more.
- It then picks the best option from the bunch.

### Controllers running in the Controller Manager
- The controller manager is a single program that runs each of these controllers as _goroutines_. 
- The controllers do the job of converging to the desired state as specified in the resources deployed through the API server.
- There is a controller for each resource, to basically manage it.
- ★ "Resources are _descriptions_ of what should be running in the cluster, Controllers are the **_active_** kubernetes components that perform the actual work as a result of the deployed resources."
- Each controller subscribes to the API server for the resources they are responsible for. (Using "watch" mechanism).
- Keep in mind that controllers post the api server the same way a user would.

### Controllers
- **ReplicaSet Controller**: It sees that the desired pod state is not met, it creates new Pod manifests and posts them to the API server. Then it lets the scheduler and kubelet do their jobs.
- **Deployment Controller**: It works by creating or removing replicasets.
- **Namespace Controller:** When a namespace is deleted, all the resources are also deleted.
- **Endpoints Controller**: Keeps the endpoints list constantly updated with the IPs and ports of pods matching the label selector.

### Kubelet
- Registers the node its running on by creating a Node resource.
- Monitors the API server for pods that have been scheduled to its node.
- It then starts those Pod's containers.
- It also monitors all the containers and reports their status to the api server.
- Kubelet is the thing running the container liveness probes and restarts then when they fail.
- It can also run containerized versions of the control plane.

### kube-proxy
- One kube-proxy runs on each node.
- The kube-proxy watches for changes to services using the api server's watch mechanism and updates the [[iptables]] rules depending on the ips of the pods. 
- iptables perform the actual load-balancing.

### How DNS server works
- All pods use the cluster's internal DNS by default.
- kube-dns pod watches for changes to services and endpoints and updates the dns records with every change.

### How Ingress controllers work
- Its a [[Reverse Proxy]].
- Watches for changes in ingress, services, and endpoints resources and updates the config of the proxy server (e.g. nginx) accordingly.
- It does **not** involve iptables, the reverse proxy handles the traffic in this case.
- It routes directly to the Pods and does not go through services at all.

### Event resource
- The control plane and kubelet emit Events.
- `kubectl get events --watch` to retrieve events as they occur.

### Understanding what a running Pod is
- A "pause" container is created when a Pod is created on a Node.
- This container is what holds all the containers of a Pod together!
- It serves as an infrastructure container so that the containers of a Pod share the same network and [[Linux Namespace|linux namespace]].
- The lifecycle of this Pod is tied to that of the other containers in the Pod.

### Inter-pod networking
- It's [[NAT]]-less and each Pod gets its own unique IP address.
- This is setup by a Container Network Interface (CNI) plugin.
- For communication between Pods to work, the IPs must remain consistent, hence the NAT-less.
- Pods talking to outside services does have NAT.
- Pods on a node are connected to the same bridge through **_virtual ethernet interface pairs_**.
- One interface of the pair remains in the host's namespace (node) listed `vethXXXX`. The other is moved into the "pause" container's networking namespace and renamed `eth0`.
- These work like two ends of a pipe.
- The interface of the host's network namespace is attached to a network bridge.
- So if pod A sends a network packet to pod B, the packet first goes through pod A’s veth pair to the bridge and then through pod B’s veth pair. All containers on a node are connected to the same bridge, which means they can all communicate with each other.
- For each pod on a node, there is one network interface (the host end of the veth pair) in the host's network namespace that connects to the network bridge.
- Connecting between nodes means connecting the node bridge to the physical network adapter of the node.
- Then the connection travels "over the wire" to the other node.
- It's also worth noting that each node has a certain IP range for its pods so that there is no overlap.

### Container Network Interface (CNI)
- They are the layer of network that allows for node-to-node communication.

### How Services are implemented
- The service IP isn't real. Its just used by the iptables to know "when you see IP A, re-route that to pod IPs B-Z". That's why it isn't pingable.
- kube-proxy uses iptables rules to redirect packets destined for the service IP and reroute them to one of the pod ips backing the service.
### Running highly available clusters
- Leader election means that one of them is doing stuff while the others are inactive.
- It can also mean that one is performing the writes while the others are readonly.
- Its normal to have multiple instances of the control plane, with only one being active.
- They choose who is leader by racing to see who is the first to put that title in their `metadata.annotations`.
- It uses [[Optimistic Concurrency|optimistic concurrency]] to do this safely.

## Chapter 5 - Services
### Summary
- Services provide load-balanced communication to pods based on labels, accessible within or outside the cluster.
- Use `Endpoints` to connect to external services and `Ingress` for exposing services to external clients with path-based routing.
- Readiness probes ensure a pod is ready to handle requests before traffic is sent to it.
- Headless Services, with `clusterIP` set to `None`, directly expose pod IPs without an assigned service IP.

### Creating Services

- Allow you a load-balanced way to communicate to all pods of a certain label.
- Use label selectors to decide which pods are part of the service.
- `CLUSTER-IP` means it's only accessible within the cluster, not outside.
- Services are meant to expose a group of pods to another group of pods _within the cluster and outside_.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  ports:
    - name: http
      port: 80           // Port this service is available on
      targetPort: 8080   // Container port the service will forward to
    - name: https
      port: 443          // Port this service is available on
      targetPort: 8443   // Container port the service will forward to
      
  selector:
    app: kubia  // All pods with app=kubia will be part of service
```

- You can hit the service from a pod using `exec`:

```bash
kubectl exec <pod name> -- curl -s http://<SERVICE IP>
```

- If you specify port names of the Pod manifest, then you can reference those port names  in the service. This is helpful so that when you change one number, you dont need to change the other.
- You can discover service IPs through env variables, DNS, or FQDN.

### Connecting to Services via FQDN

```
backend-database.default.svc.cluster.local
```

- `backend-database` is the service name.
- `default` is the namespace.
- These are the only two things that are necessary to hit the service.

```
kubectl exec -it <pod name> bash
...
root@kubia-3inly:/# curl http://kubia.default
> You’ve hit kubia-3inly
```

### Connecting to Services outside of the cluster
- You can create an `Endpoints` resource where you establish a set of IPs to hit.
- Couple that with a service resource to talk to an outside service.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
    - port: 80
```

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: someapi.somecompany.com
  ports:
    - port: 80
subsets:
  - addresses: 
    - ip: 11.11.11.11
    - ip: 22.22.22.22
```

- You can use direct IP addresses or FQDN of the service.

### Exposing using NodePort
- NodePort allows you to expose an app using the nodes ip by assigning the port for that app. All nodes on the cluster will make sure that the given port will route to that app.
- So you can have multiple apps using the NodePort as long as they are using different port numbers.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: biggorilla-service
spec:
  type: NodePort
  selector:
    app: biggorilla
  ports:
    - protocol: TCP
      port: 80            // Port of services interal cluster ip
      targetPort: 3000    // Target port of backing pod
      nodePort: 30123     // Service will be accessing through this port
```

### Exposing services to external clients using Ingress
- You can make services accessible externally using `NodePort`, `LoadBalancer` or `Ingress`.
- A `NodePort` service makes the service accessible from outside the cluster.
- When a client sends a request to an Ingress, the host and path in the request determines which service the request is forwarded to.
- Ingress is just a filter for the given client request.

- Notice how we can configure different hosts as well as different paths.
- You can also enable TLS (Transport Layer Security), which encrypts communication from the client to the ingress controller (_Page 179_).

```yaml
apiVersion: v1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
    - host: kubia.example.com
      http:
        paths:
          - path: /kubia
            backend:
              serviceName: kubia-nodeport
              servicePort: 80
		  - path: /foo
            backend:
              serviceName: foo
              servicePort: 80
    - host: foo.example.com
      http:
        paths:
          - path: /
            backend:
              serviceName: bar
              servicePort: 80
            
```

### Readiness Probes
- Probe a pod by sending a request to it and seeing the response.
- When the response is successful, we can start sending client requests to that pod.
- `readinessProbe` is added to the Pod manifest.

### Creating a Headless Service
- Setting the `clusterIP` field to `None` makes a Service _headless_.
- This means you k8s won't assign an IP to the service.
- This will give you the Pod IP's directly.

## Chapter 6: Volumes
### Summary
- Volumes allow containers within a Pod to share data, and they are ephemeral.
- `emptyDir` volumes provide a shared empty directory for containers.
- `gitRepo` volumes clone a git repository into an `emptyDir`.
- PersistentVolumes (PVs) and PersistentVolumeClaims (PVCs) provide persistent storage, decoupling storage from pods.

### Introducing Volumes
- Volumes all containers within the same Pod to share data.
- Each container can mount the volume in any location in their filesystem.
- Volumes are ephemeral — they will be lost when the Pod restarts.

### Using `emptyDir`
- Used to share data between containers.
- Literally just creates an empty directory which containers can read/write to.
- In the example below, the volume is being shared between the two containers. Notice how they are mounting it at different locations.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
	- image: luksa/fortune
	  name: html-generator
	  volumeMounts:
	  - name: html
	    mountPath: /var/htdocs
	- image: nginx:alpine
	  name: web-server
	  volumeMounts:
	  - name: html
		mountPath: /usr/share/nginx/html
	    readOnly: true
	  ports:
	  - containerPort: 80
	    protocol: TCP
	volumes:
	- name: html
	  emptyDir: {}
```

### Using `gitRepo`
- It just `emptyDir` but then a git repo is cloned into that directory.
- Can use this to serve the latest version of your git repo.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gitrepo
spec:
  containers:
	- image: nginx:alpine
	  name: web-server
	  volumeMounts:
	  - name: html
		mountPath: /usr/share/nginx/html
	    readOnly: true
	  ports:
	  - containerPort: 80
	    protocol: TCP
	volumes:
	- name: html
	  gitRepo:
	    repository: https://github.com/maxcelant/pylox.git
	    revision: master
	    directory: .      // Clone into the root dir of the volume
```

### Using `hostPath`
- This type of volume allows you to point to a node's filesystem.
- Don't store Pod specific data on a node though, because if that pod changes nodes, that will be lost.
- Only good for persistent storage on a _one node cluster_ like Minikube.
>![[Pasted image 20240506160006.png]]

### PersistentVolumes and PersistentVolumeClaims
- You create a `PersistentVolume` resource where you specify the size and access it supports.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
    - ReadOnlyMany
  persistentVolumeClaimPolicy: Retain
  gcePersistentDisk:
    pdName: mongodb
    fsType: ext4
```

- By setting the `persistentVolumeClaimPolicy: Retain`, that means that even after this volume detaches from the Pod, itll still hold that data.
- User submits a `PersistentVolumeClaim` resource where they configure how much capacity and what access rights to they need and k8s will bind a `PersistentVolume` to the Pod based on that claim!

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  resource:
    requests:
      storage: 1Gi
  accessModes:
    - ReadWriteOnce
```

- PersistentVolumes, like cluster nodes, dont belong to any specific namespace.
- In your pod manifest, you reference the PVC not the PV directly, decoupling the storage from the requester.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
	- image: mongodb
	  name: mongodb
	  volumeMounts:
	  - name: mongodb-data
		mountPath: /data/db
	  ports:
	  - containerPort: 27017
	    protocol: TCP
	volumes:
	- name: mongodb-data
	  persistentVolumeClaim:
	    claimName: mongodb-pvc   // Reference the claim!
```

## Chapter 7: ConfigMaps
### Summary
- Kubernetes uses ConfigMaps and Secrets to securely manage and decouple application configurations from container images.
- Embedding configs in Docker images is insecure; Kubernetes overcomes this with dynamic config management.
- `ENTRYPOINT` and `CMD` define container behavior, while environment variables and command-line arguments can be injected into pods.
- Secrets manage sensitive information securely, and Kubernetes ensures atomic updates to config files.

### Problem with Docker and Configs
- Using configuration files in Docker containers is tricky because you would have to "bake it" into the container image. 
- This is roughly equivalent to hard-coding config values into the app.
- This is also insecure.

### `ENTRYPOINT` and `CMD`
- `ENTRYPOINT` defines the executable invoked when the container is started.
- `CMD` specifies the args that get passed to the `ENTRYPOINT`.
- Use `exec` form for `ENTRYPOINT`

### Command and arguments in K8s
- You can also run your pods and inject command line arguments into them with the Pod yaml.

```yaml
kind: Pod
spec:
  containers:
  - image: some/image
	command: ["/bin/command"]
	args: ["arg1", "arg2", "arg3"]
```

### Environment variables in k8s
- Setting env vars is done in the container definition
```yaml
kind: Pod
spec:
 containers:
 - image: luksa/fortune:env
   env:
   - name: INTERVAL
value: "30"
   name: html-generator
```

### `ConfigMap`s purpose
- We want to decouple the app's config from the app itself. Because those options can change!
- ConfigMap is a map of key-value pairs. These contents are passed to the containers as environment variables or as files in a volume.
- You can have ConfigMap's with the same name, living in the different namespaces (dev, prod). 
- The actual contents of the ConfigMaps will be different (because its different namespaces) but this allows you to use the same name in the Pod specs of each namespace.

### Creating a `ConfigMap`
```yaml
apiVersion: v1
data:
  sleep-interval: "25"
kind: ConfigMap
metadata:
  creationTimestamp: 2016-08-11T20:31:08Z
  name: fortune-config
  namespace: default
  resourceVersion: "910025"
  selfLink: /api/v1/namespaces/default/configmaps/fortune-config
  uid: 88c4167e-6002-11e6-a50d-42010af00237
```
- Your sources can be anything from a json file to a whole directory.
```bash
$ kubectl create configmap my-config
	--from-file=foo.json  
	--from-file=bar=foobar.conf  
	--from-file=config-opts/
	--from-literal=some=thing
```

### Using env variables from a `ConfigMap`
- The environment variables is called `INTERVAL`.
- We reference the key `sleep-interval`.
```yaml
metadata:
  name: fortune-env-from-configmap
spec:
  containers:
  - image: luksa/fortune:env
    env:
    - name: INTERVAL
	  valueFrom:
		configMapKeyRef:
		  name: fortune-config
		  key: sleep-interval
```
- You can also expose all the keys from a `ConfigMap` using a prefix.
```yaml
spec:
  containers:
  - image: some-image
    envFrom:
    - prefix: CONFIG_
      configMapRef:
        name: my-config-map
```
- So if it had `FOO` and `BAR` in the config map, that would be `CONFIG_FOO` and `CONFIG_BAR`.
- You can also pass config map keys as command-line arguments.

### Using a `ConfigMap` volumes
- Mostly used for larger config files.
- This gives you the ability to update the config without recreating the pod or restarting the container.
- Anything in the `configmap-files` directory will be added to the configmap.

```bash
$ kubectl create configmap fortune-config \
			--from-file=configmap-files

configmap "fortune-config" created
```
- You can also mount volumes with only a _portion_ of the key-values in a configmap.
```yaml
volumes:
- name: config
  configMap:
    name: fortune-config
    items:
    - key: my-nginx-config.conf // Which keys to include
      path: gzip.conf           // Entrys value stored here
```

### How Mounting Directories Works
- When you mount a volume at a certain directory, the original content at the position will be overridden by the mounted volume.
- So make sure you put it in a place that doesn't already have important info!
- You can use `subPath` to mount it there _without_ affecting the pre-existing data there.

```yaml
spec:
  containers:
  - image: some/image
    volumeMounts:
    - name: myvolume
      mountPath: /etc/someconfig.conf // Only mounting a file
      subPath: myconfig.conf   // Only mounting this entry
```

### Atomically updating the config
- What happens if the app detects a config file change and reloads before k8s is done updating all the files in the config volume?
- When the configmap is updated, k8s creates a dir, writes all the files to it and the relinks a `..data` symbolic link to point to this new dir.
- Doing this effectively atomically updates all of them at once.
- Updating a config map mounted at a volume does not happen synchronously across all instances. It can take up to a whole minute for pods to sync.

### Using `Secrets` resource
- Secrets are encoded in base64. This allows it contain binary values, not just plain-text.
- You can expose secrets as env variables similar to configmaps, but this is not recommended because env variables are usually dumped in error reports.
- In general, use secret volumes.
- When a secret volume is exposed to a container (or as a env variable), it is decoded and written to the file in actual form.

### Pod Yaml Definiton Breakdown

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-https
spec:
  containers:
    - image: luksa/fortune:env
      name: html-generator
      env:
        - name: INTERVAL
          valueFrom:
            configMapKeyRef:
              name: fortune-config
              key: sleep-interval
      volumeMounts:
        - name: html
          mountPath: /var/htdocs
    - image: nginx:alpine
      name: web-server
      volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
          readOnly: true
        - name: config
          mountPath: /etc/nginx/conf.d
          readOnly: true
        - name: certs
          mountPath: /etc/nginx/certs/
          readOnly: true
  ports:
    - containerPort: 80
    - containerPort: 443
  volumes:
    - name: html
      emptyDir: {}
    - name: config
      configMap:
        name: fortune-config
        items:
          - key: my-nginx-config.conf
            path: https.conf
    - name: certs
      secret:
        secretName: fortune-https
```

- It mounts three volumes: `html`, `config`, and `certs`.
- The `config` volume is mapped to a `configMap` called `fortune-config`
	- From that, we want the `my-nginx-config.conf` key. 
	- This will be available at `/etc/nginx/conf.d/https.conf`.
	- More info on the `conf.d` convention [[060124 - Linux dir.d convention|here]].
- The `certs` volume is populated from the data in `fortune-https`.
	- This data is mounted at `/etc/nginx/certs/`

### Image Pull Secrets
- When pulling images from a private docker hub repo, you need to specify `imagePullSecrets`.


## Chapter 8: Accessing Pod Metadata
### Summary
- You can use the `downwardAPI` to get metadata about the Pod's environment through a volume or environment vars.'
- You can speak to the Kubernetes API from a Pod through a proxy.
- You can also authenticate and/or use an ambassador to communicate with it.
- For more complex tasks, you can use a dedicated kubernetes client.

### Intro to the Downward API
- The downward api allows us to send metadata about the pod and its environment through env variables or files (using `downwardAPI` volume).
- The downwardAPI is _not_ a REST endpoint that can be hit. Its just metadata about the environment which can be reached using env variables or volumes.

### Exposing metadata through env variables
- These are just a few things that can be looked at from the `downwardAPI`.
- `annotations` and `labels` cannot be exposed through environment variables because they can't be changed while the pod is alive.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward
spec:
  containers:
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
```

### Exposing metadata through volume
- `path` refers to the file name in which the metadata will be stored.
- If you want to expose as containers resource field, then you need to specify the name of that container.
```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
    volumeMounts:
    - name: downward
      mountPath: /etc/downward
  volumes:
  - name: downward
    downwardAPI:
      items:
      - path: "podName"
        fieldRef:
          fieldPath: metadata.name
      - path: "podNamespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
      - path: "containerCpuRequestMilliCores"
        resourceFieldRef:
          containerName: main
          resource: requests.cpu
          divisor: 1m
      - path: "containerMemoryLimitBytes"
        resourceFieldRef:
          containerName: main
          resource: limits.memory

```

### Kubernetes API
- When you need metadata other than that of the Pod, you will need to use the Kubernetes API.
- You need to use a proxy to talk to the k8s API.
- It allows you to learn about how resources can be used.

```bash
$ curl http://localhost:8001/apis/batch/v1
```

- For instance this can show you information about `batch/v1`. Giving information about `Jobs` like what operations can be done on that resource, etc.

```bash
$ curl http://localhost:8001/apis/batch/v1/jobs
```

- Going further, you can look at the jobs running in your cluster.
- You can also look at a specific job (or any resource) if you have it's `name`.
- You can get the exact same info by doing: `kubectl get job my-job -o json`

### Talking to API Server from a Pod
- For a Pod to talk to the API server, you need to authenticate the communication.
- To find the k8s API you can do `kubectl get svc` and look for the `kubernetes` service.
- You can also do `grep` the `KUBERNETES_SERVICE` from within the pod in the `env` folder.

```
root@curl:/# env | grep KUBERNETES_SERVICE

KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_HOST=10.0.0.1
KUBERNETES_SERVICE_PORT_HTTPS=443
```

- You can also point to `curl https://kubernetes:443`
- To verify you are talking to the API server, you need to check if the server's cert is signed by the CA.
- The `ca.cert`, `token` and `namespace` can all be found in a secret volume attached to every Pod.
- `namespace` contains the namespace that the Pod is running in.

### Using an ambassador container
- You can use this to communicate with the k8s API securely.
- You run an ambassador container along side your app container and connect your app to it. Then let the ambassador talk to the API server.
- `kubectl proxy` binds to port 8001. Since both containers share a [[Loopback and Network Interface|network interface]], this works!

### Using a client library
- If you plan on doing more than simple API requests, its better to use a dedicated k8s API client library.

## Chapter 9: Deployments
### Summary
- Concurrent updates replace all pods at once, while rolling updates switch versions gradually, both methods being prone to errors.
- Use Deployments to manage updates declaratively, ensuring smoother rollouts and rollback capabilities.
- Choose between `Recreate` (all at once) and `RollingUpdate` (one by one) for updating pods.
- Parameters like `maxSurge` and `maxUnavailable` help manage the rollout rate and availability.

### Updating Pods Manually — Concurrent
- Before Deployments were a thing, the old way of rolling out a new container version was changing the Pod template to refer to container `v2`, and slowly deleting the pods of `v1` and letting the ReplicaSet start up the new ones.
- This method means deleting all the pods at once and letting the ReplicaSet create all of the new ones at the same time.

### Updating Pods Manually — Rolling
- Performing a manual rolling update is more difficult. It involves having to ReplicaSets and winding down the Pods in `v2` while increasing those in `v2`.
- This is obviously more error prone, because its more commands also your app may not support multiple concurrent version types.

### Updates to Same Image Tag
- Currently, if you update an image but use the same tag, it probably wont be updated because the image is cached on the node, and that will be pulled instead.

### Rolling Updates of a ReplicationController
- This method is no longer the way to do things but you use to be able to do a rolling-update using a replicationcontroller.
- `kubia-v1` is the controller you want to replace.
- `kubia-v2` is the name of the new one
- `foo/kubia:v2` is the image to start running in the Pods.

```bash
kubectl rolling-update kubia-v1 kubia-v2 --image=foo/kubia:v2
```

- The Pods get a new label with key `deployments` and some `id` as the value. That value is to denote which replicationcontroller is in charge of it.

>![[Pasted image 20240607221719.png]]

- All it does it starts scaling the old controller down while scaling the other up.

### Why rolling-update is obsolete
- rolling-update actually _modifies_ resources, which is kind of against kubernetes law in a way. Since it should be a _declarative_.
- Also, this is performed by the client and not the server. So if something happens to the connection along the way, it could cause some issues.

### Deployments to Update Declaratively
- Deployment manages the replicasets, replicasets manage the pods
- The deployment name shouldn't have an app version, since it controls that.

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v1
name: nodejs
```

- Use `--record` flag when creating a deployment. It records revision history which is useful for rollbacks.
- When you look at the names of the pods run by a deployment, the middle hash is the replicaset that controls them. So when you do rolling updates, you'll see those change.

```bash
$ kubectl get po
NAME                     READY     STATUS    RESTARTS   AGE
kubia-1506449474-otnnh   1/1       Running   0          14s
kubia-1506449474-vmn7s   1/1       Running   0          14s
kubia-1506449474-xis6m   1/1       Running   0          14s
```

- All you need to do is reference a new container image, and it will automatically start the update!
- `kubectl set image` lets you set a new container image.
- Unlike the rolling-update, this is performed by the control plane, not the client.

### Deployment Strategies
- `Recreate` removes all the old pods and starts all the new ones at once.
- `RollingUpdate` removes them one by one, keeping the app available the whole time.

### Controlling Rate of Rollout
- `maxSurge` means "how many MORE pods can we create than your replica count?"
	- So if you have total pods set to 3 and `maxSurge` set to 1, then at most, there will be 4 pods.
- `maxUnavailable` means "how many pods can we make unavailable relative to your replica count?"
	- So if you have a total of 4 pods and set `maxUnavailable` to 1, then there will always be 3 running.
- Remember that these are both _relative_ to your replica count.

>![[Pasted image 20240608092553.png]]

### Rolling Back
- The deployment will keep the replicaset of the previous version so that if you need to perform a rollback, all you need to do is start increasing the pod count on that replicaset and decreasing the current version.
- `undo` command is used to abort or undo a roll out.

```bash
$ kubectl rollout undo deployment kubia
```

- Each replicaset stores the complete information of the deployment at that specific revision, so don't delete it manually.
- `revisionHistoryLimit` is how many recent replicasets to keep.
- You can also pause a rollout. This can work like a canary release. Can also allow you to make multiple changes to deployment and then kick it off.

```bash
$ kubectl rollout pause deployment kubia
```
### Slowing Roll Outs
- `minReadySeconds` specifies how long a pod should be ready before the pod is treated as _available_.
- Once the pod is available, the rollout will continue.
- You should set `minReadySeconds` to a higher value, to make sure that pods are steady before continuing.
- If a pod starts failing before `minReadySeconds` amount of time, it will automatically roll back.
- For example, if you set it to 10 seconds. Then a Pod needs to be ready for _at least_ 10 seconds

## Chapter 14 - Managing Pods Computational Resources
### Summary
- You can give max and mins for cpu and memory resources in a cluster.
- Quality of Service classes categorize your pods in a "to-kill" priority.
- LimitRange resources give resource mins and maxes for newly created pods in a namespace.
- ResourceQuota set hard resource limits (as well as other limits) for the whole namespace. 
- Pod resource monitoring is done by cAdvisor to Heapster.

### Requesting Resources
- The _minimum_ amount of resources your pod needs, not the _max_.
- Specified for each container in a Pod.
- `cpu: 200m` means 200 milicores which means 1/5 of a cpu. 

```bash
k exec -it requests-pod top
```

- `top` can give you the CPU consumption.
- The scheduler takes this into account when assigning a pod to a node.
- Scheduling always works on how much was _actually_ allotted not how much is currently being used. 
	- So if a node is technically 80% full but its only really using 70% and a new pod requests at least 25%, it can't be scheduled to that node.
- Scheduler prioritizes nodes with heavy requests on them to bundle pods tightly to hopefully free up and remove unused nodes.
- If the scheduler can't fulfill your resource requests on any node, the pod will remain `Pending`.
- Any free cpu is split up accordingly between the pods on the node.

>![[Pasted image 20240624115700.png]]

### Limiting Resources
- You should limit the memory given to a container, because that cannot be taken back like the cpu can.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: limited-pod
spec:
  containers:
  - image: busybox
    command: ["dd", "if=/dev/zero", "of=/dev/null"]
    name: main
	resources:
	  limits:
		cpu: 1
		memory: 20Mi
```

- Sum of all resource limits are _allowed_ to exceed 100% of the nodes capacity.
- CPU is throttled, so it cannot exceed its limit.
- If a process tries to exceed its memory allocated, it is killed and restarted.
- `OOMKilled` means a pod was killed because "Out of Memory".
- You can see why a pod was killed by doing a `describe` on it.
- `top` command shows the memory and cpu amounts of the whole node the container is running on.

### Quality of Service (QoS) classes
- Resource limits can be overcommitted. QoS decides which pod stays and which pod is killed when necessary.
- Can be found in pod yaml at `status.qosClass`.
- `BestEffort` are pods with no resource limits or requirements. They are the first killed.
- `Guarenteed` are pods that have limits set (or their requests match their limits)
- `Burstable` are pods which resource limits don't match its requests.
- Priority is `Guarenteed` > `Burstable` > `BestEffort`
>![[Pasted image 20240624142848.png]]

- To determine which pod is killed when the pods have the same QoS, you need to look at OutOfMemory (OOM) score.
	- percentage of available memory the process is consuming and fixed OOM score (based on pods QoS class and containers requested memory.)
- Basically the one using more memory gets killed off first.

### LimitRange object
- Used by LimitRange Admission control plugin. 
- When you post your pod spec, the resource is validated to make sure the pod doesn't go above the resource limits.
- These limits apply to all pods created in the same namespace as the limitrange resource.
- You can also use this to set defaults for pods without explicit resource limits.

### ResourceQuota object
- Used by ResourceQuota Admission control plugin
- Allows you to specify the _overall_ cpu and memory the pods in a namespace are allowed to consume.

```yaml
apiVersion: v1
	kind: ResourceQuota
	metadata:
	  name: cpu-and-mem
	spec:
	  hard:
		requests.cpu: 400m
		requests.memory: 200Mi
		limits.cpu: 600m
		limits.memory: 500Mi
```

- It will block new pods created that will cause you to exceed these hard limits.
- If you try to create a pod without requests or limits it will fail if you have a resource quota.
- Can also limit different resource types.
- `activeDeadlineSeconds` is the amount of time a pod will live for before it is killed and restarted.
- You can apply quotas for pods with a specific QoS.

### Monitoring pod usage
- **cAdvisor** on the kubelet performs a basic collection of resource consumption data.
- **Heapster** is a component that gathers that data for the whole cluster.

```bash
$ k top node // displays cpu and memory usage
```

```bash
$ k top pod --all-namespaces // actual cpu and mem usage of pods
```

## Chapter 15: Automatic Scaling of Pods and Cluster Nodes
### Summary
- `HorizontalPodAutoscaler` is a resource that will automatically scale your pods to keep them below a resource threshold.
- Vertical autoscaling is not yet available (sad).
- Cluster autoscaling allows kubernetes to provision more/less nodes from the cloud provider depending on the usage.
- `PodDisruptionBudget` allows you to set a min and max for number of pods that are always available of a label in the cluster.
### Horizontal pod autoscaling
- Performed by Horizontal controller which is configured by a HorizontalPodAutoscaler (HPA).
- Uses heapster to get its stats.
- Calculating the required replica count is simple when it considers only a single metric.

```ad-example
Pod 1: 60% CPU utilization
Pod 2: 90% CPU utilization
Pod 3: 50% CPU utilization
Target Utilization: 50% CPU
(60 + 90 + 50) / 50 = 200 / 50 = 4 (replicas)
```

- HPA only modifies on the Scale sub-resource.
- This allows it to operate on any scalable resource (e.g. pods, deployments, statefulsets).
- Autoscaler compares a pods actual cpu consumption and its requested amount so pods will need to have that set.
- `autoscale` command can create a HPA quickly.
- Remember that a container's cpu utilization is the container's **actual** cpu usage divided by its requested cpu.

```bash
$ k autoscale deployment kubia --cpu-percentage=30 --min=1 --max=5
```

### Creating a HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: kubia    // Does not need to match name of deployment
spec:
  maxReplicas: 5
  metrics:
    - resource:
        name: cpu
        targetAverageUtilization: 30
      type: Resource
  minReplicas: 1
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: kubia
status:
  currentMetrics: []
  currentReplicas: 3
  desiredReplicas: 0
```

- There is a limit on how many replicas an hpa can create in a single operation.
### Running commands in the cluster

```bash
$ kubectl run -it --rm --restart=Never loadgenerator --image=busybox
➥ -- sh -c "while true; do wget -O - -q http://kubia.default; done"
```

- This allows you to create an unmanaged pod on the cluster that will be deleted when you type CTRL+C.

### Metric Types
- `Resource`: bases its autoscaling on a resource.

```yaml
... spec:
  maxReplicas: 5
  metrics:
  - type: Resource
	resource:
      name: cpu
      targetAverageUtilization: 30
```

- `Pods`: any metrics related to pods directly like queries per second (QPS).

```yaml
spec:
  metrics:
  - type: Pods
	resource:
      metricName: qps
      targetAverageValue: 100
```

- `Object`: scales pods on a metric that doesn't pertain directly to those pods. The autoscaler obtains a single metric from the single object (e.g. Ingress).

```yaml
... spec:
  metrics:
  - type: Object
    resource:
      metricName: latencyMillis
      target:
        apiVersion: extensions/v1beta1
        kind: Ingress
        name: frontend
      targetValue: 20
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: kubia
...
```

### Vertical pod autoscaling
- Kubernetes does not currently support this.
- You can use a Admission Control plugin called InitialResources to set usage requests for a deployment by looking at the historical usage for that deployment.

### Cluster Autoscaler
- Automatically provisions additional nodes when the existing nodes are full.
- Nodes are grouped by type (called pools). So you need to specify the node type when you want an additional one.

>![[Pasted image 20240628130934.png]]


- If the cpu and memory requests on a given node are below 50%, that node is considered "unnecessary".
- If a system pod is running on a node, the node won't be relinquished.
- If a node is selected for shut down, all the pods are evicted.

### Cordoning and draining nodes
- `kubectl cordon <node>` marks the node as unschedulable.
- `kubectl drain <node>` marks the node as unschedulable and then evicts all the pods from the node.

### PodDisruptionBudget resource
- Allows you to set how many pods of a certain label should always be available.

```yaml
$ kubectl get pdb kubia-pdb -o yaml

apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: kubia-pdb
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: kubia
status:
...
```

- the `kubectl drain` command will adhere to it and will never evict a pod with the `app=kubia` label if that would bring the number of such pods below three.

## Chapter 17: Best Practices for Developing Apps
### Summary


### Understanding the pod's lifecycle
- Like tiny VMs dedicated to running only a single application.
- Local IP and hostname will change.
- Anything written to the disk or containers filesystem will be lost when a pod is moved or killed.
- Using volumes will persist data on container restarts (but not pod restarts).
- Using volumes to preserve files across container restarts like this is a double-edged sword. What if the data gets corrupted and causes the newly created process to crash again?
- The replicaset does not care if one of its pods is dead, only that the desired pod count is correct.

### Init containers
- 1 or more containers that you can set to run before your main container.
- Can be used to ping the service when your app is coming up and then have it stop when the service finally up and running.

```yaml
spec:
  initContainers:
  - name: init
    image: busybox
    command:
    - sh
    - -c
	- 'while true; do echo "Waiting for fortune service to come up...";
	  'wget http://fortune -q -T 1 -O /dev/null >/dev/null 2>/dev/null
	  '&& break; sleep 1; done; echo "Service is up! Starting main'
	  'container."'
```

- You’re defining an init container, not a regular container. The init container runs a loop that runs until the fortune Service is up.

### Post-start hook
- Initiates after the app has started.
- Runs in parallel with the main process.
- Until the hook completes, the container will be in `Waiting` state with reason `ContainerCreating`. The pod status will be `Pending` instead of `Running`.
- If the hook fails or returns a non-zero exit code, the main container is killed.

### Pre-stop hook
- Executes immediately before a container is terminated.
- Can be used to initiate a graceful shut down.

```yaml
 lifecycle:
   preStop:
     httpGet:
        port: 8080
        path: shutdown
```

- This is a pre-stop hook that performs an HTTP GET request. The request is sent to `http://POD_IP:8080/shutdown`.
- Lifecycle hooks target containers, not pods.
- Your app should react to a `SIGTERM` signal.

### Shell form vs direct execution
- If the SIGTERM signal isn't hitting your app it could be because your Dockerfile is set to use the shell form (`ENTRYPOINT /mybinary`) instead of executing it directly (`ENTRYPOINT ["/mybinary"]`).
- The problem is that in shell form, it creates a shell in the container _then_ runs your app in the shell as a "child process".
- This means that if a signal hits the shell, it'll have to forward it to the app.

### Ensuring all client requests are handled properly
- Have readiness probes so that k8s knows that the pod is ready to accept connections.
- Wait for a few seconds, then stop accepting new connections.  
- Close all keep-alive connections not in the middle of a request. 
- Wait for all active requests to finish. 
- Then shut down completely.
- Maybe adding a pre-stop hook to sleep for 5 seconds.

### Making the apps easy to run and manage
- Don't use `latest` tag for containers.
- Make small and manageable containers.
- Use more than one label for your resources.
- Add annotations to describe your resources.
- In microservice architecture, pods could contain lists of names the other services the pod is using.
- Use a `terminationMessagePath` field in the container definition in the pod spec.
	- It will show up in a `k describe pod` to see why a pod terminated.
	- You can see the reason why the container died without having to inspect its logs.
- You can use `k logs` with `--previous` to see the logs of the previous container.
- You can copy logs to your local using `k cp foo-pod:/var/log/foo.log foo.log`

### Best practices for development and testing
- You can connect to a backend service using those environment variables to find the service. If you need it on the outside, you can use a `NodePort` to connect from outside -> inside.
- Run `kubectl proxy` on your local machine, run your app locally, and it should be ready to talk to your local `kubectl proxy`.

### Using Minikube
- Use `minikube mount` to mount your local filesystem into the minikube vm.
- You can copy local docker images to the minikube vm.

```bash
$ docker save <image> | (eval $(minikube docker-env) && docker load)
```

- Just make sure the `imagePullPolicy` in your pod spec isn't set to `Always`.

## Chapter 18: Extending Kubernetes

### CustomResourceDefinitions
- You create this definition describing your custom resource.
- Then you can start creating that resource.

```ad-quote
_Instead of dealing with Deployments, Services, ConfigMaps, and the like, you’ll create and manage objects that represent whole applications or software services. A custom controller will observe those high-level objects and create low-level objects based on them._
```


### Example `Website` CRD
- We want the `Website` resource to create a service and a pod with our app.

```yaml
kind: Website
metadata:
  name: kubia
spec:
  gitRepo: https://github.com/luksa/kubia-website-example.git
```

- We also need to create the crd itself

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: websites.extensions.example.com // long to prevent name clash
spec:
  scope: Namespaced
group: extensions.example.com
version: v1
names:
  kind: Website
  singular: website  
  plural: websites
```

- Website controller talks to the API server through a proxy (in an ambassador container).
- The proper way to watch objects through the API server is to not only watch them, but also periodically re-list all objects in case any watch events were missed.

### Running the controller as a pod
- You can run it locally and use `kubectl proxy` as an ambassador to the api server.
- Once its ready to deploy to prod, make it a deployment!

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: website-controller
spec:
  replicas: 1   // just one replica
  template:
metadata:
  name: website-controller
  labels:
    app: website-controller
spec:
  serviceAccountName: website-controller   // special service account
  containers:
  - name: main
    image: luksa/website-controller
  - name: proxy
    image: luksa/kubectl-proxy:1.6.2   // proxy sidecar
```

- One container runs your controller, whereas the other one is the ambassador container used for simpler communication with the API server.


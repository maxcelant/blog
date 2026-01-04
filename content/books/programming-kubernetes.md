---
title: Programming Kubernetes
---
#### 1. Introduction
- Watch events and the Kubernetes Event resource are two different things: 
	- watch events are HTTPS events from the API Server to controllers.
	- Event resource is a resource like Pods with a TTL.
- Controllers are edge-driven but level-triggered. The event is fired based on an event (create, update, etc). The event itself is just a "blip", it doesn't care about the previous state. It will read the current state.
- Optimistic concurrency relies on the resource version of an object implanted by etcd in order to know if a resource was modified concurrently by another actor.

#### 2. Kubernetes API Basics
- Kinds are a 1:1 mapping with the Go type. Singular and capital.
- Versions are different representations of the same object. The API server does lossless conversions.
- Resource are lowercase and plural i.e. `pods`.
- Resources get have *subresources* to do specific things i.e. */pod/nginx/logs*.
- GVR identifies the HTTP path.
- The process of mapping a GVR to a GVK is called REST mapping.
- A request follows a chain of processes in the API server.
	1. Attaching filters to the request.
	2. Authn and Authz.
	3. Mutating admission (defaulting per resource and also any custom webhooks you have).
	4. Schema validation.
	5. Validating admission (default plus any validation webhooks you have).
	6. Persist in etcd.

![[Pasted image 20251129112830.png]]

#### 3. Basics of client-go
- *client-go* is a web service client library that supports CRUD + WATCH mechanism.
- *apimachinery* holds all the generic building block to implement  a kubernetes-like API.
- It's called a `clientset` because it contains multiple clients for all native K8s resources.
- In essence `runtime.Object` means a Kubernetes object in Go is a data structure that can:
	- Return and set the GVK.
	- Be deep-copied (cloned without any memory sharing to the original).
- Kubernetes objects embed the `metav1.TypeMeta` and this allows the `runtime.Object` to be implemented.
- Storing the version (and kind) inside the struct makes the data _harder_ to work with because it couples business logic to wire-format details.
- To fill the `TypeMeta` it involves the concept of a scheme.
- Most top level objects embed a `ObjectMeta` field which becomes the `metadata` section.
- etcd automatically keeps track of revision numbers, it'll return it to the api server and that's what the api server uses as it's `resourceVersion`.
- The client set main interface (called `Interface`) has methods to get all of the different resources.

```go
type Interface interface { 
	Discovery() discovery.DiscoveryInterface 
	AppsV1() appsv1.AppsV1Interface 
	AppsV1beta1() appsv1beta1.AppsV1beta1Interface 
	AppsV1beta2() appsv1beta2.AppsV1beta2Interface
	...
}

type AppsV1Interface interface {
	DeploymentsGetter
	StatefulSetsGetter
}

type DeploymentsGetter interface {
    Deployments(namespace string) DeploymentInterface
}
```

- Each of these methods returns an interface that provides getters for those resources. Using the builder pattern, it would look like this:

```go
AppsV1().Deployments("default")...
```

##### Informers
- The `watch` mechanism is also an interface, but it's usually digested through the *Informer*.
- Informers also have event handles for add/update/delete which the client needs to implement.
	- *controller-runtime* uses this to create `NamespacedName` objects and add them to the queue. 
- Shared informer factories allow multiple controllers to use the same informer so that you don't have unnecessary duplicate watches. 

##### Work Queue
- A priority queue in *client-go*, used to build controllers.
- Can add any arbitrary item and `Get()` will return the highest priority item and once processed you need to call `Done(item)`.
- There are also extensions of the base work queue to add delaying (`AddAfter`) and rate limiting (`AddRateLimited`).

##### API Machinery in Depth
- Each *GVK* corresponds to one Go type, however, a Go type can belong to multiple GVKs.
- Each *GVR* corresponds to one HTTP (base) path.
	- `GET /apps/v1/namespaces/{ns}/deployments`
- *REST Mapping* maps a GVK to a GVR.
- `RestMapper` enables us to request the GVR for a GVK.
- `DeferredDiscoveryRESTMapper` uses discovery info from the K8s API to dynamically build up the REST mapping.
- *Scheme* connects the world of Go with the implementation-independent world of GVKs.
- Allows us to map Go types to possible GVKs.
- This means that when an incoming YAML/JSON comes in with `apiVersion: dummy/v1`, it knows to map it to `Foo{}`.

```go
scheme.AddKnownTypes(schema.GroupVersion{Group: "dummy", Version: "v1"},
    &Foo{},
)
```
- The scheme also stores a list of conversion functions and defaulters.

![[Pasted image 20251130115109.png]]

#### 4. Custom Resources
- `kubectl` uses discovery information from the API server to find out about the new resources.
- When `kubectl` sees a resource it doesn't understand:
	1. It will hit the endpoint to get all of the APIs that are registered to the API server via `/apis`. This returns the `group` and `version` of each api group.
	2. `kubectl` asks the API server about all existing resources in each group via `/apis/<groupVersion>` discovery endpoint.
	3. It scans all of the `Kinds` associated with those `GroupVersion` until it finds or doesn't find the resource that the user wants. 
	4. All future calls will be cached locally in `~/.kubectl` so it can use them on subsequent requests.
- CRDs are resources in themselves, and have statuses as well.
- CRs are validated by the API server during creation and updates via OpenAPI v3 schema.
- More complex validations are done by validating webhooks.

##### Subresources
- Subresource endppints use a different protocol than the main resource endpoint. These include things like `/api/v1/namespace/{ns}/pods/name/logs` or `/portforward`.
- `/status` endpoint allows us to split the privileges from the user defined spec and the controller defined status.
- Since they are different endpoints we can have RBAC specific to each one.
- You cannot modify the status from the main endpoint.
- If you hit the `/status` endpoint with a payload, the API server only cares about the `status` object.
- The `generation` field is only affected by the `spec` being changed.
	- In contrast, `resourceVersion` changes on _any update_ and mostly used for optimistic concurrency reasons.
	- `generation` is used to see if the object has changed.
- Optimistic concurrency semantics still apply here for spec and status, because both update the `resourceVersion`.

##### Developer's View on Custom Resources
- There are three main types of clients:
	1. Unstructured dynamic clients. Used by operators like the garbage collector when you have unknown structured objects. You basically get nested dictionaries in return.
	2. Client sets. These are specific clients for specific registered resources. Like what we saw about with deployments.
	3. Generic clients. Allow you to work with resources that are registered to the scheme. This is what `controller-runtime` gives us. More on this below.
- `controller-runtime` offers a generic client to get an API object as opposed to creating a clientset for a specific resource.

```go
podList := &corev1.PodList{}
err := cl.List(context.TODO(), client.InNamespace("default"), podList)
```

- The client accepts any `runtime.Object` and uses the scheme to find the associated GVK for `corev1.PodList`. In the second step. Next the `List()` method uses discovery info to find the GVR to access `/api/v1/namesapce/default/pods`.

#### 5. Automating Code Generation
- Go doesn't support metaprogramming and (didn't) support generics at the time so it relied heavily on code generation.
- Generators are used for:
	- Creating the deep copy funcs
	- Creating typed client sets
	- Creating informers for CRs
	- Creating listers for CRs
- Tags are important. There are global and local tags.
- Global are located above the `package` lne in `doc.go`.
- Local are above type declarations.
- `//+k8s:deepcopy-gen=package` tells `deepcopy-gen` to create a deep copy for every type in the package.

>[!question] Why do we need to deep copy?
>We don't want to mutate shared objects. Kubernetes shares a lot of objects (shared informers for instance), we don't want consumers modifying the same object. So we make deep copies and work with those.

#### 6. Solutions for Writing Operators
- You can use the *client-go* package directly to make a controller.
- `syncHandler` is basically the reconcile function.
- It also uses a clienset of your CRD as opposed to the generic client.

```go
if when, err := c.syncHandler(key); err != nil {
	c.workqueue.AddRateLimited(key)
	return fmt.Errorf("error syncing '%s': %s, requeuing", key, err.Error())
} else if when != time.Duration(0) {
	c.workqueue.AddAfter(key, when)
} else {
	// Finally, if no error occurs we Forget this item so it does not
	// get queued again until another change happens.
	c.workqueue.Forget(obj)
}
```

#### 8. Custom API Servers
- A custom API supports:
	- Any storage medium (not just etcd).
	- Can write any custom subresources (not just `/status` and `/scale`).
	- Can implement graceful deletions.
	- Can implement all operations like validation, admission and conversion in most efficient way.

##### Architecture: Aggregation
- Built using generic API server library *k8s.io\/apiserver*.
- API groups served by a custom API server are proxied by the `kube-apiserver` process to the custom API server process.
- So the `kube-apiserver` knows about all the custom API servers and groups they serve.
- The component doing this is called the `kube-aggregator`.
- This proxying is called *API Aggregation*.
- The request follows this path:
	1. Request received by API server.
	2. Passes the handler chain (authn, authz, audit logging, etc)
	3. API server knows the aggreagated APIs, intercepts `/api/aggregated-API-group-name`
	4. Forwards request to custom API server.

![[Pasted image 20251205111620.png]]

- Aggregator uses a `APIService` resource to discovery custom API servers. The resource just has the groups and versions, not resources or more details.
- There's this idea of a `GroupPriorityMinimum` which is basically the priority of that group when you have conflicting group names by the API server.
	- The lower the number, the higher the priority.
	- You could potentially use this to override native kubernetes objects with custom implementations if you make this number lower than the native group.

##### Inner Structure of a Custom API Server
- Has the same basic internal structure as the Kubernetes API server except no `kube-aggregator`.
- Has its own handler chain.
- Has its own resource handler pipeline for decoding, conversion, admission, admission, REST mapping, and encoding.

##### Delegated Authorization
- Authz is based on username and group list.
- RBAC maps identities to roles, and roles to authz rules, which accept or reject requests.
- A custom API server has it's own authn and authz because it is not certain that they were proxied by the `kube-apiserver`, although they should be.
- The custom API server will send a `SubjectAccessRequest` to the `kube-apiserver` (if it isn't cached), and the `kube-apiserver` will respond in the status field whether to allow or deny.

##### Writing Custom API Server
- At a high level, we start with config options which set up that initial handler chain. We can hit basic endpoints (like `/healthz` and `/logs`), but since we have no `/apis`, there is nothing to serve in terms of resources (yet).

![[Pasted image 20251205115053.png]]

##### Conversions
- All types have an `internal` version which all types can be converted to. It does this so that it can work on the objects in a unified way.
	- Note: This is _not_ the same thing as the stored version.
- Conversions must be roundtrippable, you should be able to convert from one version to another and back with out dataloss.
- Conversions are fast because they actually do share data structures between versions. In order to avoid bugs, you should create a deep copy before working with the object. 

```go
func autoConvert_restaurant_PizzaSpec_To_v1beta1_PizzaSpec(
  in *restaurant.PizzaSpec,
  out *PizzaSpec,
  s conversion.Scope,
) error {
  out.Toppings = *(*[]PizzaTopping(unsafe.Pointer(&in.Toppings))  
  return nil 
}
```

- This just reiterates the slice header as a `PizzaTopping` and points `out.Toppings` to it! Shared state!

##### Defaulting
- Similarly to conversions, defaulting functions need to be registered into the scheme for custom types.
- Pointers types are used for values so that the defaulter knows when a value was set or not. If it wasn't set, it'll default to `nil`.
##### Validation
- Happens after deserialization, defaulting and converted to internal type.
- Validation only needs to occur once for the internal version, not for each external version.
##### Codecs
- When a request comes in, the `CodecFactory` has the job of decoding the bytes (JSON, YAML, Protobuf) into a `runtime.Object` of the internal type so that the other parts of the system can work with it.
- Tightly coupled with the `Scheme`. The `Scheme` keeps a register of the different GVKs and how they relate to each other, while the `CodecFactory` is the implementation of how those objects are serialized and deserialized.

##### Generic Registry
- The _generic registry_ supplies the interface for creating a REST registry for a GVR.
- Includes things like `Getter`, `Updater`, `Watcher`, which need to be implemented for the registry of that resource if they want to be able to perform those actions.
- Registries can be customized with _strategies_. There are strategies of each CRUD type`RESTCreateStrategy`. They are just interfaces and you need to implement the methods of the interface. [ref](https://github.com/programming-kubernetes/pizza-apiserver/blob/master/vendor/github.com/programming-kubernetes/pizza-apiserver/pkg/registry/restaurant/pizza/strategy.go#L36-L38)
- The `genericregistry.Store` object is how you create the API for a given GVR. [ref](https://github.com/programming-kubernetes/pizza-apiserver/blob/master/vendor/github.com/programming-kubernetes/pizza-apiserver/pkg/registry/restaurant/pizza/etcd.go#L31-L44)
- We embed the `genericregistry.Store` in the `registry.REST` type, because we want to use this as our extension point as opposed to messing with the `Store` itself. Eventually this will become the interface `StandardStorage`

```go
type StandardStorage interface {
	Getter
	Lister
	CreaterUpdater
	GracefulDeleter
	CollectionDeleter
	Watcher

	// Destroy cleans up its resources on shutdown.
	// Destroy has to be implemented in thread-safe way and be prepared
	// for being called more than once.
	Destroy()
}
```

- We store the versions in the `apiGroup.VersionedResourcesStorageMap` so that now we have an idea of which resources are in which group-version.

```go
	v1alpha1storage := map[string]rest.Storage{}
	v1alpha1storage["pizzas"] = customregistry.RESTInPeace(
		pizzastorage.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter)
	)
	v1alpha1storage["toppings"] = customregistry.RESTInPeace(
		toppingstorage.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter)
	)
	apiGroupInfo.VersionedResourcesStorageMap["v1alpha1"] = v1alpha1storage

```

##### Running it
- By doing `kubectl get pods --v=7` we can actually see which requests are made to the server. 
- `ls foo* | xargs -n 1 kubectl apply -f` will take each file and apply it one at a time.
- ==Go back and do this my self!!==

##### Certs and Trust
- A certificate authority bundle (`caBundle`)  contains one or more X.509 certs that have:
	- Subject Alternate Name (SAN)
	- Public key
	- Signature signed by CAs private key.
- The client uses the public key to validate the signature and ensure it's legit and want tampered with.
- The CN (common name) in the cert must match the DNS name of the custom api server otherwise the proxying won't work.
- ==Practice this with openssl!!==

##### Sharing etcd
- Using etcd operator you can try run it in-cluster.
- If you plan on using the same etcd cluster as the main API server for your custom one, you can add an etcd prefix which will be added to all resources. This will help distinguish which resources come from your API server vs the default one.
- Alternatively, you can use etcd proxies, which will automatically check for key collisions.

#### Chapter 9: Advanced Custom Resources
- Conversion webhook flow:
	1. Client requests a version of an object. 
	2. etcd stores that object with some version. 
	3. If the versions don't match, the storage object is sent to the conversion webhook and returned with the correct version. 
	4. Converted object is sent back to the client.
- CRDs do **not** have an internal type. They are directly converted between types. 
- It only occurs to and from etcd.
- The `ConversionRequest` payload sent to the webhook includes a slice of `runtime.RawExtension` which literally hold the raw object as a slice of bytes.
- The webhook will use the `codec` to decode the raw bytes into a `runtime.Object` then it'll call `convert` to transform that object to the correct GVK.

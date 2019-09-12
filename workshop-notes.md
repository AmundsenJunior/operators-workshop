# Operators Workshop

*July 17, 2019*  
*Hooker Colt Brewery, Hartford, CT*  
*Matt Dorn*  
*RedHat Senior Software Engineer*  

### Book recommendations:
* *[Site Reliability Engineering](https://landing.google.com/sre/books/)* 
* *[Programming Kubernetes](https://programming-kubernetes.info/)*
* *[Cloud Native Infrastructure](https://www.cnibook.info/)*

### Workshop resources

*The [Operator Framework](https://github.com/operator-framework)*, comprised of:
* [operator-sdk (Golang)](https://github.com/operator-framework/operator-sdk)
* [ansible-operator module](https://docs.openshift.com/container-platform/4.1/applications/operator_sdk/osdk-ansible.html)
* [helm app operator](https://docs.openshift.com/container-platform/4.1/applications/operator_sdk/osdk-helm.html)
* [operator-lifecycle-manager](https://github.com/operator-framework/operator-lifecycle-manager)
* [operator-metering](https://github.com/operator-framework/operator-metering)

[Workshop online exercises](https://learn.openshift.com/operatorframework/)

## Deep dive into K8s API

### client-go

[k8s.io/client-go](https://github.com/kubernetes/client-go) is both the public Go package for developing applications
(e.g., operators) that work with the Kubernetes API and the internal package leveraged by Kubernetes tools (e.g.,
`kubectl`) to talk to the API. Composed of:
* Clients
  * Clientset
  * Discovery
  * RESTclient
* Utilities for writing controllers - how to efficiently watch the cluster via API
  * Workqueue
  * Informers/Shared Informers

[k8s.io/api](https://github.com/kubernetes/api)
* base package of K8s API

[k8s.io/apimachinery](https://github.com/kubernetes/apimachinery)
* scheme, typing, encoding, decoding, conversion packages
* can build a K8s-like API that has nothing to do with K8s using this package

_(Assume that I've aliased my `kubectl` command in a terminal in the below commands: `$alias kc=kubectl`)_

*Watch* is an open GET request against the API waiting for Add/Update/Delete events against Pod resources:
```sh
$ kc get pods -w
```

### Some K8s Concepts

#### Authenticating for API access

* Clients outside the cluster use *kubeconfig* (e.g., `~/.kube/config`) to talk to API
* Controllers inside the cluster use *ServiceAccounts* to talk to API
  * a pod, e.g., fetches token of its service account from a mounted volume
  * uses an env var injected in every pod to know the address of the API

#### Resource schema components

* GVK (TypeMeta) - group, version, kind
* Metadata (ObjectMeta) - name, namespace, labels
* Spec - what you want
* Status - read-only as a user, but is written by controller (which retrieves from API)

#### Labels

Labels are essential to operating:
* key/value pairs assigned and viewable
* operators - and controllers on resources generally - use labels to manage its resources

### K8s API Fundamentals

Get list of all resource/controller API versions in the cluster:
```sh
$ kc api-versions
```

Get pods with high verbosity to show K8s API requests/responses:
```sh
$ kc get pods --v=8
```

Open proxy to k8s API:
```sh
$ kc proxy --port=8001
```

### K8s API CRUD operations

Hit base of API:
```sh
$ curl -X GET http://localhost:8001
```

Get full OpenAPI API definition:
```sh
$ curl localhost:8001/openapi/v2
```

Get all pods:
```sh
$ curl localhost:8001/api/v1/pods
```

Get names of all pods:
```sh
$ curl localhost:8001/api/v1/pods | jq .items[].metadata.name
```

Get pods of a particular namespace:
```sh
$ curl localhost:8001/api/v1/namespaces/namespaceName/pods
```

Get one pod's details in a namespace:
```sh
$ curl localhost:8001/api/v1/namespaces/namespaceName/pods/podName
```

With a manifest from a pod (`--export` flag strips out extraneous read-only cluster information),
```sh
$ kc get pods podName --export -o json > podmanifest.json
```

Update the value of the image...
```sh
$ sed -i 's|nginx:1.13-alpine|nginx:1.14-alpine|g' podmanifest.json
```

And update the pod manifest on the cluster.
```sh
$ curl -X PUT localhost:8001/api/v1/namespaces/namespaceName/pods/podName \
  -H "Content-type: application/json" \
  -d @podmanifest.json
```

Alternatively, patch the pod with a new container image:
```sh
$ curl -X PATCH http://localhost:8001/api/v1/namespaces/namespaceName/pods/podName \
  -H "Content-type: application/strategic-merge-patch+json" \
  -d '{"spec":{"containers":[{"name": "server","image":"nginx:1.15-alpine"}]}}'
```

Finally, delete the pod:
```sh
$ curl -X DELETE localhost:8001/api/v1/namespaces/namespaceName/pods/podName
```

### K8s ReplicaSets

Get pods by label:
```sh
$ kc get pods -l app=myfirstapp --show-labels
```

Imperatively scale the replicaset:
```sh
$ kc scale replicaset myfirstreplicaset --replicas=6
```

Get the scaling of a replicaset via API:
```sh
$ curl localhost:8001/apis/extensions/v1beta1/namespaces/namespaceName/replicasets/myfirstreplicaset/scale
```

Scale a replicaset to 5:
```sh
$ curl  -X PUT localhost:8001/apis/extensions/v1beta1/namespaces/myproject/replicasets/myfirstreplicaset/scale \
  -H "Content-type: application/json" \
  -d '{"kind":"Scale","apiVersion":"extensions/v1beta1","metadata":{"name":"myfirstreplicaset","namespace":"myproject"},"spec":{"replicas":5}}'
```

Get replicaset status:
```sh
$ curl -X GET http://localhost:8001/apis/extensions/v1beta1/namespaces/myproject/replicasets/myfirstreplicaset/status
```
The /status endpoint is also used to allow a controller to PUT a desired status of replicaset.

### ReplicaSets and Deployments

_the underrated, OG operators_

Examine these to learn how operators work generally.

A Primary Resource in an operator is the CRD (usually), and that CRD creates Secondary Resources (pods, services, etc):
* ReplicaSets are a primary, with Pods as secondary
* Deployments are a primary, with ReplicaSets as secondary

Alot of K8s, including *replicaset*, implements the reconciler pattern (act to make actual state match desired state)
from Cloud Native Infrastructure.

*OwnerReferences* link secondary to primary resources. If a primary is deleted, the secondaries will also be deleted.

*Finalizers* are metadata values on a primary resource that exist on the manifest until all the cleanup operations by
the controller are completed, which then deletes the finalizer value.

Always question what is needed if it can be addressed by existing resource/controllers in k8s before writing custom
logic in your operator to manage your CRs.

## Operators

An Operator is a combination of K8s custom resource and custom controller and human operational knowledge:
* Custom Resource Definition (CRD) - pod, configmap, ingress (e.g.)
* Controller Pattern (declarative) - replicaset, daemonset, deployment (e.g.)
* Human Operational Knowledge - domain- or application-specific SRE/DevOps experience in running a system, such as:
  * installing
  * scaling properly - how to specifically, correctly roll out (e.g.)
  * self-healing
  * cleaning up
  * updates
  * backup
  * restore

### Custom Resource Definitions (CRDs)

A CRD extends the API for providing the means to make requests.

Let's proxy into the cluster and get a list of resources:
```sh
$ kc proxy
$ curl localhost:8001
$ curl localhost:8001/api/v1 | jq .resources[].name
```

Create a CRD (requires a Cluster Admin role to extend the API):  
`manifest.yaml`
```yaml
  kind: CustomResourceDefinition
  spec:
    group: app.company.com
    scope: Namespaced # or Cluster
  metadata:
    name: crdname
```

Create the CRD:
```sh
$ kc create f manifest.yaml
```

Verify that the CRDS is created successfully in the cluster:
```sh
$ kc get crd
```
This will return nothing if successful, or will error if the CRD is not an available resource in the API.

Get the man-like reference for a resource type:
```sh
$ kc explain crd
```

### Controllers

* A CRD needs a controller to act upon its presence. The Controller does the work of creating the resources via human
  knowledge operationalized by code (go, ansible, helm).
* Controllers see desired state and compare to current state, via declarative manifests, and take action to meet that
  desired state.
  * E.g., the Informer pattern
    1. reflector - get pods, then watch get pods
    1. localstore of all objects in cluster
    1. controller worker/business logic to then act on events that exist in the localstore
* Controllers work either inside or outside the cluster.
* The controller is deployed on the cluster as pods (in a Deployment, e.g.), where the code in a Docker image is
  watching the API for any CRD events that it can act upon.

To see API/Controllers activity:
```sh
$ kc get events
```

## Operator Framework
* Operator SDK - Go, Ansible, Helm (with a Python SDK in development)
* Operator Lifecycle Manager - manage operators on any K8s cluster
* Operator Metering - track usage in cluster of operators

[OperatorHub.io](https://operatorhub.io) - app store for public operators

### Creating the etcd operator

1. Create a CRD that defines the EtcdCluster kind
1. Create a ServiceAccount for the operator Deployment to have access
1. Create an RBAC role that defines access for the operations the Operator would do
1. Bind the RBAC role to the ServiceAccount
1. Create the Deployment of the Operator
1. Once the operator deployment is up, create a CR of the operator (a running etcd cluster)
1. The operator creates an instance of the CRD (the "Operand"), the pods of the CR, and the services

### Operator-SDK

*operator-sdk* builds upon *[controller-runtime](https://github.com/kubernetes-sigs/controller-runtime)*, which is built
upon *client-go*.

[Install the *operator-sdk* package and CLI](https://github.com/operator-framework/operator-sdk/blob/master/doc/user/install-operator-sdk.md):  
```sh
$ brew install operator-sdk
```

Set up your operator development environment, with Go code project scaffolding and CRD/CR manifests:
```sh
$ cd GOPATH/src/github.com/githubusername
$ export GO111MODULE=on
$ operator-sdk new app-operator
$ operator-sdk add api --api-version app.company.com/v1alpha1 --kind App
```

`pkg/api/app/app_types.go`
* `type AppSpec` - struct of what the spec of the CR is to be
* `type AppStatus` - struct of 

Add a controller to the operator definition:
```sh
$ operator-sdk add controller --api-version app.company.com/v1alpha1 --kind App
```
`pkg/controller/app/app_controller.go` contains the controller operations, including the reconciler.

Code generation that updates package files from changes:
```sh
$ operator-sdk generate k8s
```

Put validation information into CRD against `*_types.go` so that validation can be made at the API call instead of by
the controller after a change is persisted into etcd:
```sh
$ operator-sdk generate openapi
```

Run `go build` and install the operator locally (requires that CRD is already present in cluster):
```sh
$ operator-sdk up local --kubeconfig ~/.kube/config --namespace namespaceName
```

## TODOs
* Examine in-depth the structure of the operator code
* Create a wholly new operator as example of using `operator-sdk` and/or the Helm operator
* Create a demo/presentation process to walk through these notes and the example operators

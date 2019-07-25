Operators Workshop
Hooker Colt Brewery, Hartford, CT
July 17, 2019

Matt Dorn, RedHat Senior Software Engineer

Book recommendations: sre, programming kubernetes, cloud native infrastructure

Operator Framework (https://github.com/operator-framework), comprised of
  operator-sdk (Golang)
  ansible-operator
  helm app operator

Deep dive into K8s API

workshop.coreostrain.me - resources
  operator/coreostrainme

https://learn.openshift.com/operatorframework/ - online exercises


client-go
  clients
    clientset
    discovery
    restclient
  utilities for writing controllers - how to efficiently watch the cluster via API
    workqueue
    informers/shared informers

k8s.io/api (github.com/kubernetes/api)
  base package of k8s API
k8s.io/api-machinery
  scheme, typing, encoding, decoding, conversion packages
  can build a k8s-like API that has nothing to do with k8s using this package

kc get pods -w
  watch is an open GET request against the API waiting for Add/Update/Delete events against Pod resources

clients outside the cluster use kubeconfig to talk to API
controllers inside the cluster use ServiceAccounts to talk to API
  a pod, e.g., fetches token of its service account from a mounted volume
  uses and env var injected in every pod to know the address of the API

in a controller
  informer
    reflector - get pods, then watch get pods
    localstore of all objects in cluster
    controller worker/business logic to then act on events that exist in the localstore

resource schema components
  GVK (TypeMeta) - group, version, kind
  Metadata (ObjectMeta) - name, namespace, labels
  Spec - what you want
  Status - read-only as a user, but is written by controller (which retrieves from API)

labels are essential to operating
  key/value pairs assigned and viewable
  operators - and controllers on resources generally - use labels to manage its resources



K8s API Fundamentals


kc api-versions
  get list of all resource/controller api versions in the cluster

kc get pods --v=8
  get pods with high verbosity to show k8s api requests/responses

kc proxy --port=8001
  open proxy to k8s API


K8s API CRUD operations


curl -X GET http://localhost:8001
  hit base of API

curl localhost:8001/openapi/v2
  get full openapi API definition

curl localhost:8001/api/v1/pods
  get all pods

curl localhost:8001/api/v1/pods | jq .items[].metadata.name
  get names of all pods

curl localhost:8001/api/v1/namespaces/namespaceName/pods
  get pods of a particular namespace

curl localhost:8001/api/v1/namespaces/namespaceName/pods/podName
  get one pod's details in a namespace

kc get pods podName --export -o json > podmanifest.json
  with a manifest from a pod (export flag strips out extraneous info)
sed -i 's|nginx:1.13-alpine|nginx:1.14-alpine|g' podmanifest.json
  update the value of the image
curl -X PUT localhost:8001/api/v1/namespaces/namespaceName/pods/podName -H "Content-type: application/json" -d @podmanifest.json
  and update the pod manifest on the cluster
curl -X PATCH http://localhost:8001/api/v1/namespaces/namespaceName/pods/podName -H "Content-type: application/strategic-merge-patch+json" -d '{"spec":{"containers":[{"name": "server","image":"nginx:1.15-alpine"}]}}'
  patch the pod with a new container image
curl -X DELETE localhost:8001/api/v1/namespaces/namespaceName/pods/podName
  then delete the pod


K8s ReplicaSets

kc get pods -l app=myfirstapp --show-labels
  get pods by label

kc scale replicaset myfirstreplicaset --replicas=6
  imperative scale the replicaset

curl localhost:8001/apis/extensions/v1beta1/namespaces/namespaceName/replicasets/myfirstreplicaset/scale
  get the scaling of a replicaset via API

curl  -X PUT localhost:8001/apis/extensions/v1beta1/namespaces/myproject/replicasets/myfirstreplicaset/scale -H "Content-type: application/json" -d '{"kind":"Scale","apiVersion":"extensions/v1beta1","metadata":{"name":"myfirstreplicaset","namespace":"myproject"},"spec":{"replicas":5}}'
  scale a replicaset to 5

curl -X GET http://localhost:8001/apis/extensions/v1beta1/namespaces/myproject/replicasets/myfirstreplicaset/status
  get replicaset status

/status endpoint is used to allow a controller to PUT a desired status of replicaset


ReplicaSets and Deployments (the underrated, OG operators)
  study to learn how operators work generally

Primary Resource in an operator is the CRD (usually), and that CRD creates Secondary Resources (pods, services, etc)
  ReplicaSets are a primary, with Pods as secondary
  Deployments are a primary, with ReplicaSets as secondary

Alot of k8s, including replicaset, implements the reconciler pattern (act to make actual state match desired state) from Cloud Native Infrastructure

OwnerReferences link secondary to primary resources. If a primary is deleted, the secondaries will also be deleted.

Finalizers - metadata values on a primary resource that exists on the manifest until all the cleanup operations by the controller are completed, which then deletes the finalizer value


Always question what is needed if it can be addressed by existing resource/controllers in k8s before writing custom logic in your operator to manage your CRs.



Operators

Operator is a combination of K8s custom resource and custom controller and human operational knowledge

  Custom Resource Definition (CRD) - pod, configmap, ingress (e.g.)
  Controller Pattern (declarative) - replicaset, daemonset, deployment (e.g.)
  Human Operational Knowledge - domain- or application-specific
    installing
    scaling properly - how to specifically, correctly roll out (e.g.)
    self-healing
    cleaning up
    updates
    backup
    restore

$ kubectl proxy
$ curl localhost:8001 # to hit the K8s API
$ curl localhost:8001/api/v1 | jq .resources[].name # list of resources

Create a CRD (requires a Cluster Admin role to extend the API)
manifest.yaml
  kind: CustomResourceDefinition
  spec:
    group: app.company.com
    scope: Namespaced # or Cluster
  metadata:
    name: crdname

$ kc create f manifest.yaml (create CRD)
$ kc get crd (verify CRD is created)

$ kc explain # get man for a resource type

A CRD extends the API for providing the means to make requests.
A CRD needs a controller to act upon its presence. The Controller does the work of creating the resources via human knowledge operationalized by code (go, ansible, helm).

Controllers see desired state and compare to current state, via declarative manifests, and take action to meet desired state.

Controllers work either inside or outside the cluster.

$ kc get events # to see API/Controllers activity

The controller is deployed on the cluster as pods (in a Deployment), where the code in a Docker image is watching the API for any CRD events that it can act upon.

Operator Framework
  Operator SDK - go, ansible, helm
  Operator Lifecycle Manager - manage operators on any K8s cluster
  Operator Metering - track usage in cluster of operators

OperatorHub.io - app store for public operators

Creating the etcd operator

Create a CRD that defines the EtcdCluster kind
Create a ServiceAccount for the operator Deployment to have access
Create an RBAC role that defines access for the operations the Operator would do
Bind the RBAC role to the ServiceAccount
Create the Deployment of the Operator
Once the operator deployment is up, create a CR of the operator (a running etcd cluster)
The operator creates an instance of the CRD (the "Operand"), the pods of the CR, and the services



Operator-SDK

Operator-SDK builds upon Controller-Runtime which is built upon client-go

$ cd GOPATH/src/github.com/amundsenjunior
$ export GO111MODULE=on # enable go modules
$ operator-sdk new app-operator # creates the scaffolding for an operator
$ operator-sdk add api --api-version app.company.com/v1alpha1 --kind App # creates manifests of CRD and CR

pkg/api/app/app_types.go
  type AppSpec - struct of what the spec of the CR to be
  type AppStatus

$ operator-sdk add controller --api-version app.company.com/v1alpha1 --kind App

pkg/controller/app/app_controller.go
  contains the controller operations, including reconciler

$ operator-sdk generate k8s
  code generation that updates package files from changes

$ operator-sdk generate openapi
  puts validation information into CRD against `*_types.go` so that validation can be made at the API call instead of by the controller after a change is persisted into etcd

$ operator-sdk up local --kubeconfig /Users/scott.russell/.kube/config --namespace namespaceName
  runs go build and installs operator locally
  requires that CRD is already present in cluster

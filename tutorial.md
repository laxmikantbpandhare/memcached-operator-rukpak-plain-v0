# Oria Operator Tutorial
### An in-depth walkthrough of building and running a oria-operator with memcached-operator.

## Prerequisites

- [Operator SDK](https://sdk.operatorframework.io/docs/installation/) v1.8.0 or newer
- User authorized with `cluster-admin` permissions.
- [GNU Make](https://www.gnu.org/software/make/)

## Overview

We will create a sample project to let you know how it works and this sample will:

- Create a Memcached-Operator Deployment using this [tutorial](https://sdk.operatorframework.io/docs/building-operators/golang/tutorial/)
- Generate the manifests using `config/default`
- Run the Operator and after success, delete the `ClusterRole` and `ClusterRoleBinding`. Now, the `memcached-operator` will start complaining about it.
- Create similar ClusterRole template in ScopeTemplate and assciate it with ScopeInstance
- Run the `oria-operator`, it will create `ClusterRole` and `ClusterRoleBinding`. This will make sure that `memcached-operator` will not complain about RBAC's. 

### Create a new project

Use the [Operator SDK](https://sdk.operatorframework.io/docs/installation/) CLI to create a new memcached-operator from the [tutorial](https://sdk.operatorframework.io/docs/building-operators/golang/tutorial/).


```sh
mkdir -p $HOME/projects/memcached-operator
cd $HOME/projects/memcached-operator
# we'll use a domain of example.com
# so all API groups will be <group>.example.com
operator-sdk init --domain example.com --repo github.com/example/memcached-operator
```

### Create a new API and Controller 

```sh
$ operator-sdk create api --group cache --version v1alpha1 --kind Memcached --resource --controller
Writing scaffold for you to edit...
api/v1alpha1/memcached_types.go
controllers/memcached_controller.go
...
```

Define the API for the Memcached Custom Resource(CR) by modifying the Go type definitions at `api/v1alpha1/memcached_types.go` to have the following spec and status:

```
// MemcachedSpec defines the desired state of Memcached
type MemcachedSpec struct {
	//+kubebuilder:validation:Minimum=0
	// Size is the size of the memcached deployment
	Size int32 `json:"size"`
}

// MemcachedStatus defines the observed state of Memcached
type MemcachedStatus struct {
	// Nodes are the names of the memcached pods
	Nodes []string `json:"nodes"`
}
```

### Now create docker image and push to the dockerhub

Change below code in Makefile

```
-IMG ?= controller:latest
+IMG ?= $(IMAGE_TAG_BASE):$(VERSION)
```

Once done, you do not have to set IMG or any other image variable in the CLI. The following command will build and push an operator image tagged as example.com/memcached-operator:v0.0.1 to Docker Hub:

```
make docker-build docker-push
```

### Generating CRD manifests 

Make a directory and generate the manifests

```sh
$ mkdir manifest
$ kustomize build config/default > manifest/manifest.yaml
```

Need to update `manifest.yaml` file.

Update image tag from `controller:latest` to `example.com/memcached-operator:v0.0.1`

Then, remove `ClusterRole` with name `memcached-operator-manager-role` and `ClsuerRoleBinding` with name `memcached-operator-manager-rolebinding`.

Now, create the `ScopeTemplate` as shown below.

```yaml
apiVersion: operators.io.operator-framework/v1alpha1
kind: ScopeTemplate
metadata:
  name: scopetemplate-sample-manager
spec:
  clusterRoles:
  - generateName: manager-role
    rules:
    - apiGroups:
      - "*"
      resources:
      - "*"
      verbs:
      - create
      - delete
      - get
      - list
      - patch
      - update
      - watch
    - apiGroups:
      - cache.example.com
      resources:
      - memcacheds/finalizers
      verbs:
      - update
    - apiGroups:
      - cache.example.com
      resources:
      - memcacheds/status
      verbs:
      - get
      - patch
      - update
    subjects:
    - kind: ServiceAccount
      name: memcached-operator-controller-manager
      namespace: memcached-operator-system
```

Before applying `ScopeTemplate` manifests, we need to install CRD for the same from `oria-operator`.

Go to `oria-operator` and run below command.

```
$ make install
```

Then, apply manifest and look for successful execution.

```
$ kubectl apply -f manifest/manifest.yaml
namespace/memcached-operator-system created
customresourcedefinition.apiextensions.k8s.io/memcacheds.cache.example.com created
serviceaccount/memcached-operator-controller-manager created
role.rbac.authorization.k8s.io/memcached-operator-leader-election-role created
...
```

The `memcached-operator` is running successfully without anu errors. Check pod logs with below command.


```
$ kubectl get pods -n memcached-operator-system
NAME                                                     READY   STATUS    RESTARTS   AGE
memcached-operator-controller-manager-684554467c-4jflr   2/2     Running   0          9m45s

$ kubectl describe pod memcached-operator-controller-manager-684554467c-4jflr manager -n memcached-operator-system
```

Now, delete the `ClusterRole` and `ClusterRoleBinding`.

```
$ kubectl get clusterroles  
memcached-operator-manager-role              2022-10-04T23:57:11Z
memcached-operator-metrics-reader            2022-10-04T23:57:11Z
memcached-operator-proxy-role                2022-10-04T23:57:11Z

$ kubectl delete clusterrole memcached-operator-manager-role
clusterrole.rbac.authorization.k8s.io "memcached-operator-manager-role" deleted

$ kubectl get clusterrolebinding
memcached-operator-manager-rolebinding         ClusterRole/memcached-operator-manager-role                                        24m

$ kubectl delete clusterrolebinding memcached-operator-manager-rolebinding
clusterrolebinding.rbac.authorization.k8s.io "memcached-operator-manager-rolebinding" deleted
```

Then, check the pod logs again and now you will be able to see that pod is 
complaining about RBAC.



Create ScopeInstance that refers to ScopeTemplate.

```yaml
apiVersion: operators.io.operator-framework/v1alpha1
kind: ScopeInstance
metadata:
  name: scopeinstance-sample-manager
spec:
  scopeTemplateName: scopetemplate-sample-manager
```

Now, run the `oria-operator`

```
$ make run
```

Apply the ScopeTemplate and ScopeInstance.

```
$ kubectl apply -f operators_v1_scopetemplate.yaml

$ kubectl apply -f operators_v1_scopeinstance.yaml
```

The `oria-operator` will create the ClusterRole and ClusterRoleBinding. Then, the memcached-operator will pick these RBACs and now it will stop complaining.

```
1.6649296259379733e+09	INFO	Starting workers	{"controller": "memcached", "controllerGroup": "cache.example.com", "controllerKind": "Memcached", "worker count": 1}
```
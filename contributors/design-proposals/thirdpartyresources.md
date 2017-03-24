# Moving ThirdPartyResources to beta

## Background
There are a number of important issues with the alpha version of
ThirdPartyResources that we wish to address to move TPR to beta. The list is
tracked [here](https://github.com/kubernetes/features/issues/95), and also
includes feedback from existing Kubernetes ThirdPartyResource users. This
proposal covers the steps we believe are necessary to move TPR to beta and to
prevent future challenges in upgrading.


## Goals
1. Ensure ThirdPartyResource authors can version their APIs and offer their
users a safe migration path between versions (including alpha->beta for the
TPR, but also including a discussion of how a TPR author could perform a
migration for their end users)
2. Enable ThirdPartyResources to specify how they will appear in API
discovery to be consistent with other resources and avoid naming confilcts
3. Move TPR into their own API group to allow the extensions group to be
[removed](https://github.com/kubernetes/kubernetes/issues/43214)
4. Support cluster scoped TPR resources
5. Identify other features required for TPR to become beta

Non-goals
1. Solve automatic conversion of TPR between versions or automatic migration of
existing TPR

### Desired API Semantics
TPRs are intended to look like normal kube-like resources to external clients.
In order to do that effectively, they should respect the normal get, list,
watch, create, patch, update, and delete semantics.

In "normal" Kubernetes APIs, if I have a persisted resource in the same group
with the same name in v1 and v2, they are backed by the same underlying object.
A change made to one is reflected in the other. API clients, garbage collection,
namespace cleanup, version negotiation, and controller all build on this.

The convertibility of Kubernetes APIs provides a seamless interaction between
versions.  A TPR does not have the ability to convert between versions, so this
experience is broken.

Allowing a single, user specified version for a given TPR will provide this
semantic by preventing server-side versioning altogether.


### Avoiding Naming Problems
There are several identifiers that a Kubernetes API resource has which share
value-spaces within an API group and must not conflict.  They are:
1. Resource-type value space
  1. plural resource-type name - like "configmaps"
  2. singular resource-type name - like "configmap"
  3. short names - like "cm"
2. Kind-type value space
  1. Kind name - like "ConfigMap"
  2. ListKind name - like "ConfigMapList"
If these values conflict within their value-spaces then no client will be able
to properly distinguish intent.

The actual name of of the TPR-registration (resource that describes the TPR to
create) resource can only protect one of these values from conflict.  Since
Kubernetes API types are accessed via a URL that looks like `/apis/<group>/<version>/namespaces/<namespace-name>/<plural-resource-type>`,
the name of the TPR-registration object will be `<plural-resource-type>.<group>`.

Conflicts with other parts of the value-space can not be detected with static
validation, so there will be a spec/status split with `status.conditions` that
reflect the acceptance status of a TPR-registration.


## New API
In order to: 
1. eliminate opaquely derived information
1. allow the expression of complex transformations
1. handle TPR-registration value-space conflicts
1. [stop using the extensions API group](https://github.com/kubernetes/kubernetes/issues/43214)

We can create a type `ThirdParty.apiextension.k8s.io`.
```go
// ThirdPartySpec describe how a user wants their resource to appear
type ThirdPartySpec struct {
	// Group is the group this resource belongs in
	Group string `json:"group" protobuf:"bytes,1,opt,name=group"`
	// Version is the version this resource belongs in
	Version string `json:"version" protobuf:"bytes,2,opt,name=version"`
	// Name is the plural name of the resource
	Name string `json:"name" protobuf:"bytes,3,opt,name=name"`
	// Singular is the singular name of the resource.  Defaults to lowercased <kind>
	Singular string `json:"singular,omitempty" protobuf:"bytes,4,opt,name=singular"`
	// ShortNames are short names for the resource.
	ShortNames []string `json:"shortNames,omitempty" protobuf:"bytes,5,opt,name=shortNames"`
	// Kind is the serialized kind of the resource
	Kind string `json:"kind" protobuf:"bytes,6,opt,name=kind"`
	// ListKind is the serialized kind of the list for this resource.  Defaults to <kind>List
	ListKind string `json:"listKind,omitempty" protobuf:"bytes,7,opt,name=listKind"`

	// ClusterScoped indicates that this resource is cluster scoped as opposed to namespace scoped
	ClusterScoped bool `json:"clusterScoped" protobuf:"bytes,8,opt,name=clusterScoped"`
}

type ConditionStatus string

// These are valid condition statuses. "ConditionTrue" means a resource is in the condition.
// "ConditionFalse" means a resource is not in the condition. "ConditionUnknown" means kubernetes
// can't decide if a resource is in the condition or not. In the future, we could add other
// intermediate conditions, e.g. ConditionDegraded.
const (
	ConditionTrue    ConditionStatus = "True"
	ConditionFalse   ConditionStatus = "False"
	ConditionUnknown ConditionStatus = "Unknown"
)

// ThirdPartyConditionType is a valid value for ThirdPartyCondition.Type
type ThirdPartyConditionType string

const (
	// NameConflict means the names chosen for this ThirdParty conflict with others in the group.
	NameConflict ThirdPartyConditionType = "NameConflict"
	// Terminating means that the ThirdParty has been deleted and is cleaning up.
	Terminating ThirdPartyConditionType = "Terminating"
)

// ThirdPartyCondition contains details for the current condition of this pod.
type ThirdPartyCondition struct {
	// Type is the type of the condition.
	Type ThirdPartyConditionType `json:"type" protobuf:"bytes,1,opt,name=type,casttype=ThirdPartyConditionType"`
	// Status is the status of the condition.
	// Can be True, False, Unknown.
	Status ConditionStatus `json:"status" protobuf:"bytes,2,opt,name=status,casttype=ConditionStatus"`
	// Last time we probed the condition.
	// +optional
	LastProbeTime metav1.Time `json:"lastProbeTime,omitempty" protobuf:"bytes,3,opt,name=lastProbeTime"`
	// Last time the condition transitioned from one status to another.
	// +optional
	LastTransitionTime metav1.Time `json:"lastTransitionTime,omitempty" protobuf:"bytes,4,opt,name=lastTransitionTime"`
	// Unique, one-word, CamelCase reason for the condition's last transition.
	// +optional
	Reason string `json:"reason,omitempty" protobuf:"bytes,5,opt,name=reason"`
	// Human-readable message indicating details about last transition.
	// +optional
	Message string `json:"message,omitempty" protobuf:"bytes,6,opt,name=message"`
}

// ThirdPartyStatus indicates the state of the ThirdParty
type ThirdPartyStatus struct {
	// Conditions indicate state for particular aspects of a ThirdParty
	Conditions []ThirdPartyCondition `json:"conditions" protobuf:"bytes,1,opt,name=conditions"`
}

// +genclient=true

// ThirdParty represents a resource that should be exposed on the API server.  Its name MUST be in the format
// <.spec.name>.<.spec.group>.
type ThirdParty struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// Spec describes how the user wants the resources to appear
	Spec ThirdPartySpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`
	// Status indicates the actual state of the ThirdParty
	Status ThirdPartyStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}

// ThirdPartyList is a list of ThirdParty objects.
type ThirdPartyList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// Items individual ThirdParties
	Items []ThirdParty `json:"items" protobuf:"bytes,2,rep,name=items"`
}
```


## Behavior
### Create
When a new TPR-registration is completed, no synchronous action is taken.
A controller will run to confirm that value-space of the reserved names doesn't
collide and sets the "NameConflict" condition to `false`.

A custom `http.Handler` which accepts a delegate (like our current
genericapiserver), will look at the `request.RequestInfo` (in the context) and
check its Lister (created by generated client code) for a match with proper
conditions.  If no match is found, it delegates.  If a match is found, it runs
it's own RESTStorage.  If the match was `Terminating`, it will allow reads and
delete, but reject mutations.

### Delete
When a TPR-registration is deleted, it will be handled as a finalizer like a
namespace is done today.  The `Terminating` condition will be updated (like
namespaces) and that will cause mutating requests to be rejected by the REST
handler (see above).  The finalizer will remove all the associated storage.
Once the finalizer is done, it will delete the TPR-registration itself.


## Migration from existing TPR
Because of the changes required to meet the goals, there is not a silent
auto-migration from the existing TPR to the new TPR.  It will be possible, but
it will be manual.  At a high level, you simply:
 1. Stop all clients from writing to TPR (revoke edit rights for all users) and
 stop controllers.
 2. Get all your TPR-data.  
 `$ kubectl get TPR --all-namespaces -o yaml > data.yaml`
 3. Delete the old TPR-data.  
 `$ kubectl delete TPR --all --all-namespaces`
 4. Delete the old TPR-registration.  
 `$ kubectl delete TPR/name`
 5. Create a new TPR-registration.  
 `$ kubectl create -f new_tpr.name`
 6. Recreate your new TPR-data.
 `$ kubectl create -f data.yaml`
 7. Restart controllers.

There are a couple things that you'll need to consider:
 1. Garbage collection.  You may have created links that weren't respected by
 the GC collector in 1.6 (why did you do this, it was documented as not
 working?).  Unlink them before you delete.
 2. Controllers will observe deletes.  Part of this migration actually deletes
 the resource.  Your controller will see the delete.  You ought to shut down
 your TPR controller while you migrate your data.  If you do this, your
 controller will never see a delete.


# stop all clients from writing to TPR (revoke edit rights for all users)? / stop controllers
$ kubectl get TPR --all-namespaces -o yaml > data.yaml


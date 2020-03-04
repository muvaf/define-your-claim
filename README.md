# Composition Repo

This repo will contain the examples to show how the solutions presented in the various discussions around Crossplane composition story.

For now it's private.


There are examples of two solutions for now:
* @negz's latest proposal in @bassam's gist: https://gist.github.com/bassam/c2a5a00134768e0201533ac0ee3a57d0#gistcomment-3188972
  * This is the one that we talked on in the meeting in Feb 25.
  * Will be called Solution A.
* @muvaf's proposal that is built on top of @negz's last proposal: https://gist.github.com/bassam/c2a5a00134768e0201533ac0ee3a57d0#gistcomment-3195047
  * Will be called Solution B.


## Scenarios

We'd like to enable two main scenarios with composition stuff.

### Infra-operator

1. I would like to define my own resource classes which can include N resources which could refer to each other.
  * A resource class with 1 `GKECluster`, 3 `NodePool`s.
2. I would like to expose a singular resource for my users to change the configuration of the underlying N resources. I would like to define which parameters can be changed by the users so that I can enforce a policy.
  * A resource that represents 1 `GKECluster`, 3 `NodePool`s but expose only `networkRef`.
  * The field I expose can be applied to N resources that are in the composition or one specific resource. I should be able to specify which.
3. I would like that single resource to be a namespaced object in some cases and cluster-scoped in other cases.
  * An `MLCluster` resource and the claim it binds; either `KubernetesCluster`(core type) or `MLClusterClaim` that I defined.

### App Developer

I am developing Wordpress and I'm bundling my app's infrastructure requirements via Crossplane CRDs.

1. I would like to include one resource to request a Kubernetes cluster.
  * The resource I bundle in the package should have the same fields with the CRD that is registered in the cluster.
  * I would like to follow Kubernetes approach for this contract; group, version and kind. If I bundle KubernetesCluster.crossplane.io/v1alpha1 and the cluster has the CRD of this, I want to be sure that;
    * Fields are the same with what I tested in my local.
    * My minimum requirement is met: Kubernetes Cluster. I don't want to end up with an EC2 VM even though I bundled a `KubernetesCluster.crossplane.io/v1alpha1` because infra-op defined that GVK should give an EC2 VM.
2. I would like to be sure that the keys in the connection secrets end up as what I expect in the end of the provisioning because I mount them to my pods.
  * KubernetesCluster.crossplane.io/v1alpha1 should have a predefined set of keys that show what corresponds to what and I'd like to be able to use them OR there should be a way for me to make the pairing between what I use and what I'll get. But in the end, the Kind.Group/Version should give me a pre-determined set of keys that I know during development time.

The composition story we end up with could be used to bundle an app, too, but that is considered as secondary goal. Though if we stick with Kubernetes approaches, we'll probably not need much changes if we want to bundle Wordpress as composition, since it's all Kubernetes resources anyway.

## Solutions

The folders show in which scenario, what persona is going to write and what resources will be generated. I tried to write examples that map to our current PVC / PV pattern where app dev bundles a PVC while infra-op writes storage class.

### Solution A

https://gist.github.com/bassam/c2a5a00134768e0201533ac0ee3a57d0#gistcomment-3188972

This is the one that gained much consensus in the meeting. Although @muvaf stated his concerns about `ResourceDefinition`, much of the design is doable. I will not go into too much detail about this here since linked gist comment is already extensive.

* Infra-operator will write two `ResourceDefinition.crossplane.io/v1alpha1`s to define CRDs:
  * One for claim CRD generation: MySQLClaim Namespace-scoped CRD
  * One for resource CRD generation: SecureDatabase Cluster-scoped CRD

* Infra-operator will write `CompositionDefinition.crossplane.io/v1alpha1` to act as resource classes:
  * One replica resource class: CompositionDefinition
  * 3 replicas resource class: CompositionDefinition

Note that it's not necessary to write both. They can write only one of them and let the app dev bundle that.

* App dev will bundle one `MySQLClaim` resource in the app.

### Solution B

The main difference with Solution A is that here `ResourceDefinition` and `CompositionDefinition` is merged. The resulting object is used to generate two CRDs with full validation: `SecureDatabase` and `SecureDatabaseClass`

* Infra-operator will write one `ResourceDefinition` to define CRDs:
  * As resource class CRD: `SecureDatabaseClass`
  * As resource CRD: `SecureDatabase`

* Infra-operator will write one `ClaimDefinition` to define CRD:
  * As claim: `MySQLClaim`

Note that it's not necessary to write both. They can write only one of them and let the app dev bundle that.

* Infra-operator will write strongly typed resource classes:
  * One replica resource class: `SecureDatabaseClass`
  * 3 replicas resource class: `SecureDatabaseClass`

* App dev will bundle one `MySQLClaim`.

## Comparison

### Solution A Advantages
* Less burden during initial setup of the types, i.e. you write only `ResourceDefinition`
* More flexibility for resource class writers in the sense that they are able to define the bindings/transformations as they wish.
  * This could be especially useful when you'd like to write 3 resource classes for different clouds where bindings are different.

### Solution B Advantages

* Resulting CRDs are strong-typed, fully validated.
* Referencing other resources in the same composition is easier since direct identifiers are used during authoring.
* Resource class writers are more constrained in the sense that you cannot put in a kind that was not specified in `ResourceDefinition`, hence there is no scenario where app dev wanted a cluster but got database.


## Conclusion

The main difference between those solutions is where you define the bindings and connection details and that change what you can generate. I think if we alter it a bit more we can get best of both solutions. For example, when we talked about Solution B, @negz proposed that maybe `ResourceDefinition` and `ClaimDefinition` could expose the list of connection keys and resource classes would just match them.

Maybe a mechanism that will allow the resource class writers override the defaults on the definition resources could help here for both connection keys and field bindings.

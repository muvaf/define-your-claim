# Composition Repo

This repo will contain the examples to show how the solutions presented in the various discussions around Crossplane composition story.

There are examples of two solutions for now:
* @negz's latest proposal in @bassam's gist: https://gist.github.com/bassam/c2a5a00134768e0201533ac0ee3a57d0#gistcomment-3188972
  * This is the one that we talked on in the meeting in Feb 25.
  * Will be called Solution A.
* @muvaf's proposal that is built on top of @negz's last proposal: https://gist.github.com/bassam/c2a5a00134768e0201533ac0ee3a57d0#gistcomment-3195047
  * Will be called Solution B.

For personas and general scenarios, see https://docs.google.com/document/d/1enAtUVjnbZP-20L5sboKccLH1aSwb_hBtHEtHDkXPZ0/edit#heading=h.917564hy241x

## Solutions

The folders show in which scenario, what persona is going to write and what resources will be generated. I tried to write examples that map to our current PVC / PV pattern where app dev bundles a PVC while infra-op writes storage class.

### Solution A

https://gist.github.com/bassam/c2a5a00134768e0201533ac0ee3a57d0#gistcomment-3188972

This is the one that gained much consensus in the meeting. Although @muvaf stated his concerns about `ResourceDefinition`, much of the design is doable. I will not go into too much detail about this here since linked gist comment is already extensive.

* Infra-operator will write two `ResourceDefinition.crossplane.io/v1alpha1`s to define CRDs:
  * [One](a/infra-op-writes/definitions/def-mysqlclaim.yaml) for claim CRD generation: [MySQLClaim Namespace-scoped CRD](a/generated/mysqlclaim_crd.yaml)
  * [One](a/infra-op-writes/definitions/def-securedatabase.yaml) for resource CRD generation: [SecureDatabase Cluster-scoped CRD](a/generated/securedatabase_crd.yaml)

* Infra-operator will write `CompositionDefinition.crossplane.io/v1alpha1` to act as resource classes:
  * One resource class if you'd like to bind resources directly to claim: [CompositionDefinition](a/infra-op-writes/classes/bind-to-claim.yaml)
  * One resource class if you'd like to bind resources to a resource, SecureDatabase: [CompositionDefinition](a/infra-op-writes/classes/bind-to-resource.yaml)

Note that you can also bind claim to resource through the same composition mechanism.

* App dev will bundle one `MySQLClaim` resource in the app:
  * In case you composed one `SecureDatabase` to bind to [`MySQLClaim`](a/dev-writes/binds-to-resource.yaml).
  * In case you composed all resources to bind directly to [`MySQLClaim`](a/dev-writes/binds-to-composition-elements.yaml).
  
Note that app dev writes the same `MySQLClaim` but infra-op decides what it binds to.

### Solution B

The main difference with Solution A is that here `ResourceDefinition` and `CompositionDefinition` is merged. The resulting object is used to generate two CRDs with full validation: `SecureDatabase` and `SecureDatabaseClass`

* Infra-operator will write one [`ResourceDefinition`](b/infra-op-writes/definitions/def-securedatabase.yaml) to define CRDs:
  * CRD will be generated to be resource class: [`SecureDatabaseClass`](b/generated/securedatabaseclass_crd.yaml)
  * CRD will be generated to be resource: [`SecureDatabase`](b/generated/securedatabase_crd.yaml)

* Infra-operator will write one `ClaimDefinition` to define CRD or refer to an existing claim in `ResourceDefinition`:
  * CRD will be generated to be claim: [`MySQLClaim`](b/generated/mysqlclaim_crd.yaml)
  * It's possible that a common set of claims are used across an organization as a way of standardizing input parameters and output connection keys.

* Infra-operator will write strongly typed resource classes:
  * One resource class if you'd like to bind resources directly to claim: [`SecureDatabaseClass`](b/infra-op-writes/classes/bind-to-claim.yaml)
  * You cannot bind more than one resource to `MySQLClaim` in this solution. Though we can change that, I think that's a fair opinion for the system to have, i.e. classic PVC model preserved and we allow static provisioning just like today.

* App dev will bundle one `MySQLClaim`.
  * One `SecureDatabase` to bind to [`MySQLClaim`](b/dev-writes/binds-to-resource.yaml).

You can look at the [infra composition diagram](infra-provisioning.jpg) and [app composition diagram](app-composition.jpg) to see the flows.

## Comparison

In solution B:
* Resulting CRDs (SecureDatabase, SecureDatabaseClass and MySQLClaim) are strong-typed, fully validated.
  * Solution A requires you to write `CompositionDefinition` as class. It's easier since you have more freedom but it comes with the expense that you don't have validation and when you'd like to write another class with only a few changes, you will have to duplicate the bindings on that `CompositionDefinition`, too.
* Referencing other resources in the same composition is easier since direct identifiers are used during authoring.
  * This could probably be incorporated into solution A, too.
* Resource class writers are more constrained in the sense that you cannot put in a kind that was not specified in `ResourceDefinition`, hence there is no scenario where app dev wanted a cluster but got database.
  * It's unlikely that this scenario happens but if we introduce some policies about permitting which classes users can use, differentiating them would be much easier if the kinds are different. For instance, if user has permissions only to create Azure resources, we'd know which typed classes they can use. You can do that with `CompositionDefinition` too, but it'd be more cumbersome.

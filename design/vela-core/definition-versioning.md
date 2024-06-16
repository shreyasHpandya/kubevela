# Versioning Support for KubeVela Definitions

<!-- toc -->
- [Versioning Support for KubeVela Definitions](#versioning-support-for-kubevela-definitions)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
  - [Proposal](#proposal)
    - [How to handle existing components](#how-to-handle-existing-components)
  - [Acceptance Criteria](#acceptance-criteria)
  - [Design Details](#design-details)
<!-- /toc -->

## Summary

Support explicit versioning for KubeVela Definitions and a way to specify which
version of the Definition is to be used in an Application spec.


## Motivation

- The current component versioning does not provide the application developers
  to pin the version of the Definition in the Application, it always uses the
  latest component spec when reconciliation is triggered. While we don't want
  application developers to bother with such details, some application
  developers have a use case where they want to pin the definition version to
  avoid auto-upgrade to latest Definition version.

### Goals

- Support KubeVela Definitions versioning using SemVer
- Allow pinning, specific and non-specific versions of a Definition in the
  application

### Non-Goals

- A complicated versioning feature
- Support for version range in Application. For eg.`type: my-component@>1.2.0`

## Proposal

KubeVela Definitions (referred to as Definition for the rest of the document) to
explicitly specify a version as part of the definition follows.

    apiVersion: core.oam.dev/v1beta1
    kind: ComponentDefinition
    metadata:
    name: <ComponentDefinition name>
    annotations:
        definition.oam.dev/description: <Function description>
    spec:
        version: <semver>
        workload: # Workload Capability Indicator
            definition:
                apiVersion: <Kubernetes Workload resource group>
                kind: <Kubernetes Workload types>
        schematic:  # Component description
            cue: # Details of components defined by CUE language
                template: <CUE format template>

The `version` must be a valid SemVer without the prefix `v` for example,
`1.1.0`, `1.2.0-rc.0`.

The Application will then have the ability to refer to the version when using a
Definition like this

    apiVersion: core.oam.dev/v1beta1
    kind: Application
    metadata:
        name: app-with-comp-versioning
    spec:
        components:
            - name: backend
              type: my-component-type@1.0.2

When specifying a Definition version in application spec, it is possible to
specify a non non-specific version, like `v1.0` or `v1`, the KubeVela will
auto-upgrade the application to use the latest version in the specified version
series. For instance, the application specifies `type: my-component-type@1.0`
and `v1.0.1` of the Definition is available, KubeVela will re-render the
application using this version.

In case no version is specified then the version is always upgraded to the
latest.

KubeVela will initiate the reconciliation of the application as soon as a new
version of a component is available and the application is eligible (based on
the version specificity) to be upgraded.

> **Proposal Notes:** We are intentionally skipping a flag like `auto-upgrade:
> true|false` if one does not want to auto upgrade their component, they should
> always use specific semVer in the application spec. This is also consistent
> with existing behaviour where we always use the latest version of the
> component when the application reconciles. We are just providing a way to opt
> out of this behaviour by pinning the component version in the application.
> This is also in line with the philosophy of keeping complexity at the platform
> level instead of the application level.

> **Cannon:** Ideally the upgrades to Definition should be backwards compatible
> all the way, updates to Definitions should never force the application spec to
> change. If a new component is changing something significant it should be a
> Definition with a new name and not the new version of the existing Definition.

### How to handle existing components

The existing component will not have the `version` in its spec. We will treat
them under legacy versioning behaviour. They will continue to behave the same
way as current behaviour, including how their usage in an application is
upgraded only when it is modified. The Definition will be *enrolled* to new
versioning behaviour as soon the `version` field is set in the definition spec.
All the usages of this component by existing application will be enrolled for
auto-upgrade behaviour of the new versioning scheme.

> Proposal Notes: At first glance, it seems like switching over to the new the
> versioning scheme can cause chaos by triggering auto-upgrade across clusters,
> but this is again in line with the philosophy of keeping control in the hands
> of the platform team rather than the application developer. The platform team
> should plan carefully when they enrol their component to new versioning. For
> instance avoid changing anything except `version` in the spec when a component
> is enrolled on new versioning.
>

## Acceptance Criteria

**User Stories**

As a Definition author <br>
I should be able to publish every revision of my Definitions with Semantic
Versioning scheme <br>
So that an Application developer can use a specific revision of the Definition

**BDD Acceptance Criteria**


Given a Definition spec <br>
Given a version field of the spec set to V <br>
When the Definition is applied to the KubeVela <br>
Then it should be listed as one the many versions in Definition list

TODO: Add more BDD Acceptance Criteria



## Design Details

To support versioning we can use the existing `definitionrevisions.core.oam.dev`
resource.

TODO: add more details
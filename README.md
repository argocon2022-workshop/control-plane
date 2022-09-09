# Control Plane

Control Plane repository defines the desired state of shared infrastructure components and enables self-service onboarding process for the application
developer teams.

Repository contains the following directories:

* **argocd** - directory contains Argo CD instance configuration. The configuration includes RBAC settings and infrastructure application definitions.
* **clusters** - directory contains adminstrator level cluster configurations.

## GitOps Process

The idea is to leverage the GitOps approach and pull requests to allow application developer teams to self-onboard by proposing infrastructure changes via pull requests.
To enable the GitOps process we just need to create an Argo CD application that manages Argo CD configuration and propogate repository changes to the Argo CD Kubernetes namespace.

## Multi-Tenancy

Argo CD leverages Projects to separate teams from each other and to where the team can deploy their applications. This task is naturally done by the Argo CD administrator
who is responsible for running the shared Argo CD. However, it does not mean application developers should create tickets to self-onboard. With GitOps, they can send a pull request
that introduces requires Argo CD Projects.

To create project:

* Copy the `argocd/USERNAME-project.yaml` file and replace `USERNAME` with your GitHub username.
* Commit changes and create a pull request to the `main` branch.


## Cluster Infrastructure

Before starting to manage anything, application developer teams need a Kubernetes namespace. To be precise, a set of namespaces, one for each environment. Kubernetes namespaces
are used to isolate application resources from each other and separate teams' permissions, so namespaces must be managed by administrators. 

To create namespace:

*  Copy the `clusters/argocon/USERNAME-namespaces.yaml` file and replace `USERNAME` with your GitHub username. 
*  Commit changes and create a pull request to the `main` branch.

## Let's Deploy Something

Once we are done with configuring infrastructure it's time to use it and deploy something. Jump to
[https://github.com/argocon2022-workshop/demo-app](https://github.com/argocon2022-workshop/demo-app) repository to continue!

## Automating Cluster Management

If you've noticed we've manually created two applications to install kyverno and external-secrets onto the managed cluster.
Both kyverno and external-secrets are infrastructure components that are typically installed into all managed clusters. We might
continue to manually create applications for each managed cluster, but this is error-prone and tedious. The process can be automated
using config management tools and some scripting on top but there is a better way. Argo CD provides a first class support
to cluster administrator use cases - ApplicationSet CRD. ApplicationSet automates application management and provide
featutes to automatically create an app for each managed cluster:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: addons
spec:
  generators:
    - matrix:
        generators:
          - git:
              repoURL: https://github.com/argocon2022-workshop/control-plane
              revision: HEAD
              directories:
                - path: clusters/addons/*
          - clusters:
              selector:
                matchExpressions:
                - {key: 'akuity.io/argo-cd-cluster-name', operator: NotIn, values: [in-cluster]}
  template:
    metadata:
      name: '{{path.basename}}-{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/argocon2022-workshop/control-plane
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: '{{server}}'
        namespace: '{{path.basename}}'
```
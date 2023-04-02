---
layout: post
title: GitOps with Flux!
date: 2023-04-03 04:12:00+0600
description: Learn how to streamline your infrastructure deployment with GitOps and Flux.
categories: gitops
tags: gitops flux kubernetes devops
giscus_comments: true
related_posts: false
---

In this post, we will explore GitOps and how it can be implemented with Flux on a Kubernetes cluster.

### What is Flux?

Flux is a popular GitOps tool used to manage and automate deployments on Kubernetes clusters. It uses a Git repository as the source of truth for your infrastructure and application configurations.

### Creating a Kubernetes Cluster

For this demo, we'll be using KinD (Kubernetes IN Docker), which allows you to create a local Kubernetes cluster for development and testing purposes.

First, create a cluster using KinD:

```
kind create cluster --name demo
```

Output:

```
Creating cluster "demo" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-demo"
You can now use your cluster with:

kubectl cluster-info --context kind-demo
```

Once the cluster is created, you can check its info:

```
kubectl cluster-info --context kind-demo
```

Output:

```
Kubernetes control plane is running at https://127.0.0.1:65535
CoreDNS is running at https://127.0.0.1:65535/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.


The cluster is ready!
```

### Installing Flux CLI

To install the Flux CLI, run the following command:

```
brew install fluxcd/tap/flux
```

### Bootstrap Flux

Before we can bootstrap Flux, we need to create a GitHub personal access token. You can create a token with either the `admin:org` or `repo` permission, depending on whether you need Flux to create a repository or you already have an existing one.

Export the token as an environment variable:

```
export GITHUB_TOKEN=<your-token>
```

Run the bootstrap command to create a repository on your personal GitHub account and configure Flux to sync with it:

```
flux bootstrap github \
  --owner=<your-github-username> \
  --repository=<your-github-repository> \
  --path=clusters/demo \
  --personal
```

Output:

```
► connecting to github.com
✔ repository "https://github.com/<your-github-username>/<your-github-repository>" created
► cloning branch "main" from Git repository "https://github.com/<your-github-username>/<your-github-repository>.git"
✔ cloned repository
► generating component manifests
✔ generated component manifests
✔ committed sync manifests to "main" ("0f908b5013bb6b21c0f6a1b37980a8d85a1dfad2")
► pushing component manifests to "https://github.com/<your-github-username>/<your-github-repository>.git"
► installing components in "flux-system" namespace
✔ installed components
✔ reconciled components
► determining if source secret "flux-system/flux-system" exists
► generating source secret
✔ public key: ecdsa-sha2-nistp384 AAAAE2VjZHNhLXNoYTItbmlzdHAzODQAAAAIbmlzdHAzODQAAABhBBOH8h11Rm92kX9XASEtIsuMDu3AqMTIUZV/6DV0N+w6Kb0tgvE3FY0DXqCkbhpSbiqSK5ojMVrpR3WNjqGGE4Bp+gzEthJ/+L3ocajYbIJRP45pTXnVFWT98fvuz0A9WA==
✔ configured deploy key "flux-system-main-flux-system-./clusters/demo" for "https://github.com/<your-github-username>/<your-github-repository>"
► applying source secret "flux-system/flux-system"
✔ reconciled source secret
► generating sync manifests
✔ generated sync manifests
✔ committed sync manifests to "main" ("c09327b2fcd9db4ac1d5cbe907425a44658631af")
► pushing sync manifests to "https://github.com/<your-github-username>/<your-github-repository>.git"
► applying sync manifests
✔ reconciled sync configuration
◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
✔ Kustomization reconciled successfully
► confirming components are healthy
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
```

This command will create a private git repository and generate Flux manifests that are committed to the main branch. It will also install the Flux components in the flux-system namespace.
Once the repository has been created and configured, you can clone it locally using the following command:

```
git clone git@github.com:<your-github-username>/<your-github-repository>.git
```

### Directory Structure

The directory structure of the repository after Flux is bootstrapped is as follows:

```yaml
<your-github-repository>
└── clusters
    └── demo
        └── flux-system
            ├── gotk-components.yaml
            ├── gotk-sync.yaml
            └── kustomization.yaml
```

This directory contains the Flux manifests that were generated during the bootstrap process.
That's it! You have now bootstrapped Flux and have a Kubernetes cluster ready for GitOps.
Every file located in the directory `clusters/demo` are deployed to your kubernetes cluster because of manifest `gotk-sync.yaml`:

```yaml
# This manifest was generated by flux. DO NOT EDIT.
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  secretRef:
    name: flux-system
  url: ssh://git@github.com/<your-github-username>/<your-github-repository>
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 10m0s
  path: ./clusters/demo
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

This file contains two manifests, `GitRepository` and `Kustomization`. The `Kustomization` manifest references the `GitRepository` on the path `./clusters/demo`.
This `Kustomization` manifest has an interval of 10 minutes which Flux named Reconcile.
Now let's give this directory a good structure.

### How to do it

Create the following three manifests in the path of your cluster:

- `clusters/demo/repositories.yaml`

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: repositories
  namespace: flux-system
spec:
  interval: 3m
  path: ./repositories
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```
- `clusters/demo/infrastructure.yaml`

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 3m
  dependsOn:
    - name: repositories
  path: ./deploy/infrastructure/overlays/demo
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

- `clusters/demo/applications.yaml`

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: applications
  namespace: flux-system
spec:
  interval: 3m
  dependsOn:
    - name: infrastructure
  path: ./deploy/applications/overlays/demo
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

As you can see, those three `Kustomization` have paths and dependencies on each other. Let's create directories:

```
<your-github-repository>
├── clusters
│   └── demo
│       ├── applications.yaml
│       ├── flux-system
│       │   ├── gotk-components.yaml
│       │   ├── gotk-sync.yaml
│       │   └── kustomization.yaml
│       ├── infrastructure.yaml
│       └── repositories.yaml
├── deploy
│   ├── applications
│   │   └── overlays
│   │       └── demo
│   └── infrastructure
│       └── overlays
│           └── demo
└── repositories
```

Create all the necessary repositories in the `repositories` directory. For example:

- `repositories/bitnami.yaml`

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 720h
  timeout: 5m
  url: https://charts.bitnami.com/bitnami
```

- `repositories/banzaicloud.yaml`

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: banzaicloud
  namespace: flux-system
spec:
  interval: 720h
  timeout: 5m
  url: https://kubernetes-charts.banzaicloud.com
```

We can use these repositories to deploy infrastructure or applications. Let's create our infrastructure stack for two environments, production and staging.
Since we do not want to repeat the code for every cluster and environment, we can use the `base` directory in the infrastructure and application directories.
Create the following files to deploy MySQL:

- `deploy/infrastructure/base/mysql/helm-release.yaml`

```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: mysql
spec:
  interval: 720h
  chart:
    spec:
      chart: mysql
      version: '9.4.5'
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
      interval: 720h
  upgrade:
    remediation:
      remediateLastFailure: true
  values:
  valuesFrom:
    - kind: ConfigMap
      name: mysql-values
    - kind: ConfigMap
      name: mysql-values-overrides
```

- `deploy/infrastructure/base/mysql/kustomization.yaml`

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - helm-release.yaml
configMapGenerator:
  - name: mysql-values
    files:
      - values.yaml=values.yaml
configurations:
  - kustomizeconfig.yaml
```

- `deploy/infrastructure/base/mysql/kustomizeconfig.yaml`

```yaml
---
nameReference:
- kind: ConfigMap
  version: v1
  fieldSpecs:
  - path: spec/valuesFrom/name
    kind: HelmRelease
```

- `deploy/infrastructure/base/mysql/values.yaml`

```yaml
---
fullnameOverride: mysql
architecture: standalone
auth:
  rootPassword: "demo"
  createDatabase: true
  database: "demo"
```

Now we are ready to use this base in the overlay directory by addressing this directory and overriding the Helm value for every deployment. For this to be done, we need to create namespaces for production and staging. Create the following files in the infrastructure overlays directory:

- `deploy/infrastructure/overlays/demo/production/kustomization.yaml`

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
resources:
  - namespace.yaml
  - mysql
```

- `deploy/infrastructure/overlays/demo/production/namespace.yaml`

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

- `deploy/infrastructure/overlays/demo/production/mysql/kustomization.yaml`

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../../base/mysql
configMapGenerator:
- name: mysql-values-overrides
  files:
  - values-overrides.yaml=values.yaml
```

- `deploy/infrastructure/overlays/demo/production/mysql/values-overrides.yaml`

```yaml
---
auth:
  rootPassword: "demo-prod"
  createDatabase: true
  database: "demo-prod"
```

- `deploy/infrastructure/overlays/demo/staging/kustomization.yaml`

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: staging
resources:
  - namespace.yaml
  - mysql
```

- `deploy/infrastructure/overlays/demo/staging/namespace.yaml`

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
```

- `deploy/infrastructure/overlays/demo/staging/mysql/kustomization.yaml`

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../../base/mysql
configMapGenerator:
- name: mysql-values-overrides
  files:
  - values-overrides.yaml=values.yaml
```

- `deploy/infrastructure/overlays/demo/staging/mysql/values-overrides.yaml`

```yaml
---
auth:
  rootPassword: "demo-stag"
  createDatabase: true
  database: "demo-stag"
```

Now that the infrastructure part is complete, we can deploy Drupal to the `applications` directory, same as `infrastructure`, with the help of base. Create the
following files:

- `deploy/applications/base/drupal/helm-release.yaml`

```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: drupal
spec:
  interval: 720h
  chart:
    spec:
      chart: drupal
      version: '13.0.9'
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
      interval: 720h
  upgrade:
    remediation:
      remediateLastFailure: true
  values:
  valuesFrom:
    - kind: ConfigMap
      name: drupal-values
    - kind: ConfigMap
      name: drupal-values-overrides
```

- `deploy/applications/base/drupal/kustomization.yaml`

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - helm-release.yaml
configMapGenerator:
  - name: drupal-values
    files:
      - values.yaml=values.yaml
configurations:
  - kustomizeconfig.yaml
```

- `deploy/applications/base/drupal/kustomizeconfig.yaml`

```yaml
---
nameReference:
- kind: ConfigMap
  version: v1
  fieldSpecs:
  - path: spec/valuesFrom/name
    kind: HelmRelease
```

- `deploy/applications/base/drupal/values.yaml`

```yaml
---
fullnameOverride: drupal
persistence:
  enabled: false
mariadb:
  enabled: false
externalDatabase:
  host: ""
  user: bn_drupal
  password: ""
  database: bitnami_drupal
drupalPassword: demo
```

- `deploy/applications/overlays/demo/production/kustomization.yaml`

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
resources:
  - drupal
```

- `deploy/applications/overlays/demo/production/drupal/kustomization.yaml`

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../../base/drupal
configMapGenerator:
- name: drupal-values-overrides
  files:
  - values.yaml=values-overrides.yaml
```

- `deploy/applications/overlays/demo/production/drupal/values-overrides.yaml`

```yaml
---
fullnameOverride: drupal-prod
externalDatabase:
  host: mysql
  user: root
  password: demo-prod
  database: demo-prod
```

- `deploy/applications/overlays/demo/staging/kustomization.yaml`

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: staging
resources:
  - drupal
```

- `deploy/applications/overlays/demo/staging/drupal/kustomization.yaml`

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../../base/drupal
configMapGenerator:
- name: drupal-values-overrides
  files:
  - values.yaml=values-overrides.yaml
```

- `deploy/applications/overlays/demo/staging/drupal/values-overrides.yaml`

```yaml
---
fullnameOverride: drupal-stag
externalDatabase:
  host: mysql
  user: root
  password: demo-stag
  database: demo-stag
```

The final look of the directory should be like this:

```
gitops
├── clusters
│   └── demo
│       ├── applications.yaml
│       ├── flux-system
│       │   ├── gotk-components.yaml
│       │   ├── gotk-sync.yaml
│       │   └── kustomization.yaml
│       ├── infrastructure.yaml
│       └── repositories.yaml
├── deploy
│   ├── applications
│   │   ├── base
│   │   │   └── drupal
│   │   │       ├── helm-release.yaml
│   │   │       ├── kustomization.yaml
│   │   │       ├── kustomizeconfig.yaml
│   │   │       └── values.yaml
│   │   └── overlays
│   │       └── demo
│   │           ├── production
│   │           │   ├── drupal
│   │           │   │   ├── kustomization.yaml
│   │           │   │   └── values-overrides.yaml
│   │           │   └── kustomization.yaml
│   │           └── staging
│   │               ├── drupal
│   │               │   ├── kustomization.yaml
│   │               │   └── values-overrides.yaml
│   │               └── kustomization.yaml
│   └── infrastructure
│       ├── base
│       │   └── mysql
│       │       ├── helm-release.yaml
│       │       ├── kustomization.yaml
│       │       ├── kustomizeconfig.yaml
│       │       └── values.yaml
│       └── overlays
│           └── demo
│               ├── production
│               │   ├── kustomization.yaml
│               │   ├── mysql
│               │   │   ├── kustomization.yaml
│               │   │   └── values-overrides.yaml
│               │   └── namespace.yaml
│               └── staging
│                   ├── kustomization.yaml
│                   ├── mysql
│                   │   ├── kustomization.yaml
│                   │   └── values-overrides.yaml
│                   └── namespace.yaml
└── repositories
    ├── banzaicloud.yaml
    └── bitnami.yaml
```

We need to push our code and wait a couple of minutes for Flux to reconcile the repository and deploy everything to the Kubernetes cluster. We can also force
reconcile with the following command:

```shell
flux reconcile kustomization repositories --with-source
```

You can access the full code repository here: https://github.com/pooryasheikh/gitops

### To-do

- Configure image auto-update
- Configure Sops
- Add HashiCorp Vault to the cluster and read secrets with an init container
- Create a common Helm chart and use it as a HelmGitRepository

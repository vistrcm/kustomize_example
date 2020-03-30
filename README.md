# kustomize_example

This repository demonstrates how to use [kustomize](https://kustomize.io) to modify kubernetes deployment on a per-environment basis. 

## Problem statement

We have an application configured via environment variables. Application deployed into the kubernetes cluster - deployment of the application described in a set of kubernetes manifests.

Application configuration contains next options/environment variables:

* LOG_LEVEL - may be used to configure the logging level.
* DB_URL - may be the URL for the data layer. 
 
The goal is to deploy the application to the 3 different environments using different application and runtime properties:

Dev environment: we only want to set `LOG_LEVEL` to `debug.` No changes in `DB_URL.`

Staging: change DB_URL to use a staging database.

Production: change DB_URL to use the production database and increase the number of replicas of the application to 10.

Nice to have: define label `stage` common for all the deployment objects with the value equals to the type of the environment. 

## kustomize as a solution

One of the possible solutions is to use [kustomize](https://kustomize.io). Kustomize started as 3rd party helper for kubectl, and now it is integrated into kubectl. So it is the default and recommended way to modify kubernetes manifests.

To use kustomize for the problem stated above, we can utilize the "overlays" feature of the kustomize. The idea is straightforward: define k8s deployment in the `base` layer and use `overlays` to apply per-environment changes in the form of patches and common objects.  

1. Define k8s deployment.

Kubernetes manifests presented in the `base/` is usual k8s Deployment and ConfigMap. The configuration decoupled from the deployment definition.
```shell script
$ cat base/configMap.yaml
...
data:
  LOG_LEVEL: info
  DB_URL: jdbc://SHARED_DB
...
```

Deployment use environment variables defined in the ConfigMap.
```shell script
$ cat base/deployment.yaml
...
envFrom:  # use environment variables from the configMap
- configMapRef:
    name: kust-config
...
```

The only new thing here is the `kustomization.yaml.` File, which defines the behavior of the customization.

You can use `kubectl kustomize ./base` to see resulting manifests:
```shell script
$ kustomize build ./base
...
data:
  DB_URL: jdbc://SHARED_DB
  LOG_LEVEL: info
kind: ConfigMap
...
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kust-test
spec:
  replicas: 1
...
  template:
...
    spec:
      containers:
...
        envFrom:
        - configMapRef:
            name: kust-config
...
```
2. create overlays

Create overlays per environment. Let's review the example of the production environment.

To change value of we can create a patch for the ConfigMap:
```shell script
$ cat overlays/prod/configMap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kust-config
data:
  DB_URL: jdbc://PRODUCTION_DB
```

> Note: you don't need to overwrite whole configuration, only variables that should be different from the base.

To increase amount of replicas use another patch `overlays/prod/increase_replicas.yaml`:
```shell script
$ cat overlays/prod/increase_replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kust-test
spec:
  replicas: 10
```

Common labels may be set via `kustomization.yaml` feature:
```shell script
$ cat overlays/prod/kustomization.yaml
...
# set common labels
commonLabels:
  stage: prod
...
```

Let's see result of the customization:
```shell script
$ kubectl kustomize ./overlays/prod
...
data:
  DB_URL: jdbc://PRODUCTION_DB
  LOG_LEVEL: info
kind: ConfigMap
metadata:
  labels:
    stage: prod
  name: prod-kust-config
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    stage: prod
  name: prod-kust-test
spec:
  replicas: 10
  selector:
    matchLabels:
      app: kust-test
      stage: prod
  template:
...
    spec:
      containers:
...
        envFrom:
        - configMapRef:
            name: prod-kust-config
...
```


You can also take a look on differences between resulting definitions for prod and dev:
```shell script
$ diff <(kubectl kustomize ./overlays/dev) <(kubectl kustomize ./overlays/prod)
3,4c3,4
<   DB_URL: jdbc://SHARED_DB
<   LOG_LEVEL: debug
---
>   DB_URL: jdbc://PRODUCTION_DB
>   LOG_LEVEL: info
8,9c8,9
<     stage: dev
<   name: dev-kust-config
---
>     stage: prod
>   name: prod-kust-config
15,16c15,16
<     stage: dev
<   name: dev-kust-test
---
>     stage: prod
>   name: prod-kust-test
18c18
<   replicas: 1
---
>   replicas: 10
22c22
<       stage: dev
---
>       stage: prod
27c27
<         stage: dev
---
>         stage: prod
36c36
<             name: dev-kust-config
---
>             name: prod-kust-config
```

## Repository structure

```text
.
├── base         - base k8s definitions
└── overlays     - overlays collection
    ├── dev      - development environment
    ├── prod     - production environment
    └── staging  - staging environment
```


### links

* [Declarative Management of Kubernetes Objects Using Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
* [Kustomize website](https://kustomize.io)

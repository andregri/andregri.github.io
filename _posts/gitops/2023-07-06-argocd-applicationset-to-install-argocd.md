---
layout: single
title: Install Kyverno with an ArgoCD ApplicationSet
toc: true
tags: gitops
---
This post talks about different methods to bootstrap Kyverno on Kubernetes using ArgoCD:

- ArgoCD Application + Kyverno Helm chart
- ArgoCD ApplicationSet + Kyverno manifests
- ArgoCD ApplicationSet + Kyverno Helm chart
- ArgoCD ApplicationSet + Kyverno local Helm chart

At the time of writing this post, ArgoCD version is v2.7.6 and Kyverno version is 1.10.0.

# 1. App + Helm Application

Kyverno official [documentation](https://main.kyverno.io/docs/installation/platform-notes/#notes-for-argocd-users) provides a sample of ArgoCD Application manifest to install Kyverno through Helm:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kyverno
  namespace: argocd
spec:
  destination:
    namespace: kyverno
    server: https://kubernetes.default.svc
  project: default
  source:
    chart: kyverno
    repoURL: https://kyverno.github.io/kyverno
    targetRevision: 2.6.0
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - Replace=true
```

However, the manifest above is not complete, because the Documentation itself says “*to ignore differences in aggregated ClusterRoles which Kyverno uses by default”.* The suggested solution requires to modify the global settings of ArgoCD, but it may not be possible if ArgoCD is multi-tenant. This Github issue https://github.com/argoproj/argo-cd/issues/2382 provides an alternative using the `ignoreDifferences` attribute of Application specification:

```yaml
spec:
  ignoreDifferences:
	- group: rbac.authorization.k8s.io
	    kind: ClusterRole
	    jsonPointers:
	    - /rules
```

The final ArgoCD Application manifest (also on Github repository [argocd-kyverno)](https://github.com/andregri/argocd-kyverno/blob/main/app-helm/app.yaml) is:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kyverno
  namespace: argocd
spec:
  destination:
    namespace: kyverno
    server: https://kubernetes.default.svc
  project: default
  source:
    chart: kyverno
    repoURL: https://kyverno.github.io/kyverno
    targetRevision: 2.6.0
  ignoreDifferences:
  - group: rbac.authorization.k8s.io
    kind: ClusterRole
    jsonPointers:
      - /rules
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - Replace=true
```

Create the ArgoCD App:

```console
$ kubectl create -n argocd -f https://raw.githubusercontent.com/andregri/argocd-kyverno/main/app-helm/app.yaml
application.argoproj.io/kyverno created
```

Even if the paragraph [“Ownership Clashes”](https://main.kyverno.io/docs/installation/platform-notes/#ownership-clashes) in the Kyverno documentation talks about conflicts between Helm and ArgoCD for the `app.kubernetes.io/instance` label, I didn't have any problem using this provisioning method.

Check the Kyverno pods:

```console
$ kubectl get pods -n kyverno
NAME                       READY   STATUS    RESTARTS   AGE
kyverno-5f7d9f9675-pr8qn   1/1     Running   0          35s
```

Cleanup the resources:

```console
$ kubectl delete -n argocd -f https://raw.githubusercontent.com/andregri/argocd-kyverno/main/app-helm/app.yaml
application.argoproj.io "kyverno" deleted

$ kubectl delete ns kyverno
namespace "kyverno" deleted

$ kubectl delete mutatingwebhookconfigurations kyverno-policy-mutating-webhook-cfg kyverno-resource-mutating-webhook-cfg kyverno-verify-mutating-webhook-cfg

$ kubectl delete validatingwebhookconfigurations kyverno-policy-validating-webhook-cfg kyverno-resource-validating-webhook-cfg kyverno-cleanup-validating-webhook-cfg kyverno-exception-validating-webhook-cfg
```

# 2. ApplicationSet + manifests

Since Kyverno documentation warns about ownership clashes between ArgoCD and Helm, this time we provision Kyverno using an ArgoCD ApplicationSet and Kyverno yaml manifests, that are available at [“Install Kyverno using YAMLs”](https://kyverno.io/docs/installation/methods/#install-kyverno-using-yamls).

The directory structure of the ApplicationSet available on [Github](https://github.com/andregri/argocd-kyverno/tree/main/appset) is:

- appset/
    - app/
        - templates/
            - kyverno-v1.10.0.yaml
        - Chart.yaml
    - config/test/
        - config.json
    - appset.yaml

The ApplicationSet uses the [Git file generator](https://argocd-applicationset.readthedocs.io/en/stable/Generators-Git/#git-generator-files):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: appset-kyverno
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/andregri/argocd-kyverno.git
      revision: HEAD
      files:
      - path: "appset/config/**/config.json"
  template:
    metadata:
      name: 'kyverno-{{ env }}'
    spec:
      project: default
      source:
        repoURL: https://github.com/andregri/argocd-kyverno.git
        targetRevision: main
        path: "appset/app"
      ignoreDifferences:
      - group: rbac.authorization.k8s.io
        kind: ClusterRole
        jsonPointers:
          - /rules
      destination:
        server: https://kubernetes.default.svc
        namespace: kyverno
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true
          - Replace=true
```

According to issue  https://github.com/argoproj/argo-cd/issues/3474, it turns out that at the moment ArgoCD can only recognize application declarations made in `argocd` namespace. So remember to create the ApplicationSet there.

Create the ApplicationSet:

```console
$ kubectl create -f https://raw.githubusercontent.com/andregri/argocd-kyverno/main/appset/appset.yaml
applicationset.argoproj.io/appset-kyverno created
```

Sync the app:

```console
$ argocd app sync kyverno-test

$ oc get pods -n kyverno
NAME                                             READY   STATUS             RESTARTS      AGE
kyverno-admission-controller-7964964bd8-vft5j    0/1     CrashLoopBackOff   4 (50s ago)   2m32s
kyverno-background-controller-686d748ff9-t7rvw   1/1     Running            0             2m32s
kyverno-cleanup-controller-64b6bc868c-4vzjp      0/1     Running            1 (23s ago)   2m32s
kyverno-reports-controller-5f99c5fbdc-v5lww      1/1     Running            0             2m32s

$ oc logs kyverno-admission-controller-7964964bd8-vft5j -n kyverno
...
setup "msg"="sanity checks failed" "error"="CRDs not installed"
```

The installation is not completed properly because kyverno-admiossion-controoler and kyverno-cleanup-controller pods fails.

Cleanup:

```console
$ kubectl delete -f https://raw.githubusercontent.com/andregri/argocd-kyverno/main/appset/appset.yaml
applicationset.argoproj.io "appset-kyverno" deleted

$ kubectl delete ns kyverno
namespace "kyverno" deleted

$ kubectl delete mutatingwebhookconfigurations kyverno-policy-mutating-webhook-cfg kyverno-resource-mutating-webhook-cfg kyverno-verify-mutating-webhook-cfg

$ kubectl delete validatingwebhookconfigurations kyverno-policy-validating-webhook-cfg kyverno-resource-validating-webhook-cfg kyverno-cleanup-validating-webhook-cfg kyverno-exception-validating-webhook-cfg
```

# 3. ApplicationSet + Helm chart

Now we start from example ArgoCD application provided by Kyverno but inside an ApplicationSet.

The ApplicationSet using the Git file generator is similar to the ApplicationSet using the manifests but uses the Kyverno Helm chart:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: appset-kyverno
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/andregri/argocd-kyverno.git
      revision: HEAD
      files:
      - path: "appset-helm/config/**/config.json"
  template:
    metadata:
      name: 'kyverno-{{ env }}'
    spec:
      project: kyverno
      source:
        repoURL: https://github.com/andregri/argocd-kyverno.git
        targetRevision: main
        path: "appset-helm/app"
      ignoreDifferences:
      - group: rbac.authorization.k8s.io
        kind: ClusterRole
        jsonPointers:
          - /rules
      destination:
        server: https://kubernetes.default.svc
        namespace: kyverno
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
          - Replace=true
```

Create the ApplicationSet:

```console
$ kubectl create -f https://raw.githubusercontent.com/andregri/argocd-kyverno/main/appset-helm/appset.yaml
applicationset.argoproj.io/appset-kyverno created
```

Sync the app:

```console
$ argocd app sync kyverno-test

$ kubectl get application kyverno-test -n argocd
NAME           SYNC STATUS   HEALTH STATUS
kyverno-test   Synced        Healthy
```

Cleanup:

```console
$ kubectl delete -f https://raw.githubusercontent.com/andregri/argocd-kyverno/main/appset-helm/appset.yaml

$ kubectl delete ns kyverno

$ kubectl delete mutatingwebhookconfigurations kyverno-policy-mutating-webhook-cfg kyverno-resource-mutating-webhook-cfg kyverno-verify-mutating-webhook-cfg

$ kubectl delete validatingwebhookconfigurations kyverno-policy-validating-webhook-cfg kyverno-resource-validating-webhook-cfg kyverno-cleanup-validating-webhook-cfg kyverno-exception-validating-webhook-cfg
```

# 4. ApplicationSet + local Helm chart

Again we start from example ArgoCD application provided by Kyverno but inside an ApplicationSet. However, this time the chart is present locally under the path `appset-local-helm/app/charts/kyverno-3.0.1.tgz`:

- appset-local-helm/
    - app/
        - charts/
            - kyverno-3.0.1.tgz
        - templates/
        - Chart.yaml

The Chart.yaml file specifies to get the Chart from a local path:

```yaml
apiVersion: v2
name: isp-kyverno
description: A Helm chart for Kyverno
type: application
version: 0.1.0
appVersion: "v1.10.0"
dependencies:
- name: kyverno
  version: 3.0.1
  repository: "file://./charts/"
```

The ApplicationSet configuration file is the same as the example, apart from the source path: `"appset-local-helm/app"`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: appset-kyverno
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/andregri/argocd-kyverno.git
      revision: HEAD
      files:
      - path: "appset-local-helm/config/**/config.json"
  template:
    metadata:
      name: 'kyverno-{{ env }}'
    spec:
      project: kyverno
      source:
        repoURL: https://github.com/andregri/argocd-kyverno.git
        targetRevision: main
        path: "appset-local-helm/app"
      ignoreDifferences:
      - group: rbac.authorization.k8s.io
        kind: ClusterRole
        jsonPointers:
          - /rules
      destination:
        server: https://kubernetes.default.svc
        namespace: kyverno
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
          - Replace=true
```

Create the ApplicationSet:

```console
$ kubectl create -f https://raw.githubusercontent.com/andregri/argocd-kyverno/main/appset-local-helm/appset.yaml
```

Sync the app:

```console
$ argocd app sync kyverno-test

$ kubectl get application kyverno-test -n argocd
NAME           SYNC STATUS   HEALTH STATUS
kyverno-test   Synced        Healthy
```

Cleanup:

```console
$ kubectl delete -f https://raw.githubusercontent.com/andregri/argocd-kyverno/main/appset-local-helm/appset.yaml

$ kubectl delete ns kyverno

$ kubectl delete mutatingwebhookconfigurations kyverno-policy-mutating-webhook-cfg kyverno-resource-mutating-webhook-cfg kyverno-verify-mutating-webhook-cfg

$ kubectl delete validatingwebhookconfigurations kyverno-policy-validating-webhook-cfg kyverno-resource-validating-webhook-cfg kyverno-cleanup-validating-webhook-cfg kyverno-exception-validating-webhook-cfg
```

# Conclusions

ArgoCD ApplicationSet seems to be the standard and new way of ArgoCD to bootstrap cluster instead of the app of apps pattern.

Even though Kyverno provides an example of installation through ArgoCD and Helm and then claims that there clashes between the two, the trials above shows that it is possible to install Kyverno with ArgoCD.

Since Kyverno creates aggregated ClusterRoles resources in Kubernetes, add the ignoreDifferences field in the ArgoCD app.

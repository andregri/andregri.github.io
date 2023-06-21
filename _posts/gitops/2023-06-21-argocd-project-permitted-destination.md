---
layout: single
title: Manage permitted destination of an ArgoCD Project
toc: true
tags: gitops
---
This post talks about an ArgoCD feature that allows to select which namespaces are allowed to host ArgoCD Applications belonging to the same project.

At time of writing this post, ArgoCD latest version is [2.7.4](https://github.com/argoproj/argo-cd/tree/v2.7.4).

# What is an AppProject in ArgoCD

AppProject is a custom resource of ArgoCD that allows to group Apps logically. As you can read on the [official documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/#projects), three out of four features provided by AppProjects start with *“restict where/what …”*. It is not hard to understand that they are used to restrict the App “scope”, particularly when ArgoCD is used by many teams or tenants. For instance, Team A shouldn't create Apps on namespaces of Team B.

# Create an AppProject

I took this simple AppProject from official [documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#projects) that allows to create ArgoCD apps only on `guestbook` namespace:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: guestbook-project
  namespace: argocd
  # Finalizer that ensures that project is not deleted until it is not referenced by any application
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: Example Project
  # Allow manifests to deploy from any Git repos
  sourceRepos:
  - '*'
  # Only permit applications to deploy to the guestbook namespace in the same cluster
  destinations:
  - namespace: guestbook
    server: https://kubernetes.default.svc
```

Create the project from the manifest uploaded in my [repository](https://github.com/andregri/argocd-example-apps/blob/master/01-project-permitted-destination/guestbook-project.yaml):

```console
$ kubectl apply -f https://raw.githubusercontent.com/andregri/argocd-example-apps/master/01-project-permitted-destination/guestbook-project.yaml 
appproject.argoproj.io/guestbook-project created
```

List the projects:

```console
$ kubectl get appprojects -n argocd
NAME                AGE
default             26d
guestbook-project   64s
```

## Try to create an App in a namespace not permitted

Again, I took a simple App from the [documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#applications), but this time I changed two attributes:

- `.spec.project: my-project` to create the App in the AppProject `my-project`.
- `.spec.destination.namespace: default` to create the resource in the `default` namespace, that it is different from the permitted namespace `guestbook`!

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: my-project
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: default
```

Create the App:

```console
$ kubectl apply -f https://raw.githubusercontent.com/andregri/argocd-example-apps/master/01-project-permitted-destination/not-permitted-app.yaml
application.argoproj.io/guestbook created
```

List the apps:

```yaml
$ kubectl get app -n argocd
NAME        SYNC STATUS   HEALTH STATUS
guestbook   Unknown       Unknown
```

The application is not synced because of an error. Get the `status` of the resource:

```yaml
$ kubectl get app guestbook -n argocd -o jsonpath="{.status}" | jq
{
  "conditions": [
    {
      "lastTransitionTime": "2023-06-20T15:50:24Z",
      "message": "application destination {https://kubernetes.default.svc default} is not permitted in project 'guestbook-project'",
      "type": "InvalidSpecError"
    }
  ],
  "health": {
    "status": "Unknown"
  },
  "sync": {
    "status": "Unknown"
  }
}
```

The App is not created because *“application destination {https://kubernetes.default.svc default} is not permitted in project '*guestbook*-project'".*

## Change the App destination namespace to a permitted namespace

Update the destination namespace in the App manifest with a permitted namespace:

- `.spec.destination.namespace: default` that is permitted by AppProject `guestbook-project`

Create the namespace `guestbook` if it doesn't exist:

```yaml
$ kubectl create ns guestbook
namespace/guestbook created
```

Apply the changes:

```yaml
$ kubectl apply -f https://raw.githubusercontent.com/andregri/argocd-example-apps/master/01-project-permitted-destination/permitted-app.yaml
application.argoproj.io/guestbook configured
```

Check the App:

```yaml
$ kubectl get app guestbook -n argocd
NAME        SYNC STATUS   HEALTH STATUS
guestbook   OutOfSync     Missing
```

This time the App was created because the `SYNC STATUS` is `OutOfSync`.

# Overriding the destination namespace

Create two Apps in the `guestbook` Project. The destination namespace of these Apps is `default`, which is the permitted namespace by the project.

App 1 points to path `guestbook/dev` of repo [argocd-example-apps](https://github.com/andregri/argocd-example-apps). However, the deployment provisioned by this App is specifying the namespace as well: `namespace: guestbook-dev`.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-dev
  namespace: argocd
spec:
  project: guestbook-project
  source:
    repoURL: https://github.com/andregri/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
```

App 2 points to path `guestbook/prod` of repo [argocd-example-apps](https://github.com/andregri/argocd-example-apps). However, the deployment provisioned by this App is specifying the namespace as well: `namespace: guestbook-prod`.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-prod
  namespace: argocd
spec:
  project: guestbook-project
  source:
    repoURL: https://github.com/andregri/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
```

According to [ArgoCD issue](https://github.com/argoproj/argo-cd/pull/8383/files), the destination namespace defined in the App should be overridden by the namespace defined in the resource.

Create the namespaces defined in the Deployments:

```console
$ kubectl create ns guestbook-dev
namespace/guestbook-dev created

$ kubectl create ns guestbook-prod
namespace/guestbook-prod created
```

Create the Apps:

```console
$ kubectl apply -f https://raw.githubusercontent.com/andregri/argocd-example-apps/master/02-project-override-namespace/app-dev.yaml
application.argoproj.io/guestbook-dev created

$ kubectl apply -f https://raw.githubusercontent.com/andregri/argocd-example-apps/master/02-project-override-namespace/app-prod.yaml
application.argoproj.io/guestbook-prod created
```

Sync the apps with argocdcli:

```console
$ argocd app sync guestbook-dev
...
GROUP  KIND        NAMESPACE      NAME          STATUS     HEALTH   HOOK  MESSAGE
apps   Deployment  guestbook-dev  guestbook-ui  OutOfSync  Missing        namespace guestbook-dev is not permitted in project 'guestbook-project'
FATA[0001] Operation has completed with phase: Failed
```

The app sync failed because the deployment is overriding the destination namespace with a namespace that is not permitted by the project.

Update the project permitted namespaces:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: guestbook-project
  namespace: argocd
  # Finalizer that ensures that project is not deleted until it is not referenced by any application
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: Example Project
  # Allow manifests to deploy from any Git repos
  sourceRepos:
  - '*'
  # Only permit applications to deploy to the guestbook-dev namespace in the same cluster
  destinations:
  - namespace: guestbook-dev
    server: https://kubernetes.default.svc
  - namespace: guestbook-prod
    server: https://kubernetes.default.svc
  - namespace: guestbook
    server: https://kubernetes.default.svc
```

Update the project:

```console
$ kubectl apply -f https://raw.githubusercontent.com/andregri/argocd-example-apps/master/02-project-override-namespace/guestbook-project.yaml 
appproject.argoproj.io/guestbook-project configured
```

Sync the app again:

```console
$ argocd app sync guestbook-dev
$ argocd app sync guestbook-prod
```

Check that the deployment was created in the namespace override by the deployment:

```console
$ kubectl get pods -n guestbook-dev
NAME                           READY   STATUS    RESTARTS   AGE
guestbook-ui-b848d5d9d-fzdtf   1/1     Running   0          11m

$ kubectl get pods -n guestbook-prod
NAME                           READY   STATUS    RESTARTS   AGE
guestbook-ui-b848d5d9d-srmq4   1/1     Running   0          41s
```

# Conclusion

- Applications in ArgoCD allows to define a destination namespace for the Kubernetes resources to deploy
- ApplicationProject in ArgoCD permits to deploy App resources only in the namespace defined in `.destination.namespace`
- If the kubernetes resource defines explicitly a namespace, then it overrides the Application destination namespace.

# Resources

- https://argo-cd.readthedocs.io/en/stable/user-guide/projects/#managing-projects
- https://github.com/argoproj/argo-cd/pull/8383/files → **# The namespace will only be set for namespace-scoped resources that have not set a value for .metadata.namespace**

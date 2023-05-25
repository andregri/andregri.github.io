---
layout: single
title: Troubleshooting Kyverno webhooks
toc: true
tags: kubernetes policy
---

Kyverno is a policy engine designed specifically for Kubernetes.
It is based on validanting and mutating webhooks that intercept resources before they are created.
This post will show how to troubleshoot Kyverno and it will dive into Kyverno resources when it seems that your policy is not applied.

# Troubleshooting Kyverno

## Test the policy locally

Install Kyverno CLI on your machine following one of the **[guides](https://kyverno.io/docs/kyverno-cli/#building-and-installing-the-cli)**.

Prepare a policy and a resource used to test the policy, then run a dry-run test of the policy with command:

```console
$ kyverno apply path/to/policy.yaml --resource path/to/resource.yaml
```

If the policy is a mutating policy, the command should output the mutated resource.

## Check the installation on a local cluster

Since I was testing Kyverno installation, I used the manifest method:

```console
$ kubectl create -f path/to/install.yaml
```

Create the mutating policy, then the resource and check if it is mutated:

```console
$ kubectl create -f path/to/policy.yaml
$ kubectl create -f path/to/resource.yaml
```

In the logs of the Kyverno pod, you can see the log when the resource is mutated:

```console
$ kubectl logs kyverno-67d595bb95-v6c9k -n kyverno

...
I0525 06:51:32.819131       1 mutation.go:120] webhooks/resource/mutate "msg"="mutation rules from policy applied successfully" "gvk"={"group":"apps","version":"v1","kind":"Deployment"} "kind"="Deployment" "name"="nginx" "namespace"="default" "operation"="CREATE" "policy"="inject-sidecar" "rules"=["inject-sidecar"] "uid"="2cf747fe-152c-4cf1-875a-885cca6504a3" "user"={"username":"kubernetes-admin","groups":["system:masters","system:authenticated"]}
```

## Check if the version of Kyverno is compatible to the version of Kubernetes

Check the version of Kubernetes server against the [compatibility matrix of Kyverno](https://kyverno.io/docs/installation/#compatibility-matrix):

```console
$ kubectl version
```

## Troubleshoot the installation on the target cluster

Install Kyverno and wait for pods to be running:

```console
$ kubectl create -f path/to/install.yaml
```

Create the policy and check it is ready:

```console
$ kubectl create -f path/to/policy.yaml
$ kubectl get clusterpolicy
```

Check the status of webhooks registered by Kyverno:

```console
$ kubectl get validatingwebhookconfigurations

NAME                                                 WEBHOOKS   AGE
kyverno-cleanup-validating-webhook-cfg               1          17h
kyverno-exception-validating-webhook-cfg             1          17h
kyverno-policy-validating-webhook-cfg                1          7m45s
kyverno-resource-validating-webhook-cfg              0          7m45s
```

Check the mutating web hooks:

```console
$ kubectl get mutatingwebhookconfigurations

NAME                                    WEBHOOKS   AGE
kyverno-policy-mutating-webhook-cfg     1          7m57s
kyverno-resource-mutating-webhook-cfg   0          7m57s
kyverno-verify-mutating-webhook-cfg     1          7m57s
```

The two webhook configurations `kyverno-resource-validating-webhook-cfg` and `kyverno-resource-mutating-webhook-cfg` are empty but as soon as you create a validating policy or a mutating policy, those webhooks are populated, respectively.

Create the policy and check again the webhook configurations:

```console
kubectl create -f path/to/policy
kubectl get mutatingwebhookconfigurations

NAME                                    WEBHOOKS   AGE
kyverno-policy-mutating-webhook-cfg     1          7m57s
kyverno-resource-mutating-webhook-cfg   1          7m57s
kyverno-verify-mutating-webhook-cfg     1          7m57s
```

# Understanding how Kyverno works

## Webhook rule

Webhook are created dynamically based on the policies that are deployed on the cluster: if it is a mutating policy for deployments, the **webhook rule** is set for the kind “Deployment” of apiGroup “apps”:

```console
$ kubectl get mutatingwebhookconfiguration kyverno-resource-mutating-webhook-cfg -o yaml
...
rules:
  - apiGroups:
    - apps
    apiVersions:
    - v1
    operations:
    - CREATE
    - UPDATE
    resources:
    - deployments
    scope: '*'
...
```

## Webhook namespace selector

The namespace selector of the webhook defines the namespace that should not be affected by the webhook. And Kyverno defines a selector excluding the kyverno namespace to prevent applying mutating policies to the kyverno deployment itself:

```console
$ kubectl get mutatingwebhookconfiguration kyverno-resource-mutating-webhook-cfg -o yaml
...
namespaceSelector:
    matchExpressions:
    - key: kubernetes.io/metadata.name
      operator: NotIn
      values:
      - kyverno
...
```

# Conclusion

- Test the policy using kyverno cli
- Test the policy on a local cluster
- Test the policy on the target cluster
- Understand validating and mutating webhook configuration, rules and namespace selectors

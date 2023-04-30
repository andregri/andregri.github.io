---
layout: single
title: Testing Kubernetes resources with Kuttl
toc: true
tags: kubernetes testing
---

In Kubernetes the infrastructure that runs an application is described by some YAML manifests. While application source code is tested with additional code written almost always in the same language, manifest describing the architecture are almost never tested. It means that changes in the manifests affect immediately the infrastructure.

Infrastructure code is tested in different pre-production environments to detect with advance possible breaking changes. However, it is difficult to obtain test and production environments that are exactly the same.

While testing infrastructure changes on different environments is always a valid strategy, why don't test the infrastructre manifests before deploying them to each environment?

# kuttl

[kuttl](https://github.com/kudobuilder/kuttl), The KUbernetes Test TooL, is a tool for end-2-end testing in Kubernetes environments. Test cases are written in yaml, that is an advantage because it is the same format used for kubernetes manifests, but it may be limiting for writing more complex tests.

Let's see how to write the common steps of each test: arrange, act, assert,

## Arrange

In the arrange phase you define the resources to be tested. File name should follow the following format: `00-install.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
...
---
apiVersion: batch/v1
kind: Job
metadata:
...
```

## Assert

Assertions are descripted in another file named `00-assert.yaml`:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ...
status:
  succeeded: 1
```

## Act

In the act phase you run the test using the arranged data. In this case, you deploy the resources on a kubernetes run by KinD:
```console
$ kubectl kuttl test --start-kind path/to/tests
```

# Conclusion

kuttl is a great tool to test end-2-end kubernetes manifests even if test cases written in yaml may not be very expressive. However, your kubernetes manifests are written in yaml so this is your input test data to be tested.
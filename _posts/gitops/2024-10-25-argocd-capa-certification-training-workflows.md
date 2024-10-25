---
layout: single
title: Argocd CAPA Certification Training - Workflows
toc: true
tags: gitops
---

Argocd Workflows is an orchestrator of parallel job on Kubernetes.

Create the namespace **argo**:
```
kubectl create ns argo
```

Install the version 3.5.11:
```
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.5.11/install.yaml
```

List the deployments installed:
```
kubectl get deploy -n argo

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
argo-server           0/1     1            0           39s
workflow-controller   1/1     1            1           39s
```
- **argo-server** is a web ui and api server
- **workflow-controller** is the engine running the workflows

Wait for deployments to be ready:
```
kubectl -n argo wait deploy --all --for condition=Available --timeout 2m
```

## Workflow
- A workflow is a Kubernetes resource
- A workflow is made of one or more **templates**
- One template is the **entrypoint**
- Each template can be one of several **types** (e.g. container)

Create an example workflow:
```
cat <<EOF | kubectl apply -n argo -f -
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  name: hello
spec:
  serviceAccountName: argo # this is the service account that the workflow will run with
  entrypoint: main # the first template to run in the workflows
  templates:
  - name: main
    container: # this is a container template
      image: docker/whalesay # this image prints "hello world" to the console
      command: ["cowsay"]
EOF
```

Wait for the workflow to complete:
```
kubectl -n argo wait workflows/hello --for condition=Completed --timeout 2m
```

## Using the web ui
Default authentication on argo-server is **client authentication**, that is using the kubernetes bearer token.
The **server authentication mode** bypass the login.

Apply the patch:
```
kubectl patch deployment \
  argo-server \
  --namespace argo \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args", "value": [
  "server",
  "--auth-mode=server",
  "--secure=false"
]},
{"op": "replace", "path": "/spec/template/spec/containers/0/readinessProbe/httpGet/scheme", "value": "HTTP"}
]'
```

Port-forward the argo-server to access the web ui:
```
kubectl -n argo port-forward --address 0.0.0.0 svc/argo-server 2746:2746 > /dev/null &
```

Wait for the deployment to be redeployed:
```
controlplane $ kubectl -n argo rollout status --watch --timeout=600s deployment/argo-server
```

## Using the CLI

Install the argo CLI:
```
curl -sLO https://github.com/argoproj/argo-workflows/releases/download/v3.5.10/argo-linux-amd64.gz
gunzip argo-linux-amd64.gz
chmod +x argo-linux-amd64
mv ./argo-linux-amd64 /usr/local/bin/argo
```

Check the installation:
```
argo version
```

Run a workflow:
```
argo submit -n argo --serviceaccount argo --watch \
  https://raw.githubusercontent.com/argoproj/argo-workflows/master/examples/hello-world.yaml
```

List the workflows:
```
argo list -n argo
```

Get details of a specific workflow:
```
argo list -n argo @latest
```

View the logs of a specific workflow:
```
argo logs -n argo @latest
```

## Templates

There are 2 categories of templates:
- work
- orchestration

The **work** category includes work to be done:
- container
- container set
- data
- resource
- script

The **orchestration** category includes:
- DAG
- steps
- suspend

## Container template
```
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: container-
spec:
  entrypoint: main
  templates:
  - name: main
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["hello world"]
```

The **template tags** (aka **template variables**) are used to substitute data in the workflow at runtime.

There are global variables, like **workflow.name**.

For example submit this job:
```
cat <<EOF > template-tag-workflow.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: template-tag-
spec:
  entrypoint: main
  templates:
    - name: main
      container:
        image: docker/whalesay
        command: [cowsay]
        args: ["hello {{workflow.name}}"]
EOF
```

Submit the job:
```
argo submit --watch template-tag-workflow.yaml
```

Watch the logs:
```
argo logs @latest
```

And the output shows the random name assigned to the job:
```
template-tag-n58nn:  __________________________ 
template-tag-n58nn: < hello template-tag-n58nn >
...
```

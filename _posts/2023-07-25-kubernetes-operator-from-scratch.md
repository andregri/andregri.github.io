---
layout: single
title: Kubernetes operator from scratch
toc: true
tags: kubernetes operator go
---
A Kubernetes operator is an extension of Kubernetes that allows to use custom resources to manage applications and their components. Writing a proper Kubernetes operator from scratch isn't a trivial task. For this reason, there exists SDKs and frameworks that take care of boiler plate code and allow developers to focus on the business logic. However, if you don't have a good foundation of the Kubernetes internals, the SDKs and frameworks could add a layer of complexity instead of removing it. The book “Kubernetes in action” by Marko Luksa introduces the Kubernetes operators from the basics, such as the controller loop. He avoids to use external libraries that would add complexity for who is approaching operators for the first time. The resulting operator surely is not be production-ready, but is the first step of the learning journey in the extending Kubernetes.

At the moment of writing this post, the latest version of Kubernetes is v1.27.

# Custom Resource Definition

When developing an operator, the first step is writing a Custom Resource Definition (CRD) which is controlled by the operator. Like in the “Kubernetes in Action” book, the Custom Resource is called Website that requires just 2 parameters:

- the name of the resource
- the url of the public repository where the website is hosted.

The whole Custom Resource Definition is:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.extensions.andregri.com
spec:
  scope: Namespaced
  group: extensions.andregri.com
  names:
    kind: Website
    singular: website
    plural: websites
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              gitRepo:
                type: string
```

## apiVersion and Kind

- the **apiVersion** of CRDs is `apiextensions.k8s.io/v1`
- the **kind** of CRDs is `CustomResourceDefinition`

## metadata

- the **name** of the CRD is `websites.extensions.andregri.com`

## spec

- the **scope** to specify if the CRD is namespaced or not: in this case it is namespaced `scope: Namespaced`
- the **group** which our CRD belongs that is used in the apiVersion field when creating a resource of type Website `group: extensions.andregri.com`
- the **names** field specifies how to call the Custom Resource: the kind needed when writing a manifest and the names needed by kubectl:
    
    ```yaml
    names:
      kind: Website
      singular: website
      plural: websites
    ```
    
- the **versions** field contains a list of versions and the related schema in OpenAPIV3 format; in this case the schema specifies a spec object that contains a gitRepo property of type string:
    
    ```yaml
    versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                gitRepo:
                  type: string
    ```
    

# Use the Custom Resource Definition

To use this Custom Resource Definition, write the group `extensions.andregri.com` and version `v1` in the **apiVersion** field and `Website` in the **kind** field:

```yaml
apiVersion: extensions.andregri.com/v1
kind: Website
metadata:
  name: kubia
spec:
  gitRepo: https://github.com/luksa/kubia-website-example.git
```

# The controller

The objective of the controller is to watch the API server for events related to Website resources, in particular events of type `ADDED` and `DELETED`. The API path is:

```
apis/extensions.andregri.com/v1/websites?watch=true
```

When the controller receives the event of type ADDED, it creates the application components: a Service and a Deployment.

When the controller receives the event of type DELETED, it deletes all the application components.

The core of this basic controller is:

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"strings"
	"text/template"
	"time"
)

type Metadata struct {
	Name      string `json:"name"`
	Namespace string `json:"namespace"`
}

type WebsiteSpec struct {
	GitRepo string `json:"gitRepo"`
}

type Website struct {
	Metadata Metadata    `json:"metadata"`
	Spec     WebsiteSpec `json:"spec"`
}

type WebsiteWatchEvent struct {
	Type   string  `json:"type"`
	Object Website `json:"object"`
}

func main() {
	log.Println("starting controller")
	watchUrl := "http://localhost:8001/apis/extensions.andregri.com/v1/namespaces/default/websites?watch=true"
	resp, err := http.Get(watchUrl)
	if err != nil {
		log.Panic(err)
	}
	defer resp.Body.Close()

	decoder := json.NewDecoder(resp.Body)
	for {
		var event WebsiteWatchEvent
		err = decoder.Decode(&event)
		if err != nil {
			log.Panic(err)
		}
		log.Printf("received watch event of type: %s website %q: %s", event.Type, event.Object.Metadata.Name, event.Object.Spec.GitRepo)

		if event.Type == "ADDED" {
			createWebsite(event.Object)
		} else if event.Type == "DELETED" {
			deleteWebsite(event.Object)
		}

		// Add a small delay to avoid high CPU usage.
		time.Sleep(1 * time.Second)
	}
}
```

## Watch the resource events from API Server

This snippet of Go code uses the http client to watch the API Server for events related to the websites resources:

```go
watchUrl := "http://localhost:8001/apis/extensions.andregri.com/v1/namespaces/default/websites?watch=true"
resp, err := http.Get(watchUrl)
if err != nil {
	log.Panic(err)
}
defer resp.Body.Close()
```

## Read the type of the event

The controller receives the events in an infinite loop and decode the JSON response:

```go
decoder := json.NewDecoder(resp.Body)
for {
	var event WebsiteWatchEvent
	err = decoder.Decode(&event)
	if err != nil {
		log.Panic(err)
	}
	log.Printf("received watch event of type: %s website %q: %s", event.Type, event.Object.Metadata.Name, event.Object.Spec.GitRepo)
...
```

Then it reads the type of the event to control the Website resource:

```go
if event.Type == "ADDED" {
	createWebsite(event.Object)
} else if event.Type == "DELETED" {
	deleteWebsite(event.Object)
}
```

## Create the custom resource

The function `createWebsite` creates the components (Service and Deployment) of the Website resource:

```go
func createWebsite(data Website) {
	svc, err := parseTemplate("service", data)
	if err != nil {
		log.Panic(err)
	}
	createResource("api/v1", "services", data.Metadata.Namespace, svc)

	depl, err := parseTemplate("deployment", data)
	if err != nil {
		log.Panic(err)
	}
	createResource("apis/apps/v1", "deployments", data.Metadata.Namespace, depl)
}
```

For the sake of simplicity, the controller reads the resource manifests from template files using the `text/template` package and replaces the variables using `data` structure:

```go
svc, err := parseTemplate("service", data)
```

For instance, the Service template is:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Metadata.Name }}
  namespace: {{ .Metadata.Namespace }}
  labels:
    app: {{ .Metadata.Name }}
spec:
  selector:
    app: {{ .Metadata.Name }}
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 80
```

When the template is processed and a complete manifest of the service is created, the controller creates the resource:

```go
createResource("api/v1", "services", data.Metadata.Namespace, svc)
```

To create the resource from the yaml manifest, the controller sends a POST http request to the API server whose body is the yaml manifest:

```go
func createResource(apiGroup, kind, namespace, body string) error {
	url := fmt.Sprintf("http://localhost:8001/%s/namespaces/%s/%s", apiGroup, namespace, kind)
	resp, err := http.Post(url, "application/yaml", strings.NewReader(body))
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	log.Printf("response status to create resource %s/%s: %s", apiGroup, kind, resp.Status)

	return err
}
```

The steps are identical for the Deployment resource.

## Destroy the custom resource

The function deleteWebsite is called when an event of type DELETED is received by the API Server:

```go
func deleteWebsite(data Website) {
	if err := deleteResource("api/v1", "services", data.Metadata.Namespace, data.Metadata.Name); err != nil {
		log.Panic(err)
	}
	if err := deleteResource("apis/apps/v1", "deployments", data.Metadata.Namespace, data.Metadata.Name); err != nil {
		log.Panic(err)
	}
}
```

It just sends the DELETE http requests to the API Server to delete the Service and the Deployment:

```go
func deleteResource(apiGroup, kind, namespace, resource string) error {
	url := fmt.Sprintf("http://localhost:8001/%s/namespaces/%s/%s/%s", apiGroup, namespace, kind, resource)
	req, err := http.NewRequest(http.MethodDelete, url, nil)
	if err != nil {
		return err
	}

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	log.Printf("response status to delete resource %s/%s/%s: %s", apiGroup, kind, resource, resp.Status)

	return err
}
```

# Conclusion

I had hard times approaching the Kubernetes Operator at the beginning but I found the “Kubernetes in action” method successful : focus on the main basic concepts (CRD definition, API Server Events, Controller Loop) and disregard, for the moment, the complexity of complete libraries (like kubebuilder) and frameworks.

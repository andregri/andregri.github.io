---
layout: single
title: Change Kubernetes service type from LoadBalancer to NodePort and viceversa
toc: true
tags: kubernetes
---

Changing the Service type from NodePort to LoadBalancer and viceversa is useful depending on the cluster you are working on. If you are doing experiments in a local development cluster, like Kind, you don't have a real load balancer in front of your cluster and you need to expose your services externally using a NodePort service. On contrary, if you are working on a kubernetes cluster with a proper load balancer, you can expose your services through a LoadBalancer service type that uses the load balancer domain name and port.

## Convert to a LoadBalancer service

It is very likely to have a load balancer in front of your kubernetes services when you are working in a production cluster or on a kubernetes cluster hosted by a cloud provider. The load balancer is useful to expose your services using a single domain name, the load balancer domain name.

The command to convert a service type to a LoadBalancer service type is:

```bash
kubectl patch svc <service-name> -n <namespace> -p '{"spec": {"type": "LoadBalancer"}}'
```

## Convert to a NodePort service

If you are working on a development kubernetes cluster you probably don't have a load balancer to forward the incoming requests to the services. To connect to your services you have to expose them using a Node Port service type that open the same port on each node of the cluster. So to connect to the service in a multi-node cluster, you type the name of a node and the exposed port. Compared to a load balancer, you should know the node names before connecting to the service.

The command to convert a service type to a NodePort service type is:

```bash
kubectl patch svc <service-name> -n <namespace> -p '{"spec": {"type": "NodePort"}}'
```

## kubectl port-forward to expose a service locally

kubectl tool contains the port-forward command to expose a service locally on the port we choose. It means you can connect to the service using `localhost:port`.

The command to expose the service locally is:

```bash
kubectl port-forward svc/<service-name> <local-port>:<service-port>
```

For instance, if you have the service frontend on port 80, to expose it locally on port 8080:

```jsx
kubectl port-forward svc/frontend 8080:80
```

The `kubectl port-forward` is a process running on the foreground. So you need to open a new terminal if you want to issue new commands.

The [Argo CD Getting Started guide](https://argo-cd.readthedocs.io/en/stable/getting_started/#3-access-the-argo-cd-api-server) shows different methods to access the Argo CD Api Server: using a LoadBalancer service type or a exposing the service locally using `kubectl port-forward`.

I personally used the combination of a NodePort service and kubectl port-forward command when working with Istio Gateways. Istio creates a LoadbBalancer service when you create a Gateway. Since I was working in a Kind cluster, I first changed the service type and then I exposed it locally.

## Conclusion

Depending if you a have load balancer or not, you may need to change service type to connect to them.

Discover more posts about [Kubernetes](https://andregri.com/tags/#kubernetes)!

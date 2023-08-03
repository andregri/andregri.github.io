---
layout: single
title: Kubernetes ephemeral containers
toc: true
tags: kubernetes
---
Removing building and debugging tools from the images we use to run services on containers is one of the security best practices. However, you can't debug a container if it is not shipped with the debugging tools. Security and easy of debug seems to mutual exclusive but Kubernetes provides a solution: **ephemeralContainers**. As the name suggests, they are ephemeral containers attached to the same pod we want to debug and they share the same linux process namespace. The ephemeral container contains all the tools needed to troubleshoot the main container, but it is stopped as soon as we ended our debugging session.

At the time of writing this post, the latest version of Kubernetes is 1.27. 

# Demo

Create a pod with the python image (Alpine flavoured) that runs an http server on the port 8080. This image doesn't provide tools like curl.

Python allows run a http server on port 8080 in one line: `python3 -m http.server 8080`.

If you try to test the server with the command `curl`, it is not found:

```console
$ kubectl run server --image=python:3.12.0b4-alpine3.18 -- python3 -m http.server 8080
If you don't see a command prompt, try pressing enter.
/ # curl
/bin/sh: curl: not found
/ # exit
```

`kubectl debug` command allows to create an ephemeral container in the same pod we want to debug. Using the image `curlimages/curl` and curl the python http server:

```console
$ kubectl debug -it pod/server --image=curlimages/curl -- /bin/sh
Defaulting debug container name to debugger-t492c.
If you don't see a command prompt, try pressing enter.
~ $ curl localhost:8080
<!DOCTYPE HTML>
...
</html>
~ $ exit
Session ended, the ephemeral container will not be restarted but may be reattached using 'kubectl attach server -c debugger-t492c -i -t' if it is still running
```

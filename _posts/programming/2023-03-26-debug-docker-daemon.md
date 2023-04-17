---
layout: single
title: Debug docker daemon - docker container ls vs docker system df
tags: docker go
---

This post is a report of my troubleshooting process of docker daemon.

# Problem

I disabled Kubernetes from Docker desktop.

- Output of `docker container ps -a`

```bash
CONTAINER ID   IMAGE                   COMMAND                  CREATED       STATUS          PORTS                       NAMES
f9c486446ba4   kindest/node:v1.19.16   "/usr/local/bin/entr…"   13 days ago   Up 46 minutes   127.0.0.1:50057->6443/tcp   kind-control-plane
```

- Output of `docker system df`

```bash
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          31        27        5.324GB   2.274GB (42%)
Containers      85        1         3.672MB   315.3kB (8%)
Local Volumes   11        2         2.211GB   169.5MB (7%)
Build Cache     54        0         397.8MB   397.8MB
```

The 2 commands shows inconsistent results:

- for docker container there is 1 container
- for docker system there are 85 containers

# Investigation

## docker-cli repository

[https://github.com/docker/cli](https://github.com/docker/cli)

I inspected the code of docker cli to understand the logic of the commands:

- From **vendor/github.com/docker/docker/client/container_list.go**, `docker container ps -a` calls the API `resp, err := cli.get(ctx, "/containers/json", query, nil)`
- From **vendor/github.com/docker/docker/client/disk_usage.go**, `docker system df` calls the API `serverResp, err := cli.get(ctx, "/system/df", query, nil)`

However, the commands calls the API of Docker so it is need to look at the docker client.

## docker (aka moby) repository

[https://github.com/moby/moby](https://github.com/moby/moby)

commit 17d94cf3b901f52a70c268c8971b8887baa215e4

- From **api/server/router/container/container.go**, I find that **containers/json** path call the function `getContainersJSON`: `router.NewGetRoute("/containers/json", r.getContainersJSON)`
- From **api/server/router/container/container_routes.go**, I find that it calls the `Containers` function from the backend: `containers, err := s.backend.Containers(ctx, config)`:
- 

```go
// code before...
config := &types.ContainerListOptions{
		All:     httputils.BoolValue(r, "all"),
		Size:    httputils.BoolValue(r, "size"),
		Since:   r.Form.Get("since"),
		Before:  r.Form.Get("before"),
		Filters: filter,
	}

	if tmpLimit := r.Form.Get("limit"); tmpLimit != "" {
		limit, err := strconv.Atoi(tmpLimit)
		if err != nil {
			return err
		}
		config.Limit = limit
	}

	containers, err := s.backend.Containers(ctx, config)
	if err != nil {
		return err
	}
// code after...
```

- From **daemon/list.go**, I find that `Containers` calls the `reduceContainers` function

```go
// reduceContainers parses the user's filtering options and generates the list of containers to return based on a reducer.
func (daemon *Daemon) reduceContainers(ctx context.Context, config *types.ContainerListOptions, reducer containerReducer) ([]*types.Container, error) {
	if err := config.Filters.Validate(acceptedPsFilterTags); err != nil {
		return nil, err
	}

	var (
		view       = daemon.containersReplica.Snapshot()
		containers = []*types.Container{}
	)

	filter, err := daemon.foldFilter(ctx, view, config)
	if err != nil {
		return nil, err
	}

	// fastpath to only look at a subset of containers if specific name
	// or ID matches were provided by the user--otherwise we potentially
	// end up querying many more containers than intended
	containerList, err := daemon.filterByNameIDMatches(view, filter)
	if err != nil {
		return nil, err
	}

	for i := range containerList {
		t, err := daemon.reducePsContainer(ctx, &containerList[i], filter, reducer)
		if err != nil {
			if err != errStopIteration {
				return nil, err
			}
			break
		}
		if t != nil {
			containers = append(containers, t)
			filter.idx++
		}
	}

	return containers, nil
}
```
    
- From **api/server/router/system/system.go**, I find that `getDiskUsage` is the handler of **/system/df** path: `router.NewGetRoute("/system/df", r.getDiskUsage),`
- From **api/server/router/system/system_routes.go**, I find the `getDiskUsage` definition that calls the `SystemDiskUsage` from the backend:

```go
systemDiskUsage, err = s.backend.SystemDiskUsage(ctx, DiskUsageOptions{
				Containers: getContainers,
				Images:     getImages,
				Volumes:    getVolumes,
			})
```

- From **daemon/disk_usage.go**, I find the `SystemDiskUsage` function that calls the `containerDiskUsage` function: `containers, err = daemon.containerDiskUsage(ctx)`
- From the definition I find out that containerDiskUsage calls the `daemon.Containers`, the same as before:

```go
// containerDiskUsage obtains information about container data disk usage
// and makes sure that only one calculation is performed at the same time.
func (daemon *Daemon) containerDiskUsage(ctx context.Context) ([]*types.Container, error) {
	res, _, err := daemon.usageContainers.Do(ctx, struct{}{}, func(ctx context.Context) ([]*types.Container, error) {
		// Retrieve container list
		containers, err := daemon.Containers(ctx, &types.ContainerListOptions{
			Size: true,
			All:  true,
		})
		if err != nil {
			return nil, fmt.Errorf("failed to retrieve container list: %v", err)
		}
		return containers, nil
	})
	return res, err
}
```

So we find that the two paths use the same function Containers but we need to check if they use the same configuration options.

## Inspect Docker API calls

On Mac, logs are located at **$HOME/Library/Containers/com.docker.docker/Data/log/host** where you find files like **Docker.log** and **com.docker.backend.log** where you find the API calls received.

```go
curl --unix-socket /var/run/docker.sock "http://localhost/v1.41/containers/json?size=1"
```

To debug the API calls that the CLI does to the Docker daemon, you can inspect log file but you will see just the route. However, we know the route but we want to inspect the request parameters.

From this Github issue [https://github.com/moby/moby/issues/11865](https://github.com/moby/moby/issues/11865) you can  intercept http requests using the **socat** tool:

- `docker container ls -a` command maps to **GET /v1.41/containers/json?all=1 HTTP/1.1\r**
- `docker system df` maps to **GET /v1.41/system/df HTTP/1.1\r**

However from the HTTP requests you don’t have much informations on what the daemon is doing.

## Recompile docker daemon

To get more details on docker daemon logic, I followed very detailed official instruction to modify and rebuild dockerd. However, it is possible to compile only for linux platforms and since the development environment is running on a container, you can’t use the same docker socket.

After digging into docker (or moby) source code repository I didn’t find a method to compile dockerd for the architecture **darwin/amd64**. Actually, Docker for Mac run on a Linux Virtual Machine.

My second idea was to import docker data from my Mac to Linux container, but Docker data are stored in a unique and heavy file, Docker.raw, stored at **$HOME/Library/Containers/com.docker.docker/Data/vms/0/data**.
    

# Conclusions

- Docker for Mac is a Linux Virtual Machine under the hood.
- It is a difficult task to debug an error in Docker Desktop since I don’t know how to use existing data but you must be able to reproduce the error in other environments as well.
- On the contrary it is easy to modify and test the docker daemon in a development environment.

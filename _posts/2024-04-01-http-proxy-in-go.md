---
layout: single
title: Implementing an HTTP proxy in Go
toc: true
tags: go
---

I was looking for a simple project to get my hands dirty with Go again and I came up with the HTTP proxy server.
At a first glance, it seems a trivial project:
- create an HTTP server
- read the HTTP method, the URL, and the HOST from the request
- send an identical request to the target server.

As far as the client request is plain HTTP, these steps work fine.
The client sends a GET request to the proxy server and the proxy server replicates the GET request to the target server.

When the proxy server receives an HTTPS request, these steps doesn't work because communication between cient and target server are protected by TLS.
This time the client doesn't send a GET request to the proxy server.
Instead, it sends a CONNECT request to the proxy server to tunnel a bidirectional connection between the client and the target server.

To make a proper man-in-the-middle proxy server, it should generate certificates on-the-fly for the client and establish a different HTTPS connection with the target server.
This blog post ["How mitmproxy works"](https://corte.si/posts/code/mitmproxy/howitworks/index.html) explains in details how to implement it.

At the moment, I implemented a proxy server that just tunnels the https connection between the endpoints.
My proxy server is based on the code listed in the blog post ["Proxy in golang in less than 100 lines of code"](https://medium.com/@mlowicki/http-s-proxy-in-golang-in-less-than-100-lines-of-code-6a51c2f2c38c).

In the first iteration, I used the default HTTP handler provided by the "http" package, that is the [ServeMux](https://cs.opensource.google/go/go/+/refs/tags/go1.22.1:src/net/http/server.go;l=2432).
```go
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		req, err := http.NewRequest(r.Method, r.URL.String(), r.Body)
		if err != nil {
			log.Printf("Error during NewRequest() %s: %s\n", r.URL.String(), err)
			w.WriteHeader(http.StatusInternalServerError)
			return
		}

		// copy headers
		for key, values := range r.Header {
			for _, value := range values {
				req.Header.Add(key, value)
			}
		}

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			log.Printf("Error during Do() %s: %s\n", r.URL.String(), err)
			w.WriteHeader(http.StatusInternalServerError)
			return
		}
		defer resp.Body.Close()

		written, err := io.Copy(w, resp.Body)
		if err != nil {
			log.Printf("Error during Copy() %s: %s\n", r.URL.String(), err)
			w.WriteHeader(http.StatusInternalServerError)
			return
		}

		log.Printf("%s - %s - %s - %d - %dKB\n", r.Proto, r.Method, r.Host, resp.StatusCode, written/1000)
	})
```

This ServeMux handler matches the Method, the Host, and the Path of the HTTP request against a list of patterns and calls the function associated to that pattern.
In case of CONNECT requests, it redirects the connection to the same host and path specified in the request (see source code)[https://cs.opensource.google/go/go/+/refs/tags/go1.22.1:src/net/http/server.go;l=2522].
For example, if the client sends the requests https://google.com, the proxy server receives an HTTP request such as:
- method: CONNECT
- host: google.com
- path: /

The ServeMux handler supports the pattern `[METHOD ][HOST]/[PATH]`
Since my pattern is "/", the host "google.com" doesn't match.

To proxy HTTPS requests corretly, the proxy server must distinguish the incoming requests based on the HTTP method:
- if the method is `http.MethodConnect`, then tunnel the connection
- otherwise send a new request to the target server.
The custom handler to split that splits the logic is:
```go
log.Fatal(http.ListenAndServe(
  ":8080",
  http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodConnect {
      handleTunnel(w, r)
    } else {
      handleHTTP(w, r)
    }
  })),
)
```

The function `handleTunnel` that handles the HTTPS request is doing the following:
- open a TCP connection `dest_conn` from the proxy server to the target server
- if the server accepts, then the proxy will return the status code 200 to the client
- get the connection `src_conn` from the proxy server to the client using the `http.Hijacker`. `Hijack` gets the TCP connection from the `ResponseWriter`
- proxy the connections: copy the data from one connection to the other and viceversa
The code that implements the tunneling is:
```go
func handleTunnel(w http.ResponseWriter, r *http.Request) {
	dest_conn, err := net.DialTimeout("tcp", r.Host, 10*time.Second)
	if err != nil {
		http.Error(w, err.Error(), http.StatusServiceUnavailable)
		return
	}
	defer dest_conn.Close()
	w.WriteHeader(http.StatusOK)

	hj, ok := w.(http.Hijacker)
	if !ok {
		http.Error(w, "webserver doesn't support hijacking", http.StatusInternalServerError)
		return
	}
	src_conn, _, err := hj.Hijack()
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	defer src_conn.Close()

	srcConnStr := fmt.Sprintf("%s->%s", src_conn.LocalAddr().String(), src_conn.RemoteAddr().String())
	dstConnStr := fmt.Sprintf("%s->%s", dest_conn.LocalAddr().String(), dest_conn.RemoteAddr().String())

	log.Printf("%s - %s - %s\n", r.Proto, r.Method, r.Host)
	log.Printf("src_conn: %s - dst_conn: %s\n", srcConnStr, dstConnStr)

	var wg sync.WaitGroup

	wg.Add(2)
	go transfer(&wg, dest_conn, src_conn, dstConnStr, srcConnStr)
	go transfer(&wg, src_conn, dest_conn, srcConnStr, dstConnStr)
	wg.Wait()
}

func transfer(wg *sync.WaitGroup, destination io.Writer, source io.Reader, destName, srcName string) {
	defer wg.Done()
	written, err := io.Copy(destination, source)
	if err != nil {
		fmt.Printf("Error during copy from %s to %s: %v\n", srcName, destName, err)
	}
	log.Printf("copied %d bytes from %s to %s\n", written, srcName, destName)
}
```

Online I found many discussions about closing the connections but after some tests I guess it is important to close them once after that all streams are copied.
The original implementation didn't use the `sync.WaitGroup` to wait for the `Copy` functions to be completed and closed both streamer twice.
This produced an error during `io.Copy`, though the proxy worked correctly.
I preferred to defer the close of connection in the parent of `transfer` function to call the `Close` function once.

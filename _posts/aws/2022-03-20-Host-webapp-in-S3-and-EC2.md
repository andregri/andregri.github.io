---
layout: single
title: Host a Go Gin web application with AWS S3 and EC2
tags: aws cors
---

Few months ago I wrote a REST API service with Go and the
[Gin Web Framework](https://github.com/gin-gonic/gin) that performs basic CRUD
operations on PostgreSQL database. I was using it through `curl` and Postman but
I wanted the freedom to call the API from every device and location. So, I
needed a web page to make HTTP requests from a smartphone as well and a machine
running 24/7.

My first idea was to use S3 to host my single HTML document and EC2 to run the
backend and the dbms. I chose S3 because I didn't want my backend service to
serve frontend document as well. I chose EC2 because its simplicity to be set up
even though I was neglecting scalability. However, this REST API is used only by
me and scalability is not necessary at the moment.

I did not take into account an important issue: **cors**, also known as
**Cross-Origin Resource Sharing**.

First signals of this issue appeared during development on my local machine.
The web server was up and running locally and I could send GET, POST,
etc. requests from curl. I tried to make the same exact request with the
`fetch-api` from the console log of Firefox, but this time I was receiving an
error: *TypeError: NetworkError when attempting to fetch resource*. The web was
claiming that it was related to cors. After some research and trials I asked on
[Stackoverflow](https://stackoverflow.com/questions/71516936/go-gin-fetch-data-from-local-browser/71527555#71527555).
The HTTP Content-Security-Policy was blocking my requests because I had to make
them from the same domain of the backend even if it was **localhost**.

Once the local development issue was understood, I could try the service on the
public Internet: uploading the HTML file to S3 and running the web server on EC2.
The POST request is working, while the GET request fails. This time, the browser
explicitly stated a cors misconfiguration.
The POST request was working because I was just pushing data to the server,
without receiving any response back.
The GET request was failing because the server could not send any response out
of its domain, in this case the S3 domain where the web page was hosted.

Gin is extended with a separate [go package](https://github.com/gin-contrib/cors)
to handle cors headers in HTTP. I specified the exact domains that are allowed,
instead of allowing all origins using the wildcard *. Two lines of code are enough:
```go
config := cors.DefaultConfig()
config.AllowOrigins = []string{"S3 domain", "my domain"}
```

For concluding this AWS architecture consisting in hosting HTML documents on S3
and web servers on EC2 is tricky because you need to properly set up HTTP header
to support cors. Apart from that it exploits the performance of S3 (and CloudFront
in more sophisticated architectures) to retrieve HTML pages.

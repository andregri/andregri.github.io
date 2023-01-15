---
layout: single
title: Inspecting HTTP Cache-Control
tags: aws httpd
---

I was following the ACloudGuru tutorial to setup a web server with httpd on a EC2 instance and a AWS Cognito Identity Pool. 

I wanted to update the `index.html` file but on the browser I still got the old file. I blamed **httpd** not using the most recent html file and I restarted the service several times. However, running `curl localhost` on the web server machine, returned the most updated page. It meant that httpd was loading the most updated page.

I refreshed the page many time making sure to press the **SHIFT** key to prevent cached result, but yet I got the old page.

Is it due to AWS that is keeping a cache of my requests preventing me from retrieving the new page? It is akward, because I have never encountered such a behaviour. Moreover I never heard of a cache invalidation for a EC2 instance or a VPC network.

Is it due to a mismatch between timezones since the EC2 instance is hosted in North Virginia? I changed the timezone with `sudo timedatectl set-timezone UTC` but nothing changes.

# Inspecting HTTP requests

Without getting any result, I decided to inspect the HTTP requests and to send custom requests using the Firefox Developer Tools.

The current state of `/var/www/html/index.html` is (timezone is **UTC**):

```yaml
ls -l
total 584
-rw-r--r-- 1 root root 590557 Jun 19 10:54 fact.jpg
-rw-r--r-- 1 root root   1533 Jun 19 12:30 index.html
```

I open a new **private** **tab** in **Firefox** and I enter `http://<ip address of ec2 instance>`. The request is set at 12:50 GMT:

```yaml
GET / HTTP/1.1
Host: 18.208.106.244
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:101.0) Gecko/20100101 Firefox/101.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-GB,en;q=0.5
Accept-Encoding: gzip, deflate
DNT: 1
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Pragma: no-cache
Cache-Control: no-cache
```

The **response** is:

```yaml
HTTP/1.1 200 OK
Date: Sun, 19 Jun 2022 12:50:11 GMT
Server: Apache/2.4.53 ()
Last-Modified: Sun, 19 Jun 2022 12:30:51 GMT
ETag: "5fd-5e1cc286fa77b"
Accept-Ranges: bytes
Content-Type: text/html; charset=UTF-8
Content-Encoding: gzip
Content-Length: 879
Connection: keep-alive
```

The `Date` parameter is populated with the request date and time.

The `Last-Modified` parameter has the last change date of `index.html`.

Now, I update the `/var/www/html/index.html` file, so the current state of the folder is:

```yaml
ls -l
total 584
-rw-r--r-- 1 root root 590557 Jun 19 10:54 fact.jpg
-rw-r--r-- 1 root root   1533 Jun 19 12:55 index.html
```

I open a private tab in Firefox and request the web server ip address again, but this time I hold the **SHIFT** key before hitting the *refresh* button on the browser:

The **request** is sent at 12:57 GMT:

```yaml
GET / HTTP/1.1
Host: 18.208.106.244
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:101.0) Gecko/20100101 Firefox/101.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-GB,en;q=0.5
Accept-Encoding: gzip, deflate
DNT: 1
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Pragma: no-cache
Cache-Control: no-cache
```

and the **response** is:

```yaml
HTTP/1.1 200 OK
Date: Sun, 19 Jun 2022 12:50:11 GMT
Server: Apache/2.4.53 ()
Last-Modified: Sun, 19 Jun 2022 12:30:51 GMT
ETag: "5fd-5e1cc286fa77b"
Accept-Ranges: bytes
Content-Type: text/html; charset=UTF-8
Content-Encoding: gzip
Content-Length: 879
Connection: keep-alive
```

Note that `Date` and `Last-Modified` didn’t change from the previous request, even though the `Cache-Control` header is `no-cache`.

Now I edit the **request** headers to for the `Cache-Control: max-age=0` (13:03 GMT):

```yaml
GET / HTTP/1.1
Host: 18.208.106.244
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:101.0) Gecko/20100101 Firefox/101.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-GB,en;q=0.5
Accept-Encoding: gzip, deflate
DNT: 1
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Cache-Control: max-age=0, no-cache
Pragma: no-cache
```

But Firefox still adds the `no-cache` and the `Pragma: no-cache` header.

However, the **response** is:

```yaml
HTTP/1.1 200 OK
Date: Sun, 19 Jun 2022 13:03:02 GMT
Server: Apache/2.4.53 ()
Last-Modified: Sun, 19 Jun 2022 12:55:35 GMT
ETag: "5fd-5e1cc80debb24"
Accept-Ranges: bytes
Content-Type: text/html; charset=UTF-8
Content-Encoding: gzip
Content-Length: 878
Connection: keep-alive
```

Forcing `Cache-Control: max-age=0` returns the updated files because the `Last-Modified` header is the same as the modification date of `index.html` on Linux.

# But how does `Cache-Control` work?

## `no-cache`

According to MDN ([https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control#no-cache_2](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control#no-cache_2)) for `no-cache`:

> The `no-cache` request directive asks caches to validate the response with the origin server before reuse.
> 
> 
> ```
> Cache-Control: no-cache
> ```
> 
> `no-cache` allows clients to request the most up-to-date response even if the cache has a [fresh](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching#freshness) response.
> 
> Browsers usually add `no-cache` to requests when users are **force reloading** a page.
> 

## `max-age`

According to MDN ([https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control#max-age_2](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control#max-age_2)) for `max-age`:

> The `max-age=N` request directive indicates that the client allows a stored response that is generated on the origin server within *N* seconds — where *N* may be any non-negative integer (including `0`).
> 
> 
> ```
> Cache-Control: max-age=3600
> ```
> 
> In the case above, if the response with `Cache-Control: max-age=604800` was generated more than 3 hours ago (calculated from `max-age` and the `Age` header), the cache couldn't reuse that response.
> 
> Many browsers use this directive for **reloading**, as explained below.
> 
> ```
> Cache-Control: max-age=0
> ```
> 
> `max-age=0` is a workaround for `no-cache`, because many old (HTTP/1.0) cache implementations don't support `no-cache`. Recently browsers are still using `max-age=0` in "reloading" — for backward compatibility — and alternatively using `no-cache` to cause a "force reloading".
> 
> If the `max-age` value isn't non-negative (for example, `-1`) or isn't an integer (for example, `3599.99`), then the caching behavior is undefined. However, the [Calculating Freshness Lifetime](https://httpwg.org/specs/rfc7234.html#calculating.freshness.lifetime) section of the HTTP specification states:
> 
> > Caches are encouraged to consider responses that have invalid freshness information to be stale.
> > 
> 
> In other words, for any `max-age` value that isn't an integer or isn't non-negative, the caching behavior that's encouraged is to treat the value as if it were `0`.
> 

# Conclusion

It seems that httpd returns the cached result when `Cache-Control: no-cache`. To invalidate cache instead, `Cache-Control: max-age=0` is needed.

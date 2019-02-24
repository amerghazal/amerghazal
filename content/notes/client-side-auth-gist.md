---
title: "Client Side Auth Gist"
date: 2019-02-24T00:13:33-06:00
categories:
- code
- examples
tags:
- golang
- ssl
- certificates
- auth
---

Assuming you have client/server certificates already provisioned, this gist walks you through setting up a client-cert auth enforcing server with gin/gonic.

<!--more-->

If you're unfamiliar with this auth flow, here's what it generally looks like:

![cliencert auth flow](/images/clientcertauth.png)

Run Server

```
% go run server.go &
[2] 72578
```

without client cert
```
% curl -k https://localhost:8080/ping
2018/12/15 23:52:54 http: TLS handshake error from [::1]:62141: tls: client didn't provide a certificate
curl: (35) error:14094412:SSL routines:SSL3_READ_BYTES:sslv3 alert bad certificate
```

with client cert

```
% curl -v -k \
--cacert certs/ca.pem \
--key certs/client.key \
--cert certs/client.pem \
https://localhost:8080/ping
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8080 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: certs/ca.pem
  CApath: none
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Request CERT (13):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Certificate (11):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS handshake, CERT verify (15):
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
...
...
> GET /ping HTTP/2
> Host: localhost:8080
> User-Agent: curl/7.54.0
> Accept: */*
>
* Connection state changed (MAX_CONCURRENT_STREAMS updated)!
[GIN] 2018/12/15 - 23:49:56 | 200 |      67.769µs |             ::1 | GET      /ping
< HTTP/2 200
< content-type: application/json; charset=utf-8
< content-length: 18
< date: Sun, 16 Dec 2018 05:49:56 GMT
<
* Connection #0 to host localhost left intact
{"message":"pong"}

```

server.go

```
package main

import (
    "crypto/tls"
    "crypto/x509"
    "github.com/gin-gonic/gin"
    "io/ioutil"
    "log"
    "net/http"
)

func main() {
    cert, _ := tls.LoadX509KeyPair("certs/server.pem", "certs/server.key")
    certpool := x509.NewCertPool()
    capem, _ := ioutil.ReadFile("certs/ca.pem")
    if !certpool.AppendCertsFromPEM(capem) {
        log.Fatalf("Can't parse client certificate authority")
    }
    tlsConfig := tls.Config{
        Certificates: []tls.Certificate{cert},
        ClientAuth:   tls.RequireAndVerifyClientCert,
        ClientCAs:    certpool,
    }
    r := gin.Default()
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(200, gin.H{
            "message": "pong",
        })
    })
    server := http.Server{
        Addr:      ":8080",
        Handler:   r,
        TLSConfig: &tlsConfig,
    }
    _ = server.ListenAndServeTLS("", "")
}

```


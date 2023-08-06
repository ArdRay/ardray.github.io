---
layout: post
title:  "Nextcloud: Your web server is not properly set up to resolve..."
date:   2023-08-01
---

## Issue

Nextcloud: 27.0.0

```
Your web server is not properly set up to resolve "/.well-known/webfinger". Further information can be found in the documentation.
Your web server is not properly set up to resolve "/.well-known/nodeinfo". Further information can be found in the documentation.
```

## Configuration

HAproxy: 2.9

```
frontend fe_https
    bind :443 ssl crt /usr/local/etc/certs alpn h2,http/1.1

    http-request capture req.hdrs len 512
    log-format "%ci:%cp [%tr] %ft [[%hr]] %hs %{+Q}r"

    acl host-nextcloud hdr(host) -i cloud.domain.cloud
    acl url_discovery path_beg -i /.well-known/caldav /.well-known/carddav
    acl url_webfinger path_beg -i /.well-known/webfinger
    acl url_nodeinfo path_beg -i /.well-known/nodeinfo

    http-request redirect location https://%[hdr(host)]/remote.php/dav/ code 301 if url_discovery host-nextcloud
    http-request redirect location https://%[hdr(host)]/index.php/.well-known/webfinger code 301 if url_webfinger host-nextcloud
    http-request redirect location https://%[hdr(host)]/index.php/.well-known/nodeinfo code 301 if url_nodeinfo host-nextcloud

    http-response set-header Strict-Transport-Security "max-age=16000000; includeSubDomains;"

    use_backend be_nextcloud if host-nextcloud

    default_backend no-match

backend be_nextcloud
    http-request set-header Connection keep-alive
	http-request set-header Host cloud.domain.cloud
	http-request set-header X-Forwarded-Proto https
    option httpchk GET /
	http-check send hdr Host cloud.domain.cloud
	http-check expect status 200,302
    server nextcloud nextcloud:80 check resolvers docker_resolver
```
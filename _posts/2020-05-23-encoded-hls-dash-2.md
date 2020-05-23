---
layout: post
title: Encoded HLS/DASH with clear keys [2]
categories:
  - html5-video
tags:
  - nginx
  - HLS
---

## Protect the HLS encryption.key

To address the problem of simple automated downloading of encrypted HLS streams, I went with the idea suggested on the internet to use some kind of authentication cookies. The basic problem is that the encryption key file has to be refenced from the manifest file and with the Kaltura VOD module this is a fixed built in link and always provides the encryption file, without questions.

I wanted to put some kind of authentication on this single file (eg. `/hls/videoname.mp4/encryption.key`), so at first I went with the heavy-weight option: using the Nginx Javascript scripting engine NJS, and hacked the nytimes dockerfile I've used, to [compile nginx with the NJS module](https://hub.docker.com/repository/docker/robymus/nginx-vod-njs). As this module is not well documented and would need a few trial and errors, I've searched more and found the [guide on how to configure Nginx with subrequest authentication](https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-subrequest-authentication/). This is flexible enough for the level of protection I'm trying to achieve.

For this, I needed to include the http_auth_request module in nginx, so a I've created a new [docker image](https://hub.docker.com/repository/docker/robymus/nginx-vod-auth) for this purpose.

Setting up the things was quite straightforward from here, I've added a sub-location to the HLS location, with a request sent to an external service:

```
	location /hls/ {
        vod hls;
        vod_secret_key "secret$vod_filepath";
        vod_hls_encryption_method aes-128;
        alias /opt/video/;
        add_header Access-Control-Allow-Headers '*';
        add_header Access-Control-Allow-Origin '*';
        add_header Access-Control-Allow-Methods 'GET, HEAD, OPTIONS';
        add_header Access-Control-Expose-Headers 'Server,range,Content-Length,Content-Range';
        expires 30d;

        location ~ encryption\.key$ {
            vod hls;
            auth_request            /check_access;
            auth_request_set        $auth_status $upstream_status;
        }
	}
```

The only tricky part was to repeat the `vod hls` directive inside the sublocation as well, otherwise nginx treated it as a normal file. So this sublocation does a query to the /check_access internal (accessible only for internal requests inside Nginx) location, which is set up like this:

```
    location /check_access {
        internal;
        proxy_pass                      https://r2.io/tmp/check_access.php;
        proxy_pass_request_body off;
        proxy_set_header                Content-Length "";
        proxy_set_header                X-Real-IP $remote_addr;
        proxy_set_header                X-Original-URI $request_uri;
    }

```

It simply redirects the request to an external URL, passes the requester IP and URI, and also all the request cookies are automatically passed, so these all can be used for authentication. The external service has to simply return a HTTP status of 401 (Unauthorized) or 200 (OK). This gives some flexibility on how to handle authorization. As during normal playback the key is downloaded only once, right after the manifest was downloaded (and this single download is shielded by Safari on MacOS), using the authorization service to allow only a single download of the encryption key could be sufficient - the Safari HLS player will perform this first download and subsequent attempts to download will fail. This doesn't seem like a foolproof way to protect the key, as a sufficiently advanced plugin might hijack the key download before the browser, but the possiblities are hardly limited.

As I'm planning to run the authorization services on a separate machine, not on the video servers, I also needed a way to drop cookies on the video server (so they can be used for subsequent access verifications), so I set up a `/grant_access` location, which just proxies an external service once again, which can drop cookies "through the proxy link", resulting in cookies being dropped on the video playback server.

## Sources

Test scripts / sources available at [https://github.com/robymus/nginx-shaka-eme](https://github.com/robymus/nginx-shaka-eme/tree/5dbc5a7549fe95a68d8734a6fb5a2232000af7f8).

## What's next?

I will conclude the prototype/proof-of-concept at this point, as further tuning depends on the actual video playback scenario. I've explored how to generate encoded DASH and HLS videos on the fly with the Kaltura VOD module and how to play them back using Shaka Player. Also played with the possibilities of protecting the decryption keys with an extra added layer of authentication. The goal of this excercise was to create a simple system where automated downloaders possibly can't gain access to the encoded streams.
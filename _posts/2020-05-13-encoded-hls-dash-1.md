---
layout: post
title: Encoded HLS/DASH with clear keys [1]
categories:
  - html5-video
tags:
  - nginx
  - EME
  - HLS
  - DASH
---

## Idea

I wanted to create a simple video on demand delivery system with encoded streams, but built only on open source technology, without DRM licensing. I plan to create a level of protection where the delivered videos can't be downloaded in a single generic step, like installing a browser plugin which catches all the played videos and saves them locally. With normal HLS/DASH delivery, this is plugin is in theory possible, and the internet being the internet, probably exists.

I've been thinking about building a player with clear keys in Encrypted Media Extensions, but the time it came it, it was not mature and well supported, but checking on [caniuse.com](https://caniuse.com/#feat=eme) it should work on pretty much everything, even the dreaded Safari browsers.

## Setup

With the [Kaltura nginx-vod-module](https://github.com/kaltura/nginx-vod-module) for on-the-fly http packetization and encryption, and [Shaka Player](https://github.com/google/shaka-player) for encrypted stream playback, all the components were in place to test this in practice.

The nginx-vod-module needs to be compiled with nginx, so I looked around for an existing Docker container prebuilt with it. I went with [nytimes/nginx-vod-module-docker](https://github.com/nytimes/nginx-vod-module-docker). I've set up a local network test environment with SSL, while I was at it, I've tried to find a recent secure ciper configuration for nginx SSL, so I've configured it according to [syslink.pl](https://syslink.pl/cipherlist/).

I've configured nginx-vod-module for both encrypted HLS and DASH delivery, based on the examples in nginx-vod-module and nytimes/nginx-vod-module-docker.

## DASH

I've set up DASH with clear key encryption, and first tried to deliver the key to Shaka Player using HTTP key delivery, like:

```javascript
  player.configure({ 
    drm: { 
      servers: { 
        'org.w3.clearkey': 'https://r2.io/tmp/dash_clear_key.php' 
      } 
    } 
  });
```

While this worked fine in Chrome, the network log of developer console showed up the retrieved keys, and as this is available for developers, it might be available for plugins as well, making it possible again, to create a single click downloading solution. The next approach was directly passing the encryption keys to the player, without HTTP delivery:

```javascript
  player.configure({
    drm: {
      clearKeys: {
        // 'key-id-in-hex': 'key-in-hex',
        '0123456789abcdef0123456789abcdef': 'e890d7c082e819c66e60328634f89780',
      }
    }
  });  

````

This also worked well, so I concluded this as a working solution: I can create a key delivery mechanism where the key is not delivered to the player in cleartext, but using some (not necessarily strong) encryption, which could be deduced by analyzing the javascript sources (or even by hooking the Shaka Player configure method), but it is a two step downloading process, which is already, the goal is to prevent single step downloads.

After being overly happy because this worked fine in Chrome and Firefox on Linux, in Safari - both iOS and MacOS - it was a disaster.

To be honest, it was expected on iOS, but the non-encrypted DASH working well on MacOS, I hoped for the encrypted on to work as well. Instead it's [error 6001](https://shaka-player-demo.appspot.com/docs/api/shaka.util.Error.html):

> REQUESTED_KEY_SYSTEM_CONFIG_UNAVAILABLE
>
> None of the requested key system configurations are available. This may happen under the following conditions:
> - The key system is not supported.
> - The key system does not support the features requested (e.g. persistent state).
> - A user prompt was shown and the user denied access.
> - The key system is not available from unsecure contexts. (i.e. requires HTTPS) See https://bit.ly/2K9X1nY

So it's a no, no worries, HLS is needed for iOS anyway, so let's dive into that.

## HLS

Good news, setting up encryption was very simple and worked immediately both on iOS and MacOS. The encryption to go with is simple AES-128, it's very straighforward to set up with the nginx-vod-module:

```
  vod_secret_key "secret$vod_filepath";
  vod_hls_encryption_method aes-128;
```

The key can use nginx variables, so it's reasonably flexible, even though it's not queried from external service (like in the case of DASH encryption). But how HLS actually implements this encryption was not what I was looking for. In the manifest file an `EXT-X-KEY` tag is added, which specifies the url to retrieve the encryption key:

```
#EXT-X-KEY:METHOD=AES-128,URI="https://.../test.mp4/encryption.key"
```

So having the the manifest file the key can easily be retrieved alongside the fragments, and the video reconstructed. We are almost back to single step downloading process. Interestingly, while the `encryption.key` file shows up in Safari network log, it does not allow it to be read ("An error occured trying to load the resource"), so there is a chance that it is blocked from plugins also. However, the manifest contains the URL so it can be downloaded by the plugin agan, automatically.

There is work to be done here. Maybe changing to encryption key URL to a token based one, which allows downloading only once.To be continued :)

On a side note, for Shaka Player to play HLS on platforms where HLS is not natively supported, it requires (mux.js)[https://github.com/videojs/mux.js/] to be loaded before Shaka Player. It still doesn't supported AES-128 encrypted HLS though.

## Sources

Test scripts / sources available at [https://github.com/robymus/nginx-shaka-eme](https://github.com/robymus/nginx-shaka-eme/tree/ad4bc13a4fa1db4df20011e3faaef94d8821c8c5).

## What's next

- full prototype with key generation and encrypted key delivery for DASH
- minimal encrypted HLS key protection (token based single download?)
- combined prototype with secure token with expiration

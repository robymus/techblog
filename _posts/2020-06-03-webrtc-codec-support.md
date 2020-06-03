---
layout: post
title: WebRTC codec support detection
categories:
  - webrtc
tags:
  - WebRTC
  - TypeScript
  - NPM
---

I've been debugging a crazy problem in Firefox, it just didn't want to connect to WebRTC peers at all, complaining about ICE connection problems, STUN and TURN. The real problem was that the H.264 plugin was disabled, and the other peer wanted to send H.264 stream only, resulting in Firefox dropping the connection and writing an incorrect error message. (I know the other peer is not 100% compliant, as WebRTC should always support VP8 video for fallback, but let's just go with this for now).

It was a clean install of Firefox on Fedora 31, and according the internet, while it's rare, it sometimes happens, that the H.264 plugin gets disabled. To make it possible to detect this situation, I've endeavoured to create a small library to detect supported plugins by the browser's WebRTC stack.

## The solution

The magic is simple: create a minimal SDP describing the codec we want to test, as if it came from a remote peer, and feed it to a PeerConnection, to get an answer SDP, which can be parsed and checked if the codec is supported. After a few trial and errors I've arrived at a minimal SDP, which is acceptable by the browsers. For SDP creation and parsing, I've used the [sdp-jingle-json](https://github.com/otalk/sdp-jingle-json) library, it seemed quite simple, not overly bloated and sufficient for my usecase.

For publishing, it is even easier, as the browser as initiator will create the SDP offer, which can be parsed locally, without passing it through any signaling.

## The JavaScript/TypeScript development struggle is real

Now on to implementation. I wanted to take this opportunity to go through the full experience of creating a TypeScript package (this is practically the first time I've written something in TypeScript, not just reading or patching things), with all the tooling, understand how compilation and testing works, how to structure and prepare a package for publication into npm registry.

This seemed simple, but oh it became complicated. 

Setting up TypeScript and its configuration and compiling, this was easy. Mostly. Except that I sdp-jingle-json library is not a TypeScript library and didn't have type definition files pregenerated. Easy, I've generated the d.ts file with [dts-gen](https://github.com/Microsoft/dts-gen), then patched it up manually to link it to the library, now this works, next challenge.

I plan to use it in a legacy codebase, so I wanted to create a bundled minimized version, which includes its dependency library as well. Let's see [webpack](https://webpack.js.org/). Setting it up was surprisingly simple, let's continue. I've chosen the UMD2 module format, which (as is its name) a universal module format, and works pretty well with SystemJS, which is used in the target codebase.

Testing. As this library is intended for usage in a browser, and it works with the browser's WebRTC stack, tests should be run in a browser as well. [Jest](https://jestjs.io/) and [Puppeteer](https://pptr.dev/) to the rescue. Puppeteer is an interface to a headless (or not) Chromium, and it has good interopartion library for the Jest testing framework, so it should be relatively simple to make it work. That's what I thought. I've run into a bunch of dead-ends using outdated setup instructions from old posts, but finally closed in a good solution where I had everything running well.

Naively, I thought with all this setup, my test scripts will run in Chromium's JavaScript context fully, but that's not the case, all this setup just let's me control the browser, open webpages and check them.

So let's create a testing html page, that runs the tests, and just check the results. Simple. I need a http server, to serve these to puppeteer, luckily webpack's development server is perfect for this, some configuration hacking, and here we go.

Creating the testing htmls were simple, so finally put everything together. Oh no, not working, i mean sometimes working, mostly not. The problem was tricky, during jest/puppeteer startup, the webpack development server is launched but it's not ready to serve the files yet, when the test scripts try to access it - easy workaround, let's retry a few times if loading fails.

There was one final thing, the Chromium that puppeteer installs (probably all Chromium, though I haven't test it) didn't support H.264 codec with WebRTC PeerConnection, so naturally my tests just failed. I have confirmed this by running Chromium manually and trying to establish connection with a H.264 publishing peer, without success. No worries here, just accept as success that the detection code ran without errors and returned a not supported response.

I've used the excellent [np](https://github.com/sindresorhus/np) package to publish npm, it worked quite well at first try, except it failed at the 2FA step, and after that there was no way to continue it, but only the push tags to git and create github release steps remained, which were pretty easy to do manually.

## What did I learn

I've created my first package in NPM, learned a lot about tooling, the bunch of configuration files, their purpose and I'm in general less scared of the node.js / npm development/runtime ecosystem. I might even start to like it in the future :)

## Sources

Source on GitHub: [https://github.com/robymus/webrtc-codec-support](https://github.com/robymus/webrtc-codec-support)

Package in npm repository: [https://www.npmjs.com/package/webrtc-codec-support](https://www.npmjs.com/package/webrtc-codec-support)
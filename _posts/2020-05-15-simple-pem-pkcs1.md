---
layout: post
title: Simple Pem Keystore - PKCS#1 key support
categories:
  - java
tags:
  - Java
  - PKI
  - Gradle
---

## What is Simple Pem Keystore?

Java by default uses its own JKS (Java Key Store) format to store certificates and keys for cryptography, like SSL. THere is a tool shipped with Java (keytool) for converting from the more widely used PEM format to JKS and that's the way it was done for a long time.

With the advent of Let's Encrypt and other free SSL certificate authorities, our certificate lifetime drop down from the old-style 1-3 years to 90 days. That means in each 90 days the renewed certificates should be converted to JKS and the Java applications using them restarted - as the JKS keystore and the standard key manager does not support reloading, actually actively prohibiting it by using a caching mechanism.

Here comes the [simple-pem-keystore](https://github.com/robymus/simple-pem-keystore) project to solve both of these problems: I've created a simple keystore and key manager implementation, that plugs into the Java PKI infrastructure, and allows existing code to use PEM format certificates directly, with automatic change monitoring and reload support.

## PKCS#1 key support

As the original project was developed for my own inhouse projects, it supported only Let's Encrypt certificates and keys, where the keys are in PKCS#8 format. An other popular ACME client [acme.sh](https://github.com/acmesh-official/acme.sh) generates the keys in PKCS#1 format, so these didn't work with version 0.1-0.2 of Simple Pem Keystore.

After receiving a feature request through Github Issues, I've implemented PKCS#8 support in the key store. To be honest, I had this already implemented for my other Java crypto project, the [Wowza Let's Encrypt Converter](https://github.com/robymus/wowza-letsencrypt-converter), so this was a pretty straightforward backporting of the PEM loader to this project.

## Gradle build script

While I was revisiting the project, I upgraded the Gradle build script, change deprecated elements. The most significant change was moving from the old `maven` plugin to the new `maven-publish` plugin. There was a lot of magic preparing for Maven Central publication with the old plugin, moving to the new structure resulted in a much cleaner and more straightforward build file.

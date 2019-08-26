---
layout: post
title: "Generating X-Hub-Signature"
categories: [programming]
tags: [ programming, github, crypto, cyberchef]
---

Today will be a quick one, because I could not google this exact question.

Lets say, you have a microservice that accepts github webhooks, and you want to test it with set secret.
Github client libraries have ValidatePayload/ValidateSignatur: 
[https://github.com/google/go-github/blob/master/github/messages.go#L201](https://github.com/google/go-github/blob/master/github/messages.go#L201) but no generate signature.

You can use crypto packages to create signature for any payload like so:
[https://www.digitalocean.com/community/tutorials/how-to-use-node-js-and-github-webhooks-to-keep-remote-projects-in-sync](https://www.digitalocean.com/community/tutorials/how-to-use-node-js-and-github-webhooks-to-keep-remote-projects-in-sync) 

But if you just want to create a single signature for single payload, to unit test your endpoints, that seems like a overkill. 

Fortunately IT swiss army knife [CyberChef](https://gchq.github.io/CyberChef) exists.

And it contains [HMAC](https://gchq.github.io/CyberChef/#recipe=HMAC(%7B'option':'UTF8','string':'secret'%7D,'SHA1')&input=e30) recipe
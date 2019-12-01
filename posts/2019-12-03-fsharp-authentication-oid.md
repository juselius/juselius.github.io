---
title: AzureAd Authentication with F#
---

Almost all applications need some form of authentiation and authorization
scheme. It's a a pain in the neck for both developers and end users, and I
want as little to do with it as possible. OpenId Connect and AzureAd
offers a great way of delegating the mess to somebody else, and getting Single
Sign-On (SSO) as an added bonus.

This is the first post of in a two-part series. In the first post we cover
AzureAD and OpenID Connect authentication. In the second post we cover Windows
Authentication (Kerberos Negotiate) in F# on .NET Core 3, using docker running
in Kubernetes.



---
title: AzureAd Authentication with F#
---

Almost all applications need authentication and authorization
in some form. Authentication a pain in the neck for both developers and end
users, and I want as little to do with it as possible. OpenId Connect and
AzureAd offers a great way of delegating the job to somebody else, and as an
added bonus, getting Single Sign-On (SSO) in the process.

This is the first post of in a two-part series. In the first post we cover
AzureAD and OpenID Connect authentication. In the second post we cover Windows
Authentication (Kerberos Negotiate) in F# on .NET Core 3, using docker running
in Kubernetes.

A complete example program is available on
[Github](https://github.com/juselius/FSharpAzureAuthentication).


### Overview

In the following example we will be using F# on .NET Core 3.0, together with
ASP.NET Core and [Giraffe](https://github.com/giraffe-fsharp/Giraffe). Giraffe
offers a functional escape valve from the dreads of builder patterns,
dependency injection and other complications of APS.NET Core. Our strategy is to
configure the necessary middleware and then escape as quickly as possible into
the functional domain.

The Identity returned by authentication process is very limited in scope (as
with Windows Authentication). We'll use Microsoft Graph API:s to obtain more
information about the user, like full name and email address.

### AzureAd registration

AzureAd is a pretty big and complex machinery, with plenty of toggles. In this
example we will set up AzureAd to authenticate a server-side application with
permissions to act on the behalf of users in the specified organisation(s)
(tenants). We will additionally require users to be authenticated with the
tenant in order to access any application API:s, although this is actually
optional (but usually a good idea).

To register your application with AzureAd, follow these steps:

#. Log in to the [Azure portal](https://portal.azure.com) with your account.
#. Navigate to _Azure Active Directory_ -> _App registrations_ from the main
   menu and choose _New registration_.
#. Give your app a name, choose _Single tenant_, and give a callback
   URL of *https://localhost:8085/signin-oidc* (for the example app).
#. Copy the _Client Id_ and _Tenant Id_ into the ``appsettings.json`` file
   below.
#. Navigate to the _Certificates & secrets_ menu and create a new _secret_,
   and copy it to ``appsettings.json``.
#. Navigate to the _API permissions_ to give the app any additional permissions
   you need. By default it can read basic information about the user.
#. That's it, for now!

```json
{
   "AzureAd": {
        "Instance": "https://login.microsoftonline.com",
        "CallbackPath": "/signin-oidc",
        "BaseUrl": "https://localhost:8085",
        "ClientId": "",
        "TenantId": "",
        "ClientSecret": "",
        "Scopes": ".default",
        "GraphResourceId": "https://graph.microsoft.com/",
        "GraphScopes": ".default"
    },
    "AllowedHosts": "*"
}
```

### ASP.NET Core middleware

#### AzureAd

#### Graph

### Giraffe application


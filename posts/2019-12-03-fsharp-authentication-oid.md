---
title: AzureAd Authentication with F#
---

Almost all applications need authentication and authorization
in some form. Authentication a pain in the neck for both developers and end
users, and personally I want as little to do with it as possible. OpenId Connect
and AzureAd offers a great way of delegating the job to somebody else, gaining
Single Sign-On (SSO) as an added bonus.

We will cover these common cases:

#. Authenticating a server-side application, acting on behalf of the user.
#. Authenticating the user with the server application.
#. Authenticating with Microsoft Graph, to access a wealth of useful API:s.

This is the first post of in a two-part series. In the first post we cover
AzureAD and OpenID Connect authentication. In the second post we cover Windows
Authentication (Kerberos Negotiate) in F# on .NET Core 3, using Docker running
on Kubernetes.

The complete example code is available on
[GitHub](https://github.com/juselius/FSharpAzureAuthentication).

### Overview

In the following we will be using F# on .NET Core 3.0, together with
ASP.NET Core and [Giraffe](https://github.com/giraffe-fsharp/Giraffe). Giraffe
offers a functional escape valve from the dreads of OOP builder patterns,
dependency injection and other complexities of ASP.NET Core. Our strategy is to
configure the necessary ASP.NET middleware, and then escape as quickly as
possible into the functional domain.

### AzureAd registration

AzureAd is a complex machinery, with plenty of toggles. In this
example we will set up AzureAd to authenticate a server-side application with
permission to act on the behalf of users in the specified organisation
(tenants). Additionally we require users to be authenticated with the
tenant in order to access any application API:s. This step is actually
optional, but usually a good idea.

To register your application with AzureAd, follow these steps:

#. Log in to the [Azure portal](https://portal.azure.com) with your account.
#. Navigate to **Azure Active Directory** -> **App registrations** from the main
   menu and choose _New registration_.
#. Give your app a name, choose **Single tenant**, and give a callback
   URL of *https://localhost:8085/signin-oidc* (for the example app).
#. Copy the _Client Id_ and _Tenant Id_ into the ``appsettings.json`` file
   below.
#. Navigate to the **Certificates & secrets** menu and create a new _secret_,
   and copy it to ``appsettings.json``.
#. Navigate to **Authentication** and tick both the *Access tokes* and *ID tokes*
   boxes under the **Implicit grant** section. You can also add
   *https://localhost:8085/signout* under **Logout URL**.
#. Navigate to **API permissions** to give the app any additional permissions
   you need (e.g. ``User.Read.All`` for mail, etc.). By default it can only read
   very basic information about the user.

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

Next, we need to configure the necessary ASP.NET Core middleware to enable
*OpenId Connect* authentication:

* Authentication scheme
* Cooke policy
* CORS
* HSTS
* HTTPS redirect
* AzureAd authentication
* Graph authentication

The code for the [AzureAd](https://github.com/juselius/FSharpAzureAuthentication/blob/master/src/AzureAd.fs)
and [Graph](https://github.com/juselius/FSharpAzureAuthentication/blob/master/src/Graph.fs) authentication middleware have been
tranlated to F# from this C#
[implementation](https://github.com/Azure-Samples/active-directory-dotnet-webapp-openidconnect) by Microsoft.

The complete ASP.NET Core WebHost and middleware configuration for AzureAd:

```fsharp
let configureCors (builder : CorsPolicyBuilder) =
  builder.WithOrigins([| "https://login.microsoftonline.com" |])
    .AllowAnyMethod()
    .AllowAnyHeader() |> ignore

let cookiePolicyOptions (opt : CookiePolicyOptions) =
  opt.CheckConsentNeeded <- fun _ -> true
  opt.MinimumSameSitePolicy <- Http.SameSiteMode.None

let authenticationOptions (opt : AuthenticationOptions) =
  opt.DefaultScheme <-
    CookieAuthenticationDefaults.AuthenticationScheme
  opt.DefaultChallengeScheme <-
    OpenIdConnectDefaults.AuthenticationScheme
  opt.DefaultAuthenticateScheme <-
    CookieAuthenticationDefaults.AuthenticationScheme

let hstsOptions (opt : HstsOptions) =
  opt.IncludeSubDomains <- true
  opt.MaxAge <- TimeSpan.FromDays(365.0)

let configureApp (app : IApplicationBuilder) =
  app.UseDefaultFiles()
    .UseHttpsRedirection()
    .UsePathBase(PathString "/")
    .UseStaticFiles(staticFileOptions)
    .UseCookiePolicy()
    .UseCors(configureCors)
    .UseAuthentication()
    .UseSession()
    .UseGiraffe WebApp.webApp

let configureServices (services : IServiceCollection) =
  let sp  = services.BuildServiceProvider()
  let config = sp.GetService<IConfiguration>()
  let jsonSerializer = Thoth.Json.Giraffe.ThothSerializer()
  services.AddGiraffe() |> ignore
  services.AddSingleton<IJsonSerializer>(jsonSerializer) |> ignore
  services.Configure(cookiePolicyOptions) |> ignore
  services.Configure(hstsOptions) |> ignore
  services.AddCors() |> ignore
  services.AddSingleton<IGraphAuthProvider, GraphAuthProvider>() |> ignore
  services.AddResponseCaching() |> ignore
  services.AddDistributedMemoryCache() |> ignore
  services.AddSession() |> ignore
  services.AddHttpContextAccessor() |> ignore
  services.AddAuthentication(authenticationOptions)
      .AddAzureAd(fun options -> config.Bind("AzureAd", options))
      .AddCookie() |> ignore

WebHost
  .CreateDefaultBuilder()
  .UseKestrel()
  .ConfigureAppConfiguration(appSettings)
  .Configure(Action<IApplicationBuilder> configureApp)
  .ConfigureServices(configureServices)
  .UseWebRoot(publicPath)
  .UseUrls("https://0.0.0.0:" + port.ToString() + "/")
  .Build()
  .Run()
```

### Giraffe application

In this example we set up a simple API using Giraffe (see
[WebApp.fs](https://github.com/juselius/FSharpAzureAuthentication/blob/master/src/WebApp.fs))
, which allows the user to log in,
log out and retrieve some basic user information:

```fsharp
let authenticate : HttpHandler =
  challenge OpenIdConnectDefaults.AuthenticationScheme
  |> requiresAuthentication

let signIn (next : HttpFunc) (ctx : HttpContext) =
  authenticate next ctx

let signOut (next : HttpFunc) (ctx : HttpContext) =
  task {
    do! ctx.SignOutAsync()
    return! next ctx
  }

let webApp (next : HttpFunc) (ctx : HttpContext) =
  choose [
    routex "(/?)" >=> indexHtml
    route "/signin" >=> signIn >=> redirectTo false "/?signin"
    route "/signout" >=> signOut >=> redirectTo false "/?signout"
    route "/api/me" >=> getUser
  ] next ctx
```

The ``/signin`` endpoint triggers the authentication process by challenging the
client to authenticate using the OpenId Connect scheme, using the
``requiresAuthentication`` and ``challenge`` functions from Giraffe. If the user
isn't already authenticated, it redirects the user to the *OpenId Connect*
endpoint defined in ``appsettings.json``. Here we are using
[https://login.microsoftonline.com ](https://login.microsoftonline.com), for
normal AzureAd authentication (e.g. using 2FA, etc.). If authentication succeeds
the user is redirected back to the main page, with an added query string
allowing the endpoint to take any additional actions necessary after login.

The ``/signout`` endpoint triggers the standard logout process, and clears
credentials and identities from the request context.

When the user has logged in, the request context contains only some very basic
identity information about the user, like full name and user principal name
(UPN). The ``/api/me`` endpoint calls ``getUser``, which makes a request to
the Microsoft Graph API, requesting the user's e-mail address:

```fsharp
let private getAzureUser (ctx : HttpContext) =
  let name = ctx.User.Identity.Name
  let emailDecoder =
    Decode.field "value" (Decode.list (Decode.field "mail" Decode.string))
  let upn =
    try
        ctx.User.FindFirst(ClaimTypes.Name).Value.ToLower()
    with _ -> "unknown"
  task {
    let! token = graphAppToken ctx
    let query =
      graphUrlf "users?$filter=userPrincipalName eq '%s'&$select=mail" upn
    let content =
      Http.RequestString (
          query, headers = [ HttpRequestHeaders.Authorization token ]
      )
    let email =
      match Decode.fromString emailDecoder content with
      | Ok e -> try e.Head.ToLower() with _ -> "unknown"
      | Error _ -> "unknown"
    printfn "azureUser: %A" (name, upn, email)
    return (name, email)
  }

let getUser (next : HttpFunc) (ctx : HttpContext) =
  task {
    let! name, email = getAzureUser ctx
    return! json (name, email) next ctx
  }
```

The key function in the above code is ``graphAppToken ctx`` (defined in
[Graph.fs](https://github.com/juselius/FSharpAzureAuthentication/blob/master/src/Graph.fs)
in the example code):

```fsharp
let graphAppToken (ctx : HttpContext) =
    let authProvider = ctx.RequestServices.GetService<IGraphAuthProvider>()
    task {
        let! accessToken = authProvider.GetUserAccessTokenAsync ""
        let authHeader = AuthenticationHeaderValue("Bearer", accessToken)
        return authHeader.ToString ()
    }
```

This function requests a bearer token for the Graph API:s, which is then added
to  the standard HTTP Authorization tokens for the REST request.

Microsoft Graph contains a ton of useful REST API:s, which are a breeze to
use. Have fun! I have suffered so you don't have to.

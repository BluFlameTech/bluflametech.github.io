---
title: "Why You Need an ID Token and an Access Token"
date: 2021-05-10T09:00:00-04:00
header:
  teaser: /assets/images/posts/saml_oauth_oidc/security.jpg
categories:
- blog 
tags:
- security
- authorization
- web
- identity
---

If you read our last post [Secure Access Using SAML, OAuth and OIDC](/blog/saml-oauth-oidc/) or if you have a decent
understanding of OAuth and you're looking at OpenID Connect like "Why do I need an ID token and an access token?" then this post is for
you! We're going to discuss OAuth access tokens, OpenID Connect ID tokens, what those tokens are and why you probably
want to use both.

![Security](/assets/images/posts/saml_oauth_oidc/security.jpg)

_<small>Image by [Elena Nelyubina](https://www.istockphoto.com/portfolio/ElenaNelyubina?mediatype=illustration) on [iStock](https://www.istockphoto.com) by Getty Images</small>_

### OAuth vs OpenID Connect

A typical OAuth [Authorization Code] flow involving a website and third party resource provider looks like this.

![OAuth2 Auth Code Flow](/assets/images/posts/access_id_tokens/oauth.png)

_<small>OAuth Authorization Code Flow</small>_

The sequence of events can be described as follows.
1. The user makes a request to the Application
2. The in order to fulfill the request, the Application requires authorization from a 3rd party, so makes an auth code request
3. Granting of an auth code requires the user's agreement, so a request is forwarded to the user
4. If th user authorizes the request, an auth code is issued to the Application
5. The auth code, client id and client secret are submitted to the AuthZ Server in exchange for an access token
6. The access token can then be used to authorize requests to the Resource Server

Now, let's take a look at how a similar flow works with OpenID Connect.

![OIDC Auth Code Flow](/assets/images/posts/access_id_tokens/oidc.png)

_<small>OpenID Connect Authorization Code Flow</small>_

There is only one difference in the OpenID Connect flow: the issuance of an ID token. The details of that ID token
are also quite strictly defined by the [OpenID Connect specification](https://openid.net/specs/openid-connect-basic-1_0.html#IDToken).
The OpenID Connect ID token is a signed JWT with clearly defined claims. It can also have other claims, which are left
open-ended.

### The Access Token

Before we get into why ID tokens are needed, it's appropriate to discuss access tokens. After all, access tokens came first
with OAuth. OpenID Connect is an extension of OAuth. In short, an access token is for the Resource Server. Only
the Resource Server needs to know what it is since it's only used to authorize access to resources from the Resource Server.
Think of it like a [temporary] key that unlocks resources owned by the Resource Server. It doesn't matter what's in it
as long as the Resource Server can validate it.

With that being said, it has become relatively common to use JWT access tokens. However, practically, access tokens do 
not have to be JWTs. Another trend is the use of bearer access tokens, which are tokens that can be used from any source. Like 
bearer bonds, bearer tokens endow the holder with whatever rights are associated with the asset, regardless of who (or what) that holder may be.
Bearer tokens come with some great advantages: authorization overhead is low and distribution is easily scaled. The disadvantage
is that if a bearer token is compromised then it can be used to impersonate an authorized user until it expires. For that reason,
bearer access tokens are often short-lived.

#### Storing the Access Token

Let's say for the sake of argument that you have a bearer access token that is used by a SPA (Single Page Application) to
access resources from a REST API. It would be pretty slow to request an access token for every resource request made. So,
naturally, you want the access token to live for the duration of several requests. You need to store it somewhere. But you
also want its storage location to be relatively safe (i.e. not easily compromised).

Options for access token storage include: local storage, cookie, session or memory. However, nearly all of those
options leave the access token open to XSS attacks. Storing it in the session also makes it vulnerable to session hijacking.
Hiding the access token from the session and from JavaScript in general seems like a pretty good idea. And, indeed, that
is why a somewhat common practice is to save the access token in an HTTP-only cookie with HTTPS access strictly enforced.

### The ID Token

You could be forgiven for thinking that an easy approach might be to include user data, along with authorization data
in the access token. After all, access tokens are frequently JWTs and JWTs can be easily consumed with JavaScript. So,
why wouldn't you just make the access token available to the application, pad it with useful data and use that in your
web application? 

Unfortunately, for the reasons cited above, you probably don't want your access token to be available to your application,
and if it is, then you should assume that it's not consumable by your application. 
[React Authentication: How to Store JWT in a Cookie by Ryan Chenkie](https://medium.com/@ryanchenkie_40935/react-authentication-how-to-store-jwt-in-a-cookie-346519310e81)
does a pretty decent job explaining why you probably don't want to make your access token available to your application and why
you may want to store it in a HTTP-only cookie. But all of this raises the obvious question, "If access tokens aren't
available to the web application then how do you get all that great authorization data into the application?" The answer, of course,
is the ID token.

With OpenID Connect, ID tokens are issued at the same as access tokens. The access token is considered closed to the application
(and it very well may actually be closed to the application), but the ID token contains all the great authorization data
that your application needs. It can have user information, access scopes and other claims. But another important difference
between ID tokens and access tokens is that ID tokens, if compromised cannot be used to gain access into another system.
ID tokens are strictly for the benefit of the authorized application.

### Why You Want Both an Access Token and an ID Token

Access Tokens are needed, well, for access. They are what allow the application to consume resources that it doesn't own.
But access tokens should be considered closed to the application. So, the application can either query the necessary authorization data
it needs from a separate API call, or an ID token can be issued at the time of authorization in order to provide that data.
And the great thing about using a specification like OpenID Connect is that the contents of the ID token are pretty well-defined.
So, it's relatively easy to integrate with multiple authorization servers and still get the authorization data you need.

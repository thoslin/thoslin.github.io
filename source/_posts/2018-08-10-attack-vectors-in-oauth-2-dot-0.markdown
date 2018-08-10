---
layout: post
title: "Attack Vectors in OAuth 2.0"
date: 2018-08-10 17:44
comments: true
categories: 
---
Yesterday I was checking out which OAuth grant type is best fit for mobile applications. It turns out PKCE(Proof Key of Code Exchange) has become the de facto standard.

## The issue with implicit grant

The flow for [implicit grant](https://www.oauth.com/playground/implicit.html) is fairly simple.

- User agent redirects resource owner to authorization server
- Resource owner authorizes the access
- User agent redirects resource owner back to the client application with an access token in URL fragment.

In step 1, user agent, ie, mobile application, will normally open the browser to redirect resource owner to authorization server. The security implication here is that, using a web-view inside the application may expose resource owner credentials to application.

The attack vectors lie in step 3 after resource owner authorizes the access, the browser will redirect resource owner back to the application. For most mobile platforms, this is done by a special URL pattern, such as “com.foo.bar://callback-url”, applications bind handlers to this URL. So there’s a chance a malicious application also binds a handler to this URL, and gets the token in the URL fragment.

## Don’t ignore state parameter

Another attack vector is that, a malicious user places his own access token in the URL fragment. Now the user’s session is replaced with the attacker’s session. The resource the user creates will actually show up in attacker’s account.

Therefore it is crucial for the client application to verify the state parameter after the redirect, to ensure that the session is the one resource owner initiates. The client may generate and store the state in cookie or session before the redirect to authorization server.

This is also the case for [authorization code grant](https://www.oauth.com/playground/authorization-code.html) [without client_secret](https://aaronparecki.com/oauth-2-simplified/#single-page-apps) (which is a trend to replace implicit grant). The attacker may authorize the authorization server with his own credentials, normally the redirect will happen. However the attacker traps the redirect, copy the URL, and send it to the target user, or initialize. A victim may already login, and clicks on the link, now he gets the attacker’s token, and session is replaced.

## PKCE to the rescue

The implicit grant and the authorization code grant are both vulnerable if resource owner is redirected to a malicious client. But can we enforce some checks in the authorization server make sure the request comes from a valid client?

[PKCE](https://www.oauth.com/playground/authorization-code-with-pkce.html) solves this by introducing proof key for code exchange. It is an enhancement on authorization code grant thus the workflow is about the same, except the client needs to

- generate a key, which a random string and should be stored in the client.
- generate a challenge from the key, which is base64encode(sha256(key)), should be sent to authorization server.

So the idea here is that client sends the challenge in step 1 and prove it has the key to generate the  challenge when exchanging for token in step 3. Such that a malicious client is unable to obtain a code without the key, as it is only held by the righteous client.

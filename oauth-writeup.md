## Creating an Oauth API Client as a Low Privilege User

While testing an application for a private program, I decided to look for some ways of performing privilege escalation. This app had multiple permission levels. I was seeing what’s possible as a low level user attempting to perform admin-level API calls. I had an admin account for this site, so I could browse all of the features without a problem. One of the features of this application allows admins to create Oauth API clients for their instance of this application. This is one of the actions I was able to perform as a low level user. What’s the impact of this though? 

To start, what’s the purpose of this feature? It allows admins to create multiple Oauth clients configured however the user would like. When creating a new Oauth client, the admin determines the client type, scopes, the grant types, Oauth 1.0 or Oauth 2.0, redirect URL (if applicable), and allowed IP addresses. The response to the request contains the client’s ID and secret. Knowing very little about Oauth and needing to learn a lot more if I want to escalate this, I did some research on the different parts of Oauth, and determined the following…

**Client Type** - Determines your configuration. What grant types are available, whether or not you need a redirect URL, what scopes were available, and what grant types were available.

**Scope** - Determines the limits your token will have. Your permission levels. 

**Grant Type** - Determines the response. It’s the flow of how you’ll retrieve your access token.

This application supports multiple different grant types. We can use Authorization Code, Password, Refresh Token, Client Credentials, or Implicit. Reading through documentation for these, I came up with the following conclusions…

**Authorization Code** - Returns a code that is then exchanged for an access token. Requires the user to sign in.

**Password** - Returns an access token in exchange for the user’s password when the user logs in.

**Refresh Token** - Returns a new access token in exchange for a refresh token.

**Client Credentials** - Returns an access token in exchange for the client ID and secret.

**Implicit** - Returns an access token after the user has logged in.

### Exploitation

For a user, Oauth scopes won’t grant any additional privileges. It’ll only limit their current privileges. Since I was trying to escalate my privileges, I want to avoid using my account and use the client credentials grant type, which would allow me to act as a client and not as a user. This would allow me to submit my obtained client ID and client secret in exchange for an access token. This is perfect as long as the scopes available for this grant type give me decent privileges. It looks like there were two scopes available to me, one being read only and the other being read + write. Of course I’m going to choose the read + write scope. 

#### Request to Get Client ID and Secret
```
POST /api/v2/clients HTTP/1.1
...

{"name":"Attacker Client","permissions":"PARTNER","oauth1Enabled":false,
"oauth2Enabled":true,"grantTypes":["client_credentials"],"returnUrl":"",
"scopes":["ROLE_PARTNER"],"oauthOption":"2","persistentSso":false,"role":"PARTNER",
"withOidcScope":false,"withUserScopes":false,"refreshTokenSeconds":2592000}
```

#### Request to Get Access Token

```
POST /oauth2/token?client_id=myClientID&client_secret=mySecret&grant_type=client_credentials&scope=ROLE_PARTNER HTTP/1.1
...
```

Reading this application’s API documentation, I could see exactly what API calls were available with this specific scope. Turns out I should be able to call any API endpoint that interacts with user accounts. All I needed to do was submit my newly acquired access token in the Authorization header using the bearer schema when submitting an API request: `Authorization: bearer _mytoken_`. This worked. I wasn’t able to access any of the admin-only features, but this scope gave me the ability to read/write to users’ accounts and perform any actions to any user on the instance of this application. I thought I could trick the system by requesting additional scopes when requesting an access token. The application happily returned an access token, but sadly did not grant me additional privileges (at least from what I saw). In the end, it ended up being a high severity bug, which paid fairly well and taught me a lot.

A good resource on the different grant types: https://alexbilbie.com/guide-to-oauth-2-grants/

Thanks for reading.

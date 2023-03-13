# Setup for Authentication-Tokens



This will be done via dedicated section within the portfolio.json. There can be multiple sources by using the *tokenSourceUid* as dictionary-key. If specified, there can one *primaryUiTokenSourceUid* used to represent the application login. Otherwise tokens will be retrieved on demand when there needed while using the application (for example when a webservice-based datasource will be initialized).

```
{
	"primaryUiTokenSourceUid":"2DB9CFC8-67B2-8DA9-4790-BC7FDE7ECB73",
	"tokenSources": {
		"2DB9CFC8-67B2-8DA9-4790-BC7FDE7ECB73": {
        	... 
        },
		"76FB7A74-9990-00A3-4045-8C6098FE5E5E": {
        	... 
        },
		"7EB2B050-E40A-C38D-4964-67703163BA26": {
        	... 
        },
	}
}
```



per *tokenSource* there are individually configurations for two usecases:

1. in which way the token will be **issued** (when a new one is needed)
2. in which way the token will be **validated**



## Issuing

 For issuing Tokens there are multiple strategies, from which can be chosen by specifying the ***issueMode***:

```
"issueMode": <NAME_OF_THE_STRATEGY>
```



### Issue mode "RAW-INPUT"

There is not much to say - the logon overlay will open and ask the user to **provide a string based token as raw text input**. This input will be validated (using the configured validation method) and **directly persisted** as token.

```
"issueMode": "RAW-INPUT"
```



### Issue mode "HTTP-GET"

In this way the **token will be loaded via HTTP-GET** request using a given URL like this:

```
"issueMode": "HTTP-GET"
"retrieveEndpointUrl": "https://anydomain.com/demoAccessToken.txt"
"retrieveEndpointAuthorization": "basic %232432-23452-234234234%"
```

* the **token will NEVER be persisted**.
* the application will **NOT open any logon** overlay, because the token can be retrieves in background on demand.
* the *retrieveEndpointAuthorization* is optional and can refer another *tokenSourceUid* as placeholder to fill the HTTP-Authorization-Header (may be for *basic* or *bearer* auth).



### Issue mode "LOCAL_BASICAUTH_GENERATION"

the logon overlay will open and **ask the user to provide credentials** asking for a *Logon-Name* and a *Logon-Pass*. Then a **basic-auth-string will be generated**, validated (using the configured validation method) **and persisted** as token. In addition to that the following self-explaining options will be possible:

```
"issueMode": "LOCAL_BASICAUTH_GENERATION"
"localLogonNameToLower": false,
"localLogonNamePersistation": "NEVER" | "OPT-IN" | "OPT-OUT" | "ALWAYS"
```



### Issue mode "LOCAL_JWT_GENERATION"

The logon overlay will open and ask the user to provide credentials asking for a LogonName (but only if the *localLogonNameInputLabel* is not empty) and/or a LogonPass (but only if the *localLogonPassInputLabel* is not empty). Then **a JWT will be generated**, validated (using the configured validation method) **and persisted** as token. In addition to that the following self-explaining options will be possible:

```
"issueMode": "LOCAL_JWT_GENERATION"
"localLogonNameToLower": false,
"localLogonNamePersistation": "NEVER" | "OPT-IN" | "OPT-OUT" | "ALWAYS",
"localLogonNameSyntax": null,
"localLogonNameInputLabel":"Employee number",
"localLogonPassInputLabel":"Portal password",
"localLogonSaltDisplayLabel": null,
"jwtExpMinutes": 1440,
"jwtSelfSignKey": ".......",
"jwtSelfSignAlg": "SHA256"
"claims": { ... }
```

**NOTE:** the ["Claims"-Section](## Claims) contains the security matadata which is usually located inside of the token-content - see the detailed description in the dedicated heading below...

The "jwtSelfSignKey" can be a hard coded public-/private-/symmetric- key (corresponding tho the selected "jwtSelfSignAlg") OR a Pattern to generate a a key at runtime to be used for hashing. This could be for example in one of this ways:

```
"jwtSelfSignKey": "thisIsSomeSalt_%logonName%_%logonPass%"
```

 OR

```
"jwtSelfSignKey": "%azp%_%applicationUrl%"
```

 As **placeholders** there are %logonName%*, *%logonPass%*, *%applicationUrl%* and ALL keys from the "claims"-Section (like *%sub%* or *%azp%*).



### Issue mode "OAUTH_CIBA_CODEGRAND"

The logon overlay will open an **IFRAME** displaying the *authEndpointUrl* (with the necessary OAuth query parametes) so that the **user can submit his credentials directly to the OAuth server**. Then a **token will be retrieved following the [OAuth-CIBA flow](https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0-03.html)**, validated (using the configured validation method) **and persisted** as token. Lets see the configuration structure:

```
"issueMode": "OAUTH_CIBA_CODEGRAND",
"clientId":"xxxxxxxx",
"clientSecret":"xxxxxxxx",
"authEndpointUrl": "https://theOAuthServer/authorize",
"additionalAuthArgs": {
	...
},
"retrieveEndpointUrl": "https://theOAuthServer/token",
"additionalRetrieveArgs":{
    ...
},
"retrieveViaGet": false
```

**NOTE:** when using an **OAuth-Proxy**, the *clientSecret* can be empty because it will be appended at server side. In addition to that the *retrieveViaGet* can be activated to resolve CORS Problems (not yet recommended).

The *additionalAuthArgs* and *additionalRetrieveArgs* are dictionaries of strings and can be used to provide arguments that should be added to the OAuth requests. Sample:

```
"additional...Args":{
	 "login_hint": "guest",
     "scope": "%tenant% %profile%",
     "viewmode": "2"
}
```

Any **placeholders** within the wellknown key ***scope***  (in this example *%tenant% and *%profile%* will be resolved from corresponding entires within the *applicationScope* section on top level of the portfolio.json file!

The **mandatory OAuth arguments** (*clientid*, *return_uri*, *state* and *clientSecret*) **will be set automatically **and don't need to be specified as additional arg.



## Claims

Claims (as defined [here](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1)) are the representation of identity oder lifetime related metadata of a token. If using JWT-Tokens, than they are inside of the encoded token body, otherwise they could be request for any non-JWT token via HTTP call to a [OAuth 2.0 Token Introspection](https://datatracker.ietf.org/doc/html/rfc7662)-Endpoint.

In our case you can declare a key-value structure like this:

```
"claims":{
    "sub": "websuer-%logonName%",
    "iss": "%applicationTitle%",
    "azp": "%logonPassHash%",
    "aud": "%appliactionUrl%",
    "scope"; "secenv:1 tenant:%tenant%",               
}
```

This configuration is used when **generating JWTs** directly from the application frontend (if enabled) and/or when **validating tokens**.



### Placeholders within Claims

The following **placeholders** can be used:

| Placeholder          | Semantics                                                    | when generating a JWT:              | when validating a Token                                      |
| -------------------- | ------------------------------------------------------------ | ----------------------------------- | ------------------------------------------------------------ |
| *%applicationTitle%* | the title of the application as defined within the portfolio.json | will be resolved before generating  | will be resolved before comparison                           |
| *%appliactionUrl%*   | the root URL of the application                              | will be resolved before generating  | will be resolved before comparison                           |
| *%tenant%* / ...     | **(dynamic) ANY Key-Name** from the **"*applicationScope*"**-Section within the portfolio.json | will be resolved before generating  | will be resolved before comparison                           |
| *%logonName%*        | the user name that has been entered during logon             | will be requested before generating | is not present and will be **replaced by "*"** before comparison, so that only the pattern of the constant string will need to match |
| *%logonPassHash%*    | an sha256-hash of the password that has been entered during logon | will be requested before generating | not available                                                |
| *%localSalt%*        | a random **fingerprint**, that will be **generated once** (when requested the first time) and **persisted to the local store of the browser** (key: "salt_*tokenSourceUid*") | will be generated on fist usage     | will be generated on fist usage                              |

**Special Case:** The Token-Validation has a Special treatment of the wellknown claim **"*scope*"**. This is because the scope is defined as a space-separated list. So the validation will require that the declared scopes are present at any index of this list, but additional (unexpected) entries ware allowed and will not lead to a negative validation outcome!



## Validation

 For the validation of tokens there are multiple strategies, from which can be chosen by specifying the ***validationMode***:

```
"validationMode": <NAME_OF_THE_STRATEGY>,
"validationOutcomeCacheMins": 15
```

The validation will be triggered:

* ... when a token requests internally (for example when initializing a datasource)
* ... directly after a new token was created/retrieved, before the token is persisted

* ... exclusive for the *primaryUiToken* as predecessor on any UI navigation. For this, a minimum of 15 minutes for the *validationOutcomeCacheMins* is recommended .

  

### Validation mode "LOCAL_JWT_VALIDATION"

Will compare the *claims* from the configuration (after resolving placeholders) with the claims inside of the encoded JWT content requiring a match. after that the *jwtValidationKey* (after resolving placeholders) will be used to verify the JWT signature.

```
"validationMode": "LOCAL_JWT_VALIDATION"
"jwtValidationKey": "xxxxx", 
"claimValidationIgnoresCasing": true
"claims":{...}
```

**NOTE:** the ["Claims"-Section](## Claims) contains the security matadata which is usually located inside of the token-content - see the detailed description in the dedicated heading ...



### Validation mode "OAUTH_INTROSPECTION_ENDPOINT"

Will call a [OAuth 2.0 Token Introspection](https://datatracker.ietf.org/doc/html/rfc7662)-endpoint using the *validationEndpointUrl* to retrieve information about the validity of the token and corresponding claims. If the reply indicates validity (expiration check is included), the *claims* from the configuration (after resolving placeholders) will be compared with the claims received as reply from the introspection endpoint.

```
"validationMode": "OAUTH_INTROSPECTION_ENDPOINT"
"validationEndpointUrl": "https://theOAuthServer/introspect"
"validationEndpointAuthorization": "basic %232432-23452-234234234%" 
"claimValidationIgnoresCasing": true
"claims":{...}
```

**NOTE:** the ["Claims"-Section](## Claims) contains the security metadata which is usually located inside of the token-content - see the detailed description in the dedicated heading ...

The '*validationEndpointAuthorization*' is optional and can be used to fill the **HTTP-*Authorization*-Header** (may be for "*basic*" or "*bearer*" auth). Therefore it is possible to **refer to another token** by using a foreign *tokenSourceUid* as placeholder name (see the sample).



### Validation mode "GITHUB_VALIDATION_ENDPOINT"

...is **planned in future versions**, almost like the "OAUTH_INTROSPECTION_ENDPOINT", but following another schema for the HTTP requests which are unfortunately NOT compliant to the [OAuth 2.0 Token Introspection](https://datatracker.ietf.org/doc/html/rfc7662)-Endpoint. Same could be done for a small set of other very popular providers (like Google, Microsoft, etc.)



## Working without a primary auth server

Here we want to describe a Usecase where "LOCAL_JWT_GENERATION" and "LOCAL_JWT_VALIDATION" are combined to create a application login which does not need a primary auth server. Any user can logon to the application, and the tokens wild represent just a session-identity. a real validation will be happen within the first usage of that token (may be when initializing a backend-datasource, which's URL needs to be configured by the user first).

**The Idea: **

Use the *%localSalt%* placeholder within the:

```
"jwtSelfSignKey": "%localSalt%"
```

as well as in the:

```
"jwtValidationKey": "%localSalt%"
```

This will archive, that the persistent fingerprint of the local browser is used to create and validate JWTs locally. When the tokens are used, then it is required to run a **validation on server side**, which means that **any permission must not be assumed without a 3rd parity clearance**!

There are two flavors:

1. transporting the password-hash over the claims, and check them on server side (which is not recommended for security reasons!):

```
"claims":{
   "sub": "%logonUser",
   "azp": "%logonPassHash%",
   ...              
}
```

2. just asking for the users email-address during logon (and not for any password), and send an email to the user (out of the backend) containing a confirmation-link to allow the session:

```
"localLogonNameInputLabel":"Your email address",
"localLogonPassInputLabel": null,
"claims":{
   "sub": "%logonUser"           
}
```

In additional to that you can increase security by requiring the user to enter the *%localSalt%* during confirmation. For that it is also possible to get it displayed on the UI by setting a label to a non-empty string:

```
"localLogonSaltDisplayLabel": "your Fingerprint",
```



*T.Korn 02/2023*
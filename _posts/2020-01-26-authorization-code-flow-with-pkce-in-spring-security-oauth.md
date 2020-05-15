---
layout: post
title: "Authorization Code Flow with PKCE in Spring Security OAuth"
author: "Anar Sultanov"
image: /images/posts/2020-01-26/image.jpg
---
[RFC 7636](https://tools.ietf.org/html/rfc7636): Proof Key for Code Exchange (PKCE, pronounced "pixy") describes an extension to the Authorization Code flow to protect public clients from authorization code interception attack.
<br>In this tutorial, we are going to look at how to implement this extension in an OAuth 2.0 authorization server built using Spring Security OAuth, which does not support it out of the box.

<i class="fa fa-exclamation-triangle"></i> <b>Warning:</b>
Spring Security OAuth is deprecated and is not recommended for use in new projects.

## 1. Authorization Server
Let's start by creating an authorization server.

#### 1.1. Configuration
We create a configuration class for the authorization server and configure an in-memory client store with two initial clients, public and private:
{% highlight java %}-
@Configuration
@EnableAuthorizationServer
public class AuthServerConfig extends AuthorizationServerConfigurerAdapter {

    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) {
        security.allowFormAuthenticationForClients();
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .withClient("public")
                .secret("{noop}")
                .redirectUris("http://public-client/")
                .authorizedGrantTypes("authorization_code")
                .scopes("read")
                .autoApprove(true)
                .and()
                .withClient("private")
                .secret("{noop}secret")
                .redirectUris("http://private-client/")
                .authorizedGrantTypes("authorization_code")
                .scopes("read")
                .autoApprove(true);
    }
}
{% endhighlight %}

Then we create a configuration class in which we configure http security and an in-memory authentication manager with one user:
{% highlight java %}
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }

    protected void configure(HttpSecurity http) throws Exception {
        http
                .requestMatchers().antMatchers("/login", "/oauth/authorize")
                .and()
                .authorizeRequests().anyRequest().authenticated()
                .and()
                .formLogin().permitAll();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
                .withUser("john")
                .password(passwordEncoder().encode("pass"))
                .roles("USER");
    }
}
{% endhighlight %}

#### 1.2. Test

To test the authorization server, we can enter the address in a browser:
```
http://localhost:8080/oauth/authorize?response_type=code&client_id=public&redirect_uri=http://public-client/&scope=read
```
We will be redirected to the login page and after entering credentials (`john`/`pass`) we will be redirected
to the client redirect uri with the authorization code in the request parameter, e.g.:
<br/>`http://public-client/?code=UT41zH`.

We can exchange this code for an access token, for example, using `curl`:
{% highlight console %} 
> curl localhost:8080/oauth/token -d client_id=public -d grant_type=authorization_code -d redirect_uri=http://public-client/ -d code=HbGFZh
{"access_token":"91a46591-e953-4a68-90d3-acfd021230d2","token_type":"bearer","expires_in":43199,"scope":"read"}
{% endhighlight %}

Note that in the case of a private client, you will also need to add its secret to the request: <span class="nobr">`-d client_secret=secret`</span>.

## 2. Code Challenge

Now we can move on to implementing the PKCE extension and we will start from the Authorization Request.

By the standard a client needs to create and record a secret named the `code_verifier` from which it derives a transformed version
called `code_challenge`, which is sent in the Authorization Request along with the transformation method called `code_challenge_method`.
The standard provides two transformation methods, `plain` and `S256`, but if the client is capable of using `S256`, it **must** use `S256`.
In case of `code_challenge_method` is not present in the request transformation method defaults to `plain`.

In order to make our server handle these two parameters, we need to implement custom AuthorizationCodeServices. 
For simplicity, we will use an in-memory storage implemented using `Map`. 
The difference from the standard `InMemoryAuthorizationCodeServices` provided by Spring is that in addition to saving the code and authentication, 
we also need to save the code challenge and transformation method.

First, we create enumeration for our transformation methods, having also provided value for private clients who do not need it:
{% highlight java %}
public enum CodeChallengeMethod {
    S256,
    PLAIN,
    NONE
}
{% endhighlight %}

Next, create a container class to hold authentication and its associated `code_challenge` and `code_challenge_method`. 
Here we also provide a constructor for clients who do not require these parameters:
{% highlight java %}
public class PkceProtectedAuthentication {
    private final String codeChallenge;
    private final CodeChallengeMethod codeChallengeMethod;
    private final OAuth2Authentication authentication;
    
    public PkceProtectedAuthentication(OAuth2Authentication authentication) {
        this.codeChallenge = null;
        this.codeChallengeMethod = CodeChallengeMethod.NONE;
        this.authentication = authentication;
    }

    public PkceProtectedAuthentication(String codeChallenge, CodeChallengeMethod codeChallengeMethod, OAuth2Authentication authentication) {
        this.codeChallenge = codeChallenge;
        this.codeChallengeMethod = codeChallengeMethod;
        this.authentication = authentication;
    }
}
{% endhighlight %}

Finally, we implement our `AuthorizationCodeServices`, or rather the `createAuthorizationCode` method, 
and we will leave `consumeAuthorizationCode` for later.

We will require the `code_challenge` parameter only from public clients. 
We could add a parameter to client details about this requirement (for example, if for some reason we do not want to force 
all public clients to use PKCE or even use it for some private clients), but in this example we will consider all clients 
with a missing secret as requiring it, and in accordance with the standard, we will return an "invalid_request" error if `code_challenge` is not sent.

Also, according to the standard, we will use the “plain” transformation method if `code_challenge_method` is not in the request,
and return an error if the unsupported transformation is requested.

{% highlight java %}
public class PkceAuthorizationCodeServices implements AuthorizationCodeServices {

    private final RandomValueStringGenerator generator = new RandomValueStringGenerator();
    private final Map<String, PkceProtectedAuthentication> authorizationCodeStore = new ConcurrentHashMap<>();

    private final ClientDetailsService clientDetailsService;
    private final PasswordEncoder passwordEncoder;

    public PkceAuthorizationCodeServices(ClientDetailsService clientDetailsService, PasswordEncoder passwordEncoder) {
        this.clientDetailsService = clientDetailsService;
        this.passwordEncoder = passwordEncoder;
    }

    @Override
    public String createAuthorizationCode(OAuth2Authentication authentication) {
        PkceProtectedAuthentication protectedAuthentication = getProtectedAuthentication(authentication);
        String code = generator.generate();
        authorizationCodeStore.put(code, protectedAuthentication);
        return code;
    }

    private PkceProtectedAuthentication getProtectedAuthentication(OAuth2Authentication authentication) {
        Map<String, String> requestParameters = authentication.getOAuth2Request().getRequestParameters();

        if (isPublicClient(requestParameters.get("client_id")) && !requestParameters.containsKey("code_challenge")) {
            throw new InvalidRequestException("Code challenge required.");
        }

        if (requestParameters.containsKey("code_challenge")) {
            String codeChallenge = requestParameters.get("code_challenge");
            CodeChallengeMethod codeChallengeMethod = getCodeChallengeMethod(requestParameters);
            return new PkceProtectedAuthentication(codeChallenge, codeChallengeMethod, authentication);
        }

        return new PkceProtectedAuthentication(authentication);
    }

    private CodeChallengeMethod getCodeChallengeMethod(Map<String, String> requestParameters) {
        try {
            return Optional.ofNullable(requestParameters.get("code_challenge_method"))
                    .map(String::toUpperCase)
                    .map(CodeChallengeMethod::valueOf)
                    .orElse(CodeChallengeMethod.PLAIN);
        } catch (IllegalArgumentException e) {
            throw new InvalidRequestException("Transform algorithm not supported");
        }
    }
    
    private boolean isPublicClient(String clientId) {
        String clientSecret = clientDetailsService.loadClientByClientId(clientId).getClientSecret();
        return clientSecret == null || passwordEncoder.matches("", clientSecret);
    }

    @Override
    public OAuth2Authentication consumeAuthorizationCode(String code) {
        throw new UnsupportedOperationException();
    }
}
{% endhighlight %}

It remains only to configure the server to use our implementation of `AuthorizationCodeServices`:
{% highlight java %}
public class AuthServerConfig extends AuthorizationServerConfigurerAdapter {

    private final PasswordEncoder passwordEncoder;

    public AuthServerConfig(PasswordEncoder passwordEncoder) {
        this.passwordEncoder = passwordEncoder;
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
        endpoints.authorizationCodeServices(new PkceAuthorizationCodeServices(endpoints.getClientDetailsService(), passwordEncoder));
    }
    
    // ...
}
{% endhighlight %}

## 3. Code Verifier

Having received the authorization code, the client must exchange it for a token by sending a request with 
the `code_verifier` parameter to the token endpoint.

After receiving an access token request, the server needs to calculate the code challenge from the received `code_verifier` 
according to transformation method and compare it with the previously associated `code_challenge`.

First, we will teach each of the transformation methods to calculate `code_challenge` from `code_verifier`, 
for this we will add an abstract method to enum and implement it in each method according to the standard:
{% highlight java %}
public enum CodeChallengeMethod {
    S256 {
        @Override
        public String transform(String codeVerifier) {
            try {
                MessageDigest digest = MessageDigest.getInstance("SHA-256");
                byte[] hash = digest.digest(codeVerifier.getBytes(StandardCharsets.US_ASCII));
                return Base64.getUrlEncoder().encodeToString(Hex.encode(hash));
            } catch (NoSuchAlgorithmException e) {
                throw new IllegalStateException(e);
            }
        }
    },
    PLAIN {
        @Override
        public String transform(String codeVerifier) {
            return codeVerifier;
        }
    },
    NONE {
        @Override
        public String transform(String codeVerifier) {
            throw new UnsupportedOperationException();
        }
    };

    public abstract String transform(String codeVerifier);
}
{% endhighlight %}

Then we add a method to `PkceProtectedAuthentication` class for getting authentication, which compares the result of the `code_verifier` transformation with `code_challenge` 
and returns authentication if they match, otherwise it returns an error response indicating `invalid_grant` by throwing an exception:
{% highlight java %}
public class PkceProtectedAuthentication {

    // ...
    public OAuth2Authentication getAuthentication(String codeVerifier) {
        if (codeChallengeMethod == CodeChallengeMethod.NONE) {
            return authentication;
        } else if (codeChallengeMethod.transform(codeVerifier).equals(codeChallenge)) {
            return authentication;
        } else {
            throw new InvalidGrantException("Invalid code verifier.");
        }
    }
}
{% endhighlight %}

We cannot override and use `AuthorizationCodeServices#consumeAuthorizationCode(String)`, since we also need to pass a code verifier.
Therefore, we create another method that gets an instance of` PkceProtectedAuthentication` from the store using an authorization code 
and tries to extract `OAuth2Authentication` from it by providing a code verifier:
{% highlight java %}
public class PkceAuthorizationCodeServices implements AuthorizationCodeServices {

    // ...
    
    public OAuth2Authentication consumeAuthorizationCodeAndCodeVerifier(String code, String verifier) {
        return authorizationCodeStore.get(code).getAuthentication(verifier);
    }
}
{% endhighlight %}

Finally, we need to implement a token granter for the authorization code grant type that will support the additional `code_verifier` parameter.
We do this by extending `AuthorizationCodeTokenGranter` with the only difference that it reads `code_verifier` from the request parameters 
and uses it to obtain the stored authentication:
{% highlight java %}
public class PkceAuthorizationCodeTokenGranter extends AuthorizationCodeTokenGranter {

    private final PkceAuthorizationCodeServices authorizationCodeServices;

    public PkceAuthorizationCodeTokenGranter(AuthorizationServerTokenServices tokenServices, PkceAuthorizationCodeServices authorizationCodeServices, ClientDetailsService clientDetailsService, OAuth2RequestFactory requestFactory) {
        super(tokenServices, authorizationCodeServices, clientDetailsService, requestFactory);
        this.authorizationCodeServices = authorizationCodeServices;
    }

    @Override
    protected OAuth2Authentication getOAuth2Authentication(ClientDetails client, TokenRequest tokenRequest) {
        Map<String, String> parameters = tokenRequest.getRequestParameters();
        String authorizationCode = parameters.get("code");
        String redirectUri = parameters.get("redirect_uri");
        if (authorizationCode == null) {
            throw new InvalidRequestException("An authorization code must be supplied.");
        } else {
            String codeVerifier = parameters.getOrDefault("code_verifier", "");
            OAuth2Authentication storedAuth = authorizationCodeServices.consumeAuthorizationCodeAndCodeVerifier(authorizationCode, codeVerifier);
            if (storedAuth == null) {
                throw new InvalidGrantException("Invalid authorization code: " + authorizationCode);
            } else {
                OAuth2Request pendingOAuth2Request = storedAuth.getOAuth2Request();
                String redirectUriApprovalParameter = pendingOAuth2Request.getRequestParameters().get("redirect_uri");
                if ((redirectUri != null || redirectUriApprovalParameter != null) && !pendingOAuth2Request.getRedirectUri().equals(redirectUri)) {
                    throw new RedirectMismatchException("Redirect URI mismatch.");
                } else {
                    String pendingClientId = pendingOAuth2Request.getClientId();
                    String clientId = tokenRequest.getClientId();
                    if (clientId != null && !clientId.equals(pendingClientId)) {
                        throw new InvalidClientException("Client ID mismatch");
                    } else {
                        Map<String, String> combinedParameters = new HashMap<>(pendingOAuth2Request.getRequestParameters());
                        combinedParameters.putAll(parameters);
                        OAuth2Request finalStoredOAuth2Request = pendingOAuth2Request.createOAuth2Request(combinedParameters);
                        Authentication userAuth = storedAuth.getUserAuthentication();
                        return new OAuth2Authentication(finalStoredOAuth2Request, userAuth);
                    }
                }
            }
        }
    }
}
{% endhighlight %}

## 4. Test

To test our flow, we can first try to authorize using a public client without the `code_challenge`:
```
http://localhost:8080/oauth/authorize?response_type=code&client_id=public&redirect_uri=http://public-client/&scope=read
```
And our client will receive the corresponding error message through a redirect:
```
http://public-client/?error=invalid_request&error_description=Code%20challenge%20required.
```

To successfully obtain the code, we need to generate a code verifier, 
for which we will use the following value: `4cc9b165-1230-4607-873b-3a78afcf60c5`.

To use the **plain** transformation method, we can specify the same value for `code_challenge`:
```
http://localhost:8080/oauth/authorize?response_type=code&client_id=public&redirect_uri=http://public-client/&scope=read&code_challenge=4cc9b165-1230-4607-873b-3a78afcf60c5
```
Or use the **s256** method, indicating it in the request and adding our value by first SHA256 hashing and base64url encoding it:
```
`http://localhost:8080/oauth/authorize?response_type=code&client_id=public&redirect_uri=http://public-client/&scope=read&code_challenge=YmRmMTkyODk4YjJhYmM4MWQyOGNlZWYxMWJmODExMTYyMWZjY2ZhMGNjMGJjZTZlMjAwMGZlMzdmODc0MjcwZQ==&code_challenge_method=s256
```

Having received the code, we can exchange it for a token by adding `code_verifier` to the request:
{% highlight console %} 
> curl localhost:8080/oauth/token -d client_id=public -d grant_type=authorization_code -d redirect_uri=http://public-client/ -d code=piGcej -d code_verifier=4cc9b165-1230-4607-873b-3a78afcf60c5
{"access_token":"5e3a5084-fb95-45db-b2b1-aebee20f7930","token_type":"bearer","expires_in":43199,"scope":"read"}
{% endhighlight %}

Trying to log in using a private client with a secret, we can make sure that our changes did not affect them in any way.

## 5. Conclusion
Now we can make our mobile and SPA clients more secure by using the Authorization Code with PKCE flow. 

_Full source code can be found on [GitHub](https://github.com/AnarSultanov/examples/tree/master/spring-boot-oauth2-pkce)._

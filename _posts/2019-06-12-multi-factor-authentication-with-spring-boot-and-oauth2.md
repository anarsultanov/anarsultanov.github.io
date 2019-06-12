---
layout: post
title: "Multi-factor Authentication with Spring Boot and OAuth2"
author: "Anar Sultanov"
image: /images/posts/2019-06-12/image.jpg
---
Multi-factor Authentication (MFA) is an authentication method which requires more than one piece of evidence to verify a user's identity. It provides an additional level of security and reduces the likelihood of unauthorized access.
<br>In this tutorial, we are going to implement an authorization server with support for multi-factor authentication in the [Resource Owner Password Credentials Flow](https://tools.ietf.org/html/rfc6749#section-4.3). 

## 1. The Auth Server
Let's start by creating an authorization server as a simple Spring Boot application.

#### 1.1. Maven Dependencies
First we need to add the dependencies to `pom.xml`:
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.security.oauth.boot</groupId>
    <artifactId>spring-security-oauth2-autoconfigure</artifactId>
    <version>2.1.5.RELEASE</version>
</dependency>
```

#### 1.2. Security Configuration
We need to create a configuration class that extends the `WebSecurityConfigurerAdapter`:
{% highlight java %}
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
	// ...
}
{% endhighlight %}

There we create beans of the `PasswordEncoder` and `AuthenticationManager` types for further use:
{% highlight java %} 
@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();

@Override
@Bean
public AuthenticationManager authenticationManagerBean() throws Exception {
    return super.authenticationManagerBean();
}
{% endhighlight %}	

For the Password Credentials Flow, we do not need any endpoints to be publicly accessible:
{% highlight java %} 
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
            .anyRequest()
            .authenticated();
}
{% endhighlight %}

And we use simple in-memory authentication manager with only one user:
{% highlight java %} 
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth.inMemoryAuthentication()
            .withUser("john")
            .password(passwordEncoder().encode("pass"))
            .roles("USER");
}
{% endhighlight %}
	
#### 1.3. Authorization Configuration
Then we create another configuration class that extends `AuthorizationServerConfigurerAdapter` 
and annotate it with `@EnableAuthorizationServer`:
{% highlight java %} 
@Configuration
@EnableAuthorizationServer
public class AuthServerConfig extends AuthorizationServerConfigurerAdapter {
	// ...
}
{% endhighlight %}

Then we inject the previously created beans into it:
{% highlight java %} 
private PasswordEncoder passwordEncoder;
private AuthenticationManager authenticationManager;

@Autowired
public AuthServerConfig(PasswordEncoder passwordEncoder, AuthenticationManager authenticationManager) {
    this.passwordEncoder = passwordEncoder;
    this.authenticationManager = authenticationManager;
}
{% endhighlight %}
	
We need to set injected `authenticationManager` to `AuthorizationServerEndpointsConfigurer` 
to enable "password" grant type:
{% highlight java %} 
@Override
public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
    endpoints.authenticationManager(authenticationManager);
}
{% endhighlight %}

We use an in-memory service to store client data and add a client to it:
{% highlight java %} 
@Override
public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
    clients.inMemory()
            .withClient("client")
            .secret(passwordEncoder.encode("secret"))
            .authorizedGrantTypes("password")
            .scopes("read");
}
{% endhighlight %}

Finally, we make the `/oauth/check_token` endpoint accessible to authenticated clients:
{% highlight java %} 
@Override
public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
    endpoints.authenticationManager(authenticationManager);
}
{% endhighlight %}

Our authorization server is ready. We can run and test it (by default, it will be started on port `8080`):
{% highlight console %} 
> curl client:secret@localhost:8080/oauth/token -d grant_type=password -d username=john -d password=pass
{"access_token":"a9e7acca-38a2-4c3a-99ce-11b0aabfe09f","token_type":"bearer","expires_in":43199,"scope":"read"}
{% endhighlight %}

We can check the received token:
{% highlight console %} 
> curl client:secret@localhost:8080/oauth/check_token?token=a9e7acca-38a2-4c3a-99ce-11b0aabfe09f
{"exp":1560228050,"user_name":"john","authorities":["ROLE_USER"],"client_id":"client","scope":["read"]}
{% endhighlight %}

Everything works, and we proceed to the implementation of MFA.

## 2. Multi-factor Authentication
We are going to build multi-factor authentication based on [Time-based One-time Passwords (TOTP)](https://tools.ietf.org/html/rfc6238).

#### 2.1. Maven Dependency
To validate TOTP on the server, we need to add the [GoogleAuth library](https://github.com/wstrange/GoogleAuth) to our dependencies.
```
<dependency>
    <groupId>com.warrenstrange</groupId>
    <artifactId>googleauth</artifactId>
    <version>1.1.2</version>
</dependency>
```

#### 2.2. MFA Service
Next, we create a simple service that implements the two required methods:
{% highlight java %} 
@Service
public class MfaService {

    private static final Map<String, String> SECRET_BY_USERNAME = Map.of("john", "JBSWY3DPEHPK3PXP");
    private GoogleAuthenticator googleAuthenticator = new GoogleAuthenticator();

    public boolean isEnabled(String username) {
        return SECRET_BY_USERNAME.containsKey(username);
    }

    public boolean verifyCode(String username, int code) {
        return code == googleAuthenticator.getTotpPassword(SECRET_BY_USERNAME.get(username));
    }
}
{% endhighlight %}
We use the `Map` to store usernames and secrets for the MFA, and assume that if it 
contains the username as a key, then the MFA is enabled for user with this username.

#### 2.3. Resource Owner Password Token Granter
We need to create a custom implementation of the token granter for the "password" grant type.
Its difference from the `ResourceOwnerPasswordTokenGranter` provided by the Spring framework 
is that it throws an exception if the MFA is enabled for the user. 
The exception contains a temporary access token with the `PRE_AUTH` authority, 
which is used for the second authentication step. 
Note that you must ensure that no resources are available to users with this authority.

We implement an exception:
{% highlight java %} 
public class MfaRequiredException extends OAuth2Exception {

    public MfaRequiredException(String mfaToken) {
        super("Multi-factor authentication required");
        this.addAdditionalInformation("mfa_token", mfaToken);
    }

    public String getOAuth2ErrorCode() {
        return "mfa_required";
    }

    public int getHttpErrorCode() {
        return 403;
    }
}
{% endhighlight %}

Then we implement a token granter:
{% highlight java %} 
public class PasswordTokenGranter extends AbstractTokenGranter {
    private static final String GRANT_TYPE = "password";
    private static final GrantedAuthority PRE_AUTH = new SimpleGrantedAuthority("PRE_AUTH");

    private final AuthenticationManager authenticationManager;
    private final MfaService mfaService;

    public PasswordTokenGranter(AuthorizationServerEndpointsConfigurer endpointsConfigurer, AuthenticationManager authenticationManager, MfaService mfaService) {
        super(endpointsConfigurer.getTokenServices(), endpointsConfigurer.getClientDetailsService(), endpointsConfigurer.getOAuth2RequestFactory(), GRANT_TYPE);
        this.authenticationManager = authenticationManager;
        this.mfaService = mfaService;
    }

    @Override
    protected OAuth2Authentication getOAuth2Authentication(ClientDetails client, TokenRequest tokenRequest) {
        Map<String, String> parameters = new LinkedHashMap<>(tokenRequest.getRequestParameters());
        String username = parameters.get("username");
        String password = parameters.get("password");
        parameters.remove("password");
        Authentication userAuth = new UsernamePasswordAuthenticationToken(username, password);
        ((AbstractAuthenticationToken) userAuth).setDetails(parameters);

        try {
            userAuth = this.authenticationManager.authenticate(userAuth);
        } catch (AccountStatusException | BadCredentialsException e) {
            throw new InvalidGrantException(e.getMessage());
        }

        if (userAuth != null && userAuth.isAuthenticated()) {
            OAuth2Request storedOAuth2Request = this.getRequestFactory().createOAuth2Request(client, tokenRequest);
            if (mfaService.isEnabled(username)) {
                userAuth = new UsernamePasswordAuthenticationToken(username, password, Collections.singleton(PRE_AUTH));
                OAuth2AccessToken accessToken = getTokenServices().createAccessToken(new OAuth2Authentication(storedOAuth2Request, userAuth));
                throw new MfaRequiredException(accessToken.getValue());
            }
            return new OAuth2Authentication(storedOAuth2Request, userAuth);
        } else {
            throw new InvalidGrantException("Could not authenticate user: " + username);
        }
    }
}
{% endhighlight %}

#### 2.3. MFA Token Granter
Next, we introduce a new "mfa" grant type which requires the following parameters:
- `grant_type` Value MUST be set to `mfa`.
- `mfa_token`  The temporary access token obtained from Resource Owner Password Token Granter.
- `mfa_code`   The TOTP code

We implement a token granter for this grant type:
{% highlight java %} 
public class MfaTokenGranter extends AbstractTokenGranter {
    private static final String GRANT_TYPE = "mfa";

    private final TokenStore tokenStore;
    private final ClientDetailsService clientDetailsService;
    private final AuthenticationManager authenticationManager;
    private final MfaService mfaService;

    public MfaTokenGranter(AuthorizationServerEndpointsConfigurer endpointsConfigurer, AuthenticationManager authenticationManager, MfaService mfaService) {
        super(endpointsConfigurer.getTokenServices(), endpointsConfigurer.getClientDetailsService(), endpointsConfigurer.getOAuth2RequestFactory(), GRANT_TYPE);
        this.tokenStore = endpointsConfigurer.getTokenStore();
        this.clientDetailsService = endpointsConfigurer.getClientDetailsService();
        this.authenticationManager = authenticationManager;
        this.mfaService = mfaService;
    }

    @Override
    protected OAuth2Authentication getOAuth2Authentication(ClientDetails client, TokenRequest tokenRequest) {
        Map<String, String> parameters = new LinkedHashMap<>(tokenRequest.getRequestParameters());
        final String mfaToken = parameters.get("mfa_token");
        if (mfaToken != null) {
            OAuth2Authentication authentication = loadAuthentication(mfaToken);
            final String username = authentication.getName();
            if (parameters.containsKey("mfa_code")) {
                int code = parseCode(parameters.get("mfa_code"));
                if (mfaService.verifyCode(username, code)) {
                    return getAuthentication(tokenRequest, authentication);
                }
            } else {
                throw new InvalidRequestException("Missing MFA code");
            }
            throw new InvalidGrantException("Invalid MFA code");
        } else {
            throw new InvalidRequestException("Missing MFA token");
        }
    }

    private OAuth2Authentication loadAuthentication(String accessTokenValue) {
        OAuth2AccessToken accessToken = this.tokenStore.readAccessToken(accessTokenValue);
        if (accessToken == null) {
            throw new InvalidTokenException("Invalid access token: " + accessTokenValue);
        } else if (accessToken.isExpired()) {
            this.tokenStore.removeAccessToken(accessToken);
            throw new InvalidTokenException("Access token expired: " + accessTokenValue);
        } else {
            OAuth2Authentication result = this.tokenStore.readAuthentication(accessToken);
            if (result == null) {
                throw new InvalidTokenException("Invalid access token: " + accessTokenValue);
            }
            return result;
        }
    }

    private int parseCode(String codeString) {
        try {
            return Integer.parseInt(codeString);
        } catch (NumberFormatException e) {
            throw new InvalidGrantException("Invalid MFA code");
        }
    }

    private OAuth2Authentication getAuthentication(TokenRequest tokenRequest, OAuth2Authentication authentication) {
        Authentication user = authenticationManager.authenticate(authentication.getUserAuthentication());
        Object details = authentication.getDetails();
        authentication = new OAuth2Authentication(authentication.getOAuth2Request(), user);
        authentication.setDetails(details);

        String clientId = authentication.getOAuth2Request().getClientId();
        if (clientId != null && clientId.equals(tokenRequest.getClientId())) {
            if (this.clientDetailsService != null) {
                try {
                    this.clientDetailsService.loadClientByClientId(clientId);
                } catch (ClientRegistrationException e) {
                    throw new InvalidTokenException("Client not valid: " + clientId, e);
                }
            }
            return refreshAuthentication(authentication, tokenRequest);
        } else {
            throw new InvalidGrantException("Client is missing or does not correspond to the MFA token");
        }
    }

    private OAuth2Authentication refreshAuthentication(OAuth2Authentication authentication, TokenRequest request) {
        Set<String> scope = request.getScope();
        OAuth2Request clientAuth = authentication.getOAuth2Request().refresh(request);
        if (scope != null && !scope.isEmpty()) {
            Set<String> originalScope = clientAuth.getScope();
            if (originalScope == null || !originalScope.containsAll(scope)) {
                throw new InvalidScopeException("Unable to narrow the scope of the client authentication to " + scope + ".", originalScope);
            }

            clientAuth = clientAuth.narrowScope(scope);
        }
        return new OAuth2Authentication(clientAuth, authentication.getUserAuthentication());
    }
}
{% endhighlight %}

#### 2.4. Registration of custom token granters 
We need to register our custom token granters, for this we inject `MfaService` to our `AuthServerConfig`:
{% highlight java %} 
private MfaService mfaService;

@Autowired
public AuthServerConfig(// ...
						MfaService mfaService) {
    // ...
    this.mfaService = mfaService;
}
{% endhighlight %}
Next, we merge the default token granters with ours in the `CompositeTokenGranter`:
{% highlight java %} 
private TokenGranter tokenGranter(final AuthorizationServerEndpointsConfigurer endpoints) {
    List<TokenGranter> granters = new ArrayList<>(List.of(endpoints.getTokenGranter()));
    granters.add(new PasswordTokenGranter(endpoints, authenticationManager, mfaService));
    granters.add(new MfaTokenGranter(endpoints, authenticationManager, mfaService));
    return new CompositeTokenGranter(granters);
}
{% endhighlight %}

And we register it in the `AuthorizationServerEndpointsConfigurer`, after removing 
the `authenticationManager`, so that the default `ResourceOwnerPasswordTokenGranter` is not created:
{% highlight java %} 
@Override
public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
    endpoints.tokenGranter(tokenGranter(endpoints));
}
{% endhighlight %}

Finally, we give our client access to a new grant type:
{% highlight java %} 
@Override
public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
    // ...
            .authorizedGrantTypes("password", "mfa")
    // ...
}
{% endhighlight %}

## 3. Manual testing
#### 3.1. User with enabled MFA
We can start the server and request a token on behalf of a user with MFA enabled, 
using the "password" grant type:
{% highlight console %} 
> curl client:secret@localhost:8080/oauth/token -d grant_type=password -d username=john -d password=pass
{"error":"mfa_required","error_description":"Multi-factor authentication required","mfa_token":"149e20a5-922a-4af6-9778-95cd42b76776"}
{% endhighlight %}

We can check the received token and make sure that it contains the correct `PRE_AUTH` autority:
{% highlight console %} 
> curl client:secret@localhost:8080/oauth/check_token?token=149e20a5-922a-4af6-9778-95cd42b76776
{"exp":1560307040,"user_name":"john","authorities":["PRE_AUTH"],"client_id":"client","scope":["read"]}
{% endhighlight %}

We should get an error if we try to request an access token using the received MFA token 
and a random incorrect code:
{% highlight console %} 
> curl client:secret@localhost:8080/oauth/token -d grant_type=mfa -d mfa_token=149e20a5-922a-4af6-9778-95cd42b76776 -d mfa_code=000000
{"error":"invalid_grant","error_description":"Invalid MFA code"}
{% endhighlight %}

To generate a valid code, we can use [Google Authenticator](https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2&hl=en)
 or another similar application. 
We should scan the QR code or enter the secret code manually:
<br><img src="{{ "/images/posts/2019-06-12/qr.png" | absolute_url }}" alt="" width="160"/>
<br>`JBSWY3DPEHPK3PXP`

Let's request a token with the correct code received from the authenticator app:
{% highlight console %} 
> curl client:secret@localhost:8080/oauth/token -d grant_type=mfa -d mfa_token=149e20a5-922a-4af6-9778-95cd42b76776 -d mfa_code=197214
{"access_token":"149e20a5-922a-4af6-9778-95cd42b76776","token_type":"bearer","expires_in":42994,"scope":"read"}
{% endhighlight %}

We can check the token again to make sure that it contains the correct user role:
{% highlight console %} 
> curl client:secret@localhost:8080/oauth/check_token?token=149e20a5-922a-4af6-9778-95cd42b76776
{"exp":1560307040,"user_name":"john","authorities":["ROLE_USER"],"client_id":"client","scope":["read"]}
{% endhighlight %}

#### 3.2. User with disabled MFA
To make sure that our changes did not affect the users with the MFA turned off, 
we will add a new user and restart the server:
{% highlight java %} 
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth.inMemoryAuthentication()
            // ...
            .and()
            .withUser("anna")
            .password(passwordEncoder().encode("qwerty"))
            .roles("USER");
}
{% endhighlight %}

Then we can request a token on behalf of this user 
and make sure that we get the token with the correct role:
{% highlight console %} 
> curl client:secret@localhost:8080/oauth/token -d grant_type=password -d username=anna -d password=qwerty
{"access_token":"572efff1-77b1-4d11-aef2-3610eed8dc88","token_type":"bearer","expires_in":43199,"scope":"read"}

> curl client:secret@localhost:8080/oauth/check_token?token=572efff1-77b1-4d11-aef2-3610eed8dc88
{"exp":1560307416,"user_name":"anna","authorities":["ROLE_USER"],"client_id":"client","scope":["read"]}
{% endhighlight %}

## 4. Conclusion

We have created a working authorization server with multi-factor authentication support, but for production use it still requires some work that is beyond the scope of this article.

_Full source code can be found on [GitHub](https://github.com/AnarSultanov/examples/tree/master/spring-boot-oauth2-mfa)._

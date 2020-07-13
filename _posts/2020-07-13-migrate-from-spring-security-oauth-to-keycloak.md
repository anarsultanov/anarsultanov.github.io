---
layout: post
title: "Migrate from Spring Security OAuth to Keycloak"
author: "Anar Sultanov"
image: /images/posts/2020-07-13/image.jpg
---
[Spring Security OAuth](https://spring.io/projects/spring-security-oauth) is deprecated, and many of us wonder what to use as an alternative.
Security fixes for the deprecated project will be [supported until May 2022](https://spring.io/blog/2020/05/07/end-of-life-for-spring-security-oauth), 
so we have time to wait for the recently [announced](https://spring.io/blog/2020/04/15/announcing-the-spring-authorization-server) [Spring Authorization Server](https://github.com/spring-projects-experimental/spring-authorization-server), 
or we can migrate to one of the plenty of authorization servers available today.
<br>In this tutorial, we are going to look at how to migrate from Spring Security OAuth to [Keycloak](https://www.keycloak.org/).

## 1. Prerequisites
Let's start by setting up our test servers.

#### 1.1. Spring OAuth2 Server
We will not go into the details of creating an authorization server using Spring Security OAuth 
and will only go through the configuration that interests us as part of this article.

To store the client, we configure an in-memory client details service:
{% highlight java %}
public class AuthServerConfig extends AuthorizationServerConfigurerAdapter {

    // ...
    
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .withClient("client-id")
                .secret(passwordEncoder.encode("client-secret"))
                .authorizedGrantTypes("password", "refresh_token")
                .scopes("read")
                .accessTokenValiditySeconds(600)
                .refreshTokenValiditySeconds(3600);
    }
}
{% endhighlight %}

For user authentication, we will use the database-backed [UserDetailsService](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/core/userdetails/UserDetailsService.html), 
or rather [the implementation](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/provisioning/JdbcUserDetailsManager.html) provided by the framework 
and an H2 database with user schema documented in the [Spring Security Appendix](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#user-schema).

To [initialize the database](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto-initialize-a-database-using-spring-jdbc) 
and populate it with data, we will use SQL scripts: _schema.sql_ and _data.sql_, respectively.
For demonstration, we will use one created user with username `john`, password `pass`, and role `ROLE_USER`.

You can see the full configuration of the authorization server in [this commit](https://github.com/AnarSultanov/examples/commit/19fb572ef216680789df2bd2df405d33eeded883).

#### 1.2. Keycloak Server
To simplify the example, we will use Keycloak server embedded in a Spring Boot application.
I want to warn you that I am not sure how reliable this method is, and I would not use it in production without a detailed analysis.
You can read how to run the pre-configured Keycloak server using Spring Boot in [this article](https://www.baeldung.com/keycloak-embedded-in-spring-boot-app),
find the full configuration in [this commit](https://github.com/AnarSultanov/examples/commit/dbc84242bb72e9fe2ed9c6b8defe4390f943ab57), 
or just download [standalone server distribution](https://www.keycloak.org/downloads).

To start with, we need to understand one of the core concepts of the Keycloak server - realm.
A realm is an isolated unit for managing users, credentials, roles, and groups. Each user belongs to a realm, and realms can only manage and authenticate the users they control.

When we start the Keycloak server, by default, the main realm is created, which gives its admin users the right to view and manage other realms 
and should not be used to manage users and their data.

Keycloak allows us to export and import a realm definition file in a JSON format, which contains everything that can be configured in the administrator console.
We will use such a file to the pre-configure realm on our authorization server. Our [initial file](https://raw.githubusercontent.com/AnarSultanov/examples/dbc84242bb72e9fe2ed9c6b8defe4390f943ab57/spring-boot-oauth2-keycloak/keycloak-server/src/main/resources/realm.json) is the export of a newly created realm with a default configuration. 
In case of using a standalone Keycloak server, we can import this file or just create a new realm using the admin console.

## 2. Migration
To support customization, Keycloak has a number of Service Provider Interfaces (SPI) for which we can implement our providers and use them for migration.

### 2.1. Clients
Unfortunately, [the interface](https://www.keycloak.org/docs-api/10.0/javadocs/org/keycloak/storage/client/ClientStorageProvider.html) for providing an alternative storage mechanism for clients is currently incomplete, 
therefore we will need to migrate our clients manually.

We can create and configure the client or import a JSON file with its configuration in the administrator console, 
but in our case, we will add it to the _clients_ array in our file with realm configuration:
{% highlight json %}
{
  "id": "d9983f84-3163-4302-8e7c-2e0be67ce129",
  "clientId": "client-id",
  "secret": "client-secret",
  "surrogateAuthRequired": false,
  "enabled": true,
  "alwaysDisplayInConsole": false,
  "clientAuthenticatorType": "client-secret",
  "redirectUris": [],
  "webOrigins": [],
  "notBefore": 0,
  "bearerOnly": false,
  "consentRequired": false,
  "standardFlowEnabled": false,
  "implicitFlowEnabled": false,
  "directAccessGrantsEnabled": true,
  "serviceAccountsEnabled": false,
  "publicClient": false,
  "frontchannelLogout": false,
  "protocol": "openid-connect",
  "attributes": {
    "access.token.lifespan": "600",
    "saml.assertion.signature": "false",
    "saml.multivalued.roles": "false",
    "saml.force.post.binding": "false",
    "saml.encrypt": "false",
    "saml.server.signature": "false",
    "saml.server.signature.keyinfo.ext": "false",
    "exclude.session.state.from.auth.response": "false",
    "saml_force_name_id_format": "false",
    "saml.client.signature": "false",
    "tls.client.certificate.bound.access.tokens": "false",
    "saml.authnstatement": "false",
    "display.on.consent.screen": "false",
    "saml.onetimeuse.condition": "false"
  },
  "authenticationFlowBindingOverrides": {},
  "fullScopeAllowed": true,
  "nodeReRegistrationTimeout": -1,
  "defaultClientScopes": [
    "web-origins",
    "role_list",
    "profile",
    "roles",
    "email"
  ],
  "optionalClientScopes": [
    "address",
    "phone",
    "offline_access",
    "microprofile-jwt"
  ]
}
{% endhighlight %}

Let's pay attention to several important parameters so that the client configuration matches the configuration on the legacy server:
```
"clientId": "client-id",
"secret": "client-secret",
"publicClient": false,
"directAccessGrantsEnabled": true,
"access.token.lifespan": "600",
```
All parameters are pretty straightforward, possibly except for _directAccessGrantsEnabled_ which enables the password grant type.

We may also notice that there is no configuration for the refresh token grant type. 
This type is enabled by default, and Keycloak does not allow us to configure the lifespan of refresh tokens for each client individually. 
We can only change the SSO Session Idle setting in the realm that affects all clients:
```
"ssoSessionIdleTimeout": 3600
```

### 2.2. Users
Keycloak provides the User Storage SPI to write extensions for using external user databases and credential stores. 
But there is another strategy for its implementation. Instead of using user federated storage, 
we can create a user in the Keycloak database and copy attributes from our external storage into this local copy.
With this approach, users will be imported from a legacy system into Keycloak when they first log in using Keycloak. 
This is called on-demand migration.

The User Storage SPI has capability interfaces for synchronizing the local Keycloak storage and the external storage, 
but this can quickly become painful and messy, and we will not discuss them in this article.

##### 2.2.1. Remote User Service
Before proceeding with the implementation, we need to decide how we will receive user data from the legacy system. 
We could connect directly to its database, but it is considered bad practice to read data from a database owned by another service.
Therefore, we will create an API on our old authorization server and will use it. 

We need to create endpoints in the Spring OAuth2 server to get user data and validate credentials.
We already have all the services for implementing them, so we will only create a rest controller:
{% highlight java %}
@RestController
@RequestMapping("/users")
public class UserController {

    private final UserDetailsService userDetailsService;
    private final AuthenticationManager authenticationManager;

    public UserController(UserDetailsService userDetailsService, AuthenticationManager authenticationManager) {
        this.userDetailsService = userDetailsService;
        this.authenticationManager = authenticationManager;
    }

    @GetMapping("/{username}")
    public ResponseEntity<UserDetailsDto> getUserDetails(@PathVariable("username") String username) {
        try {
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            List<String> authorities = userDetails.getAuthorities()
                    .stream()
                    .map(GrantedAuthority::getAuthority)
                    .collect(Collectors.toList());
            return ResponseEntity.ok(new UserDetailsDto(userDetails.getUsername(), authorities, userDetails.isEnabled()));
        } catch (UsernameNotFoundException e) {
            return ResponseEntity.notFound().build();
        }
    }

    @PostMapping("/{username}/validate-password")
    public ResponseEntity<?> validateCredentials(@PathVariable("username") String username, @RequestBody PasswordDto dto) {
        try {
            authenticationManager.authenticate(new UsernamePasswordAuthenticationToken(username, dto.getPassword()));
            return ResponseEntity.noContent().build();
        } catch (BadCredentialsException e) {
            return ResponseEntity.badRequest().build();
        }
    }
}
{% endhighlight %}

And for the Keycloak server, we define the appropriate interface that will be used to create the RESTEasy client:
{% highlight java %}
@Path("/users")
@Consumes(MediaType.APPLICATION_JSON)
@Produces(MediaType.APPLICATION_JSON)
public interface RemoteUserService {

    @GET
    @Path("/{username}")
    UserDetailsDto getUserDetails(@PathParam("username") String username);

    @POST
    @Path("/{username}/validate-password")
    Response validateLogin(@PathParam("username") String username, PasswordDto passwordDto);
}
{% endhighlight %}

For transmitting user data, we use a DTO like this:
{% highlight java %}
public class UserDetailsDto {

    private String username;
    private List<String> authorities;
    private boolean enabled;
 
    // ... 
}
{% endhighlight %}

And we need another DTO to transmit the password:
{% highlight java %}
public class PasswordDto {

    private String password;

    // ...
}
{% endhighlight %}

##### 2.2.2. User Storage SPI
To implement User Storage SPI, we must define the provider class and provider factory. 
Provider class instances do all the work of interacting with user data.
Provider factories create instances of the provider class for each transaction.

The main requirement of SPI is the implementation of the [UserStorageProvider](https://www.keycloak.org/docs-api/10.0/javadocs/org/keycloak/storage/UserStorageProvider.html) interface, 
in which only `close()` method inherited from [Provider](https://www.keycloak.org/docs-api/10.0/javadocs/org/keycloak/provider/Provider.html) is required to be implemented, 
and the rest of the methods are not interesting to us. 
When the transaction is completed, this method is invoked and the instance is then garbage collected,
but since we have nothing to close, we simply ignore it:
{% highlight java %}
public class RemoteUserStorageProvider implements UserStorageProvider {

    private final KeycloakSession session;
    private final ComponentModel model;
    private final RemoteUserService userService;

    public RemoteUserStorageProvider(KeycloakSession session, ComponentModel model, RemoteUserService userService) {
        this.session = session;
        this.model = model;
        this.userService = userService;
    }

    @Override
    public void close() {
        // noop
    }
}
{% endhighlight %}
There are other mix-in [interfaces](https://www.keycloak.org/docs/latest/server_development/#provider-capability-interfaces) 
that our provider class may implement to support integration, and we are going to use some of them.

We implement [UserLookupProvider](https://www.keycloak.org/docs-api/10.0/javadocs/org/keycloak/storage/user/UserLookupProvider.html) to be able to receive user data stored by this provider:
{% highlight java %}
public class RemoteUserStorageProvider implements UserStorageProvider, UserLookupProvider {

    // ...
    
    @Override
    public UserModel getUserByUsername(String username, RealmModel realm) {
        UserDetailsDto userDetails = userService.getUserDetails(username);
        UserModel userModel = session.users().addUser(realm, username);
        userModel.setFederationLink(model.getId());
        userModel.setEnabled(userDetails.isEnabled());

        if (userDetails.getAuthorities() != null && !userDetails.getAuthorities().isEmpty()) {
            for (String authority : userDetails.getAuthorities()) {
                RoleModel roleModel = realm.getRole(authority);
                if (roleModel != null) {
                    userModel.grantRole(roleModel);
                }
            }
        }

        return userModel;
    }

    @Override
    public UserModel getUserById(String id, RealmModel realm) {
        return null;
    }

    @Override
    public UserModel getUserByEmail(String email, RealmModel realm) {
        return null;
    }
}
{% endhighlight %}

- `getUserByUsername()` method is invoked when a user logs in. 
We try to get data from our legacy system, and if the user is found we create its local model and copy all the data from the DTO into it.
<br/>Our DTO contains only the status of the user and a list of his authorities, but even if we had more data, 
we could write it all into a model, since it is by default supports basic user metadata, such as name and email address, as well as arbitrary user attributes.
<br/>It should also be noted that to be able to grant the appropriate roles, 
we need to create realm roles that correspond to the authorities in our old authorization server, 
otherwise, the user will be created only with default roles. 
We can do this using the admin console or by editing the realm file and adding [the necessary roles](https://github.com/AnarSultanov/examples/blob/48dc844682c9a0ea9de756bb7a4d34e69fd85236/spring-boot-oauth2-keycloak/keycloak-server/src/main/resources/realm.json#L61) to it.
<br/>And the last thing we should pay attention to in this method is to set up a federation link, 
which forces the authorization server to delegate credential requests for this user to the linked provider.
- `getUserById()` method is not relevant in our implementation since we save users in a local database with a new ID, 
and then their data is read from there, and not from the provider.
- `getUserByEmail()` method returns null since we do not store emails, 
otherwise, we would implement it similarly to `getUserByUsername()` method to allow users to log in using their emails.

Next, we implement [CredentialInputValidator](https://www.keycloak.org/docs-api/10.0/javadocs/org/keycloak/credential/CredentialInputValidator.html) to be able to validate passwords:
{% highlight java %}
public class RemoteUserStorageProvider implements UserStorageProvider, UserLookupProvider, CredentialInputValidator {

    // ...
    
    @Override
    public boolean supportsCredentialType(String credentialType) {
        return credentialType.equals(PasswordCredentialModel.TYPE);
    }

    @Override
    public boolean isConfiguredFor(RealmModel realm, UserModel user, String credentialType) {
        return supportsCredentialType(credentialType);
    }

    @Override
    public boolean isValid(RealmModel realm, UserModel user, CredentialInput input) {
        if (!supportsCredentialType(input.getType())) return false;

        Response response = userService.validateLogin(user.getUsername(), new PasswordDto(input.getChallengeResponse()));
        boolean valid = response.getStatusInfo().getFamily() == Response.Status.Family.SUCCESSFUL;

        if (valid) {
            user.setFederationLink(null);
            session.userCredentialManager().updateCredential(realm, user, input);
        }

        return valid;
    }
}
{% endhighlight %}

- `supportsCredentialType()` method returns whether the provider supports a specific credential type. We just check if the credential type is a password.
- `isConfiguredFor()` method is used to determine if a specific credential type is configured for the user. 
Assuming that all of our users have a password, we can only check the type of credentials, otherwise, we would need to get this information from the legacy system.
- `isValid()` method is responsible for validating passwords. 
We make sure that we support the credential type and make a request to validate the credentials.
If the credentials are valid, we update them in the local storage and unlink the user from the provider.


We are done with the provider and now we need to create a factory for it by implementing the [UserStorageProviderFactory](https://www.keycloak.org/docs-api/10.0/javadocs/org/keycloak/storage/UserStorageProviderFactory.html) interface:
{% highlight java %}
public class RemoteUserStorageProviderFactory implements UserStorageProviderFactory<RemoteUserStorageProvider> {

    public static final String PROVIDER_NAME = "remote";
    public static final String USER_SERVICE_URI = "userServiceUri";

    @Override
    public String getId() {
        return PROVIDER_NAME;
    }
    
    @Override
    public RemoteUserStorageProvider create(KeycloakSession keycloakSession, ComponentModel componentModel) {
        String uri = componentModel.getConfig().getFirst(USER_SERVICE_URI);
        return new RemoteUserStorageProvider(keycloakSession, componentModel, buildClient(uri));
    }

    @Override
    public List<ProviderConfigProperty> getConfigProperties() {
        ProviderConfigProperty userServiceUrl = new ProviderConfigProperty(
                USER_SERVICE_URI,
                "URI",
                "Remote user service URI.",
                ProviderConfigProperty.STRING_TYPE,
                "http://localhost:8080"
        );
        return List.of(userServiceUrl);
    }

    private static RemoteUserService buildClient(String uri) {
        ResteasyClient client = new ResteasyClientBuilder().disableTrustManager().build();
        ResteasyWebTarget target = client.target(uri);
        return target
                .proxyBuilder(RemoteUserService.class)
                .classloader(RemoteUserService.class.getClassLoader())
                .build();
    }
}
{% endhighlight %}

- `getId()` method identifies the factory and will also be the name shown in the admin console.
- `create()` method is called once per transaction to instantiate a provider instance. 
And since [ResteasyClient](https://docs.jboss.org/resteasy/docs/4.5.5.Final/javadocs/org/jboss/resteasy/client/jaxrs/ResteasyClient.html) is not thread-safe, we create a new instance each time and pass it to the provider.
- `getConfigProperties()`method returns a list of [ProviderConfigProperty](https://www.keycloak.org/docs-api/10.0/javadocs/org/keycloak/provider/ProviderConfigProperty.html), 
each of which declares the metadata that is required to store the provider configuration variable. 
This method is optional, but with it, we made our URI customizable, not just hard-coded.


When we are done with the implementation we need to declare the provider factory class within 
the <span class="nobr">`META-INF/services/org.keycloak.storage.UserStorageProviderFactory`</span> file:
```
dev.sultanov.keycloak.server.provider.RemoteUserStorageProviderFactory
```
And in case of using the standalone server, we must place class files for our provider implementation in a jar and copy it into the `standalone/deployments/` directory.

All that remains for us is to enable the user storage provider on the User Federation page in the administration console 
or do this by [adding a new component](https://github.com/AnarSultanov/examples/blob/48dc844682c9a0ea9de756bb7a4d34e69fd85236/spring-boot-oauth2-keycloak/keycloak-server/src/main/resources/realm.json#L1390) into the realm file before starting the pre-configured server.

## 3. Test
Now we can start both of our authorization servers and make sure that everything works.

Get tokens for our user from Spring OAuth2 server:
{% highlight console %} 
> curl client-id:client-secret@localhost:8080/oauth/token -d grant_type=password -d username=john -d password=pass
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1OTQ2NjczNDksInVzZXJfbmFtZSI6ImpvaG4iLCJhdXRob3JpdGllcyI6WyJST0xFX1VTRVIiXSwianRpIjoiMTBmNzFkZWEtZTlkNy00MDk2LWJlMjgtYmM3OTNlMTFlMGViIiwiY2xpZW50X2lkIjoiY2xpZW50LWlkIiwic2NvcGUiOlsicmVhZCJdfQ.ez5sF_eEsDBO7ZT2ksQknbnniquosEg-0mwUBzUp99c","token_type":"bearer","refresh_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJqb2huIiwic2NvcGUiOlsicmVhZCJdLCJhdGkiOiIxMGY3MWRlYS1lOWQ3LTQwOTYtYmUyOC1iYzc5M2UxMWUwZWIiLCJleHAiOjE1OTQ2NzAzNDksImF1dGhvcml0aWVzIjpbIlJPTEVfVVNFUiJdLCJqdGkiOiI0YjE0Njc4OS01NGQxLTRiYmMtYTVjNy1iOWFlOTM3OGE5MTkiLCJjbGllbnRfaWQiOiJjbGllbnQtaWQifQ.qVoNTqZYH0bpm5g3WRr4ZSnyqSYUJYHAKJSDDQG2q-w","expires_in":599,"scope":"read","jti":"10f71dea-e9d7-4096-be28-bc793e11e0eb"}
{% endhighlight %}

Let's try to get tokens for the same user from Keycloak, we use the same credentials for both the client and the user, the only thing we change is the URL:
{% highlight console %} 
> curl client-id:client-secret@localhost:8083/auth/realms/sultanov/protocol/openid-connect/token -d grant_type=password -d username=john -d password=pass
{"access_token":"eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJna0VBRFZ3bEhHNmlINk40NmN3Tmc1ZjYtM1l5VExvdE13NjFZOVVKSDU0In0.eyJleHAiOjE1OTQ2Njc0NDgsImlhdCI6MTU5NDY2Njg0OCwianRpIjoiMTBmMDVmMzYtM2EyMC00MzdiLThjZGEtNWFkMjY0MTExZDM0IiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo4MDgzL2F1dGgvcmVhbG1zL3N1bHRhbm92IiwiYXVkIjoiYWNjb3VudCIsInN1YiI6IjgxMzIzMTZkLWU3YTYtNDE5NS05ZmQ0LWIxOWMxOGY5M2JlMyIsInR5cCI6IkJlYXJlciIsImF6cCI6ImNsaWVudC1pZCIsInNlc3Npb25fc3RhdGUiOiJhMWI1ZDhlMy0xZDVhLTQ5ZjYtYWVhMy00ODFlNzM5YjkzNmQiLCJhY3IiOiIxIiwicmVhbG1fYWNjZXNzIjp7InJvbGVzIjpbIlJPTEVfVVNFUiIsIm9mZmxpbmVfYWNjZXNzIiwidW1hX2F1dGhvcml6YXRpb24iXX0sInJlc291cmNlX2FjY2VzcyI6eyJhY2NvdW50Ijp7InJvbGVzIjpbIm1hbmFnZS1hY2NvdW50IiwibWFuYWdlLWFjY291bnQtbGlua3MiLCJ2aWV3LXByb2ZpbGUiXX19LCJzY29wZSI6InByb2ZpbGUgZW1haWwiLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsInByZWZlcnJlZF91c2VybmFtZSI6ImpvaG4ifQ.nbw3qBiEtf2bI9MGeIEutlH6ebZ0suNWxY6mSRAleG0yk7sQ4GapSUO0qXXSe7jGhjOI0j_6HOl5V5CHPbYL92cGebmp3jqZBeK8dh6R2lK8KmbmlAyPohWIqevr1grcQfjjf3ys3DfxfRFr0KGnWotxu-IlE44lSfV1t0yUiwUbH9wMkMiya389WsgODY3sJa3KqUu1erieaf3eZMw230M7LsucVsv1gHiRqDo5tcen8w-ude6u6LwZmYi0Wv9hOrW9zC9BvV7GxL4A8x3w_x73l1as3bGkAwaNFtthBllPhYaWGsZkDxmbKVlUL3-ENZaBCTsQeQsq2EjMVfxWKw","expires_in":600,"refresh_expires_in":3600,"refresh_token":"eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICIzMGYwOWRjNy0wOWIwLTQyOTctOTAwZS1lN2U5ODBkOWYxYTEifQ.eyJleHAiOjE1OTQ2NzA0NDgsImlhdCI6MTU5NDY2Njg0OCwianRpIjoiYjQ0NDEzNDAtNGI0NS00MTQ4LTgzMzgtYWIyNWJlNTRhMGY0IiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo4MDgzL2F1dGgvcmVhbG1zL3N1bHRhbm92IiwiYXVkIjoiaHR0cDovL2xvY2FsaG9zdDo4MDgzL2F1dGgvcmVhbG1zL3N1bHRhbm92Iiwic3ViIjoiODEzMjMxNmQtZTdhNi00MTk1LTlmZDQtYjE5YzE4ZjkzYmUzIiwidHlwIjoiUmVmcmVzaCIsImF6cCI6ImNsaWVudC1pZCIsInNlc3Npb25fc3RhdGUiOiJhMWI1ZDhlMy0xZDVhLTQ5ZjYtYWVhMy00ODFlNzM5YjkzNmQiLCJzY29wZSI6InByb2ZpbGUgZW1haWwifQ.puZ1dGNKDxeTbuQYyPEZCbQFxPgxvxzckNwAxaxWmFo","token_type":"bearer","not-before-policy":0,"session_state":"a1b5d8e3-1d5a-49f6-aea3-481e739b936d","scope":"profile email"}
{% endhighlight %}

Our user has been successfully migrated to the Keycloak database.
We can verify that the authentication of this user no longer requires any external requests 
by stopping the Spring authorization server and trying to get tokens from Keycloak again.

## 4. Conclusion
Using the on-demand approach for user migration can take some time, but it allows us to migrate user accounts without downtime, 
and reduces the risk of failure, allowing us to return to our previous system with minimal losses in case of unforeseen situations.

_Full source code can be found on [GitHub](https://github.com/AnarSultanov/examples/tree/master/spring-boot-oauth2-keycloak)._

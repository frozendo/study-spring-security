# Spring Security With OAuth2

Spring Security provides features to enable login with **OAuth 2.0** or **OpenID Connect**. For more information on these specifications, read [this](specification).

We can split Spring Secutiry OAuth2 into three concepts: **OAuth 2.0 Login**, **OAuth 2.0 Client** and **OAuth 2.0 Resource Server**.

## OAuth 2.0 Login

The **OAuth 2.0 Login** feature provides the ability to get users to log in to an application using their existing account at an *OAuth 2.0 Provider* or *OpenID Connect Provider*. Spring Boot brings full auto-configuration capabilities for OAuth 2.0 Login.

### Default Provider

Spring Security supports four providers by default: *Google*, *GitHub*, *Facebook* and *Okta*. For these providers, the `CommonOAuth2Provider` class pre-defines a set of default client properties, reducing the number of configurations we need to make when using these providers. The configuration follows the pattern `spring.security.oauth2.client.registration.{registrationId}`, where `{registrationId}` can receive the value from any of the four available providers. For example, when we use Google as our provider, we just need to enter our *client-id* and *client-secret*:

```
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: google-client-id
            client-secret: google-client-secret
```

It's also possible to override the default settings using `spring.security.oauth2.client.registration.{registrationId}` and `spring.security.oauth2.client.provider.{providerId}`. With these options, we can change the default values provided by Spring for each provider, such as the type of authorization grant used or authorization uri that will be called during login.

### Spring Boot Default Options

Spring Boot 2.x brings full auto-configuration capabilities for *OAuth 2.0 Login*, with classes ready to perform all OAuth steps. 

To use this auto-configuration, we first need to register with an available provider and obtain a *client-id* and *client-secret*. At the time of registration, it's necessary to inform the *redirect_uri*, that is the URI where the provider will send the end user after authentication. If we use the default values defined by Spring, the URI will follow the default template `{basicUrl}/login/oauth2/code/{registrationId}`, where `{basicUrl}` is the basic URL of the application, and `{registrationId}` is the provider identifier.

Spring Boot also provides a default auto-generated login page, depending on the provider used. This page is shown when we access the application's URL without prior authentication.

If the registration with the provider was done correctly, and the application configured with the correct values, the authentication code flow will work perfectly, include the request for user information from the *UserInfo Endpoint*, executed by classes defined by Spring Boot.

### Overriding Spring Boot Auto Configuration

Automatic configuration is provided by the `OAuth2ClientAutoConfiguration` class. As always with the Spring Boot applications, it's possible override this if necessary. To do this, we just need to create a new bean for the `ClientRegistrationRepository` and `WebSecurityConfigurerAdapter` classes, with our own logic. 

```
@Configuration
public class OAuth2LoginConfig {

    @EnableWebSecurity
    public static class OAuth2LoginSecurityConfig extends WebSecurityConfigurerAdapter {

        @Override
        protected void configure(HttpSecurity http) throws Exception {
            http
                .authorizeRequests(authorize -> authorize
                    .anyRequest().authenticated()
                )
                .oauth2Login(withDefaults());
        }
    }

    @Bean
    public ClientRegistrationRepository clientRegistrationRepository() {
        return new InMemoryClientRegistrationRepository(this.googleClientRegistration());
    }

    private ClientRegistration googleClientRegistration() {
        return ClientRegistration.withRegistrationId("google")
            .clientId("google-client-id")
            .clientSecret("google-client-secret")
            .scope("openid", "profile", "email", "address", "phone")
            .build();
    }
}
```

The `httpSecurity.oauth2Login` method provides several configuration options for customizing _OAuth 2.0 Login_. The main configuration options are grouped into their protocol endpoint counterparts. For example, it's possible configure all endpoints supported by OAuth and OpenID Connect. The main goal of `oauth2Login` DSL was to calign the nomenclature, with what is defined in the specifications.

We can change the default login page, that originally is auto-generated by the `DefaultLoginPageGeneratingFilter`, with the client registration name as a link, and is able to initiate the authorization request. The link destination for each client follows a default pattern. To change this page, we used the method `oauth2Login().loginPage()`, and provide a controller with an equivalent request mapping. When we do that, we also need to change the URI of the *authorization endpoint* with `oauth2Login().authorizationEndpoint().baseUri()` method. But when we do that, the link for each client needs to match the one configured. 

The *redirection endpoint* can also be changed. As mentioned earlier, the default value is `login/oauth2/code/{registrationId}`. If we want to customize this, we can use the `oauth2Login().redirectionEndpoint().baseUri()` method. This URI need to correspond to a valid endpoint in the application, and we need to ensure that correspondence to redirect uri in the `ClientRegistration.redirectUri`.

The user info endpoint includes several configuration options. It's possible, for example, to change the default class that maps user attributes to an `OAuthAuthenticationToken` object. This can be done by creating a bean of the class `GrantedAuthoritiesMapper` or delegating the mapper to a custom `OAuth2UserService`. The `OAuth2UserService` class takes the user attributes from the *UserInfo Endpoint* and returns an *AuthenticatedPrincipal* in the form of an *OAuth2User*. `DefaultOAuth2UserService` is an implementation of an `OAuth2UserService`, and uses a _RestOperations_ to make these requests. If necessary, it's possible to customize the pre-processing of the *UserInfo Request*, or the post-handling of the *UserInfo Response*.

To work with OpenID Connect, we use `OidcUserService`, another implementation of `OAuth2UserService`. This class takes advantage of `DefaultOAuth2UserService` when requesting the user attributes in *UserInfo Endpoint*. Again, it's possible to customize pre-processing and/or post-handling, and to do that, it just need to provide a customized implementation of `DefaultOAuth2UserService`.

## OAuth 2.0 Client

**The OAuth 2.0 Client** features provide support for the Client role as defined in the OAuth 2.0 Authorization Framework. At a high level, the core features available are **Authorization Grant support** and **Http Client support**.

The `HttpSecurity.oauth2Client` DSL provides several configuration options to customize the core components used by OAuth 2.0 Client. In addition, `HttpSecurity.oauth2Client.authorizationCodeGrant` allows customization of the Authorization Code Grant. 

### Core Interfaces / Classes

`ClientRegistration` is a representation of a client registered with an OAuth 2.0 or OpenID Connect Provider, and stores property values configured in `spring.security.oauth2.client.registration`, such as _registrationId_, _clientId_, _scopes_ and _redirectUri_. A `ClientRegistration` can be initially configured using the discovery of an OpenID Connect Provider's *Configuration endpoint* or an Authorization Server's *Metadata endpoint*. Each instance of `ClientRegistration` has a `ClientRegistrationRepository`, that serves as a repository, provides the ability to retrieve a subset of registered customer information.

`OAuth2AuthorizedClient` is a representation of an *Authorized Client*, that is when the Resource Owner has granted authorization to the client to access their protected resources. This class is responsible for associating an `OAuth2AccessToken` to a `ClientRegistration`. To store an `OAuth2AuthorizedClient` between web requests, we use `OAuth2AuthorizedClientRepository`. Now, to manage this information, we use the `OAuth2AuthorizedClientService` class. From the developer's perspective, theses classes provides the ability to lookup an `OAuth2AccessToken` associated with a client so that it can be used to initiate a protected resource request.

`OAuthAuthorizationClientManager` is the implementation of `AuthenticationManager` responsible for managing OAuth Client authorizations. It's the responsible for managing the flow, from the authorization or re-authorization, passing throught delegating the persistence of an `OAuth2AuthorizedClient`, until delegating the return of a success or an error. The `DefaultOAuth2AuthorizedClientManager` is default implementation of `OAuthAuthorizationClientManager`, but it is designed to be used within the context of a `HttpServletRequest`. When operating out of this context, use `AuthorizedClientServiceOAuth2AuthorizedClientManager` instead.

`OAuth2AuthorizationClientProvider` is the implementation of `AuthenticationProvider` used by `OAuthAuthorizationClientManager`. It implements a strategy to authorizing or re-authorizing an OAuth 2.0 Client. Usually different implementations work with one type of authorization grant.

When an authorization attempt is successful, `OAuth2AuthorizationSuccessHandler` is delegated to save `OAuth2AuthorizedClient`. On the order side, `RemoveAuthorizedClientOAuth2AuthorizationFailureHandler` removes this object when an error occur, as a refresh token no longer valid. The `DefaultOAuth2AuthorizedClientManager` is also associated with a `contextAttributesMapper`, which is responsible for mapping attributes of `OAuth2AuthorizeRequest` to a `Map` of attributes associated to the `OAuth2AuthorizationContext`. The `Function<OAuth2AuthorizeRequest, Map<String, Object>>` type is default, but can be change as needed.

### Authorization Grant Support

*OAuth 2.0 Client* supports different types of authorization grant: **Authorization Code**, **Client Credentials** and **Resource Owner Password Credentials**. Of the three, only *Authorization Code* communicates with Authorization Server's Authorization Endpoint, to get the Authorization Code that will be change to an access token later.

To build the URI to redirect user to the Authorization Server, the `OAuth2AuthorizationRequestRedirectFilter` uses an `OAuth2AuthorizationRequestResolver` to resolve an `OAuthAuthorizationRequest` from the provided web request. The default implementation, `DefaultOAuth2AuthorizationRequestResolver`, matches on the default path `oauth/authorization/{registrationId}`, and uses this to build the request for the associated `ClientRegistration`. `OAuth2AuthorizationRequestResolver` can customize the Authorization Request with additional parameters, and this can be done by adding a `Consumer<OAuth2AuthorizationRequest.Builder>` to the `DefaultOAuth2AuthorizationRequestResolver`. If you need more, it's possibly take full control building the Authorization Request URI overriding the `OAuthAuthorizationRequest.authorizationRequestUri` property. 

While waiting the callback from the Authorization Server, the `AuthorizationRequestRepository` is responsible for the persistence of the `OAuth2AuthorizationRequest` until the moment the Authorization Response is received. `HttpSessionOAuth2AuthorizationRequestRepository` is the default implementation, but we can configure it to use a custom implementation as well.

Once we have a valid Authorization Code, we need to change it to an Access token in the Token Endpoint. The other two grants, *Client Credentials* and *Resource Owner Password Credentials* also comunicate with the Token Endpoint, but directly, without pass by Authorization Endpoint. To do this, it uses an implementation of `OAuth2AccessTokenResponseClient`.

Once we have a valid authorization code, we need to change it to an access token in the token's terminal. The other two grants, * Customer credentials * and * Resource owner password credentials *, also communicate with the token endpoint, but directly, without going through the authorization endpoint. To do this, it uses an implementation of `OAuth2AccessTokenResponseClient`.

* `DefaultAuthorizationCodeTokenResponseClient` is the implementation to request an Access Token in Authorization Code grant
* `DefaultRefreshTokenTokenResponseClient` is the implementation to request a new Access Token with a valid Refresh Token
* `DefaultClientCredentialsTokenResponseClient` is the implementation to Client Credentials grant
* `DefaultPasswordTokenResponseClient` is the implmentation to Resource Owner Password Credentials grant

This four classes uses `RestOperation` for exchanging information with the Authorization Server's Token Endpoint, and they are all very flexible and allow the customization of the pre-processing of the Token Request and/or post-handling of the Token Response.

To customize the pre-processing of the Token Request, we use the `setRequestEntityConverter` defined by interface `OAuthAccessTokenResponseClient`, and we pass a custom Converter, with an request equivalent to the grant type that are used.

On the other end, if you need to customize the post-handling of the Token Response, just use the method `setRestOperations` with a custom configured `RestOperations`.

## OAuth 2.0 Resource Server

Spring Security supports protecting endpoints using two forms of Oauth 2.0 Bearer Tokens: **JWT** and **Opaque Tokens**. At the first look, it works like a *basic authentication*: a security filter intecept the request and verify if it is an authenticated request. If not, so the application denied access. 

### JWT Token

To use the JWT Token in a Spring Boot application, we only need two dependencies: `spring-security-oauth2-resource-server` and `spring-security-oauth2-jose`. For configuration, we just need a simple one: define the **issuer-uri** of the Authorization Server.

```
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://idp.example.com/issuer
```

The `issuer-uri` is the value contained in the `iss` claim for JWT tokens that the authorization server will issue. Resource Server will use this property to further self-configure, discover the authorization server's public keys, and subsequently validate incoming the JWTs.

When the application is started, the Resource Server will automatically configure itself to validate JWT-encoded Bearer Tokens, finalizing the configuration to communicate with the Authorization Server and configure itself to validate JWT tokens. After that, the application will attempt to process any request containing an `Authorization: Bearer` header. When receive the request, the token will be validated using the keys provided by the Authorization Server, and each scope will be mapped to an authority with the prefix `SCOPE_`.

Authentication is done by `JwtAuthenticationProvider`, an implementation of `AuthorizationProvider` that leverages a `JwtDecoder` and `JwtAuthenticationConverter` to authenticate a *JWT*. The authentication filter obtains the token and passes a `BearerTokenAuthentication` to the `ProviderManager`, which triggers the provider. Then, `JwtAuthenticationProvider` decodes, verifies and validates the *JWT*, and if everything is ok, converts the token into a collection of granted authorities. If authentication is successful, the `Authentication` that is returned is of type `JwtAuthenticationToken`, and will be set on `SecurityContextHolder`. 

There are two beans that Spring Boot generates on Resource Server. The first is a `WebSecurityConfigurerAdapter`, that configures the app as resource server. This class can be override by creating a class that extends `WebSecurityConfigurerAdapter`, and defines the `HttpSecurity.configure` method. The second bean is a `JwtDecoder`, which decodes String tokens in validated instances of JWT. This bean can be override by two ways: 

* Replacing the **jwk-set-uri**, using `jwkSetUri` DSL method. This change the uri used by the `JwtDecoder`
* Using the DSL method `decoder`, which will completely replace any Boot auto configuration of `JwtDecoder`

It's also possible to expose a new `JwtDecoder` bean, overriding the default.

The OAuth JWT token will typically either have a `scope` or `scp` attribute, with the scopes (or authorities) it's been granted. Resource Server will create a list of granted authorities with these scopes, prefixing each scope with the string **SCOPE_**. With this, we can protect an endpoint or method using the Spring Security features provided, such as `PreAuthorize`. In some cases, it's necessary change this default. To this, we need to provided a new bean of JwtAuthenticationConverter, and indicates the way that the application will map the scopes into a list of authorities. For example, we can change the prefix uses to **ROLE**, with the  `JwtAuthenticationConverter.setAuthorityPrefix` method. For more flexibility, the DSL supports entirely replacing the converter with any class that implements `Converter<Jwt, AbstractAuthenticationToken>`, and use this class with `jwtAuthenticationConveter` DSL method.

### Opaque Tokens

To work with Opaque Token, we always use `spring-security-oauth2-resource-server` and in some cases we need to add the `oauth2-oidc-sdk` in order to have a working minimal Resource Server that supports opaque Bearer Tokens. 

An opaque token can be verified via an *Introspection Endpoint*, hosted by authorization server. **Introspection Endpoint** is an OAuth 2.0 endpoint that takes a parameter representing an OAuth 2.0 token and returns a JSON document representing the meta information surrounding the token, including whether this token is currently active. To configure an application as a resource server that uses introspection consists of two basic steps. First include the needed dependencies and second, indicate the introspection endpoint details, use the properties configuration: 

``` 
security:
  oauth2:
    resourceserver:
      opaque-token:
        introspection-uri: https://idp.example.com/introspect
        client-id: client
        client-secret: secret
```

Both client-id and client-secret are required, as they are used to validate the resource server against the authorization server. When using introspection, the authorization server's word is the law, so if responses that the token is valid, then it is. The scopes that the token contains are also returned by this endpoint.

When the application start, with these dependencies are used, Resource Server will automatically configure itself to validate Opaque Bearer Tokens. After this, the application will attempt to process any request containing an `Authorization: Bearer` header. When receive the request, the token will be validate with the authorization server using the introspection endpoint, and inspect thee response looking for an attribute `active: true`. After, if everything is ok, map each scope to an authority with the prefix `SCOPE_`.

When the application is started, with these dependencies used, the Resource Server will automatically configure itself to validate Opaque Bearer Tokens. After that, the application will attempt to process any request containing an `Authorization: Bearer` header. Upon receiving the request, the token will be validated with the authorization server using the introspection endpoint, and will inspect the response looking for an `active: true` attribute. Then, if everything is ok, map each scope to an authority with the prefix `SCOPE_`.

Authentication is done by `OpaqueTokenAuthenticationProvider`, an `AuthorizationProvider` implementation that leverages a `OpaqueTokenIntrospector` to authenticate an *Opaque Token*. The authentication filter obtains the token and passes a `BearerTokenAuthentication` to the `ProviderManager`, which triggers the provider. Then,  `OpaqueTokenAuthenticationProvider` introspects the opaque token and adds the granted authorities using an `OpaqueTokenIntrospector`. If authentication is successful, the `Authentication` that is returned is of type `JwtAuthenticationToken`, and will be set on `SecurityContextHolder`. 

There are two beans that Spring Boot generates on the Resource Server when we work with Opaque Token. The first is the same as that used with JWT, the `WebSecurityConfigurerAdapter`, and again, this class can be override by creating a class that extends `WebSecurityConfigurerAdapter`, and defines the `HttpSecurity.configure` method. The second bean is a `OpaqueTokenInstrospector`, which decodes String tokens into validated instances of `OAuthAuthenticatedPrincipal`. This bean can be override by different ways: 

* Overridden the **introspection-uri**, using DSL methods `introspectionUri` and `introspectionClientCredentials`. This changes the uri used by the `OpaqueTokenInstrospector`
* Using the DSL method `introspector`, which will completely replace any Boot auto configuration of `OpaqueTokenInstrospector`

It's also possible to expose a new `OpaqueTokenInstrospector` bean, overriding the default.

The OAuth Introspection Endpointn will typically return a `scope` attribute, with the scopes (or authorities) that have been granted. The Resource Server will create a list of granted authorities with these scopes, prefixing each scope with the string **SCOPE_**. With this, we can protect an endpoint or method using the provided Spring Security features, such as `PreAuthorize`. By default, Opaque Token support will extract the scope claim from an introspection response and parse it into individual `GrantedAuthority` instances, but this can be customized. To do this, we need to provide a new OpaqueTokenInstrospector bean and indicate how the application will map scopes into a list of authorities. 
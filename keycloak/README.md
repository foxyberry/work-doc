# Integration of Authentication in Spring with Keycloak
## Overview
This document explains how to add authentication procedures in a Spring environment and integrate with Keycloak as the authentication server.

1. Keycloak is added as a service in `Docker Compose`, run, and connected to a `PostgreSQL` database to maintain user information.
2. Users authenticate through `Keycloak`. Once a user completes authentication via `Keycloak`, Keycloak issues a `JWT (JSON Web Token)` to the user.
3. The token received through authentication is included in the `Authorization` header when the user sends a request to the Spring server. The Spring server validates the `JWT` to verify the user's authentication status.
4. For the above settings, Spring will add Spring Security security settings and specify which resources require what permissions.
 

### Token Verification Process in Spring Server
```mermaid
sequenceDiagram
    participant User
    participant Client as Spring Application
    participant Keycloak as Keycloak Auth Server

    User->>Client: Send request with JWT in Authorization header
    Note right of Client: JWT Format: Bearer {token}
    Client->>Client: Parse JWT from Authorization header
    Client->>Keycloak: Validate JWT signature
    Keycloak-->>Client: JWT is valid/invalid
    alt JWT is valid
        Client->>Client: Extract roles from JWT
        Client->>Client: Grant access based on roles
        Client-->>User: Access granted to endpoint
    else JWT is invalid
        Client-->>User: Access denied to endpoint
    end
```

## Adding Keycloak Service in Docker Compose
Add the keycloak service definition to your `docker-compose.yml` file.

```yml
version: '3.7'

services:
  postgres:
    image: postgres
    container_name: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: keycloak
    networks:
      - elk
  keycloak:
    image: quay.io/keycloak/keycloak:23.0.4
    volumes:
      - ./realms:/opt/keycloak/data/import/
    container_name: keycloak
    ports:
      - '8888:8888'
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: keycloak
      KC_HTTP_PORT: 8888
      KC_HOSTNAME_URL: http://keycloak:8888
      KC_HOSTNAME_ADMIN_URL: http://keycloak:8888
      #KC_LOG_LEVEL: DEBUG
    command:
      - "start-dev" ## "start"
      - "--import-realm"
    networks:
      - elk
    depends_on:
      - postgres

volumes:
  postgres_data:

networks:
  elk:
    driver: bridge
```

1. container_name: keycloak is needed to fix the name of the keycloak service. This prevents the addition of a prefix to the container service name according to the project name or folder with the `-p` option.

2. A volume setting for synchronizing realm files for automatic import, `./realms:/opt/keycloak/data/import/`, has been added.

3. Persistence is achieved by connecting to a database, using PostgreSQL in this case. For this, the `KC_DB`, `KC_DB_URL`, `KC_DB_USERNAME` variables are set. The domain name `postgres` is used in the `DB URL` because, in the Docker Compose file, the postgres service was set with `container_name: postgres`.

4. `KC_HTTP_PORT` is set to `8888` to specify the access port for keycloak.

5. `KC_HOSTNAME_URL` and `KC_HOSTNAME_ADMIN_URL` are set to designate the default Hostname for `keycloak` use. Internally within Docker, services attempt to connect to each other using their service name (container_name) as the domain. Therefore, when making requests to the keycloak service from inside Spring, it's necessary to use `http://keycloak:8888`. Conversely, it's important for keycloak itself to be aware that its service's domain is `keycloak` to correctly set the `issuer URL` in cookies. If this is not correctly set, resulting in an incorrect `issuer URL`, authentication attempts will fail.

6. The realm file is set to be automatically loaded by starting with `--import-realm`.

7. Since SSL configuration is not used, it starts with `start-dev`.



## Creating Realm, Client, and User in Keycloak Docker Service 
1. The hostname of the keycloak service in Docker is set to `keycloak`. While accessing keycloak internally within Docker at http://keycloak:8888 poses no issue, accessing keycloak externally for admin management requires specifying where the keycloak domain points. Therefore, adding host information is necessary on the server where Docker Desktop is launched. Add the following to `vim /etc/hosts`:
```
127.0.0.1  keycloak
```
2. Launch the keycloak Docker service and access it at `keycloak:8888`. Complete the admin login.
3. Create a new Realm named `app-login-realm`.
4. Create a client named `app-login-client`.
5. Create a realm role named `USER`.
6. Create a user and map the role. Name the user `USER1` and set the password to `USER`.



## Adding Spring Security Configuration
#### application.yml
```yml
spring:
  profiles:
    default: local
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: app-login-client
            authorization-grant-type: authorization_code
            scope: openid
        provider:
          keycloak:
            user-name-attribute: preferred_username
            issuer-uri: http://keycloak:8888/realms/app-login-realm
        resourceserver:
            jwt:
            issuer-uri: http://keycloak:8888/realms/app-login-realm

```
1. registration > keycloak > client-id must be identical to the `client-id` name created in the previous steps.
2. `issuer-uri` used must be identical to the realm name created earlier, `app-login-realm`.

####  KeycloakLogoutHandler.kt
```java
package com.tutti.dig.config

import com.tutti.dig.logger
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.http.ResponseEntity
import org.springframework.security.core.Authentication
import org.springframework.security.oauth2.core.oidc.user.OidcUser
import org.springframework.security.web.authentication.logout.LogoutHandler
import org.springframework.stereotype.Component
import org.springframework.web.client.RestTemplate
import org.springframework.web.util.UriComponentsBuilder


@Component
class KeycloakLogoutHandler : LogoutHandler {

    private val restTemplate: RestTemplate = RestTemplate()
    override fun logout(request: HttpServletRequest?, response: HttpServletResponse?, authentication: Authentication?) {
        logoutFromKeycloak(authentication?.principal as OidcUser)
    }

    private fun logoutFromKeycloak(user: OidcUser) {
        val endSessionEndpoint = user.issuer.toString() + "/protocol/openid-connect/logout"
        val builder = UriComponentsBuilder
            .fromUriString(endSessionEndpoint)
            .queryParam("id_token_hint", user.idToken.tokenValue)
        val logoutResponse: ResponseEntity<String> = restTemplate.getForEntity(
            builder.toUriString(), String::class.java
        )
        if (logoutResponse.statusCode.is2xxSuccessful) {
            logger.info("Successfully logged out from Keycloak")
        } else {
            logger.error("Could not propagate logout to Keycloak")
        }
    }
}
```

#### SecurityConfig.kt

```java
package com.tutti.dig.config

import org.springframework.boot.autoconfigure.security.servlet.PathRequest
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.authentication.AuthenticationManager
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity
import org.springframework.security.config.http.SessionCreationPolicy
import org.springframework.security.core.GrantedAuthority
import org.springframework.security.core.authority.SimpleGrantedAuthority
import org.springframework.security.oauth2.jwt.Jwt
import org.springframework.security.oauth2.jwt.JwtDecoder
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.util.matcher.AntPathRequestMatcher


@EnableWebSecurity
@Configuration
class SecurityConfig(
        private val keycloakLogoutHandler: KeycloakLogoutHandler,
        private val jwtDecoder: JwtDecoder
) {

    @Bean
    @Throws(Exception::class)
    fun resourceServerFilterChain(http: HttpSecurity): SecurityFilterChain? {

        http.cors().and().csrf().disable()
                .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS) 
                .and()
                .authorizeHttpRequests()
                .requestMatchers(AntPathRequestMatcher("/swagger-ui/**"), AntPathRequestMatcher("/v3/api-docs/**"), AntPathRequestMatcher("/swagger-ui.html"), AntPathRequestMatcher("/webjars/**"), AntPathRequestMatcher("/swagger-resources/**"))
                .permitAll()
                .requestMatchers(AntPathRequestMatcher("/v1*"))
                .hasRole("USER")
                .anyRequest()
                .authenticated()

        http.oauth2ResourceServer { oauth2 ->
            oauth2.jwt().jwtAuthenticationConverter(jwtAuthenticationConverter()).decoder(jwtDecoder)
        }
        return http.build()
    }

    private fun jwtAuthenticationConverter(): JwtAuthenticationConverter {
        val converter = JwtAuthenticationConverter()
        converter.setJwtGrantedAuthoritiesConverter(KeycloakJwtGrantedAuthoritiesConverter())
        return converter
    }

    @Bean
    @Throws(Exception::class)
    fun authenticationManager(http: HttpSecurity): AuthenticationManager {
        return http.getSharedObject(AuthenticationManagerBuilder::class.java).build()
    }

}

class KeycloakJwtGrantedAuthoritiesConverter :
        org.springframework.core.convert.converter.Converter<Jwt, Collection<GrantedAuthority>> {
    override fun convert(jwt: Jwt): Collection<GrantedAuthority> {
        val realmAccess = jwt.claims["realm_access"] as? Map<String, Any>
        val roles = realmAccess?.get("roles") as? List<String>

        return roles?.map { roleName -> SimpleGrantedAuthority("ROLE_$roleName") }?.toSet()
                ?: emptySet()
    }
}
```
1. Documentation for Swagger is accessible without requiring authentication.
2. URLs starting with `/v1*` require `USER` permissions.


## Obtaining Authentication Cookie from Keycloak
Authenticate against Keycloak using the user's username and password. Upon verification of user information, a JWT cookie is issued. Below is an example curl request with the username `user1` and password `user1`. The `client_id `must match the one set earlier, and the grant type is set to `password`.


```shell
curl --location 'http://keycloak-server-domain:8888/realms/app-login-realm/protocol/openid-connect/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'client_id=app-login-client' \
--data-urlencode 'username=user1' \
--data-urlencode 'password=user1' \
--data-urlencode 'grant_type=password'

```
The result from the above request can be as follows:

```
{
    "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJMaU1Hd0ZHT0pWUFpxcjFuNnF4X3Z0MWVhTXJmbzIxeGxqQl91TWszNTYwIn0.eyJleHAiOjE3MTA3NzQ0ODQsImlhdCI6MTcxMDc3NDE4NCwianRpIjoiZGE1YmI5MDEtNmE4Ny00NjA5LTlhMzAtOGUwNjVkNzFmYzMxIiwiaXNzIjoiaHR0cDovL2tleWNsb2FrOjg4ODgvcmVhbG1zL2FwcC1sb2dpbi1yZWFsbSIsImF1ZCI6ImFjY291bnQiLCJzdWIiOiIzNTc3MDkxYy04NDAyLTQ3YTAtYWY5OC04ODc3YTE1MDM3ODMiLCJ0eXAiOiJCZWFyZXIiLCJhenAiOiJhcHAtbG9naW4tY2xpZW50Iiwic2Vzc2lvbl9zdGF0ZSI6Ijc3MmNmYjI3LWQ4NmEtNGNmYS05ODFlLTEzN2Y2MjViY2Y2ZCIsImFjciI6IjEiLCJhbGxvd2VkLW9yaWdpbnMiOlsiLyoiXSwicmVhbG1fYWNjZXNzIjp7InJvbGVzIjpbImRlZmF1bHQtcm9sZXMtYXBwLWxvZ2luLXJlYWxtIiwib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiIsInVzZXIiXX0sInJlc291cmNlX2FjY2VzcyI6eyJhY2NvdW50Ijp7InJvbGVzIjpbIm1hbmFnZS1hY2NvdW50IiwibWFuYWdlLWFjY291bnQtbGlua3MiLCJ2aWV3LXByb2ZpbGUiXX19LCJzY29wZSI6ImVtYWlsIHByb2ZpbGUiLCJzaWQiOiI3NzJjZmIyNy1kODZhLTRjZmEtOTgxZS0xMzdmNjI1YmNmNmQiLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsInByZWZlcnJlZF91c2VybmFtZSI6InVzZXIxIn0.CsNbz-yxd5poDeUG-X6rz_DqU6i7iPGYR7Jk-D7Yp78mKyJjJsScTgqaFtIpDcOS8aHtZFInUxvicy6iLnfb-CaNFSu4uHx65k3u0xGr2ph5_ZBYuCy7SPPEb0FcuNljqFhnHkM9w3MZCvGksLWjkKol3ZZGTjOBu4uJ4svhsY973dEu0_z-GxyU7V5fxAsbNdY_l-REhz1cvnki9vtv3QgWJt6V8nxnbezKk08-Dbl7RXGMFf5zLvKoGEBBctesLFyagf2MfiT_YkeQbglljm-h0wxUe1B2cZUf71Go0sjQPNmBQEbLHL6_7zBhb9_7JdZsj61dyJ9cUe7HbGgcGA",
    "expires_in": 300,
    "refresh_expires_in": 1800,
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICI1OTYyZjIwOC00ZTdkLTQ2YWUtODU5Yi1mZDUyYzkyMjZjZjAifQ.eyJleHAiOjE3MTA3NzU5ODQsImlhdCI6MTcxMDc3NDE4NCwianRpIjoiMDFlNDcyODMtOGJmMi00NzQ5LTk5MGYtMTlhMzQ3NTYzNjEzIiwiaXNzIjoiaHR0cDovL2tleWNsb2FrOjg4ODgvcmVhbG1zL2FwcC1sb2dpbi1yZWFsbSIsImF1ZCI6Imh0dHA6Ly9rZXljbG9hazo4ODg4L3JlYWxtcy9hcHAtbG9naW4tcmVhbG0iLCJzdWIiOiIzNTc3MDkxYy04NDAyLTQ3YTAtYWY5OC04ODc3YTE1MDM3ODMiLCJ0eXAiOiJSZWZyZXNoIiwiYXpwIjoiYXBwLWxvZ2luLWNsaWVudCIsInNlc3Npb25fc3RhdGUiOiI3NzJjZmIyNy1kODZhLTRjZmEtOTgxZS0xMzdmNjI1YmNmNmQiLCJzY29wZSI6ImVtYWlsIHByb2ZpbGUiLCJzaWQiOiI3NzJjZmIyNy1kODZhLTRjZmEtOTgxZS0xMzdmNjI1YmNmNmQifQ.LfJT2bCNPP__VS2S58qe7YS-ZUSdAb7VBmWTjh5hFAM",
    "token_type": "Bearer",
    "not-before-policy": 0,
    "session_state": "772cfb27-d86a-4cfa-981e-137f625bcf6d",
    "scope": "email profile"
}
```

Decoding the `access_token` at jwt.io site reveals the following user and token information:
```
{
  "exp": 1710774484,
  "iat": 1710774184,
  "jti": "da5bb901-6a87-4609-9a30-8e065d71fc31",
  "iss": "http://keycloak:8888/realms/app-login-realm",
  "aud": "account",
  "sub": "3577091c-8402-47a0-af98-8877a1503783",
  "typ": "Bearer",
  "azp": "app-login-client",
  "session_state": "772cfb27-d86a-4cfa-981e-137f625bcf6d",
  "acr": "1",
  "allowed-origins": [
    "/*"
  ],
  "realm_access": {
    "roles": [
      "default-roles-app-login-realm",
      "offline_access",
      "uma_authorization",
      "user"
    ]
  },
  "resource_access": {
    "account": {
      "roles": [
        "manage-account",
        "manage-account-links",
        "view-profile"
      ]
    }
  },
  "scope": "email profile",
  "sid": "772cfb27-d86a-4cfa-981e-137f625bcf6d",
  "email_verified": false,
  "preferred_username": "user1"
}
```

The `access_token` obtained must be included in the `Authorization` header for subsequent requests to the Spring server. Here's an example request where the `access_token` is used:

```
curl --location 'http://spring-server-domain/v1/reports/geo-query?polygon=126.90727686146623%2C37.53054723433391&polygon=126.90835778517184%2C37.53085785794988&polygon=126.90949701621935%2C37.52790593739182&polygon=126.90832966272394%2C37.52759509171798' \
--header 'Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJMaU1Hd0ZHT0pWUFpxcjFuNnF4X3Z0MWVhTXJmbzIxeGxqQl91TWszNTYwIn0.eyJleHAiOjE3MTA3NzQ0ODQsImlhdCI6MTcxMDc3NDE4NCwianRpIjoiZGE1YmI5MDEtNmE4Ny00NjA5LTlhMzAtOGUwNjVkNzFmYzMxIiwiaXNzIjoiaHR0cDovL2tleWNsb2FrOjg4ODgvcmVhbG1zL2FwcC1sb2dpbi1yZWFsbSIsImF1ZCI6ImFjY291bnQiLCJzdWIiOiIzNTc3MDkxYy04NDAyLTQ3YTAtYWY5OC04ODc3YTE1MDM3ODMiLCJ0eXAiOiJCZWFyZXIiLCJhenAiOiJhcHAtbG9naW4tY2xpZW50Iiwic2Vzc2lvbl9zdGF0ZSI6Ijc3MmNmYjI3LWQ4NmEtNGNmYS05ODFlLTEzN2Y2MjViY2Y2ZCIsImFjciI6IjEiLCJhbGxvd2VkLW9yaWdpbnMiOlsiLyoiXSwicmVhbG1fYWNjZXNzIjp7InJvbGVzIjpbImRlZmF1bHQtcm9sZXMtYXBwLWxvZ2luLXJlYWxtIiwib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiIsInVzZXIiXX0sInJlc291cmNlX2FjY2VzcyI6eyJhY2NvdW50Ijp7InJvbGVzIjpbIm1hbmFnZS1hY2NvdW50IiwibWFuYWdlLWFjY291bnQtbGlua3MiLCJ2aWV3LXByb2ZpbGUiXX19LCJzY29wZSI6ImVtYWlsIHByb2ZpbGUiLCJzaWQiOiI3NzJjZmIyNy1kODZhLTRjZmEtOTgxZS0xMzdmNjI1YmNmNmQiLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsInByZWZlcnJlZF91c2VybmFtZSI6InVzZXIxIn0.CsNbz-yxd5poDeUG-X6rz_DqU6i7iPGYR7Jk-D7Yp78mKyJjJsScTgqaFtIpDcOS8aHtZFInUxvicy6iLnfb-CaNFSu4uHx65k3u0xGr2ph5_ZBYuCy7SPPEb0FcuNljqFhnHkM9w3MZCvGksLWjkKol3ZZGTjOBu4uJ4svhsY973dEu0_z-GxyU7V5fxAsbNdY_l-REhz1cvnki9vtv3QgWJt6V8nxnbezKk08-Dbl7RXGMFf5zLvKoGEBBctesLFyagf2MfiT_YkeQbglljm-h0wxUe1B2cZUf71Go0sjQPNmBQEbLHL6_7zBhb9_7JdZsj61dyJ9cUe7HbGgcGA'

```
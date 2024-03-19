# Spring 인증 설정과 keycloak 연동 
## 개요
이번 문서에서는 Spring 환경에서 인증 절차를 추가하고, keycloak을 인증 서버로 사용하는 연동을 설명합니다.  

1. keycloak은 도커 컴포즈에 서비스로 추가하여 실행하고, postgres DB 연결하여 유저의 정보를 유지합니다.
2. 유저는 Keycloak을 통해 인증을 합니다. 유저가 Keycloak을 통해 인증을 완료 하면, Keycloak은 유저에게 JWT를 발급합니다. 
3. 인증을 통해 받은 토큰은 유저가 Spring 서버에 요청을 보낼 떄, Authorization 헤더에 포함되어 서버로 전송됩니다. Spring 서버는 JWT 를 검증하여 유저의 인증 상태를 확인합니다. 
4. 위의 설정을 위하여 Spring에서는 Spring Security 보안 설정을 추가할 것이고, 어떤 리소스에 어떤 권한이 필요한지를 명세합니다. 

### Spring 서버의 토큰 검증 과정 
```Mermaid
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

## 도커 컴포즈에 Keycloak 서비스 추가
docker-compose.yml 파일에 keycloak 서비스 정의를 추가합니다.

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

1. keycloak 서비스의 이름을 keycloak으로 고정하기 위해서 `container_name: keycloak` 설정이 필요하다. 이는 -p 옵션으로 프로젝트 이름이나 폴더에 따른 컨테이너 서비스 이름에 prefix가 추가 되는것을 방지하기 위해서이다.
2. realm 파일을 자동으로 import 하기 위하여, reaml 파일을 동기화 할 볼륨 설정이 필요하다. `./realms:/opt/keycloak/data/import/` 으로 볼륨 설정을 추가하였다.
3. 영속성을 위해 DB를 연결했고, DB로는 postgres를 사용했다. 이를 위해서 `KC_DB`, `KC_DB_URL`, `KC_DB_USERNAME` 변수를 세팅했다. DB URL에 postgres를 도메인 이름으로 쓴 것은,  도커 컴포즈 파일안에서 postgres 서비스를 설정할 때, `container_name: postgres`로 설정했기 때문이다. 
4. `KC_HTTP_PORT` 를 설정하여 keycloak 접속 포트를 8888로 설정하였다.
5. `KC_HOSTNAME_URL` 와 `KC_HOSTNAME_ADMIN_URL`를 설정하여 keycloak에서 사용할 기본 Hostname 을 지정하였다. 이를 지정한 이유는 다음과 같다. 도커 내부에서 각 서비스로 연결을 시도할 때는 서로의 서비스명(container_name)을 기본으로 사용한다. 우리가 keycloak의 서비스의 컨테이너 이름을 keycloak으로 설정했기 때문에 (`container_name: keycloak`) 스프링 내부에서 keycloak 서비스로 요청을 보낼 때는 `http://keycloak:8888` 으로 요청을 보내야 한다. 반면, keycloak 입장에서도 본인 서비스의 도메인이 `keycloak`이 되었다는 것을 알아야 한다. 기준이 될 hostname이 무언인지 알아야 쿠키를 구울 때 그 기준으로 issuerurl 값을 만든다. 이 값이 제대로 설정되지 않아 issuerurl 값이 제대로 나오지 않을 경우, 인증 시도시 실패한다.

6. `--import-realm` 으로 시작해서 realm 파일을 자동 로딩하게 설정하였다. 

7. ssl 설정 사용을 하지 않기 때문에, start-dev로 시작하였다.  


## 키클락 도커 서비스에 realm 과 client와 유저 생성 
1. keycloak의 hostname을 설정했기 때문에 docker 를 띄운 서버에서 hosts 정보 추가가 필요하다. vim /etc/hosts에 다음을 추가한다.
```
127.0.0.1  keycloak
```
2. keycloak을 도커 서비스를 띄우고, keycloak:8888로 접속한다. 
3. Realm 을 새로 생성한다. 이름은 `app-login-realm` 으로 지정했다.
4. client를 생성한다. 이름은 ` app-login-client` 으로 만들었다.
5. realm role을 만든다. 롤 이름을 `USER` 으로 만들었다. 
6. user를 생성하고, role 매핑을 한다. 유저의 이름을 `USER1` 으로 하고, 비밀번호도 `USER` 으로 지정했다.



## Spring Security 설정 추가 
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
1. registration > keycloak > client-id 는 위의 단계에서 생성한 client-id 이름과 동일해야 한다. 
2. issuer-uri에서 사용하고 있는 `app-login-realm` 은 위에서 생성한 realm 이름과 동일 해야 한다. 


####  KeycloakLogoutHandler.kt
```java
package com.tutti.dpnc.config

import com.tutti.dpnc.logger
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
package com.tutti.dpnc.config

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
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS) // 세션을 사용하지 않음
                .and()
                .authorizeHttpRequests()
                .requestMatchers(AntPathRequestMatcher("/swagger-ui/**"), AntPathRequestMatcher("/v3/api-docs/**"), AntPathRequestMatcher("/swagger-ui.html"), AntPathRequestMatcher("/webjars/**"), AntPathRequestMatcher("/swagger-resources/**"))
                .permitAll() // Swagger 관련 경로에 대해 인증을 요구하지 않음
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
1. swagger 에 대한 문서는 권한 없이 접속할 수 있게 하였다. 
2. `/v1*` 로 시작하는 url은 `USER` 권한이 있어야 한다. 


## keycloak으로부터 인증 쿠키 발급 방법
유저의 이름과 비밀번호 기반으로 Keycloak에 인증을 시도한다. 유저 정보가 확인되면 JWT 쿠키가 발급된다. 
아래는 curl 요청 예시이다. user의 이름은 `user1`이고, 비밀번호는 `user1`이다. `client_id`는 위에서 설정한 `client id`가 되어야 한다. grant_type은 password 로 설정해둔다. 

```shell
curl --location 'http://서버도메인:8888/realms/app-login-realm/protocol/openid-connect/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'client_id=app-login-client' \
--data-urlencode 'username=user1' \
--data-urlencode 'password=user1' \
--data-urlencode 'grant_type=password'

```
위의 요청으로 결과를 받을 경우 다음과 같은 값을 받을 수 있다.

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

그리고 위의 값에서 `access_token`을 jwt.io 사이트에서 decode 확인해보면 다음처럼 해당 유저와 토큰에 대한 정보가 들어 았따. 

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


위에서 받은 JWT 쿠키에는 access_token 값이 있는데, 이 값은 이후 Spring 서버에 요청을 보낼때마다 헤더값에 추가가 되어야 한다. 다음은 그 요청의 예이다. `access_token` 의 값을 Authorization에 넣으면 된다. 
```
curl --location 'http://abcdegf1234.asuscomm.com:8080/v1/reports/geo-query?polygon=126.90727686146623%2C37.53054723433391&polygon=126.90835778517184%2C37.53085785794988&polygon=126.90949701621935%2C37.52790593739182&polygon=126.90832966272394%2C37.52759509171798' \
--header 'Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJMaU1Hd0ZHT0pWUFpxcjFuNnF4X3Z0MWVhTXJmbzIxeGxqQl91TWszNTYwIn0.eyJleHAiOjE3MTA3NzQ0ODQsImlhdCI6MTcxMDc3NDE4NCwianRpIjoiZGE1YmI5MDEtNmE4Ny00NjA5LTlhMzAtOGUwNjVkNzFmYzMxIiwiaXNzIjoiaHR0cDovL2tleWNsb2FrOjg4ODgvcmVhbG1zL2FwcC1sb2dpbi1yZWFsbSIsImF1ZCI6ImFjY291bnQiLCJzdWIiOiIzNTc3MDkxYy04NDAyLTQ3YTAtYWY5OC04ODc3YTE1MDM3ODMiLCJ0eXAiOiJCZWFyZXIiLCJhenAiOiJhcHAtbG9naW4tY2xpZW50Iiwic2Vzc2lvbl9zdGF0ZSI6Ijc3MmNmYjI3LWQ4NmEtNGNmYS05ODFlLTEzN2Y2MjViY2Y2ZCIsImFjciI6IjEiLCJhbGxvd2VkLW9yaWdpbnMiOlsiLyoiXSwicmVhbG1fYWNjZXNzIjp7InJvbGVzIjpbImRlZmF1bHQtcm9sZXMtYXBwLWxvZ2luLXJlYWxtIiwib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiIsInVzZXIiXX0sInJlc291cmNlX2FjY2VzcyI6eyJhY2NvdW50Ijp7InJvbGVzIjpbIm1hbmFnZS1hY2NvdW50IiwibWFuYWdlLWFjY291bnQtbGlua3MiLCJ2aWV3LXByb2ZpbGUiXX19LCJzY29wZSI6ImVtYWlsIHByb2ZpbGUiLCJzaWQiOiI3NzJjZmIyNy1kODZhLTRjZmEtOTgxZS0xMzdmNjI1YmNmNmQiLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsInByZWZlcnJlZF91c2VybmFtZSI6InVzZXIxIn0.CsNbz-yxd5poDeUG-X6rz_DqU6i7iPGYR7Jk-D7Yp78mKyJjJsScTgqaFtIpDcOS8aHtZFInUxvicy6iLnfb-CaNFSu4uHx65k3u0xGr2ph5_ZBYuCy7SPPEb0FcuNljqFhnHkM9w3MZCvGksLWjkKol3ZZGTjOBu4uJ4svhsY973dEu0_z-GxyU7V5fxAsbNdY_l-REhz1cvnki9vtv3QgWJt6V8nxnbezKk08-Dbl7RXGMFf5zLvKoGEBBctesLFyagf2MfiT_YkeQbglljm-h0wxUe1B2cZUf71Go0sjQPNmBQEbLHL6_7zBhb9_7JdZsj61dyJ9cUe7HbGgcGA'

```
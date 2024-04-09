# 스프링 시큐리티로 JWT 구현하기

> 스프링에서 제공하는 샘플을 정리하였습니다.
>
> [spring-security-samples/servlet/spring-boot/java/jwt/login at main · spring-projects/spring-security-samples](https://github.com/spring-projects/spring-security-samples/tree/main/servlet/spring-boot/java/jwt/login)

> 작성 당시 스프링 버전은 아래와 같습니다.
> SpringBoot : 3.2.4
> SpringSecurity : 6.1.5

## WebSecurityConfigurerAdapter

**5.7.0-M2** 버전부터 `WebSecurityConfigurerAdapter` 클래스가 deprecated 되었다. 그래서 더 이상 해당 클래스를 상속해서 설정하는 방식은 사용되지 않는다.

```jsx
@Configuration
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests((authz) -> authz
                .anyRequest().authenticated()
            )
            .httpBasic(withDefaults());
    }
}
```

스프링 시큐리티에서는 이제 `SecurityFilterChain` 을 **빈으로** 등록해서 사용하는 방법을 추천하고 있다.

```jsx
@Configuration
public class SecurityConfiguration {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests((authz) -> authz
                .anyRequest().authenticated()
            )
            .httpBasic(withDefaults());
        return http.build();
    }
}
```

스프링 시큐리티 샘플에도 이 방법을 사용하고 있기 때문에 해당 방법으로 진행해볼 예정이다.

아래는 `WebSecurityConfigurerAdapter` 을 사용하던 방법에서 어떻게 변경해야 하는지를 가이드해주고 있다.

[Spring Security without the WebSecurityConfigurerAdapter](https://spring.io/blog/2022/02/21/spring-security-without-the-websecurityconfigureradapter)

### authorizeHttpRequests

`authorizeHttpRequests` 는 HTTP 요청에 대한 **인가** 설정을 지정할 수 있도록 도와준다.

> **인가(Authorization)** : 권한을 부여하는 행위. 해당 주체가 어떤 권한을 가지고 있는지 결정하는 과정이다.
> **인증(Authentication)** : 누구인지 확인하는 과정.

`authorizeHttpRequests` 는 5.5 버전부터는 매개변수로 람다식을 전달할 수 있는 오버로드 된 메서드가 추가되었다. 기존에 사용하던 메서드는 6.1버전 부터는 deprecated 되었다.

```jsx
// 해당 방법은 deprecated 되었다.
http.authorizeRequests().anyRequest().authenticated();
```

추천하는 방법은 새롭게 추가된 람다식을 매개변수로 받는 메서드를 사용한다. 사용방법은 (요청 → 누구에게 인가) 와 같은 방식으로 코드를 작성한다.

> `anyRequest().authenticated()`

예를 들어 `anyRequest()` 는 모든 요청에 대해서라는 **요청**을 나타내고, `authenticated()` 는 인증받은 사용자에게라는 **누구에게 인가**해줄지에 대한 내용을 작성한다.

```jsx
http
  .authorizeHttpRequests((authorize) -> authorize
          .anyRequest().authenticated()
  );
```

요청을 나타내는 메서드는 아래와 같다.

- **anyRequest**
- **requestMatchers**
  - requestMatchers("/resources/\*\*", "/signup", "/about")
  - requestMatchers(HttpMethod.GET, "/token")
  - requestMatchers(matchers -> matchers.getMethod() == "GET" && matchers.isUserInRole("ADMIN"))
    - matcher를 활용해서 다양하게 설정가능.

인가를 나타내는 메서드는 아래와 같다.

- **permitAll**
  - 모든 사용자에게 인가
- **hasRole(”Role”)**
  - 특정 역할을 가진 사용자에게 인가
- **authenticated**
  - 인증 받은 사용자만 인가
- **hasAuthority**
  - 특정 권한을 가진 사용자에게 인가
- 등등

### CSRF

> `CSRF 란`
> 크로스 사이트 요청 위조(CSRF라고도 함)는 공격자가 사용자가 의도하지 않은 작업을 수행하도록 유도할 수 있는 웹 보안 취약점이다. 공격자가 인증 된 브라우저에 저장된 쿠키의 세션 정보를 활용해 실제 사용자인 척 속여 의도치 않은 작업을 수행하게 만든다.

CSRF의 경우 세션을 통해서 공격하는 방식이기 때문에 JWT를 사용하는 경우 CSRF 설정을 disabled 하는 경우가 많다고 한다.

```jsx
.csrf((csrf) -> csrf.disable())
```

일단 예제에서는 토큰을 발급하기 위한 주소인 `/token` 에 대해서만 CSRF 토큰을 인증하지 않도록 설정하고 있다.

```jsx
csrf((csrf) -> csrf.ignoringRequestMatchers("/token"))
```

### httpBasic

해당 메서드는 HTTP Basic 인증을 사용하도록 설정한다. HTTP Basic 인증이란 클라이언트에서 사용자 이름과 비밀번호를 Base64로 인코딩하여 Authorization 헤더에 담아 전송하여 인증하는 방식이다.

```jsx
httpBasic(Customizer.withDefaults());
```

`Customizer.withDefaults()` 는 스프링 시큐리티에서 제공하는 기본값을 사용하여 보안 기능을 활성화한다.

> `Customizer.withDefaults()`
> 스프링 시큐리티에 다른 설정을 보면 `Customizer.withDefaults()` 로 많이 사용되고 있는 것을 확인하였다. 이걸 사용해서 전달하면 스프링 시큐리티에서 제공하는 **기본값**을 사용한다는 의미라고 한다.

`Customizer.withDefaults()`를 사용하면 다음과 같은 기본 설정이 적용된다.

1. **인증 Entry Point** : `BasicAuthenticationEntryPoint` 를 사용하여 인증되지 않은 요청에 대해 HTTP 401 Unauthorized 응답을 반환
2. **인증 실패 처리** : `Http403ForbiddenEntryPoint` 를 사용하여 인증에 실패한 요청에 대해 HTTP 403 Forbidden 응답을 반환
3. **인증된 사용자 정보 저장** : `SecurityContextPersistenceFilter` 를 사용하여 인증된 사용자 정보를 `SecurityContextHolder` 에 저장

### oauth2ResourceServer

해당 설정을 하기 위해서는 JWT를 사용할때 사용하는 옵션이다. 스프링 시큐리티에서 OAuth 2.0 리소스 서버를 구성하고 JWT를 사용하여 인증하도록 설정하는 코드이다.

해당 설정이 있어야 요청 헤더에 JWT 토큰을 포함하여 요청 하는 경우 서버에서 이 토큰을 검증하고 인증정보를 추출한다.

```jsx
oauth2ResourceServer((oauth2) -> oauth2.jwt(Customizer.withDefaults()))
```

### sessionManagement

`sessionManagement` 는 세션 관리 기능을 작동시키는 것으로 JWT에서는 세션을 사용하지 않기 때문에 세션 정책을 사용하지 않도록 설정한다.

```jsx
sessionManagement((session) -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
```

### exceptionHandling

`exceptionHandling` 는 스프링 시큐리티에서 예외 처리를 구성하는데 사용한다. 주요 기능은 다음과 같다.

1. **인증 예외 처리(Authentication Exceptions)**

   `authenticationEntryPoint` 메서드를 통해 인증되지 않은 요청에 대한 응답을 정의할 수 있다.

2. **권한 예외 처리(Authorization Exceptions)**

   `accessDeniedHandler` 메서드를 통해 권한이 없는 요청에 대한 처리를 정의할 수 있다.

3. **예외 핸들러 등록**

   `defaultAuthenticationEntryPointFor`, `defaultAccessDeniedHandlerFor` 등의 메서드를 사용하여 특정 예외 타입에 대한 핸들러를 등록할 수 있다.

```jsx
exceptionHandling((exceptions) -> exceptions
    .authenticationEntryPoint(new BearerTokenAuthenticationEntryPoint())
    .accessDeniedHandler(new BearerTokenAccessDeniedHandler()));
```

### build

모든 설정이 완료된 후 `build()` 를 호출하면 `SecurityFilterChain` 을 반환해줍니다.

```jsx
http.build();
```

## JWT 발급을 위한 빈 등록

### **NimbusJwtEncoder**

`NimbusJwtEncoder` 는스프링 시큐리티에서 제공하는 클래스로 JWT를 생성하는데 사용된다. JWS 서명에 사용되는 공개키/개인키를 포함하는 \*\*\*\*`JWKSource` 를 생성자에 제공하여 `NimbusJwtEncoder` 를 생성할 수 있다.

`JWKSource` 를 생성하기 위해서는 아래와 같이 JWK가 필요하다. `JWK`는 `Json Web Key` 의 약자이고 키 정보를 포함하는 객체이다. 다양한 암호화 알고리즘과 키 속성을 지원한다.

```jsx
@Bean
JwtEncoder jwtEncoder() {
    JWK jwk = new RSAKey.Builder(this.publicKey).privateKey(this.privateKey).build();
    JWKSource<SecurityContext> jwks = new ImmutableJWKSet<>(new JWKSet(jwk));
    return new NimbusJwtEncoder(jwks);
}
```

현재 토큰의 암호화 시에는 `Nimbus-jose-jwt` 라이브러리를 사용하고 있다. 알고리즘 방식은 RSA 방식을 사용하기 때문에 공개키와 개인키가 필요하다.

> `공개키/개인키`

공개키와 개인키는 암호화와 복호화에 사용되는 한쌍의 키이다. 개인키로 암호화 한 정보는 한 쌍이 되는 공개키로만 복호화가능하고, 공개키로 암호화한 정보는 한 쌍이 되는 개인키로만 복호화가 가능하다.

>

### NimbusJwtDecoder

`NimbusJwtDecoder` 는 복호화시에 사용되며 공개키를 사용한다.

```jsx
@Bean
JwtDecoder jwtDecoder() {
    return NimbusJwtDecoder.withPublicKey(this.publicKey).build();
}
```

## 토큰 발급하기

토큰을 발급받기 위한 컨트롤러로 아래와 같이 작성한다. 토큰 발급 HTTP Method는 Post 방식을 사용한다.

```jsx
@RestController
public class TokenController {

		@PostMapping("/token")
		public String token(Authentication authentication) {

		}
}
```

`/token` 주소로 사용자가 `Basic Auth` 방식으로 유저정보를 전달하면 스프링 시큐리티에서는 전달 받은 정보에 해당하는 유저가 존재하는지 확인한다.

일치하는 정보가 존재한다면 `Authentication` 객체를 생성하여 해당 메서드로 전달해준다.

### JwtClaimsSet

jwt를 사용하다보면 claim 이라는 단어를 자주접하게 되는데 이게 무엇인지 먼저 알아보았다.

> `Claim`

**claim**은 JWT의 페이로드(payload) 부분에 포함되며, 다음과 같은 정보를 저장하는데 사용한다.

> 1. **사용자 정보**: 토큰 소유자의 ID, 이름, 이메일 등의 정보를 포함한다.
> 2. **인증 정보**: 토큰 발급 시간, 만료 시간, 발급자 등의 인증 관련 정보를 포함한다.
> 3. **권한 정보**: 토큰 소유자의 권한(Roles, Scopes 등)을 포함한다.
> 4. **기타 정보**: 응용 프로그램에 필요한 기타 정보를 포함할 수 있다.

`JwtClaimsSet` 은 JWT 토큰을 생성할때 포함이 될 정보를 저장할 때 사용되는 클래스이다. 빌더를 제공하기 때문에 아래와 같이 객체를 생성할 수 있다.

```jsx
JwtClaimsSet claims = JwtClaimsSet.builder()
    .issuer("self")
    .issuedAt(now)
    .expiresAt(now.plusSeconds(expiry))
    .subject(authentication.getName())
    .claim("scope", scope)
    .build();
```

- **issuer** : 토큰 발행인
- **issuedAt** : 토큰이 발급된 시간
- **expiresAt** : 토큰의 만료 시간
- **subject** : 이 토큰을 사용하는 사용자
- **claim** : 기본적으로 자주 사용되는 이름은 위와 같이 제공해주고 있다. 추가정보를 저장할 때는 claim을 통해서 저장이 가능하다.

### 토큰 발급

토큰 발급 시 위에서 빈으로 등록한 `JwtEncoder` 를 사용한다. **JwtEncoder**는 암호화 시 `JwtEncoderParameters` 를 입력받는다. `JwtEncoderParameters` 는 **JwsHeader** 와 **JwtClaimsSet** 2개의 데이터를 가진다.

`JwsHeader` 는 JWS 토큰의 처리와 검증에 사용된다. JWT가 어떤 알고리즘으로 서명되었는지, JWT의 타입은 무엇인지에 대한 정보를 가지고 있다.

`JwtEncoderParameters` 는 생성자가 private 이기 때문에 from 메서드를 통해서 객체를 생성해야 한다. 아래와 같이 코드를 작성하면 토큰을 발급받을 수 있다.

```jsx
encoder.encode(JwtEncoderParameters.from(claims)).getTokenValue();
```

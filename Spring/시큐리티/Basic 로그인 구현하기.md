## 개요

지금까지는 로그인 기능을 구현할 때 사용자 정보를 `body` 에 담아서 처리하였다. 이러한 경우 컨트롤러에서 사용자 정보를 받아 DB의 데이터를 조회해서 일치여부를 확인하는 작업만 하면 기능이 완료되었다.

그러다 로그인 기능 구현 시 body로 전송하지 않고 header의 `Authorization` 키에 담아 인코딩하여 전송하는 경우도 있다고 듣게 되었다.

그래서 이번에는 header를 통해서 전송하는 방법에 대한 시행착오를 작성해보려고 한다.

### 시행착오

처음 의도는 헤더에 `아이디:비밀번호` 값을 인코딩 후 전송 후 컨트롤러에서 받아서 처리하려고 생각하였다.

```java
header: {
	Authorization: `Basic ${btoa(`${form.email}:${form.password}`)}`
}
```

하지만 아무리 요청을 보내도 컨트롤러에서 응답을 받을 수 없었다. 이유는 아래와 같다.

### Http Basic

테스트 당시 프로젝트에서는 스프링 시큐리티를 도입하였다.
`HttpSecurity` 의 옵션 중 httpBasic 의 기본 옵션을 사용하도록 설정하였다.

```java
http
		.httpBasic(Customizer.withDefaults())
```

- `BasicAuthenticationFilter` 해당 필터가 추가된다.
- 해당 필터는 헤더에서 기본 인증을 요구하는 경우 처리된다.

그렇다. 로그인 시 기본인증 방식으로 인증을 요청보냈기 때문에 `BasicAuthenticationFilter` 에서 이미 401 에러가 발생하여 컨트롤러까지 전달되지 못하였다.

### 전환

그래서 기존에 사용하려면 컨트롤러 방식을 포기하고 필터를 통해 기본인증을 처리하도록 변경하였다.

> ⚡ 주의
> 앞으로 알아볼 내용들은 나의 시행착오를 통해 정상적으로 처리가 완료될때까지의 코드이다. 이 방법이 일반적으로 사용되는 방법이라고는 할 수 없을 듯 하다.

## 필터를 활용한 로그인

### 플로우

1. 사용자가 로그인 요청
2. 기본인증을 처리할 필터에서 담당**(URL 정보 일치 시 필터 호출)**
3. 필터는 `LoginAuthenticationToken`(인증객체) 을 생성하여 `AuthenticationManager`에게 전달
4. `AuthenticationManager` 는 등록 된 `Provider` 중 `LoginAuthenticationToken` 를 지원하는 Provider에게 인증 요청
5. `AuthenticationProvider` 는 인증 객체를 `LoginService` 에게 전달
6. `LoginService` 는 DB에 접속하여 ID 정보에 해당하는 유저 정보를 로드하여 인증 여부 확인
7. `LoginService` 는 유저 정보를 `AuthenticationProvider` 에게 전달 후 `AuthenticationProvider` 는 비밀번호 일치 여부 확인 후 최종 인증 객체 생성 후 전달
8. 인증에 성공하면 `LoginSuccessHandler` 호출
9. 인증에 실패하면 `LoginFailureHandler` 호출

### LoginAuthenticationFilter

먼저 `AbstractAuthenticationProcessingFilter` 를 상속 받는 `LoginAuthenticationFilter` 필터를 생성한다.

**AbstractAuthenticationProcessingFilter**

- `UsernamePasswordAuthenticationFilter` 필터 또한 상속하고 있는 추상 클래스이다.
- `RequestMatcher` 와 일치하는 요청을 받으면 필터 호출
- 요청과 일치하는 경우 `attemptAuthentication` 메서드 호출

- LoginAuthenticationFilter 코드
  ```java
  package com.test.filter;

  import com.test.auth.LoginAuthenticationToken;
  import jakarta.servlet.http.HttpServletRequest;
  import jakarta.servlet.http.HttpServletResponse;
  import lombok.extern.slf4j.Slf4j;
  import org.springframework.http.HttpMethod;
  import org.springframework.security.authentication.AuthenticationServiceException;
  import org.springframework.security.core.Authentication;
  import org.springframework.security.core.AuthenticationException;
  import org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter;
  import org.springframework.security.web.authentication.AuthenticationConverter;
  import org.springframework.security.web.authentication.www.BasicAuthenticationConverter;
  import org.springframework.security.web.util.matcher.AntPathRequestMatcher;

  @Slf4j
  public class LoginAuthenticationFilter extends AbstractAuthenticationProcessingFilter {

      private static final AntPathRequestMatcher SSO_LOGIN_ANT_PATH_REQUEST_MATCHER
              = new AntPathRequestMatcher("/api/auth/login", HttpMethod.POST.name());

      private AuthenticationConverter authenticationConverter = new BasicAuthenticationConverter();

      public LoginAuthenticationFilter() {
          super(SSO_LOGIN_ANT_PATH_REQUEST_MATCHER);
      }

      @Override
      public Authentication attemptAuthentication(
              HttpServletRequest request,
              HttpServletResponse response
      ) throws AuthenticationException {
          if (!request.getMethod().equals(HttpMethod.POST.name())) {
              throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
          }

          Authentication converted = authenticationConverter.convert(request);

          if (converted == null) {
              throw new AuthenticationServiceException("Authentication request cannot be null");
          }

          LoginAuthenticationToken authentication = new LoginAuthenticationToken(converted.getPrincipal(), converted.getCredentials());
          return this.getAuthenticationManager().authenticate(authentication);
      }
  }
  ```

1. 필터 조건을 `new AntPathRequestMatcher("/api/auth/login", HttpMethod.*POST*.name());` 와 같이 추가.
2. `attemptAuthentication` 를 오버라이딩 하여 인증 객체를 만들고 `AuthenticationManager` 에게 인증을 위임
   - 헤더에 **Basic Authorization** 에서 유저정보를 추출하는 건 `BasicAuthenticationConverter` 에 구현되어 있어 이를 활용
3. 인증객체를 return 하면 로그인 완료!

### AuthenticationManager

`AuthenticationManager` 는 필터에서 인증을 위임받아 실제로 인증객체를 통해 인증을 처리, 관리하는 부분이다.

실제 인증 로직이 들어가는 부분은 아니고, 인증 로직을 구현한 `Provider` 들을 관리하는 역할을 한다.

`AuthenticationManager` 인터페이스이기 때문에 이를 구현한 `ProviderManager` 를 생성하였다.

- SecurityConfig의 코드 일부
  ```java
  private final LoginAuthenticationProvider authenticationProvider;

  @Bean
  public AuthenticationManager authenticationManager() {
      return new ProviderManager(authenticationProvider);
  }

  public Filter loginAuthenticationFilter() {
      LoginAuthenticationFilter authenticationFilter = new LoginAuthenticationFilter();
      authenticationFilter.setAuthenticationManager(authenticationManager());
      return authenticationFilter;
  }
  ```

1. `AuthenticationManager` 는 이미 스프링에서 제공하는 ProviderManager를 생성하였다. 이때 인증 방법이 기본 인증 방식밖에 없기 때문에 이를 구현한 `LoginAuthenticationProvider` 만 등록하였다.
2. `AuthenticationManager` 를 필터 생성 시 전달하였으며, 필터에서는 해당`AuthenticationManager` 를 통해 인증을 진행하게 된다.

### LoginAuthenticationProvider

이 클래스는 실제로 인증 로직을 구현하는 부분이다. 해당 클래스에서 모두 구현하지 않고 아이디값을 통해 DB에서 유저 정보가 존재하는지 유무는 `UserDetailsService` 를 구현한 `UserService` 에서 담당하고 있다.

`UserService` 에서 구한 유저 정보를 통해 비밀번호 일치 여부를 확인하고 실제 인증에서 사용 될`UsernamePasswordAuthenticationToken` 인증객체를 생성하여 전달한다.

- LoginAuthenticationProvider 코드
  ```java
  package com.test.auth;

  import com.test.auth.service.UserService;
  import lombok.RequiredArgsConstructor;
  import lombok.extern.slf4j.Slf4j;
  import org.springframework.security.authentication.AuthenticationProvider;
  import org.springframework.security.authentication.BadCredentialsException;
  import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
  import org.springframework.security.core.Authentication;
  import org.springframework.security.core.AuthenticationException;
  import org.springframework.security.core.userdetails.UserDetails;
  import org.springframework.security.crypto.password.PasswordEncoder;
  import org.springframework.stereotype.Component;
  import org.springframework.util.Assert;

  @Slf4j
  @Component
  @RequiredArgsConstructor
  public class LoginAuthenticationProvider implements AuthenticationProvider {

      private final UserService userService;
      private final PasswordEncoder encoder;

      @Override
      public Authentication authenticate(Authentication authentication) throws AuthenticationException {
          Assert.notNull(authentication.getCredentials(), "Credentials cannot be null");

          String email = authentication.getName();
          String password = (String) authentication.getCredentials();

          UserDetails userDetails = userService.loadUserByUsername(email);

          if (!encoder.matches(password, userDetails.getPassword())) {
              throw new BadCredentialsException("Invalid password");
          }

          UsernamePasswordAuthenticationToken authenticationToken = UsernamePasswordAuthenticationToken
                  .authenticated(userDetails, password, userDetails.getAuthorities());

          return authenticationToken;
      }

      @Override
      public boolean supports(Class<?> authentication) {
          return LoginAuthenticationToken.class.isAssignableFrom(authentication);
      }
  }
  ```
- `supports` 메서드는 전달받은 인증객체를 해당 클래스에서 처리가능한지 여부를 반환한다.
  - `LoginAuthenticationToken` 인증객체를 전달받은 경우 처리한다.
- `authenticate` 메서드는 `supports` 에서 true가 반환되면 호출된다.

### LoginService

- LoginService 코드
  ```java
  package com.test.auth.service;

  import com.test.auth.repository.AuthRepository;
  import lombok.RequiredArgsConstructor;
  import lombok.extern.slf4j.Slf4j;
  import org.springframework.security.core.userdetails.UserDetails;
  import org.springframework.security.core.userdetails.UserDetailsService;
  import org.springframework.security.core.userdetails.UsernameNotFoundException;
  import org.springframework.stereotype.Service;
  import org.springframework.transaction.annotation.Transactional;

  @Slf4j
  @Service
  @RequiredArgsConstructor
  @Transactional(readOnly = true)
  public class UserService implements UserDetailsService {

      private final AuthRepository authRepository;

      @Override
      public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
          return authRepository.findById(email)
                  .orElseThrow(() -> new UsernameNotFoundException("User not found"));
      }
  }
  ```
- 코드는 DB에서 id에 해당하는 값을 전달하여 유저 정보를 가져오는 역할을 한다.

### LoginSuccessHandler

`LoginSuccessHandler` 는 로그인에 성공 후 호출되는 핸들러이다. 로그인에 성공한 경우 유저 정보와 JWT 토큰을 생성하여 클라이언트에게 전달해주는 역할을 한다.

### LoginFailureHandler

`LoginFailureHandler` 는 로그인에 실패 후 호출되는 핸들러이다. 클라이언트에게 인증 실패 메시지를 전달한다.

### Configuration

```java
http
		.httpBasic(Customizer.withDefaults())
```

위의 설정을 사용하는 경우 `BasicAuthenticationFilter` 가 호출되기에 해당 필터를 새롭게 만든 `LoginAuthenticationFilter` 를 앞에 배치하여 인증 처리를 먼저 하였다.

```java
http
		.addFilterBefore(loginAuthenticationFilter(), BasicAuthenticationFilter.class)

```

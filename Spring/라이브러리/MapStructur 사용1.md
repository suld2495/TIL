# **MapStruct**

MapStruct는 Java bean 유형 간의 매핑 구현을 단순화하는 코드 생성기입니다.

- 컴파일 시점에 코드를 생성하여 런타임에서 안정성을 보장합니다.
- 다른 매핑 라이브러리보다 속도가 빠릅니다.
- Annotation processor를 이용하여 객체 간 매핑을 자동으로 제공합니다.

<aside>
⚠️ Lombok 라이브러리에 먼저 dependency (의존성) 추가가 되어있어야 합니다.

</aside>

### 설치

아래의 3개를 설치해주면 된다. 주의할 점은 Lombok 밑에 추가해주어야 한다는 것이다.

```jsx
dependencies {
    ...
    implementation 'org.mapstruct:mapstruct:1.5.3.Final'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.3.Final'
    annotationProcessor 'org.projectlombok:lombok-mapstruct-binding:0.2.0' // builder 사용시 추가
    ...
}
```

`JdbcTemplate` 으로부터 받아온 User 객체를 UserResponse 객체로 변환할 예정이다.

```jsx
package com.example.mapstructur.domain;

import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@NoArgsConstructor
public class User {
    private String id;
    private String name;
    private String password;
    private String createdAt;
    private String updatedAt;
}
```

```jsx
package com.example.mapstructur.dto;

import lombok.Builder;
import lombok.Getter;

@Getter
@Builder
public class UserResponse {
    private String id;
    private String name;
    private String createdAt;
    private String updatedAt;
}
```

### Mapper 인터페이스

먼저 Mapper 인터페이스를 생성한다. `org.mapstruct.Mapper` 어노테이션을 추가만 해주면 우리가 해줄 작업은 거의 끝이 난다.

메서드의 파라미터로 변환에 사용될 타입을, 메서드의 반환타입으로는 변환될 타입을 지정하면 된다.

```jsx
package com.example.mapstructur.mapper;

import com.example.mapstructur.domain.User;
import com.example.mapstructur.dto.UserResponse;
import org.mapstruct.Mapper;

@Mapper(componentModel = "spring")
public interface UserMapper {
    UserResponse userToUserResponse(User user);
}

```

<aside>
💡 `componentModel="spring"` 을 통해서 spring container에 **Bean**으로 등록 해 준다.

</aside>

런타임시 아래와 같이 구현체가 자동으로 생성된다.

```jsx
package com.example.mapstructur.mapper;

import com.example.mapstructur.domain.User;
import com.example.mapstructur.dto.UserResponse;
import javax.annotation.processing.Generated;
import org.springframework.stereotype.Component;

@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2024-03-25T20:29:07+0900",
    comments = "version: 1.5.2.Final, compiler: IncrementalProcessingEnvironment from gradle-language-java-8.6.jar, environment: Java 21.0.1 (Oracle Corporation)"
)
@Component
public class UserMapperImpl implements UserMapper {

    @Override
    public UserResponse userToUserResponse(User user) {
        if ( user == null ) {
            return null;
        }

        UserResponse.UserResponseBuilder userResponse = UserResponse.builder();

        userResponse.id( user.getId() );
        userResponse.name( user.getName() );
        userResponse.createdAt( user.getCreatedAt() );
        userResponse.updatedAt( user.getUpdatedAt() );

        return userResponse.build();
    }
}
```

객체 간 속성명이 동일한 경우 위 처럼 간단하게 지정해주면 된다. 하지만 속성명이 다르거나 복잡한 구조인 경우에는 좀 더 추가적인 설정이 필요하다.

이는 다음에 한번 사용해본 후 정리해볼 예정이다.

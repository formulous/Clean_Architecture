# 주니어 개발자의 클린 아키텍처 맛보기를 맛보기

`Oct.10.2022 formulous (Backend)`

아래는 Robert C. Martin이 블로그에 기재한 Clean Architecture 라는 글에서 발췌한 우아한형제들 기술 블로그의 내용을 발췌한 내용이다.

(원본 출처 : https://techblog.woowahan.com/2647/)

## 의존성 규칙

![image](https://user-images.githubusercontent.com/88424067/183782795-fec7d029-83cb-4db5-bc3e-56220ece2d02.png)

출처 : http://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html

대부분의 아키텍처는 세부적인 차이가 있더라도 공통적인 목표는 계층을 분리해 관심사를 분리하는 것이다.
의존성 규칙에서는 모든 소스코드의 의존성은 반드시 저수준 정책에서 고수준 정책으로 향해야 한다고 말한다.

즉, 업무의 로직(고수준)을 담당하는 코드가 DB 나 Web 같은 세부 사항(저수준)에 의존하지 않아야 함을 의미한다.

이를 통해 업무 로직이 세부 사항들에 영향을 받지 않게 할 수 있다.

## What Data Crosses the Boundaries

DB 형식의 데이터 구조나 Framework 에 종속적인 데이터를 사용하게 되면, 저수준의 데이터 형식을 고수준에서 알아야 하기에 의존성 규칙을 위반하게 된다.

의존성 규칙을 지키기 위해 경계 간의 데이터를 전달할 때 간단한 구조의 `Data Trasfer Object(DTO)`를 이용하는 것을 추천한다.
하지만 DTO 에서도 고민해 볼 만한 문제점은 존재한다.

### DTO 간의 중복 코드

DTO 는 클라이언트로부터 전달되는 데이터를 받아서 처리한다. 이 때, 일반적으로 요청은 크게 등록, 수정, 조회, 삭제의 형태로 오는데, 생성과 수정 요청은 거의 비슷한 형식의 데이터나 검증 로직을 필요로 하기에 중복을 처리할 수 있는 방법을 고민해 볼 가치가 있다.

아래는 중복의 예시를 보여준다.

```javascript
public class CreateRequest {
  private String name;
  private LocalDate startDate;
  private LocalDate endDate;
  ...
}

public class UpdateRequest {
  private String name;
  private LocalDate startDate;
  private LocalDate endDate;
  ...
}
```

`Clean Architecture` 글에서 이에 대한 좋은 내용을 언급하는데, 그 내용은 중복에도 종류가 있다는 것이다.

1. 진짜 중복
  한 인스턴스가 변경되면, 동일한 변경을 그 인스턴스의 모든 복사본에 반드시 적용해야함.
  
2. 우발적 중복
  중복으로 보이는 두 코드의 영역이 각자의 경로로 발전한다면, 즉 서로 다른 속도와 다른 이유로 변경된다면 이 두 코드는 진짜 중복이 아니다.

여기서 저장과 수정 사이의 DTO 중복은 우발적 중복에 속한다고 생각해 볼 수 있다.
그 예시로 스케줄을 저장해야 하는 기능을 개발한다고 생각해보자. 스케줄을 처음 저장할 때는 시작 일자가 과거 시간이 될 필요가 없기 때문에 이에 대한 유효성 검사를 진행한다.

```javascript
public class CreateRequest {
  public void assertValidation() {
    if(startDate.isBefore(LocalDate.now())) {
      throw new IllegalArrgumentException();
    }
  }
}
```

하지만 수정을 하는 경우에는 이미 해당 데이터가 과거 날짜일 수 있기 때문에 StartDate 에 대한 유효성 검사가 필요 없게된다.

```javascript
public class CreateRequest {
  public void assertValidation() {
    if(startDate.isBefore(LocalDate.now())) { ---> 불필요한 체크
      throw new IllegalArrgumentException();
    }
  }
}
```

이렇게 서로 다른 이유로 변경이 될 수 있는 상황에서는 DTO를 분리하는게 적절하다.

## Entity가 DTO를 직접 참조하는 문제

Entity를 생성할 때 Controller로 부터 생성된 Request(DTO) 객체를 이용하는 경우가 있다.

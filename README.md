# 주니어 개발자의 클린 아키텍처 맛보기를 맛보기

`Oct.10.2022 formulous (Backend)`

이 글은 Robert C. Martin이 블로그에 기재한 Clean Architecture 라는 글에서 발췌한 우아한형제들 기술 블로그의 내용을 발췌한 내용이다.

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

```javascript
public class Entity {
  ...
  public static Entity of(Request request) {
    ...
  }
}
```

위의 코드와 같이 Entity가 직접 DTO를 참조하게 된다면 DTO의 변경에 의해 Entity도 변경될 수 있다.
이럴 때는 Entity를 생성하는 UseCase 계층에서 DTO의 데이터를 꺼내서 Entity를 생성하는 것이 DTO의 변경에 의해 Entity가 변경할 가능성을 낮출 수 있다.

## Crossing Boundaries

제어의 흐름은 내부에서 외부로 향할 수 있지만 소스코드의 의존성은 이렇게 될 경우 의존성 규칙을 위반하게 되므로, 의존성 역전의 원칙을 이용하여 해결해야 한다.
-> 고수준 정책은 저수준 정책에 의존해서는 안된다.

간단한 예시로 일반적으로 조회 요청이 일어나면 이를 처리하기 위한 제어의 흐름과 소스코드의 의존성은 아래와 같다.

![image](https://user-images.githubusercontent.com/88424067/183795630-752509b3-1051-40b5-8f62-e08ae5e881b4.png)

위와 같이 소스코드 의존성을 가지게 된다면 Service가 구체적인 RDMBS를 구현한 구현체를 직접 참조하게 되는데 이는 의존성 규칙을 위반하게 된다.
이러한 문제의 가장 간단한 해결책은 DIP(의존성 역전의 원칙)을 적용하는 것이다.

![image](https://user-images.githubusercontent.com/88424067/183795754-96d33289-ca16-4a6c-8ead-e65f4a70032b.png)

추상화된 Repository 인터페이스를 두어서 Service가 이를 참조하고, 구체적인 RDRepository가 이러한 인터페이스를 구현하게 된다면 소스코드의 의존성을 역전시킬 수 있다.
이렇게 DIP를 통해 Service는 DB의 구체적인 세부사항을 몰라도 되고, DB의 변경에도 영향을 받지 않게 된다.

## 의존성 예시 : 배달과 배달료의 관계

배달료를 구할 때 크게 두가지의 정책이 있다. 첫 번째는 배달을 수행하는 라이더의 거리 기반 요금제에 따른 요금, 두 번째는 구역 할증에 따른 추가요금이다.
따라서 배달료을 구하기 위해서는 아래와 같은 제어의 흐름이 발생한다.

![image](https://user-images.githubusercontent.com/88424067/183799140-1dfd98e4-1770-4470-922b-321fe8054d98.png)

이러한 제어 흐름은 배달에 관한 정책이 거리 배달료 정책과, 구역 배달료 정책에 강하게 의존하게 되는 문제가 발생한다.
즉, 각각의 배달료 정책이 변경될 때마다 배달에 관한 부분도 변경될 여지가 생기는 것이다.

이런 경우에 대해서 DIP를 사용해 의존성 문제를 해결할 수 있게 된다. 배달 도메인에서는 거리 배달료나 구역 배달료 정책이 변경돼도 영향을 받지 않게 되는것이다.

![image](https://user-images.githubusercontent.com/88424067/183800187-eabefe8f-3671-4705-8fd1-724340eeecc8.png)

여기서 배달료의 정보를 얻기 위해서는 DistanceFee와 RegionFee를 의존성 주입을 받아 사용해야 하는데, 사실 클라이언트 입장에서는 이런 구체적인 정보를 모르고 사용하는 것이 좋다.
이럴 때 Facade를 제공하여 해결할 수 있다.

![image](https://user-images.githubusercontent.com/88424067/183817601-27174104-5390-4721-af6b-cbb497ae6e23.png)'

Facade를 제공하면 클라이언트는 배달료 정보를 얻기 위해 Facade 인터페이스에만 의존하면 된다.

## 인터페이스 선언 시 주의할 점

인터페이스를 분리하여 구체적인 세부사항을 분리 시킬 수 있었지만, 인터페이스를 선언할 때도 주의할 점이 몇가지가 존재한다.

### 1. 인터페이스 분리

인터페이스를 작성할 때는 클라이언트가 필요로 하는 메서드를 기반으로 분리되어야 한다.
인터페이스를 구현체 클래스에 의존하여 작성하면 클라이언트가 필요하지 않은 메서드까지 노출되기 때문이다.

예를 들어 특정 도메인 객체가 배달료, 배달팁, 배달시간의 할증에 괸한 정보를 가지고 있다고 하면 클라이언트가 사용할 인터페이스를 다음과 같이 구현체에 의존하여 작성했었다.

```javascript
public interface ExtraService {
  long calculateExtraDeliveryFee(...);
  long calculateExtraDeliveryTip(...);
  long calculateExtraDeliveryTime(...);
}
```

위의 코드에서 각각의 메서드를 소비하는 클라이언트들은 모두 달랐지만 같은 인터페이스를 바라보았기 때문에 각각의 변경에 따라 영향을 받을 수 있는 클라이언트가 많아지게 되었다.
이러한 변경의 영향을 줄이기 위해 아래 코드와 같이 각각의 클라이언트를 기반으로 인터페이스를 분리하게 됐다.

```javascript
public interface ExtraFeeService {
  long calculateExtraDeliveryFee(...);
}
public interface ExtraTipService {
  long calculateExtraDeliveryTip(...);
}
public interface ExtraTimeService {
  long calculateExtraDeliveryTime(...);
}
```

### 2. 인터페이스 명

인터페이스를 생성할 때 메서드 명에 구현체 클래스를 나타내곤 하는데 이는 직접적인 의존성은 없다고 하지만 결국 클라이언트가 세부적 사항을 알게하기 때문에 인터페이스의 이름, 메서드 이름을 정할 때도 구체적인 클래스의 이름을 드러내지 않는 것이 중요하다 생각한다.

### 3. 인터페이스 메서드 파라미터

인터페이스의 파라미터를 넘기는데에는 Entity, DTO, 필요한 데이터를 각각 넘기는 3가지 방법이 고려될 수 있다.

1. Entity, DTO를 전달하는 방법

  일반적으로 인터페이스를 사용하면 Entity 자체를 넘기거나 DTO를 파라미터로 사용한다.
  이러한 이유는 업무 로직의 세부사항은 언제든지 변경될 수 있기 때문이다. Entity나 DTO를 사용하면 이러한 변경이 클라이언트에 영향을 미치는 것을 최소화할 수 있다.
  
  ```javascript
  //DTO를 사용할 때 팩토리 메서드를 사용하여 Entity를 받게 구현한다면, 필요한 데이터가 추가되더라도 클라이언트의 변경을 줄일 수 있다.
  public class DTO {
    public static ofEntity(Entity entity) {
      ...
    }
  }
  ```
  
  보통 Entity보다 DTO를 사용하는 경우는 인터페이스가 특정 도메인에 종속되는 것이 아니라 범용적일 때 사용하고 있다.

2. 필요한 데이터를 각각 넘기는 방법

  모든 인터페이스에 Entity나 DTO를 넘기는 것은 과할 수 있다. 본인이 생각하기에 불변에 가까운 유틸성 인터페이스는 필요한 인자를 파라미터로 받으면 오히려 명확할 수 있다.
  
  ```javascript
  public interface DateUtil {
    long between(Date from, Date to);
  }
  ```

# 주니어 개발자의 클린 아키텍처 맛보기를 맛보기

`Oct.10.2022 formulous (Backend)`

아래는 Robert C. Martin이 블로그에 기재한 Clean Architecture 라는 글에서 발췌한 내용임.

## 의존성 규칙

![image](https://user-images.githubusercontent.com/88424067/183782795-fec7d029-83cb-4db5-bc3e-56220ece2d02.png)

출처 : http://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html

대부분의 아키텍처는 세부적인 차이가 있더라도 공통적인 목표는 계층을 분리해 관심사를 분리하는 것임.

의존성 규칙에서는 모든 소스코드의 의존성은 반드시 저수준 정책에서 고수준 정책으로 향해야 한다고 말함.

즉, 업무의 로직(고수준)을 담당하는 코드가 DB 나 Web 같은 세부 사항(저수준)에 의존하지 않아야 함을 의미함.

이를 통해 업무 로직이 세부 사항들에 영향을 받지 않게 할 수 있음.


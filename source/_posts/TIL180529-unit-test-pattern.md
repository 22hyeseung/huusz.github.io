---
title: Unit test style
date: 2018-05-29 23:18:53
category: TIL
tags:
  - Unit test
  - Test style
  - AAA(3A)
  - Arrange Act Assert
  - Given When Then
---

# Test style

1. Setup, Exercise, Verify and Teardown ([Four-phases test pattern](http://xunitpatterns.com/Four%20Phase%20Test.html))
2. Given, When, Then ([BDD](https://dannorth.net/introducing-bdd/))
3. Arrange, Act, Assert([AAA](https://xp123.com/articles/3a-arrange-act-assert/))

모두 기본적인 아이디어는 같다.

4. [Setup/ Given/ Arrange] 테스트 할 대상에게, 테스트를 위해 사전에 필요한 조건들을 사전에 갖추게 하고 (기본 값, 파라미터, 선행 되어야 할 함수 실행 등)
5. [Exercise/ When/ Act]테스트 대상 함수를 호출하고
6. [Verify/ Then/ Assert]테스트 대상이 예상한 대로 작동하는지 확인한다.

<sub>* Four-phases test pattern의 Teardown은 테스트에 의해 만들어진 fixture를 해제하는 단계로 필수는 아니다.</sub>

---

Reference

- [Meszaros - Four-phases test pattern](http://xunitpatterns.com/Four%20Phase%20Test.html)
- [Bill Wake - 3A(AAA)](https://xp123.com/articles/3a-arrange-act-assert/)
- [MS docs: AAA example](https://msdn.microsoft.com/ko-kr/library/hh694602.aspx)
- [BDD - Given When Then](https://dannorth.net/introducing-bdd/)
- [martinfowler.com: Given-When-Then](https://martinfowler.com/bliki/GivenWhenThen.html)
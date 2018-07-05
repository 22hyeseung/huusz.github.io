---
title: 테스트 하지 않던 코드 테스트 하기
date: 2018-07-02 23:04:53
category: Test
tags:
  - Unit test
  - React
  - TDD
---

# 테스트

내가 테스트 코드를 작성하기 시작한 것은 불과 2달 전이다. 그 전까지는 테스트 코드를 작성하지 않았다. 해본 적이 없었고, 어떻게 시작하는 지 몰라서였다. 어디서 부터 시작 해야 할 지 모르겠는 그 막막함 때문에 항상 시도만 하다가 테스트 파일을 지워버리곤 했었다. 구글에서 검색해서 나오는 블로그 글이나 아티클들은 아주 간단한 함수들을 예제로 하고 있어서 실무 코드에 활용하기가 쉽지 않았다. 이 글에서는 내가 테스트 코드 작성을 어떻게 시작했는 지와 기존에 테스트 코드 없이 이미 작성한 코드로 어떻게 (미약하게 나마) TDD를 했는 지를 이야기 하려고 한다. 참고로 예제 코드는 React로 작성되었고, 테스트 프레임 워크로는 Jest를 사용하였다.

<br>

## 1. 함수로 추출하자.

프로젝트의 전반적인 컴포넌트 구조는 

- [부모] Redux와 API 요청 및 React 라이프 사이클 함수를 호출하는 Container 컴포넌트

- [자식] 실제 View를 반환하는 순수 함수로 작성된 Presentational 컴포넌트

이렇게 두 컴포넌트가 중심이 된다.

사이드 이펙트가 발생할 수 있는 모든 요소는 Container 컴포넌트에 있고, Presentational 컴포넌트는 최대한 순수하게 유지하고 있다. 그래서, Container 컴포넌트가 정말 길고 복잡하고, 가독성도 매우 떨어졌다. 내가 테스트를 위해 가장 먼저 한 일은 Container 컴포넌트에 숨어 있는 비즈니스 로직을 순수 함수로 추출하는 일이다. 비즈니스 로직은 DOM 조작이 필요하지 않기 때문에 순수 함수로 떼어낸다면 쉽게 테스트 할 수 있다. 어떤 식으로 함수를 추출할 수 있는지는, 먼저 아래 '테스트 없이 작성된 코드' 예시를 보자. (이 코드는 예시만을 위한 것으로, 많은 부분이 생략되어 동작하지 않을 것이다.)

### 1) 발견

```jsx
// RegisterFormContainer.js
class RegisterFormContainer extends Component {
  handleOpen = () => {
    UserActions.openModal();
  }

  handleClose = () => {
    UserActions.closeModal();
  }

  handleSubmit = async () => {
    // ************* 함수로 추출할 부분은 바로 여기 이다!! *****************
    const { validateValues } = this.props.form;

    if(!validateValues) {
      throw Error();
    }

    const command = {
      username: validateValues.username || validateValues.email,
      email: validateValues.email,
      tel: validateValues.tel || null,
      date: new Date(),
    };
    // ************************************************************

    try {
      await postRegisterRequest(command);
      this.handleClose();
    } catch(err) {
      Modal.info({
        message: 'Register failed.'
        icon: true,
      })
    }

  }

  render() {
    return(
      <Modal 
        visible={this.props.visible}
        open={this.handleOpen}
        close={this.handleClose}
      >
        <RegisterForm submit={this.handleSubmit}/>
      </Modal>
    )
  }
} 

export default connect(({user}) => ({
  form: {
    validateValues: user.form.validateValues,
    visible: user.form.isModalVisible,
  }
}))(RegisterFormContainer)
```

## 2) 추출

processRegisterCommand 라는 이름의 함수를 외부 파일로 생성하고, 일단 발견한 부분을 무작정 추출해온다.
**당연히 코드는 정상적으로 돌아가지 않을 것이다.** 아무것도 수정하지 않는다. 다만, 어떤 것을 리턴할 것인지만 정한다.

```js
// processRegisterCommand.js
export default processRegisterCommand() {
  const { validateValues } = this.props.form;

  if(!validateValues) {
    throw Error();
  }

  const command = {
    username: validateValues.username || validateValues.email,
    email: validateValues.email,
    tel: validateValues.tel || null,
    date: new Date(),
  };

  return command;
}
```

아까의 RegisterFormContainer 컴포넌트에서 새로 만든 함수를 불러와서 적용한다.

```jsx
// RegisterFormContainer.js
class RegisterFormContainer extends Component {
  // (생략)

  handleSubmit = async () => {
    // ************* 함수로 추출한 부분이다!! *****************
    const command = processRegisterCommand();
    // ***************************************************

    try {
      await postRegisterRequest(command);
      this.handleClose();
    } catch(err) {
      Modal.info({
        message: 'Register failed.'
        icon: true,
      })
    }
  }
  // (생략)
}
```

## 2. 함수가 하는 일을 테스트 수트로 작성한다.

이제 테스트 파일을 만든다. 그리고 아까 추출한 processRegisterCommand 함수를 불러온다. 테스트 수트는 describe 구문으로 작성할 수 있다. describe 구문에는 함수가 하는 일을, test 구문에는 그 일을 마치기 위해 필요한 단계들 혹은 확인해야 할 것들을 잘게 쪼갠다.

유의할 점은, 처음부터 모든 시나리오를 빠짐없이 적으려 해서는 안 된다는 것이다. 오래 생각하지 않아도 바로 바로 눈에 띄는 것들 위주로 일단 작성하는 게 중요하다. 테스트를 처음 해보는 입장에서는 본격적으로 테스트 코드를 작성 해보기도 전에 힘이 빠져 버린다. 부족한 부분은 나중에 더 추가하면 그만이다.

```js
// processRegisterCommand.js
export default processRegisterCommand() {
  const { validateValues } = this.props.form;

  if(!validateValues) {
    throw Error();
  };

  const command = {
    username: validateValues.username || validateValues.email,
    email: validateValues.email,
    tel: validateValues.tel || null,
    date: new Date(),
  };

  return command;
};
```

```js
// processRegisterCommand.test.js
import processRegisterCommand from '../processRegisterCommand';

describe('유효성 검사를 통과한 값들을 command 데이터 포맷으로 정제하여 반환한다.', () => {
  test('validateValues 파라미터가 존재하면 예외를 발생시키지 않는다.');
  test('validateValues 파라미터가 존재하지 않으면, 예외를 발생시킨다.');
  test('validateValues 파라미터가 존재하면 command 객체를 반환한다.');
  test('validateValues의 username 속성이 존재하지 않으면, email 속성으로 대체한다.');
  test('validateValues의 tel 속성이 존재하지 않으면, null로 대체한다.');
});
```

사실 이렇게까지 여러 테스트 케이스를 미리 작성하지 않아도 된다. 하나씩 차례 차례 해 나가도 전혀 상관 없다. 다만, 이렇게 하고 나면 내가 추출한 함수가 어떤 일을 하고 그 일을 위해 어떤 것을 체크해야 하는 지가 한 눈에 보인다. 여기서 어떤 것을 체크해야 하는 지? 가 바로 테스트 해야 할 부분이다.

## 3. 테스트 코드를 작성해보자!

### 1) Red - 실패하는 테스트 코드 작성

일단 처음에는 누가봐도 무조건 실패할 것 같은 구문부터 시작한다. 이 테스트 코드는 실패할 수 밖에 없다. 애초에 함수 자체가 현재 오류 상태이기 때문이다. 

```js
// processRegisterCommand.test.js
import processRegisterCommand from '../processRegisterCommand';

describe('유효성 검사를 통과한 값들을 command 데이터 포맷으로 정제하여 반환한다.', () => {

  // ***************************** 첫번째 테스트 코드 *************************
  test('validateValues 파라미터가 존재하면 예외를 발생시키지 않는다.', () => {
    const param = {
      username: 'user'
      email: 'user@email.com',
      tel: '01012345678',
      agreement: [true, true, true],
    };
    expect(() => processRegisterCommand(param)).not.toThrowError();
  });
   // ********************************************************************

  test('validateValues 파라미터가 존재하지 않으면, 예외를 발생시킨다.');
  test('validateValues 파라미터가 존재하면 command 객체를 반환한다.');
  test('validateValues의 username 속성이 존재하지 않으면, email 속성으로 대체한다.');
  test('validateValues의 tel 속성이 존재하지 않으면, null로 대체한다.');
})
```

### 2) Green - 프로덕션 코드를 수정해서 테스트 성공시키기

이제 테스트가 성공하도록 processRegisterCommand 함수를 수정해주면 된다. 오류가 생기는 부분은 분명 'this.props' 일 것이다. 이 구문 자체가 리액트 의존적인 코드이기 때문이다. 지금 함수는 리액트와 아무 상관 없는 일반 자바스크립트 함수이다. 그러니 this.props.form.validateValues 가 아니라, validateValues 값 자체를 파라미터로 받는다. 그러면 오류를 반환하지 않게 되므로, 테스트에 통과할 것이다.

```js
// processRegisterCommand.js
export default processRegisterCommand(validateValues) {
  // const { validateValues } = this.props.form; <- 이 부분을 지우고, 파라미터로 받는다.
  if(!validateValues) {
    throw Error();
  };

  const command = {
    username: validateValues.username || validateValues.email,
    email: validateValues.email,
    tel: validateValues.tel || null,
    date: new Date(),
  };

  return command;
};
```

두번째 테스트 코드를 작성해보자.

```js
// processRegisterCommand.test.js
import processRegisterCommand from '../processRegisterCommand';

describe('유효성 검사를 통과한 값들을 command 데이터 포맷으로 정제하여 반환한다.', () => {
  // ...

    // ***************************** 두번째 테스트 코드 *************************
  test('validateValues 파라미터가 존재하지 않으면, 예외를 발생시킨다.', () => {
    expect(() => processRegisterCommand(undefined)).toThrowError();
  });
   // ********************************************************************

  test('validateValues 파라미터가 존재하면 command 객체를 반환한다.');
  test('validateValues의 username 속성이 존재하지 않으면, email 속성으로 대체한다.');
  test('validateValues의 tel 속성이 존재하지 않으면, null로 대체한다.');
});
```

이 코드는 성공할 것이다. 프로덕션 코드에 이미 에러 처리 구문이 포함되어 있기 때문이다.

```js
// processRegisterCommand.js
export default processRegisterCommand(validateValues) {
  if(!validateValues) {
    // validateValues가 undefined이면 에러를 던진다.
    throw Error();
  };

  const command = {
    username: validateValues.username || validateValues.email,
    email: validateValues.email,
    tel: validateValues.tel || null,
    date: new Date(), 
  };

  return command;
};
```

이미 작성된 코드에 테스트를 추가하는 일은 TDD처럼 항상 테스트 실패(Red) 단계로 시작할 수 없다. 이미 작성된 코드를 기반으로 테스트 시나리오를 만들기 때문에 이처럼 작성과 동시에 성공하는 테스트 위주로 만들어지게 된다. 앞에서 작성한 나머지 테스트들도 로직에 큰 문제가 없다면 대부분 성공할 것이다.


```js
// processRegisterCommand.test.js
import processRegisterCommand from '../processRegisterCommand';

describe('유효성 검사를 통과한 값들을 command 데이터 포맷으로 정제하여 반환한다.', () => {
  // ...

  // ***************************** 세번째 테스트 코드 *************************
  test('validateValues 파라미터가 존재하면 command 객체를 반환한다.', () => {
    const param = {
      username: 'user'
      email: 'user@email.com',
      tel: '01012345678',
      agreement: [true, true, true],
    };

    const command = {
      username: 'user',
      email: 'user@email.com',
      tel: '01012345678',
      date: new Date(), // 이 부분 때문에 이 함수는 아직 순수 함수가 아니다. 나중에 리팩터 할 것이다.
    };

    const actual = processRegisterCommand(param);
    expect(actual).toEqual(command);
  });
  // ********************************************************************

  test('validateValues의 username 속성이 존재하지 않으면, email 속성으로 대체한다.');
  test('validateValues의 tel 속성이 존재하지 않으면, null로 대체한다.');
});
```

### 3) Refactor - 테스트가 실패하지 않는 범위 내에서 코드 개선하기

위에서 command 객체의 date 속성에 `new Date()` 값이 들어간다. 이 부분 때문에 processRegisterCommand 함수는 순수 함수가 아니다. 순수 함수는 input이 같으면 output도 항상 같아야 한다. 그런데 이 함수는 2018년 7월 2일에 실행하면 date 속성이 2018-07-02인 command 객체를 반환하고, 2018년 7월 3일에 실행하면 date 속성이 2018-07-03인 command 객체를 반환한다. 즉 input이 동일함에도, 함수를 실행하는 시점에 따라 output이 바뀐다. 이를 순수 함수로 만들어주는 리팩터링을 이 단계에서 시도할 수 있다.

```js
// processRegisterCommand.js
export default processRegisterCommand(validateValues, date) {
  if(!validateValues) {
    throw Error();
  };

  const command = {
    username: validateValues.username || validateValues.email,
    email: validateValues.email,
    tel: validateValues.tel || null,
    date, // date: date, 와 같다.
  };

  return command;
};
```

date를 파라미터로 받도록 리팩터 하였다. 이렇게 하면 비로소 processRegisterCommand 함수는 순수 함수가 된다. 앞서 수정해 준 것과 동일하게 테스트 코드도 약간 수정해준다.

```js
// processRegisterCommand.test.js
import processRegisterCommand from '../processRegisterCommand';
// ...
 // ***************************** 세번째 테스트 코드 *************************
  test('validateValues 파라미터가 존재하면 command 객체를 반환한다.', () => {
    const param = {
      username: 'user'
      email: 'user@email.com',
      tel: '01012345678',
      agreement: [true, true, true],
    };

    const date = new Date();

    const command = {
      username: 'user',
      email: 'user@email.com',
      tel: '01012345678',
      date, 
    };

    const actual = processRegisterCommand(param, date);
    expect(actual).toEqual(command);
  });
  // ********************************************************************
```

단계는 여기서 끝이다! 이제 위 Red ~ Refactor까지를 반복하면 된다. 다만 앞서 말했듯 이미 작성된 코드에 테스트 코드를 추가하는 것이므로 Red 단계를 못 볼 가능성이 높다. 이제 나머지 테스트 코드들도 작성해보자.

```js
  // ...
  // ***************************** 나머지 테스트 코드 *************************
  test('validateValues의 username 속성이 존재하지 않으면, email 속성으로 대체한다.', () => {
    const param = {
      username: undefined,
      email: 'user@gamil.com',
      tel: '01012345678',
      agreement: [true, true, true],
    };

    const date = new Date();

    const command = {
      username: 'user@gamil.com',
      email: 'user@gamil.com',
      tel: '01012345678',
      date,
    };

    const actual = processRegisterCommand(param, date);
    expect(actual).toEqual(command);
  });

  test('validateValues의 tel 속성이 존재하지 않으면, null로 대체한다.', () => {
     const param = {
      username: 'user',
      email: 'user@gamil.com',
      tel: undefined,
      agreement: [true, true, true],
    };

    const date = new Date();

    const command = {
      username: 'user',
      email: 'user@gamil.com',
      tel: null,
      date,
    };

    const actual = processRegisterCommand(param, date);
    expect(actual).toEqual(command);
  });
});
```

모든 테스트 코드가 성공하는 것을 확인했으니, 간단한 리팩터링을 시도한다.

```js
// processRegisterCommand.js
export default processRegisterCommand(source, date) {
  // 1. validateValues 파라미터 명을 source로 변경해주었다.
  if(!source) {
    throw Error();
  };

  // 2. command 변수를 굳이 선언하지 않고, 곧바로 리턴해주었다.
  return {
    username: source.username || source.email,
    email: source.email,
    tel: source.tel || null,
    date,
  };
};
```

이렇게 하면 한결 깔끔해진 함수와 함께, 첫 번째 테스트 코드가 완성된다. 

## ++ 4. 스펙을 추가하자! (선택)
테스트 코드 작성 중 추가적으로 필요한 스펙이 떠오르면 그때 그때 추가해나가면 되는데, 이 때 TDD를 연습해 볼 수 있다. 만약 param 파라미터의 agreement 배열이 모두 true가 아니면 null을 반환하는 스펙을 추가한다고 해보자.

1. 먼저 테스트 코드부터 작성한다.

```js
  // ...
  test('agreement가 하나라도 false인 경우 null을 반환한다.', () => {
    const param = {
      username: 'user',
      email: 'user@gamil.com',
      tel: undefined,
      agreement: [true, true, false],
    };

    const date = new Date();

    const actual = processRegisterCommand(param, date);
    expect(actual).toBeNull();
  })
```

당연히 실패하는 테스트이다. 이 때가 Red 단계이다!

2. 프로덕션 코드를 수정한다.

```js
// processRegisterCommand.js
export default processRegisterCommand(source, date) {
  if(!source) {
    throw Error();
  };

  // ************************ 추가된 부분 ************************
  // source의 agreement 속성에 false가 포함되어 있으면, null을 리턴한다.
  if(source.agreement.find(false)) {
    return null;
  };
  // ********************************************************

  return {
    username: source.username || source.email,
    email: source.email,
    tel: source.tel || null,
    date,
  };
};
```

테스트가 성공하게끔, 프로덕션 코드를 수정해준다. 성공하면, 이 때가 Green 단계이다! 리팩터 단계는 딱히 떠오르지 않아 생략하였다.

이 글을 통해 말하고자 했던 바는, TDD가 아니고 (나도 TDD는 아직 아주아주 허접이니까), 너무 어렵게 생각하지 말고, 아주 작은 부분 부터 하나씩 뜯어서 테스트 해보자는 것이다. 꼭 TDD가 아니더라도 이렇게 조금씩 테스트를 작성해보는 것 만으로도 연습이 된다. 어느정도 테스트 작성이 손에 익게 되면 그 다음에 TDD도 연습해보고, 더 다양한 테스팅 툴도 사용해 보면 된다. 조급하게 생각하지 말자! 경력 30년된 우리 시티오님이 테스트는 10년은 연습해야 한다고 했다!

<!-- 
# 첫 TDD

내 인생 첫 TDD는 CTO님과의 페어 프로그래밍 이었다. **비즈니스 적으로 반드시 필요한**, 하지만 **아주 단순하고 간단한** 기능을 추가하는 것이었다. 볼드체로 표시한 이유가 있다. **반드시 필요하지만 아주 단순하고 간단한** 것부터 시작하자는 것이 이 글의 결론이다. TDD에 대한 이야기는, 아직 TDD가 어쩧니 저쩧니 논할 만큼 충분히 경험해보지 못했기 때문에 소감 정도만 쓰고 넘어가자면, 굉장히 재밌었다. 개인적으로 테스트 코드를 성공하게 만드는 단계보다, 리팩토링 단계에서 더 희열을 느꼈다. 그리고 테스트 수트를 작성하면서 요구사항에 대해 먼저 생각하게 되고, 발생할 수 있는 갖기지 예외 상황을 고려하게 된다. 그러다보니 TDD로 작성한 함수가 그렇지 않은 함수보다 더 방어적으로 작성되는 경향이 있었고, 오류가 발생했을 때 어느 부분에서 발생한 오류인지 더 빨리 찾을 수 있게 되었다. 이게 TDD의 효과인지, 테스트의 효과인지 헷갈리지만, 전보다 코드 품질이 좋아진다는 느낌은 확실히 받았다. -->



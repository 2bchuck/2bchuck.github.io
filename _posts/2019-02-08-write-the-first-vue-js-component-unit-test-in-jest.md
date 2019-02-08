---
layout: post
title: "Jest에서 Vue.js 컴포넌트 첫번째 단위 테스트 작성"
description: "공식 툴과 Jest 프레임워크를 가지고 VueJS 컴포넌트의 단위테스트를 어떻게 작성하는지 배웁니다."
date: 2019-02-08
tags: [translations, vue.js, jest, TDD, Alex Jover]
comments: true
share: true
---

※ [Alex Jover](https://alexjover.com/blog/)씨의 「[Write the first Vue.js Component Unit Test in Jest](https://alexjover.com/blog/write-the-first-vue-js-component-unit-test-in-jest/)」를 저자의 허락 하에 번역했습니다」)

> <a href="https://leanpub.com/testingvuejscomponentswithjest" alt="Test Vue.js components with Jest"><img width="180" src="https://alexjoverm.github.io/images/vuejest.jpg"></a><br>
> [Alex Jover](https://alexjover.com/blog/)씨의 책에서는 vue-test-utils와 Jest의 최신 버전으로 업데이트 된 이 글의 내용을 확인하실 수 있습니다. [책 사러가기](https://leanpub.com/testingvuejscomponentswithjest)

[avoriaz](https://github.com/eddyerburgh/avoriaz)를 바탕으로 만들어진 VueJS의 공식 테스트 라이브러리인 [vue-test-utils](https://github.com/vuejs/vue-test-utils)는 정식 출시를 눈앞에 두고 있습니다. [@EddYerburgh](https://twitter.com/EddYerburgh)는 VueJS 어플리케이션의 단위 테스트를 쉽게 만들수 있는 기능들이 포함된 매우 훌륭한 라이브러리를 만들었습니다.

그리고 [Jest](https://jestjs.io/)는 페이스북(Facebook)에서 테스트를 쉽게 만들 수 있도록 만든 테스팅 프레임워크로 다음과 같은 멋진 기능들을 가지고 있습니다:

* 손대지 않아도 되는 설정(config)
* 인터랙티브 모드
* 병렬 테스트 실행
* 틀에서 벗어난 스파이(spy), 스텁(stub)과 목(mock)
* 내장 코드 커버리지
* 스냅샷 테스트
* 모듈 모킹 유틸

이 라이브러리르 사용하지 않고 karma, mocha, chai, sinon 외 기타등등을 섞어서 이미 테스트를 작성했을 수도 있습니다. 하지만 라이브러리를 사용하면 얼마나 쉬워질 수 있는지 살펴봅시다.

## vue-test 샘플 프로젝트 설정

[`vue-cli`](https://github.com/vuejs/vue-cli)에서 모든 질문에 NO라고 답을해서 프로젝트를 만든다고 가정합시다.

```bash
npm install -g vue-cli
vue init webpack vue-test
cd vue-test
```

그 다음 몇 가지 의존성 모듈들을 설치해줘야 합니다.

```bash
# Install dependencies
npm i -D jest vue-jest babel-jest
```

[`jest-vue-preprocessor`](https://github.com/vire/jest-vue-preprocessor)는 jest가 `.vue` 파일들을 이해할 수 있기 위해 필요하고 `babel-jest`는 Babel과의 연동을 위해 필요합니다.

`beta.1`이 출시되어 `vue-test-utils`는 npm을 통해서 설치할 수 있습니다.

```bash
npm i -D vue-test-utils
```

다음과 같이 `package.json`에 Jest 설정을 추가해봅시다.

```json
...
"jest": {
  "moduleNameMapper": {
    "^vue$": "vue/dist/vue.common.js"
  },
  "moduleFileExtensions": [
    "js",
    "vue"
  ],
  "transform": {
    "^.+\\.js$": "<rootDir>/node_modules/babel-jest",
    ".*\\.(vue)$": "<rootDir>/node_modules/vue-jest"
  }
}
...
```

`moduleFileExtensions`는 Jest에게 어떤 확장자를 찾아야할지 알려주고 `transform`는 해당 확장자에 사용할 전처리기(preprocessor)를 알려줍니다.

마지막으로 `package.json`에 `test` 스크립트를 추가합니다:

```json
{
  "scripts": {
    "test": "jest",
    ...
  },
  ...
}
```

## 컴포넌트 테스트
저는 여기서 싱글 파일 컴포넌트를 사용할 예정이고, `html`, `css`, `js` 파일들로 각각 나눴을 때 작동할지 확인해보지 않았습니다. 다음처럼 한다고 가정해봅시다.

먼저 `src/components`안에 `MessageList.vue` 컴포넌트를 만듭니다.

```html
<template>
  <ul>
    <li v-for="message in messages">
    {% raw %}
      {{ message }}
    {% endraw %}
    </li>
  </ul>
</template>

<script>
export default {
  name: 'list',
  props: ['messages']
}
</script>
```

그리고 `App.vue`를 사용하기 위해 다음과 같이 변경해줍시다.

```html
<template>
  <div id="app">
    <MessageList :messages="messages"/>
  </div>
</template>

<script>
import MessageList from './components/MessageList'

export default {
  name: 'app',
  data: () => ({ messages: ['Hey John', 'Howdy Paco'] }),
  components: {
    MessageList
  }
}
</script>
```

이미 테스트할 수 있는 컴퍼넌트들이 몇 개 생겼습니다. 프로젝트 루트에 `test`폴더를 만들고 그 안에 `App.test.js`를 만들어봅시다.

```javascript
import Vue from "vue";
import App from "../src/App";

describe("App.test.js", () => {
  let cmp, vm;

  beforeEach(() => {
    cmp = Vue.extend(App); // 원본 컴포넌트의 복제본을 생성합니다
    vm = new cmp({
      data: {
        // data를 가짜 data로 바꿔줍니다
        messages: ["Cat"]
      }
    }).$mount(); // Instances and mounts the component
  });

  it('messages는 ["Cat"]과 같다', () => {
    expect(vm.messages).toEqual(["Cat"]);
  });
});
```

이제 `npm test`(또는 단축 버전인 `npm t`)를 실행하면 테스트가 실행되고 통과되어야 합니다. 테스트를 수정해야 하니, **watch 모드**에서 실행하는 것이 더 좋겠습니다.

```bash
npm t -- --watch
```

### 중첩된 컴포넌트의 문제
이 테스트는 매우 단순합니다. 출력값이 예상했던 바와 같은지 확인해봅시다. Jest의 멋진 스냅샷기능을 이용하면 출력값의 스냅샷을 만들어 다음 번 실행되는 결과와 비교할 수 있습니다. 바로 다음의 `it` 구문을 `App.test.js`에 추가해주세요.

```javascript
it("has the expected html structure", () => {
  expect(vm.$el).toMatchSnapshot();
});
```

`test/__snapshots__/App.test.js.snap` 파일이 만들어질 겁니다. 열어서 확인해봅시다.

```javascript
// Jest Snapshot v1, https://goo.gl/fbAQLP

exports[`App.test.js는 예상했던 html 구조 1을 가진다`] = `
<div
  id="app"
>
  <ul>
    <li>
      Cat
    </li>
  </ul>
</div>
`;
```

`MessageList` 컴포넌트가 렌더된 부분에 큰 문제점이 있습니다. 혹시 아셨나요? **단위 테스트는 독립된 단위로 테스트 되어야 합니다.** 그 말인 즉슨 `App.test.js`에서는 `App` 컴포넌트만 테스트해야 한다는 말입니다.

이로 인해 여러가지 문제들이 발생할 수 있습니다. 자식 컴포넌트들이 `created` 훅에서 Vuex 액션이나 상태 변경같이 부수 효과를 일으키는 작업을 하고 있다고 상상해보셨나요? 별로 만들고 싶지 않은 상황임에 틀림없습니다.

다행히 **얕은 렌더링**(Shallow Rendering)으로 이런 상황을 해결할 수 있습니다.

### 얕은 렌더링이란 무엇인가?
[얕은 렌더링](https://airbnb.io/enzyme/docs/api/shallow.html)은 자녀를 제외하고 컴퍼넌트를 렌더링하는 기술입니다. 다음과 같은 상황에 유용합니다.

* 의도한 컴포넌트만 테스트하고 싶을 때(단위 테스트의 뜻입니다)
* 자식 컴포넌트들이 만들 수 있는 부작용들(HTTP 호출이나 액션을 호출하는 등)을 피하고 싶을 때

## vue-test-utils로 컴포넌트 테스트하기
`vue-test-utils`은 얕은 렌더링을 비롯한 여러 기능들을 제공한다. 이전 테스트를 다음처럼 고칠 수 있습니다.:

```javascript
import { shallowMount } from "@vue/test-utils";
import App from "../src/App";

describe("App.test.js", () => {
  let cmp;

  beforeEach(() => {
    cmp = shallowMount(App, {
      // 컴포넌트의 얕은 인스턴스를 생성합니다
      data: {
        messages: ["Cat"]
      }
    });
  });

  it('messages는 ["Cat"]과 같다', () => {
    // cmp.vm을 통해 모든 Vue 인스턴스와 메소드에 접근할 수 있습니다
    expect(cmp.vm.messages).toEqual(["Cat"]);
  });

  it("예상했던 html 구조를 가진다", () => {
    expect(cmp.element).toMatchSnapshot();
  });
});
```

Jest를 왓치 모드로 작동시키고 있었다면, 테스트는 여전히 통과되지만, 스냅샷은 일치하지 않는다. `u`를 눌러서 다시 생성시킨 다음 다시 확인해보세요.

```javascript
// Jest Snapshot v1, https://goo.gl/fbAQLP

exports[`App.test.js는 예상했던 html 구조 1을 가진다`] = `
<div
  id="app"
>
  <!--  -->
</div>
`;
```

보셨나요? 자식 컴포넌트들은 렌더되지 않았고 `App` 컴포넌트를 트리에서 **분리해서** 테스트 했습니다. 또한, 자식 컴포넌트에 `created`를 비롯한 어떤 라이프사이클 훅이 있더라도 호출되지 않습니다.

**얕은 렌더의 구현**이 궁금하면, [소스 코드](https://github.com/vuejs/vue-test-utils/blob/dev/packages/create-instance/create-component-stubs.js)를 확인해보세요. 기본적으로 `components` 키, `render` 메소드 및 라이프사이클 훅들을 스터빙(stubbing)하는 방식입니다.

같은 맥락으로, `MessageList.test.js`를 다음처럼 구현할 수 있습니다.

```javascript
import { mount } from "@vue/test-utils";
import MessageList from "../src/components/MessageList";

describe("MessageList.test.js", () => {
  let cmp;

  beforeEach(() => {
    cmp = mount(MessageList, {
      // `propsData`를 쓰면 props가 오버라이드 된다는 걸 명심하세요
      propsData: {
        messages: ["Cat"]
      }
    });
  });

  it('messages 프로퍼티로 ["Cat"]을 가진다', () => {
    expect(cmp.vm.messages).toEqual(["Cat"]);
  });

  it("예상했던 html 구조를 가진다", () => {
    expect(cmp.element).toMatchSnapshot();
  });
});
```

[Github에서 완성된 예제 코드](https://github.com/alexjoverm/vue-testing-series/tree/lesson-1)를 보실 수 있습니다.

다음: [Jest에서 Vue.js 컴포넌트 깊은 렌더링 테스트](test-deeply-rendered-vue-js-components-in-jest.md)
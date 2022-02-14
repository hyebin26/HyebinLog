---
title: useState가 async인 이유
date: "2022-02-15T01:00:00.284Z"
description: "리액트의 상태를 관리하는 훅인 useState가 async인 이유를 알아보자."
---

참고:[https://github.com/facebook/react/issues/11527](https://github.com/facebook/react/issues/11527)

참고: [jinsunee velog](https://velog.io/@jinsunee/setState%EA%B0%80-%EB%B9%84%EB%8F%99%EA%B8%B0%ED%95%A8%EC%88%98%EC%9D%B8-%EC%9D%B4%EC%9C%A0)

## 1.내부의 일관성 보장(Guaranteeing Internal Consistency)

만약 state가 동기적으로 업데이트 된다고 해도, props는 아니다.(너는 부모의 컴포넌트가 리렌더링 될 때까지 props를 알 수가 없고, 만약에 동기적으로 된다면 batching(일괄처리)는 필요없을 것이다.

현재 React에서 제공하는 객체(state,props,refs)는 내부적으로 서로 일치한다. 즉, 하나의 개체만 사용하는 경우, 완전히 조정(reconciled)된 트리를 참조하도록 보장된다.

근데 만약 state만 동기적으로 발생한다면 ?

```jsx
console.log(this.state.value) // 0
this.setState({ value: this.state.value + 1 })
console.log(this.state.value) // 1
this.setState({ value: this.state.value + 1 })
console.log(this.state.value) // 2

console.log(this.props.value) // 0
this.props.onIncrement()
;({ value: this.state.value + 1 })
console.log(this.props.value) // 0
this.props.onIncrement()
console.log(this.props.value) // 0
```

state만 동기적으로 실행된다면 this.state는 즉시 값을 출력할 수 있다. 하지만 props는 부모의 리렌더링 없이 리렌더링 할 수 없다. 이것이 의미하는 것은 우리는 batch(일괄처리)를 포기해야 한다는 것이다.

이러한 예는 이론에서만 나타나는 것이 아니다. React Redux는 React state가 아닌 것과 props를 혼합하였기 떄문에 이러한 종류의 문제들을 겪곤 했다.

이러한 문제를 리액트는 this.state 와 this.props를 재조정(reconciliation)후에 업데이트 한다. 그래서 리팩토링 전후에 모두 0이 인쇄되는 것을 볼 수 있다.

이러한 문제는 특정한 경우엔 상당히 불편하다. 특히 한 곳에서 완전한 상태 업데이트를 표현하는 방법을 생각하는 대신에, 더 많은 배경에서 상태를 여러 번 변경하는 것을 생각한 사람들에게는 불편할 수 있다. 이 부분에 대해상태 업데이트를 한 곳에서 집중적으로 유지하는 것이 디버깅 관점에서는 더 명확하다고 생각하지만, 위의 부분도 공감하고 있다.

요악하자면, **React 모델은 항상 간결한 코드로 이끌지는 않지만 내부적으로 일관성이 있고 상태를 올리는 것이 안전하다는 것을 보장하고 있다.**

## 2. 동시 업데이트 활성화(Enabling Concurrent Updates)

컨셉적으로, React는 구성 요소 당 하나의 업데이트 대기열이 있는 것처럼 작동한다. 이것이 논의가 필요없는 이유는 업데이트가 한 곳에서 비동기적으로 정확한 순서로 진행되기 때문이다.

우리가 "비동기 렌더링"을 설명한 한 가지 방법은 이벤트 핸들러, 네트워크 응답, 애니메이션 등의 출처에 따라 React가 setState()호출에 다른 우선 순위를 할당할 수 있다는 것이다.

예를들어, 만약에 우리가 메시지를 타이핑한다면, TextBox 컴포넌트안에 있는 타이핑 setState()을 즉시 호출해야 한다. 하지만 만약에 타이핑하는 동안 새로운 메시지를 받는 다면, 스레드 차단으로 인해 입력이 더듬거리게 하는 것보다 새 MessageBubble의 렌더링을 특정 시간 값까지 지연시키는 것이 더 나을 것이다.

만약에 우리가 특정 업데이트가 "낮은 우선순위"를 갖도록 하면 렌더링을 몇 밀리초의 작은 덩어리로 분할하여 사용자에게 눈에 띄지 않게 할 수 있다.

하지만 이렇게 하다보면 성능이 낮다고 생각할수도 있다. 하지만 **비동기적 렌더링은 성능 최적화에만 국한 되지 않는다. 우리는 이것이 React컴포넌트 모델이 할 수 있는 근본적인 변화라고 생각한다.**

예를 들어, 화면이 또 다른 화면으로 이동하는 경우를 고려해보자. 너는 새로운 화면이 렌더링되는 동안 로딩 스피너를 보여줄 것이다.

하지만 만약에 이동이 빠르게 일어난다면, 계속해서 스피너를 호출하고 이동하여, 바로 스피너를 숨기면 사용자 경험이 저하된다. 더군다나 비동기 종속성(데이터,코드,이미지)이 서로 다른 여러 수준의 컴포넌트들이 있는 경우, 하나씩 짧게 깜박이는 스피너가 발생하게 된다. 이것은 시각적으로 좋지 않고, 모든 DOM reflow 때문에 실제로 앱을 느리게 만든다.

만약에 너가 단순하게 다른 뷰를 렌더링하는 setState를 사용할 때, "background"에서 업데이트 된뷰를 렌더링하는 것을 시작할 수 있다면 좋지 않을까? 너가 어떠한 조정코드 없이, 특정한 시간보다 업데이트가 더 걸린다면 스피너를 보여줄 수 있는 것을 선택할 수 있고, 그렇지 않으면 전체의 새로운 하위트리의 async dependencies가 만족될 때 React가 원활한 변화를 수행하도록 한다고 상상해보자. 더욱, 우리가 대기하는 동안 "이전 화면"은 interactive하게 유지되고(다른 요소로 변하는(transition) 것을 선택할 수 있는 등) React는 시간이 너무 오래 걸리면 스피너를 표시해야 한다고 강제한다.

현재 React 모델과 수명 주기에 대한 약간의 조정으로 우리는 실제로 이것을 구현했다. 이것은 this.state가 "동기적"으로 실행되지 않기 떄문에 가능하다. 즉시 실행되면 "이전 버전"이 여전히 볼수 있고 interactive한 동안에 background에서 새로운 버전을 렌더링하는 것을 실현할 방법이 없다.

## 리렌더링 방지

React는 state,props값에 따라 리렌더링이 일어난다.

만약에 state가 동기적이고, 하나의 이벤트안에 3개의 state가 변경된다면 3번의 리렌더링이 발생한다. 이에 대비하여 setState를 비동기 함수로 처리해서 컴포넌트 내의 비동기 함수를 처리하는 콜백 큐가 다 비워지면 리렌더링하도록 설계되었다.
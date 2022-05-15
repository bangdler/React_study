# 방태관리

라고 쓰고, React Hooks Study

Hook을 사용해보면서 궁금했거나, 찾아보면서 공부한 내용을 정리해보았다.

여러 블로그를 찾아보며 공부한 내용(사실 삽질한 기록...)으로 잘못된 부분이 있을 수 있습니다. 양해 부탁드립니다!

## Hooks는 뭘까. 왜 필요할까.

React 공식홈페이지를 찾아보았다.

`Hook은 React 버전 16.8부터 React 요소로 새로 추가되었습니다. Hook을 이용하여 기존 Class 바탕의 코드를 작성할 필요 없이 상태 값과 여러 React의 기능을 사용할 수 있습니다.`

`Hook을 사용하면 컴포넌트로부터 상태 관련 로직을 추상화 할 수 있습니다.
Hook은 계층의 변화 없이 상태 관련 로직을 재사용할 수 있도록 도와줍니다.`

`componentDidMount 와 componentDidUpdate 등 생명주기 메서드를 기반으로 쪼개는 것 보다는, Hook을 통해 서로 비슷한 것을 하는 작은 함수의 묶음으로 컴포넌트를 나누는 방법을 사용할 수 있습니다.`

- => 여러개의 useEffect 를 사용하여 각 상태에 해당하는 함수들을 묶을 수 있다?

`Hook은 Class없이 React 기능들을 사용하는 방법을 제시합니다.`

- => JavaScript의 this 등 문법적으로 몰라도 사용할 수 있다?

대충 종합해보면, Hooks를 통해 보다 사용하기 편리한 함수형 컴포넌트에서도 클래스 컴포넌트에서 사용하던 `상태관리` 기능들을 지원하게 되었다고 볼 수 있을 것 같다.  

## UseState
useState 함수는 초기값 initialValue를 받아서 [상태, 상태를 변경할 핸들러]의 배열을 반환한다.

함수형 컴포넌트는 렌더가 필요할 때마다 함수를 다시 호출하는데, props를 인자로 받아서 리액트 컴포넌트를 리턴하는 것이 함수형 컴포넌트의 개념이기 때문에 렌더링 = 함수호출 이다.

따라서 함수형 컴포넌트에서 상태관리를 하기 위해서는 함수가 다시 호출되었을 때 이전의 상태를 기억하고 있어야 하며, React Hooks는 이를 클로저를 통해서 해결한다.

어떤 식으로 클로저를 활용하여 구현할까? 크롱 수업 때 잠시 구현해봤지만 다시 해보며 정리했다.

### 구현해보기

개념적으로 아래와 같이 구현할 수 있을 것 같다. 

파라미터로 넘어온 initialVal을 state 에 할당하고, setState를 호출하면 새로운 파라미터로 넘어온 newVal을 다시 state에 할당한다.

실제로 state과 setState를 사용하는 시점은 customUseState의 호출이 끝난 후이지만, 클로저를 통해 그 이후에도 접근할 수 있다.
```javascript
const customUseState = (initialVal) => {
    let state = initialVal;
    const setState = (newVal) => {
        state = newVal;
    };
    return [state, setState];
};

const [count, setCount] = customUseState(0)
```

하지만 사실 위의 방법에서 state 는 customUseState의 호출이 끝나고 나면 그대로 리턴되어 더이상 변경할 수 없는 상태가 된다. 즉 setState 를 통해 값을 바꾸더라도 값을 참조할 수 없다.

```javascript
console.log(count) // 0 출력
setCount(count+1) // count 를 1로 변경한다. 
console.log(count) // 0 출력
```

react와 좀 더 유사한 구조로 만들어서 비교해보기로 했다.

state 는 함수 외부에서 선언하고, 어떤 상태를 가진 data를 반환하는 component 함수, 컴포넌트 return 값을 출력하는 render 함수를 추가했다.

setState 실행 시 render 함수를 다시 실행하며, component 내부에서 비동기적으로 setState 가 발생하도록 구현하였다.

```javascript
let _state;

const customUseState = (initialVal) => {
    const state = _state || initialVal
    const setState = (newVal) => {
        _state = newVal
        render() // state 변경 시 render 실행
    }
    return [state, setState];
};

function component() {
    const [count, setCount] = customUseState(0)
    if(count < 1) {
        setTimeout(()=> {
            setCount(count + 1)
        },1000)
    }
    return `${count}번 클릭하였습니다.`
}

function render() {
    console.log(component())
}

render()
// 0번 클릭하였습니다. 1번 클릭하였습니다 출력 
```

이 경우, state 가 여러개로 늘어나면 출력이 이상해진다.
전역 _state 에 값을 저장하므로 의도치 않게 모든 상태가 함께 변경된다. 
```javascript
function component() {
    const [count, setCount] = customUseState(0)
    const [input, setInput] = customUseState('Hi')

    if(count < 1) {
        setTimeout(()=> {
            setCount(count + 1)
        },1000)
    }

    return`${count}번 클릭하였습니다. ${input}이 입력되었습니다.`
}
//출력
//0번 클릭하였습니다. Hi이 입력되었습니다.
//1번 클릭하였습니다. 1이 입력되었습니다.
```
여러개의 state 를 관리할 수 있도록 배열을 사용해봤다.

이를 위해 상태를 저장하는 states 배열과 배열 내의 상태를 관리하는 index 를 추가했다. 

useState 가 실행될 때마다 index 가 변경되어 states 배열에 상태를 하나씩 추가한다.

이 때 setState 함수는 변경되는 index 에 영향을 받으므로 curIndex 로 선언 시 index 를 기억하도록 해주었다.

setState 실행 시 다시 render 가 실행되므로, index 가 상태의 수보다 커지지 않도록 render 실행 시 0으로 초기화 해줘야 한다. 
```javascript
let states = [];
let index = 0;

const customUseState = (initialVal) => {
    const curIndex = index // 선언 시 index 를 setState 에서 사용한다. 
    const state = states[index] || initialVal
    const setState = (newVal) => {
        states[curIndex] = newVal
        render()
    }
    index++
    return [state, setState];
};

function component() {
    const [count, setCount] = customUseState(0)
    const [input, setInput] = customUseState('Hi')

    if(count < 1) {
        setTimeout(()=> {
            setCount(count + 1)
            setInput("bye")
        },1000)
    }

    return`${count}번 클릭하였습니다. ${input}이 입력되었습니다.`
}

function render() {
    index = 0 // 초기화
    console.log(component())
}
//출력
//0번 클릭하였습니다. Hi이 입력되었습니다.
//1번 클릭하였습니다. Hi이 입력되었습니다.
//1번 클릭하였습니다. bye이 입력되었습니다.
```

### React Hook 규칙 관련
React Hook 규칙을 보면 아래와 같다.

`최상위(at the Top Level)에서만 Hook을 호출해야 합니다.`

`반복문, 조건문 혹은 중첩된 함수 내에서 Hook을 호출하지 마세요.`

이유는 `React Hook 이 호출되는 순서에 의존`하기 때문이라고 한다.

간단한 예를 위의 customUseState에 적용해보면 이해를 도울 수 있을 것 같다.

page, value 상태를 추가하고, page 는 `count===0`일 때만 useState 를 적용한다.

처음 선언 시 각각의 setState 함수들은 선언되는 순서대로 index 를 가지고 있고, states[index] 에 해당되는 상태를 변경할 수 있다.

이후 count 가 1이 되면서 render 시에 page 관련 useState 가 선언되지 않았고, 이후 상태가 가리키는 index 가 변경되게 된다.
```javascript
function component() {
    let page, setPage
    const [count, setCount] = customUseState(0) // index 0
    if(count===0) {
        [page, setPage] = customUseState(10) // index 1
    }
    const [input, setInput] = customUseState('Hi') // index 2 -> 1
    const [value, setValue] = customUseState('something') // index 3 -> 2 로 인해 setInput 이 value  를 변경

    if(count < 1) {
        setTimeout(()=> {
            setCount(count + 1)
            setInput("bye") // 처음 선언 시 states[2] 를 변경한다.
        },1000)
    }

    return`${count}번 클릭하였습니다. ${input}이 입력되었습니다. ${page}는 page. ${value}는 value`
}
// 출력
// 0번 클릭하였습니다. Hi이 입력되었습니다. 10는 page. something는 value
// 1번 클릭하였습니다. Hi이 입력되었습니다. undefined는 page. something는 value
// 1번 클릭하였습니다. Hi이 입력되었습니다. undefined는 page. bye는 value
```

### SetState 는 비동기로 동작한다.

아래의 예시를 실행해보면 + 버튼을 한번 클릭했을 때 콘솔에는 0 이 찍힌다. 

```javascript
function App() {
  const [count, setCount] = useState(0);

  return (
      <>
        <div>count {count}</div> // count=1
        <button onClick={() => {
            setCount(count + 1) // 비동기 실행
            console.log(count) // 0 출력
        }}>+</button>
      </>
  );
}
```

클릭 시 setCount 를 연속으로 사용하면 어떨까? 한 번의 클릭으로 0 에서 2로 만들고 싶다면?

각각의 setCount 는 비동기로 실행되고 그 때의 count 는 같은 값이므로 결과는 위와 같다.

연속으로 setState 함수를 실행하면서 이전값을 변경하기 위해서는 setState 에 callback 을 전달해야 한다.
```javascript
function App() {
  const [count, setCount] = useState(0);

  return (
      <>
        <div>count {count}</div>  // count=1
        <button onClick={() => {
            setCount(count + 1) 
            setCount(count + 1)
            console.log(count) // 0 출력
        }}>+</button>
      </>
  );
}
```

아래와 같이 setCount 의 내부 함수로 count 를 연속으로 업데이트할 수 있다.

그렇다면 이 때 rendering 은 몇 번 발생할까?
```javascript
function App() {
    const [count, setCount] = useState(0);
    
    console.log("render", count)

    return (
        <>
            <div>count {count}</div> // count=2
            <button onClick={() => {
                setCount(prevCount => prevCount + 1) // callback 함수로 최신값에 접근할 수 있다. 
                setCount(prevCount => prevCount + 1)
            }}>+</button>
        </>
    );
}
```


- 1번 클릭 시 render 는 1번 발생한다.

   <img alt="img.png" height="100" src="img.png" width="100"/> 

   <img alt="img_1.png" height="100" src="img_1.png" width="100"/>

### batch update ?
setState 가 비동기적으로 작동하는 이유는 일정 시간 동안 변화하는 상태를 모아 한 번에 렌더링 하기 위해서라고 한다.

즉, 불필요한 렌더링을 방지하기 위함. 

리액트는 바로 전달받은 state로 값을 바꾸는 것이 아닌 이전의 리액트 엘리먼트 트리와 전달받은 state가 적용된 엘리먼트 트리를 비교하는 작업을 거치고, 최종적으로 변경된 부분만 DOM에 적용합니다.

이것을 batch update 라고 하며, 16ms 단위로 진행된다고 한다. 16ms 동안 변경된 상태 값들을 모아서 단 한번 리렌더링을 진행.

비슷하지만 약간 다른 예시이다. 두가지 상태를 사용하여 렌더링하는 경우에도 한번 클릭 시 렌더링은 1번만 발생한다.

- 한번 클릭 시

  ![img_2.png](img_2.png)
```javascript
function App() {
    const [count1, setCount1] = useState(0);
    const [count2, setCount2] = useState(0);
    const [renderCount, setRenderCount] = useState(0)

    useEffect(() => {
        setRenderCount(renderCount+1)
    }, [count1, count2])

    return (
        <div className="App">
            <div>count1 click:{count1}</div>
            <div>count2 click:{count2}</div>
            <button onClick={() => {
                setCount1(prevCount => prevCount + 1)
                setCount2(prevCount => prevCount + 1)
            }}>count +</button>
            <div>render count : {renderCount}</div>
        </div>
    );
}
```
단 `setCount2(prevCount => prevCount + 1)` 부분을 ` setTimeout(()=>setCount2(prevCount => prevCount + 1),0)` 변경해주면 렌더링은 두번씩 발생한다.

### 번외 useEffect 의 unMount
useEffect 는 리액트 컴포넌트가 렌더링 될 때마다 특정 작업을 수행하도록 설정 할 수 있는 Hook이다.

두번째 인자를 통해 useEffect 가 실행되는 경우를 지정할 수 있다. 
  - 없는 경우 : 렌더링 시마다 실행
  - 빈 배열 : 가장 처음 렌더링(mounting 될 때)시만 실행
  - 특정 상태값을 가지는 배열 : 해당 상태값이 변경될 때만 실행 

이를 통해 주로 데이터 가져오기, DOM 변경 등을 수행한다.

또한 useEffect 의 return 값(clean up 함수)을 통해 컴포넌트가 사라졌을 때 (unmount시) 특정 작업을 수행할 수 있다고 한다.

언제 사용할 수 있을까 궁금하여 좀 더 찾아보았다.

useEffect 를 사용하여 fetch와 같이 네트워크 요청을 하는 경우, 컴포넌트가 사라지더라도 불필요하게 네트워크 요청을 할 수 있으므로 이 경우 clean-up 함수를 사용한다고 한다.

#### 예시 구현 

우선 보이기를 클릭하면 fetch 버튼이 나오고, fetch 버튼을 누르면 애플 키워드가 들어간 단어를 fetch 요청하여 보여주도록 구성한다.
 
setTimeout 으로 fetch 요청에 약 2초간 delay 를 준 후, fetch 버튼 클릭 후 바로 숨기기를 클릭하면 화면에는 나타나는 게 없음에도 fetch 가 발생한다.

<img alt="img_4.png" height="100" src="img_4.png" width="180"/>
<br>
<img alt="img_3.png" height="350" src="img_3.png" width="200"/>

이 때 fetch 버튼이 포함된 컴포넌트의 useEffect 에 return 으로 fetch 요청 취소를 주어 해당 컴포넌트가 사라질 경우 요청을 취소해보았다.

useEffect 에 abortController 를 생성하고, return 에서 abort 메서드를 실행한다.

정상 fetch 요청한 경우
![img_5.png](img_5.png)

fetch 요청 직후 숨기기 버튼 클릭하여 fetch 취소한 경우
![img_6.png](img_6.png)

전체 코드
```javascript
const App = () => {
  const [visible, setVisible] = useState(false);
  return (
          <div>
            <button
                    onClick={() => {
                      setVisible(!visible);
                    }}
            >
              {visible ? '숨기기' : '보이기'}
            </button>
            <hr />
            {visible? <Info /> : <div>숨겨졌습니다.</div>}
          </div>
  );
};

const Info = () => {
  const [data, setData] = useState([]);
  const [fetchState, setFetchState] = useState(false)

  useEffect(() => {
    console.log("effect");
    const abortController = new AbortController()
    if (fetchState) {
      setTimeout(()=> {
        fetch(KEYWORD_URL, {signal: abortController.signal})
                .then(res => res.json())
                .then(prefixData => prefixData.suggestions.map((v) => v.value))
                .then(finalData => {
                  setData(finalData)
                  console.log("fetch",finalData)
                })
                .catch(err => {
                  if (err.name === 'AbortError') {
                    console.log('fetch aborted');
                  } else {
                    console.error(err.message);
                  }
                })
      }, 2000)

    }
    else {
      setData([])
    }
    return () => {
      console.log("cleanup")
      abortController.abort()
    }
  }, [fetchState]);

  return (
          <div>
            <div>
              <button onClick={()=>setFetchState(!fetchState)}>fetch toggle btn</button>
            </div>
            <div>
              <div>
                <b>data</b>
                {data.map((item, index)=><li key={index}>{item}</li>)}
              </div>

            </div>
          </div>
  );
};
```
### 참고 사이트
- https://ko.reactjs.org/docs/hooks-rules.html#explanation
- https://yeoulcoding.me/m/149
- https://medium.com/humanscape-tech/%EC%9E%90%EB%B0%94%EC%8A%A4%ED%81%AC%EB%A6%BD%ED%8A%B8-%ED%81%B4%EB%A1%9C%EC%A0%80%EB%A1%9C-hooks%EA%B5%AC%ED%98%84%ED%95%98%EA%B8%B0-3ba74e11fda7
- https://taenami.tistory.com/49
- https://velog.io/@kaitlin_k/React%EC%9D%98-useEffect
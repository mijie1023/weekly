# Function Component
## 闭包环境
```
function Counter() {
  const [count, setCount] = useState(0);

  const log = () => {
    setCount(count + 1);
    // console.error('...count=' + count);
    setTimeout(() => {
      console.log(count);
    }, 3000);
  };

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={log}>Click me</button>
    </div>
  );
}

// output
0
1
2
```

但由于对 state 的读取没有通过 this. 的方式，使得 每次 setTimeout 都读取了当时渲染闭包环境的数据，虽然最新的值跟着最新的渲染变了，但旧的渲染里，状态依然是旧值;

## 解决打印出 3 3 3
1. 使用 useRef
```
function Counter() {
  const count = useRef(0);

  const log = () => {
    count.current++;
    setTimeout(() => {
      console.log(count.current);
    }, 3000);
  };

  return (
    <div>
      <p>You clicked {count.current} times</p>
      <button onClick={log}>Click me</button>
    </div>
  );
}
```

通过 useRef 创建的对象，其值只有一份，而且在所有 Rerender 之间共享;

2. 保留原有逻辑，使用useRef包装count，并在useEffect中更新；
```
// 这个 Hook 可以产生一个值，其最新值永远与入参同步
function Counter() {
  const [count, setCount] = useState(0);
  const currentCount = useRef(count);

  useEffect(() => {
    currentCount.current = count;
  });

  const log = () => {
    setCount(count + 1);
    setTimeout(() => {
      console.log(currentCount.current);
    }, 3000);
  };

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={log}>Click me</button>
    </div>
  );
}
```

3. 自定义Hooks 包装useRef
```
function useCurrentValue(value) {
  const ref = useRef(0);

  useEffect(() => {
    ref.current = value;
  }, [value]);

  return ref;
}

function Counter() {
  const [count, setCount] = useState(0);
  const currentCount = useCurrentValue(count);

  const log = () => {
    setCount(count + 1);
    setTimeout(() => {
      console.log(currentCount.current);
    }, 3000);
  };

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={log}>Click me</button>
    </div>
  );
}
```

## 永远要对 useEffect 的依赖诚实
```
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []); //  [count]

  return <h1>{count}</h1>;
}

1  1
1  2
1  3
```

使用 eslint-plugin-react-hooks 自动补齐依赖；保证正确维护 dependencies

## 避免在每次渲染前实例化setInterval
```
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(c => c + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <h1>{count}</h1>;
}
```
?? setCount 每次拿到最新count

使用 userReducer
```
function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);
  const { count, step } = state;

  useEffect(() => {
    const id = setInterval(() => {
      dispatch({ type: "tick" });
    }, 1000);
    return () => clearInterval(id);
  }, [dispatch]);

  return <h1>{count}</h1>;
}

function reducer(state, action) {
  switch (action.type) {
    case "tick":
      return {
        ...state,
        count: state.count + state.step
      };
  }
}
```

## 避免 useEffect dependencies维护不全，应将函数定义写在  useEffect 内部; useCallback 可优化到外部；
```
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    function getFetchUrl() {
      return "https://v?query=" + count;
    }

    getFetchUrl();
  }, [count]);

  return <h1>{count}</h1>;
}

// useCallback 可收敛 function & dependencies
function Counter() {
  const [count, setCount] = useState(0);

  const getFetchUrl = useCallback(() => {
    return "https://v?query=" + count;
  }, [count]);

  useEffect(() => {
    getFetchUrl();
  }, [getFetchUrl]);

  return <h1>{count}</h1>;
}
```

## 为什么 useCallback 比 componentDidUpdate 更好用
```
function Parent() {
  const [ count, setCount ] = useState(0);
  const [ step, setStep ] = useState(0);

  const fetchData = useCallback(() => {
    const url = 'https://v/search?query=' + count + "&step=" + step;
  }, [count, step])

  return (
    <Child fetchData={fetchData} />
  )
}

function Child(props) {
  useEffect(() => {
    props.fetchData()
  }, [props.fetchData])

  return (
    // ...
  )
}
```

useCallback 封装的依赖会被会透传到useEffect中;

## 将函数抽到组件外部
```
function useFetch(count, step) {
  return useCallback(() => {
    const url = "https://v/search?query=" + count + "&step=" + step;
  }, [count, step]);
}
```

count 与 step 都会频繁变化，每次变化就会导致 useFetch 中 useCallback 依赖的变化，进而导致**重新生成函数**; 然而实际上这种函数是没必要每次都重新生成的，反复生成函数会造成大量性能损耗

```
function useDraggable(count, step) {
  return useCallback(() => {
    // 上报日志
    report(count, step);

    // 对区域进行初始化，非常耗时
    // ... 省略耗时代码
  }, [count, step]);
}
```

## 利用 Ref 保证耗时函数依赖不变
```
function useFetch(count, step) {
  const countRef = useRef(count);
  const stepRef = useRef(step);

  useEffect(() => {
    countRef.current = count;
    stepRef.current = step;
  });

  return useCallback(() => {
    const url =
      "https://v/search?query=" + countRef.current + "&step=" + stepRef.current;
  }, [countRef, stepRef]); // 依赖不会变，却能每次拿到最新的值
}
```

将需要更新的区域与耗时区域分离,再将需更新的内容通过 Ref 提供给耗时的区域，实现性能优化;

## 用 useMemo 做局部 PureRender  更细粒度的优化渲染
```
const Child = (props) => {
  useEffect(() => {
    props.fetchData()
  }, [props.fetchData])

  return useMemo(() => (
    // ...
  ), [props.fetchData])
}
```

## 使用 Context 做批量透传
```
const Store = createContext(null);

function Parent() {
  const [count, setCount] = useState(0);
  const [step, setStep] = useState(0);
  const fetchData = useFetch(count, step);

  return (
    <Store.Provider value={{ setCount, setStep, fetchData }}>
      <Child />
    </Store.Provider>
  );
}


// child

const Child = memo((props) => {
  const { setCount } = useContext(Store)

  function onClick() {
    setCount(count => count + 1)
  }

  return (
    // ...
  )
})
```


## 使用 useReducer 为 Context 传递内容瘦身
```
const Store = createContext(null);

function Parent() {
  const [state, dispatch] = useReducer(reducer, { count: 0, step: 0 });

  return (
    <Store.Provider value={dispatch}>
      <Child />
    </Store.Provider>
  );
}

const Child = useMemo((props) => {
  const dispatch = useContext(Store)

  function onClick() {
    dispatch({
      type: 'countInc'
    })
  }

  return (
    // ...
  )
})
```

## 将 state 也放到 Context 中, useMemo 配合 useContext
```
const Store = createContext(null);

function Parent() {
  const [state, dispatch] = useReducer(reducer, { count: 0, step: 0 });

  return (
    <Store.Provider value={{ state, dispatch }}>
      <Count />
      <Step />
    </Store.Provider>
  );
}

const Count = () => {
  const { state, dispatch } = useContext(Store);
  return useMemo(
    () => (
      <button onClick={() => dispatch("incCount")}>
        incCount {state.count}
      </button>
    ),
    [state.count, dispatch]
  );
};

const Step = () => {
  const { state, dispatch } = useContext(Store);
  return useMemo(
    () => (
      <button onClick={() => dispatch("incStep")}>incStep {state.step}</button>
    ),
    [state.step, dispatch]
  );
};
```

## 使用自定义 Hook 处理副作用 -- 异步取数场景
```
const useDataApi = (initialUrl, initialData) => {
  const [url, setUrl] = useState(initialUrl);

  const [state, dispatch] = useReducer(dataFetchReducer, {
    isLoading: false,
    isError: false,
    data: initialData
  });

  useEffect(() => {
    let didCancel = false;

    const fetchData = async () => {
      dispatch({ type: "FETCH_INIT" });

      try {
        const result = await axios(url);
        if (!didCancel) {
          dispatch({ type: "FETCH_SUCCESS", payload: result.data });
        }
      } catch (error) {
        if (!didCancel) {
          dispatch({ type: "FETCH_FAILURE" });
        }
      }
    };

    fetchData();

    return () => {
      didCancel = true;
    };
  }, [url]);

  const doFetch = url => setUrl(url);

  return { ...state, doFetch };
};

// 结合useEffect useReducer

function App(props) {
  const { dispatch } = useContext(Store);

  const { data, isLoading, isError } = useDataApi("https://v", {
    showLog: true
  });

  useEffect(() => {
    dispatch({
      type: "updateLoading",
      data,
      isLoading,
      isError
    });
  }, [dispatch, data, isLoading, isError]);
}
```

## Function Component 的 DefaultProps 怎么处理
```
const Child = ({ type }) => {
  useEffect(() => {
    console.log("type", type);
  }, [type]);

  return <div>Child</div>;
};

Child.defaultProps = {
  type: { a: 1 }
};
```

不断刷新父元素，只会打印出一次日志
因此建议对于 Function Component 的参数默认值，建议使用 React 内置方案解决，因为纯函数的方案不利于保持引用不变;

## useRef 避免冗余render
```
function App() {
  const [count, forceUpdate] = useState(0);

  const schema = { b: 1 };

  return (
    <div>
      <Child schema={schema} />
      <div onClick={() => forceUpdate(count + 1)}>Count {count}</div>
    </div>
  );
}

const Child = memo(props => {
  useEffect(() => {
    console.log("schema", props.schema);
  }, [props.schema]);

  return <div>Child</div>;
});
```

父组件每次刷新，子组件都会打印日志，也就是 子组件 [props.schema] 完全失效了，因为引用一直在变化;

useRef 改进：
```
function App() {
  const [count, forceUpdate] = useState(0);
  const schema = useRef({ b: 1 });

  return (
    <div>
      <Child schema={schema.current} />
      <div onClick={() => forceUpdate(count + 1)}>Count {count}</div>
    </div>
  );
}
```

https://github.com/dt-fe/weekly/blob/v2/104.%E7%B2%BE%E8%AF%BB%E3%80%8AFunction%20Component%20%E5%85%A5%E9%97%A8%E3%80%8B.md

# javascript 新语法
## private class fields
```
class IncreasingCounter {
  #count = 0;

  get value() {
    return this.#count;
  }

  increment() {
    this.#count++;
  }
}
```

## Numeric literals BigInt
```
1234567890123456789 * 123;
// -> 151851850485185200000

1234567890123456789n * 123n;
// -> 151851850485185185047n
```

## BigInt formatting
```
const nf = new Intl.NumberFormat("fr");
nf.format(12345678901234567890n);
// -> '12 345 678 901 234 567 890'
```

## flat & flatmap
```
const array = [1, [2, [3]]];
array.flat(Infinity);
// -> [1, 2, 3]

[2, 3, 4].flatMap(duplicate);
```

## fromEntries
```
const object = { x: 42, y: 50, abc: 9001 }
const result = Object.fromEntries(
  Object.entries(object)
    .filter(([ key, value]) => key.length === 1)
    .map(([ key, value ]) => [ key, value * 2])
)
// -> { x: 84, y: 100 }
```

## Map to Object conversion
```
const map = new Map(Object.entries(object));

const objectCopy = Object.fromEntries(map);
```

## globalThis
```
const getGlobalThis = () => {
  if (typeof self !== "undefined") return self; // web worker 环境
  if (typeof window !== "undefined") return window; // web 环境
  if (typeof global !== "undefined") return global; // node 环境
  if (typeof this !== "undefined") return this; // 独立 js shells 脚本环境
  throw new Error("Unable to locate global object");
};
```

## Intl.RelativeTimeFormat
```
const rtf = new Intl.RelativeTimeFormat("en", { numeric: "auto" });

rtf.format(-1, "day");
// -> 'yesterday'
rtf.format(0, "day");
// -> 'today'
rtf.format(1, "day");
// -> 'tomorrow'
rtf.format(-1, "week");
// -> 'last week'
rtf.format(0, "week");
// -> 'this week'
rtf.format(1, "week");
// -> 'next week'
```

## Promise.allSettled/Promise.any
```
const promises = [
  fetch("/api-call-1"),
  fetch("/api-call-2"),
  fetch("/api-call-3")
];

await Promise.allSettled(promises);
```

即便某个 fetch 失败了，也不会导致 reject 的发生，这样在不在乎是否有项目失败，只要拿到都结束的信号的场景很有用

```
const promises = [
  fetch("/api-call-1"),
  fetch("/api-call-2"),
  fetch("/api-call-3")
];

try {
  const first = await Promise.any(promises);
  // Any of ths promises was fulfilled.
  console.log(first);
} catch (error) {
  // All of the promises were rejected.
}
```

只要有子项 fulfilled，就会完成 Promise.any，哪怕第一个 Promise reject 了，而第二个 Promise fulfilled 了，Promise.any 也会 fulfilled，而对于 Promise.race，这种场景会直接 rejected;

## WeakRef
```
const obj = {};
const weakObj = new WeakRef(obj);
```

使用 weakObj 与 obj 没有任何区别，唯一不同时，obj 可能随时被 GC，而一旦被 GC，弱引用拿到的对象可能就变成 undefined，所以要做好错误保护。

# useRef
The returned object will persist for the full lifetime of the component.

返回的object存在于component整个生命周期；

Essentially, useRef is like a “box” that can hold a mutable value in its .current property.

本质是在 .current 上维护一个变量的盒子；

The only difference between useRef() and creating a {current: ...} object yourself is that useRef will give you the same ref object on every render.

每次渲染引用不变；
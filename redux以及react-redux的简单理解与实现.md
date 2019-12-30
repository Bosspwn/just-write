# redux使用以及简易实现
redux是目前一种比较通用的状态管理方案。对项目中复杂繁琐的各种状态进行统一的管理，是基于flux工作流思想衍生的一套解决方案。同时比较大量的运用了函数式编程的思想，了解并能熟练运用其原理对函数式的理解以及编程能力会有比较大的提升。
## 日常使用中的redux
### createStore
```javascript
export const createStore = (reducer) => {
  // 当前的state
  let state = null;
  // 当前的订阅事件
  let listeners = [];
  // 暴露api，获取当前的state
  const getState = () => state;
  // 暴露api，新增订阅事件
  const subscribe = listener => listeners.push(listener);
  // 暴露api，派发action
  const dispatch = action => {
    state = reducer(state, action);
    listeners.forEach(listener => listener());
  }
  // 初始化state
  dispatch({})
  return {
    getState,
    subscribe,
    dispatch
  }
}
```
createStore是redux的最基础也是最核心功能，他维护一个内部的state状态，并且向外暴露一个获取state接口（getState），一个派发事件改变当前state值（dispatch），一个监听事件（subscribe），同时接受一个reducer作为参数。createStore通过闭包保存了当前的state，同时也是闭包令其只能被当前函数访问，而无法被外界获取。这样就保证了state的不变性，想对state进行更改就必须通过reducer。

### 2. reducer
reducer是用来对store中数据进行修改的操作。
```javascript
const initState = {
  count: 0
}
export const counter = (state = initState, action) => {
  if (!state) {
    state = initState
  }
  switch (action.type) {
    case 'ADD': 
      return {
        ...state,
        count: state.count + 1,
      }
    case 'MINUS':
      return {
        ...state,
        count: state.count - 1,
      }
    default: 
      return { ...state }
  }
}
```
reducer接受一个state，一个action，返回一个新的state。基本思想是通过用户的行为（即action）来判断数据如何改变。单纯的reducer是无法引起数据变动的，必须通过dispatch来触发。

### 3. action
```javascript
export const add = () => { return { type: 'ADD' } };
export const minus = () => { return { type: 'MINUS' } }
```
action 表明用户行为。基本来说action是一个约定优于配置的思想，约定action函数返回一个type表明用户行为或用户希望数据的变化方式（举例：type: 'ADD' 表明用户希望数据增加），一个payload表明数据变化的参数（举例：payload: 123，表明用户希望数据变为123）

*不仅redux，包括vuex，flux等大多数状态管理方案，很多地方都采用了约定优于配置的模式，包括这些方案的衍生方案，如redux的衍生方案dva等*

## react-redux
很多情况下，redux并不是单独使用，而是配合react使用。react也很支持在react的状态管理方案上选用redux，甚至在16.8之后的hooks版本里官方引入了redux。但是如果只用基本的redux的话，在react中显得过于繁琐。下面先说一下普通的redux在react中使用的流程：
- 需要在index配置里实例化一个store，在根组件将store写入props，并监听根组件的render事件，每次store的值更新后需要render页面
- 因为通过props传参，所以只能在一级子页面中获取到props，想要在更下层的props中使用store中的数据，只能通过一层一层的props传递，或者自己写context，不仅繁琐，而且与业务代码高度耦合。很明显是增加开发成本（和加班时间）。

因此redux为了解决这种繁琐，在redux的基础上封装了react-redux，一套专门为react设计的redux解决方案。主要利用了context的机制。
### context
context是react官方的高级（Advanced）api，一般来说，简单的项目中完全不需要使用context，甚至官方不建议在稳定版的app中使用context（就和redux一样，标准的“如果你不知道你的项目中是否需要用到redux，那么就是不需要”），主要原因应该是无法保证context的可靠。因为一是因为还处在实验阶段的context方案并不是很成熟，在版本的更新迭代中可能变动较大，二是因为context的作用类似于将组件的作用域扩大，超过了组件本身的控制范围，那么就有可能破坏组件树的关系。三是context的更新依赖setState，如果使用pureComponent，或者单纯的使用shouldComponentUpdate生命周期，有可能造成组件的state不更新，那就有可能造成context的不更新。那么多个依赖context的组件，其数据都不会更新，因此官方更推荐在多组件间使用props或state进行值传递。

简而言之，如果你能保证context的可靠性，那么使用context可以带来相当便捷的开发体验。

*市面上的很多解决方案都会考虑这种便捷性，最流行的两套状态管理解决方案，redux通过官方并不是十分推荐的context进行开发体验优化，同样的mobx也通过es6官方不推荐的装饰器来优化开发体验*

### Provider
react-redux中有两个常用的组件，即**Provider**和**Connect**，Provider的主要作用是将store写入context，以便所有组件都能获取。
```javascript
import { Component } from 'react';
import PropTypes from 'prop-types';

export class Provider extends Component {
  static childContextTypes = {
    store: PropTypes.object
  }
  getChildContext() {
    return {
      store: this.props.store,
    }
  }
  render() {
    return this.props.children
  }
}
```

### connect
简单来说，connect是一个HOC（高阶组件），主要作用是用来将store中的数据转化为props注入到子组件中。connect接受两个参数mapStateToProps，mapDispatchToProps，返回一个新函数，新函数接受一个react组件（jsx），并返回一个新的组件。本质上来说，connect只是函数式的一些应用。高阶组件的名称来源是js中的高阶函数，即一个函授接受参数并返回一个函数，最著名的应用就是柯里化（curry）。connect接受的两个参数，mapStateToProps是一个Object对象，会匹配存储在store上的state中含有当前键值对的数据返回到props。同样的，mapDispatchToprops会将派发事件转为props，这样方便了开发者在组件中的调用。同时，connect还负责在store中值变化时通知组件。
```javascript
import React, { Component } from 'react';
import PropTypes from 'prop-types';

export const connect = (mapStateToProps = state => state, mapDispatchToProps = {}) => WrapComponent => {
  return class Connect extends Component {
    static contextTypes = {
      store:PropTypes.object
    }
    constructor(props, context) {
      super(props, context);
      this.state = {}
    }
    update() {
      const { store } = this.context;
      this.setState({
        ...this.state.props,
        ...mapStateToProps(store.getState()),
        ...mapDispatchToProps(store.dispatch)
      })
    }
    componentDidMount() {
      const { store } = this.context;
      store.subscribe(() => this.update());
      this.update();
    }
    render() {
      return <WrapComponent {...this.state} />
    }
  }
}
```
app.js
```javascript
const mapDispatchToProps = dispatch => ({
  add: params => dispatch(add(params)),
  minus: params => dispatch(minus(params))
})
```
也可以将mapDispatchToProps中传入的函数抽离出来，即
```javascript
// connect.js
const bindActionCreator = (creator, dispatch) => (...args) => dispatch(creator(...args));
const bindActionCreators = (creators, dispatch) => {
  const result = {};
  Object.entries(creators).forEach(item => {
    result[item[0]] = bindActionCreator(item[1], dispatch)
  })
  return result
}

...

update() {
  const { store } = this.context;
  this.setState({
    ...this.state.props,
    ...mapStateToProps(store.getState()),
    ...bindActionCreators(mapDispatchToProps, store.dispatch)
  })

}
...
// app.js
const mapDispatchToProps = { add, minus };

```
其中，bindActionCreators的作用是将返回对象的action与调用dispatch的步骤组合起来。

至此，这个简易的自制react-redux已经实现了其基本的功能，那么在项目中的使用也就比直接用redux要方便了。

```javascript
// app.js
import React, { Component } from 'react';
import { add, minus } from './actions';
import { connect } from './redux/connect'
const mapStateToProps = state => ({...state});
const mapDispatchToProps = { add, minus };
class App extends Component {
  render() {
    const { count, add, minus } = this.props;
    return (
      <div className="App">
        <button onClick={add}>增加</button>
        <button onClick={minus}>减少</button>
        <div>{count}</div>
      </div>
    );
  }
}
export default connect(mapStateToProps, mapDispatchToProps)(App)

...

// index.js
const store = createStore(counter);
ReactDOM.render(<Provider store={store}><App/></Provider>, document.getElementById('root'))

```

## redux中间件
redux可以解决很多情况下的状态管理问题，但由以上的思路可以看出来目前存在的一个问题：redux的状态管理是同步的。但前端开发在很多情况下都不会单纯的使用同步的事件，与服务器的交互，获取数据，是业务代码中非常重要的一环，而这一环恰恰是异步的。那么按目前的redux的设计来说，就比较捉襟见肘了。

比如，在目前的状况下，想要使用redux获取并存储服务器返回的数据，那么需要以下几步：
```javascript
// action.js

// 通知reduer请求开始
export const fetchStart = () => { return { type: 'FETCH_START' } }

// 通知reducer请求成功
export const fetchSuccess = (data) => { 
  return { 
    type: 'FETCH_SUCCESS', 
    payload: { data: data }
  }
}

// 通知reducer请求失败
export const fetchFail = (err) => {
  return {
    type: 'FETCH_FAIL',
    payload: { err: err }
  }
}

...

// reducer.js
const initState = {
  isFetching: false
}
export const result = (state = initState, action) => {
  switch (action.type) {
    case 'FETCH_START': {
      return { ...state, isFetching: true }
    }
    case 'FETCH_SUCCESS': {
      return { ...state, isFetching: false, ...action.payload }
    }
    case 'FETCH_FAIL': {
      return { ...state, isFetching: false, ...action.payload }
    }
  }
}

// 使用场景
const mapDispatchToProps = dispatch => ({
  fetchStart: params => dispatch(fetchStart(params)),
  fetchSuccess: params => dispatch(fetchSuccess(params)),
  fetchFail: params => dispatch(fetchFail(params)),
})

try {
  fetchStart();
  const { data, success } = await fetch('xxx');
  if (success) {
    fetchSuccess(data);
  } else {
    fetchFail(data)
  }
} catch (err) {
  fetchFail(err)
}
```

很明显，看起来就非常麻烦。

为了解决这种麻烦，redux引入了`中间件（middleware）`。中间件的概念更多的引用在服务端，但虽然客户端和服务端的中间件解决的问题并不相同，但思想大致类似，简而言之，就是封装一个可以链式组合的函数，用于处理一些复杂的问题。就如同redux middleware，主要就是用在action被发起之后，到达 reducer之前的过程中，可以被定制的操作。比如说，你可以利用中间件来进行日志记录、调用异步接口或者路由等等。

### 实现redux middleware
市面上优秀的redux中间件有很多，虽然他们都是redux中间件，希望解决的问题也是redux使用中的不便，但他们的设计思想并不相同。引用gitbook上对redux 异步action的介绍：

>Thunk middleware 并不是 Redux 处理异步 action 的唯一方式：
>- 你可以使用 redux-promise 或者 redux-promise-middleware 来 dispatch Promise 来替代函数。
>- 你可以使用 redux-observable 来 dispatch Observable。
>- 你可以使用 redux-saga 中间件来创建更加复杂的异步 action。
>- 你可以使用 redux-pack 中间件 dispatch 基于 Promise 的异步 Action。
>- 你甚至可以写一个自定义的 middleware 来描述 API 请求，就像这个 真实场景的案例 中的做法一样。
>你也可以先尝试一些不同做法，选择喜欢的，并使用下去，不论有没有使用到 middleware 都行。

无论如何，想要使用redux中间件，首先必须要有一个中间件。无论是最流行的redux-thunk，还是上面介绍的redux-promise，redux-observable，redux-saga等，都是优秀的解决方案，大家可以根据他们的适用场景进行选择。下面我们来实现一个简易的thunk。

```javascript
export const thunk = ({ dispatch, getState }) => next => action => {
  if (typeof action === 'function') {
    return action(dispatch, getState)
  }
  return next(action)
}
```
不是吧，为什么看起来这么简单？因为实际上，redux-thunk的源码就非常的简洁
```javascript
// reudx-thunk
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }
    return next(action);
  };
}
const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;
export default thunk;
```

redux-thunk的源码只有11行，因此无法不让人佩服这些“代码艺术家”。

解析一下thunk，会发现他做了以下的事情
```javascript
// 不使用箭头函数后thunk结构如下

// thunk 是一个经过柯里化的函数。
function thunk({ dispatch, getState }) {
    // thunk 接受dispatch和getState两个方法，返回一个可以接受函数的action creator。
    return function(action) {
    // 如果这个action creator返回的是一个函数，则执行这个函数，否则就按原来的action执行。
      if (typeof action === "function") {
        return action(dispatch, getState);
      }
      return next(action);
    };
  };
}
```

由此可见，实际上redux-thunk的用途就是将原本的redux中dispatch只能接收对象变为了可以接收函数，是对dispatch方法的改造。

那么，中间件有了，应该如何用最小的代价应用到项目中，甚至让使用者感觉并没有做任何额外的操作呢？

既然redux-thunk是对dispatch的改造，那么想要“无缝衔接”，最好的办法就是将createStore也改造啦

```javascript
export const createStore = (reducer, enhancer) => {
  // 如果有第二个参数
  if (enhancer) {
    return enhancer(createStore)(reducer)
  }
  ...
}

// 函数组合，函数式中比较常用且比较重要的方法。主要作用是接收上一个函数的返回值作为参数进行下一个函数的计算
// 举例：将字符bytedance先全部大写，然后再倒序，再截取首位字母，那么正常的方法是写三个函数依次调用，即
// const upperCase = (str) => str.toUpperCase();
// const reverse = (str) => str.split('').reverse().join('');
// const substrHead = (str) => str.substr(0, 1);
// 那么结果就是 
// const result = substrHead(reverse(upperCase('bytedance')))
// 很明显，这三步每一步都依赖上一步的返回值，那么通过compose函数，则可以写成
// const result = compose(substrHead, reverse, upperCase)('bytedance')
// 至少看起来更加清晰简洁
const compose = (...funcs) => {
  if (funcs.length === 0) {
    return arg => arg
  }
  if (funcs.length === 1) {
    return funcs[0]
  }
  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}

// 比较难以理解的操作，实际上可以理解成一个类似递归的东西，enhancer虽然是createStore的参数但还是要调用createStore
export const applyMiddleware = (...middlewares) => createStore => (...args) => {
  const store = createStore(...args);
  // 暂存dispatch方法
  let dispatch = store.dispatch;
  const dispatchAndGetState = {
    getState: store.getState,
    dispatch: (...args) => dispatch(...args),
  }
  // 明显，只能使用一个中间件不符合预期，我们需要的是可以任意加减中间件，因此需要使用compose函数组合。
  // 更新dispatch，将中间件对dispatch的修改应用。
  dispatch = compose(...middlewares.map(middleware => middleware(mid)))(store.dispatch);

  return {
    ...store,
    dispatch
  }
}
```
使用
```javascript
// action.js
const changeCount = (count) => { return { type: 'CHANGE_COUNT', payload: count } };
export const changeCountAsync = (count) => dispatch => {
  setTimeout(() => {
    dispatch(changeCount(count))
  }, 100);
}

// index.js
import { createStore, applyMiddleware } from './redux/redux';
import { counter } from './reducers/';
import { Provider } from './redux/provider';
import { thunk } from './redux/redux-thunk'

const store = createStore(counter, applyMiddleware(thunk));
ReactDOM.render(<Provider store={store}><App/></Provider>, document.getElementById('root'))
```

上面基本就是redux和react-redux的基本使用和简单实现，当然在真实的使用场景中需要注意的点更多，情形更加复杂，可能需要引入能处理更复杂情况的中间件等，不做赘述。
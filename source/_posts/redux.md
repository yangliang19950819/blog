---
title: 手写redux
cover: /img/redux.webp
---

### redux简介 一个可以用于多端的状态管理框架

- Redux is a predictable state container for JavaScript apps. It helps you write applications that behave consistently, run in different environments (client, server, and native).


### redux 核心函数createStore combindreducers applyMiddleware bindActionCreaters


- createStore函数可接收2个参数reducer,enhancer配置对象.由于此文章实现redux主要功能，也为了方便理解，所有对applyMiddleware函数的执行结果作为第三个参数传入。

- combindreducers执行传入reduers一个reducer的对象，返回一个总的reducer函数，在总的reducer函数返回新的state。

- applyMiddleware的原理主要是重写store.dispatch函数，用洋葱模式的形式将每个中间件作为next参数传入一层一层的包裹执行。

- bindActionCreaters对dispacth做一层封装，可以执行异步操作等，拿到数据后出发dispatch改变数据。


### 手写代码

```js
function createStore(reducer,state = {},rewriteCreateStore){
    if(rewriteCreateStore){
        createStore = rewriteCreateStore(createStore);
        return createStore(reducer,state)
    }
    state = state;
    listeners = [];    // 用于存放订阅的函数
    function subscribe(listener){
        if(typeof listener === 'function'){
            listeners.push(listener)
        }else{
            throw new Error (`${listener} is not function`)
        }
        return function(listener){      // 返回函数用于取消订阅的事件   
            const index = listeners.indexOf(listener);
            if(index > -1){
                listeners.splice(index,1)
            }
        }
    }
    function getState(){
        return state
    }
    function dispatch(action){
        state = reducer(state,action);
        listeners.forEach((listener) => {
            listener()
        })
    }
    dispatch({type:Symbol()})   //用于初始化state 防止直接getState时 没有值
    return {
        getState,
        subscribe,
        dispatch
    }
}

function combindReducers(reducers){
    const newState = {};
    return function(state,action){
        Object.keys(reducers).forEach(key => {
            const reducer = reducers[key];
            const prevState = state[key];
            const nextState = reducer(prevState,action)
            newState[key] = nextState
        })
        return newState
    }
}

function counterReducer (state = { count:0 },action){
    switch(action.type){
        case 'INCREMENT':
            return{
                ...state,
                count:state.count + 1,
            }
        case 'DECREMENT':
            return{
                ...state,
                count:state.count - 1,
            }
        default:
            return state        
    }
}

function infoReducer (state = {name:'有个名字',description:'一个称谓'},action){
    switch(action.type){
        case 'SET_NAME':
            return {
                ...state,
                name:action.payload.name
            }
        case 'SET_DESCRIPTION':
            return {
                ...state,
                description:action.payload.description
            }  
        default:
            return state      
    }
}

const reducer = combindReducers({
    counter:counterReducer,
    info:infoReducer
})

function compose(funcs){
    if(funcs.length === 0)return  (...arg) => arg;
    if(funcs.length === 1)return funcs[0];
    return funcs.reduce((acc,cur) => (...arg) => acc(cur(...arg)))
}

function applyMiddleware(...middlewares){
    return function(createStore){
        return function(reducer,state){
            const store = createStore(reducer,state);
            const chain = middlewares.map(middleware => middleware(store));
            const dispatch = compose(chain)(store.dispatch)
            return {
                ...store,
                dispatch
            }
        }
    }
}

function expectionMiddleware(store){
    return function (next){
        return function(action){
            try {
                console.log('caghterror')
                next(action)
            }catch(e){
                console.log(e)
            }   
        }
    }
}

function loggerMiddleware(store){
    return function(next){
        return function (action){
            console.log('logger')
            next(action)
        }
    }
}


function timeMiddleware(store){
    return function(next){
        return function (action){
            console.log(new Date().getTime(),'🀄️')
            next(action)
        }
    }
}

const rewriteCreateStore = applyMiddleware(expectionMiddleware,timeMiddleware,loggerMiddleware)

const store = createStore(reducer,{},rewriteCreateStore)

store.subscribe(()=>{
    console.log(store.getState(),'🌈')
})

function increment (payload){
    return {
        type:'INCREMENT'
    }
}

function setName (payload){
    return {
        type:'SET_NAME',
        payload
    }
}


function bindActionCreators(actions,dispatch){
    const newActions = {}
    Object.keys(actions).forEach(key => {
        newActions[key] = (...arg) => {
            dispatch(actions[key].apply(this,...arg))
        }
    })
    return newActions
}

const actions = bindActionCreators({
    increment,setName
},store.dispatch)

store.dispatch({type:'INCREMENT'})
actions.setName({name:'故事的小黄花'})

```

![avatar](/img/redux-res.webp)

- 可以看到执行结果中加入的监听错误和时间以及日志的中间件的执行，以及store直接dispatch和通过actions函数执行都改变了state
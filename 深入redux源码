redux 源码总共775行 值得学习的的思想有很多 看我注释一一道来   :)
> index.js
      
      function isCrushed() {}
       //  利用一个function判断代码是否被压缩 如果被压缩 名字会改变   
       if (
         process.env.NODE_ENV !== 'production' &&
         typeof isCrushed.name === 'string' &&
         isCrushed.name !== 'isCrushed'
       ) {
         warning(
           'You are currently using minified code outside of NODE_ENV === \'production\'. ' +
           'This means that you are running a slower development build of Redux. ' +
           'You can use loose-envify (https://github.com/zertosh/loose-envify) for browserify ' +
           'or DefinePlugin for webpack (http://stackoverflow.com/questions/30030031) ' +
           'to ensure you have the correct code for your production build.'
         )
       }
       export {
         createStore,
         combineReducers,
         bindActionCreators,
         applyMiddleware,
         compose
       }  //对外暴露 五个函数
   
> compose.js

             export default function compose(...funcs) {
             if (funcs.length === 0) {
               return arg => arg
             }
             if (funcs.length === 1) {
               return funcs[0] //当传入参数为1时 返回这个参数
             }
             return funcs.reduce((a, b) => (...args) => a(b(...args))) // 当传入参数为多个时  一次调用
           }
           
           compose(funcA, funcB, funcC) 变为 compose(funcA(funcB(funcC()))) //如果有多层回调时  利用函数式编程的方式将代码右移
           compose.js
           
 
> createStore.js   

           import isPlainObject from 'lodash/isPlainObject'
           import $$observable from 'symbol-observable'
          
           export const ActionTypes = {
             INIT: '@@redux/INIT'
           }  // 设置初始actionType
           
           
           export default function createStore(reducer, preloadedState, enhancer) {  // enhancer: applyMiddleWare
             if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') { // 参数后置 如果 第二个参数是function  那个就是enhancer参数  
               enhancer = preloadedState
               preloadedState = undefined
             }
           
             if (typeof enhancer !== 'undefined') {
               if (typeof enhancer !== 'function') {
                 throw new Error('Expected the enhancer to be a function.')
               }
           
               return enhancer(createStore)(reducer, preloadedState)  // 如果有enhancer参数  就调用enhancer参数
             }
           
             if (typeof reducer !== 'function') {        // 如果reducer参数不是一个函数 那么就抛出一个异常
               throw new Error('Expected the reducer to be a function.')
             }
           
             let currentReducer = reducer
             let currentState = preloadedState
             let currentListeners = []
             let nextListeners = currentListeners
             let isDispatching = false
           
             function ensureCanMutateNextListeners() {
               if (nextListeners === currentListeners) {
                 nextListeners = currentListeners.slice()
               }
             }
           
    
             function getState() {
               return currentState
             }
           
             function subscribe(listener) {   // 监听器  返回解绑监听器的一个函数
               if (typeof listener !== 'function') {
                 throw new Error('Expected listener to be a function.')
               }
           
               let isSubscribed = true
           
               ensureCanMutateNextListeners()
               nextListeners.push(listener)
           
               return function unsubscribe() {   
                 if (!isSubscribed) {
                   return
                 }
           
                 isSubscribed = false
           
                 ensureCanMutateNextListeners()
                 const index = nextListeners.indexOf(listener)
                 nextListeners.splice(index, 1)
               }
             }
           
        
             function dispatch(action) {   // dispatch 方法  返回action
               if (!isPlainObject(action)) {   // 如果action不是一个对象 抛出一个异常
                 throw new Error(
                   'Actions must be plain objects. ' +
                   'Use custom middleware for async actions.'
                 )
               }
           
               if (typeof action.type === 'undefined') {
                 throw new Error(
                   'Actions may not have an undefined "type" property. ' +
                   'Have you misspelled a constant?'
                 )
               }
           
               if (isDispatching) {
                 throw new Error('Reducers may not dispatch actions.')
               }
           
               try {
                 isDispatching = true
                 currentState = currentReducer(currentState, action)  // 调用 reducer
               } finally {
                 isDispatching = false
               }
           
               const listeners = currentListeners = nextListeners
               for (let i = 0; i < listeners.length; i++) {
                 const listener = listeners[i]
                 listener()
               }
           
               return action
             }
           
             function replaceReducer(nextReducer) {  // 替换store中当前的reducer
               if (typeof nextReducer !== 'function') {
                 throw new Error('Expected the nextReducer to be a function.')
               }
           
               currentReducer = nextReducer
               dispatch({ type: ActionTypes.INIT })
             }
           
             function observable() {  //  可观察对象
               const outerSubscribe = subscribe
               return {
                 subscribe(observer) {
                   if (typeof observer !== 'object') {
                     throw new TypeError('Expected the observer to be an object.')
                   }
                   function observeState() {
                     if (observer.next) {
                       observer.next(getState())
                     }
                   }
                   observeState()
                   const unsubscribe = outerSubscribe(observeState)
                   return { unsubscribe }
                 },
           
                 [$$observable]() {
                   return this
                 }
               }
             }
             return {
               dispatch,
               subscribe,
               getState,
               replaceReducer,
               [$$observable]: observable
             }
           }


> combineReducers.js

           import { ActionTypes } from './createStore'  // 获取初始的actinType
           import isPlainObject from 'lodash/isPlainObject'
           import warning from './utils/warning'  // 通用警告
           
           function getUndefinedStateErrorMessage(key, action) {  // state中没有相应key
             const actionType = action && action.type
             const actionName = (actionType && `"${actionType.toString()}"`) || 'an action'
           
             return (
               `Given action ${actionName}, reducer "${key}" returned undefined. ` +
               `To ignore an action, you must explicitly return the previous state. ` +
               `If you want this reducer to hold no value, you can return null instead of undefined.`
             )
           }
           
           function getUnexpectedStateShapeWarningMessage(inputState, reducers, action, unexpectedKeyCache) {
             const reducerKeys = Object.keys(reducers)
             const argumentName = action && action.type === ActionTypes.INIT ?
               'preloadedState argument passed to createStore' :
               'previous state received by the reducer'
           
             if (reducerKeys.length === 0) {
               return (
                 'Store does not have a valid reducer. Make sure the argument passed ' +
                 'to combineReducers is an object whose values are reducers.'
               )
             }
           
             if (!isPlainObject(inputState)) {
               return (
                 `The ${argumentName} has unexpected type of "` +
                 ({}).toString.call(inputState).match(/\s([a-z|A-Z]+)/)[1] +
                 `". Expected argument to be an object with the following ` +
                 `keys: "${reducerKeys.join('", "')}"`
               )
             }
           
             const unexpectedKeys = Object.keys(inputState).filter(key =>
               !reducers.hasOwnProperty(key) &&
               !unexpectedKeyCache[key]
             )
           
             unexpectedKeys.forEach(key => {
               unexpectedKeyCache[key] = true
             })
           
             if (unexpectedKeys.length > 0) {
               return (
                 `Unexpected ${unexpectedKeys.length > 1 ? 'keys' : 'key'} ` +
                 `"${unexpectedKeys.join('", "')}" found in ${argumentName}. ` +
                 `Expected to find one of the known reducer keys instead: ` +
                 `"${reducerKeys.join('", "')}". Unexpected keys will be ignored.`
               )
             }
           }
           
           function assertReducerShape(reducers) {
             Object.keys(reducers).forEach(key => {
               const reducer = reducers[key]
               const initialState = reducer(undefined, { type: ActionTypes.INIT })
           
               if (typeof initialState === 'undefined') {
                 throw new Error(
                   `Reducer "${key}" returned undefined during initialization. ` +
                   `If the state passed to the reducer is undefined, you must ` +
                   `explicitly return the initial state. The initial state may ` +
                   `not be undefined. If you don't want to set a value for this reducer, ` +
                   `you can use null instead of undefined.`
                 )
               }
           
               const type = '@@redux/PROBE_UNKNOWN_ACTION_' + Math.random().toString(36).substring(7).split('').join('.')
               if (typeof reducer(undefined, { type }) === 'undefined') {
                 throw new Error(
                   `Reducer "${key}" returned undefined when probed with a random type. ` +
                   `Don't try to handle ${ActionTypes.INIT} or other actions in "redux/*" ` +
                   `namespace. They are considered private. Instead, you must return the ` +
                   `current state for any unknown actions, unless it is undefined, ` +
                   `in which case you must return the initial state, regardless of the ` +
                   `action type. The initial state may not be undefined, but can be null.`
                 )
               }
             })
           }
    
           export default function combineReducers(reducers) { // 接受一个对象 value为reducer  返回一个整体的reducer
             const reducerKeys = Object.keys(reducers) // 获取每一个reducer的key值
             const finalReducers = {}
             for (let i = 0; i < reducerKeys.length; i++) {
               const key = reducerKeys[i]
           
               if (process.env.NODE_ENV !== 'production') {  // 如果是开发环境 
                 if (typeof reducers[key] === 'undefined') {
                   warning(`No reducer provided for key "${key}"`)
                 }
               }
           
               if (typeof reducers[key] === 'function') {  // 如果传入reducer是一个函数
                 finalReducers[key] = reducers[key]
               }
             }
             const finalReducerKeys = Object.keys(finalReducers)
           
             let unexpectedKeyCache
             if (process.env.NODE_ENV !== 'production') {
               unexpectedKeyCache = {}
             }
           
             let shapeAssertionError
             try {
               assertReducerShape(finalReducers)
             } catch (e) {
               shapeAssertionError = e
             }
           
             return function combination(state = {}, action) {  
               if (shapeAssertionError) {
                 throw shapeAssertionError
               }
           
               if (process.env.NODE_ENV !== 'production') {
                 const warningMessage = getUnexpectedStateShapeWarningMessage(state, finalReducers, action, unexpectedKeyCache)
                 if (warningMessage) {
                   warning(warningMessage)
                 }
               }
           
               let hasChanged = false
               const nextState = {}
               for (let i = 0; i < finalReducerKeys.length; i++) {
                 const key = finalReducerKeys[i]
                 const reducer = finalReducers[key]    // 获取每一个reducer
                 const previousStateForKey = state[key]  
                 const nextStateForKey = reducer(previousStateForKey, action)  
                 if (typeof nextStateForKey === 'undefined') {
                   const errorMessage = getUndefinedStateErrorMessage(key, action)
                   throw new Error(errorMessage)
                 }
                 nextState[key] = nextStateForKey
                 hasChanged = hasChanged || nextStateForKey !== previousStateForKey  //记录上次reducer是否改变
               }
               return hasChanged ? nextState : state
             }
           }
           
           
>  applyMiddleware.js



                    import compose from './compose'
                    
                    export default function applyMiddleware(...middlewares) {
                      return (createStore)  //createStore中enhancer 传入createStore参数
                              => (reducer, preloadedState, enhancer)
                               => {  
                        const store = createStore(reducer, preloadedState, enhancer)
                        let dispatch = store.dispatch
                        let chain = []
                    
                        const middlewareAPI = {
                          getState: store.getState,
                          dispatch: (action) => dispatch(action)
                        }
                        chain = middlewares.map(middleware => middleware(middlewareAPI)) // 调用中间件
                        dispatch = compose(...chain)(store.dispatch)  // 将dispatch作为参数传入中间件
                    
                        return {
                          ...store,
                          dispatch  // 返回一个已经进行包装的dispatch
                        }
                      }
                    }
  中间件示例: thunk源码
  
                function createThunkMiddleware(extraArgument) {
                  return ({ dispatch, getState })  // 参数middlewareAPI
                             => next  // 获取传入dispacth 
                             => action => {     
                    if (typeof action === 'function') {  // 如果action是一个参数 则调用action
                      return action(dispatch, getState, extraArgument);  // 调用thunk产生的异步函数 在异步完成时发起dispatch
                    }
                
                    return next(action); 
                  };
                }
                
                const thunk = createThunkMiddleware();
                thunk.withExtraArgument = createThunkMiddleware;
                
                export default thunk;
 
 
 >   中间原理   触发createStore  判断是否有enhancer 如果有 
      调用enhancer 如果没有 enhancer 走正常dispatch流程
      巧妙之处在于 compose  将一层层回调转换为横向扩展 
     reduce 
     action 作为介质  store作为基石  dispatch作为发射器  
     middle作为中间件 reducer作为一个过滤件 完成单向数据的流动 redux 只是一个工具 如果想要和react一起用 需要配合react-redux一起使用
           
         

       
       
       
       
       
       
       
       
       

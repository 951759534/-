> react与redux是两个独立的部分 react只是一个负责view的一个js库 而redux是一个状态管理的库 这两者通过react-redux联系在一起   
接下来 就进行react-redux的源码的解读 更好的理解react react-redux redux的关系  更好的理解react-redux如何将react与redux连接起来    

>           index.js 

            // 对外暴露四个属性  Provider  createProvider connectAdvanced  connect
            import Provider, { createProvider } from './components/Provider'
            import connectAdvanced from './components/connectAdvanced'
            import connect from './connect/connect'
            
            export { Provider, createProvider, connectAdvanced, connect }
            
>          Provider.js
           
          
           import { Component, Children } from 'react' //    引入react组件
           import PropTypes from 'prop-types'       //  引入属性检查
           import { storeShape, subscriptionShape } from '../utils/PropTypes'  // 类型检查限制
           import warning from '../utils/warning'              
           
           let didWarnAboutReceivingStore = false
           function warnAboutReceivingStore() {
             if (didWarnAboutReceivingStore) {
               return
             }
             didWarnAboutReceivingStore = true
           
             warning(
               '<Provider> does not support changing `store` on the fly. ' +
               'It is most likely that you see this error because you updated to ' +
               'Redux 2.x and React Redux 2.x which no longer hot reload reducers ' +
               'automatically. See https://github.com/reactjs/react-redux/releases/' +
               'tag/v2.0.0 for the migration instructions.'
             )
           }
           
           export function createProvider(storeKey = 'store', subKey) {  // 封装Provider方法 调用时返回provider组件
               const subscriptionKey = subKey || `${storeKey}Subscription`
           
               class Provider extends Component {
                   getChildContext() {
                     return { [storeKey]: this[storeKey], [subscriptionKey]: null }
                   }
           
                   constructor(props, context) {
                     super(props, context)
                     this[storeKey] = props.store;
                   }
           
                   render() {
                     return Children.only(this.props.children)    // Provider有且只接受一个chirldren   required
                   }
               }
           
               if (process.env.NODE_ENV !== 'production') {
                 Provider.prototype.componentWillReceiveProps = function (nextProps) {
                   if (this[storeKey] !== nextProps.store) {
                     warnAboutReceivingStore()
                   }
                 }
               }
           
               Provider.propTypes = {
                   store: storeShape.isRequired,
                   children: PropTypes.element.isRequired,
               }
               Provider.childContextTypes = {
                   [storeKey]: storeShape.isRequired,
                   [subscriptionKey]: subscriptionShape,
               }
           
               return Provider
           }
           
           export default createProvider()
           
                                 
>              connectAdvanced.js


                    import hoistStatics from 'hoist-non-react-statics'
                    import invariant from 'invariant'
                    import { Component, createElement } from 'react'
                    
                    import Subscription from '../utils/Subscription'
                    import { storeShape, subscriptionShape } from '../utils/PropTypes'
                    
                    let hotReloadingVersion = 0
                    const dummyState = {}
                    function noop() {}
                    function makeSelectorStateful(sourceSelector, store) {
                      const selector = {                         // 定义一个selector对象
                        run: function runComponentSelector(props) {
                          try {
                            const nextProps = sourceSelector(store.getState(), props)  
                            if (nextProps !== selector.props || selector.error) {
                              selector.shouldComponentUpdate = true
                              selector.props = nextProps
                              selector.error = null
                            }
                          } catch (error) {
                            selector.shouldComponentUpdate = true
                            selector.error = error
                          }
                        }
                      }
                    
                      return selector
                    }
                    
                    export default function connectAdvanced(
                      selectorFactory,
                      {
                        getDisplayName = name => `ConnectAdvanced(${name})`,
                        methodName = 'connectAdvanced',
                        renderCountProp = undefined,
                        shouldHandleStateChanges = true,
                        storeKey = 'store',
                        withRef = false,
                        ...connectOptions
                      } = {}
                    ) {
                      const subscriptionKey = storeKey + 'Subscription'
                      const version = hotReloadingVersion++
                    
                      const contextTypes = {
                        [storeKey]: storeShape,
                        [subscriptionKey]: subscriptionShape,
                      }
                      const childContextTypes = {
                        [subscriptionKey]: subscriptionShape,
                      }
                    
                      return function wrapWithConnect(WrappedComponent) {
                        invariant(
                          typeof WrappedComponent == 'function',
                          `You must pass a component to the function returned by ` +
                          `connect. Instead received ${JSON.stringify(WrappedComponent)}`
                        )
                    
                        const wrappedComponentName = WrappedComponent.displayName
                          || WrappedComponent.name
                          || 'Component'
                    
                        const displayName = getDisplayName(wrappedComponentName)
                    
                        const selectorFactoryOptions = {
                          ...connectOptions,
                          getDisplayName,
                          methodName,
                          renderCountProp,
                          shouldHandleStateChanges,
                          storeKey,
                          withRef,
                          displayName,
                          wrappedComponentName,
                          WrappedComponent
                        }
                    
                        class Connect extends Component {
                          constructor(props, context) {
                            super(props, context)
                    
                            this.version = version
                            this.state = {}
                            this.renderCount = 0
                            this.store = props[storeKey] || context[storeKey]
                            this.propsMode = Boolean(props[storeKey])
                            this.setWrappedInstance = this.setWrappedInstance.bind(this)
                    
                            invariant(this.store,
                              `Could not find "${storeKey}" in either the context or props of ` +
                              `"${displayName}". Either wrap the root component in a <Provider>, ` +
                              `or explicitly pass "${storeKey}" as a prop to "${displayName}".`
                            )
                    
                            this.initSelector()
                            this.initSubscription()
                          }
                    
                          getChildContext() {
                            const subscription = this.propsMode ? null : this.subscription
                            return { [subscriptionKey]: subscription || this.context[subscriptionKey] }
                          }
                    
                          componentDidMount() {
                            if (!shouldHandleStateChanges) return
                            this.subscription.trySubscribe()
                            this.selector.run(this.props)
                            if (this.selector.shouldComponentUpdate) this.forceUpdate()
                          }
                    
                          componentWillReceiveProps(nextProps) {
                            this.selector.run(nextProps)
                          }
                    
                          shouldComponentUpdate() {
                            return this.selector.shouldComponentUpdate
                          }
                    
                          componentWillUnmount() {
                            if (this.subscription) this.subscription.tryUnsubscribe()
                            this.subscription = null
                            this.notifyNestedSubs = noop
                            this.store = null
                            this.selector.run = noop
                            this.selector.shouldComponentUpdate = false
                          }
                    
                          getWrappedInstance() {
                            invariant(withRef,
                              `To access the wrapped instance, you need to specify ` +
                              `{ withRef: true } in the options argument of the ${methodName}() call.`
                            )
                            return this.wrappedInstance
                          }
                    
                          setWrappedInstance(ref) {
                            this.wrappedInstance = ref
                          }
                    
                          initSelector() {
                            const sourceSelector = selectorFactory(this.store.dispatch, selectorFactoryOptions)
                            this.selector = makeSelectorStateful(sourceSelector, this.store)
                            this.selector.run(this.props)
                          }
                    
                          initSubscription() {
                            if (!shouldHandleStateChanges) return
                            const parentSub = (this.propsMode ? this.props : this.context)[subscriptionKey]
                            this.subscription = new Subscription(this.store, parentSub, this.onStateChange.bind(this))
                    
                    
                            this.notifyNestedSubs = this.subscription.notifyNestedSubs.bind(this.subscription)
                          }
                    
                          onStateChange() {
                            this.selector.run(this.props)
                    
                            if (!this.selector.shouldComponentUpdate) {
                              this.notifyNestedSubs()
                            } else {
                              this.componentDidUpdate = this.notifyNestedSubsOnComponentDidUpdate
                              this.setState(dummyState)
                            }
                          }
                    
                          notifyNestedSubsOnComponentDidUpdate() {
                            this.componentDidUpdate = undefined
                            this.notifyNestedSubs()
                          }
                    
                          isSubscribed() {
                            return Boolean(this.subscription) && this.subscription.isSubscribed()
                          }
                    
                          addExtraProps(props) {
                            if (!withRef && !renderCountProp && !(this.propsMode && this.subscription)) return props
                            const withExtras = { ...props }
                            if (withRef) withExtras.ref = this.setWrappedInstance
                            if (renderCountProp) withExtras[renderCountProp] = this.renderCount++
                            if (this.propsMode && this.subscription) withExtras[subscriptionKey] = this.subscription
                            return withExtras
                          }
                    
                          render() {
                            const selector = this.selector
                            selector.shouldComponentUpdate = false
                    
                            if (selector.error) {
                              throw selector.error
                            } else {
                              return createElement(WrappedComponent, this.addExtraProps(selector.props))
                            }
                          }
                        }
                    
                        Connect.WrappedComponent = WrappedComponent
                        Connect.displayName = displayName
                        Connect.childContextTypes = childContextTypes
                        Connect.contextTypes = contextTypes
                        Connect.propTypes = contextTypes
                    
                        if (process.env.NODE_ENV !== 'production') {
                          Connect.prototype.componentWillUpdate = function componentWillUpdate() {
                            if (this.version !== version) {
                              this.version = version
                              this.initSelector()
                              let oldListeners = [];
                    
                              if (this.subscription) {
                                oldListeners = this.subscription.listeners.get()
                                this.subscription.tryUnsubscribe()
                              }
                              this.initSubscription()
                              if (shouldHandleStateChanges) {
                                this.subscription.trySubscribe()
                                oldListeners.forEach(listener => this.subscription.listeners.subscribe(listener))
                              }
                            }
                          }
                        }
                    
                        return hoistStatics(Connect, WrappedComponent)
                      }
                    }

           
           
           
     
>               connect.js

                import connectAdvanced from '../components/connectAdvanced'
                import shallowEqual from '../utils/shallowEqual'
                import defaultMapDispatchToPropsFactories from './mapDispatchToProps'
                import defaultMapStateToPropsFactories from './mapStateToProps'
                import defaultMergePropsFactories from './mergeProps'
                import defaultSelectorFactory from './selectorFactory'
                
          
                function match(arg, factories, name) {
                  for (let i = factories.length - 1; i >= 0; i--) {
                    const result = factories[i](arg)
                    if (result) return result
                  }
                
                  return (dispatch, options) => {
                    throw new Error(`Invalid value of type ${typeof arg} for ${name} argument when connecting component ${options.wrappedComponentName}.`)
                  }
                }
                
                function strictEqual(a, b) { return a === b }
                
           
                export function createConnect({
                  connectHOC = connectAdvanced,
                  mapStateToPropsFactories = defaultMapStateToPropsFactories,
                  mapDispatchToPropsFactories = defaultMapDispatchToPropsFactories,
                  mergePropsFactories = defaultMergePropsFactories,
                  selectorFactory = defaultSelectorFactory
                } = {}) {
                  return function connect(
                    mapStateToProps,
                    mapDispatchToProps,
                    mergeProps,
                    {
                      pure = true,
                      areStatesEqual = strictEqual,
                      areOwnPropsEqual = shallowEqual,
                      areStatePropsEqual = shallowEqual,
                      areMergedPropsEqual = shallowEqual,
                      ...extraOptions
                    } = {}
                  ) {
                    const initMapStateToProps = match(mapStateToProps, mapStateToPropsFactories, 'mapStateToProps')
                    const initMapDispatchToProps = match(mapDispatchToProps, mapDispatchToPropsFactories, 'mapDispatchToProps')
                    const initMergeProps = match(mergeProps, mergePropsFactories, 'mergeProps')
                
                    return connectHOC(selectorFactory, {
                      
                      methodName: 'connect',
                
                  
                      getDisplayName: name => `Connect(${name})`,
                
                      shouldHandleStateChanges: Boolean(mapStateToProps),
                
                      initMapStateToProps,
                      initMapDispatchToProps,
                      initMergeProps,
                      pure,
                      areStatesEqual,
                      areOwnPropsEqual,
                      areStatePropsEqual,
                      areMergedPropsEqual,
                
                      ...extraOptions
                    })
                  }
                }
                
                export default createConnect()




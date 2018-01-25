## TL;DR

## 

## Since you want to read it anyway...

As we mentioned in last chapter, When `RCTBridge` is done with initialization it will notify `RCTRootView` to call `AppRegistry.runApplication` to run our JavaScript code. This chapter will focus on how does JavaScript method get called from native code.

_RCTRootView.m_

```objectivec
- (void)runApplication:(RCTBridge *)bridge
{
  //...
  [bridge enqueueJSCall:@"AppRegistry"
                 method:@"runApplication"
                   args:@[moduleName, appParameters]
             completion:NULL];
}
```

_RCTCxxBridge.mm_

```objectivec
- (void)enqueueJSCall:(NSString *)module method:(NSString *)method args:(NSArray *)args completion:(dispatch_block_t)completion
{
  //...
  [self _runAfterLoad:^(){
    //...
    if (strongSelf->_reactInstance) {
      strongSelf->_reactInstance->callJSFunction([module UTF8String], [method UTF8String],
                                             convertIdToFollyDynamic(args ?: @[]));
      //...Completion code
    }
  }];
}
```

_Instance.cpp_

```cpp
void Instance::callJSFunction(std::string &&module, std::string &&method,
                              folly::dynamic &&params) {
  //...
  nativeToJsBridge_->callFunction(std::move(module), std::move(method),
                                  std::move(params));
}
```

_NativeToJsBridge.cpp_

```cpp
void NativeToJsBridge::callFunction(
    std::string&& module,
    std::string&& method,
    folly::dynamic&& arguments) {
  //...
  runOnExecutorQueue([module = std::move(module), method = std::move(method), arguments = std::move(arguments), systraceCookie]
    (JSExecutor* executor) {
      //...
      executor->callFunction(module, method, arguments);
    });
}
```

After three bridged call, the `JSCExecutor` finally join the party.

_JSCExecutor.cpp_

```cpp
void JSCExecutor::callFunction(const std::string& moduleId, 
                               const std::string& methodId,
                               const folly::dynamic& arguments) {
  //...
  // This weird pattern is because Value is not default constructible.
  // The lambda is inlined, so there's no overhead.
  auto result = [&] {
    JSContextLock lock(m_context);
    try {
      if (!m_callFunctionReturnResultAndFlushedQueueJS) {
        bindBridge();
      }
      return m_callFunctionReturnFlushedQueueJS->callAsFunction({
        Value(m_context, String::createExpectingAscii(m_context, moduleId)),
        Value(m_context, String::createExpectingAscii(m_context, methodId)),
        Value::fromDynamic(m_context, std::move(arguments))
      });
    } catch (...) {
      //...Throw error
    }
  }();
  callNativeModules(std::move(result));
}

void JSCExecutor::bindBridge() throw(JSException) {
  //...
  std::call_once(m_bindFlag, [this] {
    auto global = Object::getGlobalObject(m_context);
    auto batchedBridgeValue = global.getProperty("__fbBatchedBridge");
    if (batchedBridgeValue.isUndefined()) {
      auto requireBatchedBridge = global.getProperty("__fbRequireBatchedBridge");
      if (!requireBatchedBridge.isUndefined()) {
        batchedBridgeValue = requireBatchedBridge.asObject().callAsFunction({});
      }
      if (batchedBridgeValue.isUndefined()) {
        throw JSException("Could not get BatchedBridge, make sure your bundle is packaged correctly");
      }
 
    }

    auto batchedBridge = batchedBridgeValue.asObject();
    m_callFunctionReturnFlushedQueueJS = batchedBridge.getProperty("callFunctionReturnFlushedQueue").asObject();
    m_invokeCallbackAndReturnFlushedQueueJS = batchedBridge.getProperty("invokeCallbackAndReturnFlushedQueue").asObject();
    m_flushedQueueJS = batchedBridge.getProperty("flushedQueue").asObject();
    m_callFunctionReturnResultAndFlushedQueueJS = batchedBridge.getProperty("callFunctionReturnResultAndFlushedQueue").asObject();
  });
}
    
```




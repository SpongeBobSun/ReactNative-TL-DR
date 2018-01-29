## TL;DR

We will talk about how native methods get called from JavaScript in this chapter.

## Since you want to read it anyway...

If you've played with `JavaScriptCore` before \(maybe in a hybrid app\) , you might recall we can inject a native function to `JSContext` in JavaScriptCore \( read more on this link - [JSObjectMakeFunctionWithCallback](https://developer.apple.com/documentation/javascriptcore/1451336-jsobjectmakefunctionwithcallback?language=objc) \). This could be a way for JavaScript to communicate with native code. So you might think that `ReactNative` is using the same approach to bring native calls to JavaScript.

This is not quite accurate. `ReactNative` did use this approach to let JavaScript call native methods, but not all methods could be called like this. Actually `ReactNative` is centralizing all native calls from JavaScript through `NativeModule`.

To understand this, let's start with reviewing the initialization part of `JSCExecutor`.

_JSCExecutor.cpp_

```cpp
JSCExecutor::JSCExecutor(std::shared_ptr<ExecutorDelegate> delegate,
                             std::shared_ptr<MessageQueueThread> messageQueueThread,
                             const folly::dynamic& jscConfig) throw(JSException) :
  //...
  m_jscConfig(jscConfig) {
    initOnJSVMThread();

    {
      //...
      installGlobalProxy(m_context, "nativeModuleProxy",
                         exceptionWrapMethod<&JSCExecutor::getNativeModule>());
    }
  }
  //...
}
```




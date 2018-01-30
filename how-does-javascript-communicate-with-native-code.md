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

JSValueRef JSCExecutor::getNativeModule(JSObjectRef object, JSStringRef propertyName) {
  if (JSC_JSStringIsEqualToUTF8CString(m_context, propertyName, "name")) {
    return Value(m_context, String(m_context, "NativeModules"));
  }
  return m_nativeModules.getModule(m_context, propertyName);
}
```

_JSCHelpers.cpp_

```cpp
void installGlobalProxy(
    JSGlobalContextRef ctx,
    const char* name,
    JSObjectGetPropertyCallback callback) {
  JSClassDefinition proxyClassDefintion = kJSClassDefinitionEmpty;
  proxyClassDefintion.attributes |= kJSClassAttributeNoAutomaticPrototype;
  proxyClassDefintion.getProperty = callback;

  const bool isCustomJSC = isCustomJSCPtr(ctx);
  JSClassRef proxyClass = JSC_JSClassCreate(isCustomJSC, &proxyClassDefintion);
  JSObjectRef proxyObj = JSC_JSObjectMake(ctx, proxyClass, nullptr);
  JSC_JSClassRelease(isCustomJSC, proxyClass);

  Object::getGlobalObject(ctx).setProperty(name, Value(ctx, proxyObj));
}

```

During the initialization we've created a JavaScript object and injected it to `JSContext`. Also we've replaced this object's getters to our native implementation which will return a "NativeModule" object from 'm\_nativeModules'. We **won't** talk about about how does `NativeModules`  get initialized and loaded in this chapter. For now we just assume this is a map which holds native module instances and native module names.


## TL;DR

We will talk about how native methods get called from JavaScript in this chapter.

## Since you want to read it anyway...

If you've played with `JavaScriptCore` before \(maybe in a hybrid app\) , you might recall we can inject a native function to `JSContext` in JavaScriptCore \( read more on this link - [JSObjectMakeFunctionWithCallback](https://developer.apple.com/documentation/javascriptcore/1451336-jsobjectmakefunctionwithcallback?language=objc) \). This could be a way for JavaScript to communicate with native code. So you might think that `ReactNative` is using the same approach to bring native calls to JavaScript.

But this is not quite accurate. `ReactNative` did use this approach to let JavaScript call native methods, but not all methods could be called like this. Actually `ReactNative` is centralizing all native calls from JavaScript through `NativeModule`.

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
  /**
   * Bob's note:
   * Inject object to JS Context
   */
  Object::getGlobalObject(ctx).setProperty(name, Value(ctx, proxyObj));
}
```

During the initialization we've created a JavaScript object and injected it to `JSContext`. Also we've replaced this object's getters to our native implementation which will return a "NativeModule" object from 'm\_nativeModules'. For now we just assume this is a map which holds native module instances and native module names.

The `NativeModuleProxy` we just injected in JSContext will provide native module access to JavaScript and will be held in 'NativeModule.js'

_NativeModule.js_

```js
let NativeModules : {[moduleName: string]: Object} = {};
if (global.nativeModuleProxy) {
  NativeModules = global.nativeModuleProxy;
} else {
  const bridgeConfig = global.__fbBatchedBridgeConfig;
  /**
   * Bob's note:
   * This '__fbBAtchedBridgeCOnfig' will only be set in 'RCTObjcExecutor',
   * which is only used in debug mode.
   */
   //...code to generate native module
}

module.exports = NativeModules;
```

 You could use debugging tools in `Safari` to inspect native modules in `NativeModuleProxy`.

![](/assets/NativeModuleProxy.png)When JavaScript request to access a property in `NativeModuleProxy`, the native getter - `JSCExecutor::getNativeModule` will be called. So the question is - what's a `NativeModule` ?


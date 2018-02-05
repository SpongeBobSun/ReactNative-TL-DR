## TL;DR

We will talk about how native methods get called from JavaScript in this chapter.

## Since you want to read it anyway...

If you've played with `JavaScriptCore` before \(maybe in a hybrid app\) , you might recall we can inject a native function to `JSContext` in JavaScriptCore \( read more on this link - [JSObjectMakeFunctionWithCallback](https://developer.apple.com/documentation/javascriptcore/1451336-jsobjectmakefunctionwithcallback?language=objc) \). This could be a way for JavaScript to communicate with native code. So you might think that `ReactNative` is using the same approach to bring native calls to JavaScript.

But this is not quite accurate. `ReactNative` did use this approach to let JavaScript call native methods, but not all methods could be called like this. `ReactNative` is centralizing all native calls from JavaScript through `NativeModule`.

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
      /**
       * Bob's note:
       * Create & inject an object to JS Context
       */
      installGlobalProxy(m_context, "nativeModuleProxy",
                         exceptionWrapMethod<&JSCExecutor::getNativeModule>());
    }
  }
  //...
}

void JSCExecutor::initOnJSVMThread() throw(JSException) {
  //...
  /**
   * Bob's note:
   * Inject two callbacks in JS Context
   */
  installNativeHook<&JSCExecutor::nativeFlushQueueImmediate>("nativeFlushQueueImmediate");
  installNativeHook<&JSCExecutor::nativeCallSyncHook>("nativeCallSyncHook");
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

During the initialization we've created a JavaScript object and injected it to `JSContext`. Then we've replaced this object's getters to our native implementation which will return a "NativeModule" object from 'm\_nativeModules'. Those native modules are created during `RCTCxxBridge` initialization as we discussed in last chapter.

Also, there are two native callbacks injected to JS Context as well.

* `nativeFlushQueueImmediate`  will execute batched native calls
* `nativeCallSyncHook` will call native methods synchronously. 

Those are important callbacks in this chapter but we will discuss it later. First we will focus on `NativeModuleProxy` , which will provide native modules to JavaScript and will be held in 'NativeModule.js'

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

![](/assets/NativeModuleProxy.png)When JavaScript request to access a property in `NativeModuleProxy`, the native getter - `JSCExecutor::getNativeModule` will be called and return exported native methods in this module. Actually this is a tricky part so let's go through this 'native getter' again.

_JSCExecutor.cpp_

```cpp
JSValueRef JSCExecutor::getNativeModule(JSObjectRef object, JSStringRef propertyName) {
  //...
  return m_nativeModules.getModule(m_context, propertyName);
}
```

The 'm\_nativeModules' here is **not** a `ModuleRegistry` but a `JSCNativeModules` which is a wrapper for module registry.

_JSCNativeWrapper.cpp_

```cpp
JSValueRef JSCNativeModules::getModule(JSContextRef context, JSStringRef jsName) {
  //...Null check

  std::string moduleName = String::ref(context, jsName).str();

  const auto it = m_objects.find(moduleName);
  if (it != m_objects.end()) {
    return static_cast<JSObjectRef>(it->second);
  }

  auto module = createModule(moduleName, context);
  if (!module.hasValue()) {
    // Allow lookup to continue in the objects own properties, which allows for overrides of NativeModules
    return nullptr;
  }

  // Protect since we'll be holding on to this value, even though JS may not
  module->makeProtected();

  auto result = m_objects.emplace(std::move(moduleName), std::move(*module)).first;
  return static_cast<JSObjectRef>(result->second);
}

folly::Optional<Object> JSCNativeModules::createModule(const std::string& name, JSContextRef context) {
  //...log

  if (!m_genNativeModuleJS) {
    auto global = Object::getGlobalObject(context);
    m_genNativeModuleJS = global.getProperty("__fbGenNativeModule").asObject();
    m_genNativeModuleJS->makeProtected();
  }

  auto result = m_moduleRegistry->getConfig(name);
  //...Null check

  Value moduleInfo = m_genNativeModuleJS->callAsFunction({
    Value::fromDynamic(context, result->config),
    Value::makeNumber(context, result->index)
  });

  //...Null check

  folly::Optional<Object> module(moduleInfo.asObject().getProperty("module").asObject());

  //...log
  return module;
}
```

This will inject native module object to JavaScript lazily by calling `genModule` in `NativeModules.js`.

_NativeModules.js_

```js
function genModule(config: ?ModuleConfig, moduleID: number): 
?{name: string, module?: Object} {
  //...null check

  const [moduleName, constants, methods, promiseMethods, syncMethods] = config;
  //...module name assert. Not allowing start with 'RCT'.

  if (!constants && !methods) {
    // Module contents will be filled in lazily later
    return { name: moduleName };
  }

  const module = {};
  methods && methods.forEach((methodName, methodID) => {
    //...Validate method types
    module[methodName] = genMethod(moduleID, methodID, methodType);
  });
  Object.assign(module, constants);

  //...debug

  return { name: moduleName, module };
}

// export this method as a global so we can call it from native
global.__fbGenNativeModule = genModule;
```

The generate module part is pretty simple. It just iterate through methods and save it to the module object. When we generate methods for module we will meet the key part of this chapter.

_NativeModules.js_

```js
function genMethod(moduleID: number, methodID: number, type: MethodType) {
  let fn = null;
  if (type === 'promise') {
    fn = function(...args: Array<any>) {
      return new Promise((resolve, reject) => {
        BatchedBridge.enqueueNativeCall(moduleID, methodID, args,
          (data) => resolve(data),
          (errorData) => reject(createErrorFromErrorData(errorData)));
      });
    };
  } else if (type === 'sync') {
    fn = function(...args: Array<any>) {
      //...debug code
      return global.nativeCallSyncHook(moduleID, methodID, args);
    };
  } else {
    fn = function(...args: Array<any>) {
      const lastArg = args.length > 0 ? args[args.length - 1] : null;
      const secondLastArg = args.length > 1 ? args[args.length - 2] : null;
      const hasSuccessCallback = typeof lastArg === 'function';
      const hasErrorCallback = typeof secondLastArg === 'function';
      hasErrorCallback && invariant(
        hasSuccessCallback,
        'Cannot have a non-function arg after a function arg.'
      );
      const onSuccess = hasSuccessCallback ? lastArg : null;
      const onFail = hasErrorCallback ? secondLastArg : null;
      const callbackCount = hasSuccessCallback + hasErrorCallback;
      args = args.slice(0, args.length - callbackCount);
      BatchedBridge.enqueueNativeCall(moduleID, methodID, args, onFail, onSuccess);
    };
  }
  fn.type = type;
  return fn;
}
```

There are two ways to call a native method from JavaScript as above code block shows us.

* global.nativeCallSyncHook
* BatchedBridge.enqueueNativeCall

`nativeCallSyncHook` will be used when you export your module methods using `RCT_EXPORT_BLOCKING_SYNCHRONOUS_METHOD` . This is seldom used and may cause performance problem - your module method will running on the JavaScript thread instead of a separate dispatch queue. We will not discuss more about it but we've learned one thing from it: you can make synchronous native module methods. This may come handy some day but use it carefully.

A more common and recommended way to export module methods is using `RCT_EXPORT_METHOD`. This will lead us to the second way of calling native methods from JavaScript: `enqueueNativeCall`.

_MessageQueue.js_

```js
enqueueNativeCall(
    moduleID: number,
    methodID: number,
    params: any[],
    onFail: ?Function,
    onSucc: ?Function,
  ) {
    if (onFail || onSucc) {
      //...debug code
      // Encode callIDs into pairs of callback identifiers by shifting left and using the rightmost bit
      // to indicate fail (0) or success (1)
      // eslint-disable-next-line no-bitwise
      onFail && params.push(this._callID << 1);
      // eslint-disable-next-line no-bitwise
      onSucc && params.push((this._callID << 1) | 1);
      this._successCallbacks[this._callID] = onSucc;
      this._failureCallbacks[this._callID] = onFail;
    }

    //...debug code
    this._callID++;

    this._queue[MODULE_IDS].push(moduleID);
    this._queue[METHOD_IDS].push(methodID);

    //...debug code
    this._queue[PARAMS].push(params);

    const now = new Date().getTime();
    if (
      global.nativeFlushQueueImmediate &&
      (now - this._lastFlush >= MIN_TIME_BETWEEN_FLUSHES_MS ||
        this._inCall === 0)
    ) {
      var queue = this._queue;
      this._queue = [[], [], [], this._callID];
      this._lastFlush = now;
      global.nativeFlushQueueImmediate(queue);
    }
    //...debug & spy code
  }
```

This function will first add informations for a native call \( module id, method id & parameters \) to a 'queue'. When the time is right it will use `global.nativeFlushQueueImmediate` to trigger actual function calls in native. It will also save success & failure callbacks to a global array with current call id. Question is - when does the `nativeFlushQueueImmediate` get triggered?

First condition is this callback has to be exist. That means the native bridges must be initialized properly.

Then we have a timing check - how long has it  been since the last flush? Is it longer than some throttle we've set \(MIN\_TIME\_BETWEEN\_FLUSHES\_MS = 5ms\)? Or do we still have some pending JavaScript function calls from native? **If the last flush is 5ms ago or we don't have any pending JavaScript calls, the native call queue will be flushed. **

The reason why we're checking pending calls is when we finishing calling some JavaScript functions from native, it will flush queue for us. We've mentioned this in previous [chapter](./how-does-native-code-communicate-with-javascript).


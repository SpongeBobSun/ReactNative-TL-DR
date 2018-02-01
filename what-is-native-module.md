## TL;DR

We will talk about how does a `NativeModule` get initialized and how to prepare them for JavaScript in this chapter. If you want to know how does native method get called from JavaScript you should go to next chapter.

Below chart should cover contents in this chapter.



## Since you want to read it anyway....

We've mentioned `NativeModule` several times in previous chapters and now is the time we talk about it.

`NativeModule` are modules written in native code as it implies from its name. So `NativeModule` can provide native APIs that can't be accessed from pure JavaScript. There are several native modules shipped with ReactNative such as `UIManager`, `AsyncStorage`. Also you may already written some native modules of your own. If you haven't you can read [this](https://facebook.github.io/react-native/docs/native-modules-ios.html) article for more information.

If you have already using native modules in your project \( or you've read the link above \) you should already know in order to use a Objective-C class as native module you need:

* Implementing the `RCTBridgeModule` in your class

* Include `RCT_EXPORT_MODULE()` macro in your class.

* Declare method for JavaScript using `RCT_EXPORT_METHOD` macro.

`RCTBridgeModule` is a protocol which provides interface needed to register a bridge module. It also contains some useful macros and callback defines in its header - such as the two macros we mentioned above.

_RCTBridgeModule.h_

```objectivec
@protocol RCTBridgeModule <NSObject>
//...Comments
#define RCT_EXPORT_MODULE(js_name) \
RCT_EXTERN void RCTRegisterModule(Class); \
+ (NSString *)moduleName { return @#js_name; } \
+ (void)load { RCTRegisterModule(self); }

// Implemented by RCT_EXPORT_MODULE
+ (NSString *)moduleName;

@optional
//...optional methods
@end
```

The module register part is in the `RCT_EXPORT_MODULE()` macro which will call `RCTRegisterModule` .

_RCTBridge.m_

```objectivec
static NSMutableArray<Class> *RCTModuleClasses;
NSArray<Class> *RCTGetModuleClasses(void)
{
  return RCTModuleClasses;
}

/**
 * Register the given class as a bridge module. All modules must be registered
 * prior to the first bridge initialization.
 */
void RCTRegisterModule(Class);
void RCTRegisterModule(Class moduleClass)
{
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    RCTModuleClasses = [NSMutableArray new];
  });

  RCTAssert([moduleClass conformsToProtocol:@protocol(RCTBridgeModule)],
            @"%@ does not conform to the RCTBridgeModule protocol",
            moduleClass);

  // Register module
  [RCTModuleClasses addObject:moduleClass];
}
```

When we declare our module using `RCT_EXPORT_MODULE` , we will add our class to this 'RCTModuleClasses' array. As we mentioned in Chapter 1 there is a function call in the `start` method of `RCTCxxBridge` which will initialize native modules. And that's where those registered module classes come to play.

_RCTCxxBridge.mm_

```objectivec
  -(void)start {
    //...
    [self _initModules:RCTGetModuleClasses() withDispatchGroup:prepareBridge lazilyDiscovered:NO];
    //...
  }

- (void)_initModules:(NSArray<id<RCTBridgeModule>> *)modules
   withDispatchGroup:(dispatch_group_t)dispatchGroup
    lazilyDiscovered:(BOOL)lazilyDiscovered
{
  //...Assertion

  // Set up moduleData for automatically-exported modules
  NSArray<RCTModuleData *> *moduleDataById = [self registerModulesForClasses:modules];

#ifdef RCT_DEBUG
  if (lazilyDiscovered) {
    //...debug code
  }
  else
#endif
  {
    //...

    for (RCTModuleData *moduleData in _moduleDataByID) {
      if (moduleData.hasInstance && (!moduleData.requiresMainQueueSetup || RCTIsMainQueue())) {
        (void)[moduleData instance];
      }
    }
    _moduleSetupComplete = YES;
    [self _prepareModulesWithDispatchGroup:dispatchGroup];
  }
  //...
}

- (NSArray<RCTModuleData *> *)registerModulesForClasses:(NSArray<Class> *)moduleClasses
{

  NSMutableArray<RCTModuleData *> *moduleDataByID = [NSMutableArray arrayWithCapacity:moduleClasses.count];
  for (Class moduleClass in moduleClasses) {
    NSString *moduleName = RCTBridgeModuleNameForClass(moduleClass);

    /**
     * Bob's note:
     * Don't initialize old JS executor class
     */

    // Don't initialize the old executor in the new bridge.
    // TODO mhorowitz #10487027: after D3175632 lands, we won't need
    // this, because it won't be eagerly initialized.
    if ([moduleName isEqualToString:@"RCTJSCExecutor"]) {
      continue;
    }

    //... Check for module name collisions, 
    //... throw exception if module with specified name already exists
    RCTModuleData *moduleData = _moduleDataByName[moduleName];

    // Instantiate moduleData
    moduleData = [[RCTModuleData alloc] initWithModuleClass:moduleClass bridge:self];

    _moduleDataByName[moduleName] = moduleData;
    [_moduleClassesByID addObject:moduleClass];
    [moduleDataByID addObject:moduleData];
  }
  [_moduleDataByID addObjectsFromArray:moduleDataByID];

  return moduleDataByID;
}
```

Several things happened during this initialization part:

* Collect all registered module classes.
* Generate registered module names.
* Create `RCTModuleData` .
* Create module instance by calling `[RCTModuleData instance]`.
* Save `RCTModuleData` .

The actual instance of our native module is created in `RCTModuleData`.

_RCTModuleData.mm - creation of native module instance._

```objectivec
- (instancetype)initWithModuleClass:(Class)moduleClass
                             bridge:(RCTBridge *)bridge
{
  return [self initWithModuleClass:moduleClass
                    moduleProvider:^id<RCTBridgeModule>{ return [moduleClass new]; }
                            bridge:bridge];
}
- (instancetype)initWithModuleClass:(Class)moduleClass
                     moduleProvider:(RCTBridgeModuleProvider)moduleProvider
                             bridge:(RCTBridge *)bridge
{
  if (self = [super init]) {
    _bridge = bridge;
    _moduleClass = moduleClass;
    _moduleProvider = [moduleProvider copy];
    [self setUp];
  }
  return self;
}
- (id<RCTBridgeModule>)instance
{
  if (!_setupComplete) {
      //...
      if (_requiresMainQueueSetup) {
      //...

      RCTUnsafeExecuteOnMainQueueSync(^{
        [self setUpInstanceAndBridge];
      });
    } else {
      [self setUpInstanceAndBridge];
    }
  }
  return _instance;
}

- (void)setUpInstanceAndBridge
{
  //...
  {
    std::unique_lock<std::mutex> lock(_instanceLock);

    if (!_setupComplete && _bridge.valid) {
      if (!_instance) {
        //...
        /**
         * Bob's note:
         * Create our native module instance.
         */
        _instance = _moduleProvider ? _moduleProvider() : nil;
        //...
        //...nil check for instance
      }

      //...

      // Bridge must be set before methodQueue is set up, as methodQueue
      // initialization requires it (View Managers get their queue by calling
      // self.bridge.uiManager.methodQueue)
      [self setBridgeForInstance];
    }

    [self setUpMethodQueue];
  }
  //...finish setup
}
```

`RCTModuleData` is a rather important wrapper for our native modules. Not only it will build our module instance using registered classes, but also it will create a new method queue for per module.

_RCTModuleData.mm - Create method queue_

```objectivec
- (void)setUpMethodQueue
{
  if (_instance && !_methodQueue && _bridge.valid) {
    //...
    BOOL implementsMethodQueue = [_instance respondsToSelector:@selector(methodQueue)];
    if (implementsMethodQueue && _bridge.valid) {
      /**
       * Bob's note:
       * Using specified dispatch queue for method in module.
       */
      _methodQueue = _instance.methodQueue;
    }
    if (!_methodQueue && _bridge.valid) {
      // Create new queue (store queueName, as it isn't retained by dispatch_queue)
      _queueName = [NSString stringWithFormat:@"com.facebook.react.%@Queue", self.name];
      _methodQueue = dispatch_queue_create(_queueName.UTF8String, DISPATCH_QUEUE_SERIAL);

      // assign it to the module
      if (implementsMethodQueue) {
        @try {
          [(id)_instance setValue:_methodQueue forKey:@"methodQueue"];
        }
        @catch (NSException *exception) {
          //...
        }
      }
    }
  //...
  }
}
```

Also we can specify a dispatch queue for our native module. We can get a important conclusion - `ReactNative`_ will use a new dispatch queue for native modules on **iOS**_. This is very different on Android version. You can read more about Android version performance on `ReactNative`' s doc. So if you have some tasks on JavaScript which is very time consuming, you can consider to move them in a native module and execute them in a new JS context.

Another reason why `RCTModuleData` is an important wrapper is - it will generate a methods list. This list will be used to find methods for JavaScript.

_RCTModuleData.mm - Generate methods list_

```objectivec
- (NSArray<id<RCTBridgeMethod>> *)methods
{
  if (!_methods) {
    NSMutableArray<id<RCTBridgeMethod>> *moduleMethods = [NSMutableArray new];

    if ([_moduleClass instancesRespondToSelector:@selector(methodsToExport)]) {
      [moduleMethods addObjectsFromArray:[self.instance methodsToExport]];
    }

    unsigned int methodCount;
    Class cls = _moduleClass;
    while (cls && cls != [NSObject class] && cls != [NSProxy class]) {
      Method *methods = class_copyMethodList(object_getClass(cls), &methodCount);

      for (unsigned int i = 0; i < methodCount; i++) {
        Method method = methods[i];
        SEL selector = method_getName(method);
        if ([NSStringFromSelector(selector) hasPrefix:@"__rct_export__"]) {
          IMP imp = method_getImplementation(method);
          auto exportedMethod = ((const RCTMethodInfo *(*)(id, SEL))imp)(_moduleClass, selector);
          id<RCTBridgeMethod> moduleMethod = [[RCTModuleMethod alloc] initWithExportedMethod:exportedMethod
                                                                                 moduleClass:_moduleClass];
          [moduleMethods addObject:moduleMethod];
        }
      }

      free(methods);
      cls = class_getSuperclass(cls);
    }

    _methods = [moduleMethods copy];
  }
  return _methods;
}
```

When we export a method using `RCT_EXPORT_METHOD`, it will automatically add a '\_\_rct\_export\_\_' prefix on our method name. And `RCTModuleData` is going to use this prefix to find exported method in our class. Then it will wrap our method using `RCTModuleMethod` and save it to a class global array. We will talk about this `RCTModuleMethod` later.

Now we have saved our native module instances \( wrapped by RCTModuleData \) , let's continue the initialize part of `RCTCxxBridge`.

_RCTCxxBridge.mm_

```objectivec
- (void)_initializeBridge:(std::shared_ptr<JSExecutorFactory>)executorFactory {
  //...
  _reactInstance->initializeBridge(
      std::make_unique<RCTInstanceCallback>(self),
      executorFactory,
      _jsMessageThread,

      [self _buildModuleRegistry]);
  //...
}

- (std::shared_ptr<ModuleRegistry>)_buildModuleRegistry
{
  //...
  auto registry = std::make_shared<ModuleRegistry>(
         createNativeModules(_moduleDataByID, self, _reactInstance),
         moduleNotFoundCallback);
  //...
  return registry;
}
```

_RCTCxxUtils.mm - createNativeModules_

```cpp
std::vector<std::unique_ptr<NativeModule>> createNativeModules(
  NSArray<RCTModuleData *> *modules, 
  RCTBridge *bridge, 
  const std::shared_ptr<Instance> &instance)
{
  std::vector<std::unique_ptr<NativeModule>> nativeModules;
  for (RCTModuleData *moduleData in modules) {
    if ([moduleData.moduleClass isSubclassOfClass:[RCTCxxModule class]]) {
      nativeModules.emplace_back(std::make_unique<CxxNativeModule>(
        instance,
        [moduleData.name UTF8String],
        [moduleData] { return [(RCTCxxModule *)(moduleData.instance) createModule]; },
        std::make_shared<DispatchMessageQueueThread>(moduleData)));
    } else {
      /**
       * Create another wrapper for our native module
       */
      nativeModules.emplace_back(std::make_unique<RCTNativeModule>(bridge, moduleData));
    }
  }
  return nativeModules;
}
```

This will build a `ModuleRegistry`  by creating another wrapper \( `RCTNativeModule` \) for native modules using `createNativeModules`. Then `RCTCxxBridge` will use it to initialize 'reactInstance', which will send it all the way down to `JsToNativeBridge`. `JsToNativeBridge` will handle native function calls from JavaScript code.

_NativeToJsBridge.cpp - Class JsToNativeBridge_

```cpp
void callNativeModules(
    JSExecutor& executor, folly::dynamic&& calls, bool isEndOfBatch) override {
  //...
  for (auto& call : parseMethodCalls(std::move(calls))) {
    m_registry->callNativeMethod(call.moduleId, call.methodId, std::move(call.arguments), call.callId);
  }
  //...
}
```

_ModuleRegistry.cpp - Dispatch calls to RCTNativeModule_

```cpp
void ModuleRegistry::callNativeMethod(
  unsigned int moduleId, 
  unsigned int methodId, 
  folly::dynamic&& params, 
  int callId) {
  //...module id validation
  modules_[moduleId]->invoke(methodId, std::move(params), callId);
}
```

We will talk about how JavaScript calling native methods in next chapter. Now we should assume all native calls from JavaScript will be handled by `JsToNativeBridge`, then it will be dispatched by `ModuleRegistry` using 'module id' and 'method id' to `RCTNativeModule`. So let's take a close look at `RCTNativeModule`.

_RCTNativeModule.mm_

```objectivec
void RCTNativeModule::invoke(unsigned int methodId, folly::dynamic &&params, int callId) {
  //...weakify variables
  dispatch_block_t block = [weakBridge, weakModuleData, methodId, params=std::move(params), callId] {
    //...
    invokeInner(weakBridge, weakModuleData, methodId, std::move(params));
  };

  /**
   * Bob's note:
   * As we mentioned before, 
   * method will be dispatched on module's queue.
   */
  dispatch_queue_t queue = m_moduleData.methodQueue;
  if (queue == RCTJSThread) {
    block();
  } else if (queue) {
    dispatch_async(queue, block);
  }
}

static MethodCallResult invokeInner(
  RCTBridge *bridge, 
  RCTModuleData *moduleData, 
  unsigned int methodId, 
  const folly::dynamic &params) {
  //...nil check for params

  id<RCTBridgeMethod> method = moduleData.methods[methodId];
  //...nil check for method

  NSArray *objcParams = convertFollyDynamicToId(params);
  @try {
    /**
     * Bob's note:
     * Use actual native module instance for invocation.
     */
    id result = [method invokeWithBridge:bridge module:moduleData.instance arguments:objcParams];
    return convertIdToFollyDynamic(result);
  }
  @catch (NSException *exception) {
    // Pass on JS exceptions
    //...
  }
  return folly::none;
}
```

The actual method invoke is happened in `RCTModuleMethod`. And some conversion is needed before & after method invoke. Also we are using the actual native module instance rather than those wrapped instance.

_RCTModuleMethod.mm - invoke method_

```objectivec
- (id)invokeWithBridge:(RCTBridge *)bridge
                module:(id)module
             arguments:(NSArray *)arguments
{
  if (_argumentBlocks == nil) {
    /**
     * Bob's note:
     * Tricky part
     */
    [self processMethodSignature];
  }

#if RCT_DEBUG
  //...debug code
#endif

  // Set arguments
  NSUInteger index = 0;
  for (id json in arguments) {
    RCTArgumentBlock block = _argumentBlocks[index];
    if (!block(bridge, index, RCTNilIfNull(json))) {
      // Invalid argument, abort
      RCTLogArgumentError(self, index, json, "could not be processed. Aborting method call.");
      return nil;
    }
    index++;
  }

  // Invoke method
#ifdef RCT_MAIN_THREAD_WATCH_DOG_THRESHOLD
  if (RCTIsMainQueue()) {
    CFTimeInterval start = CACurrentMediaTime();
    [_invocation invokeWithTarget:module];
    CFTimeInterval duration = CACurrentMediaTime() - start;
    if (duration > RCT_MAIN_THREAD_WATCH_DOG_THRESHOLD) {
      //...Warning about main thread blocking
    }
  } else {
    [_invocation invokeWithTarget:module];
  }
#else
  [_invocation invokeWithTarget:module];
#endif

  /**
   * Bob's note:
   * Below line doesn't make sence.
   * Shame on whoever left this.
   */
  index = 2;
  [_retainedObjects removeAllObjects];

  if (_methodInfo->isSync) {
    void *returnValue;
    [_invocation getReturnValue:&returnValue];
    return (__bridge id)returnValue;
  }
  return nil;
}
```

Looks like it's pretty straight forward - get `NSInvocation` for required method and invoke it on module object. But there is one tricky part - `processMethodSignature`. This method will generate `NSInvocation` and parameters which match the selector. I'll not paste source code of it due to it's long and a bit complicated but you can read it yourself if you are interested.

One thing to notice is - 'Promise' and 'Reject' callback calls will be throw back to JavaScript code using `[RCTCxxBridge enqueueCallBack]`. This will eventually be handled by our good old `JSCExecutor` using `JSCExecutor::invokeCallback` which using a similar approach as `JSCExecutor::callFunction`.


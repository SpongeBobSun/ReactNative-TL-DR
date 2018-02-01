## TL;DR

We will talk about how does a `NativeModule` get initialized in this chapter.

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

The module register part is in the `RCT_EXPORT_MODULE()` macro.

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

This will build a `ModuleRegistry` with whatever returned by `createNativeModules`. Then `RCTCxxBridge` will use it to initialize 'reactInstance', which will send it all the way down to `JsToNativeBridge`. `JsToNativeBridge` will handle native function calls from JavaScript code.

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

We will talk about how JavaScript calling native methods in next chapter. Now we should assume all native calls from JavaScript will be handled by `JsToNativeBridge`, then it will be dispatched by our `ModuleRegistry` using 'module id' and 'method id'.


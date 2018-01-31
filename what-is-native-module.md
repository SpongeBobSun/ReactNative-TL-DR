## TL;DR

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
    RCT_PROFILE_BEGIN_EVENT(RCTProfileTagAlways,
                            @"-[RCTCxxBridge initModulesWithDispatchGroup:] moduleData.hasInstance", nil);
    // Dispatch module init onto main thread for those modules that require it
    // For non-lazily discovered modules we run through the entire set of modules
    // that we have, otherwise some modules coming from the delegate
    // or module provider block, will not be properly instantiated.
    for (RCTModuleData *moduleData in _moduleDataByID) {
      if (moduleData.hasInstance && (!moduleData.requiresMainQueueSetup || RCTIsMainQueue())) {
        // Modules that were pre-initialized should ideally be set up before
        // bridge init has finished, otherwise the caller may try to access the
        // module directly rather than via `[bridge moduleForClass:]`, which won't
        // trigger the lazy initialization process. If the module cannot safely be
        // set up on the current thread, it will instead be async dispatched
        // to the main thread to be set up in _prepareModulesWithDispatchGroup:.
        (void)[moduleData instance];
      }
    }
    RCT_PROFILE_END_EVENT(RCTProfileTagAlways, @"");

    // From this point on, RCTDidInitializeModuleNotification notifications will
    // be sent the first time a module is accessed.
    _moduleSetupComplete = YES;
    [self _prepareModulesWithDispatchGroup:dispatchGroup];
  }
  //...Profile code
}

- (NSArray<RCTModuleData *> *)registerModulesForClasses:(NSArray<Class> *)moduleClasses
{

  NSMutableArray<RCTModuleData *> *moduleDataByID = [NSMutableArray arrayWithCapacity:moduleClasses.count];
  for (Class moduleClass in moduleClasses) {
    NSString *moduleName = RCTBridgeModuleNameForClass(moduleClass);

    // Don't initialize the old executor in the new bridge.
    // TODO mhorowitz #10487027: after D3175632 lands, we won't need
    // this, because it won't be eagerly initialized.
    if ([moduleName isEqualToString:@"RCTJSCExecutor"]) {
      continue;
    }

    // Check for module name collisions
    RCTModuleData *moduleData = _moduleDataByName[moduleName];
    if (moduleData) {
      if (moduleData.hasInstance) {
        // Existing module was preregistered, so it takes precedence
        continue;
      } else if ([moduleClass new] == nil) {
        // The new module returned nil from init, so use the old module
        continue;
      } else if ([moduleData.moduleClass new] != nil) {
        // Both modules were non-nil, so it's unclear which should take precedence
        RCTLogError(@"Attempted to register RCTBridgeModule class %@ for the "
                    "name '%@', but name was already registered by class %@",
                    moduleClass, moduleName, moduleData.moduleClass);
      }
    }

    // Instantiate moduleData
    // TODO #13258411: can we defer this until config generation?
    moduleData = [[RCTModuleData alloc] initWithModuleClass:moduleClass bridge:self];

    _moduleDataByName[moduleName] = moduleData;
    [_moduleClassesByID addObject:moduleClass];
    [moduleDataByID addObject:moduleData];
  }
  [_moduleDataByID addObjectsFromArray:moduleDataByID];

  RCT_PROFILE_END_EVENT(RCTProfileTagAlways, @"");

  return moduleDataByID;
}
```




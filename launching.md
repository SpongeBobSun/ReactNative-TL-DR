## TL;DR

This chapter will talk about the initialize part of ReactNative. We will dig in the source code from `AppDelegate.m` and go through functions, classes step by step. Contents in this chapter can be represented by below figure.

![](/assets/RN start in native.png)

## Since you want to read it anyway...

First create a simple react-native app using `react-native init` command. Open `AppDelegate` and this is where we start.

_AppDelegate.m_

```objectivec
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
  NSURL *jsCodeLocation;

#ifdef DEBUG
  jsCodeLocation = [[RCTBundleURLProvider sharedSettings] jsBundleURLForBundleRoot:@"index" fallbackResource:nil];
#else
  jsCodeLocation = [[NSBundle mainBundle] URLForResource:@"main" withExtension:@"jsbundle"];
#endif
  /**
   * Bob's note:
   * Module name is the app name here.
   */
  RCTRootView *rootView = [[RCTRootView alloc] initWithBundleURL:jsCodeLocation
                                                      moduleName:@"AppName"
                                               initialProperties:nil
                                                   launchOptions:launchOptions];
/*
 * Make key window & root view controller, etc
 * ...
 */
}
```

The magic here is `RCTRootView`. Seems like it will load our js file from a 'jsCodeLocation' and find something \( to be accurate it's a bundle \) named 'AppName' in it.

So let's dig into it.

_RCTRootView.m_

```objectivec
- (instancetype)initWithBundleURL:(NSURL *)bundleURL
                       moduleName:(NSString *)moduleName
                initialProperties:(NSDictionary *)initialProperties
                    launchOptions:(NSDictionary *)launchOptions
{
  RCTBridge *bridge = [[RCTBridge alloc] initWithBundleURL:bundleURL
                                            moduleProvider:nil
                                             launchOptions:launchOptions];

  return [self initWithBridge:bridge moduleName:moduleName initialProperties:initialProperties];
}

- (instancetype)initWithBridge:(RCTBridge *)bridge
                    moduleName:(NSString *)moduleName
             initialProperties:(NSDictionary *)initialProperties
{
  // Should call on main queue
  RCTAssertMainQueue();

  //...Assert check

  if (self = [super initWithFrame:CGRectZero]) {
    self.backgroundColor = [UIColor whiteColor];

    // ...Set varaibles & subscribe events

    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(javaScriptDidLoad:)
                                                 name:RCTJavaScriptDidLoadNotification
                                               object:_bridge];
    //...Subscribe events

    // ...TV OS Setup

    [self showLoadingView];

    // Immediately schedule the application to be started.
    // (Sometimes actual `_bridge` is already batched bridge here.)
    [self bundleFinishedLoading:([_bridge batchedBridge] ?: _bridge)];
  }
  return self;
}
```

As we can see from above code, the JS bundle is handled by `RCTBridge` \( which is a rather important class and we will talk about it later \). And we subscribed a 'load finish' notification here, which will be handled by 'javaScriptDidLoad' selector. At the end, we will call `bundleFinishedLoading` with our `RCTBridge`.

_RCTRootView.m_

```objectivec
- (void)bundleFinishedLoading:(RCTBridge *)bridge
{
  /**
   * Bob's note:
   * Bridge maybe invalid and will call this method again when JavaScript code is loaded in 'bridge'
   */
  if (!bridge.valid) {
    return;
  }

  [_contentView removeFromSuperview];
  _contentView = [[RCTRootContentView alloc] initWithFrame:self.bounds
                                                    bridge:bridge
                                                  reactTag:self.reactTag
                                            sizeFlexiblity:_sizeFlexibility];
  [self runApplication:bridge];

  //...Content view style
}

- (void)runApplication:(RCTBridge *)bridge
{
  NSString *moduleName = _moduleName ?: @"";
  NSDictionary *appParameters = @{
    @"rootTag": _contentView.reactTag,
    @"initialProps": _appProperties ?: @{},
  };

  [bridge enqueueJSCall:@"AppRegistry"
                 method:@"runApplication"
                   args:@[moduleName, appParameters]
             completion:NULL];
}
```

Here goes our first Javascript call in `runApplication`. But in current context \( App code launch\), our bridge may not be ready or valid to be accurate. That's means our first `bundleFinishedLoad` call will return immediately. `RCTRootView` will waiting for `RCTJavaScriptDidLoadNotification` and call this method again with a valid bridge.

So before we get any further in Javascript code, we should first look into the `RCTBridge` class to see how the Javascript code is executed and keep our `RCTRootView` waiting.

First we are going to look into the `RCTBridge`'s constructor, which is called on the init function of `RCTRootView` .

Long story short - after two customized initialize methods, RCTBridge will find the true bridge class and start it. ![](/assets/RCTBRidge_init_and_start.png)

The `[self.batchedBridge start]` is a rather important method. But there is a reflection check to determine which bridge should be use. This sounds weird. Why do we have different bridge classes?

_RCTBridge.m_

```objectivec
- (void)setUp
{
  //...

  Class bridgeClass = self.bridgeClass;  

  //...

  self.batchedBridge = [[bridgeClass alloc] initWithParentBridge:self];
  [self.batchedBridge start];
}

- (Class)bridgeClass
{
  // In order to facilitate switching between bridges with only build
  // file changes, this uses reflection to check which bridges are
  // available.  This is a short-term hack until RCTBatchedBridge is
  // removed.

  Class batchedBridgeClass = objc_lookUpClass("RCTBatchedBridge");
  Class cxxBridgeClass = objc_lookUpClass("RCTCxxBridge");

  Class implClass = nil;

  if ([self.delegate respondsToSelector:@selector(shouldBridgeUseCxxBridge:)]) {
    if ([self.delegate shouldBridgeUseCxxBridge:self]) {
      implClass = cxxBridgeClass;
    } else {
      implClass = batchedBridgeClass;
    }
  } else if (cxxBridgeClass != nil) {
    implClass = cxxBridgeClass;
  } else if (batchedBridgeClass != nil) {
    implClass = batchedBridgeClass;
  }

  RCTAssert(implClass != nil, @"No bridge implementation is available, giving up.");
  return implClass;
}
```

Reason is - since facebook is deprecating the old `RCTBatchedBridge` \( mentioned in the comments \).  And we didn't specify to using it in our delegate \( actually now we don't have any delegate in RCTBridge, which we should have - see my note in below code block \), so the new bridge class - `RCTCxxBridge` will be used. Interesting thing is, `RCTCxxBridge` is a sub-class of `RCTBridge`. And there is a static `RCTBridge` instance in 'RCTBridge.m'. When the real bridge initialized, it will set this static instance to itself. Sounds like it doesn't make too much sense. Let's find out what's going on here and why facebook's coder designed this.

_Methods defined in RCTBridge.h_

```objectivec
- (instancetype)initWithDelegate:(id<RCTBridgeDelegate>)delegate
                   launchOptions:(NSDictionary *)launchOptions;

/**
 * DEPRECATED: Use initWithDelegate:launchOptions: instead
 *
 * The designated initializer. This creates a new bridge on top of the specified
 * executor. The bridge should then be used for all subsequent communication
 * with the JavaScript code running in the executor. Modules will be automatically
 * instantiated using the default contructor, but you can optionally pass in an
 * array of pre-initialized module instances if they require additional init
 * parameters or configuration.
 */

 /**
  * Bob's note:
  * Hmmm....RCTRootView is still using this constructor.
  * Seems like facebook doesn't update there code in time.
  */
- (instancetype)initWithBundleURL:(NSURL *)bundleURL
                   moduleProvider:(RCTBridgeModuleListProvider)block
                    launchOptions:(NSDictionary *)launchOptions;

- (void)enqueueJSCall:(NSString *)moduleDotMethod args:(NSArray *)args;
- (void)enqueueJSCall:(NSString *)module method:(NSString *)method 
                                           args:(NSArray *)args 
                                     completion:(dispatch_block_t)completion;


- (JSValue *)callFunctionOnModule:(NSString *)module
                           method:(NSString *)method
                        arguments:(NSArray *)arguments
                            error:(NSError **)error;

- (id)moduleForName:(NSString *)moduleName;
- (id)moduleForClass:(Class)moduleClass;

- (NSArray *)modulesConformingToProtocol:(Protocol *)protocol;
- (BOOL)moduleIsInitialized:(Class)moduleClass;
```

_Methods implemented in RCTCxxBridge.mm_

![](/assets/Methods_implemented_in_rctcxxbridge.png)

_Constructor in RCTCxxBridge.mm_

```objectivec
- (instancetype)initWithParentBridge:(RCTBridge *)bridge
{
  RCTAssertParam(bridge);

  if ((self = [super initWithDelegate:bridge.delegate
                            bundleURL:bridge.bundleURL
                       moduleProvider:bridge.moduleProvider
                        launchOptions:bridge.launchOptions])) {
    //...Set Initial State
    /*
     * Bob's note:
     * Set the static instance in `RCTBridge` to self.
     */
    [RCTBridge setCurrentBridge:self];
  }
  return self;
}
```

Turns out the key methods like `-enqueueJSCall*` , `-executeApplicationScript*` is re-implemented in `RCTCxxBridge` . And all other modules like `RCTLog`, `RCTProfile` is calling `[RCTBridge currentBridge]` when they need a 'bridge'.

![](/assets/Calling_rctbridge_currentbridge.png)

Also, I've found [this commit](https://github.com/facebook/react-native/commit/b2f3a65eab1b6cad5d287c4546bc1cc6b741ea9f#diff-883359f85083d00b7266ec2acebcca9f) on GitHub, which removed `RCTBatchedBridge` from Xcode project files. So `RCTBridge` is a 'bridge' to the real bridge in use, which was one of `RCTBatchedBridge` and `RCTCxxBridge` . But `RCTBatchedBridge` is now removed, so the real bridge we need is `RCTCxxBridge` . But you can still find the implementation of `RCTBatchedBridge` if you are interested.

Ok enough chit chat let's continue to find out what's going on in the `start` method of `RCTCxxBridge`.

_RCTCxxBridge.mm_

```objectivec
- (void)start
{
  //...Send notification 'RCTJavaScriptWillStartLoadingNotification'
  // Set up the JS thread early
  _jsThread = [[NSThread alloc] initWithTarget:[self class]
                                      selector:@selector(runRunLoop)
                                        object:nil];
  _jsThread.name = RCTJSThreadName;
  _jsThread.qualityOfService = NSOperationQualityOfServiceUserInteractive;
#if RCT_DEBUG
  _jsThread.stackSize *= 2;
#endif
  [_jsThread start];

  //...Other codes we will talk about later.
}

+ (void)runRunLoop
{
  @autoreleasepool {
    // copy thread name to pthread name
    pthread_setname_np([NSThread currentThread].name.UTF8String);

    // Set up a dummy runloop source to avoid spinning
    CFRunLoopSourceContext noSpinCtx = {0, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL};
    CFRunLoopSourceRef noSpinSource = CFRunLoopSourceCreate(NULL, 0, &noSpinCtx);
    CFRunLoopAddSource(CFRunLoopGetCurrent(), noSpinSource, kCFRunLoopDefaultMode);
    CFRelease(noSpinSource);

    // run the run loop
    while (kCFRunLoopRunStopped != CFRunLoopRunInMode(kCFRunLoopDefaultMode, ((NSDate *)[NSDate distantFuture]).timeIntervalSinceReferenceDate, NO)) {
      RCTAssert(NO, @"not reached assertion"); // runloop spun. that's bad.
    }
  }
}
```

When starting a bridge, the first thing it will do is create a new thread with a customized run loop and start it. The reason why we need a customized run loop is -

* We need to keep our thread alive
* We don't want to our thread thread trapped in a dead loop

This thread just created is going to handle all Javascript dispatch, which we will talk about later. Now let's continue to the `start` method.

_RCTCxxBridge.mm_

```
- (void)start {
    //...Create new thread for javascript dispatch.
    dispatch_group_t prepareBridge = dispatch_group_create();

    [_performanceLogger markStartForTag:RCTPLNativeModuleInit];

    [self registerExtraModules];
    // Initialize all native modules that cannot be loaded lazily
    [self _initModules:RCTGetModuleClasses() withDispatchGroup:prepareBridge lazilyDiscovered:NO];

    //...
}
```

After the js thread is created, we created a dispatch group. Then we asynchronously do the initializing works using this dispatch group. Questions are -

* _**What's a 'ModuleClass'?**_
* _**Where are those 'ModuleClasses' coming from ?**_
* _**How does it get inited?**_

Let's hold those questions for now because now we are focusing how does this 'bridge' thing get running. But long story short - this is the `NativeModule` chapter and we will definitely talk about it later.

_RCTCxxBridge.mm_

```objectivec
-(void)start {
  //...Create js thread, initialized modules.

  // This doesn't really do anything.  The real work happens in initializeBridge.
  _reactInstance.reset(new Instance);

  __weak RCTCxxBridge *weakSelf = self;

  // Prepare executor factory (shared_ptr for copy into block)
  std::shared_ptr<JSExecutorFactory> executorFactory;

  /**
   * Bob's note:
   * Only debug mode will set this 'executorClass'
   */
  if (!self.executorClass) {
    /**
     * Bob's note:
     * Could use different executor factory per JSBridge
     */
    if ([self.delegate conformsToProtocol:@protocol(RCTCxxBridgeDelegate)]) {
      id<RCTCxxBridgeDelegate> cxxDelegate = (id<RCTCxxBridgeDelegate>) self.delegate;
      executorFactory = [cxxDelegate jsExecutorFactoryForBridge:self];
    }
    if (!executorFactory) {
      BOOL useCustomJSC =
        [self.delegate respondsToSelector:@selector(shouldBridgeUseCustomJSC:)] &&
        [self.delegate shouldBridgeUseCustomJSC:self];
      // We use the name of the device and the app for debugging & metrics
      NSString *deviceName = [[UIDevice currentDevice] name];
      NSString *appName = [[NSBundle mainBundle] bundleIdentifier];
      // The arg is a cache dir.  It's not used with standard JSC.
      /**
       * Bob's note:
       * Create the real executor factory.
       */
      executorFactory.reset(new JSCExecutorFactory(folly::dynamic::object
        ("OwnerIdentity", "ReactNative")
        ("AppIdentity", [(appName ?: @"unknown") UTF8String])
        ("DeviceIdentity", [(deviceName ?: @"unknown") UTF8String])
        ("UseCustomJSC", (bool)useCustomJSC)
  #if RCT_PROFILE
        ("StartSamplingProfilerOnInit", (bool)self.devSettings.startSamplingProfilerOnLaunch)
  #endif
      ));
    }
  } else {
    /**
     * Bob's note:
     * Will only be called in JS debugging mode.
     */
    id<RCTJavaScriptExecutor> objcExecutor = [self moduleForClass:self.executorClass];
    executorFactory.reset(new RCTObjcExecutorFactory(objcExecutor, ^(NSError *error) {
      if (error) {
        [weakSelf handleError:error];
      }
    }));
  }
  //...
}
```

As we can see from the above code block, we created and a 'reactInstance'. But as the comment says it doesn't perform anything for now. Then we are going to create a JS executor to executing JS code. There is a `self.executorClass` check, which will always be null if you are not running with `Debug JS Remotely` switch on. Because currently only debug mode will set `executorClass` \(RCTWebSocketExecutor\). Also, the "delegate check" indicated we could use different "js executor factory" per JSBridge. Since we currently don't have any delegate, so the Javascript executor will be a `JSCExecutorFactory` instance, which indicated our executor will be `JSCExecutor` . This is rather a important thing to notice because there are several classes implemented this interface in react.

_RCTCxxBridge.mm_

```objectivec
-(void)start {
    //...Previous blocks

    // Dispatch the instance initialization as soon as the initial module metadata has
    // been collected (see initModules)
    dispatch_group_enter(prepareBridge);
    [self ensureOnJavaScriptThread:^{
      [weakSelf _initializeBridge:executorFactory];
      dispatch_group_leave(prepareBridge);
    }];

    // Load the source asynchronously, then store it for later execution.
    dispatch_group_enter(prepareBridge);
    __block NSData *sourceCode;
    [self loadSource:^(NSError *error, RCTSource *source) {
      if (error) {
        [weakSelf handleError:error];
      }

      sourceCode = source.data;
      dispatch_group_leave(prepareBridge);
    } onProgress:^(RCTLoadingProgress *progressData) {
      //...Display loading progress.
    }];

    // Wait for both the modules and source code to have finished loading
    dispatch_group_notify(prepareBridge, dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0), ^{
      RCTCxxBridge *strongSelf = weakSelf;
      if (sourceCode && strongSelf.loading) {
        [strongSelf executeSourceCode:sourceCode sync:NO];
      }
    });
}
```

This part is quite self explanatory - we are going to execute the JS code. Three things happened here:

* [ ] Initialize bridge
* [ ] Load JS source
* [ ] Execute JS source

Let's go through them one by one.

#### Initialize Bridge

_Initialize bridge - RCTCxxBridge.mm_

```objectivec
- (void)_initializeBridge:(std::shared_ptr<JSExecutorFactory>)executorFactory
{
  //...Valid check
  /**
   * Bob's note:
   * Retain current runloop, make sure following execution is on the 'JS thread' we've created.
   */
  __weak RCTCxxBridge *weakSelf = self;
  _jsMessageThread = std::make_shared<RCTMessageThread>([NSRunLoop currentRunLoop], ^(NSError *error) {
    if (error) {
      [weakSelf handleError:error];
    }
  });

  //...Debug check
  // This can only be false if the bridge was invalidated before startup completed
  if (_reactInstance) {
    // This is async, but any calls into JS are blocked by the m_syncReady CV in Instance
    _reactInstance->initializeBridge(
      std::make_unique<RCTInstanceCallback>(self),
      executorFactory,
      _jsMessageThread,
      [self _buildModuleRegistry]);

    //...Profile code
  }

}
```

A fancy thing to notice here is - there is a `RCTMessageThread` created. This is a C++ wrapper for `NSRunLoop` and **not** a 'thread' . Code will still executed on the 'JS thread' we've created in `start` method. Then the important part comes - we need to initialize the 'React' instance. As we can see from the `initializeBridge` function, it take 4 parameters listed as below -

* A RCTInstanceCallback
* A JSExecutorFactory will be used to create an `executor`.
* A JSMessageThread will ensure initialization running on the `JSThread` we created before.
* A `ModuleRegistry`. This is where we load & initialize all native modules. We will talk about this later in the `Native Modules` chapter.

_initializeBridge - Instance.cpp_

```cpp
void Instance::initializeBridge(
    std::unique_ptr<InstanceCallback> callback,
    std::shared_ptr<JSExecutorFactory> jsef,
    std::shared_ptr<MessageQueueThread> jsQueue,
    std::shared_ptr<ModuleRegistry> moduleRegistry) {
  callback_ = std::move(callback);
  moduleRegistry_ = std::move(moduleRegistry);

  jsQueue->runOnQueueSync([this, &jsef, jsQueue]() mutable {
    nativeToJsBridge_ = folly::make_unique<NativeToJsBridge>(
        jsef.get(), moduleRegistry_, jsQueue, callback_);

    std::lock_guard<std::mutex> lock(m_syncMutex);
    m_syncReady = true;
    m_syncCV.notify_all();
  });
}

/**
 * Bob's note: 
 * Deleted all parameters in below code to make it more clear.
 */
void Instance::loadApplication(...params) {
  nativeToJsBridge_->loadApplication(...params);
}
void Instance::setSourceURL(...params) {
  nativeToJsBridge_->loadApplication(...params);
}
void Instance::callJSFunction(...params) {
  nativeToJsBridge_->callFunction(...params);
}
```

Basically this function created a `NativeToJsBridge` on JS thread. From the rest part of `Instance.cpp` we can conclude that `ReactInstance` is bridge all function calls to `NativeToJSBridge`.

_NativeToJSBridge.cpp_

```cpp
NativeToJsBridge::NativeToJsBridge(
    JSExecutorFactory* jsExecutorFactory,
    std::shared_ptr<ModuleRegistry> registry,
    std::shared_ptr<MessageQueueThread> jsQueue,
    std::shared_ptr<InstanceCallback> callback)
    : m_destroyed(std::make_shared<bool>(false))
    , m_delegate(std::make_shared<JsToNativeBridge>(registry, callback))
    , m_executor(jsExecutorFactory->createJSExecutor(m_delegate, jsQueue))
    , m_executorMessageQueueThread(std::move(jsQueue)) {}
```

In `NativeToJSBridge`, we finally created our Javascript executor. Also, there is a `JsToNativeBridge` created in the construct list.

`NativeToJSBridge` is the one who will call Javascript functions using 'JS executor'. And `JsToNativeBridge` will handle native function calls from Javascript. We will discuss more about this two bridges in other chapters. Now we must focus on the remaining initialization code of ReactNative.

At this point, all our bridges and executors are ready to go. So let's back to where we start: `[RCTCxxBridge start]`. As we listed  before, three things happened in this method -

* [x] Initialize bridge

* [ ] Load JS source

* [ ] Execute JS source

Now we have initialized our bridges, let's load some JS source.

#### Load JS source

_RCTCxxBridge.mm_

```objectivec
-(void)start {
  //...
  // Load the source asynchronously, then store it for later execution.
  dispatch_group_enter(prepareBridge);
  __block NSData *sourceCode;
  [self loadSource:^(NSError *error, RCTSource *source) {
    //...Error check
    sourceCode = source.data;
    dispatch_group_leave(prepareBridge);
  } onProgress:^(RCTLoadingProgress *progressData) {
    //...debug code to display loading progress on top of the screen.
  }];
  //...execute Javascript source code
}

- (void)loadSource:(RCTSourceLoadBlock)_onSourceLoad onProgress:(RCTSourceLoadProgressBlock)onProgress
{  
  //...Notification & performance logger setup
  RCTSourceLoadBlock onSourceLoad = ^(NSError *error, RCTSource *source) {
    //...Performance logger
    NSDictionary *userInfo = source ? @{ RCTBridgeDidDownloadScriptNotificationSourceKey: source } : nil;
    [center postNotificationName:RCTBridgeDidDownloadScriptNotification object:self->_parentBridge userInfo:userInfo];
    _onSourceLoad(error, source);
  };

  if ([self.delegate respondsToSelector:@selector(loadSourceForBridge:onProgress:onComplete:)]) {
    [self.delegate loadSourceForBridge:_parentBridge onProgress:onProgress onComplete:onSourceLoad];
  } else if ([self.delegate respondsToSelector:@selector(loadSourceForBridge:withBlock:)]) {
    [self.delegate loadSourceForBridge:_parentBridge withBlock:onSourceLoad];
  } else if (!self.bundleURL) {
    NSError *error = //...NSError with error message
    onSourceLoad(error, nil);
  } else {
    [RCTJavaScriptLoader loadBundleAtURL:self.bundleURL onProgress:onProgress onComplete:^(NSError *error, RCTSource *source) {
      //...error check
      onSourceLoad(error, source);
    }];
  }
}
```

As we mentioned before, ReactNative allow us to use different "JSBridges" for different "RCTRootViews".  So the actual loading part is handled to `RCTJavaScriptLoader`. The loading process is pretty simple, so I'll not paste any code. Basically the loader will determine where it should load the source code - disk or url. Then it will read it to a `NSData` and wrap it to a `RCTSource` instance.

So the only item leaves on our checklist is execute JS source code.

* [x] Initialize bridge

* [x] Load JS source

* [ ] Execute JS source

#### Execute JS source

Now let's focus on the Javascript execute part.

_RCTCxxBridge.mm_

```objectivec
-(void)start {
    //...initialize bridges
    //...load js source code

    // Wait for both the modules and source code to have finished loading
  dispatch_group_notify(prepareBridge, dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0), ^{
    RCTCxxBridge *strongSelf = weakSelf;
    if (sourceCode && strongSelf.loading) {
      [strongSelf executeSourceCode:sourceCode sync:NO];
    }
  });
}

- (void)executeSourceCode:(NSData *)sourceCode sync:(BOOL)sync {

  /**
   * Bob's note:
   * This callback will be called after JavaScript code is loaded to JavaScriptCode.
   * Which will fire "RCTJavaScriptDidLoadNotification" notification.
   */
  dispatch_block_t completion = ^{
    //...code removed to make it more clear for reading.
    dispatch_async(dispatch_get_main_queue(), ^{
      [[NSNotificationCenter defaultCenter]
       postNotificationName:RCTJavaScriptDidLoadNotification
       object:self->_parentBridge userInfo:@{@"bridge": self}];
      //...code removed to make it more clear for reading.
    });
  };

  if (sync) {
    [self executeApplicationScriptSync:sourceCode url:self.bundleURL];
    completion();
  } else {
    [self enqueueApplicationScript:sourceCode url:self.bundleURL onComplete:completion];
  }
  //...Debug code
}

- (void)enqueueApplicationScript:(NSData *)script
                             url:(NSURL *)url
                      onComplete:(dispatch_block_t)onComplete
{
  [self executeApplicationScript:script url:url async:YES];

  // Assumes that onComplete can be called when the next block on the JS thread is scheduled
  if (onComplete) {
    RCTAssert(_jsMessageThread != nullptr, @"Cannot invoke completion without jsMessageThread");
    /**
     * Bob's note:
     * The JavaScript execution call will be ran on the same queue.
     * So when JavaScriptCore is done for our JavaScript code, `onComplete` will be executed.
     */
    _jsMessageThread->runOnQueue(onComplete);
  }
}

- (void)executeApplicationScriptSync:(NSData *)script url:(NSURL *)url
{
  [self executeApplicationScript:script url:url async:NO];
}

- (void)executeApplicationScript:(NSData *)script
                             url:(NSURL *)url
                           async:(BOOL)async
{
  [self _tryAndHandleError:^{
    NSString *sourceUrlStr = deriveSourceURL(url);
    if (isRAMBundle(script)) {
      //...code removed to make it more clear for reading
    } else if (self->_reactInstance) {
      self->_reactInstance->loadScriptFromString(std::make_unique<NSDataBigString>(script),
                                                 sourceUrlStr.UTF8String, !async);
    } else {
      //...code removed to make it more clear for reading
    }
  }];
}
```

Notice we have a `onComplete` callback in above code block. This call back will send a notification that indicate our JavaScript code is loaded in `JavaScriptCore`. And `RCTRootView` will listen for this notification, which we will talk about later.

_Instance.cpp_

```cpp
void Instance::loadScriptFromString(std::unique_ptr<const JSBigString> string,
                                    std::string sourceURL,
                                    bool loadSynchronously) {
  if (loadSynchronously) {
    loadApplicationSync(nullptr, std::move(string), std::move(sourceURL));
  } else {
    loadApplication(nullptr, std::move(string), std::move(sourceURL));
  }
}

void Instance::loadApplication(std::unique_ptr<RAMBundleRegistry> bundleRegistry,
                               std::unique_ptr<const JSBigString> string,
                               std::string sourceURL) {
  callback_->incrementPendingJSCalls();
  nativeToJsBridge_->loadApplication(std::move(bundleRegistry), std::move(string),
                                     std::move(sourceURL));
}
```

_NativeToJsBridge.cpp_

```cpp
void NativeToJsBridge::loadApplication(
    std::unique_ptr<RAMBundleRegistry> bundleRegistry,
    std::unique_ptr<const JSBigString> startupScript,
    std::string startupScriptSourceURL) {
  runOnExecutorQueue(
      [bundleRegistryWrap=folly::makeMoveWrapper(std::move(bundleRegistry)),
       startupScript=folly::makeMoveWrapper(std::move(startupScript)),
       startupScriptSourceURL=std::move(startupScriptSourceURL)]
        (JSExecutor* executor) mutable {
    auto bundleRegistry = bundleRegistryWrap.move();
    if (bundleRegistry) {
      executor->setBundleRegistry(std::move(bundleRegistry));
    }
    executor->loadApplicationScript(std::move(*startupScript),
                                    std::move(startupScriptSourceURL));
  });
}
```

Finally the `JSCExecutor` join the party. This file is located in `ReactCommon`, which means it's shared between both iOS and Android platform.

_JSExecutor.cpp_

```cpp
void JSCExecutor::loadApplicationScript(std::unique_ptr<const JSBigString> script, 
    std::string sourceURL) {
    //...Code removed to make it more clear for reading
    evaluateScript(m_context, jsScript, jsSourceURL);
    //...Code removed to make it more clear for reading
}

JSValueRef evaluateScript(JSContextRef context, JSStringRef script, JSStringRef sourceURL) {
  JSValueRef exn, result;
  result = JSC_JSEvaluateScript(context, script, NULL, sourceURL, 0, &exn);
  if (result == nullptr) {
    throw JSException(context, exn, sourceURL);
  }
  return result;
}
```

The key part here is `JSC_JSEvaluateScript`, which is a macro defined in `jschelpers/JavaScriptCore.h`. This macro is simply bridge the function call to Apple's `JavaScriptCore`.

_jschelpers/JavaScriptCore.h_

```
#define JSC_JSEvaluateScript(...) __jsc_wrapper(JSEvaluateScript, __VA_ARGS__)
```

Now we have finished all three tasks in our `RCTCxxBridge start` checklist -

* [x] Initialize bridge

* [x] Load JS source

* [x] Execute JS source

That means all our bridges \(RCTBridge, RCTCxxBridge\) are up and running. So let's look back to `RCTRootView`, which is the root caller of `[RCTBridge init...]`.

_RCTRootView.m_

```objectivec
- (instancetype)initWithBridge:(RCTBridge *)bridge
                    moduleName:(NSString *)moduleName
             initialProperties:(NSDictionary *)initialProperties
{
    //...code removed to make it more clear for reading

    [[NSNotificationCenter defaultCenter] addObserver:self
                                         selector:@selector(javaScriptDidLoad:)
                                             name:RCTJavaScriptDidLoadNotification
                                           object:_bridge];

    //...code removed to make it more clear for reading
}

- (void)javaScriptDidLoad:(NSNotification *)notification
{
  RCTAssertMainQueue();

  RCTBridge *bridge = notification.userInfo[@"bridge"];
  if (bridge != _contentView.bridge) {
    [self bundleFinishedLoading:bridge];
  }
}

- (void)bundleFinishedLoading:(RCTBridge *)bridge
{
  //...check nil

  [_contentView removeFromSuperview];
  _contentView = [[RCTRootContentView alloc] initWithFrame:self.bounds
                                                    bridge:bridge
                                                  reactTag:self.reactTag
                                            sizeFlexiblity:_sizeFlexibility];
  [self runApplication:bridge];

  //...UI setup (sizes & colors)
}

- (void)runApplication:(RCTBridge *)bridge
{
  NSString *moduleName = _moduleName ?: @"";
  NSDictionary *appParameters = @{
    @"rootTag": _contentView.reactTag,
    @"initialProps": _appProperties ?: @{},
  };

  RCTLogInfo(@"Running application %@ (%@)", moduleName, appParameters);
  [bridge enqueueJSCall:@"AppRegistry"
                 method:@"runApplication"
                   args:@[moduleName, appParameters]
             completion:NULL];
}
```

This leads us back to where we start. We've dug into `RCTBridge` to see how it's initialized and keep our `RCTRootView` waiting for the 'JavaScriptDidLoad' notification. After our JavaScript code is loaded, it will call `AppRegistry.runApplication` in JavaScript through our `RCTBridge / RCTCxxBridge`.

We will talk about how ReactNative is initialized in JavaScript next chapter.


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


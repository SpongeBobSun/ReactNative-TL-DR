## TL;DR

## Since you want to read it anyway....

We've mentioned `NativeModule` several times in previous chapters and now is the time we talk about it.

`NativeModule` are modules written in native code as it implies from its name. So `NativeModule` can provide native APIs that can't be accessed from pure JavaScript. There are several native modules shipped with ReactNative such as `UIManager`, `AsyncStorage`. Also you may already written some native modules of your own. If you haven't you can read [this](https://facebook.github.io/react-native/docs/native-modules-ios.html) article for more information. 

If you have already using native modules in your project \( or you've read the link above \) you should already know in order to use a Objective-C class as native module you need:

* Implementing the `RCTBridgeModule` in your class

* Include `RCT_EXPORT_MODULE()` macro in your class.  

* Declare method for JavaScript using `RCT_EXPORT_METHOD` macro.




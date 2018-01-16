First create a ReactNative app using `react-native init`  or open your existing project.

If you created a app from scratch using above command, your `AppDelegate.m` should looks like this - 

_AppDelegate.m_

```objectivec
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
  NSURL *jsCodeLocation;

  jsCodeLocation = [[RCTBundleURLProvider sharedSettings] 
                      jsBundleURLForBundleRoot:@"index" 
                              fallbackResource:nil];
  
  //...code removed to make it more clear for reading
}
```

ReactNative will load use result from `RCTBundleURLProvider` as JavaScript code location. And `RCTBundleURLProvider` will provide remote url by default. This is not what we want because we want ReactNative to load local JavaScript source code. So we need to change this part to load local JS bundle.

_AppDelegate.m_

```objectivec
  NSURL *jsCodeLocation;
  /**
   * Bob's note:
   * Prefer local built bundle.
   */
  jsCodeLocation = [[NSBundle mainBundle] URLForResource:@"main" withExtension:@"jsbundle"];
  if (!jsCodeLocation) {
    jsCodeLocation = [[RCTBundleURLProvider sharedSettings] jsBundleURLForBundleRoot:@"index" fallbackResource:nil];
  }
```

Then we need to build our JS bundle by running below command in project root.

```bash
react-native bundle --dev false \
  --reset-cache \
  --platform ios \
  --entry-file index.js \
  --bundle-output './build/main.jsbundle'
```

This will build our JS code to `./build/main.jsbundle`. Add this file to project in Xcode and hit the run button, our app will load  local 


## TL;DR {#tldr}

为了后面分析代码方便，本章将会准备一个空的ReactNative工程。

* 创建一个空的ReactNative工程，或者打开一个你现有的工程。
* 修改native代码使其从本地的JS Bundle中加载JavaScript代码。
* Build 一个JS Bundle

## 来都来了...

首先用`react-native init` 这个命令来创建一个空白的ReactNative app。当然打开一个你现有的工程也是可以的。

如果你是按照上面的命令创建的新工程，你的`AppDelegate.m` 应该长成下面的样子 - 

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

ReactNative会使用`RCTBundleURLProvider` 返回的结果作为JavaScript代码的地址，而`RCTBundleURLProvider` 默认会返回Debug模式下的url。也就是说默认情况下我们的app会从JS debug server那里去拿JavaScript代码。这个并不是我们想要的行为。我们需要我们的app从bundle中拿已经打包好的JavaScript代码，这样会更贴近我们app在生产环境下的行为。

_AppDelegate.m_

```objectivec
  NSURL *jsCodeLocation;
  /**
   * Bob's note:
   * 优先加载本地的JS Bundle
   */
  jsCodeLocation = [[NSBundle mainBundle] URLForResource:@"main" withExtension:@"jsbundle"];
  if (!jsCodeLocation) {
    jsCodeLocation = [[RCTBundleURLProvider sharedSettings] 
                        jsBundleURLForBundleRoot:@"index" 
                                fallbackResource:nil];
  }
```

然后我们需要构建我们的JS Bundle，也就是通过ReactNative提供的工具将JS code打包成一个js文件。在项目的根目录下面运行这个命令就可以。

```bash
react-native bundle --dev false \
  --reset-cache \
  --platform ios \
  --entry-file index.js \
  --bundle-output './build/main.jsbundle'
```

这个命令会把我们的JS Bundle构建到`./build/main.jsbundle` 。添加这个文件到Xcode的工程里面，我们就万事具备了。


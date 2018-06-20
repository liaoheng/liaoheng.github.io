title: Android React Native入门
author: liaoheng
tags: []
categories: []
date: 2017-09-15 17:34:00
---
> 本文需要了解基本Android开发知识

## 环境

1. [JDK](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
2. [Node](https://nodejs.org/en/download/)
3. [Watchman 可选](https://facebook.github.io/watchman/docs/install.html#build-install) `Windows问题比较多，不推荐安装`
4. [Android SDK](https://developer.android.com/studio/index.html)`配置环境`

### 配置
1. 项目结构，创建相应文件

 > ├── android
 > ├── index.js
 > ├── node_modules
 > ├── package.json

2. 编辑`package.json`文件添加：

  ```json
	{
	  "name": "[项目名]",
	  "version": "0.0.1",
	  "private": true,
	  "scripts": {
		"start": "node node_modules/react-native/local-cli/cli.js start",
		"test": "jest"
	  },
	  "dependencies": {
		"react": "16.3.1",
		"react-native": "0.55.1",
		"react-navigation": "1.5.2"
	  },
	  "devDependencies": {
		"babel-jest": "22.4.3",
		"babel-preset-react-native": "4.0.0",
		"jest": "22.4.3",
		"react-test-renderer": "16.3.1"
	  },
	  "jest": {
		"preset": "react-native"
	  }
	}
  ```

  > 也可以使用命令创建官方的HelloWorld项目，然后拷贝相关配置文件，如下：

  > ```shell
  > npm install -g create-react-native-app
  > create-react-native-app AwesomeProject
  > ```

3. 安装nodejs库，在根目录中执行命令：

  ```shell
  npm install
  npm install -g react-native-cli
  ```

4. 编辑`index.js`文件添加：

  ```javascript
  import React, { Component } from 'react';
  import {
  AppRegistry,
  StyleSheet,
  Text,
  View
  } from 'react-native';

  export default class Launch extends Component {
   render() {
     return (
       <View style={styles.container}>
        <Text style={styles.welcome}>
          Welcome to React Native!
        </Text>
        <Text style={styles.instructions}>
          To get started, edit index.js
        </Text>
        <Text style={styles.instructions}>
          Double tap R on your keyboard to reload,{'\n'}
          Shake or press menu button for dev menu
        </Text>
       </View>
     );
   }
  }

  const styles = StyleSheet.create({
   container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#F5FCFF',
   },
   welcome: {
    fontSize: 20,
    textAlign: 'center',
    margin: 10,
  },
  instructions: {
    textAlign: 'center',
    color: '#333333',
    marginBottom: 5,
  },
  });

  AppRegistry.registerComponent('ReactTest', () => Launch);
  ```

5. 在`android目录`中创建app工程项目，然后在项目中集成React Native

  - [可选] 在`settings.gradle`文件中添加：

    ```properties
    rootProject.name = '[项目名]'
    ```

  - 在`gradle.properties`文件中添加：

    ```properties
    android.useDeprecatedNdk=true
    ```

  - 在根目录`build.gradle`文件中添加：

    ```groovy
    allprojects {
       repositories {
          maven {
            url "$rootDir/../node_modules/react-native/android"
          }
       }
     }
    ```

  - 在app目录`build.gradle`文件中添加：

    ```groovy
	apply from: "../../node_modules/react-native/react.gradle"
    android {
    defaultConfig {
       ndk {
           abiFilters "armeabi-v7a", "x86"
       }
    }
    splits {
        abi {
           reset()
           enable false
           universalApk false  // If true, also generate a universal APK
           include "armeabi-v7a", "x86"
         }
      }
    }
    dependencies {
      compile "com.facebook.react:react-native:+"
    }
    ```

  - 在app项目中创建`MApplication.java`

    ```java
    public class MApplication extends Application implements ReactApplication{

        private final ReactNativeHost mReactNativeHost = new ReactNativeHost(this){

			@Override
			public boolean getUseDeveloperSupport() {
				return BuildConfig.DEBUG;
			}

			@Override
			protected List<ReactPackage> getPackages() {
				return Arrays.<ReactPackage>asList(new MainReactPackage());
			}

			@Override
			protected String getJSMainModuleName() {
				return "index";
			}
		};

		@Override
		public ReactNativeHost getReactNativeHost() {
			return mReactNativeHost;
		}
    }
    ```

  - 在app项目中创建`MainActivity.java`

    ```java
    public class MainActivity extends ReactActivity {

     @Nullable
     @Override
     protected String getMainComponentName() {
        return "ReactTest";//与index.js中registerComponent对应
     }
    }
    ```

  - 修改app项目中`AndroidManifest.xml`文件

    ```xml
    <application
        android:name=".MApplication">
           <activity android:name=".MainActivity">
              <intent-filter>
                 <action android:name="android.intent.action.MAIN" />

                 <category android:name="android.intent.category.LAUNCHER" />
              </intent-filter>
           </activity>
           <activity android:name="com.facebook.react.devsupport.DevSettingsActivity" />
    </application>
    ```

### 运行

1. 通过命令行直接运行`开发服务`和app:

  ```shell
  react-native run-android
  ```

  > 如果使用Windows系统请使用方法1，方法2会在运行app时出现异常：`java.io.IOException: Could not delete path`

2. 通过命令运行`开发服务`，然后手动运行app

  ```shell
  npm start
  #也可使用： react-native start
  ```

  `开发服务`正常启动后如下，使用中不能关闭（如果安装Watchman，`开发服务`会在后台运行可以关闭当前窗口）：

  ```shell
   \> ReactTest@0.0.1 start x:\xxxx\React
   \> node node_modules/react-native/local-cli/cli.js start
   
   Scanning 564 folders for symlinks in x:\xxxx\React\node_modules (43ms)
   ┌────────────────────────────────────────────────────────────────────────────  ┐
   │  Running packager on port 8081.                                              │
   │                                                                              │
   │  Keep this packager running while developing on any JS projects. Feel        │
   │  free to close this tab and run your own packager instance if you            │
   │  prefer.                                                                    │
   │                                                                              │
   │  https://github.com/facebook/react-native                                    │
   │                                                                              │
   └────────────────────────────────────────────────────────────────────────────┘
   Looking for JS files in
       X:\xxxx\React
       
   React packager ready.
   
   Loading dependency graph, done.
   ```

### 提示
- 默认在根目录中执行以上命令
- 出现手机或虚拟机无法连接电脑时可使用`adb reverse tcp:8081 tcp:8081`
- debug模式默认使用8081端口,如果本地以被占用，通过`react-native start --port 9988`修改端口。
- 注意需要权限：`<uses-permission android:name="android.permission.INTERNET" />`
- 参考：
  - http://facebook.github.io/react-native/docs/integration-with-existing-apps.html
  - http://facebook.github.io/react-native/docs/getting-started.html

### Demo:[ReactTest](https://github.com/liaoheng/ReactTest)
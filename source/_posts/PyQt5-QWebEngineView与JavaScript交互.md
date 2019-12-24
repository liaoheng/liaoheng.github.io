---
title: PyQt5中QWebEngineView与JavaScript交互
date: 2019-12-23 17:16:38
tags:
---

### 准备工作

#### 开发环境

- Python 3.8.1
- Windows 10

#### 安装依赖

```shell
pip install PyQt5
pip install PyQtWebEngine
```

### Python端

1. 使用`QWebChannel`的`registerObject（"JsBridge名","JsBridge"）`方法注册回调

   - `JsBridge名`：在JavaScript中调用时使用的对象名称
   - `JsBridge`：被JavaScript调用的Python对象

2. `JsBridge` 对象

   - 入参

    ```python
    @QtCore.pyqtSlot(str)
    def log(self, message):
        print(message)
    ```

   - 出参

    ```python
    @QtCore.pyqtSlot(result=str)
    def getName(self):
        return "hello"
    ```

   - 出入参

    ```python
    @QtCore.pyqtSlot(str, result=str)
    def test(self, message):
        print(message)
        return "ok"
    ```

### JavaScript端

1. 在Html的`<head>`中添加

    ```html
    <script src='qrc:///qtwebchannel/qwebchannel.js'></script>
    ```

2. 调用

   ```javascript
   new QWebChannel(qt.webChannelTransport, function(channel) {
        channel.objects.pythonBridge.test("hello",function(arg) {
            console.log("Python message: " + arg);
            alert(arg);
        });
    });
   ```

3. 调试(Chrome DevTools)
   1. 配置环境变量：`QTWEBENGINE_REMOTE_DEBUGGING = port`
   2. 使用Chromium内核的浏览器打开地址：`http://127.0.0.1:port`
   3. 使用`PyCharm`中可以在运行配置(Run/Debug Configurations)中的Environment variables中添加环境变量,用`;`号分隔，然后可以直接运行。

### Demo

#### Python

1. **JsBridge**

   ```python
   from PyQt5 import QtCore

   class JsBridge(QtCore.QObject):
       @QtCore.pyqtSlot(str, result=str)
       def test(self, message):
           print(message)
           return "ok"
   ```

2. ##### Application

   ```python
   from PyQt5 import QtCore
   from PyQt5 import QtWebEngineWidgets
   from PyQt5.QtCore import QDir
   from PyQt5.QtWebChannel import QWebChannel
   from PyQt5.QtWebEngineWidgets import QWebEngineView
   from PyQt5.QtWidgets import *
   
   class TestWindow(QMainWindow):
       def __init__(self):
           super().__init__()
           self.webView = QWebEngineView()
           self.webView.settings().setAttribute(
               QtWebEngineWidgets.QWebEngineSettings.JavascriptEnabled, True)
   
           channel = QWebChannel(self.webView.page())
           self.webView.page().setWebChannel(channel)
           self.python_bridge = JsBridge(None)
           channel.registerObject("pythonBridge", self.python_bridge)
           layout = QVBoxLayout()
           layout.addWidget(self.webView)
           widget = QWidget()
           widget.setLayout(layout)
           self.setCentralWidget(widget)
   
           self.resize(900, 600)
           self.setWindowTitle('Test')
           qr = self.frameGeometry()
           cp = QDesktopWidget().availableGeometry().center()
           qr.moveCenter(cp)
           self.move(qr.topLeft())
           self.show()
           html_path = QtCore.QUrl.fromLocalFile(QDir.currentPath() + "/assets/index.html")
           self.webView.load(html_path)
   
   if __name__ == '__main__':
       app = QApplication(sys.argv)
       m = TestWindow()
       sys.exit(app.exec_())
   ```

#### JavaScript

> ##### index.html

```html
<!DOCTYPE html>
<html lang="zh">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
    <title>Test</title>
    <script src='qrc:///qtwebchannel/qwebchannel.js'></script>
    <script src="https://code.jquery.com/jquery-3.4.1.min.js"
            integrity="sha256-CSXorXvZcTkaix6Yvo6HppcZGetbYMGWSFlBw8HfCJo="
            crossorigin="anonymous"></script>
</head>
<body>
   <button id="test">test</button>
</body>
<script>
    $(document).ready(function() {
        new QWebChannel(qt.webChannelTransport, function(channel) {
            $('#test').on('click', function() {
                channel.objects.pythonBridge.test("hello",function(arg) {
                  console.log("Python message: " + arg);
                  alert(arg);
                });
            });
        });
    });
</script>
</html>
```
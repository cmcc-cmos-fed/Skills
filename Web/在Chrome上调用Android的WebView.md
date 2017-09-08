#### Step 1

在Android的代码里设置打开WebView调试模式

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
    WebView.setWebContentsDebuggingEnabled(true);
}
```

#### Step 2

电脑与Android手机USB连接，确保开发者模式打开；

打开电脑上的Chrome，输入网址：
```
chrome://inspect
```

http://kotlinlang.org/docs/reference/

@Parcelize注解用来自动生成Parcelable实现，需要在build.gradle中声明如下配置
```
android {
  ...
  androidExtensions {
    experimental = true
  }
  ...
}
```

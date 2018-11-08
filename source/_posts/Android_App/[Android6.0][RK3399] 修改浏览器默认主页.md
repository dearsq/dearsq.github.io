title: Android6.0 RK3399 修改浏览器默认主页
date: 2018-03-31 23:33:05
tags: 
- MediaTek
- Android

---

OS: Android6.0
Hardware: RK3399

## 修改默认主页
在如下文件
packages/apps/Browser/res/values/strings.xml 
 ```
<!-- The default homepage. --> 
<string name="homepage_base" translatable="false"> 
    https://www.google.com/webhp?client={CID}&amp;source=android-home</string> 
 ```

修改一个你喜（tao）欢（yan）的主页比如百度： 
```
     <!-- The default homepage. --> 
     <string name="homepage_base" translatable="false"> 
    https://www.baidu.com/</string> 
```
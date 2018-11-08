title: Android6.0 MTK6737 替换原生 Launcher
date: 2018-03-31 23:33:05
tags: 
- MediaTek
- Android

---

### 屏蔽 Launcher3 中的 category
```
<!-- category android:name="android.intent.category.HOME" -->
<!-- category android:name="android.intent.category.LAUNCHER" -->
<!-- category android:name="android.intent.category.DEFAULT" -->
```

### 在自己的 App 中添加以上三个 category
```
<category android:name="android.intent.category.HOME" />
<category android:name="android.intent.category.LAUNCHER" />
<category android:name="android.intent.category.DEFAULT" />
```

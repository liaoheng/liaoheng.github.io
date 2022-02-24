---
title: Android密度无关像素(dp)与屏幕适配
date: 2021-07-14 16:16:34
tags:
---

## 密度无关像素(density-independent pixels)
> Android 设备不仅有不同的屏幕尺寸（手机、平板电脑、电视等），而且其屏幕也有不同的像素尺寸。也就是说，有可能一部设备的屏幕为每平方英寸 160 像素，而另一部设备的屏幕在相同的空间内可以容纳 480  像素。如果您不考虑像素密度的这些差异，系统可能会缩放图片（导致图片变模糊），或者图片可能会以完全错误的尺寸显示。

如果开发者只对屏幕`分辨率`进行适配，也就是说使用`像素(px)`作为度量单位，那么在同一`分辨率`下的布局，在不同`像素密度`的屏幕中的显示完全不同。为了应对这种情况，Android引入了`密度无关像素(dp)`来作为度量单位，但**最终的渲染还是使用像素(px)为度量单位**，dp是做为一种中间值的存在。

> 密度无关像素(dp)是一个虚拟像素单位，1 dp 约等于`中密度屏幕`（160dpi；“基准”密度）上的 1 像素。对于其他每个密度，Android 会将此值转换为相应的实际像素数。

Android支持多种多样的`屏幕密度`，但是开发者不可能对每种都进行适配，于是将`屏幕密度`按一定关系进行划分，开发者只适配同一区间的`屏幕密度`成为一种比较好的方案。Android的`屏幕密度`分级区间，系统会基于不同区间自行按对应倍数进行缩放（包括布局和图片），开发者也可以基于这些区间规制自行适配，区间如下：

|密度限定符|说明|**缩放倍数**|
| ----- | ----- | ----- |
|ldpi|适用于低密度 (ldpi) 屏幕 (\~ 120dpi) 的资源。（在120dpi左右，而非固定值，不同系统也是有差距的）|0.75（px=dp\*0.75）|
|mdpi|适用于中密度 (mdpi) 屏幕 (\~ 160dpi) 的资源（这是基准密度）。默认的layout，dimens也是基于此。|1.0（px=dp)|
|hdpi|适用于高密度 (hdpi) 屏幕 (\~ 240dpi) 的资源。|1.5（px=dp\*1.5）|
|xhdpi|适用于加高 (xhdpi) 密度屏幕 (\~ 320dpi) 的资源。|2.0（px=dp\*2）|
|xxhdpi|适用于超超高密度 (xxhdpi) 屏幕 (\~ 480dpi) 的资源。|3.0（px=dp\*3）|
|xxxhdpi|适用于超超超高密度 (xxxhdpi) 屏幕 (\~ 640dpi) 的资源。|4.0（px=dp\*4）|
|nodpi|适用于所有密度的资源。这些是与密度无关的资源。无论当前屏幕的密度是多少，系统都不会缩放以此限定符标记的资源。| |
|tvdpi|适用于密度介于 mdpi 和 hdpi 之间的屏幕（约 213dpi）的资源。这不属于“主要”密度组。它主要用于电视，而大多数应用都不需要它。对于大多数应用而言，提供 mdpi 和 hdpi 资源便已足够，系统将视情况对其进行缩放。如果您发现有必要提供 tvdpi 资源，应按一个系数来确定其大小，即 1.33\*mdpi。例如，如果某张图片在 mdpi 屏幕上的大小为 100px x 100px，那么它在 tvdpi 屏幕上的大小应该为 133px x 133px。|1.33（px=dp\*1.33）|

例如，你在中密度屏幕上的一个控件或者图片，大小为 48x48 像素，那么它在其他各种密度的屏幕上的大小应该为：

* 36x36 (0.75x) - 低密度 (ldpi)
* 48x48（1.0x 基准）- 中密度 (mdpi)
* 72x72 (1.5x) - 高密度 (hdpi)
* 96x96 (2.0x) - 超高密度 (xhdpi)
* 144x144 (3.0x) - 超超高密度 (xxhdpi)
* 192x192 (4.0x) - 超超超高密度 (xxxhdpi)

## 如何使用密度无关像素进行适配屏幕
### 系统适配
* 图片文件48x48（res/drawable/ic\_icon.png）
* 布局文件（res/layout/activity\_main.xml）

```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
    <TextView
            android:layout_marginTop="@dimen/text_margin_top"
            android:layout_marginEnd="@dimen/text_margin_end"
            android:layout_marginStart="@dimen/text_margin_start"
            android:layout_marginBottom="@dimen/text_margin_bottom"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@string/app_name" />
</FrameLayout>
```
* 资源尺寸文件（res/values/dimens.xml）

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <dimen name="text_margin_top">15dp</dimen>//缩放之后的px=15*缩放倍数
    <dimen name="text_margin_end">10dp</dimen>
    <dimen name="text_margin_start">20dp</dimen>
    <dimen name="text_margin_bottom">20px</dimen>//不会缩放px=20
</resources>
```
系统会根据`屏幕密度`分级区间自行按对应倍数进行缩放，在中密度 (mdpi)下`text_margin_top`的`像素`为15，在超高密度 (xhdpi)下，`text_margin_top`的`像素`为15x2.0，`ic_icon.png`的`像素`为96x96。

### 自定义适配
系统的缩放比在一定条件下，是可以保证应用的显示正常，但是往往不一定那么理想，于是手动去设置尺寸，更为常见，就是能过将对应尺寸的资源放入对应dp文件夹来实现，如：

* 要适配xhdpi，就将同名图片放入`res/drawable-xhdp/ic_icon.png`。
* 要适配xhdpi，就将同名文件放入`res/layout-xhdpi/activity_main.xml`。
* 要适配xhdpi，就将同名文件放入`res/values-xhdpi/dimens.xml`中。

如果使用dp会在相应`屏幕密度`分级区间自行按对应倍数进行缩放，也就是说相当于覆盖了默认的`res/values/dimens.xml`中的参数，然后进入*系统适配*（按对应倍数进行缩放），px则不会进行缩放，例如(res/values-xhdpi/dimens.xml)：

```xml
  <?xml version="1.0" encoding="utf-8"?>
  <resources>
      <dimen name="text_margin_top">5dp</dimen> //缩放之后的px=10
      <dimen name="text_margin_end">5dp</dimen>
      <dimen name="text_margin_start">10px</dimen>//不会缩放px=10
      <dimen name="text_margin_bottom">15px</dimen>
  </resources>
```
当前应用如果适配的dpi不匹配设备时，系统会先找比设备dpi更大最近的以适配的dpi，然后是更小最近的，一直到mdpi也就是默认适配文件。例如：

* 当前应用适配有hdpi、xxdpi、xxxdpi，设备的dpi为xhdpi，于是系统给应用分配的是xxdpi中的参数。
* 当前应用适配有hdpi，设备的dpi为xhdpi，于是系统给应用分配的是hdpi中的参数。
* 当前应用适配有ldpi，设备的dpi为xhdpi，于是系统给应用分配的是mdpi(也就是默认值)中的参数。

### 使用代码强行设置应用显示的`屏幕密度`(dpi)
这种方式可以比如只适配了xhdpi，想在其它的dpi中也按xhdpi进行绘制。

```java
    private float sNoCompatDensity;
    private float sNoCompatScaledDensity;

    /**
     * 强制设置dpi
     *
     * @param targetDP 密度屏幕(160,240,360...)
     * @param isV      是否横屏
     */
    private void setCustomDensity(int targetDP, boolean isV) {
        DisplayMetrics appDisplayMetrics = getResources().getDisplayMetrics();
        if (sNoCompatDensity == 0) {
            sNoCompatDensity = appDisplayMetrics.density;
            sNoCompatScaledDensity = appDisplayMetrics.scaledDensity;
        }

        float targetDensity;
        if (isV) {
            targetDensity =
                    (float) appDisplayMetrics.heightPixels / (appDisplayMetrics.heightPixels / (targetDP / 160F));
        } else {
            targetDensity =
                    (float) appDisplayMetrics.widthPixels / (appDisplayMetrics.heightPixels / (targetDP / 160F));
        }
        float targetScaledDensity = targetDensity * (sNoCompatScaledDensity / sNoCompatDensity);
        int targetDensityDpi = (int) (160 * targetDensity);

        appDisplayMetrics.density = targetDensity;
        appDisplayMetrics.scaledDensity = targetScaledDensity;
        appDisplayMetrics.densityDpi = targetDensityDpi;
    }
```
### **使用最小宽度限定符(smallestWidth)进行适配**
> 屏幕的基本尺寸，由可用屏幕区域的最小尺寸指定。具体而言，设备的 smallestWidth 是屏幕可用高度和宽度的最小尺寸（您也可将其视为屏幕的“最小可能宽度”）。无论屏幕的当前方向如何，您均可使用此限定符确保应用界面的可用宽度至少为  dp。
例如，如果布局要求屏幕区域的最小尺寸始终至少为 600dp，则可使用此限定符创建布局资源 res/layout-sw600dp/。仅当可用屏幕的最小尺寸至少为 600dp（无论 600dp 表示的边是用户所认为的高度还是宽度）时，系统才会使用这些资源。最小宽度为设备的固定屏幕尺寸特征；即使屏幕方向发生变化，设备的最小宽度仍会保持不变。

该最小宽度的计算方式为：`屏幕宽度(px)/屏幕密度缩放倍数`。如你的屏幕为720x1280 xhdpi，那么720/2=360。

使用如下：`res/values-sw360dp/dimens.xml`，就是满足最小宽度(dp)为360dpi的屏幕进行适配，这种方式是分辨率+屏幕密度(dpi)，算是现在对于**多尺寸屏幕适配**比较好的方案。

### **可用宽度(高度)限定符进行适配**
> 您可能希望根据当前可用的宽度或高度来更改布局，而不是根据屏幕的最小宽度来更改布局。例如，如果您有一个双窗格布局，您可能希望在屏幕宽度至少为 600dp 时使用该布局，但屏幕宽度可能会根据设备的屏幕方向是横向还是纵向而发生变化。

使用**最小宽度限定符**的适配方式对于等比例屏幕效果比较好，如果是屏幕长宽比例较大时就不太适用，这时可以使用**可用宽度(高度)符**，这两种方案搭配几乎可以满足所有适配需求。

计算方式为与**最小宽度限定符**一样。

使用如下：`res/values-w360dp/dimens.xml`，就是满足宽度(dp)为360dpi的屏幕进行适配或者`res/values-h720dp/dimens.xml`就是满足高度(dp)为720dpi的屏幕进行适配。

## 相关其它
### dp与px的转换规则
```
px = dp * (dpi / 160)
```
`dpi`：由上表可得，就是每平方英寸可显示的像素数量，如mdpi\~240dpi。

### 设计图中的px与dp转换
在Photoshop中的\*\*像素/英寸 (ppi)\*\*与Android的`屏幕密度`(dpi)基本是一个概念，于是在设计图(320ppi)中一个控件的外边距为10px，由上面的转换规则可知`dp=px/(dpi/160)`，这里就是10/(320/160)=5，也就是在xhdpi的布局中对应控件的外边距为5dp。

### 改变设备的`屏幕密度`(dpi)
获取设备屏幕分辨率：`adb shell wm size`获取设备`屏幕密度`(dpi)：`adb shell wm density`设置`屏幕密度`(dpi) ：`adb shell wm density 320`重置`屏幕密度`(dpi) ：`adb shell wm density reset`

**注意**：虽然改变了系统的参数，但是系统判断自定义适配规制是无效的，会跳过当前dpi，但是`最小宽度限定符`是生效的。例如：

* 当前应用适配有hdpi、xxdpi、xxxdpi，给设备设置的是hdpi，于是系统给应用分配的是xxdpi中的参数。
* 当前应用适配有sw480dp，给设备(720p)设置的是hdpi，是正常的。

### 获取屏幕参数代码片段
```java
    public void getAndroiodScreenProperty() {
        WindowManager wm = (WindowManager) getSystemService(Context.WINDOW_SERVICE);
        DisplayMetrics dm = new DisplayMetrics();
        wm.getDefaultDisplay().getMetrics(dm);
        int width = dm.widthPixels;         // 屏幕宽度（像素）
        int height = dm.heightPixels;       // 屏幕高度（像素）
        float density = dm.density;         // 屏幕密度（0.75 / 1.0 / 1.5）
        int densityDpi = dm.densityDpi;     // 屏幕密度dpi（120 / 160 / 240）
        // 屏幕宽度算法:屏幕宽度（像素）/屏幕密度
        int screenWidth = (int) (width / density);  // 屏幕宽度(dp)
        int screenHeight = (int) (height / density);// 屏幕高度(dp)

        Log.d(TAG, "屏幕宽度（像素）：" + width);
        Log.d(TAG, "屏幕高度（像素）：" + height);
        Log.d(TAG, "屏幕密度（0.75 / 1.0 / 1.5）：" + density);
        Log.d(TAG, "屏幕密度dpi（120 / 160 / 240）：" + densityDpi);
        Log.d(TAG, "屏幕宽度（dp）：" + screenWidth);
        Log.d(TAG, "屏幕高度（dp）：" + screenHeight);
    }
```
## 参考
* [支持不同的屏幕尺寸](https://developer.android.com/training/multiscreen/screensizes)
* [支持不同的像素密度](https://developer.android.com/training/multiscreen/screendensities)
* [推荐Android两种屏幕适配方案 ](https://juejin.cn/post/6844903612926296072#heading-10)
* [Android获取屏幕的宽度和高度(dp)](https://blog.csdn.net/wangliblog/article/details/22501141)
* [Android屏幕values-sw适配](https://blog.csdn.net/nihaomabmt/article/details/71215507)
---
title: Android Ripple(波纹效果)
date: 2024-02-21 13:49:59
tags:
---
> 控件中的Ripple标签，即对应一个RippleDrawable，当它被设置为一个控件的background属性时，控件在按下时，即会显示水波效果

# 自定义
## 没有边界的Ripple（Ripple With No Mask）
波纹的内直径`就是`控件的长或者宽，实现内容如下，不需要任何操作即可。
```xml
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="?attr/colorControlHighlight" />
```
![](https://pic.liaoheng.me/hc-pic/2024/02/fb5e3c11b575fa8e46d0847e7d3eea49.gif)

## 有边界的Ripple（Ripple With Mask）
波纹`不超过`控件的显示范围，实现内容如下，需要增加一个@android:id/mask配置，配置中的颜色就是波纹的背景色，会与ripple颜色重叠显示。
```xml
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="?attr/colorControlHighlight">
    <item android:id="@android:id/mask">
        <color android:color="@color/white" />
    </item>
</ripple>
```
![](https://pic.liaoheng.me/hc-pic/2024/02/cb0c71e1f80ff24750032f3c05d2289c.gif)

## 用图片作边界的Ripple（Ripple With Picture Mask）
波纹`不超过`图片的显示范围，实现内容如下，需要增加一个@android:id/mask配置，配置中的图片就是波纹的背景，会与ripple颜色重叠显示。
```xml
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="?attr/colorControlHighlight">
    <item android:id="@android:id/mask" android:drawable="@drawable/icon">
    </item>
</ripple>
```
![](https://pic.liaoheng.me/hc-pic/2024/02/bd440efc6f351d0cb406b131a7c69e59.gif)


## 用形状作边界的Ripple（Ripple With Shape Mask）
波纹`不超过`形状的显示范围，实现内容如下，需要增加一个@android:id/mask配置，配置中的形状就是波纹的背景，会与ripple颜色重叠显示。
- shape.xml
```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android" 
    android:shape="rectangle">
    <solid android:color="#ff9d77"/>
    <corners
        android:bottomRightRadius="100dp"/>
</shape>
```
```xml
<?xml version="1.0" encoding="utf-8"?>
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="#FF0000" >
    <item
        android:id="@android:id/mask"
        android:drawable="@drawable/shape"/>
</ripple>
```
![](https://pic.liaoheng.me/hc-pic/2024/02/ae148630116fff2c868330bbcd1c8cbb.gif)

## 搭配selector作为Ripple（Ripple With Selector）
> 如果在一个ripple标签中，添加一个item，在item的内部写上<selector>标签，那么这个RippleDrawable在按下的时候，同时具有水波效果和selector指定的图层。

波纹`不超过`Selector的显示范围，实现内容如下，需要增加一个@android:id/mask配置，配置中的Selector就是波纹的背景，会与ripple颜色重叠显示。
```xml
<?xml version="1.0" encoding="utf-8"?>
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="#FF0000" >
    <item>
        <selector>
            <item
                android:drawable="@drawable/icon_p"
                android:state_pressed="true">
            </item>
            <item
                android:drawable="@drawable/icon_n"
                android:state_pressed="false">
            </item>
        </selector>
    </item>
</ripple>
```
![](https://pic.liaoheng.me/hc-pic/2024/02/171202fd7e581ad3d0d53296332cedb9.gif)


# 系统默认

在控件中设置背景如下:
- 有边界:
```xml
<ImageView
    android:background="?selectableItemBackground"
 />
```
- 无边界:
```xml
<ImageView
    android:background="?selectableItemBackgroundBorderless"
 />
```

## selectableItemBackground
文件位于：
`/sdk/android/platforms/android-29/data/res/drawable/item_background_material.xml`

文件内容如下：
```xml
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="?attr/colorControlHighlight">
    <item android:id="@id/mask">
        <color android:color="@color/white" />
    </item>
</ripple>
```
文件中的mask参数，用来控制波纹动画的背景或者说是边界，这里波纹动画只会在mask中显示，也就达到动画不超过控件的目的。

## selectableItemBackgroundBorderless
文件位于： 
`/sdk/android/platforms/android-29/data/res/drawable/item_background_borderless_material.xml`
文件内容如下：
```xml
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="?attr/colorControlHighlight" />
```
文件中的没有mask参数，波纹动画效果会超过控件的大小。

## colorControlHighlight(?attr/colorControlHighlight)
文件位于： 
`/sdk/android/platforms/android-29/data/res/color/ripple_material_light.xml`这里的light与`Theme.AppCompat.Light`有关。
文件内容如下：
```xml
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:alpha="@dimen/highlight_alpha_material_light"
          android:color="@color/foreground_material_light" />
</selector>
```

# 参考
[where-can-i-found-definition-for-selectableitembackgroundborderless-reference-at](https://stackoverflow.com/questions/64654113/where-can-i-found-definition-for-selectableitembackgroundborderless-reference-at)

title: 'Android BLE Issues(问题集) '
author: liaoheng
tags: []
categories: []
date: 2016-12-19 22:08:00
---
>新进入公司做智能家居设备，于是查寻了一下Android BLE的问题，这是比较全面的一片文章，简单翻译一下。

>原文链接:http://www.slideshare.net/yeokm1/introduction-to-bluetooth-low-energy

## Android issues (the past) `BLE支持之前的问题`
>http://stackoverflow.com/questions/14118658/devices-with-android-4-2-jelly-bean-supported-bluetooth-low-energy-ble

- Before Android 4.3 (July 2013) `Android 4.3在2013年七月之前`
  - Fragmentation hell `痛苦的碎片化`
  - Proprietary Libraries by OEMs, Android <= 4.2 `各大OEM厂商提供的私有库`
    - Samsung (quite reliable)
    - HTC – buggy, unreliable
    - Motorola (reliable but conflicts with Android 4.3)
  - Architecture issues `架构上的问题`
- Testing issues `其它问题`


## Android issues (today) `BLE支持之后的问题`
>http://stackoverflow.com/questions/17870189/android-4-3-bluetooth-low-energy-unstable

1. OS fragmentation
 - 74.2% of Android devices support BLE `只有74.2%的设备支持`
 - <font color="#D32627">Few support peripheral mode: 35.3% minus Nexus 4, 5, 7 (2012/2013)</font> `只有35.3%支持手机作为外围设备，只有5.0以上才支持。`

2. All callbacks from BLE APIs are not on UI thread. `所有的回调不是在UI线程`

3. APIs considered new, some functions are buggy `考虑使用新的API的话会有一些bug`

4. Frequent connection drops (< 5.0) `5.0之前频繁连接会导致连接无法使用`
> 参考：https://code.google.com/p/android/issues/detail?id=180440

5. Max BLE connections: `最大连接数`
 - Software cap in Bluedroid code: BTA_GATTC_CONN_MAX, GATT_MAX_PHY_CHANNEL
 - Android 4.3: 4
 - 4.4 - 5.0+: 7

6. No API callback to indicate scanning has stopped `没有提供startLeScan()扫描结束的回调`
 - Scan supposed to be indefinite by API specification, but some phones stop scan after some time `API规范与实际扫描结果并不相符，并且一些手机停止扫描需要的时间比较长。`
 > 比如有一些手机会一直回调设备，直到设备连接成功，而有一些设备(Samsung...)只回调一次

 - Known offender: Samsung `比如著名的三星`
 - Solution: Restart scan at regular intervals `解决方法：扫描定时`

7. Different scan return result behaviours (See further reading) `扫描的结果并不相同`
 - Some phones filter advertisement results, some phones do not. (usually on 4.3 and 4.4)`有一些设备会过滤掉广告结果`

8. Bugs on (Samsung) phones at least < 5.0 `三星5.0之前bug`
 - Scan using service UUID filtering does not work -> no results returned `使用startLeScan()过滤UUID无法工作，没有返回结果`
 - connectGatt() must be called from UI thread`connectGatt()必须在UI线程中调用`

9. Slow LE initial discovery and connection time `初始化查找与连接设备相当缓慢`
 - HTC seems to have this issue`HTC好像有这个问题`
 > 华为，三星也会，其他没有测试过

9. A high-level view on issues collated by Anaren `一些高层view问题的整理`
 - https://atmosphere.anaren.com/wiki/Android_Issues_With_Bluetooth_Low_Energy

10. A more comprehensive list of issues has been collated by iDevicesInc `全面问题清单`
 - https://github.com/iDevicesInc/SweetBlue/wiki/Android-BLE-Issues
 - May be able to overcome using: https://github.com/iDevicesInc/SweetBlue `或许可以使用这个库`

> 个人推荐使用基于RxJava的库:https://github.com/Polidea/RxAndroidBle
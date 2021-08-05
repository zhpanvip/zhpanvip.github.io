---
layout: article
index_img: https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4fd6f3ea8b644f5c9cb64d3622926cb3~tplv-k3u1fbpfcp-watermark.image
title: 用BVP一比一还原自如客APP裸眼3D效果（Android原生）
date: 2021-08-02 22:45:08
categories:
- 自定义View
tags: [自定义View,BVP]
---


前几天，[自如大前端](https://juejin.cn/post/6989227733410644005)开源了一个裸眼3D效果的Banner轮播图的实现方案。看着非常有意思，于是趁着空闲时间结合我的开源库[BannerViewPager](https://github.com/zhpanvip/BannerViewPager)码了一个自如裸眼3D效果的demo。demo基本实现了自如APP的Banner效果。


关于实现原理，[自如客APP裸眼3D效果的实现](https://juejin.cn/post/6989227733410644005)这篇文章已经写得很清楚了，本篇文章就不再赘述了，这里主要看一下代码实现。

## 一、监听传感器的ViewSensorLayout

裸眼3D效果的核心其实就是SensorLayout的实现，这个View通过监听传感器来计算View的位移，然后通过Scroller进行滑动。首先，在构造方法中设置传感器的监听事件：

```java
public SensorLayout(@NonNull Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
    mScroller = new Scroller(context);
    mSensorManager = (SensorManager) getContext().getSystemService(Context.SENSOR_SERVICE);
    // 重力传感器
    if (mSensorManager != null) {
        Sensor accelerateSensor = mSensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER);
        // 地磁场传感器
        Sensor magneticSensor = mSensorManager.getDefaultSensor(Sensor.TYPE_MAGNETIC_FIELD);
        mSensorManager.registerListener(this, accelerateSensor, SensorManager.SENSOR_DELAY_GAME);
        mSensorManager.registerListener(this, magneticSensor, SensorManager.SENSOR_DELAY_GAME);
    }
}
```

然后，在传感器发生变化的时候通过Scroller来移动View，代码如下：

```java
@Override
public void onSensorChanged(SensorEvent event) {
    if (event.sensor.getType() == Sensor.TYPE_ACCELEROMETER) {
        mAccelerateValues = event.values;
    }
    if (event.sensor.getType() == Sensor.TYPE_MAGNETIC_FIELD) {
        mMagneticValues = event.values;
    }
    float[] values = new float[3];
    float[] R = new float[9];
    if (mMagneticValues != null && mAccelerateValues != null)
        SensorManager.getRotationMatrix(R, null, mAccelerateValues, mMagneticValues);
    SensorManager.getOrientation(R, values);
    // x轴的偏转角度
    values[1] = (float) Math.toDegrees(values[1]);
    // y轴的偏转角度
    values[2] = (float) Math.toDegrees(values[2]);
    double degreeX = values[1];
    double degreeY = values[2];
    if (degreeY <= 0 && degreeY > mDegreeYMin) {
        hasChangeX = true;
        scrollX = (int) (degreeY / Math.abs(mDegreeYMin) * mXMoveDistance * mDirection);
    } else if (degreeY > 0 && degreeY < mDegreeYMax) {
        hasChangeX = true;
        scrollX = (int) (degreeY / Math.abs(mDegreeYMax) * mXMoveDistance * mDirection);
    }
    if (degreeX <= 0 && degreeX > mDegreeXMin) {
        hasChangeY = true;
        scrollY = (int) (degreeX / Math.abs(mDegreeXMin) * mYMoveDistance * mDirection);
    } else if (degreeX > 0 && degreeX < mDegreeXMax) {
        hasChangeY = true;
        scrollY = (int) (degreeX / Math.abs(mDegreeXMax) * mYMoveDistance * mDirection);
    }
    smoothScroll(hasChangeX ? scrollX : mScroller.getFinalX(), hasChangeY ? scrollY : mScroller.getFinalY());
}
```
代码中的mDirection表示的是移动的方向，这个参数会开放给外面，来设置跟随传感器移动还是与传感器反向移动。
而smoothScroll通过Scroller实现弹性滑动，代码如下：

```
public void smoothScroll(int destX, int destY) {
    int scrollY = getScrollY();
    int delta = destY - scrollY;
    mScroller.startScroll(destX, scrollY, 0, delta, 200);
    invalidate();
}

@Override
public void computeScroll() {
    if (mScroller.computeScrollOffset()) {
        scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
        postInvalidate();
    }
}
```
上述的代码核心来自[自如客APP裸眼3D效果的实现](https://juejin.cn/post/6989227733410644005)。通过一下布局，便可以实现裸眼3D的效果了。

```xml
<FrameLayout android:layout_width="match_parent"
    android:layout_height="match_parent"
    xmlns:android="http://schemas.android.com/apk/res/android">

  <com.zhpan.sample.banner3d.SensorLayout
      android:layout_width="match_parent"
      android:layout_height="match_parent">

    <ImageView
        android:id="@+id/iv_background"
        android:layout_width="match_parent"
        android:scaleType="centerCrop"
        android:scaleX="1.3"
        android:layout_height="match_parent" />

  </com.zhpan.sample.banner3d.SensorLayout>

  <ImageView
      android:id="@+id/iv_mid"
      android:layout_width="match_parent"
      android:scaleType="fitXY"
      android:layout_marginEnd="@dimen/dp_16"
      android:layout_marginStart="@dimen/dp_16"
      android:layout_gravity="bottom"
      android:layout_height="@dimen/dp_100" />

  <com.zhpan.sample.banner3d.SensorLayout
      android:id="@+id/sensor_layout"
      android:layout_gravity="bottom"
      android:layout_width="match_parent"
      android:layout_height="wrap_content">

    <ImageView
        android:id="@+id/iv_foreground"
        android:layout_width="match_parent"
        android:scaleType="fitXY"
        android:layout_marginStart="@dimen/dp_16"
        android:layout_marginEnd="@dimen/dp_16"
        android:layout_height="@dimen/dp_100" />

  </com.zhpan.sample.banner3d.SensorLayout>

</FrameLayout>
```

![123.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4fd6f3ea8b644f5c9cb64d3622926cb3~tplv-k3u1fbpfcp-watermark.image)

## 二、BannerViewPager实现裸眼3D轮播图

自如APP的Banner是两个Banner叠加在一起来实现的。这里我稍微做了一些优化，即背景层使用ImageView也可以达到一样的效果，而前景层使用[BannerViewPager](https://github.com/zhpanvip/BannerViewPager)实现无线轮播。布局文件代码如下：

```xml
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

  <com.zhpan.sample.banner3d.SensorLayout
      android:id="@+id/sensor_layout"
      android:layout_width="match_parent"
      android:layout_height="@dimen/dp_200">

    <ImageView
        android:id="@+id/iv_background"
        android:layout_width="match_parent"
        android:scaleType="centerCrop"
        android:scaleX="1.3"
        android:layout_height="match_parent" />

  </com.zhpan.sample.banner3d.SensorLayout>

  <com.zhpan.bannerview.BannerViewPager
      android:id="@+id/bvp_foreground"
      android:layout_width="match_parent"
      android:layout_height="@dimen/dp_220" />

</FrameLayout>
```
Activity中实现轮播图的代码如下：

```kotlin
mBannerForeground.apply {
  // 为BannerViewPager设置Adapter
  adapter = ForegroundAdapter()
  // 设置自动轮播
  setAutoPlay(true)
  // 设置指示器相关样式
  setIndicatorStyle(IndicatorStyle.ROUND_RECT)
  setIndicatorSlideMode(IndicatorSlideMode.SCALE)
  setIndicatorSliderWidth(
    resources.getDimensionPixelOffset(R.dimen.dp_7),
    resources.getDimensionPixelOffset(R.dimen.dp_10)
  )
  setIndicatorMargin(0, 0, resources.getDimensionPixelOffset(R.dimen.dp_16), 0)
  setIndicatorSliderColor(
    resources.getColor(R.color.gray_88),
    resources.getColor(R.color.dark_gray)
  )
  setIndicatorGravity(IndicatorGravity.END)
  setScrollDuration(800)
  setIndicatorSliderGap(resources.getDimensionPixelOffset(R.dimen.dp_3))
} // 页面切换的监听
.registerOnPageChangeCallback(object : OnPageChangeCallback() {
  override fun onPageSelected(position: Int) {
    val banner3DData = mBannerForeground.data[position]
    // 切换页面后改变背景
    mIvBackGround.setImageResource(banner3DData?.background!!)
  }
}).create(arrayList)
```
[BannerViewPager](https://github.com/zhpanvip/BannerViewPager)的使用非常简单，可以轻松实现一个带有指示器的，且能无限轮播的Banner。如果你不了解[BannerViewPager](https://github.com/zhpanvip/BannerViewPager)，可以到GitHub主页查看使用步骤。

最终，实现的效果几乎与自如APP一模一样。如下图：


![ezgif-3-aa7e1b542b89.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/36876a30beb74e899c6a6b6583fea81a~tplv-k3u1fbpfcp-watermark.image)

注：图片来源自如APP


最后，相关代码我已提交至GitHub：[AndroidSample](https://github.com/zhpanvip/AndroidSample/tree/master/app/src/main/java/com/zhpan/sample/banner3d)

同时，欢迎关注[BannerViewPager](https://github.com/zhpanvip/BannerViewPager)轮播图库。



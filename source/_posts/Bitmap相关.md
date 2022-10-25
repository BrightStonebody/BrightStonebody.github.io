---
title: Bitmap相关
date: 2022-10-16 23:40:15
tags:
---

## 1. 实现圆角
1. 使用ClipPath先绘制圆角的path，然后再裁剪
```java
    @Override
    protected void onDraw(Canvas canvas) {
        clipPath.reset();
        rectF.set(0, 0, getWidth(), getHeight());
        clipPath.addRoundRect(rectF, radius, radius, Path.Direction.CW);
        // int save = canvas.save();
        canvas.clipPath(clipPath);
        canvas.drawRect(rectF, paint);
        super.onDraw(canvas);
        // canvas.restoreToCount(save);
    }

```
2. paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC_IN));
paint设置Xfermode取交集。
```java
    Paint paint = new Paint();
    paint.setAntiAlias(true);
    Bitmap target = Bitmap.createBitmap(width, height, Config.ARGB_8888);
    Canvas canvas = new Canvas(target);
    RectF rect = new RectF(0, 0, width, height);
    // 设置圆形
    // canvas.drawCircle(width / 2, height / 2, Math.min(width, height) / 2, paint);
    // 设置圆角
    canvas.drawRoundRect(rect, mRadius, mRadius, paint);
    // 核心代码取两个图片的交集部分
    paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC_IN));
    canvas.drawBitmap(source, 0, 0, paint);
    return target;
```

## 2. bitmap加载避免oom
1. inJustDecodeBounds
BitmapFactory.Options#inJustDecodeBounds 可以不将图片加载到内存，获取图片的宽高
2. inSampleSize 采样率压缩
3. options.inPreferredConfig = Bitmap.Config.RGB_565
使用565减少单个像素的内存占用
4. BitmapRegionDecoder 读取图片文件的一个区域
```java
BitmapRegionDecoder bitmapRegionDecoder = BitmapRegionDecoder
        .newInstance(inputStream, false);
Bitmap bitmap = bitmapRegionDecoder.decodeRegion(
        new Rect(0, 0, 938, 938),   //解码区域
        null);  //解码选项 BitmapFactory.Options 类型
```
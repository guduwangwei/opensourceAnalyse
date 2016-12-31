

#### ucrop

*  在分析整个Library的思路之前的我们先看看项目用的的工具类

   * `BitmapLoaderTask` 从网路或者content中读取图片 转化成bitmap

     ~~~java
     // 这里是AsyncTask 我们拷贝他的doInBackground看一看

     @Override
         @NonNull
         protected BitmapWorkerResult doInBackground(Void... params) {
             if (mInputUri == null) {
                 return new BitmapWorkerResult(new NullPointerException("Input Uri cannot be null"));
             }

             try {
               // mInputUri 的schme
               // 如果是https| https 从网络下载
               // 如果是content  获取mImputUri
               // 总结一下这个功能就是解析出 用户选择的图片的路径
                 processInputUri();
             } catch (NullPointerException | IOException e) {
                 return new BitmapWorkerResult(e);
             }

             final ParcelFileDescriptor parcelFileDescriptor;
             try {
                parcelFileDescriptor=mContext.getContentResolver().openFileDescriptor(mInputUri, "r");
             } catch (FileNotFoundException e) {
                 return new BitmapWorkerResult(e);
             }

             final FileDescriptor fileDescriptor;
             if (parcelFileDescriptor != null) {
                 fileDescriptor = parcelFileDescriptor.getFileDescriptor();
             } else {
                 return new BitmapWorkerResult(new NullPointerException("ParcelFileDescriptor was null for given Uri: [" + mInputUri + "]"));
             }

             final BitmapFactory.Options options = new BitmapFactory.Options();
             options.inJustDecodeBounds = true;
             BitmapFactory.decodeFileDescriptor(fileDescriptor, null, options);
             if (options.outWidth == -1 || options.outHeight == -1) {
                 return new BitmapWorkerResult(new IllegalArgumentException("Bounds for bitmap could not be retrieved from the Uri: [" + mInputUri + "]"));
             }

             options.inSampleSize = BitmapLoadUtils.calculateInSampleSize(options, mRequiredWidth, mRequiredHeight);
             options.inJustDecodeBounds = false;

             Bitmap decodeSampledBitmap = null;

             boolean decodeAttemptSuccess = false;
             while (!decodeAttemptSuccess) {
                 try {
                     decodeSampledBitmap = BitmapFactory.decodeFileDescriptor(fileDescriptor, null, options);
                     decodeAttemptSuccess = true;
                 } catch (OutOfMemoryError error) {
                    
                     options.inSampleSize *= 2;
                 }
             }

             if (decodeSampledBitmap == null) {
                 return new BitmapWorkerResult(new IllegalArgumentException("Bitmap could not be decoded from the Uri: [" + mInputUri + "]"));
             }

             if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN) {
                 BitmapLoadUtils.close(parcelFileDescriptor);
             }

             int exifOrientation = BitmapLoadUtils.getExifOrientation(mContext, mInputUri);
             int exifDegrees = BitmapLoadUtils.exifToDegrees(exifOrientation);
             int exifTranslation = BitmapLoadUtils.exifToTranslation(exifOrientation);

             ExifInfo exifInfo = new ExifInfo(exifOrientation, exifDegrees, exifTranslation);

             Matrix matrix = new Matrix();
             if (exifDegrees != 0) {
                 matrix.preRotate(exifDegrees);
             }
             if (exifTranslation != 1) {
                 matrix.postScale(exifTranslation, 1);
             }
             if (!matrix.isIdentity()) {
                 return new BitmapWorkerResult(BitmapLoadUtils.transformBitmap(decodeSampledBitmap, matrix), exifInfo);
             }

             return new BitmapWorkerResult(decodeSampledBitmap, exifInfo);
         }

     ~~~

     ​

   * `BitmapCropTask`

     ~~~

     ~~~

     ​	

   * `RotationGestureDetector ` 旋转手势检测

     ~~~java
     // 这里看一下他的 onTouchEvent方法
     public boolean onTouchEvent(@NonNull MotionEvent event) {
        // event.getActionMasked 不包括索引信息
             switch (event.getActionMasked()) {
                 //主要手指down的动作
                 case MotionEvent.ACTION_DOWN:
                     sX = event.getX();
                     sY = event.getY();
                     mPointerIndex1 = event.findPointerIndex(event.getPointerId(0));
                     mAngle = 0;
                     mIsFirstTouch = true;
                     break;
                //  次要手指down的动作
               case MotionEvent.ACTION_POINTER_DOWN:
                     fX = event.getX();
                     fY = event.getY();
                     mPointerIndex2 = event.findPointerIndex(event.getPointerId(event.getActionIndex()));
                     mAngle = 0;
                     mIsFirstTouch = true;
                     break;
                 case MotionEvent.ACTION_MOVE:
                 // 
                     if (mPointerIndex1 != INVALID_POINTER_INDEX && mPointerIndex2 != INVALID_POINTER_INDEX
                             && event.getPointerCount() > mPointerIndex2) {
                         float nfX, nfY, nsX, nsY;
                         // 两个手指的x,y坐标
                         nsX = event.getX(mPointerIndex1);
                         nsY = event.getY(mPointerIndex1);
                         nfX = event.getX(mPointerIndex2);
                         nfY = event.getY(mPointerIndex2);

                         if (mIsFirstTouch) {
                             mAngle = 0;
                             mIsFirstTouch = false;
                         } else {
                             calculateAngleBetweenLines(fX, fY, sX, sY, nfX, nfY, nsX, nsY);
                         }

                         if (mListener != null) {
                             mListener.onRotation(this);
                         }
                         fX = nfX;
                         fY = nfY;
                         sX = nsX;
                         sY = nsY;
                     }
                     break;
                 case MotionEvent.ACTION_UP:
                     mPointerIndex1 = INVALID_POINTER_INDEX;
                     break;
                 case MotionEvent.ACTION_POINTER_UP:
                     mPointerIndex2 = INVALID_POINTER_INDEX;
                     break;
             }
             return true;
         }
       
     ~~~

     ​

     ~~~java
      //计算旋转角度
     private float calculateAngleBetweenLines(float fx1, float fy1, float fx2, float fy2,
                                                  float sx1, float sy1, float sx2, float sy2) {
             return calculateAngleDelta(
                     (float) Math.toDegrees((float) Math.atan2((fy1 - fy2), (fx1 - fx2))),
                     (float) Math.toDegrees((float) Math.atan2((sy1 - sy2), (sx1 - sx2))));
         }

         private float calculateAngleDelta(float angleFrom, float angleTo) {
             //
             mAngle = angleTo % 360.0f - angleFrom % 360.0f;
              // 这里只是做了一个简单的转化 
              // 
             if (mAngle < -180.0f) {
                 mAngle += 360.0f;
             } else if (mAngle > 180.0f) {
                 mAngle -= 360.0f;
             }

             return mAngle;
         }
     ~~~

     上面的代码是计算距离上一次move手指旋转的距离 可以看看作者这里写的文章（我也是看这里才回的）

      [计算旋转角度原文在这里](https://my.oschina.net/u/2360415/blog/694272)

     但是我们还是把作者的图拿过来啦 对着代码简单分析一下

     ![](https://github.com/guduwangwei/opensourceAnalyse/blob/master/pic/roatecom.png)

     > <e = <b   <f = <e+<d  即 <e = <f-<d = <c-<d  
     >
     >   tan c= k/L1               tan f = M /L 
     >
     >   c = arctan k/L1        d= arctan M/L 
     >
     > 那个根据 上面罗列的公式 我们就可以 <b  = arctan K/L1 - arctan M/L  

      注 ："< "  代表  "角"  

     我们再来看作者的代码	`calculateAngleDelta（angleFrom angleTo）` 这里传入 的两个参数就是对应我们的 ` arctan K/L1   arctan M/L  ` 

     得到 旋转的角度后  `OnRotationGestureListener`  回调给注册他的类

     ----

     getAction() 和getActionMasked（)的区别

     ~~~java
       public static final int ACTION_MASK             = 0xff;
     /**
          * Return the kind of action being performed.
          * Consider using {@link #getActionMasked} and {@link #getActionIndex} to retrieve
          * the separate masked action and pointer index.
          * @return The action, such as {@link #ACTION_DOWN} or
          * the combination of {@link #ACTION_POINTER_DOWN} with a shifted pointer index.
          */
         public final int getAction() {
             return nativeGetAction(mNativePtr);
         }

         /**
          * Return the masked action being performed, without pointer index information.
          * Use {@link #getActionIndex} to return the index associated with pointer actions.
          * @return The action, such as {@link #ACTION_DOWN} or {@link #ACTION_POINTER_DOWN}.
          */
         public final int getActionMasked() {
             return nativeGetAction(mNativePtr) & ACTION_MASK;
         }
     ~~~

​              我们看到 `ACTION_MASK = 0xff `二进制就是 1111 1111  那么 当 `nativeGetAction(mNativePtr) `的值在 0~255 之间的时候  我们可以片段 两个方法得到的值是相等的 ，那个当 `nativeGetAction(mNativePtr)` 的大于255 的时候 两个值就不想等了 什么时候`nativeGetAction(mNativePtr)`的值大于 255呢？答案就是 多点触控的时候

*  思路 

  ​





----

~~~java
class TransformImageView extends ImageView 
 这个类用来显示用户选择要裁剪的图片 那么他们怎么加载图片的呢，
 BitmapLoadUtils.decodeBitmapInBackground()
 那么来看一下 BitmapLoadTask的doinBackground的方法
 
 if (mInputUri == null) {
            return new BitmapWorkerResult(new NullPointerException("Input Uri cannot be null"));
        }

        try {
        // 解析协议头 获取输入
            processInputUri();
        } catch (NullPointerException | IOException e) {
            return new BitmapWorkerResult(e);
        }

        final ParcelFileDescriptor parcelFileDescriptor;
        try {
            parcelFileDescriptor = mContext.getContentResolver().openFileDescriptor(mInputUri, "r");
        } catch (FileNotFoundException e) {
            return new BitmapWorkerResult(e);
        }

        final FileDescriptor fileDescriptor;
        if (parcelFileDescriptor != null) {
            fileDescriptor = parcelFileDescriptor.getFileDescriptor();
        } else {
            return new BitmapWorkerResult(new NullPointerException("ParcelFileDescriptor was null for given Uri: [" + mInputUri + "]"));
        }

        final BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true;
        BitmapFactory.decodeFileDescriptor(fileDescriptor, null, options);
        if (options.outWidth == -1 || options.outHeight == -1) {
            return new BitmapWorkerResult(new IllegalArgumentException("Bounds for bitmap could not be retrieved from the Uri: [" + mInputUri + "]"));
        }
        // 计算需要缩放比例
        options.inSampleSize = BitmapLoadUtils.calculateInSampleSize(options, mRequiredWidth, mRequiredHeight);
        options.inJustDecodeBounds = false;
        Bitmap decodeSampledBitmap = null;
        boolean decodeAttemptSuccess = false;
        while (!decodeAttemptSuccess) {
            try {
                 // google 上说这样的加载bitmap 更加节省内存   
                decodeSampledBitmap = BitmapFactory.decodeFileDescriptor(fileDescriptor, null, options);
                decodeAttemptSuccess = true;
            } catch (OutOfMemoryError error) {
                Log.e(TAG, "doInBackground: BitmapFactory.decodeFileDescriptor: ", error);
                options.inSampleSize *= 2;
            }
        }

        if (decodeSampledBitmap == null) {
            return new BitmapWorkerResult(new IllegalArgumentException("Bitmap could not be decoded from the Uri: [" + mInputUri + "]"));
        }

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN) {
            BitmapLoadUtils.close(parcelFileDescriptor);
        }

        int exifOrientation = BitmapLoadUtils.getExifOrientation(mContext, mInputUri);
        int exifDegrees = BitmapLoadUtils.exifToDegrees(exifOrientation);
        int exifTranslation = BitmapLoadUtils.exifToTranslation(exifOrientation);

        ExifInfo exifInfo = new ExifInfo(exifOrientation, exifDegrees, exifTranslation);

        Matrix matrix = new Matrix();
        if (exifDegrees != 0) {
            matrix.preRotate(exifDegrees);
        }
        if (exifTranslation != 1) {
            matrix.postScale(exifTranslation, 1);
        }
        if (!matrix.isIdentity()) {
            return new BitmapWorkerResult(BitmapLoadUtils.transformBitmap(decodeSampledBitmap, matrix), exifInfo);
        }
~~~



*  `TransFormImageView extends ImageView `(这里有两个算法,只找到这个，具体的推导过程找不到了 我的数学一个不好)[点这里](https://judepereira.com/blog/calculate-the-real-scale-factor-and-the-angle-of-rotation-from-an-android-matrix/)

~~~java
  // 这个类 提供了基本的变化， scale rotate 
   // 算出当前图片的缩放比例 
  //  mscale_x x方向的比例 skew_y y方向的错切系数
      public float getMatrixScale(@NonNull Matrix matrix) {
          return (float) Math.sqrt(Math.pow(getMatrixValue(matrix, Matrix.MSCALE_X), 2)
                  + Math.pow(getMatrixValue(matrix, Matrix.MSKEW_Y), 2));
      }
     /**
       * This method calculates rotation angle for given Matrix object.
       */
  // ??? 并不知道为什么要这么算
      public float getMatrixAngle(@NonNull Matrix matrix) {
          return (float) -(Math.atan2(getMatrixValue(matrix, Matrix.MSKEW_X),
                  getMatrixValue(matrix, Matrix.MSCALE_X)) * (180 / Math.PI));
      }
~~~

*    `cropImgeView extends TransFormImageView `

     ~~~

     ~~~

```java
  /**
* This method calculates image minimum and maximum scale values for current {@link #mCropRect}.
*/
 private void calculateImageScaleBounds(float drawableWidth, float drawableHeight) {
      float widthScale = Math.min(mCropRect.width() / drawableWidth, mCropRect.width() / drawableHeight);
      float heightScale = Math.min(mCropRect.height() / drawableHeight, mCropRect.height() / drawableWidth);
      mMinScale = Math.min(widthScale, heightScale);
      mMaxScale = mMinScale * mMaxScaleMultiplier;

  }
```

   // 注释写的很明白 计算image的 max scale value 和 min scale value 那个到底是怎么算的呢， 如果没有看到作者的代码你会怎么算呢
  // 这里因为图片是可以旋转的 所以计算的是四个值，去其中最小的

* `setImageToWrapCropBounds`

~~~java
       protected boolean isImageWrapCropBounds(float[] imageCorners) {
            mTempMatrix.reset(); // 重置为单位矩阵
            /**
             这里为什么什么设置的是 -getCurrentAngele
              我发现 如果 改成 getCurrentAngle 似乎也没有什么影响 
              不明白 ？？？？？？
            */
            mTempMatrix.setRotate(-getCurrentAngle());
            // image 的顶点的集合
            float[] unrotatedImageCorners = Arrays.copyOf(imageCorners, imageCorners.length);
            // 计算 image 基于当前matrix 变化后的，顶点的坐标集合
            mTempMatrix.mapPoints(unrotatedImageCorners);
            // 裁剪框的四个顶点的坐标集合
            float[] unrotatedCropBoundsCorners = RectUtils.getCornersFromRect(mCropRect);
            //同理 计算裁剪框 基于当前matrix 变化后的 顶点坐标的集合
            mTempMatrix.mapPoints(unrotatedCropBoundsCorners);

            return RectUtils.trapToRect(unrotatedImageCorners).contains(RectUtils.trapToRect(unrotatedCropBoundsCorners));
        }
~~~

* [单位矩阵](http://baike.baidu.com/link?url=NZS4B6_14nymCcGidVXn7TfE9-NqYiXkRmFNbFwd4jvInnE9qGvWMFlrCopQr_YMAcZM3N5cGdRr0pYMsRa6CQHTBfK2w9OK6RIkwR43S5Cq2raQ9Jnp9KYwIwkNp0NO)

~~~java
  private float[] calculateImageIndents() {
        mTempMatrix.reset();
        mTempMatrix.setRotate(-getCurrentAngle());

        float[] unrotatedImageCorners = Arrays.copyOf(mCurrentImageCorners, mCurrentImageCorners.length);
    // 注释清楚明白 获得图片的四个顶点的左边 
    /**
     * 0------->1
     * ^        |
     * |        |
     * |        v
     * 3<-------2
    */
        float[] unrotatedCropBoundsCorners = RectUtils.getCornersFromRect(mCropRect);
       // 计算旋转 -getCurrentAngle的角度后对应的坐标
        mTempMatrix.mapPoints(unrotatedImageCorners);
        mTempMatrix.mapPoints(unrotatedCropBoundsCorners);

        RectF unrotatedImageRect = RectUtils.trapToRect(unrotatedImageCorners);
        RectF unrotatedCropRect = RectUtils.trapToRect(unrotatedCropBoundsCorners);

        float deltaLeft = unrotatedImageRect.left - unrotatedCropRect.left;
        float deltaTop = unrotatedImageRect.top - unrotatedCropRect.top;
        float deltaRight = unrotatedImageRect.right - unrotatedCropRect.right;
        float deltaBottom = unrotatedImageRect.bottom - unrotatedCropRect.bottom;

        float indents[] = new float[4];
        indents[0] = (deltaLeft > 0) ? deltaLeft : 0;
        indents[1] = (deltaTop > 0) ? deltaTop : 0;
        indents[2] = (deltaRight < 0) ? deltaRight : 0;
        indents[3] = (deltaBottom < 0) ? deltaBottom : 0;

        mTempMatrix.reset();
        mTempMatrix.setRotate(getCurrentAngle());
        mTempMatrix.mapPoints(indents);

        return indents;
    }
~~~

~~~java

    /**
     * 点集 转换成RectF
     * Takes an array of 2D coordinates representing corners and returns the
     * smallest rectangle containing those coordinates.
     *
     * @param array array of 2D coordinates
     * @return smallest rectangle containing coordinates
     */
    public static RectF trapToRect(float[] array) {
        // Float.POSITIVE_INFINITY 无穷大    Float.NEGATIVE_INFINITY 负无穷
        RectF r = new RectF(Float.POSITIVE_INFINITY, Float.POSITIVE_INFINITY,
                Float.NEGATIVE_INFINITY, Float.NEGATIVE_INFINITY);
        for (int i = 1; i < array.length; i += 2) {
            float x = Math.round(array[i - 1] * 10) / 10.f;
            float y = Math.round(array[i] * 10) / 10.f;
            r.left = (x < r.left) ? x : r.left;
            r.top = (y < r.top) ? y : r.top;
            r.right = (x > r.right) ? x : r.right;
            r.bottom = (y > r.bottom) ? y : r.bottom;
        }
        r.sort();
        return r;
    }
~~~




```java
    /**
     * 如果图片没有填满 裁剪框 这个方法会填满裁剪框 如果已经是填满的就 什么都不做
     * If image doesn't fill the crop bounds it must be translated and scaled properly to fill those.
     * <p/>
     * Therefore this method calculates delta X, Y and scale values and passes them to the
     * {@link WrapCropBoundsRunnable} which animates image.
     * Scale value must be calculated only if image won't fill the crop bounds after it's translated to the
     * crop bounds rectangle center. Using temporary variables this method checks this case.
     */
    public void setImageToWrapCropBounds(boolean animate) {
        // todo 这里 mbitmapLaidOut = false 为什么不能进行
        if (mBitmapLaidOut && !isImageWrapCropBounds()) {

            float currentX = mCurrentImageCenter[0];
            float currentY = mCurrentImageCenter[1];
            float currentScale = getCurrentScale();

            float deltaX = mCropRect.centerX() - currentX;
            float deltaY = mCropRect.centerY() - currentY;
            float deltaScale = 0;

            mTempMatrix.reset();
            mTempMatrix.setTranslate(deltaX, deltaY);

            final float[] tempCurrentImageCorners = Arrays.copyOf(mCurrentImageCorners, mCurrentImageCorners.length);
            mTempMatrix.mapPoints(tempCurrentImageCorners);

            boolean willImageWrapCropBoundsAfterTranslate = isImageWrapCropBounds(tempCurrentImageCorners);

            if (willImageWrapCropBoundsAfterTranslate) {
                final float[] imageIndents = calculateImageIndents();
                deltaX = -(imageIndents[0] + imageIndents[2]);
                deltaY = -(imageIndents[1] + imageIndents[3]);
            } else {
                RectF tempCropRect = new RectF(mCropRect);
                mTempMatrix.reset();
                mTempMatrix.setRotate(getCurrentAngle());
                mTempMatrix.mapRect(tempCropRect);

                final float[] currentImageSides = RectUtils.getRectSidesFromCorners(mCurrentImageCorners);

                deltaScale = Math.max(tempCropRect.width() / currentImageSides[0],
                        tempCropRect.height() / currentImageSides[1]);
                deltaScale = deltaScale * currentScale - currentScale;
            }

            if (animate) {
                post(mWrapCropBoundsRunnable = new WrapCropBoundsRunnable(
                        CropImageView.this, mImageToWrapCropBoundsAnimDuration, currentX, currentY, deltaX, deltaY,
                        currentScale, deltaScale, willImageWrapCropBoundsAfterTranslate));
            } else {
                postTranslate(deltaX, deltaY);
                if (!willImageWrapCropBoundsAfterTranslate) {
                    zoomInImage(currentScale + deltaScale, mCropRect.centerX(), mCropRect.centerY());
                }
            }
        }
    }

```

* `UcropView extends CropImageView `

* ​

* `OverLayVIew`  裁剪框

   

  ​
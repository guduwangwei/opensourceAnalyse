#### ucrop

*  在分析整个Library的思路之前的我们先看看项目用的的工具类

   * `BitmapLoaderTask` 从网路或者content中读取图片 转化成bitmap

     ~~~

     ~~~

     ​

   * `BitmapCropTask`

     ~~~

     ~~~

     ​	

   * `RotationGestureDetector ` 旋转手势检测

   * ​

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

             if (mAngle < -180.0f) {
                 mAngle += 360.0f;
             } else if (mAngle > 180.0f) {
                 mAngle -= 360.0f;
             }

             return mAngle;
         }


     ~~~

     上面的代码是计算距离上一次move手指旋转的距离 可以看看作者这里写的文章（我也是看这里才回的）

     [计算旋转角度](https://my.oschina.net/u/2360415/blog/694272)

     但是我们还是把作者的图拿过来啦 对着代码简单分析一下

     ![](https://github.com/guduwangwei/opensourceAnalyse/blob/master/pic/roatecom.png)

     ​

     > <e = <b   <f = <e+<d  即 <e = <f-<d = <c-<d  
     >
     > tan c = 

     ​

     ​

     ​

     ​

     ​

     ​

     ​

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

        options.inSampleSize = BitmapLoadUtils.calculateInSampleSize(options, mRequiredWidth, mRequiredHeight);
        options.inJustDecodeBounds = false;

        Bitmap decodeSampledBitmap = null;

        boolean decodeAttemptSuccess = false;
        while (!decodeAttemptSuccess) {
            try {
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




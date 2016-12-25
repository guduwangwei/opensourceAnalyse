#### ucrop

*  在分析整个Library的思路之前的我们先看看项目用的的工具类

  ~~~
  BitmapLoaderTask 从网路或者content中读取图片 转化成bitmap
  BitmapCropTask
  RotationGestureDetector
  ~~~

  ​







~~~
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




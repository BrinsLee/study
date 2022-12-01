## 一、框架设计

模型图

![](E:\workspace\Android-Notes\Images\camera2.png)

## 优点

- 改进了新硬件的性能，Supported Hardware Level的概念，不同厂商对Camera2的支持程度不同，从低到高有LEGACY、LIMITED、FULL 和 LEVEL_3四个级别
- 以更快的间隔拍摄图片
- 显示来自多个摄像机的预览
- 直接应用效果和滤镜

## 开发流程

### CameraManager

负责查询和建立相机连接的系统服务，功能不多：

1. 将相机信息封装到CaneraCharacteristics，并提供获取CaneraCharacteristics实例的方式
2. 根据指定的相机ID连接相机设备
3. 提供将闪光灯设置成手电筒的快捷方式

### CameraCharacteristics

是一个只读的相机信息提供者，其内部携带大量的相机信息，包括代表相机朝向的LENGS_FACIN；判断闪光灯是否可用的FLASH_INFO_AVAILABLE；获取所有可用AE模式的CONTROL_AE_AVAILABLE_MODES等，和Camera1中的Camera.Parameters类似

### CameraDevice

CameraDevice 代表当前连接的相机设备，它的职责有以下四个：

1. 根据指定参数创建CameraCaptureSession
2. 根据指定的模板创建CaptureRequest
3. 关闭相机设备
4. 监听相机状态，例如断开连接、开启成功和开启失败等

### Surface

Surface 是一块用于填充图像数据的内存空间，例如你可以使用 SurfaceView 的 Surface 接收每一帧预览数据用于显示预览画面，也可以使用 ImageReader 的 Surface 接收 JPEG 或 YUV 数据。每一个 Surface 都可以有自己的尺寸和数据格式，你可以从 CameraCharacteristics 获取某一个数据格式支持的尺寸列表。

### CameraCaptureSession

CameraCaptureSession 实际上就是配置了目标 Surface 的 Pipeline 实例，我们在使用相机功能之前必须先创建 CameraCaptureSession 实例。一个 CameraDevice 一次只能开启一个 CameraCaptureSession，绝大部分的相机操作都是通过向 CameraCaptureSession 提交一个 Capture 请求实现的，例如拍照、连拍、设置闪光灯模式、触摸对焦、显示预览画面等。

### CaptureRequest

CaptureRequest 是向 CameraCaptureSession 提交 Capture 请求时的信息载体，其内部包括了本次 Capture 的参数配置和接收图像数据的 Surface。CaptureRequest 可以配置的信息非常多，包括图像格式、图像分辨率、传感器控制、闪光灯控制、3A 控制等，可以说绝大部分的相机参数都是通过 CaptureRequest 配置的。值得注意的是每一个 CaptureRequest 表示一帧画面的操作，这意味着你可以精确控制每一帧的 Capture 操作。

### CaptureResult

CaptureResult 是每一次 Capture 操作的结果，里面包括了很多状态信息，包括闪光灯状态、对焦状态、时间戳等等。例如你可以在拍照完成的时候，通过 CaptureResult 获取本次拍照时的对焦状态和时间戳。需要注意的是，CaptureResult 并不包含任何图像数据，前面我们在介绍 Surface 的时候说了，图像数据都是从 Surface 获取的。

### ImageReader

用于从相机打开的通道中读取需要的格式的原始图像数据，可以设置多个

## 开发流程

![](E:\workspace\Android-Notes\Images\camera流程.png)

### 1、获取CameraManager

```kotlin
private val cameraManager: CameraManager by lazy { getSystemService(CameraManager::class.java)}
```

### 2、获取相机信息

```kotlin
val cameraIdList = cameraManager.cameraIdList
cameraIdList.forEach { cameraId ->
    val cameraCharacteristics = cameraManager.getCameraCharacteristics(cameraId)
    if (cameraCharacteristics
        .isHardwareLevelSupported(CameraCharacteristics.INFO_SUPPORTED_HARDWARE_LEVEL_FULL)) {
        if (cameraCharacteristics[CameraCharacteristics.LENS_FACING] == CameraCharacteristics
            .LENS_FACING_FRONT) {
            frontCameraId = cameraId
            frontCameraCharacteristics = cameraCharacteristics
        } else if (cameraCharacteristics[CameraCharacteristics.LENS_FACING] == CameraCharacteristics.LENS_FACING_BACK) {
            backCameraId = cameraId
            backCameraCharacteristics = cameraCharacteristics
        }
    }
}
```

通过CameraManager获取到所有摄像头cameraId,通过循环判断是前摄像头（`CameraCharacteristics.LENS_FACING_FRONT`）还是后摄像头（`CameraCharacteristics.LENS_FACING_BACK`）

### 3、初始化ImageReader

```kotlin
private var jepgImageReader: ImageReader? = null
jpegImageReader = ImageReader.newInstance(imageSize.width, imageSize.height, ImageFormat.JPEG, 5)
jpegImageReader?.setOnImageAvailableListener(OnJpegImageAvailableListener(), cameraHandler)

private inner class OnJpegImageAvailableListener : ImageReader.OnImageAvailableListener {
    private val dateFormat: DateFormat = SimpleDateFormat("yyyyMMddHHmmssSSS", Locale.getDefault())
    private val cameraDir: String = "${Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DCIM)}/Camera"

    @WorkerThread
  	override fun onImageAvailable(imageReader: ImageReader) {
    		  
  	val image = imageReader.acquireNextImage()
    val captureResult = captureResults.take()
    if (image != null && captureResult != null) {
      image.use {
        val jpegByteBuffer = it.planes[0].buffer// Jpeg image data only occupy the planes[0].
        val jpegByteArray = ByteArray(jpegByteBuffer.remaining())
        jpegByteBuffer.get(jpegByteArray)
        val width = it.width
        val height = it.height
        saveImageExecutor.execute {
          val date = System.currentTimeMillis()
          val title = "IMG_${dateFormat.format(date)}"// e.g. IMG_20190211100833786
          val displayName = "$title.jpeg"// e.g. IMG_20190211100833786.jpeg
          val path = "$cameraDir/$displayName"// e.g. /sdcard/DCIM/Camera/IMG_20190211100833786.jpeg
          val orientation = captureResult[CaptureResult.JPEG_ORIENTATION]
          val location = captureResult[CaptureResult.JPEG_GPS_LOCATION]
          val longitude = location?.longitude ?: 0.0
          val latitude = location?.latitude ?: 0.0

          // Write the jpeg data into the specified file.
          File(path).writeBytes(jpegByteArray)

          // Insert the image information into the media store.
          val values = ContentValues()
          values.put(MediaStore.Images.ImageColumns.TITLE, title)
          values.put(MediaStore.Images.ImageColumns.DISPLAY_NAME, displayName)
          values.put(MediaStore.Images.ImageColumns.DATA, path)
          values.put(MediaStore.Images.ImageColumns.DATE_TAKEN, date)
          values.put(MediaStore.Images.ImageColumns.WIDTH, width)
          values.put(MediaStore.Images.ImageColumns.HEIGHT, height)
          values.put(MediaStore.Images.ImageColumns.ORIENTATION, orientation)
          values.put(MediaStore.Images.ImageColumns.LONGITUDE, longitude)
          values.put(MediaStore.Images.ImageColumns.LATITUDE, latitude)
          contentResolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values)

          // Refresh the thumbnail of image.
          val thumbnail = getThumbnail(path)
          if (thumbnail != null) {
            runOnUiThread {
              thumbnailView.setImageBitmap(thumbnail)
              thumbnailView.scaleX = 0.8F
              thumbnailView.scaleY = 0.8F
              thumbnailView.animate().setDuration(50).scaleX(1.0F).scaleY(1.0F).start()
            }
          }
        }
      }
    }
  }
    
}
```

ImageReader使获取图像数据的重要途经，通过它可以获取到不同格式的图像数据，例如JEPG、YUV、RAW等。通过ImageReader.newInstance(int width, int height, int format, int maxImages)创建ImageReader

- width：图像数据宽度
- height：图像数据高度
- format：图像数据格式，ImageFormat.JEPG, ImageFormat.YUV_420_888等
- maxImages：最大Image个数，Image对象池的大小，指定了能从ImageReader获取Image对象的最大值

ImageReader其他方法回调

- ImageReader.OnImageAvailableListener：有新图像数据的回调
- acquireLatestImage()：获取最新Image，删除旧的，如果没有可用的，返回null
- acquireNextImage()：获取下一个最新的可用的Image，没有则返回null
- close()：释放与此ImageReader关联的所有资源
- getSurface()：获取当前ImageReader生成Image的Surface

### 4、打开相机设备

```kotlin
val cameraStateCallback = CameraStateCallback()
cameraManager.openCamera(cameraId, cameraStateCallback, mainHandler)

......

 private inner class CameraStateCallback : CameraDevice.StateCallback() {
        @MainThread
        override fun onOpened(camera: CameraDevice) {
            cameraDeviceFuture!!.set(camera)
            cameraCharacteristicsFuture!!.set(getCameraCharacteristics(camera.id))
        }

        @MainThread
        override fun onClosed(camera: CameraDevice) {

        }

        @MainThread
        override fun onDisconnected(camera: CameraDevice) {
            cameraDeviceFuture!!.set(camera)
            closeCamera()
        }

        @MainThread
        override fun onError(camera: CameraDevice, error: Int) {
            cameraDeviceFuture!!.set(camera)
            closeCamera()
        }
    }
```

`cameraManager.openCamera(@NonNull String cameraId,@NonNull final CameraDevice.StateCallback callback, @Nullable Handler handler)`的三个参数:

- camreaId：摄像头的唯一标识
- callback：设备连接状态变化回调
- handler：回调执行的Handler对象，传入null则使用当前主线程的Handler

其中CameraStateCallback回调:

-  onOpened：表示相机打开成功，可以真正开始使用相机，创建Capture会话
- onDisconnected：当相机断开连接时回调该方法，需要进行释放相机的操作
- onError：当相机打开失败时，需要进行释放相机的操作
- onClosed：调用Camera.close()后回调方法

### 5、创建Capture Session

```kotlin
val sessionStateCallback = SessionStateCallback()
......
val cameraDevice = cameraDeviceFuture?.get()
cameraDevice?.createCaptureSession(outputs, sessionStateCallback, mainHandler)

private inner class SessionStateCallback : CameraCaptureSession.StateCallback() {
        @MainThread
        override fun onConfigureFailed(session: CameraCaptureSession) {
            captureSessionFuture!!.set(session)
        }

        @MainThread
        override fun onConfigured(session: CameraCaptureSession) {
            captureSessionFuture!!.set(session)
        }

        @MainThread
        override fun onClosed(session: CameraCaptureSession) {

        }
    }
```

`mCameraDevice.createCaptureSession()`创建Capture会话，它接受了三个参数：

- outputs：用于接受图像数据的surface集合，这里传入的是一个preview的surface
- callback：用于监听 Session 状态的CameraCaptureSession.StateCallback对象
- handler：用于执行CameraCaptureSession.StateCallback的Handler对象，传入null则使用当前的主线程Handler

### 6、创建CaptureRequest

CaptureRequest是向CameraCaptureSession提交Capture请求时的信息载体，其内部包括了本次Capture的参数配置和接收图像数据的Surface

```kotlin
if (cameraDevice != null) {
  previewImageRequestBuilder = cameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW)
  captureImageRequestBuilder = cameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_STILL_CAPTURE)
}

......

val cameraDevice = cameraDeviceFuture?.get()
val captureSession = captureSessionFuture?.get()
val previewImageRequestBuilder = previewImageRequestBuilder!!
val captureImageRequestBuilder = captureImageRequestBuilder!!
if (cameraDevice != null && captureSession != null) {
  val previewSurface = previewSurface!!
  val previewDataSurface = previewDataSurface
  previewImageRequestBuilder.addTarget(previewSurface)
  // Avoid missing preview frame while capturing image.
  captureImageRequestBuilder.addTarget(previewSurface)
  if (previewDataSurface != null) {
    previewImageRequestBuilder.addTarget(previewDataSurface)
    // Avoid missing preview data while capturing image.
    captureImageRequestBuilder.addTarget(previewDataSurface)
  }
  val previewRequest = previewImageRequestBuilder.build()
  captureSession.setRepeatingRequest(previewRequest, RepeatingCaptureStateCallback(), mainHandler)
}

......

private inner class RepeatingCaptureStateCallback : CameraCaptureSession.CaptureCallback() {
  @MainThread
  override fun onCaptureStarted(session: CameraCaptureSession, request: CaptureRequest, timestamp: Long, frameNumber: Long) {
    super.onCaptureStarted(session, request, timestamp, frameNumber)
  }

  @MainThread
  override fun onCaptureCompleted(session: CameraCaptureSession, request: CaptureRequest, result: TotalCaptureResult) {
    super.onCaptureCompleted(session, request, result)
  }
}
```

除了模式的配置，CaptureRequest还可以配置很多其他信息，例如图像格式、图像分辨率、传感器控制、闪光灯控制、3A(自动对焦-AF、自动曝光-AE和自动白平衡-AWB)控制等。在createCaptureSession的回调中可以进行设置，最后通过`build()`方法生成CaptureRequest对象。

### 7、预览

Camera2中，通过连续重复的Capture实现预览功能，每次Capture会把预览画面显示到对应的Surface上。连续重复的Capture操作通过

```
captureSession.setRepeatingRequest(previewRequest, RepeatingCaptureStateCallback(), mainHandler)
```

实现，该方法有三个参数：

- request：CaptureRequest对象
- listener：监听Capture 状态的回调
- handler：用于执行CameraCaptureSession.CaptureCallback的Handler对象，传入null则使用当前的主线程Handler

停止预览使用`mCaptureSession.stopRepeating()`方法。

### 8、拍照

设置requestSession后，就可以真正的开始拍照操作

```kotlin
val captureImageRequest = captureImageRequestBuilder.build()
captureSession.capture(captureImageRequest, CaptureImageStateCallback(), mainHandler)

......

private inner class CaptureImageStateCallback : CameraCaptureSession.CaptureCallback() {

  @MainThread
  override fun onCaptureStarted(session: CameraCaptureSession, request: CaptureRequest, timestamp: Long, frameNumber: Long) {
    super.onCaptureStarted(session, request, timestamp, frameNumber)
    // Play the shutter click sound.
    cameraHandler?.post { mediaActionSound.play(MediaActionSound.SHUTTER_CLICK) }
  }

  @MainThread
  override fun onCaptureCompleted(session: CameraCaptureSession, request: CaptureRequest, result: TotalCaptureResult) {
    super.onCaptureCompleted(session, request, result)
    captureResults.put(result)
  }
}
```

### 9、关闭相机

和其他硬件资源的使用一样，当我们不再需要使用相机时记得调用 CameraDevice.close() 方法及时关闭相机回收资源。关闭相机的操作至关重要，因为如果你一直占用相机资源，其他基于相机开发的功能都会无法正常使用，严重情况下直接导致其他相机相关的 APP 无法正常使用，当相机被完全关闭的时候会通过 CameraStateCallback.onCllosed() 方法通知你相机已经被关闭。那么在什么时候关闭相机最合适呢？个人的建议是在 onPause() 的时候就一定要关闭相机，因为在这个时候相机页面已经不是用户关注的焦点，大部分情况下已经可以关闭相机了。

```kotlin
cameraDevice?.close()
previewDataImageReader?.close()
jpegImageReader?.close()
```




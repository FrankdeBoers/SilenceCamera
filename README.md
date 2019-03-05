# SilenceCamera
静默拍照，不需要相机预览，可以在后台完成拍照并保存。  可用于人脸验证解锁。

#### 一、首先需要了解一下相机开发相关知识。相机开发有两个重要的系统类：
- 1、Camera.java  (android.hardware.Camera)
- 2、SurafaceView.java (android.view.SurafaceView)
https://github.com/FrankdeBoers/SilenceCamera/blob/master/screen/camera.png

##### 1.2 Surface、SurfaceView、SurfaceHolder的关系
https://github.com/FrankdeBoers/SilenceCamera/blob/master/screen/surface.png
 
> Surface是用来处理屏幕内容合成器所管理的原始缓存区的工具，它通常由图像缓冲区的消费者创建(如SurfaceTexture、MediaRecorder)，然后被移交到生产者(如MediaPlayer)。

> SurfaceHolder 一个抽象接口，给持有Surface的对象使用，它可以控制Surface的大小和格式，编辑Surface的像素，以及监听Surface的变化。 这个接口通常通过SurfaceView获得。
> SurfaceHolder.Callback有三个回调方法：
> 1. surfaceCreated()
> 1. surfaceChanged()
> 1. surfaceDestroyed()

> SurfaceView 提供了嵌入式图层的专用Surface。可以控制Surface的格式和大小，SurfaceView负责把Surface显示在屏幕的正确位置。
> SurfaceView继承自View，用于在屏幕上面显示相机的预览画面。
> SurfaceView中有两个对象，一个是Surface，一个是SurfaceHolder，我们可以通过getHolder()方法，获得当前SurfaceView的SurfaceHolder对象，通过SurfaceHolder的回调，可以知道surface的状态。

##### 1.3 Camera
Camera类的接口和方法有很多，一个最简单的Camera应用的实现：

// 在surfaceCreated()中

```
Camera mCamera = Camera.open(0);
Camera.Parameters mParameters = mCamera.getParameters();
mParameters.setPreviewSize(640, 480);
mParameters.setPictureSize(640, 480);
mCamera.setParameters(mParameters);
mCamera.setPreviewDisplay(mSurfaceView.getHolder);
mCamera.startPreview();
```


经过以上简单的代码，已经实现了Camera预览。

关于Camera正常的预览拍照，可以参考https://github.com/FrankdeBoers/TensorCamera下的LocalCameraManager.java文件。

#### 二、如何实现无预览拍照
由相机开发经验可以知道，从camera读取到的预览（preview）图像流一定要输出到一个可见的（Visible）SurfaceView上，也就是SurfaceView无法在息屏或者用户正常使用APP的时候工作，不然会弹出Camera预览画面，这样用户就知道相机启动了。  
常规模式的SurfaceView无法实现静默拍照。我们返回第一小节，在对Surface的介绍中，可以看到，SurfaceTexture也可以创建Surface。  而这个SurfaceTexture就是我们实现静默拍照的切入点。
查看SurfaceTexture的官方介绍
> A SurfaceTexture may also be used in place of a SurfaceHolder when specifying the output destination of the older {@link android.hardware.Camera} API. Doing so will cause all the frames from the image stream to be sent to the SurfaceTexture object rather than to the device's display.

> 解释：SurfaceTexture也可以在确认相机的输出目标后，用来替代SurfaceHolder。 替换后，相机的图像流数据，不会再输出到设备屏幕，而是输出到SurfaceTexture上。
根据上面的解释，SurfaceTexture也可以用作展示Camera的预览，但是SurfaceTexture可以不显示，这样就可以实现无预览拍照。


```
/**
 * 作者：Frank
 * 时间：20180102
 * 
 * 此类功能：用于后台拍照并上传的工具类
 * 主要有两个过程：1、调用摄像头拍照； 2、上传照片
 * 调用方式：CameraUtil.getCameraInstance().takePhoto(getApplicationContext());
 */

import android.content.Context;
import android.hardware.Camera;
import android.media.MediaScannerConnection;
import android.net.Uri;
import android.content.ContentResolver;
import android.content.ContentUris;
import android.database.Cursor;
import android.provider.MediaStore;
import android.os.Environment;
import android.text.format.DateFormat;
import android.util.Log;
import android.graphics.Bitmap;
import android.graphics.BitmapFactory;
import android.graphics.Matrix;

import java.io.File;
import java.io.FileOutputStream;
import java.util.Calendar;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.ByteArrayOutputStream;

public class CameraUtil {
    private final static String TAG = "CameraUtil";

    // 后台拍照保存路径，也是上传图片的路径
    private String mImgOriginalPath = "null";

    // 照片路径
    private String IMG_FILE_PATH
            = Environment.getExternalStoragePublicDirectory("DCIM").getAbsolutePath() + "/Camera";

    private Camera mCamera;

    // 默认前置或者后置相机 0:后置 1:前置
    private int mCameraId = Camera.CameraInfo.CAMERA_FACING_BACK;
    private Camera.Parameters mParameters;

    private Context mContext;

    public CameraUtil() {
    }

    public static CameraUtil getCameraInstance() {
        return CameraUtilHolder.cameraUtil;
    }

    public static class CameraUtilHolder {
        public static CameraUtil cameraUtil = new CameraUtil();
    }

// =======================================================================
// =======================================================================
    /**
     * 开始预览
     * 打开相机---设置相机参数---打开预览界面
     */
    private void startPreview() {
        Log.d(TAG, "[startPreview] >> begin. test");
        if (mCamera == null) {
            try {
                mCamera = Camera.open(mCameraId);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        setCameraParameters();
        mCamera.startPreview();
    }

    /**
     * 释放相机资源
     * 拍照完成后释放相机资源
     */
    private void releaseCamera() {
        if (mCamera != null) {
            mCamera.setPreviewCallback(null);
            mCamera.stopPreview();
            mCamera.release();
            mCamera = null;
        }
    }

    /**
     * 设置Camera参数
     * 设置预览界面的宽高，图片保存的宽、高
     */
    private void setCameraParameters() {
        if (mCamera != null) {
            Log.d(TAG, "setCameraParameters >> begin. mCamera != null");
            if (mParameters == null) {
                mParameters = mCamera.getParameters();
            }           
            mParameters.setPreviewSize(640, 480);
            mParameters.setPictureSize(640, 480);                       
            mCamera.setParameters(mParameters);
        } else {
            Log.e(TAG, "setCameraParameters >> mCamera == null!!");
        }
    }

    /**
     * 开始拍照
     */
    public void takePhoto(Context context, final XiaoXunNetworkManager xiaoXunNetworkManager) {
        Log.d(TAG, "takePhoto >> ");
        mContext = context;
        startPreview();

        mCamera.takePicture(null, null, new Camera.PictureCallback() {
            @Override
            public void onPictureTaken(final byte[] data, Camera mCamera) {
                // 启动存储照片的线程
                final File dir = new File(IMG_FILE_PATH);
                if (!dir.exists()) {
                    dir.mkdirs();      // 创建文件夹
                }
                String name = "IMG_" + DateFormat.format("yyyyMMdd_hhmmss", Calendar.getInstance()) + ".jpg";
                final File file = new File(dir, name);
                FileOutputStream outputStream;
                try {
                    outputStream = new FileOutputStream(file);
                    if (Constant.PROJECT_NAME.equals("SW730")) {
                        outputStream.write(rotateAndMirror(data, 180, false));     // 写入sd卡中
                    } else {
                        outputStream.write(rotateAndMirror(data, 0, true));
                    }                // 写入sd卡中
                    outputStream.close();                      // 关闭输出流
                }catch (FileNotFoundException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }  catch (IOException e) {
                    e.printStackTrace();
                }
                releaseCamera();
            }
        });
    }
}
```







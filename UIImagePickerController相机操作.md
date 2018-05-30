

###前言

 使用UIImagePickerController来实现在项目中添加视频和图片的功能。

###属性和方法

####1.属性####

属性                    | 说明 
-----------------      |------
sourceType             | 拾取源类型，sourceType是枚举类型：UIImagePickerControllerSourceTypePhotoLibrary：照片库，默认值UIImagePickerControllerSourceTypeCamera：摄像头UIImagePickerControllerSourceTypeSavedPhotosAlbum：相簿  
mediaTypes             | 媒体类型,默认情况下此数组包含kUTTypeImage，所以拍照时可以不用设置；但是当要录像的时候必须设置，可以设置为kUTTypeVideo（视频，但不带声音）或者kUTTypeMovie（视频并带有声音）  
videoMaximumDuration   | 视频最大录制时长，默认为10 s  
videoQuality           |  视频质量，枚举类型：UIImagePickerControllerQualityTypeHigh：高清质量UIImagePickerControllerQualityTypeMedium：中等质量，适合WiFi传输UIImagePickerControllerQualityTypeLow：低质量，适合蜂窝网传输UIImagePickerControllerQualityType640x480：UIImagePickerControllerQualityTypeIFrame1280x720：UIImagePickerControllerQualityTypeIFrame960x540：960*540
showsCameraControls   | 是否显示摄像头控制面板，默认为YES
cameraOverlayView     | 摄像头上覆盖的视图，可用通过这个视频来自定义拍照或录像界面
cameraViewTransform   | 摄像头形变
cameraCaptureMode     |摄像头捕获模式，捕获模式是枚举类型：UIImagePickerControllerCameraCaptureModePhoto：拍照模式UIImagePickerControllerCameraCaptureModeVideo：视频录制模式
cameraDevice          |摄像头设备，cameraDevice是枚举类型：UIImagePickerControllerCameraDeviceRear：前置摄像头UIImagePickerControllerCameraDeviceFront：后置摄像头
cameraFlashMode        | 闪光灯模式，枚举类型：UIImagePickerControllerCameraFlashModeOff：关闭闪光灯UIImagePickerControllerCameraFlashModeAuto：闪光灯自动UIImagePickerControllerCameraFlashModeOn：打开闪光灯


####2.方法####

`+ (BOOL)isSourceTypeAvailable:(UIImagePickerControllerSourceType)sourceType`
指定的源类型是否可用，sourceType是枚举类型：
UIImagePickerControllerSourceTypePhotoLibrary：照片库
UIImagePickerControllerSourceTypeCamera：摄像头
UIImagePickerControllerSourceTypeSavedPhotosAlbum：相簿

`+ (NSArray *)availableMediaTypesForSourceType:(UIImagePickerControllerSourceType)sourceType`
指定的源设备上可用的媒体类型，一般就是图片和视频

`+ (BOOL)isSourceTypeAvailable:(UIImagePickerControllerSourceType)sourceType;`
指定来源是否支持：
UIImagePickerControllerSourceTypePhotoLibrary：来自图库
UIImagePickerControllerSourceTypeCamera：来自相机
UIImagePickerControllerSourceTypeSavedPhotosAlbum：来自相册

`+ (BOOL)isCameraDeviceAvailable:(UIImagePickerControllerCameraDevice)cameraDevice`
指定的摄像头是否可用，cameraDevice是枚举类型：
UIImagePickerControllerCameraDeviceRear：前置摄像头
UIImagePickerControllerCameraDeviceFront：后置摄像头

`+ (BOOL)isFlashAvailableForCameraDevice:(UIImagePickerControllerCameraDevice)cameraDevice`
指定摄像头的闪光灯是否可用

`+ (NSArray *)availableCaptureModesForCameraDevice:(UIImagePickerControllerCameraDevice)cameraDevice`
	获得指定摄像头上的可用捕获模式，捕获模式是枚举类型：
UIImagePickerControllerCameraCaptureModePhoto：拍照模式
UIImagePickerControllerCameraCaptureModeVideo：视频录制模式

`- (void)takePicture` 代码启动拍照
`- (BOOL)startVideoCapture` 代码启动录制视频
`- (void)stopVideoCapture` 代码启动停止录制视频

#### 3.代理和补充 ####

`- (void)imagePickerController:(UIImagePickerController *)picker didFinishPickingMediaWithInfo:(NSDictionary *)info`
 代理 - 拍摄或录制完成 这里获取图片或视频信息

`- (void)imagePickerControllerDidCancel:(UIImagePickerController *)picker`
 代理 - 用户取消拍摄

`UIImageWriteToSavedPhotosAlbum(UIImage *image, id completionTarget, SEL completionSelector, void *contextInfo)`
 保存图片到本地相册

`UIVideoAtPathIsCompatibleWithSavedPhotosAlbum(NSString *videoPath)`
能否将视频保存到相簿  更安全的判断

`void UISaveVideoAtPathToSavedPhotosAlbum(NSString *videoPath, id completionTarget, SEL completionSelector, void *contextInfo)`
保存视频到本地相册

#### 要导入：`#import <MobileCoreServices/MobileCoreServices.h> `。不然后出现错误使用未声明的标识符 'kUTTypeImage' ####


###UIImagePickerController的使用流程

> 0. info表中添加权限！！！ 相册`NSPhotoLibraryUsageDescription` 相机`NSCameraUsageDescription` 麦克风`NSMicrophoneUsageDescription`
> 1. 判断设备是否可用
> 2. 创建UIImagePickerController
> 3. 设置对象属性
> 4. 展示UIImagePickerController窗口 (模态)
> 5. 在代理中处理视频或图片信息 (先save到本地)


### 此处应有代码

#### 从相册获取图片 ####

```
/**
  相册获取图片
 */
- (BOOL)openPhotoLibrary
{
    if (![UIImagePickerController isSourceTypeAvailable:UIImagePickerControllerSourceTypePhotoLibrary]) {
        return NO;
    }
    
    UIImagePickerController *picker = [[UIImagePickerController alloc] init];
    picker.sourceType = UIImagePickerControllerSourceTypePhotoLibrary;
    picker.mediaTypes =  [[NSArray alloc] initWithObjects:(NSString*)kUTTypeImage,nil];
    picker.allowsEditing = YES; // 允许简单编辑图片
    picker.delegate = self;
    [self presentViewController:picker animated:YES completion:nil];
    
    return YES;
    
}
```

#### 从相册获取视频 ####

```
/**
 相册获取视频
 */
- (BOOL)openVideoLibrary
{
    if (![UIImagePickerController isSourceTypeAvailable:UIImagePickerControllerSourceTypePhotoLibrary]) {
        return NO;
    }
    
    UIImagePickerController *picker = [[UIImagePickerController alloc] init];
    picker.sourceType = UIImagePickerControllerSourceTypePhotoLibrary;
    picker.mediaTypes =  [[NSArray alloc] initWithObjects:(NSString*)kUTTypeMovie,nil];
    picker.videoQuality = UIImagePickerControllerQualityTypeMedium;
    picker.delegate = self;
    [self presentViewController:picker animated:YES completion:nil];
    
    return YES;
    
}
```

#### 使用相机拍照 ####

```
/**
  拍照
 */
- (BOOL)openCameraForPhoto
{
    // 一般情况下用这个判断
    if (![UIImagePickerController isSourceTypeAvailable:UIImagePickerControllerSourceTypeCamera]) {
        // 该设备不支持拍照
        return NO;
    }
//    if (![UIImagePickerController isCameraDeviceAvailable:UIImagePickerControllerCameraDeviceRear]) {
//        // 前置摄像头不可用
//    }
//    if (![UIImagePickerController isCameraDeviceAvailable:UIImagePickerControllerCameraDeviceFront]) {
//        // 后置摄像头不可用
//    }
    UIImagePickerController *picker = [[UIImagePickerController alloc] init];
    picker.sourceType = UIImagePickerControllerSourceTypeCamera;
    picker.mediaTypes =  [[NSArray alloc] initWithObjects:(NSString*)kUTTypeImage,nil];
    picker.cameraCaptureMode = UIImagePickerControllerCameraCaptureModePhoto; //相机的模式 拍照/摄像
    picker.allowsEditing = YES; // 允许简单编辑图片
    picker.delegate = self;
    [self presentViewController:picker animated:YES completion:nil];
    
    return YES;
}

```

#### 使用相机录像 ####

```
/**
  录制
 */
- (BOOL)openCameraForVideo
{
    // 一般情况下用这个判断
    if (![UIImagePickerController isSourceTypeAvailable:UIImagePickerControllerSourceTypeCamera]) {
        // 该设备不支持拍照
        return NO;
    }
    UIImagePickerController *picker = [[UIImagePickerController alloc] init];
    picker.sourceType = UIImagePickerControllerSourceTypeCamera;
    picker.mediaTypes =  [[NSArray alloc] initWithObjects:(NSString*)kUTTypeMovie,nil];
    picker.videoMaximumDuration = 10.f; // 视频的最大录制时长
    picker.videoQuality = UIImagePickerControllerQualityTypeMedium;
    picker.cameraCaptureMode = UIImagePickerControllerCameraCaptureModeVideo; //相机的模式 拍照/摄像
    picker.delegate = self;
    [self presentViewController:picker animated:YES completion:nil];
    
    return YES;
}


```

#### UIImagePickerController代理 获取到媒体资源 ####

```
/** 
   UIImagePickerController代理 获取到媒体资源
 */
- (void)imagePickerController:(UIImagePickerController *)picker didFinishPickingMediaWithInfo:(NSDictionary<NSString *,id> *)info
{
    NSString *mediaType = info[UIImagePickerControllerMediaType];
    if ([mediaType isEqualToString:(NSString *)kUTTypeImage]) {
        // 图片处理
        UIImage *image = nil;
        if ([picker allowsEditing]) {
            image = info[UIImagePickerControllerEditedImage];//获取编辑后的照片
        }else {
            image = info[UIImagePickerControllerOriginalImage];//获取原始照片
        }
        if (picker.sourceType == UIImagePickerControllerSourceTypeCamera) {
            UIImageWriteToSavedPhotosAlbum(image, self, @selector(photoPath:didFinishSaveWithError:contextINfo:), nil);//保存到相簿
        }
        
        
        self.photoImageView.image = image;
        
    }else if ([mediaType isEqualToString:(NSString *)kUTTypeMovie]) {
        // 视频处理
        NSURL *videoUrl = info[UIImagePickerControllerMediaURL];//视频路径
        
        UIImage *image = [self get_videoThumbImage:videoUrl];
        
        self.videoImageView.image = image;
        
        if (picker.sourceType == UIImagePickerControllerSourceTypeCamera) {
            
            if (UIVideoAtPathIsCompatibleWithSavedPhotosAlbum([videoUrl path])) {
                UISaveVideoAtPathToSavedPhotosAlbum([videoUrl path], self, @selector(video:didFinishSavingWithError:contextInfo:), nil);//保存视频到相簿
            }
            
        }
        
        
    }
    // 选择图片后手动销毁界面
    [picker dismissViewControllerAnimated:YES completion:nil];
}

//图片保存到相册之后的回调
- (void)photoPath:(NSString *)path didFinishSaveWithError:(NSError *)error contextINfo:(void *)contextInfo
{
    if (error) {
        // 保存失败
    }else {
        // 处理图片
    }
}
//视频保存到相册之后的回调
- (void)video:(NSString *)videoPath didFinishSavingWithError:(NSError *)error contextInfo:(void *)contextInfo
{
    if (error) {
        // 保存失败
    }else {
        // 处理视频
        
    }
}


```

#### UIImagePickerController代理 用户取消操作 ####

```
/**
  用户取消操作
 */
- (void)imagePickerControllerDidCancel:(UIImagePickerController *)picker
{
//    手动销毁界面
    [picker dismissViewControllerAnimated:YES completion:nil];
}
```

#### 一些有关视频的实用方法 ####

```
/**
   获取视频时长
 */
- (CGFloat)get_videoTotalWith:(NSURL *)videoURL
{
    NSDictionary *opts = [NSDictionary dictionaryWithObject:[NSNumber numberWithBool:NO]
                                                     forKey:AVURLAssetPreferPreciseDurationAndTimingKey];
    AVURLAsset *urlAsset = [AVURLAsset URLAssetWithURL:videoURL options:opts];
    float second = 0;
    second = urlAsset.duration.value/urlAsset.duration.timescale;
    return second;
}
/**
 获取视频缩略图
 */

- (UIImage *)get_videoThumbImage:(NSURL *)videoURL
{
    NSDictionary *opts = [NSDictionary dictionaryWithObject:[NSNumber numberWithBool:NO] forKey:AVURLAssetPreferPreciseDurationAndTimingKey];
    AVURLAsset *urlAsset = [AVURLAsset URLAssetWithURL:videoURL options:opts];
    AVAssetImageGenerator *generator = [AVAssetImageGenerator assetImageGeneratorWithAsset:urlAsset];
    generator.appliesPreferredTrackTransform = YES;
    CMTime actualTime;
    NSError *error = nil;
    CGImageRef img = [generator copyCGImageAtTime:CMTimeMake(0, 600) actualTime:&actualTime error:&error];
    if (error) {
        return nil;
    }
    return [UIImage imageWithCGImage:img];
}

```

#### 待补充... ####


### 惯例鸣谢 - 参考文献

[感谢崔大大的博客:iOS开发系列--音频播放、录音、视频播放、拍照、视频录制](http://www.cnblogs.com/kenshincui/p/4186022.html)

[感谢张大大的博客:AVFoundation Programming Guide(官方文档翻译)完整版中英对照](http://yoferzhang.com/post/20160724AVFoundation/)








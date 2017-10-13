---
title: iOS自定义相机界面 # 这是标题
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
## 背景
由于项目中需要拍照功能，但是系统原生的相机功能根本满足不了项目的需要，所以就只能自定义一个相加了。苹果再AVFoundation框架中给我们提供了各个api，我们完全可以通过这些api自定义一个满足我们需求的相机。
## 关键类
AVCaptureSession 负责输入流和输出流的管理。
AVCaptureDeviceInput 输入流。连接输入采集设备的
AVCaptureStillImageOutput 输出流。AVCaptureOutput的子类，主要是负责采集图片数据的，不过这个类再10.0以后废弃掉了，改用AVCapturePhotoOutput这个替换，这个类能够支持RAW格式的图片
AVCaptureVideoPreviewLayer 集成CALayer，负责将采集的视频展示出来的一个类，提供了一个预览功能而已
## 实现过程

````Objective-C
#import "TakePictureViewController.h"
#import <AVFoundation/AVFoundation.h>
#import "UIImage+Rotate.h"
#import "UIControl+Custom.h"

#define KSCREEN_WIDTH            [[UIScreen mainScreen] bounds].size.width
#define KSCREEN_HEIGHT           [[UIScreen mainScreen] bounds].size.height

typedef NS_ENUM(NSInteger, AVCamSetupResult ) {
    AVCamSetupResultSuccess,
    AVCamSetupResultCameraNotAuthorized,
    AVCamSetupResultSessionConfigurationFailed
};
@interface TakePictureViewController ()
{
    BOOL lightOn;
    AVCaptureDevice *device;
    ActivityIndicatorTipView *activityView;
}
// AVCaptureSession对象来执行输入设备和输出设备之间的数据传递

@property (nonatomic, strong)AVCaptureSession *session;
// AVCaptureDeviceInput对象是输入流

@property (nonatomic, strong)AVCaptureDeviceInput *videoInput;

// 照片输出流对象

@property (nonatomic, strong)AVCaptureStillImageOutput *stillImageOutput;

// 预览图层，来显示照相机拍摄到的画面

@property (nonatomic, strong)AVCaptureVideoPreviewLayer *previewLayer;
// 切换前后镜头的按钮

@property (nonatomic, strong)UIButton *toggleButton;

// 放置预览图层的View
@property (nonatomic, strong)UIView *cameraShowView;



// 用来展示拍照获取的照片

@property (nonatomic, strong)UIImageView *imageShowView;

@property (nonatomic,strong) UIView *overlayView;

@property (nonatomic) dispatch_queue_t sessionQueue;

@property (nonatomic) AVCamSetupResult setupResult;

@end

@implementation TakePictureViewController

-(BOOL)prefersStatusBarHidden
{
    return YES;
}

-(instancetype)init
{
    self = [super init];
    if (self)
    {
        [self initAll];
    }
    return self;
}
- (void)initAll{
    [self initialSession];
    [self initCameraShowView];
    [self.view addSubview:self.overlayView];
    [self initAVDevice];
    [self setUpCameraLayer];
}
- (void)initialSession
{
    self.session = [[AVCaptureSession alloc] init];
}

- (void)initCameraShowView
{

    self.cameraShowView = [[UIView alloc] initWithFrame:self.view.frame];

    [self.view addSubview:self.cameraShowView];
}

// 这是获取前后摄像头对象的方法

- (AVCaptureDevice *)cameraWithPosition:(AVCaptureDevicePosition)position
{
    NSArray *devices = [AVCaptureDevice devicesWithMediaType:AVMediaTypeVideo];

    for (AVCaptureDevice *captureDevice in devices)
    {
        if (captureDevice.position == position)
        {
            return captureDevice;
        }
    }
    return nil;
}

- (AVCaptureDevice *)frontCamera
{
    return [self cameraWithPosition:AVCaptureDevicePositionFront];
}

- (AVCaptureDevice *)backCamera
{
    return [self cameraWithPosition:AVCaptureDevicePositionBack];
}
/**
 设备手电筒
 */
-(void)initAVDevice
{
    device = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];

    if (![device hasTorch])
    {
        //无手电筒
        [[[UIAlertView alloc] initWithTitle:@"tishi" message:@"无手电筒" delegate:nil cancelButtonTitle:@"确定" otherButtonTitles:nil, nil] show];
    }
    lightOn = NO;
}
- (void)configureSession{

    if ( self.setupResult != AVCamSetupResultSuccess ) {
        return;
    }

    [self.session beginConfiguration];//保证对AVCaptureSession设置的原子性与commitConfiguration配对使用

    self.videoInput = [[AVCaptureDeviceInput alloc] initWithDevice:[self backCamera] error:nil];

    self.stillImageOutput = [[AVCaptureStillImageOutput alloc] init];
     // 输出流的设置参数AVVideoCodecJPEG参数表示以JPEG的图片格式输出图片
    NSDictionary *outputSettings = [[NSDictionary alloc] initWithObjectsAndKeys:AVVideoCodecJPEG,AVVideoCodecKey,@(0.1),AVVideoQualityKey,nil];

    [self.stillImageOutput setOutputSettings:outputSettings];

    if ([self.session canAddInput:self.videoInput])
    {
        [self.session addInput:self.videoInput];
    }

    if ([self.session canAddOutput:self.stillImageOutput])
    {
        [self.session addOutput:self.stillImageOutput];
    }
    [self.session commitConfiguration];
}

- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor whiteColor];
    self.sessionQueue = dispatch_queue_create("session queue", DISPATCH_QUEUE_SERIAL);
    switch ( [AVCaptureDevice authorizationStatusForMediaType:AVMediaTypeVideo] )
    {
        case AVAuthorizationStatusAuthorized:
        {
            break;
        }
        case AVAuthorizationStatusNotDetermined:
        {
            dispatch_suspend( self.sessionQueue );
            [AVCaptureDevice requestAccessForMediaType:AVMediaTypeVideo completionHandler:^( BOOL granted ) {
                if ( ! granted ) {
                    self.setupResult = AVCamSetupResultCameraNotAuthorized;
                }
                dispatch_resume( self.sessionQueue );
            }];
            break;
        }
        default:
        {
            self.setupResult = AVCamSetupResultCameraNotAuthorized;
            break;
        }
    }
    dispatch_async(self.sessionQueue, ^{
        [self configureSession];
    });
}
- (void)viewWillAppear:(BOOL)animated
{
    [super viewWillAppear:animated];

    dispatch_async( self.sessionQueue, ^{
        switch ( self.setupResult )
        {
            case AVCamSetupResultSuccess:
            {
                [self.session startRunning];
                break;
            }
            case AVCamSetupResultCameraNotAuthorized:
            {
                dispatch_async( dispatch_get_main_queue(), ^{
                    NSString *message = @"请允许拍照权限，以用来扫描二维码";
                    UIAlertController *alertController = [UIAlertController alertControllerWithTitle:@"提示" message:message preferredStyle:UIAlertControllerStyleAlert];
                    UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"确定" style:UIAlertActionStyleCancel handler:nil];
                    [alertController addAction:cancelAction];
                    UIAlertAction *settingsAction = [UIAlertAction actionWithTitle:@"设置" style:UIAlertActionStyleDefault handler:^( UIAlertAction *action ) {
                        [[UIApplication sharedApplication] openURL:[NSURL URLWithString:UIApplicationOpenSettingsURLString] options:@{} completionHandler:nil];
                    }];
                    [alertController addAction:settingsAction];
                    [self presentViewController:alertController animated:YES completion:nil];
                } );
                break;
            }
            case AVCamSetupResultSessionConfigurationFailed:
            {
                dispatch_async( dispatch_get_main_queue(), ^{
                    NSString *message = @"没有拍照权限，所以不能扫描二维码";
                    UIAlertController *alertController = [UIAlertController alertControllerWithTitle:@"提示" message:message preferredStyle:UIAlertControllerStyleAlert];
                    UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"确定" style:UIAlertActionStyleCancel handler:nil];
                    [alertController addAction:cancelAction];
                    [self presentViewController:alertController animated:YES completion:nil];
                } );
                break;
            }
        }
    } );
}

- (void)viewDidDisappear:(BOOL)animated
{
    [super viewDidDisappear:animated];
    dispatch_async( self.sessionQueue, ^{
        if ( self.setupResult == AVCamSetupResultSuccess ) {
            [self.session stopRunning];
        }
    } );
}

- (void)setUpCameraLayer
{
    if (self.previewLayer == nil)
    {
        self.previewLayer = [[AVCaptureVideoPreviewLayer alloc] initWithSession:self.session];

        UIView * view = self.cameraShowView;

        CALayer * viewLayer = [view layer];

        // UIView的clipsToBounds属性和CALayer的setMasksToBounds属性表达的意思是一致的,决定子视图的显示范围。当取值为YES的时候，剪裁超出父视图范围的子视图部分，当取值为NO时，不剪裁子视图。

        [viewLayer setMasksToBounds:YES];

        CGRect bounds = [view bounds];

        [self.previewLayer setFrame:bounds];

        [self.previewLayer setVideoGravity:AVLayerVideoGravityResizeAspect];

        [viewLayer addSublayer:self.previewLayer];
    }
}

/**
 打开设备手电筒
 */
-(void) actionTurnOn
{
    [device lockForConfiguration:nil];

    [device setTorchMode:AVCaptureTorchModeOn];

    [device unlockForConfiguration];
}
/**
 关闭设备手电筒
 */
-(void) actionTurnOff
{
    [device lockForConfiguration:nil];

    [device setTorchMode: AVCaptureTorchModeOff];

    [device unlockForConfiguration];
}



// 拍照

- (void)actionStartCamera
{
    dispatch_async(self.sessionQueue, ^{
        AVCaptureConnection *videoConnection = [self.stillImageOutput connectionWithMediaType:AVMediaTypeVideo];

        if (!videoConnection)
        {
            return;
        }

        [self.stillImageOutput  captureStillImageAsynchronouslyFromConnection:videoConnection completionHandler:^(CMSampleBufferRef imageDataSampleBuffer, NSError *error) {

            if (imageDataSampleBuffer == NULL) {

                return;

            }

            NSData *imageData = [AVCaptureStillImageOutput jpegStillImageNSDataRepresentation:imageDataSampleBuffer];

            UIImage *originalImage = [UIImage imageWithData:imageData];
            //1.旋转照片
            UIImage *rotateImage = [originalImage rotate:UIImageOrientationRight];
    });
}
@end
````
## 注意点
1、AVCaptureSession的配置过程和startRunning是阻挡主线程的一个耗时操作，所以我们放到另外的queue中操作，能够避免阻挡主线程
2、由于我们拍照不需要质量非常高的照片，所以我们通过setOutputSettings设置了图片质量，这样减少了很大一部分内存，可以根据情况来设置图片的质量。
3、拍完照片图片是倒立的，所以我们做了一个旋转操作，将图片正立了过来

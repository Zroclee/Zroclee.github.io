#基于AVPlayer封装视频播放器
## 前言
  闲言少叙，先上重要的知识点！
> 1. AVPlayer本身并不显示视频！需要一个AVPlayerLayer播放层来显示视频,然后添加到父视图的layer中。
> 2. AVPlayer只负责视频管理和调控！而视频资源是由AVPlayerItem提供的，每个AVPlayerItem对应一个视频地址。
> 3. 待补充...


 
> AVplayerItem:

`+ (instancetype)playerItemWithURL:(NSURL *)URL;` 初始化视频资源方法

`duration`视频总时长 （有光CMTime结构体可以自己查下）

`status` 视频资源的状态  （需要监听的属性）

`loadedTimeRanges` 在线视频的缓冲进度 （需要监听的属性）

`playbackBufferEmpty`  进行跳转后没数据 （可选监听）

`playbackLikelyToKeepUp`  进行跳转后有数据  （可选监听）


> AVplayer:
`videoGravity` 视频的填充方式
`rate` 播放速度 常用判断播放状态 =1播放 =0暂停
`- (void)play; - (void)pause;` 播放暂停
`- (void)seekToTime:(CMTime)time;`跳转到指定时间
`- (CMTime)currentTime;`获取当前播放进度
`- (id)addPeriodicTimeObserverForInterval:(CMTime)interval queue:(nullable dispatch_queue_t)queue usingBlock:(void (^)(CMTime time))block;`监听当前播放进度
`- (void)removeTimeObserver:(id)observer;`移除监听 （销毁播放器的时候调用）
`- (void)replaceCurrentItemWithPlayerItem:(nullable AVPlayerItem *)item;`切换视频资源


>AVQueuePlayer:这个是用于处理播放列表的操作，待学习...


##剩下的都在代码里了

###LPAVPlayer.h ###

```
typedef NS_ENUM(NSInteger, LPAVPlayerStatus) {
    LPAVPlayerStatusReadyToPlay = 0, // 准备好播放
    LPAVPlayerStatusLoadingVideo,    // 加载视频
    LPAVPlayerStatusPlayEnd,         // 播放结束
    LPAVPlayerStatusCacheData,       // 缓冲视频
    LPAVPlayerStatusCacheEnd,        // 缓冲结束
    LPAVPlayerStatusPlayStop,        // 播放中断 （多是没网）
    LPAVPlayerStatusItemFailed,      // 视频资源问题
    LPAVPlayerStatusEnterBack,       // 进入后台
    LPAVPlayerStatusBecomeActive,    // 从后台返回
};

@protocol LPAVPlayerDelegate <NSObject>

@optional
// 数据刷新
- (void)refreshDataWith:(NSTimeInterval)totalTime Progress:(NSTimeInterval)currentTime LoadRange:(NSTimeInterval)loadTime;
// 状态/错误 提示
- (void)promptPlayerStatusOrErrorWith:(LPAVPlayerStatus)status;

@end

@interface LPAVPlayer : UIView

@property (nonatomic, weak) id<LPAVPlayerDelegate>delegate;


// 视频总长度
@property (nonatomic, assign) NSTimeInterval totalTime;
// 视频总长度
//@property (nonatomic, assign) NSTimeInterval currentTime;
// 缓存数据
@property (nonatomic, assign) NSTimeInterval loadRange;


/**
 准备播放器
 
 @param videoPath 视频地址
 */
- (void)setupPlayerWith:(NSString *)videoPath;

/** 播放 */
- (void)play;

/** 暂停 */
- (void)pause;

/** 播放|暂停 */
- (void)playOrPause:(void (^)(BOOL isPlay))block;

/** 拖动视频进度 */
- (void)seekPlayerTimeTo:(NSTimeInterval)time;

/** 跳动中不监听 */
- (void)startToSeek;

/**
 切换视频
 
 @param videoPath 视频地址
 */
- (void)replacePalyerItem:(NSString *)videoPath;


@end



```

###LPAVPlayer.h ###

```
#import <AVFoundation/AVFoundation.h>

@interface LPAVPlayer ()

/** 播放器 */
@property (nonatomic, strong) AVPlayer *player;
/** 视频资源 */
@property (nonatomic, strong) AVPlayerItem *currentItem;
/** 播放器观察者 */
@property (nonatomic ,strong)  id timeObser;
// 拖动进度条的时候停止刷新数据
@property (nonatomic ,assign) BOOL isSeeking;
// 是否需要缓冲
@property (nonatomic, assign) BOOL isCanPlay;
// 是否需要缓冲
@property (nonatomic, assign) BOOL needBuffer;

@end

@implementation LPAVPlayer


- (instancetype)initWithFrame:(CGRect)frame
{
    self = [super initWithFrame:frame];
    if (self) {
        
        self.backgroundColor = [UIColor lightGrayColor];
        
        self.isCanPlay = NO;
        self.needBuffer = NO;
        self.isSeeking = NO;
        /**
         * 这里view用来做AVPlayer的容器
         * 完成对AVPlayer的二次封装
         * 要求 : 
         * 1. 暴露视频输出的API  视频时长 当前播放时间 进度
         * 2. 暴露出易于控制的data入口  播放 暂停 进度拖动 音量 亮度 清晰度调节
         */
        
        
    }
    return self;
}

#pragma mark - 属性和方法
- (NSTimeInterval)totalTime
{
    return CMTimeGetSeconds(self.player.currentItem.duration);
}



/** 
 准备播放器
 
 @param videoPath 视频地址
 */
- (void)setupPlayerWith:(NSString *)videoPath
{
    [self creatPlayer:videoPath];
    
    [_player play];
    [self useDelegateWith:LPAVPlayerStatusLoadingVideo];
}

/**
 avplayer自身有一个rate属性
 rate ==1.0，表示正在播放；rate == 0.0，暂停；rate == -1.0，播放失败
 */

/** 播放 */
- (void)play
{
    if (self.player.rate == 0) {
        [self.player play];
    }
}

/** 暂停 */
- (void)pause
{
    if (self.player.rate == 1.0) {
        [self.player pause];
    }
}

/** 播放|暂停 */
- (void)playOrPause:(void (^)(BOOL isPlay))block;
{
    if (self.player.rate == 0) {
        
        [self.player play];
        
        block(YES);
        
    }else if (self.player.rate == 1.0) {
        
        [self.player pause];
        
        block(NO);
        
    }else {
        NSLog(@"播放器出错");
    }
}

/** 拖动视频进度 */
- (void)seekPlayerTimeTo:(NSTimeInterval)time
{
    [self pause];
    [self startToSeek];
    __weak typeof(self)weakSelf = self;
    
    [self.player seekToTime:CMTimeMake(time, 1.0) completionHandler:^(BOOL finished) {
        [weakSelf endSeek];
        [weakSelf play];
    }];
    
}

/** 跳动中不监听 */
- (void)startToSeek
{
    self.isSeeking = YES;
}
- (void)endSeek
{
    self.isSeeking = NO;
}

/**
 切换视频
 
 @param videoPath 视频地址
 */
- (void)replacePalyerItem:(NSString *)videoPath
{
    self.isCanPlay = NO;
    
    [self pause];
    [self removeNotification];
    [self removeObserverWithPlayItem:self.currentItem];
    
    self.currentItem = [self getPlayerItem:videoPath];
    [self.player replaceCurrentItemWithPlayerItem:self.currentItem];
    [self addObserverWithPlayItem:self.currentItem];
    [self addNotificatonForPlayer];
    
    [self play];
    
}



- (void)useDelegateWith:(LPAVPlayerStatus)status
{
    if (self.isCanPlay == NO) {
        return;
    }
    
    if (self.delegate && [self.delegate respondsToSelector:@selector(promptPlayerStatusOrErrorWith:)]) {
        [self.delegate promptPlayerStatusOrErrorWith:status];
    }
}


#pragma mark - 创建播放器
/**
 获取播放item
 
 @param videoPath 视频网址
 
 @return AVPlayerItem 
 */
- (AVPlayerItem *)getPlayerItem:(NSString *)videoPath
{
    // 转utf8 防止中文报错
//    videoPath = [videoPath stringByAddingPercentEscapesUsingEncoding:NSUTF8StringEncoding];
//    videoPath = [videoPath stringByAddingPercentEncodingWithAllowedCharacters:[NSCharacterSet URLQueryAllowedCharacterSet]];
    
    NSURL *url = [NSURL URLWithString:videoPath];
    
    AVPlayerItem *item = [AVPlayerItem playerItemWithURL:url];
    
    return item;
}


/**
  创建播放器
 
 
 */
- (void)creatPlayer:(NSString *)videoPath
{
    if (!_player) {
        
        self.currentItem = [self getPlayerItem:videoPath];
        
        _player = [AVPlayer playerWithPlayerItem:self.currentItem];
        
        [self creatPlayerLayer];
        
        [self addPlayerObserver];
        
        [self addObserverWithPlayItem:self.currentItem];
        
        [self addNotificatonForPlayer];
    }
}

/**
 创建播放器 layer 层
 AVPlayerLayer的videoGravity属性设置
 AVLayerVideoGravityResize,       // 非均匀模式。两个维度完全填充至整个视图区域
 AVLayerVideoGravityResizeAspect,  // 等比例填充，直到一个维度到达区域边界
 AVLayerVideoGravityResizeAspectFill, // 等比例填充，直到填充满整个视图区域，其中一个维度的部分区域会被裁剪
 */
- (void)creatPlayerLayer
{
    AVPlayerLayer *layer = [AVPlayerLayer playerLayerWithPlayer:self.player];
    
    layer.frame = self.bounds;
    
    layer.videoGravity = AVLayerVideoGravityResizeAspect;
    
    [self.layer addSublayer:layer];
}

#pragma mark - 添加 监控
/** 给player 添加 time observer */
- (void)addPlayerObserver
{
    __weak typeof(self)weakSelf = self;
    _timeObser = [self.player addPeriodicTimeObserverForInterval:CMTimeMake(1.0, 1.0) queue:dispatch_get_main_queue() usingBlock:^(CMTime time) {
        AVPlayerItem *playerItem = weakSelf.player.currentItem;
        
        float current = CMTimeGetSeconds(time);
        
        float total = CMTimeGetSeconds([playerItem duration]);
        
        if (weakSelf.isSeeking) {
            return;
        }
        
        if (weakSelf.delegate && [weakSelf.delegate respondsToSelector:@selector(refreshDataWith:Progress:LoadRange:)]) {
            [weakSelf.delegate refreshDataWith:total Progress:current LoadRange:weakSelf.loadRange];
        }
//        NSLog(@"当前播放进度 %.2f/%.2f.",current,total);
        
    }];
}
/** 移除 time observer */
- (void)removePlayerObserver
{
    [_player removeTimeObserver:_timeObser];
}

/** 给当前播放的item 添加观察者 
 
 需要监听的字段和状态
 status :  AVPlayerItemStatusUnknown,AVPlayerItemStatusReadyToPlay,AVPlayerItemStatusFailed
 loadedTimeRanges  :  缓冲进度
 playbackBufferEmpty : seekToTime后，缓冲数据为空，而且有效时间内数据无法补充，播放失败
 playbackLikelyToKeepUp : seekToTime后,可以正常播放，相当于readyToPlay，一般拖动滑竿菊花转，到了这个这个状态菊花隐藏
 
 */
- (void)addObserverWithPlayItem:(AVPlayerItem *)item
{
    [item addObserver:self forKeyPath:@"status" options:NSKeyValueObservingOptionNew context:nil];
    [item addObserver:self forKeyPath:@"loadedTimeRanges" options:NSKeyValueObservingOptionNew context:nil];
    [item addObserver:self forKeyPath:@"playbackBufferEmpty" options:NSKeyValueObservingOptionNew context:nil];
    [item addObserver:self forKeyPath:@"playbackLikelyToKeepUp" options:NSKeyValueObservingOptionNew context:nil];
}
/** 移除 item 的 observer */
- (void)removeObserverWithPlayItem:(AVPlayerItem *)item
{
    [item removeObserver:self forKeyPath:@"status"];
    [item removeObserver:self forKeyPath:@"loadedTimeRanges"];
    [item removeObserver:self forKeyPath:@"playbackBufferEmpty"];
    [item removeObserver:self forKeyPath:@"playbackLikelyToKeepUp"];
}
/** 数据处理 获取到观察到的数据 并进行处理 */
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context
{
    AVPlayerItem *item = object;
    if ([keyPath isEqualToString:@"status"]) {// 播放状态
        
        [self handleStatusWithPlayerItem:item];
        
    } else if ([keyPath isEqualToString:@"loadedTimeRanges"]) {// 缓冲进度
        
        [self handleLoadedTimeRangesWithPlayerItem:item];
        
    } else if ([keyPath isEqualToString:@"playbackBufferEmpty"]) {// 跳转后没数据
        
        // 转菊花
        if (self.isCanPlay) {
            NSLog(@"跳转后没数据");
            self.needBuffer = YES;
            [self useDelegateWith:LPAVPlayerStatusCacheData];
        }
        
        
        
    } else if ([keyPath isEqualToString:@"playbackLikelyToKeepUp"]) {// 跳转后有数据
        
        
        
        // 隐藏菊花
        if (self.isCanPlay && self.needBuffer) {
            
            NSLog(@"跳转后有数据");
            
            self.needBuffer = NO;
            
            [self useDelegateWith:LPAVPlayerStatusCacheEnd];
        }
        
    }
}
/**
  处理 AVPlayerItem 播放状态
 AVPlayerItemStatusUnknown           状态未知
 AVPlayerItemStatusReadyToPlay       准备好播放
 AVPlayerItemStatusFailed            播放出错
 */
- (void)handleStatusWithPlayerItem:(AVPlayerItem *)item
{
    AVPlayerItemStatus status = item.status;
    switch (status) {
        case AVPlayerItemStatusReadyToPlay:   // 准备好播放
            
            NSLog(@"AVPlayerItemStatusReadyToPlay");
            self.isCanPlay = YES;
            [self useDelegateWith:LPAVPlayerStatusReadyToPlay];
            
            break;
        case AVPlayerItemStatusFailed:        // 播放出错
            
            NSLog(@"AVPlayerItemStatusFailed");
            [self useDelegateWith:LPAVPlayerStatusItemFailed];
            
            break;
        case AVPlayerItemStatusUnknown:       // 状态未知
            
            NSLog(@"AVPlayerItemStatusUnknown");
            
            break;
            
        default:
            break;
    }
    
}
/** 处理缓冲进度 */
- (void)handleLoadedTimeRangesWithPlayerItem:(AVPlayerItem *)item
{
    NSArray *loadArray = item.loadedTimeRanges;
    
    CMTimeRange range = [[loadArray firstObject] CMTimeRangeValue];
    
    float start = CMTimeGetSeconds(range.start);
    
    float duration = CMTimeGetSeconds(range.duration);
    
    NSTimeInterval totalTime = start + duration;// 缓存总长度
    
    _loadRange = totalTime;
//    NSLog(@"缓冲进度 -- %.2f",totalTime);
    
}


/**
 添加关键通知
 
 AVPlayerItemDidPlayToEndTimeNotification     视频播放结束通知
 AVPlayerItemTimeJumpedNotification           视频进行跳转通知
 AVPlayerItemPlaybackStalledNotification      视频异常中断通知
 UIApplicationDidEnterBackgroundNotification  进入后台
 UIApplicationDidBecomeActiveNotification     返回前台
 
 */
- (void)addNotificatonForPlayer
{
    NSNotificationCenter *center = [NSNotificationCenter defaultCenter];
    
    [center addObserver:self selector:@selector(videoPlayEnd:) name:AVPlayerItemDidPlayToEndTimeNotification object:nil];
//    [center addObserver:self selector:@selector(videoPlayToJump:) name:AVPlayerItemTimeJumpedNotification object:nil];//没意义
    [center addObserver:self selector:@selector(videoPlayError:) name:AVPlayerItemPlaybackStalledNotification object:nil];
    [center addObserver:self selector:@selector(videoPlayEnterBack:) name:UIApplicationDidEnterBackgroundNotification object:nil];
    [center addObserver:self selector:@selector(videoPlayBecomeActive:) name:UIApplicationDidBecomeActiveNotification object:nil];
}
/** 移除 通知 */
- (void)removeNotification
{
    NSNotificationCenter *center = [NSNotificationCenter defaultCenter];
    
    [center removeObserver:self name:AVPlayerItemDidPlayToEndTimeNotification object:nil];
//    [center removeObserver:self name:AVPlayerItemTimeJumpedNotification object:nil];
    [center removeObserver:self name:AVPlayerItemPlaybackStalledNotification object:nil];
    [center removeObserver:self name:UIApplicationDidEnterBackgroundNotification object:nil];
    [center removeObserver:self name:UIApplicationDidBecomeActiveNotification object:nil];
    [center removeObserver:self];
}

/** 视频播放结束 */
- (void)videoPlayEnd:(NSNotification *)notic
{
    NSLog(@"视频播放结束");
    [self useDelegateWith:LPAVPlayerStatusPlayEnd];
    [self.player seekToTime:kCMTimeZero];
}
///** 视频进行跳转 */ 没有意义的方法 会被莫名的多次调动，不清楚机制
//- (void)videoPlayToJump:(NSNotification *)notic
//{
//    NSLog(@"视频进行跳转");
//}
/** 视频异常中断 */
- (void)videoPlayError:(NSNotification *)notic
{
    NSLog(@"视频异常中断");
    [self useDelegateWith:LPAVPlayerStatusPlayStop];
}
/** 进入后台 */
- (void)videoPlayEnterBack:(NSNotification *)notic
{
    NSLog(@"进入后台");
    [self useDelegateWith:LPAVPlayerStatusEnterBack];
}
/** 返回前台 */
- (void)videoPlayBecomeActive:(NSNotification *)notic
{
    NSLog(@"返回前台");
    [self useDelegateWith:LPAVPlayerStatusBecomeActive];
}

#pragma mark - 销毁 release
- (void)dealloc
{
    NSLog(@"--- %@ --- 销毁了",[self class]);
    
    [self removeNotification];
    [self removePlayerObserver];
    [self removeObserverWithPlayItem:self.player.currentItem];
    
}




/*
// Only override drawRect: if you perform custom drawing.
// An empty implementation adversely affects performance during animation.
- (void)drawRect:(CGRect)rect {
    // Drawing code
}
*/

@end



```



## 四、 在控制器中的调用
 
 因为控制器还有一些其他的代码，所以就整个贴出来了，截取一部分重要的贴出来。
 
 ###1. 创建播放器View ###
 
 ```
 /**
  创建播放器视图
 */
- (void)buildPlayerView
{
    CGRect rect = CGRectMake(0, 64, self.view.bounds.size.width, 250);
    
    _playView = [[LPAVPlayer alloc] initWithFrame:rect];
    
    _playView.delegate = self;
    
    [self.view addSubview:_playView];
    
    [_playView setupPlayerWith:videoPath1];
    
    
}
 
 ```
 
  ###2. 代理 -- 状态提示 ###
  
  ```
  // LPAVPlayer delegate  ----- 状态提示
- (void)promptPlayerStatusOrErrorWith:(LPAVPlayerStatus)status
{
    switch (status) {
        case LPAVPlayerStatusLoadingVideo:// 开始准备
            
            [self.activity startAnimating];
            
            break;
            
        case LPAVPlayerStatusReadyToPlay:// 准备完成
            
            [self.activity stopAnimating];
            [self.playOrPause setTitle:@"暂停" forState:UIControlStateNormal];
            self.slider.maximumValue = _playView.totalTime;
            self.slider.value = 0;
            self.loadProgress.progress = 0;
            
            break;
            
            
            
        default:
            break;
    }
}
  
  ```
  
  ###3. 代理 -- 刷新数据 ###
  
  ```
  // LPAVPlayer delegate  ----- 刷新数据
- (void)refreshDataWith:(NSTimeInterval)totalTime Progress:(NSTimeInterval)currentTime LoadRange:(NSTimeInterval)loadTime
{
    
    [self.loadProgress setProgress:(loadTime/totalTime) animated:YES];
    
    [self.slider setValue:currentTime animated:YES];
    
    self.timeLabel.text = [NSString stringWithFormat:@"%@/%@",[self otherConvertTimeFormat:currentTime],[self otherConvertTimeFormat:totalTime]];
}

- (NSString *)otherConvertTimeFormat:(NSInteger)time
{
    NSString *needStr = @"";
    
    if (time < 3600) {
        
        needStr =  [NSString stringWithFormat:@"%02li:%02li",lround(floor(time/60.f)),lround(floor(time/1.f))%60];
        
    } else {
        
        needStr =  [NSString stringWithFormat:@"%02li:%02li:%02li",lround(floor(time/3600.f)),lround(floor(time%3600)/60.f),lround(floor(time/1.f))%60];
        
    }
    return needStr;
}
  
  ```
 



## 一、对playerItem的监听
> 需要监听的字段和状态
> 1.status: 播放状态AVPlayerItemStatusUnknown,AVPlayerItemStatusReadyToPlay,AVPlayerItemStatusFailed
> 2.loadedTimeRanges  缓冲进度
> 3.playbackBufferEmpty  seekToTime后，缓冲数据为空，而且有效时间内数据无法补充，播放失败
> 4. playbackLikelyToKeepUp  seekToTime后,可以正常播放，相当于readyToPlay，一般拖动滑竿菊花转，到了这个这个状态菊花隐藏



```
/** 给当前播放的item 添加观察者 
 
 需要监听的字段和状态
 status :  AVPlayerItemStatusUnknown,AVPlayerItemStatusReadyToPlay,AVPlayerItemStatusFailed
 loadedTimeRanges  :  缓冲进度
 playbackBufferEmpty : seekToTime后，缓冲数据为空，而且有效时间内数据无法补充，播放失败
 playbackLikelyToKeepUp : seekToTime后,可以正常播放，相当于readyToPlay，一般拖动滑竿菊花转，到了这个这个状态菊花隐藏
 
 */
- (void)addObserverWithPlayItem:(AVPlayerItem *)item
{
    [item addObserver:self forKeyPath:@"status" options:NSKeyValueObservingOptionNew context:nil];
    [item addObserver:self forKeyPath:@"loadedTimeRanges" options:NSKeyValueObservingOptionNew context:nil];
    [item addObserver:self forKeyPath:@"playbackBufferEmpty" options:NSKeyValueObservingOptionNew context:nil];
    [item addObserver:self forKeyPath:@"playbackLikelyToKeepUp" options:NSKeyValueObservingOptionNew context:nil];
}
/** 移除 item 的 observer */
- (void)removeObserverWithPlayItem:(AVPlayerItem *)item
{
    [item removeObserver:self forKeyPath:@"status"];
    [item removeObserver:self forKeyPath:@"loadedTimeRanges"];
    [item removeObserver:self forKeyPath:@"playbackBufferEmpty"];
    [item removeObserver:self forKeyPath:@"playbackLikelyToKeepUp"];
}
/** 数据处理 获取到观察到的数据 并进行处理 */
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context
{
    AVPlayerItem *item = object;
    if ([keyPath isEqualToString:@"status"]) {// 播放状态
        
        [self handleStatusWithPlayerItem:item];
        
    } else if ([keyPath isEqualToString:@"loadedTimeRanges"]) {// 缓冲进度
        
        [self handleLoadedTimeRangesWithPlayerItem:item];
        
    } else if ([keyPath isEqualToString:@"playbackBufferEmpty"]) {// 跳转后没数据
        
        // 转菊花
        
    } else if ([keyPath isEqualToString:@"playbackLikelyToKeepUp"]) {// 跳转后有数据
        
        // 隐藏菊花
        
    }
}
/**
  处理 AVPlayerItem 播放状态
 AVPlayerItemStatusUnknown           状态未知
 AVPlayerItemStatusReadyToPlay       准备好播放
 AVPlayerItemStatusFailed            播放出错
 */
- (void)handleStatusWithPlayerItem:(AVPlayerItem *)item
{
    AVPlayerItemStatus status = item.status;
    switch (status) {
        case AVPlayerItemStatusReadyToPlay:   // 准备好播放
            
            break;
        case AVPlayerItemStatusFailed:        // 播放出错
            
            break;
        case AVPlayerItemStatusUnknown:       // 状态未知
            
            break;
            
        default:
            break;
    }
    
}
/** 处理缓冲进度 */
- (void)handleLoadedTimeRangesWithPlayerItem:(AVPlayerItem *)item
{
    NSArray *loadArray = item.loadedTimeRanges;
    
    CMTimeRange range = [[loadArray firstObject] CMTimeRangeValue];
    
    float start = CMTimeGetSeconds(range.start);
    
    float duration = CMTimeGetSeconds(range.duration);
    
    NSTimeInterval totalTime = start + duration;// 缓存总长度
    
    
}

```

## 二、 Player自带的time observer方法
> \- (id)addPeriodicTimeObserverForInterval:(CMTime)interval queue:(nullable dispatch_queue_t)queue usingBlock:(void (^)(CMTime time))block;

```
/** 给player 添加 time observer */
- (void)addPlayerObserver
{
    __weak typeof(self)weakSelf = self;
    _timeObser = [self.player addPeriodicTimeObserverForInterval:CMTimeMake(1.0, 1.0) queue:dispatch_get_main_queue() usingBlock:^(CMTime time) {
        AVPlayerItem *playerItem = weakSelf.player.currentItem;
        
        float current = CMTimeGetSeconds(time);
        
        float total = CMTimeGetSeconds([playerItem duration]);
        
        if (weakSelf.isSeeking) {
            return;
        }
        
        if (weakSelf.delegate && [weakSelf.delegate respondsToSelector:@selector(refreshDataWith:Progress:LoadRange:)]) {
            [weakSelf.delegate refreshDataWith:total Progress:current LoadRange:weakSelf.loadRange];
        }
//        NSLog(@"当前播放进度 %.2f/%.2f.",current,total);
        
    }];
}
/** 移除 time observer */
- (void)removePlayerObserver
{
    [_player removeTimeObserver:_timeObser];
}

```

## 三、 avplayer有关的通知

 > AVPlayerItemDidPlayToEndTimeNotification     视频播放结束通知
 > AVPlayerItemTimeJumpedNotification           视频进行跳转通知
 > AVPlayerItemPlaybackStalledNotification      视频异常中断通知
 > UIApplicationDidEnterBackgroundNotification  进入后台
 > UIApplicationDidBecomeActiveNotification     返回前台

```
- (void)addNotificatonForPlayer
{
    NSNotificationCenter *center = [NSNotificationCenter defaultCenter];
    
    [center addObserver:self selector:@selector(videoPlayEnd:) name:AVPlayerItemDidPlayToEndTimeNotification object:nil];
//    [center addObserver:self selector:@selector(videoPlayToJump:) name:AVPlayerItemTimeJumpedNotification object:nil];//没意义
    [center addObserver:self selector:@selector(videoPlayError:) name:AVPlayerItemPlaybackStalledNotification object:nil];
    [center addObserver:self selector:@selector(videoPlayEnterBack:) name:UIApplicationDidEnterBackgroundNotification object:nil];
    [center addObserver:self selector:@selector(videoPlayBecomeActive:) name:UIApplicationDidBecomeActiveNotification object:nil];
}
/** 移除 通知 */
- (void)removeNotification
{
    NSNotificationCenter *center = [NSNotificationCenter defaultCenter];
    
    [center removeObserver:self name:AVPlayerItemDidPlayToEndTimeNotification object:nil];
//    [center removeObserver:self name:AVPlayerItemTimeJumpedNotification object:nil];
    [center removeObserver:self name:AVPlayerItemPlaybackStalledNotification object:nil];
    [center removeObserver:self name:UIApplicationDidEnterBackgroundNotification object:nil];
    [center removeObserver:self name:UIApplicationDidBecomeActiveNotification object:nil];
    [center removeObserver:self];
}

/** 视频播放结束 */
- (void)videoPlayEnd:(NSNotification *)notic
{
    NSLog(@"视频播放结束");
    [self useDelegateWith:LPAVPlayerStatusPlayEnd];
    [self.player seekToTime:kCMTimeZero];
}
///** 视频进行跳转 */ 没有意义的方法 会被莫名的多次调动，不清楚机制
//- (void)videoPlayToJump:(NSNotification *)notic
//{
//    NSLog(@"视频进行跳转");
//}
/** 视频异常中断 */
- (void)videoPlayError:(NSNotification *)notic
{
    NSLog(@"视频异常中断");
    [self useDelegateWith:LPAVPlayerStatusPlayStop];
}
/** 进入后台 */
- (void)videoPlayEnterBack:(NSNotification *)notic
{
    NSLog(@"进入后台");
    [self useDelegateWith:LPAVPlayerStatusEnterBack];
}
/** 返回前台 */
- (void)videoPlayBecomeActive:(NSNotification *)notic
{
    NSLog(@"返回前台");
    [self useDelegateWith:LPAVPlayerStatusBecomeActive];
}

```

## 四、本文参考的文章

>  [作者：夜千寻墨 - AVPlayer](http://www.jianshu.com/p/95134a35d474)
>  [作者：SomeBoy - iOS开发AVPlayer深入浅出](http://www.jianshu.com/p/5016b72c52bd)
>  [作者：KenshinCui iOS开发系列--音频播放、录音、视频播放、拍照、视频录制](http://www.cnblogs.com/kenshincui/p/4186022.html#audioSession)


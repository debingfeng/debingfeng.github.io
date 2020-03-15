---
sidebarDepth : 0
---

最近因为项目上用到音乐播放，就仔细研究了一下html5 audio API,利用国庆休息的时间，进行了一些总结，有些坑还没填好已经备注文档中。
这篇文章主要是介绍一些基本使用，**下一篇将主要与大家分享audio在各个浏览器和设备上存在的问题以及如何去解决。**

[查看demo演示以及源码使用](http://demo.fengdb.com/html5/audio.html)


### 常用属性

属性 | 作用 |
---|---|----
src | 设置或返回是否在就绪（加载完成）后随即播放音频  
    currentSrc | 返回当前音频的 URL。
currentTime | 设置或返回音频中的当前播放位置（以秒计）。
duration | 返回音频的长度（以秒计）。
readyState | 返回音频当前的就绪状态。
networkState | 返回音频的当前网络状态。


### 功能播放设置

属性  | 值 | 作用 
---|---|---------
paused | bool | 设置或返回音频是否暂停。
ended | bool | 返回音频的播放是否已结束。
muted | bool |设置或返回是否关闭声音。
controls| bool| 设置或返回音频是否应该显示控件（比如播放/暂停等）。
loop | bool |设置或返回音频是否应在结束时再次播放。
autoplay |bool| 设置或返回是否在就绪（加载完成）后随即播放音频。
preload |bool| 设置或返回音频的 preload 属性的值。
volume | 范围 0-1 |设置或返回音频的音量。
playbackRate | 1.0/2.0倍速度 -2后退两倍速度 | 设置或返回音频/视频播放的速度(留下一个坑 负值不起作用)

### 常用方法

名称 | 作用
---|---
canPlayType() | 查浏览器是否可以播放指定的音频类型 "probably" - 浏览器最可能支持该音频/视频类型，"maybe" - 浏览器也许支持该音频/视频类型，"" - （空字符串）浏览器不支持该音频/视频类型
fastSeek() | 在音频播放器中指定播放时间
load() | 重新加载音频元素
play() | 开始播放音频
pause() | 暂停当前播放的音频

### 常用事件


事件名称 | 事件描述
---|---
loadstart | 客户端开始请求数据
progress | 客户端正在请求数据（或者说正在缓冲）
play | 播放中
pause | 暂停
ended | 播放结束
timeupdate | 当前播放时间发生改变的时候。常用作显示进度
canplaythrough | 歌曲已经载入完全完成
canplay | 缓冲至目前可播放状态。
error | 播放发生错误时。



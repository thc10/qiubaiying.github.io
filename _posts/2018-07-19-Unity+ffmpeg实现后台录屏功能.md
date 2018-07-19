---
layout:     post
title:      Unity + ffmpeg 实现后台录屏功能
date:       2018-07-18
anthor:     hatcherTang
header-img: img/post-bg-ios9-web.jpg
catalog:    true
tags:
    - unity
    - ffmpeg
    - console
---

---

# ffmpeg介绍

FFmpeg是一个开源免费跨平台的视频和音频流方案，提供了录制、转换以及流化音视频的完整解决方案。FFmpeg在Linux平台下开发，但它同样也可以在其它操作系统环境中编译运行，包括Windows、Mac OS X等。

### ffmpeg下载

可以通过官网的 [**下载地址**](https://www.ffmpeg.org/download.html) 免费下载。
在windows环境下将下载得到的压缩包解压后可以得到三个文件夹（bin、doc和presets）以及两个说明文件（LICENSE.txt和README.txt）,在本次实验中主要用到的是bin目录下的ffmpeg.exe程序。

### ffmpeg的设备数据
在命令行中输入
```C
ffmpeg -list_devices true -f dshow -i dummy
```
可以列出所有的设备名称，也可以通过 **-f dshow -i video="{设备名}"** 来查看详细信息。
在本次实验中用到的设备是"Screen-Capture-Record"。

### ffmpeg录屏方法
```
/** ffmpeg参数说明
     * -f ：格式
     *     gdigrab ：ffmpeg内置的用于抓取Windows桌面的方法，支持抓取指定名称的窗口
     *     dshow ：依赖于第三方软件Screen Capture Recorder（后面简称SCR）
     * -i ：输入源
     *     title ：要录制的窗口的名称，仅用于GDIGRAB方式
     *     video ：视频播放硬件名称或者"screen-capture-recorder"，后者依赖SCR
     *     audio ：音频播放硬件名称或者"virtual-audio-capturer"，后者依赖SCR
     * -preset ultrafast ：以最快的速度进行编码，生成的视频文件大
     * -c:v ：视频编码方式
     * -c:a ：音频编码方式
     * -b:v ：视频比特率
     * -r ：视频帧率
     * -s ：视频分辨率
     * -y ：输出文件覆盖已有文件不提示
     * -q : 视频品质
     * 
     * FFMPEG官方文档：http://ffmpeg.org/ffmpeg-all.html
     * Screen Capture Recorder主页：https://github.com/rdp/screen-capture-recorder-to-video-windows-free
     */
```
在使用ffmpeg的录屏使用时，有很多参数可以进行设置，以上只是部分的参数列表。以下是一个录屏开始的命令：
```
ffmpeg -f dshow -i video="screen-capture-recorder" -q 10 temp_files/2.avi
```

----

# Unity中调用ffmpeg录屏

Unity中调用ffmpeg录屏的流程是（以播放游戏视频为例）：
**获取到待播放的视频文件信息** -> **打开ffmpeg开始录屏** -> **播放视频** -> **视频播放完毕** -> **找到正在录屏的ffmpeg进程** -> **关闭进程**

### 开始录屏
首先获取得到录像对应的地图ID和时间戳，组成保存文件的文件名。
```
uint MapID = mCurVideoSummaryInfo.MapID;
long Timestamp = mCurVideoSummaryInfo.VideoMark;
fileName = string.Format("{0}_{1}.avi", MapID, Timestamp);
```
如果直接在主线程中启动录屏软件会造成主线程的阻塞，因此采取创建一个新的线程运行ffmpeg的方式（可根据实际的应用场景来选择是否创建新的线程）。
Unity中操作控制台需要借助Process类，对应的名字空间是System.Diagnostics。Process对象的设置如下所示：
```
Process ffmpegProcess = new Process ();
ffmpegProcess.StartInfo.FileName = ffPath;	// 可执行文件路径
ffmpegProcess.StartInfo.Arguments = ffArgs + fileName; // 传给可执行文件的命令行参数
// 通过下列两个属性的设置控制程序执行时不新建控制台窗口，即在后台执行
ffmpegProcess.StartInfo.CreateNoWindow = true;	// 是否显示控制台窗口
ffmpegProcess.StartInfo.UseShellExecute = false; // 是否使用操作系统Shell启动
ffmpegProcess.Start ();
```
在Start后，需要记录下当前Process的id，以便后续向这个Process发送停止录制的指令。
```
// 记录下ID以便停止录屏时查找对应的控制台
recordId = ffmpegProcess.Id;
isRecording = true;
```

### 停止录制
停止录制需要用到的几个函数需要添加依赖库，这些函数对应的名字空间是：System.Runtime.InteropServices，此处的实现如下所示：
```
#region 模拟控制台信号需要使用DLL
[DllImport("kernel32.dll")]
static extern bool GenerateConsoleCtrlEvent(int dwCtrlEvent, int dwProcessGroupId);
[DllImport("kernel32.dll")]
static extern bool SetConsoleCtrlHandler(IntPtr handlerRoutine, bool add);
[DllImport("kernel32.dll")]
static extern bool AttachConsole(int dwProcessId);
[DllImport("kernel32.dll")]
static extern bool FreeConsole();
#endregion
```
首先调用AttachConsole函数附加到当前正在录屏的Process上。因为需要向ffmpeg发送一个"Ctrl + C"指令来停止录屏，而Unity同样会响应这一信号并关闭程序，因此我们需要设置Unity不对这一信号进行响应。设置完成后，向ffmpeg所在的Process发送一个"Ctrl + C"信号，发送完后需要空转一段时间，等待ffmpeg结束运行，否则会导致录制得到的视频无法正常播放。
```
// 将当前进程附加到pid对应进程的控制台
AttachConsole (recordId);

// 将控制台事件的处理句柄设为Zero，即当前进程不响应控制台事件
// 避免在向控制台发送"Ctrl + C"指令时连带当前进程一起结束
SetConsoleCtrlHandler (IntPtr.Zero, true);

// 向控制台发送"Ctrl + C"指令以停止录制 
GenerateConsoleCtrlEvent (0, 0);

// ffmpeg不能立即停止，等待一会，否则视频无法播放
for (int i = 0; i < 10; ++i)
{
	yield return null;
}

// 卸载控制台事件的处理句柄，否则之后的ffmpeg调用无法正常停止
SetConsoleCtrlHandler (IntPtr.Zero, false);

// 剥离已附加的控制台
FreeConsole ();

isRecording = false;
```

### 程序异常退出处理
因为上述流程只有当视频播放完毕后才会停止录像，所以在录制视频过程中直接推出程序时会导致正在后台运行的ffmpeg一直在录像，占用大量电脑资源。因此需要在OnDestroy过程中对未停止的录像进行一个手动的停止。需要注意的时，当OnDestroy函数被调用时，不能通过协程的方式调用录像停止的函数。重载后的OnDestroy函数如下所示。
```
private void OnDestroy(){
	// 如果退出的时候正在录像，则需要停止录像
	if (isSaveVideos && isRecording) 
	{
		// StartCoroutine(RecordStop());
		AttachConsole (recordId);
		SetConsoleCtrlHandler (IntPtr.Zero, true);
		GenerateConsoleCtrlEvent (0, 0);

		// 杀死已有的ffmpeg进程
		Process[] processes = Process.GetProcessesByName ("ffmpeg");
		foreach (Process p in processes)
		{
			p.Kill ();
		}
		SetConsoleCtrlHandler (IntPtr.Zero, false);
		FreeConsole ();
		isRecording = false;
	}
}
```

# 录制参数的使用
在默认情况下，视频的品质（即清晰度）、录屏文件的存放位置和是否开启录像都是不能手动设置的，程序十分死板。因此修改程序使得程序能够接受命令行参数。
```
private readonly string ffArgsFormat = " -f dshow -i video=\"screen-capture-recorder\" -q {0} {1}";
// 读取录屏的一些自定参数
isSaveVideos = !ParamManager.GetInstance ().IsContain ("disableSaveVideo");
videoQuality = ParamManager.GetInstance ().GetInt ("q", 10);
savePath = ParamManager.GetInstance().GetString("savePath", Application.dataPath + "/../Record_Videos/");
if (!Directory.Exists (savePath)) 
{
	Directory.CreateDirectory (savePath);
}
ffArgs = string.Format(ffArgsFormat, videoQuality, savePath);
```
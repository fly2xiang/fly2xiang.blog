title: "在高分屏下使用 Windows 10 操作系统"
date: 2016-12-01 16:22:32
tags:
- Windows 10
- High DPI
- 4K 屏
- DPIMangler
categories: 
- 使用技巧

---

## 问题

截至到现在（2016年末），在高分辨率屏幕（3K、4K屏幕）下使用 Windows 10 操作系统仍旧会遇到不兼容高分屏的软件，表现为字体极小或界面错乱。在网上找到了几个解决方案，经过试用能够解决问题，现在整理一下。

<!-- more -->

## 解决方案

解决方案按照推荐使用的顺序来写。

### 提取并修改 mainfest 文件

首先要做的是修改注册表，将下面的代码保存为 `.reg` 文件，然后双击导入注册表。也可以手动定位到对应的位置进行添加或修改。这一步主要是为了让软件在运行是优先使用外部的 mainfest 文件。

```reg
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\SideBySide]
"PreferExternalManifest"=dword:00000001

```

然后下载 [ResourcesExtract](http://www.nirsoft.net/utils/resources_extract.html) 软件用于提取 exe 文件中的 mainfest 文件。

下载后执行，选择 exe 文件和要保存 mainfest 文件的位置，然后勾选 mainfest 资源，其他的不选，点击 开始(start)，应该就能够提取出 mainfest 文件了。

打开提取出的 mainfest 文件，可以看出这是一个 XML 文件，查找 dpiAware 这个标签，将此标签内部的值改为 false，保存。

将 mainfest 文件重命名为 `程序名.exe.mainfest`，例如 Photoshop (此篇文章撰写时Photoshop已兼容高分屏下的Windows 10，此处只是举例说明) 的可执行文件为 `photoshop.exe`，则将 mainfest 重命名为 `photoshop.exe.mainfest`。

最后将 mainfest 文件放到与可执行文件相同的文件夹下，此时再打开可执行程序应该就没有问题了。

### DPIMangler + CFF Explorer

如果上面的步骤中出错了（提取不出 mainfest 或提前出的 mainfest 文件没有 dpiAware 标签），或者最后不生效可以使用此方法。

首先下载 [DPIMangler](https://github.com/GeorgeHahn/DPIMangler/releases) 和 [CFF Explorer](http://www.ntcore.com/exsuite.php) 。

DPIMangler 是一个 dll 钩子，钩住了应用程序向 Windows 系统设置 DPI 的系统调用。CFF Explorer 可以修改应用程序调用系统函数的地址。

首先解压 DPIMangler.zip 得到 DPIMangler.x64.dll 和 DPIMangler.x86.dll，分别对应 64 位和 32 位的应用程序，选择对应的 dll 文件，放到应用程序同一个文件夹下。

然后用 CFF Explorer 打开应用程序 exe 文件，点击左侧菜单的 Import Addr，点击右侧的 Add，在弹出的文件选择框中选择上一步选择的 dll 文件。

此时下方的左侧的列表框中应该出现了一些内容，全选他们，他们会出现在右侧的列表框中。

最后点击 Rebuild Import Table ，提示 success 后点击菜单栏的保存按钮，提示你是否要覆盖原有文件，如果不放心可以将原来的 exe 文件备份后再保存。

此时再打开可执行程序应该就没有问题了。

### TouchZoomDesktop

一个放大镜程序，比 Windows 自带的放大镜好用，如果上面两个方法都不行，建议用这个。

### 调低分辨率

非常不建议调低分辨率来使用，液晶屏幕只有一个最合适的分辨率，调低分辨率等于比直接使用对应分辨率的屏幕效果还要差。

## 总结

Windows 从 Vista 开始已经对高分辨率屏幕提供了支持，但由于高分辨率屏幕当时不普及和开发者不重视而被忽视掉了。这也与 Windows 本身的软件生态有关。

Windows 提供了 API 让软件报告给操作系统是自己管理 DPI 缩放还是由系统缩放，但很多软件没有自己管理 DPI 缩放却又告知操作系统由自己管理DPI缩放，造成了很多DPI的适配问题。此篇文章中的前两个方式都是让应用程序不再告知操作系统，让操作系统管理DPi缩放，虽然会造成一些字体的模糊，但还是比调低分辨率来的效果好很多。

## 参考链接

[SetProcessDPIAware function](https://msdn.microsoft.com/en-us/library/windows/desktop/ms633543(v=vs.85).aspx)

[SetProcessDpiAwareness function](https://msdn.microsoft.com/en-us/library/windows/desktop/dn302122(v=vs.85).aspx)

[PROCESS_DPI_AWARENESS enumeration](https://msdn.microsoft.com/en-us/library/windows/desktop/dn280512(v=vs.85).aspx)

[在高分屏下使用windows，你有哪些经验？](https://www.zhihu.com/question/21718976)

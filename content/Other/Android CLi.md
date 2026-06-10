---
date: 2026-04-18T14:51:18+08:00
draft: true
title: Android Cli
---
## 下载安装

### 从网络上安装android cli，并确保是最新版本的android cli

首先根据自己的系统，选择下载的终端命令，跑起来，成功的截图如下。（如需检查您的机器上是否已安装 Android CLI，请运行 `which android` 或 `command -v android`：如果返回路径，则表示已安装）。

![install android cli](install_android_cli.png)

然后输入`android update`确保是使用最新版本的android cli。

![update android cli](android_update.png)

#### Linux

- 终端命令：`curl -fsSL https://dl.google.com/android/cli/latest/linux_x86_64/install.sh | bash`
- 下载链接：`https://dl.google.com/android/cli/latest/linux_x86_64/android`

#### Windows

- 终端命令：`curl.exe -fsSL https://dl.google.com/android/cli/latest/windows_x86_64/install.cmd -o "%TEMP%\i.cmd" && "%TEMP%\i.cmd"`
- 下载链接：`https://dl.google.com/android/cli/latest/windows_x86_64/android.exe`

#### Mac

- 终端命令：`curl -fsSL https://dl.google.com/android/cli/latest/darwin_arm64/install.sh | bash`
- 下载链接：`https://dl.google.com/android/cli/latest/darwin_arm64/android`

检查是不是最新版本的android cli。

输入`android update`

[Blog](https://android-developers.googleblog.com/2026/04/build-android-apps-3x-faster-using-any-agent.html)

---
title: 在WSL2中编译和调试OpenJDK
tags:
    - OpenJDK
    - WSL
    - CLion
---

## 安装WSL

### 安装WSL 1

启用“适用于 Linux 的 Windows 子系统”可选功能，以管理员身份打开 PowerShell 并运行：

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

### 启用虚拟机平台

安装 WSL 2 之前，必须启用“虚拟机平台”可选功能，以管理员身份打开 PowerShell 并运行：

```powershell
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

然后**重新启动**计算机，以完成更新。

### 更新WSL 2 Linux 内核

下载[WSL2 Linux 内核更新包](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)并安装。

### 将 WSL 2 设置为默认版本

安装新的 Linux 分发版时，请在 PowerShell 中运行以下命令，以将 WSL 2 设置为默认版本：

```powershell
wsl --set-default-version 2
```

### 安装 Ubuntu 分发版

打开`Microsoft Store`，搜索并安装`Ubuntu 18.04 LTS`。

### 修改默认源为国内镜像

打开安装好的`Ubuntu 18.04 LTS`，运行下面的命令：

备份原配置文件：

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
```

使用vim打开配置文件：

```bash
sudo vi /etc/apt/sources.list
```

输入替换命令：

```bash
:%s/security.ubuntu/mirrors.aliyun/g
:%s/archive.ubuntu/mirrors.aliyun/g
```

更新：

```bash
sudo apt update
```

## GUI桌面配置

### 安装X Server

```bash
sudo apt install x11-apps
```

### 配置环境变量

编辑`~/bash_login`文件：

```bash
vi ~/.bash_login
```

添加内容如下：

```bash
# For WSL2
export DISPLAY=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}'):0.0
export LIBGL_ALWAYS_INDIRECT=1
```

## 在Windows中安装X Window server

可以使用收费的**X410**、或者免费的**VcXsrv**等，前者对HiDPI似乎有优化。后文以VcXsrv为例子。

### 下载安装VcXsrv

下载[VcXsrv安装包](https://sourceforge.net/projects/vcxsrv/)并安装。

### 生成配置文件

启动XLaunch：按如下配置：

- Display Settings保持默认`Multiple windows`
- Start Clients保持默认`Start no client`
- Extra Settings中， `Clipboard` 、`Primary Selection` 、`Native opengl`保持默认勾选；然后**勾选**上`Disable access control`

最后点击Save configuration保存配置文件，以后就可以通过这个配置文件启动服务。如果有防火墙安全提示，请授权通过。

## 解决高分辨率屏幕上字体模糊

`VcXsrv` 应用图标上右键，选择`属性` -> `兼容性` -> `更改高DPI设置` -> `代替高DPI缩放行为` -> `应用程序`，点击确认。

然后再到 WSL 里将以下代码加入 `.bashrc` 或 `.zshrc`:

```bash
export GDK_SCALE=2
```

## 安装CLion以及编译、调试OpenJDK

这部分内容与在原生Ubuntu环境下类似，请参考[这篇文章](./2020-07-19-debug-openjdk-on-ubuntu.md).

---
title: 个人资料备份方案
published: 2024-11-05
description: '数据无价！'
image: "./cover.png"
tags: [NAS,备份]
category: 'NAS'
draft: false 
lang: ''
---
高中的时候，曾经有一块1T的移动硬盘。

里面存满了我从各种地方找来的电影、动漫、音乐，还有很多自己拍的照片视频。

但是后来挂掉了。。。

几年的数字回忆烟消云散，让我痛苦地意识到了数据备份的重要性。

虽然自那时开始就将数据在本地和网盘都存一份了，但是用的是onedrive E5。

虽然后来号没了，但是本地数据还在，所以只是损失了一些电影。

再后来就是多云盘+本地备份，主要数据在本地，手动上传到云端。

## 2023年9月4日的故事

又是惨痛的一天。

当时用于备份的硬盘是插在软路由上，软路由器定期全盘备份到硬盘。

于是当软路由一挂掉，第一反应就是插卡进恢复系统DD一下。

但是由于搞错了目标分区，直接给我的硬盘给覆盖了。

用DG尝试恢复了一些数据，但是还有有很多没来得及上传的照片无法找回了。

自那时起就决定要做全自动的备份方案，至于整了NAS那又是后面的事了。

## 工具链

### termux

手机上的终端，可安装arm64架构的linux软件。通过termux:Tasker插件可被其他软件调用执行脚本。

### Macrodroid

手机上的自动化软件，通过定时、条件或者手动等方式触发任务并调用termux执行脚本。

### Rclone

伟大，无需多言。云存储的瑞士军刀。

### Alist

为一些不支持Rclone又没有webdav的网盘提供统一接口。

### 其他的命令行工具

netcat、dd、tar、ssh ...

:::tip[关于DD | gzip]
使用DD全盘复制并gzip的时候，需要手动写满剩余空间以达到最佳压缩效果。
```shell
DD if=/dev/zero of=./zero.img bs=65536
```
:::

## 主要思路

将路由器，手机等设备线备份到NAS，再由NAS定时备份到网盘。

部分改动不频繁的如路由器手动备份，其他的自动备份。

## 路由器备份

```shell
#!/bin/bash
ssh root@192.168.3.133 "netcat -l -v -p 1234 -w 10 | pigz > /mnt/storage/Backups/Router/R5C.backup.$(date '+%Y-%m-%d').img.gz" &
sleep 3 && dd if=/dev/mmcblk2 bs=65536 count=60620800 status=progress | netcat -c 192.168.3.133 1234 &
wait
```
ssh连上NAS上的LXC并开启netcat接收数据，将接收到的数据通过管道发送到pigz压缩，再保存到镜像文件。

路由器DD复制全盘通过netcat发送给NAS压缩存档。

修改if、count以及保存文件的位置，直接在路由器上执行此命令即可。

:::caution[重要事项]
**恢复备份时候，千万注意选择正确的目标磁盘！！！**
:::
复制备份文件到U盘并恢复：

```shell
dd if=/mnt/usb4-2/R5C.backup.2024-11-05.img.gz bs=65536 count=60620800 of=/dev/mmcblk2 status=progress
```
或者直接从远程读取文件 (这种时候一般都是用恢复系统进的，就不配置ssh了)：
```shell
pigz -dc R5C.backup.2024-11-05.img.gz | netcat -c 192.168.3.1 1234
```
```shell
netcat -l -v -p 1234 -w 10 | dd of=/dev/mmcblk2 bs=65536 count=60620800 status=progress
```
又或者是用rclone：
```shell
rclone cat nas:storage/Backups/Router/R5C.backup.2024-11-05.img.gz | dd bs=65536 count=60620800 of=/dev/mmcblk2 status=progress
```
status=progress用于显示进度，在软件包里搜```coreutils-dd```

:::warning[友情提示]
路由器稳定大于天，少折腾这玩意。
:::
## 手机备份

termux本体的设置，通过tar压缩后备份到NAS。

一般是进行了什么大改动后手动备份。

```shell
#!/bin/bash
# directory 存放备份文件，只保留最新的三个
directory="/sdcard/termuxBackups"
cd "$directory"
files=($(ls -t *))
if [[ ${#files[@]} -gt 3 ]]; then
  delete_files=("${files[@]:3}")
  for file in "${delete_files[@]}"; do
    rm "$file"
    echo "删除了旧的备份文件：$file"
  done
fi
tar --use-compress-program=pigz -cf /sdcard/termuxBackups/$(date '+%Y-%m-%d').tar.gz -C /data/data/com.termux/files ./home ./usr

echo "上传到NAS"

rclone sync /sdcard/termuxBackups nas:storage/Backups/termux -P

```

恢复备份

```shell
tar -zxf /path/to/your/termux-backup.tar.gz -C /data/data/com.termux/files --recursive-unlink --preserve-permissions
```

参考：[Backing up Termux - Termux Wiki](https://wiki.termux.com/wiki/Backing_up_Termux)

各种APP保存的图片视频，去重后放到统一文件夹。

数据资料按照对应的路径使用rclone保存到NAS。

直接贴代码：

```python
import os
import shutil
import subprocess
import threading
import cv2
import hashlib
import numpy as np

PIC_SRCS = [
    "/sdcard/Pictures/Baimiao",
    "/sdcard/Pictures/QQ",
    "/sdcard/Pictures/Tieba Lite",
    "/sdcard/Pictures/Twitter",
    "/sdcard/Pictures/WeiXin",
    "/sdcard/Pictures/bili",
    "/sdcard/Pictures/ithome",
    "/sdcard/Pictures/想法",
    "/sdcard/Pictures/知乎",
    "/sdcard/tieba",
    "/sdcard/Tencent/QQ_Images",
    "/sdcard/pear/externaImga",
    "/sdcard/DCIM/tieba",
    "/sdcard/Movies/QQ",
]
PIC_DIST = "/sdcard/Pictures/杂图汇总"
COPY_LIST = [
    ("/sdcard/termuxBackups", "nas:storage/Backups/termuxBackups"),
    ("/sdcard/DCIM/Camera", "nas:storage/Media/Pictures/手机相机"),
    ("/sdcard/Pictures/Pixiv", "nas:storage/Media/Pictures/Pixiv"),
    ("/sdcard/Pictures/pixez", "nas:storage/Media/Pictures/pixez"),
    ("/sdcard/DCIM/Screenshots", "nas:storage/Media/Pictures/手机截图"),
    ("/sdcard/Pictures/杂图汇总", "nas:storage/Media/Pictures/杂图汇总"),
    ("/sdcard/MIUI/sound_recorder", "nas:storage/Backups/手机录音"),
    ("/sdcard/ADM", "nas:storage/Backups/学习资料/ADM"),
    ("/sdcard/Download/TwiTake", "nas:storage/Backups/学习资料/TwiTake"),
    ("/sdcard/Download/xunleiTmp", "nas:storage/Backups/学习资料/XunLei"),
    ("/sdcard/Download/QuarkDownloads/CloudDrive", "nas:storage/Backups/学习资料/Quark"),
]

def getAllFiles(path):
    Filelist = []
    for home, dirs, files in os.walk(path):
        for filename in files:
            Filelist.append(os.path.join(home, filename))
    return Filelist

def calculateFileHASH(file_path):
    sha256_hash = hashlib.sha256()
    with open(file_path, "rb") as f:
        for byte_block in iter(lambda: f.read(4096), b""):
            sha256_hash.update(byte_block)
    return sha256_hash.hexdigest()


def calculateHash(image_path, hash_size=16):
    image = cv2.imread(image_path, cv2.IMREAD_GRAYSCALE)
    if image is None:
        return calculateFileHASH(image_path)
    resizedImage = cv2.resize(image, (hash_size, hash_size))
    avg = np.mean(resizedImage)
    ph = 0
    for row in resizedImage:
        for pixel in row:
            ph = (ph << 1) | (1 if pixel > avg else 0)
    return f"{ph:x}"


def run_command_with_timeout(command):
    process = subprocess.Popen(command, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    def target():
        process.wait()
    thread = threading.Thread(target=target)
    thread.start()
    thread.join(timeout=3)  # 设置超时为3秒
    if thread.is_alive():
        process.kill()  # 超时则终止进程
        thread.join()  # 等待线程结束
        return False
    else:
        return process.returncode == 0  # 返回进程的返回码是否为0

if __name__ == "__main__":
    for src in PIC_SRCS:
        for file in getAllFiles(src):
            filename, ext = os.path.splitext(file)
            oldFilePath = file
            newFilePath = os.path.join(PIC_DIST, calculateHash(file) + ext.lower())
            if os.path.exists(newFilePath) and os.path.getsize(oldFilePath) >= os.path.getsize(newFilePath):
                os.remove(oldFilePath)
                print(f"删除:{oldFilePath}")
            else:
                shutil.move(src=oldFilePath, dst=newFilePath)
                print(f"移动:{oldFilePath}=>{newFilePath}")
    if not run_command_with_timeout(["rclone", "lsd", "nas:"]):
        print("NAS离线")
        exit(1)
    for src, dst in COPY_LIST:
        print(f'\n$: rclone copy "{src}" "{dst}" -v --transfers=16')
        os.system(f'rclone copy "{src}" "{dst}" -v --transfers=16')
```

:::tip
没事干多截屏，桌面、快捷栏、app列表、主题设置...

真手机丢了，这些才是难搞的
:::

## NAS备份

NAS作为中转站与本地备份，手机与路由器先备份到NAS，然后NAS再上传到云端。

使用插件[appdata backup](https://forums.unraid.net/topic/137710-plugin-appdatabackup/)，备份appdata、docker配置文件与闪存。

每周备份并且在备份完成后，执行脚本上传云端（sync模式）。

LXC容器手动备份。

媒体数据用计划任务脚本上传。

## 一些Tip

* 使用 \[ termux 🔗 termux\:Tasker 🔗 Macrodroid \] 的时候，在国产手机上经常会被禁止自动启动导致无法拉起。解决方法是创建一个脚本定时间隔调用termux。

* rclone上传到云端时，灵活调整transfers，减弱上传开始延迟且避免同时传输多个大文件。

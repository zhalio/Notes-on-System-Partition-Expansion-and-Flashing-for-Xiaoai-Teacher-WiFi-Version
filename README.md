**小爱老师WiFi版（ MTK 架构安卓设备）** System 分区扩容与刷机~~折腾~~小记

## 一、 实现逻辑

扩容原理：使用 parted 修改分区表，删除旧分区（Server, Data, Cache 等），按新大小重建分区。

救砖实现参考：[https://www.youtube.com/watch?app=desktop&amp;v=3pP4nYaj2aY&amp;themeRefresh=1]()

用sp-flash-tool刷preloader，用mtkclient刷需要刷入的bin

## 二、 准备工作

环境：电脑已配置 ADB/Fastboot 驱动，手机能进入 TWRP 或 Fastboot。

工具：
adb工具包
parted 二进制文件（用于手机端分区）。

文件：
system.img (原厂或 GSI 系统镜像)。
vbmeta.img (必须去验证，否则不开机)。
(可选) boot.img (如果 GSI 需要特定内核)。

## 三、 步骤 1：分区扩容 (在 TWRP 下操作)

1. 手机进入 TWRP，连接电脑，推送工具并进入环境。

```
adb push parted /tmp/               # 推送工具
adb shell chmod 755 /tmp/parted     # 给权限
adb shell /tmp/parted /dev/block/mmcblk0 # 运行工具
```

2. 在 parted 交互界面操作
   (以下数值基于 3GB System 扩容方案，严格按顺序执行)

```
(parted) unit mb                    # 切换单位
(parted) print                      # 确认 vendor 结束于 923MB

# 删除旧分区
(parted) rm 34                      # userdata
(parted) rm 33                      # cache
(parted) rm 32                      # vbmeta
(parted) rm 31                      # system

# 重建分区 (System 扩容至 3GB)
(parted) mkpart system ext2 923 4000
(parted) mkpart vbmeta 4000 4016
(parted) mkpart cache ext4 4016 4500
(parted) mkpart userdata ext4 4500 15572

# 命名分区
(parted) name 31 system
(parted) name 32 vbmeta
(parted) name 33 cache
(parted) name 34 userdata

(parted) quit                       # 退出
```

3. 重启到 Fastboot
   **退出 parted 后，立刻重启到 Bootloader，不要重启系统！！！！！**

    **本机只能通过twrp进fastboot，如果重新启动，进不了twrp也就无法进fastboot了**

```
adb reboot bootloader
```

## 四、 步骤 2：刷写系统 (在 Fastboot 下操作)

1. 刷入基础镜像

```
# 刷入系统 (忽略大小不匹配警告)
fastboot flash system System.img

# 刷入去验证的 Vbmeta (防止卡米的关键)
fastboot --disable-verity --disable-verification flash vbmeta vbmeta.img
# 如果报错 unknown option，尝试直接刷：fastboot flash vbmeta vbmeta.img
```

2. 重建文件系统 (必须做！)
   扩容后这两个分区是“未格式化”状态，必须擦除重建。

```
fastboot erase userdata
fastboot erase cache
# 如果 fastboot 版本较老不支持 format，erase 也足够触发安卓自动格式化
```

3. 重启

```
fastboot reboot
```

## 五、 GSI 系统选择

架构: arm32_binder64 (也叫 a64)
文件名示例: system-a64-bvN.img (b=A/B型, v=纯净版, N=无Root)

## 六、更换别的GSI

```
REM 擦除旧系统
fastboot erase system

REM 刷入新 GSI 
fastboot flash system <你的GSI镜像文件路径.img>

REM 再次清除数据 (换系统必须双清，否则卡开机动画)
fastboot erase userdata
fastboot format userdata

REM 重启
fastboot reboot
```

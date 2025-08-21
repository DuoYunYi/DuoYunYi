AML文档中外设相关：LED小灯、IR遥控器、PWM、ADC按键、GPIO、I2C、UART、SPI、WIFI、BT、Ethernet

# Uboot常用调试命令：

## 1.print

在Uboot下输入print命令将打印出uboot所有环境变量

## 2.setenv

用于设置uboot环境变量

## 3.saveenv

setenv后需要对uboot env进行保存，保存后执行reset或者断电重启方可生效

## 4.reset

uboot下重启

## 5.fatload/fatls

fatload/fatls：用于uboot下挂载和查看u盘/sdcard的命令

fatload usb 0 ：挂载u盘

fatls usb 0 ：枚举u盘中的内容

## 6.usb_update

用于加载u盘中的image对分区进行升级

示例：usb_update boot boot.img

## 7.gpio

查看当前gpio状态，或者改变gpio状态

示例：gpio status -a GPIOAO_3 //查看GPIOAO_3的状态

gpio toggle GPIOAO_3 //GPIOAO_3将进行高低电平切换

## 8.env

用于还原默认环境变量，配合saveenv和reset使用

env default -a;saveenv;reset

## 9.关闭selinux

setenv EnableSeLinux permission；save ；reset；

# Kernel常用命令

## 1.获得对vendor/system 的rw权限

Vendor：

mount -o rw，remount /vendor；

System:

mount -o remount,rw -t ext4 /dev/root;

## 2.查看内存使用情况

cat proc/meminfo;

dumpsys Meminfo

## 3.查看cpu使用情况

top命令

## 4.查看gpu/hwc 状态

dumpsys SurfaceFlinger

## 5.查看audio状态

dumpsys media.audio_policy

## 6.查看当前输出模式

cat sys/class/display/mode;
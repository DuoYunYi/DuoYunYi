# 一、信息查询

## 1.查询设备固件版本信息

AT+QGMR

```
echo -e "AT+QGMR\r\n">/dev/ttyUSB2

cat /dev/ttyUSB2
```

若查询失败，则另外开一个窗口cat /dev/ttyUSB2

## 2.查询模块固件版本信息

AT+GMR

```
echo -e -n "AT+GMR\r\n" > /de/ttyUSB2 && cat /de/ttyUSB2
```

## 3.查询模块的制造商信息、型号和版本

ATI

```
echo "ATI\r\n" > /dev/ttyUSB2 

cat /dev/ttyUSB2
```

## 4.







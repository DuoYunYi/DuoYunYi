
# 一、安装卸载文件
## 1.安装deb文件

```
sudo dpkg -i xxxxx.deb
```
## 2.卸载vsCode

```
vscode在ubuntu上到软件包名称为code

sudo apt-get remove code //只是卸载，保留配置
sudo apt-get --purge remove code //彻底清除，包括配置
```
## 3.安装-卸载adb

```
sudo apt install adb fastboot

sudo apt remove adb fastboot
```
## 4.安装搜狗输入法

```

```

## 5.dpkg查看安装路径

```
`dpkg -c package.deb` 或 `dpkg -L <installed-package>`
```

# 二、压缩-解压文件

- `-c`：创建归档文件
- `-v`：显示详细过程（可选）
- `-f`：指定输出文件名
- `-z`：使用 `gzip` 压缩

### 1.压缩tar文件

#### （1）使用 `gzip` 压缩（`.tar.gz` 或 `.tgz`）

```
tar -czvf 文件名.tar.gz 文件或目录

tar -czvf archive.tar.gz /path/to/directory
```
#### （2）使用 `bzip2` 压缩（`.tar.bz2`）

- `-j`：使用 `bzip2` 压缩

```
tar -cjvf 文件名.tar.bz2 文件或目录

tar -cjvf archive.tar.bz2 /path/to/directory
```
#### （3）使用 `xz` 压缩（`.tar.xz`）

- `-J`：使用 `xz` 压缩（压缩率更高，但更慢）

```
tar -cJvf 文件名.tar.xz 文件或目录

tar -cJvf archive.tar.xz /path/to/directory
```
### 2.解压 `.tar` 文件

#### （1）解压 `.tar`

- `-x`：解压
- `-v`：显示详细过程（可选）
- `-f`：指定输入文件
```
tar -xvf 文件名.tar

tar -xvf archive.tar
```
#### **（2）解压 `.tar.gz` 或 `.tgz`**

```
tar -xzvf 文件名.tar.gz

tar -xzvf archive.tar.gz
```
#### **（3）解压 `.tar.bz2`**

```
tar -xjvf 文件名.tar.bz2

tar -xjvf archive.tar.bz2
```
#### **（4）解压 `.tar.xz`**

```
tar -xJvf 文件名.tar.xz

tar -xJvf archive.tar.xz
```

### **3. 解压到指定目录**

使用 `-C` 参数指定目标目录：

```
tar -xzvf archive.tar.gz -C /目标/路径

tar -xzvf archive.tar.gz -C ~/Downloads
```

### 4. 其他常见压缩/解压命令

|格式|压缩命令|解压命令|
|---|---|---|
|`.zip`|`zip -r archive.zip 目录`|`unzip archive.zip`|
|`.rar`|`rar a archive.rar 目录`|`unrar x archive.rar`|
|`.7z`|`7z a archive.7z 目录`|`7z x archive.7z`|

## 5.总结

```
- **打包**：`tar -cvf file.tar dir`
- 
- **打包 + gzip**：`tar -czvf file.tar.gz dir`
- 
- **打包 + bzip2**：`tar -cjvf file.tar.bz2 dir`
- 
- **打包 + xz**：`tar -cJvf file.tar.xz dir` 
- 
- **解压**：`tar -xvf file.tar`（根据压缩方式加 `z`/`j`/`J`）
- 
- **查看内容**：`tar -tvf file.tar`
```

如果某些命令（如 `unrar`、`7z`）未安装，可以使用：
```
sudo apt install unrar p7zip-full
```

# 三、scp命令




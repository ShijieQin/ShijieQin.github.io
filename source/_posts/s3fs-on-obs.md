---
title: s3fs on obs
comments: true
date: 2017-06-21 16:47:14
updated: 2017-06-21 16:47:14
tags:
- 华为云
- obs
categories:
- 华为云
- obs
---
### s3fs简介
s3fs 是一款用于 Amazon S3 在 Linux 环境下的命令行工具，并为商业和私人免费使用，只需支付对应的存储服务的费用。它的主要功能是能够把桶（Bucket）挂载到本地目录，进行文件、文件夹的拷入、拷出和删除等操作，即完成对桶内文件、文件夹的上传、下载和删除操作。
<!-- more -->
### 下载s3fs
登录网址 http://code.google.com/p/s3fs/downloads/list 进入下载界面，选择相应
版本，单击“Download”进行下载。
本文档以s3fs-1.74.tar.gz为例
### 安装s3fs

#### 安装依赖

```
yum install fuse.x86_64 fuse-devel.x86_64 fuse-libs.x86_64 libconfuse-devel.x86_64 libconfuse.x86_64 gcc-c++.noarch curl.x86_64 libcurl-devel.x86_64 libcurl.x86_64 libxml2.x86_64 libxml2-devel.x86_64 openssl-devel.x86_64
# 检查fuse版本(必须要大于2.8.4)
rpm -qa | grep fuse
# 若小于2.8.4，则需要用源码安装新版的fuse，如下
yum remove fuse fuse-devel
tar -xvf fuse-2.8.4.tar
cd fuse-2.8.4
./configure --prefix=/usr/local/fuse
make
make install
echo /usr/local/fuse/lib >> /etc/ld.so.conf
echo "export PKG_CONFIG_PATH=/usr/lib/pkgconfig:/usr/lib64/pkgconfig/:/usr/local/fuse/lib/pkgconfig/" >> /etc/profile
source /etc/profile
ldconfig
modprobe fuse
pkg-config --modversion fuse
```

#### 解压

```
tar -zxvf s3fs-1.74.tar.gz
```
#### 安装

```
cd s3fs-1.74
./configure
make && make install
```
#### 配置秘钥
```
vi /etc/passwd-s3fs
# 添加秘钥，格式如下
AK:SK
# 更改权限
chmod 640 /etc/passwd-s3fs
```
#### 创建obs用户

```
groupadd obs
useradd -g obs obs
```

#### 用户obs用户创建cache文件夹
```
mkdir .obs_cache
```

#### 挂载

```
s3fs www.going-link.com /obs -o host=http://obs.myhwclouds.com -o umask=0022 -o uid=501 -o gid=501 -o use_cache=/home/obs/.obs_cache -o allow_other
```

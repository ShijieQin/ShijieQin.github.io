---
title: duplicity install
comments: true
date: 2017-06-20 18:17:26
updated: 2017-06-20 18:17:26
tags:
- duplicity
categories:
- duplicity
- install
---
#### 安装环境依赖
```
yum install -y gcc-c++ librsync python-lockfile python-urllib3 python-setuptools python-devel librsync-devel python-pip

pip install oss2
```
#### 安装
```
tar zxvf duplicity.tar.gz
cd duplicity
python setup.py install
```

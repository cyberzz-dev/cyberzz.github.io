---
title: 关于日志归档程序发布pyinstaller
description: 
date: 2023-11-29
slug: log_archive_pytinstaller
---

最近用python3写了一个日志归档程序。

主要功能：检测指定目录文件、压缩前一天的文件、上传到远端hdfs



由于主机多，而且py脚本有些特殊的hdfs依赖，导致执行脚本之前，必须初始化主机的依赖包。

这样存在一个问题：主机太多，怕依赖包影响其他程序。



研究了下，可以使用 pyinstaller打包依赖，省去很多麻烦。

```shell
pyinstaller --onefile  log_archive.py \
--hiddenimport=gssapi.raw.cython_converters \
--collect-submodules gssapi.raw \ 
--add-data /root/example.conf:.  \
--add-data /root/example.key:.
```

还把相关的配置文件一起打包。



实际原理：pyinstaller生成了一个最小的python环境，压缩成一个可执行文件。执行文件时，自解压到 /tmp下的一个随机目录进行执行，执行完自动删除。



自解压，忽然联想到以前研究的一个脚本 makeself.sh 在github上。

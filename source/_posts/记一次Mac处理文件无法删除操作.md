---
title: 记一次Mac处理文件无法删除操作
date: 2020-12-14 10:34:43
tags: Mac
categories: Shell
---

## 处理流程

安装了公司的安全上网软件，但是无法卸载和删除。

执行rm -rf无法删除

{% asset_img image-20201214103807333.png %}

`ls -lO`查看权限

{% asset_img image-20201214103914978.png %}

需要清理这两个标记

执行：

```shell
sudo chflags -R nohidden [文件夹名称]
sudo chflags -R noschg [文件夹名称]
sudo chmod -R +w [文件夹名称]
sudo chmod -R +r [文件夹名称]
```

最后

{% asset_img image-20201214104224717.png %}

## 补充

`ls`的`-O`选项，可以列出文件的file flag。

### File Flag

File flag是在BSD Unix中的概念，跟Linux系统中的attr是差不多的一个概念，是文件的一些标志位来存放文件的某些属性。chflags就是来修改这个file flag的。这个文件属性是跟文件系统相关的，所以这个命令在不同的文件系统上的支持程度不一样，体现在某一些flag在一些特定的文件系统上没有。

### 常见的几个属性

| 属性           | ls中显示 | chflags中使用             | 文件所有者能否修改 | 详述                                                         |
| :------------- | :------- | :------------------------ | :----------------- | :----------------------------------------------------------- |
| 隐藏           | hidden   | hidden                    | 能                 | 设置以后在GUI上看不到，ls依然可以看到d                       |
| 系统级只能添加 | sappnd   | sappnd, sappend           | 否                 | 设置以后此文件不能够截断或者复写(overwrite)，只能通过append模式添加内容 |
| 用户级只能添加 | uappnd   | uappnd, uappend           | 能                 | 设置以后此文件不能够截断或者复写(overwrite)，只能通过append模式添加内容 |
| 系统级只读     | schg     | schg, schange, simmutable | 否                 | 不能够重命名、移动、删除、更改内容                           |
| 用户级只读     | uchg     | uchg, uchange, uimmutable | 能                 | 不能够更改内容                                               |

### 基本用法

`chflags [-fhv] [-R [-H | -L | -P]] flags file`

### 例子

**为一个文件添加一个属性**

```shell
chflags uchg file
```

**为一个文件删除一个属性**

```shell
chflags nouchg file
```

在属性名字前面添加no就可以将属性删除，如果这个属性本身已no开头（比如nodump）则去掉no。

**将文件夹及其文件夹下所有文件属性进行修改**

```shell
chflags -R uchg directory
```


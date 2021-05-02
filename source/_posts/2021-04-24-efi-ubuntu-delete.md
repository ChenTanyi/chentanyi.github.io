---
title: 从 Win + Ubuntu 中删除 Ubuntu 系统及 UEFI 引导
date: 2021-04-24 13:11:30
tags: [OS, UEFI]
categories: OS
---

> 由于身边已经没有 BIOS/MBR 机器了，所以只介绍 UEFI/GPT 引导相关
> 警告！！！：如果你的机器[启动类型是 BIOS ](https://blog.csdn.net/qq_35685189/article/details/79930971)的，请不要参考文章的任何做法

懒人心血来潮写点笔记（可能写一两篇就又会忘了），记录最近的胡乱鼓捣

# 一笔带过的安装篇

由于现在双系统安装已经很成熟了（也可能只是我装多了比较熟悉），所以这里只是稍微带过。不经常鼓捣 Linux 所以一般只用常规的 Ubuntu/Debian

## 以 UEFI + Win 安装 Ubuntu 为例

0. 预留一段未分配磁盘给 Ubuntu （或者用磁盘分区工具切出来）
1. 下载 [iso](https://ubuntu.com/#download)
2. 用 [Rufus](https://rufus.ie/) 或 [UltraISO](https://www.ultraiso.com/) 制作启动U盘
3. 重启进入U盘系统并选择 Install，按照安装指引走即可
    > 分区：如果有休眠需求，需要分配一个比物理内存大的 swap，`/`(软件目录) 与 `/home`(私人文件目录) 按自己需求分配，如果不在意重装时文件丢失，可以只分配 `/` 分区

# 卸载篇

卸载一般需要两步：删引导和删分区
> 删引导相对麻烦，如果没有洁癖，且开机 UEFI 有 Windows Boot Manager，可以选择不删引导

## 删 UEFI 引导

因为 Windows 会默认帮用户写上直接的系统引导项，可以只删除不需要的引导

但由于一般新安装的系统会覆盖掉硬盘引导项(EFI\Boot\bootx64.efi)，如果不确定自己是否有 Windows 的系统引导项（开机 UEFI 界面的 Windows Boot Manager），需在删除后修复硬盘引导项

1. 右键`此电脑/我的电脑`->点击`管理`->点击`磁盘管理`，可以查到 EFI 分区信息
    <img src="/img/OS/EFI/disk_management.png" />

2. 用管理员身份打开 cmd，并依次执行以下命令挂载 EFI 分区（并暂时不要关闭窗口）:
    <img src="/img/OS/EFI/cmd.png" />
    ```cmd
    diskpart                         --打开 diskpart
    select disk 0                    --选择磁盘 0 （数字取决于上面 EFI 分区信息）
    select partition 1               --选择分区 1 （数字取决于上面 EFI 分区信息）
    assign letter=p                  --将分区挂载到 P 盘（随便选一个没在用的盘符）
    ```

3. 用管理员身份打开记事本，选择`文件`->`打开`，在这个界面上选择 P 盘（上面分配的盘符）并删除 `EFI` 文件夹下面的 `Boot` （硬盘引导） 和 `Ubuntu` （系统引导）
    <img src="/img/OS/EFI/notepad.png" />
    <img src="/img/OS/EFI/notepad_delete.png" />

4. 回到 2 的窗口，执行以下命令删除 EFI 分区的挂载:
    ```cmd
    remove letter=p                  --删除 P 盘的挂载（盘符要和挂载命令一致）
    ```
    <img src="/img/OS/EFI/cmd_diskpart.png" />

5. 重新用管理员身份打开一个 cmd 窗口，执行以下命令修复 `EFI` 下的 `Boot` 文件夹:
    ```cmd
    bcdboot c:\windows /l zh-cn       --如果 windows 不在 c 盘，请自行修改盘符
    ```

6. 重启查看效果，进入 UEFI 界面删除 Ubuntu 的系统引导项并切换 Windows Boot Manager 作为默认引导
    > 开机进 UEFI 的按键与品牌有关，可以自行搜索，一般为 Esc / Del / F1 / F10

## 删除 Ubuntu 分区

删分区一般比较简单暴力，打开磁盘管理，把装系统时分出来给 Ubuntu 的区直接右键`删除卷`就行

> 最近删了一个 Ubuntu 系统，暂时没有遇到其他问题，有遇到相关问题欢迎在 [github issue](https://github.com/chentanyi/chentanyi.github.io/issues) 提出

# 参考

* https://www.jianshu.com/p/9166e8c00ca2
* https://www.zhihu.com/question/22502670
* https://www.zhihu.com/question/23499575
* https://zhuanlan.zhihu.com/p/31365115
* https://my.oschina.net/u/4276152/blog/4258882
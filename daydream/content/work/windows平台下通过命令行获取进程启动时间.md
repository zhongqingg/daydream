+++
title = 'windows平台下通过命令行获取进程启动时间'
date = 2021-04-14T22:02:42+08:00
draft = false
+++


## 1. 基于Powershell的cmdlet
管理员身份运行powershell，执行以下命令：
```bash
Get-Process | select ProcessName, Id,starttime |findstr "name"
```
其中*select* 后面的三个选项分别列出了进程的*名称*，*PID* 以及 *启动时间*。将*name*替换成需要查找的进程名称，即可通过*findstr*命令提取出目标进程。
想获取进程其他信息，可直接执行*Get-Process*查看可用选项，加在命令中*select*的选项后面即可。

## 2. 基于CMD的WMIC
*WMIC*全称*Windows Management Instrumentation Command-line*。运行cmd.exe，执行以下命令：
```bash
wmic process where Name="process name" get CreationDate, ProcessId
```
将其中的*process name*替换成需要查找进程的名称即可，这里需要使用带后缀的全称——比如想要查找cmd的信息，应该写*Name="cmd.exe"*，而不是*Name="cmd"*。其中*CreationDate*获取了进程时间，*ProcessId*获取了*PID*。
想获取进程其他信息，可直接执行*wmic process*，查看第一行可用选项，添加载*get*的选项后面即可。

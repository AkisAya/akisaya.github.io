---
title: 使用脚本查看 macOS 电池信息
date: 2017-12-22 17:51:30
categories: python
tags:
- python
---


## shell 命令获取电池信息

刚开始使用 macbook pro 后总是查看电池的信息，担心电池衰减快导致续航崩掉 orz，毕竟 macbook 的一个卖点就是续航。后来每次都要打开 system info 切换到 battery 查看，有点麻烦，就产生了一个想法写一个脚本每天自动记录 battery 的最大容量，隔一段时间使用图表画出容量的变化。

经过一番 google 后，一开始是发现了有 python 的库 `psutil` 和 `power` 可以访问系统的信息，但是似乎都得不到想要的电池最大容量这个数据，最后在[这个网站](https://apple.stackexchange.com/questions/116429/using-bash-terminal-to-get-number-of-battery-recharge-cycles)发现了有用的 bash 命令。

<!-- more -->
笔记本以树形的方式记录了 IO 设备的信息，使用 `ioreg` 可以查看这些信息

```shell
ioreg -l -w0 | grep Capacity

//output
    | |           "MaxCapacity" = 6108
    | |           "CurrentCapacity" = 3288
    | |           "LegacyBatteryInfo" = {"Amperage"=18446744073709550947,"Flags"=4,"Capacity"=6108,"Current"=3288,"Voltage"=11363,"Cycle Count"=38}
    | |           "DesignCapacity" = 6559
```

或者

```shell
system_profiler SPPowerDataType | grep "Cycle Count" | awk '{print $3}'
```

这个命令拿到的是循环次数，稍加改造就可以拿到最大电池容量

```shell
system_profiler SPPowerDataType | grep "Full Charge Capacity" | awk '{print $5}'
```

同时 google 下 `system_profiler` 这个命令，发现这是 macOS 提供的系统信息查询的命令

>  system_profiler reports on the hardware and software configuration of the system.  It can generate plain text reports or XML reports which can be opened with System Information.app

使用该命令就如同查看 system info，只不过是同图形界面变到了 terminal 上



既然可以通过这个命令拿到想要的信息，下一步就是如何使用 python 调用这些命令并编写脚本了。

## 编写 python 脚本

[获取 python 脚本](https://github.com/AkisAya/batteryinfo)

使用 `os.popen()` 调用 shell 命令并输出

```python
comm = "system_profiler SPPowerDataType | grep 'Full Charge Capacity' | awk '{print $5}'"
os.popen(comm).read().strip()
# output
'6075'
```

为了提取更多有用的信息，可以保存 `system_profiler SPPowerDataType` 输出的信息，然后使用正则表达式提取我们需要的信息。创建一个类来完成这个任务再好不过了。

```python
import os
import re


class Battery:
    def __init__(self):
        self.power_info = os.popen("system_profiler SPPowerDataType").read()

    def __get_number_value(self, pattern):
        ret = re.search(pattern, self.power_info).group(1)
        return int(ret)

    def __get_string_value(self, pattern):
        return re.search(pattern, self.power_info).group().split(":")[1].strip()

    def max_capacity(self):
        return self.__get_number_value(r'Full Charge Capacity.*?(\d+)')

    def current_capacity(self):
        return self.__get_number_value(r'Charge Remaining.*?(\d+)')

    def cycle_count(self):
        return self.__get_number_value(r'Cycle Count.*?(\d+)')

    def design_capacity(self):
        info = os.popen('ioreg -l -w0 | grep DesignCapacity').read()
        return int(info.split("=")[1].strip())

    def percentage(self):
        return int(self.current_capacity()/self.max_capacity() * 100)

    def battery_health(self):
        return int(self.max_capacity() / self.design_capacity() * 100)

    def battery_condition(self):
        return self.__get_string_value(r'Condition.*')

    def is_charging(self):
        status = self.__get_string_value(r'Charging.*')
        if status == 'No':
            return False
        else:
            return True
```

由于 `system_profiler SPPowerDataType` 输出的信息本身就是格式化的，所以要进一步优化这个方案的话可以直接解析输出信息构造出一个树，使用 json 格式来存放信息或者使用 dict 来存放信息



### 其他

在使用 python 的 re 模块时发现，在匹配字符串的时候最多只会匹配到一行的末尾。比如：

对于字符串

```
Charge Remaining (mAh): 5268
Fully Charged: No
Charging: No
Full Charge Capacity (mAh): 6097
```

当我们使用 `re.search(r'Charging.*')` 时，会匹配到 `Charging: No` 这一行，但是也不会跨行



## macOS 定时任务

前面已经写了基本的类获取电池信息，现在就写个定时任务每天执行任务记录电池信息，数据够多后画出来看看电池的变化吧。

```python
# output_battery_info.py
from battery_info import Battery
import datetime

batt = Battery()
path = '/Users/akis/Documents/battery/info.txt'
with open(path, 'a') as file:
    logtime = datetime.datetime.now().strftime("%Y-%m-%d")
    line = '{0}\t{1}\t{2}\n'.format(logtime, batt.max_capacity(), batt.cycle_count())
    file.write(line)
```

剩下的就是使用 mac 的 launchctl 定时执行脚本了

launchctl 的使用参见 [Mac执行定时任务之Launchctl](http://blog.csdn.net/u012390519/article/details/74542042)

定义一个可执行文件 `logBatteryInfo.sh` 如下

```sh
#!/bin/sh
/usr/local/bin/python3 /Users/akis/codespace/python/batteryinfo/output_battery_info.py
```

自定义任务列表 `me.battery.launchctl.plist` 

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>me.battery.launchctl.plist</string>
    <key>ProgramArguments</key>
    <array>
      <string>/Users/akis/Documents/battery/logBatteryInfo.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
          <key>Minute</key>
          <integer>20</integer>
          <key>Hour</key>
          <integer>23</integer>
    </dict>
    <key>StandardOutPath</key>
      <string>/Users/akis/Documents/battery/logBatteryInfo.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/akis/Documents/battery/logBatteryInfo.err</string>
  </dict>
</plist>
```




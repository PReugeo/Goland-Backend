## 时间和日期

Python中提供了两个和时间日期相关的模块`time`和`calendar`以及`datetime`。

这一篇，我们重点来讲`datetime`

## 获取当前时间

要获取当前的本地时间，需要是用`datetime`模块中的`datetime`类提供的`now`函数。

```python
now = datetime.now(tz=None)
```

- `tz`是转换的时区，默认设置为None，表示获取本地时区。
- `now`函数追中返回表示当前地方时的datetime对象

```python
import datetime
now = datetime.datetime.now()
```

## 创建datetime对象

除了获取表示系统当前日期时间的`datetime`对象，你可以创建指定日期和时间的`datetime`对象。

```python
from datetime import datetime
dt =  datetime(year, month, day[, hour[, minute[, second[, microsecond[,tzinfo]]]]])
```

- `datetime`是一个类，我们可以使用这个类来创建自定义的`datetime`对象。
- `year, month, day`年月日是三个必传递参数。
- 后面的`hour,minute,second,microsecond,tzinfo`都是可选参数。

### datetime中的方法

既然`datetime`是一个类，这个类中就存在一些有用的函数。

#### 分别获取datetime中的日期和时间部分

```python
from datetime import datetime
dt = datetime(2021, 7, 2, 8, 10, 30)
# 获取date对象
dt.date()
# 获取time对象
dt.time()
# 获取时间戳
dt.timestamp()
```

## datetime和其他类型的相互转换

有时候，我们需要把一个字符串格式的时间，或者一个时间戳转换成`datetime`类型的对象，或者反过来把datetime转换成一个字符串。下面我们看看这些类型之间是如何进行转换的。

### datetime转成字符串

将`datetime`对象装换成`字符串`。可以使用`datetime`对象上的`strftime(format)`方法。

这里的`format`你可以设置转换的格式，下面是一张可以使用的常用格式的表：

| 指令 | 含义                                                         | 示例                                     |
| :--- | :----------------------------------------------------------- | :--------------------------------------- |
| %w   | 以十进制数显示的工作日，其中0表示星期日，6表示星期六。       | 0, 1, …, 6                               |
| %d   | 补零后，以十进制数显示的月份中的一天。                       | 01, 02, …, 31                            |
| %m   | 补零后，以十进制数显示的月份。                               | 01, 02, …, 12                            |
| %y   | 补零后，以十进制数表示的，不带世纪的年份。                   | 00, 01, …, 99                            |
| %Y   | 十进制数表示的带世纪的年份。                                 | 0001, 0002, …, 2013, 2014, …, 9998, 9999 |
| %H   | 以补零后的十进制数表示的小时（24 小时制）。                  | 00, 01, …, 23                            |
| %I   | 以补零后的十进制数表示的小时（12 小时制）。                  | 01, 02, …, 12                            |
| %p   | 本地化的 AM 或 PM 。                                         | AM, PM (en_US); am, pm (de_DE)           |
| %M   | 补零后，以十进制数显示的分钟。                               | 00, 01, …, 59                            |
| %S   | 补零后，以十进制数显示的秒。                                 | 00, 01, …, 59                            |
| %j   | 以补零后的十进制数表示的一年中的日序号。                     | 001, 002, …, 366                         |
| %W   | 以十进制数表示的一年中的周序号（星期一作为每周的第一天）。 在新的一年中第一个第期一之前的所有日子都被视为是在第 0 周。 | 00, 01, …, 53                            |
| %%   | 字面的 ’%’ 字符。                                            | %                                        |

[更加完整的字符串格式说明，请点击此处请查看Python官方文档](https://docs.python.org/zh-cn/3/library/datetime.html#strftime-and-strptime-format-codes)

使用这些格式指令，可以将`datetime`格式化成你需要的字符串。

```python
from datetime import datetime
now = datetime.now()
print(now.strftime("%Y-%m-%d %H:%M:%S"))
```

### 字符串转成datetime

反过来，你也可以使用`datetime.strptime`将字符串转换为`datetime`对象。

```python
datetime.strptime(str, format)
```

- `str`是要被转换的字符串格式的日期
- `format`是提供转换的格式化字符串

```python
from datetime import datetime
dt = datetime.strptime('2021-7-20 9:08:19', '%Y-%m-%d %H:%M:%S')
print(dt)
```

### 时间戳转成datetime

对于收到的时间戳使用`datetime.fromtimestamp(t)`转换成`datetime`类型的对象。

需要注意的是这里的时间戳接收的整数部分是以秒为单位的。如果时间戳是从其他语言获取到的，比如：

> javascript的`Date.now()`获得的时间戳`1626743939450`是以毫秒为单位的，需要除以1000换算为秒为单位`1626743939.45`。然后送入`datetime.fromtimestamp(1626743939.45)`进行转换。

## 时区转换

对于不同时区的时间，我们需要进行时区转换后才能使用，比如获取的是UTC时间(世界协调时间,UTC + 00:00)，如果我们想要知道这个时间对应的北京时间(北京,UTC + 8:00)，就需要进行时区转换。

```python
from datetime import datetime,timezone,timedelta
# 获取UTC时间
utc_now = datetime.utcnow()
# 将UTC时间 强制转换为UTC + 00:00 时区的时间
utc_zero = utc_now.replace(tzinfo=timezone.utc).astimezone(tz=None)
# 将UTC 0时区时间转换为北京时间
beijing_now = utc_zero.astimezone(timezone(timedelta(hours=8)))
```

- `datetime.utcnow()`返回的是当前utc时间，但其内部不带`tzinfo`时区信息

- ```
  utc_zero
  ```

  第二行的代码作用是将这个utc时间转换为带UTC 0时区信息的时间

  - `utc_now.replace(tzinfo=timezone.utc)`对`utc_now`设置utc时区
  - 利用`astimezone(tz=None)`返回一个具有`tzinfo`属性tz的新`datetime`对象。
  - 最后利用`astimezone`设置新的时区，来返回指定时区的`datetime`，达到转换时区的目的。

上面的时区转换写法相对复杂，下面的版本对其进行简化：

```python
from datetime import datetime,timezone,timedelta
# 创建带UTC 0时区信息的当前时间
utc_zero = datetime.now(timezone.utc)
# 将UTC 0时区时间直接转换为北京时间
beijing_now = utc_zero.astimezone(timezone(timedelta(hours=8)))
```

如果我们直接创建带有UTC时区信息的时间，就可以省略一步手工创建带时区的`datetime`的步骤

时区转换 `utc str -> local timezone`

```python
import datetime

import pytz
import tzlocal

local_timezone = tzlocal.get_localzone()
date_obj = datetime.datetime.utcnow() \
  		 	 .replace(tzinfo=pytz.utc) \
         .astimezone(local_timezone)
```


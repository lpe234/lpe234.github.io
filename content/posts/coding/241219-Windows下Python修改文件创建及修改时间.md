+++
date = '2024-12-19T18:10:49+08:00'
draft = false
title = 'Windows下Python修改文件创建及修改时间'
tags = ['Python', '代码片段']
+++

> 项目需要，某些场景下需要修改文件的创建(Date created)及修改时间(Date modified)，另外还有个访问时间(Date accessed)，不过这个时间打开文件夹访问时就会更新。

## 文档

需要安装`pywin32`模块。

- [pywin32](https://mhammond.github.io/pywin32/)

## 关键代码

```python
import pywintypes
import win32file

def change_file_time(filename: str, create_time: float, modified_time: float):
    """
    修改文件的创建时间和修改时间
    :param filename: 文件名
    :param create_time: 创建时间
    :param modified_time: 修改时间
    :return:
    """
    handle = win32file.CreateFile(filename, win32file.GENERIC_WRITE, 0, None, win32file.OPEN_EXISTING,win32file.FILE_ATTRIBUTE_NORMAL, None)
    win32file.SetFileTime(handle, pywintypes.Time(create_time), pywintypes.Time(modified_time), pywintypes.Time(modified_time))
```

## 业务逻辑 

将日期修改为`四、五`月份，时间修改为`[10-19]`点之间，尽量随机一些。

创建与修改时间间隔大概为`[10, 50]`天。

代码如下:

```python
def rnd_minute():
    return random.randint(1, 59)


def rnd_hour():
    return random.randint(10, 19)


def rnd_day():
    return random.randint(1, 30)


def rnd_month():
    return random.randint(4, 5)


def rnd_days():
    return random.randint(10, 50)

def main():
    files = get_all_files()

    for idx, file in enumerate(files):
        create_time = datetime.strptime(f'2023-{rnd_month()}-{rnd_day()} {rnd_hour()}:{rnd_minute()}:{rnd_minute()}',
                                        '%Y-%m-%d %H:%M:%S').timestamp()
        modified_time = datetime.strptime(f'2023-{rnd_month()}-20 {rnd_hour()}:{rnd_minute()}:{rnd_minute()}',
                                          '%Y-%m-%d %H:%M:%S').timestamp()
        # 将modified_time 添加range_day()天数
        modified_time += rnd_days() * 24 * 60 * 60
        # 修改时间
        change_file_time(file, create_time, modified_time)
```

## Gist

[https://gist.github.com/lpe234/c131cdffa0cf9721e94a4bf701d9b42b](https://gist.github.com/lpe234/c131cdffa0cf9721e94a4bf701d9b42b)

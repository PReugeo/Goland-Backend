```python
import logging
import logging.handlers
import os
import re
import time

from lib.utils import conf


class MPTimeRotatingFileHandler(logging.handlers.BaseRotatingHandler):
    """Customized TimeRotatingFileHandler to support multiprocessing logging
    简化了TimeRotatingFileHandler，去除夏令时，按周几切分等逻辑，只支持按when级别切割，不支持按几个when级别切割

    解决了以下问题
    问题1：
    原生TimeRotatingFileHandler是将日志保存在xxx.log中，在切分日志时重命名为xxx.log.<date>
    在多进程场景（gunicorn）下，每个进程都会尝试做这个操作，会导致日志丢失
    这里的解决办法是改写切分逻辑，直接将日志保存在xxx.log.<date>下

    问题2：
    按原先的逻辑，正常往固定名称的日志里写写写，如果一段时间不写，即使超过时间也不做切分，直到下次写入
    写入的时候发现当前时间已经超过下一个切分时间点，再做切分，这时候获取rolloverAt - interval的时间点作为
    切分的日志，比如9点有2条日志，然后一直到12点才有第3条日志，就会造成日志的错位
    改动方法还是跟第一个问题一样，但是由于写入的时候实时判断，就不能用以前的rolloverAt +/- interval的方法
    来计算前后日志文件时间点了，只能写入时实时计算，这也是为什么不能基于TimedRotatingFileHandler简单改动
    """

    def __init__(self, filename, when='h', interval=1, backupCount=0, encoding=None, utc=False):
        """初始化，由于日志打印到带写入时间戳的文件中，需要将delay设置为True(在第一次写入时再打开)
        不这么设置的话，在FileHandler init的时候就尝试打开文件句柄，而此时尚未初始化完，
        获取日志文件名(get_filename)会失败
        """
        logging.handlers.BaseRotatingHandler.__init__(
            self, filename, 'a', encoding=encoding, delay=True)

        self.when = when.upper()
        self.backupCount = backupCount
        self.utc = utc
        # when - 切分间隔
        # S - Seconds
        # M - Minutes
        # H - Hours
        # D - Days
        if self.when == 'S':
            self.interval = 1  # one second
            self.suffixFormat = "%Y-%m-%d_%H-%M-%S"
            self.extMatch = r"^\d{4}-\d{2}-\d{2}_\d{2}-\d{2}-\d{2}(\.\w+)?$"
        elif self.when == 'M':
            self.interval = 60  # one minute
            self.suffixFormat = "%Y-%m-%d_%H-%M"
            self.extMatch = r"^\d{4}-\d{2}-\d{2}_\d{2}-\d{2}(\.\w+)?$"
        elif self.when == 'H':
            self.interval = 60 * 60  # one hour
            self.suffixFormat = "%Y-%m-%d_%H"
            self.extMatch = r"^\d{4}-\d{2}-\d{2}_\d{2}(\.\w+)?$"
        elif self.when == 'D':
            self.interval = 60 * 60 * 24  # one day
            self.suffixFormat = "%Y-%m-%d"
            self.extMatch = r"^\d{4}-\d{2}-\d{2}(\.\w+)?$"
        else:
            raise ValueError(
                "Invalid rollover interval specified: %s" % self.when)

        self.extMatch = re.compile(self.extMatch, re.ASCII)
        self.currSuffix = self.getCurrSuffix()  # 日志文件的时间后缀

    def getTimeTuple(self, ts=None):
        """自定义方法，获取时间戳，转成timeTuple"""
        if ts is None:
            ts = time.time()
        if self.utc:
            return time.gmtime(ts)
        else:
            return time.localtime(ts)

    def getCurrSuffix(self):
        """自定义方法，获取时间后缀"""
        timeTuple = self.getTimeTuple()
        return time.strftime(self.suffixFormat, timeTuple)

    def _open(self):
        """重写logging.FileHandler的_open方法
        默认打开self.baseFilename, 改为打开带时间后缀的文件
        """
        newFn = self.rotation_filename(
            self.baseFilename + "." + self.currSuffix)
        return open(newFn, self.mode, encoding=self.encoding)

    def shouldRollover(self, record):
        """重写logging.handlers.BaseRotatingHandler的shouldRollover
        判断是否应该切分(伪)"""
        newSuffix = self.getCurrSuffix()
        if self.currSuffix != newSuffix:
            return 1
        return 0

    def doRollover(self):
        """重写logging.handlers.BaseRotatingHandler的doRollover方法"""
        # 关闭原有日志写入
        if self.stream:
            self.stream.close()
            self.stream = None
        # 清理超出保留上限的文件
        if self.backupCount > 0:
            for s in self.getFilesToDelete():
                os.remove(s)
        # 设置新的时间段, 在实际写入时再打开对应文件
        self.currSuffix = self.getCurrSuffix()

    def getFilesToDelete(self):
        """参考logging.handlers.TimeRotatingFileHandler编写
        那个实现中，backup只保留严格的文件数，如果中间有时间未请求
        那保留的文件时间轴会更长
        跟一般理解的保留N个when周期的日志，有点区别，这里按后者实现
        """
        # 计算删除截止时间戳(在这时间后的日志都保留)
        tsDeleteFrom = time.time() - self.interval * self.backupCount
        # 列举日志文件
        dirName, baseName = os.path.split(self.baseFilename)
        fileNames = os.listdir(dirName)
        result = []
        prefix = baseName + "."
        plen = len(prefix)
        for fileName in fileNames:
            # 匹配日志文件前缀
            if fileName[:plen] == prefix:
                # 获取日志文件后缀
                suffix = fileName[plen:]
                # 后缀格式不匹配，跳过
                if not self.extMatch.match(suffix):
                    continue
                # 比较后缀时间戳，是否在截止时间前
                suffixTs = time.mktime(
                    time.strptime(suffix, self.suffixFormat))
                if suffixTs <= tsDeleteFrom:
                    result.append(os.path.join(dirName, fileName))
        # 排序，从前往后删
        result.sort()
        return result


def get_logger(logger_name, sub_dir='', file_name=''):
    """获取一个用logger_name标识的logger

    Args:
        logger_name: logger的唯一标识
        sub_dir: 存放在日志文件夹的子目录，不设置默认在根目录
        file_name: 日志名称，不设置默认等同于logger_name
    """
    if not isinstance(logger_name, str):
        logger_name = 'default'
    if '' == logger_name:
        logger_name = 'default'
    # logger已定义
    if logger_name in loggers:
        return loggers[logger_name]
    # logger未定义，新建
    logger = logging.getLogger(logger_name)
    log_conf = conf.get_log_conf()
    if '' == file_name:
        file_name = logger_name
    log_path = "{}/{}/{}".format(log_conf['dir'], sub_dir, file_name)
    level = get_log_level(log_conf['level'])
    init_log(
        logger, log_path, level,
        log_conf['when'], log_conf['backup'],
        log_conf['format'], log_conf['datefmt']
    )
    # 加入到loggers
    loggers[logger_name] = logger
    return logger


def get_log_level(log_level):
    """get log level by config string"""
    log_level_dict = {
        'debug': logging.DEBUG,
        'info': logging.INFO,
        'warning': logging.WARNING,
        'error': logging.ERROR,
        'critical': logging.CRITICAL,
    }
    if log_level in log_level_dict:
        return log_level_dict[log_level]
    return logging.INFO


def init_log(this_logger, log_path, level=logging.INFO, when="H", backup=7 * 24,
             format="%(levelname)s: %(asctime)s: %(filename)s:%(lineno)d * %(thread)d %(message)s",
             datefmt="%y-%m-%d %H:%M:%S"):
    """
    refer to http://styleguide.baidu.com/style/python/index.html#%E7%BC%96%E7%A8%8B%E5%AE%9E%E8%B7%B5id9
    TODO: 这里的按小时切分有延迟
    init_log - initialize log module.
    Args:
      log_path      - Log file path prefix.
                      Log data will go to two files: log_path.log and log_path.log.wf
                      Any non-exist parent directories will be created automatically
      level         - msg above the level will be displayed
                      DEBUG < INFO < WARNING < ERROR < CRITICAL
                      the default value is logging.INFO
      when          - how to split the log file by time interval
                      'S' : Seconds
                      'M' : Minutes
                      'H' : Hours
                      'D' : Days
                      'W' : Week day
                      default value: 'D'
      format        - format of the log
                      default format:
                      %(levelname)s: %(asctime)s: %(filename)s:%(lineno)d * %(thread)d %(message)s
                      INFO: 12-09 18:02:42: log.py:40 * 139814749787872 HELLO WORLD
      backup        - how many backup file to keep
                      default value: 7
    Raises:
        OSError: fail to create log directories
        IOError: fail to open log file
    """
    formatter = logging.Formatter(format, datefmt)
    this_logger.setLevel(level)
    dir = os.path.dirname(log_path)
    if not os.path.isdir(dir):
        os.makedirs(dir)
    # add notice log
    handler = MPTimeRotatingFileHandler(
        log_path + ".log", when=when, backupCount=backup)
    handler.setLevel(level)
    handler.setFormatter(formatter)
    this_logger.addHandler(handler)
    # add wf log
    handler = MPTimeRotatingFileHandler(
        log_path + ".log.wf", when=when, backupCount=backup)
    handler.setLevel(logging.WARNING)
    handler.setFormatter(formatter)
    this_logger.addHandler(handler)

```

## propagate 作用

propagate 会把当前的`logger`设置为其`parent`, 并将`record`传入`parent`的 Handler

我们没必要配置子代的`Handler`, 因为最终虽有的`record`都会被转发到`root`, 我们只需要配置它就可以了.

![img](/Users/yanjigang01/Golang-Backend/python/image/logging_flow.png)


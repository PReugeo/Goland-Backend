```go
package log

import (
	"fmt"
	"io"
	"os"
	"path/filepath"
	"strings"
	"sync"
	"time"
)

// CalRotateTimeDuration 计算当前时间到下一个整数 duration 的时间差
func CalRotateTimeDuration(now time.Time, duration time.Duration) time.Duration {
	nowUnixNano := now.UnixNano()
	durationUnixNano := duration.Nanoseconds()
	return time.Duration(durationUnixNano - nowUnixNano%durationUnixNano)
}

// RotateLogger 定义了一个日志切割的接口，要求实现一个带有时间流转的日志切割方法
type RotateLogger interface {
	NewRotateLog(logFilePath string, options ...Option) (io.WriteCloser, error)
	rotateFile(now time.Time) error
}

type RotateLog struct {
	file *os.File

	logPath            string
	curLogLinkPath     string
	rotateTime         time.Duration
	maxAge             time.Duration
	deleteFileWildcard string

	mutex sync.Mutex
	// notify rotate event
	rotate <-chan time.Time
	// close file and write goroutine
	close chan struct{}
}

type Option func(*RotateLog)

// WithRotateTime rotate log by duration
func WithRotateTime(duration time.Duration) Option {
	return func(rl *RotateLog) {
		rl.rotateTime = duration
	}
}

// WithDeleteExpiredFile Judge expired by last modify time
// cutoffTime = now - maxAge
// Only delete satisfying file wildcard filename
func WithDeleteExpiredFile(maxAge time.Duration, fileWildcard string) Option {
	return func(rl *RotateLog) {
		rl.maxAge = maxAge
		rl.deleteFileWildcard = fmt.Sprintf("%s%s%s",
			filepath.Dir(rl.logPath), string(os.PathSeparator), fileWildcard)
	}
}

func (r *RotateLog) NewRotateLog(logPath string, opts ...Option) (io.WriteCloser, error) {
	rl := &RotateLog{
		close:   make(chan struct{}, 1),
		logPath: logPath,
	}
	for _, opt := range opts {
		opt(rl)
	}

	if err := os.Mkdir(filepath.Dir(rl.logPath), 0755); err != nil && !os.IsExist(err) {
		return nil, err
	}

	if err := rl.rotateFile(time.Now()); err != nil {
		return nil, err
	}

	if rl.rotateTime != 0 {
		go rl.handleEvent()
	}
	return rl, nil
}

func (r *RotateLog) Write(b []byte) (int, error) {
	r.mutex.Lock()
	defer r.mutex.Unlock()

	n, err := r.file.Write(b)
	return n, err
}

func (r *RotateLog) Close() error {
	r.mutex.Lock()
	defer r.mutex.Unlock()

	r.close <- struct{}{}
	return r.file.Close()
}

func (r *RotateLog) handleEvent() {
	for {
		select {
		case <-r.close:
			return
		case now := <-r.rotate:
			err := r.rotateFile(now)
			if err != nil {
				return
			}
		}
	}
}

func (r *RotateLog) rotateFile(now time.Time) error {
	if r.rotateTime != 0 {
		nextRotateTime := CalRotateTimeDuration(now, r.rotateTime)
		r.rotate = time.After(nextRotateTime)
	}
	r.mutex.Lock()
	defer r.mutex.Unlock()

	latestLogPath := r.getLatestLogPath(now.Add(-r.rotateTime))

	// 如果 curLogLinkPath 存在，则移动到 latestLogPath 并新建 curLogLinkPath
	if len(r.curLogLinkPath) > 0 {
		err := os.Rename(r.curLogLinkPath, latestLogPath)
		if err != nil {
			return err
		}
		r.curLogLinkPath = ""
	}

	if len(r.curLogLinkPath) == 0 {
		r.curLogLinkPath = strings.TrimSuffix(latestLogPath, filepath.Ext(latestLogPath))
	}

	file, err := os.OpenFile(r.curLogLinkPath, os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
	if err != nil {
		return err
	}
	if r.file != nil {
		err := r.file.Close()
		if err != nil {
			return err
		}
	}
	r.file = file

	if r.maxAge > 0 && len(r.deleteFileWildcard) > 0 {
		// 根据文件最后修改时间以及通配符删除过期文件
		cutoffTime := now.Add(-r.maxAge)
		matches, err := filepath.Glob(r.deleteFileWildcard)
		if err != nil {
			return err
		}

		toUnlink := make([]string, 0, len(matches))
		for _, path := range matches {
			fileInfo, err := os.Stat(path)
			if err != nil {
				continue
			}

			if r.maxAge > 0 && fileInfo.ModTime().After(cutoffTime) {
				continue
			}

			if len(r.curLogLinkPath) > 0 && fileInfo.Name() == filepath.Base(r.curLogLinkPath) {
				continue
			}
			toUnlink = append(toUnlink, path)
		}
		// at present
		go func() {
			for _, path := range toUnlink {
				err := os.Remove(path)
				if err != nil {
					return
				}
			}
		}()
	}

	return nil
}

func (r *RotateLog) getLatestLogPath(now time.Time) string {
	return now.Format(r.logPath)
}
```


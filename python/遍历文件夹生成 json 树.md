代码

```python
import sys
from pathlib import Path

class DirectionTree(object):
    def searchAllFiles(self, path, fileTree={}, initial=True):
        p = Path(path if isinstance(path, str) else str(path))
        dirs = [x for x in p.iterdir() if x.is_dir()]
        files = [x for x in p.iterdir() if x.is_file()]
        fileList = []
        for file in files:
            if file.name == '.DS_Store':
                continue
            fileList.append({'pathName': file.as_posix(), 'name': file.name})
        for dir in dirs:
            fileTree['pathName'] = dir.as_posix()
            fileTree['name'] = dir.name
            if 'children' not in fileTree:
                fileTree['children'] = []
            fileTree['children'] = (self.searchAllFiles(dir, {}, initial=initial))
            fileList.append(fileTree)
            fileTree = {}
        return fileList
```


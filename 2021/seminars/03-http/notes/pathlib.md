# Pathlib

[Docs](https://docs.python.org/3/library/pathlib.html)

If you’ve never used this module before or just aren’t sure which class is right for your task, `Path` is most likely what you need.

## Basic use
```python
from pathlib import Path
```
Listing subdirectories:
```python
>>> p = Path('.')
>>> [x for x in p.iterdir() if x.is_dir()]

[PosixPath('.hg'), PosixPath('docs'), PosixPath('dist'),
 PosixPath('__pycache__'), PosixPath('build')
```
Listing Python source files in this directory tree:
```python
>>> list(p.glob('**/*.py'))

[PosixPath('test_pathlib.py'), PosixPath('setup.py'),
 PosixPath('pathlib.py'), PosixPath('docs/conf.py'),
 PosixPath('build/lib/pathlib.py')]
```
Navigating inside a directory tree:

```python
>>> p = Path('/etc')
>>> q = p / 'init.d' / 'reboot'
>>> q
PosixPath('/etc/init.d/reboot')
>>> q.resolve()
PosixPath('/etc/rc.d/init.d/halt')
```
Querying path properties:
```python
>>> q.exists()
True
>>> q.is_dir()
False
```
Opening a file:
```python
>>> with q.open() as f: f.readline()
...
'#!/bin/bash\n'
```

## General properties

- Paths are immutable and hashable
- Paths of a different flavour compare unequal and cannot be ordered

## Operators

The slash operator helps create child paths
```python
>>> p = PurePath('/etc')
>>> p
PurePosixPath('/etc')
>>> p / 'init.d' / 'apache2'
PurePosixPath('/etc/init.d/apache2')
>>> q = PurePath('bin')
>>> '/usr' / q
PurePosixPath('/usr/bin')
```
A path object can be used anywhere an object implementing os.PathLike is accepted:
```python
>>> import os
>>> p = PurePath('/etc')
>>> os.fspath(p)
'/etc'
```
The string representation of a path is the raw filesystem path itself (in native form, e.g. with backslashes under Windows), which you can pass to any function taking a file path as a string:
```python
>>> p = PurePath('/etc')
>>> str(p)
'/etc'
>>> p = PureWindowsPath('c:/Program Files')
>>> str(p)
'c:\\Program Files'
```
Similarly, calling `bytes` on a path gives the raw filesystem path as a bytes object, as encoded by `os.fsencode()`:

```python
>>> bytes(p)
b'/etc'
```

## Accessing individual parts

To access the individual “parts” (components) of a path, use the following property:

```python
>>> p = PurePath('/usr/bin/python3')
>>> p.parts
('/', 'usr', 'bin', 'python3')

>>> p = PureWindowsPath('c:/Program Files/PSF')
>>> p.parts
('c:\\', 'Program Files', 'PSF')
```

## Methods and properties¶

### PurePath.**root**
A string representing the (local or global) root, if any:
```python
>>> PureWindowsPath('c:/Program Files/').root
'\\'
>>> PureWindowsPath('c:Program Files/').root
''
>>> PurePosixPath('/etc').root
'/'
```

### PurePath.**parents**
```python
>>> p = PureWindowsPath('c:/foo/bar/setup.py')
>>> p.parents[0]
PureWindowsPath('c:/foo/bar')
>>> p.parents[1]
PureWindowsPath('c:/foo')
>>> p.parents[2]
PureWindowsPath('c:/')
```
### PurePath.**parent**
```python
>>> p = PurePosixPath('/a/b/c/d')
>>> p.parent
PurePosixPath('/a/b/c')
```
```python
>>> p = PurePosixPath('/')
>>> p.parent
PurePosixPath('/')
>>> p = PurePosixPath('.')
>>> p.parent
PurePosixPath('.')
```
### PurePath.**name**
```python
>>> PurePosixPath('my/library/setup.py').name
'setup.py'
```
### PurePath.**suffix**
```python
>>> PurePosixPath('my/library/setup.py').suffix
'.py'
>>> PurePosixPath('my/library.tar.gz').suffix
'.gz'
>>> PurePosixPath('my/library').suffix
''
```
### PurePath.**stem**
```python
>>> PurePosixPath('my/library.tar.gz').stem
'library.tar'
>>> PurePosixPath('my/library.tar').stem
'library'
>>> PurePosixPath('my/library').stem
'library'
```

## Methods

### _classmethod_ Path.**cwd()**
Return a new path object representing the current directory
```python
>>> Path.cwd()
PosixPath('/home/antoine/pathlib')
```
### Path.**iterdir()**
```python
>>> p = Path('docs')
>>> for child in p.iterdir(): child
...
PosixPath('docs/conf.py')
PosixPath('docs/_templates')
PosixPath('docs/make.bat')
PosixPath('docs/index.rst')
PosixPath('docs/_build')
PosixPath('docs/_static')
PosixPath('docs/Makefile')
```
### Path.**mkdir**(mode=511, parents=False, exist_ok=False)
### Path.**open**(mode='r', buffering=- 1, encoding=None, errors=None, newline=None)
### Path.**read_bytes**()
```python
>>> p = Path('my_binary_file')
>>> p.write_bytes(b'Binary file contents')
20
>>> p.read_bytes()
b'Binary file contents'
```
### Path.**read_text**(encoding=None, errors=None)
```python
>>> p = Path('my_text_file')
>>> p.write_text('Text file contents')
18
>>> p.read_text()
'Text file contents'
```
### Path.**resolve**(strict=False)
```python
>>> p = Path()
>>> p
PosixPath('.')
>>> p.resolve()
PosixPath('/home/antoine/pathlib')
>>> p = Path('docs/../setup.py')
>>> p.resolve()
PosixPath('/home/antoine/pathlib/setup.py')
```
### Path.**touch**(mode=438, exist_ok=True)
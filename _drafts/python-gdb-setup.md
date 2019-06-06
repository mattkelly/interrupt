---
title: "Using Python's virtualenv with GDB"
author: tyler
---

<!-- excerpt start -->

GNU's GDB needs a little help to make it work well with a Python virtual environment, Conda environment, or with user-space Python installation. By adding a short snippet into your `.gdbinit` file, you can get up and running in a few minutes. 

<!-- excerpt end -->

In future posts, we'll go over useful tricks for using Python and GDB together, but this post is about getting the required bits set up and understanding why it is required. If you want to skip to the required setup, it's at the bottom of the post!

### The Goal

I want to use the PrettyTable package in a Python GDB script I am wanting to build. 

> For the purposes of this blog post, we are assuming my GDB version was compiled with Python2 and my local OSX brew'ed python installation was also Python2. This is primarily because the GNU Arm Embedded Toolchain is still compiled with the Mac OS X system Python and that happens to be Python2.

### Initial Attempt

We activate our virtual environment (or use the active Python version) and install the package:

```terminal
$ source ~/.venvs/venv/bin/activate

(venv)
$ pip install prettytable
Collecting prettytable
  Downloading https://files.pythonhosted.org/packages/ef/30/4b0746848746ed5941f052479e7c23d2b56d174b82f4fd34a25e389831f5/prettytable-0.7.2.tar.bz2
Installing collected packages: prettytable
  Running setup.py install for prettytable ... done
Successfully installed prettytable-0.7.2
```

Great! We double check that it's installed and everything checks out. 

```terminal
(venv)
$ python2
Python 2.7.15 (default, Feb 12 2019, 11:00:12)
[GCC 4.2.1 Compatible Apple LLVM 10.0.0 (clang-1000.11.45.5)] on darwin
>>> import prettytable
>>> # It worked!
```

However, when we load our GDB executable, the package isn't there, regardless of whether our virtual environment was activated when we ran GDB.

```terminal
(venv)
$ arm-none-eabi-gdb-py
GNU gdb (GNU Tools for Arm Embedded Processors 8-2018-q4-major) 8.2.50.20181213-git
...
(gdb) python-interactive 
>>> import prettytable
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ImportError: No module named prettytable
```

> One can also run `pi` , which is short for `python-interactive`


### Investigation

If we compare the outputs of `sys.path` in both instances of Python, you can see that they do indeed differ.

#### Virtual Environment

My normal shell Python2 is importing packages from the Brew Python installation (2.7.15) and the virtual environment that I have activated currently

```terminal
(venv)
$ python2
Python 2.7.15 (default, Feb 12 2019, 11:00:12)
[GCC 4.2.1 Compatible Apple LLVM 10.0.0 (clang-1000.11.45.5)] on darwin
>>> import sys
>>> sys.version
'2.7.15 (default, Feb 12 2019, 11:00:12) \n[GCC 4.2.1 Compatible Apple LLVM 10.0.0 (clang-1000.11.45.5)]'
>>> sys.path
['',
 '/Users/tyler/venv/lib/python27.zip',
 '/Users/tyler/venv/lib/python2.7',
 '/Users/tyler/venv/lib/python2.7/plat-darwin',
 '/Users/tyler/venv/lib/python2.7/plat-mac',
 '/Users/tyler/venv/lib/python2.7/plat-mac/lib-scriptpackages',
 '/Users/tyler/venv/lib/python2.7/lib-tk',
 '/Users/tyler/venv/lib/python2.7/lib-old',
 '/Users/tyler/venv/lib/python2.7/lib-dynload',
 '/usr/local/Cellar/python@2/2.7.15_3/Frameworks/Python.framework/Versions/2.7/lib/python2.7',
 '/usr/local/Cellar/python@2/2.7.15_3/Frameworks/Python.framework/Versions/2.7/lib/python2.7/plat-darwin',
 '/usr/local/Cellar/python@2/2.7.15_3/Frameworks/Python.framework/Versions/2.7/lib/python2.7/lib-tk',
 '/usr/local/Cellar/python@2/2.7.15_3/Frameworks/Python.framework/Versions/2.7/lib/python2.7/plat-mac',
 '/usr/local/Cellar/python@2/2.7.15_3/Frameworks/Python.framework/Versions/2.7/lib/python2.7/plat-mac/lib-scriptpackages',
 '/Users/tyler/venv/lib/python2.7/site-packages']
```

#### GDB

GDB is importing packages from the Mac OS X System Python (Python 2.7.10)

```terminal
(venv)
$ arm-none-eabi-gdb-py
GNU gdb (GNU Tools for Arm Embedded Processors 8-2018-q4-major) 8.2.50.20181213-git
...
(gdb) pi
>>> import sys
>>> sys.version
'2.7.10 (default, Aug 17 2018, 19:45:58) \n[GCC 4.2.1 Compatible Apple LLVM 10.0.0 (clang-1000.0.42)]'
>>> sys.path
['/Users/tyler/.local/gcc-arm-none-eabi-8-2018-q4-major/arm-none-eabi/share/gdb/python',
 '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python27.zip',
 '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7',
 '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/plat-darwin',
 '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/plat-mac',
 '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/plat-mac/lib-scriptpackages',
 '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/lib-tk',
 '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/lib-old',
 '/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/lib-dynload',
 '/Library/Python/2.7/site-packages',
 '/System/Library/Frameworks/Python.framework/Versions/2.7/Extras/lib/python',
 '/System/Library/Frameworks/Python.framework/Versions/2.7/Extras/lib/python/']
 ```

### Solution

So, how do I get my `sys.path` fixed for the Python within GDB so I can import PrettyTable?!

We need to, on initialization, take our shell's Python installation `sys.path` (virtualenv or Brew version) and append it to the one running with GDB.

To output that within a Python script, it can be as simple as the following:

```terminal
(venv)
$ python -c "import sys;print(sys.path)"
['', '/Users/tyler/venv/lib/python27.zip', ...
```

Let's apply this same strategy to our GDB instance now.

In your `~/.gdbinit` script, you can place this at the end and any future Python packages should be able to be found in your virtual environment's `site-packages` folder!

```terminal
python
import sys
import subprocess
paths = eval(subprocess.check_output('python -c "import sys;print(sys.path)"', shell=True).strip())
sys.path.extend(paths)
end
```

After appending the above to your `~/.gdbinit` script, we can test it out by trying the following:

```terminal
(venv)
$ arm-none-eabi-gdb-py
GNU gdb (GNU Tools for Arm Embedded Processors 8-2018-q4-major) 8.2.50.20181213-git
...
(gdb) pi
>>> import prettytable
>>> # Yay! It works.
```

### Explanation

If the GDB executable being used was downloaded rather than compiled from source, it's likely that
it is linked against the system's Python library and packages, rather than against a virtual 
environment, Conda environment, or another user installed flavor of Python. 

A commonly suggested solution is to either edit the `PYTHONPATH` variable or create a symlink between
the system Python and virtualenv, but both of these will break various versions of Conda, GDB, 
and on different operating systems. We want something that affects *only* GDB's Python context.

The best solution is to shell out from within GDB, ask the local shell environment what the configured
Python executable is, get its `sys.path` entries, and then append those to our current GDB session's
Python environment. This allows us to use Conda, virtualenv, pyenv, and anything similar. 


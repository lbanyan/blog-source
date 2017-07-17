---
title: WIN 10下PyCharm安装包时报Permission denied
---

## 错误trace
```
running install

error: can't create or remove files in install directory

The following error occurred while trying to add or remove files in the
installation directory:

    [Errno 13] Permission denied: 'C:\\Program Files (x86)\\python\\Lib\\site-packages\\test-easy-install-61304.write-test'

The installation directory you specified (via --install-dir, --prefix, or
the distutils default setting) was:

    C:\Program Files (x86)\python\Lib\site-packages\

Perhaps your account does not have write access to this directory?  If the
installation directory is a system-owned directory, you may need to sign in
as the administrator or "root" account.  If you do not have administrative
access to this machine, you may wish to choose a different installation
directory, preferably one that is listed in your PYTHONPATH environment
variable.

For information on other options, you may wish to consult the
documentation at:

  https://setuptools.readthedocs.io/en/latest/easy_install.html

Please make the appropriate changes for your system and try again.
```

## 问题分析
WIN 10对C:\\\\Program Files (x86)和C:\\\\Program Files目录下有权限限制，可以使用管理员权限打开PyCharm就可以安装正常了。

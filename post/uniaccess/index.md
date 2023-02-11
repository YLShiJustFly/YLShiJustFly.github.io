
# 为什么要卸载UniAcess?

   很多公司用联软的UniAcess监控员工电脑，后台运行的时候占用大量CPU和网络资源，
上网速度受到很大影响，甚至造成电脑卡顿。这个软件本身是个流氓软件，
win版和mac版卸载都很麻烦，win版网上能搜到卸载方法，但mac版的现在似乎搜不到。
现总结mac版的卸载方法如下，仅供参考。测试机macos是mac big sur版本。

1. 首先关闭电脑file vault。

- 方法1，终端输入sudo fdesetup disable，然后输入电脑密码。
- 方法2，系统偏好设置->安全性与隐私->文件保险箱，点解锁->输入密码->关闭文件保险箱。

2. 完全关闭系统sip。

关机，长按command+R进入恢复模式，选择用户，输入密码。点上面实用工具->终端，在终端中输入

```
sudo csrutil disable
```

和：

```
sudo csrutil authenticated-root disable
```

重启。

3. 获取修改UniAccess软件的权限。

- 11.0.1系统

进入系统，在终端中用户有权限操作的目录，比如/Users/xxx，其中xxx为用户名，下面新建一个目录，

```
mkdir -p /Users/xxx/myroot
```

其中，myroot为随意的名字。

然后需要获取硬盘名。

- 方法1，访达->应用程序->实用工具->磁盘工具，找到disk开头的字符，例如我的是disk1s1s1，最后的s1是快照号，舍去，得到磁盘名disk1s1。
- 方法2，终端执行

```
df -h
```

得到最后一列是Mount在/根目录的磁盘名/dev/disk1s1s1，同样舍去s1，可以得到disk1s1。

然后终端执行

```
sudo mount -o nobrowse -t apfs /dev/disk1s1 /Users/xxx/myroot
```

其中，disk1s1是你的硬盘名。把磁盘挂载到用户有读写执行权限的/Users/xxx/myroot。然后执行

```
sudo bless --folder /Users/xxx/myroot/System/Library/CoreServices --bootefi --create-snapshot
```

来恢复快照。

重启。

切记：你的硬盘整个都挂载到了/Users/xxx/myroot下，不熟悉macos系统的话最好不要操作里面的文件。

- 10.15系统及以前系统：终端执行

```
sudo mount -uw / 
```

重新挂载根目录。

4. 杀死UniAcess进程。

找到UniAccess进程，我的是dvc打头的一个名字，终端执行

```
pkill -9 dvc
```

把UniAcess进程杀掉。然后执行全局搜索并把结果写到随意一个文件：

```
find / -type f | grep -i dvc > a
```

里面应该有关于卸载时密码验证的文件，这些文件都要删掉的。为防止误删，把与UniAcess无关的行删掉，我的有adobe和iMovie的几个文件，把这些行删掉。然后终端执行

```
rm -rf `cat a`
```

全部删掉。

5. 卸载UniAccess

cd到你安装UniAcess的目录。11.0.1系统：在/Users/xxx/myroot/下面。10.15系统：在/根目录。我的是/Users/xxx/myroot/opt/LVUAAgentInstBaseRoot，找到卸载的可执行文件，我的是uninstall-exe。执行

```
./uninstall-exe
```

需要输入密码就可以卸载了。

6. 重新打开file vault和系统完整性保护。

- 先清SMC和NVRAM，这个网上教程很多。
- 关机，按command+R进入恢复模式，执行

```
sudo csrutil enable
```

和：

```
sudo csrutil authenticated-root enable
```

重新开启sip。

- 开机进入系统，重新打开file vault。

安装官网对应的显卡驱动[^1]

下载驱动
```
wget http://cn.download.nvidia.com/XFree86/Linux-x86_64/367.44/NVIDIA-Linux-x86_64-367.44.run
```

关闭开源驱动nouveau
```
subl /etc/modprobe.d/blacklist.conf

添加内容如下：
blacklist vga16fb

blacklist nouveau

blacklist rivafb

blacklist nvidiafb

blacklist rivatv

```

删除卸载旧NVIDIA驱动:

```
sudo apt-get –purge remove nvidia-*(需要清除干净)
sudo apt-get –purge remove xserver-xorg-video-nouveau
```

重启系统`sudo reboot`，按ctrl+alt+F1进入命令行界面(tty1),执行如下命令：  

检查nouveau driver确保没有被加载:  

```
lsmod | grep nouveau
```


```
sudo service lightdm stop
chmod a+x NVIDIA-Linux-x86_64-367.44.run
sudo ./NVIDIA-Linux-x86_64-367.44.run --update
```

卸载驱动的命令

```
sudo ./NVIDIA-Linux-x86_64-367.44.run --uninstall
```


使用如下命令安装CUDA

```
wget http://developer.download.nvidia.com/compute/cuda/7.5/Prod/local_installers/cuda-repo-ubuntu1404-7-5-local_7.5-18_amd64.deb
sudo dpkg -i cuda-repo-ubuntu1404-7-5-local_7.5-18_amd64.deb
sudo apt-get update
sudo apt-get install -y cuda
```

安装好CUDA后要卸载nvidia-352驱动,否则运行会出错

```
sudo apt-get remove --purge nvidia-352
```

[^1]: http://www.wxhp.org/ubuntu-install-nvidia-official-drivers.html

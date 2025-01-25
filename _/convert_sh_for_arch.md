# 将脚本修改为Arch Linux版


##### User:


```
#!/bin/bash
sudo ./system_backup.sh

hw_version=`tr -d '\0' < /proc/device-tree/model`

source ./system_config.sh
sudo echo "hdmi_force_hotplug=1" >> ./boot/config.txt.bak
sudo echo "hdmi_force_edid_audio=1" >> ./boot/config.txt.bak
sudo echo "dtparam=i2c_arm=on" >> ./boot/config.txt.bak
sudo echo "dtparam=spi=on" >> ./boot/config.txt.bak
sudo echo "enable_uart=1" >> ./boot/config.txt.bak
sudo echo "display_rotate=0" >> ./boot/config.txt.bak
sudo echo "max_usb_current=1" >> ./boot/config.txt.bak
sudo echo "config_hdmi_boost=7" >> ./boot/config.txt.bak
sudo echo "hdmi_group=2" >> ./boot/config.txt.bak
sudo echo "hdmi_mode=1" >> ./boot/config.txt.bak
sudo echo "hdmi_mode=87" >> ./boot/config.txt.bak
sudo echo "hdmi_drive=2" >> ./boot/config.txt.bak
sudo echo "hdmi_cvt 480 320 60 6 0 0 0" >> ./boot/config.txt.bak
sudo echo "dtoverlay=ads7846,cs=1,penirq=25,penirq_pull=2,speed=50000,keep_vref_on=0,swapxy=0,pmax=255,xohms=150,xmin=200,xmax=3900,ymin=200,ymax=3900" >> ./boot/config.txt.bak
[[ $hw_version =~ "Raspberry Pi 4" ]] && sudo echo "hdmi_timings=600 0 20 28 48 400 0 13 3 32 0 0 0 30 0 25000000 5" >> ./boot/config.txt.bak
sudo cp -rf ./boot/config.txt.bak /boot/config.txt
sudo cp ./usr/inittab /etc/
if [[ "$deb_version" < "12.1" ]]; then
sudo cp -rf ./usr/99-fbturbo.conf-HDMI /usr/share/X11/xorg.conf.d/99-fbturbo.conf 
fi
if [ ! -d /etc/X11/xorg.conf.d ]; then
sudo mkdir /etc/X11/xorg.conf.d 
fi
sudo cp -rf ./usr/99-calibration.conf-3508-0 /etc/X11/xorg.conf.d/99-calibration.conf
sudo touch ./.have_installed
echo "hdmi:resistance:3508:0:480:320" > ./.have_installed
version=`uname -v`
input_result=0
version=${version##* }
echo $version
if test $version -lt 2017;then
echo "reboot"
else
echo "need to update touch configuration"
wget --spider -q -o /dev/null --tries=1 -T 10 http://mirrors.zju.edu.cn/raspbian/raspbian
if [ $? -ne 0 ]; then
input_result=1
else
sudo apt-get install xserver-xorg-input-evdev  2> error_output.txt
dpkg -l | grep xserver-xorg-input-evdev > /dev/null 2>&1
if [ $? -ne 0 ]; then
input_result=1
fi
fi
if [ $input_result -eq 1 ]; then 
if [ $hardware_arch -eq 32 ]; then
sudo dpkg -i -B ./xserver-xorg-input-evdev_1%3a2.10.6-1+b1_armhf.deb 2> error_output.txt
elif [ $hardware_arch -eq 64 ]; then
sudo dpkg -i -B ./xserver-xorg-input-evdev_1%3a2.10.6-2_arm64.deb 2> error_output.txt
fi
fi
result=`cat ./error_output.txt`
echo -e "\033[31m$result\033[0m"
grep -q "error:" ./error_output.txt && exit
sudo cp -rf /usr/share/X11/xorg.conf.d/10-evdev.conf /usr/share/X11/xorg.conf.d/45-evdev.conf
fi

sudo sync
sudo sync
sleep 1
if [ $# -eq 1 ]; then
sudo ./rotate.sh $1
elif [ $# -gt 1 ]; then
echo "Too many parameters"
fi

echo "reboot now"
sudo reboot
```

This .sh is used for enable display monitor hardware with cm-4 for ubuntu22 or rpios. can rewrite it for used for arch linux?

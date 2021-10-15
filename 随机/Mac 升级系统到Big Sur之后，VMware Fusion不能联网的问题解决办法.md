## 问题如下：
* mac更新到big sur后虚拟机vm无法上网

## 解决办法：

三种方法（可以都试一下，楼主是第二个方法可以的）：

* 1、sudo rm /Library/Preferences/SystemConfiguration/NetworkInterfaces.plist && sudo killall -9 configd
* 2、将vm内dns与网关最后一位的2改为1
  ```
  vim /etc/sysonfig/network-scripts/ifcfg-ens32
  将DNS最后一位修改为2
  DNS=xxx.xxx.xxx.2
  ```
* 3、将vm内dns改为8.8.8.8

注意： 方法2和3均需要重启网卡。
```
systemctl restart network
```
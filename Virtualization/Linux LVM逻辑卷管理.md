![image](http://img1.51cto.com/attachment/201101/4/847418_1294140892kbAv.png)
-----------
*创建LVM卷，从下到上创建*
1. 文件系统
----------
2. 初始化为物理卷pv
>   pvcreate /dev/sdb1

>   查看pv pvs, pvdisplay
----------
3. 创建卷组VG，添加PV到卷组

```
vgcreate [-s=16m] vgname /dev/sdk1  /dev/sdl1
```
- 查看vg ,vgdisply
- 激活vg, vgchange -a y vgname
- 休眠vg, vgchange -a n vgname
- 移除vg, vgremov vgname
- 新增pv, vgextent /dev/sdb2
- 移除pv, 先确认pv 是否使用 
    ```
    pvdisplay  /dev/sdb15
    vgreduce  vgname  /dev/sdb2
    ```
    *如果pv 上有数据，先转移数据*
    
    ```
    pvmove /dev/sdb2 /dev/sdb1
    ```

4. 在VG卷组上创建LV逻辑卷 
- 创建 lvcreate  -L 500M  -n lv0  vgname
- 查看lvs,vgs
- 格式化lv: mkfs.ext3  /dev/vgname/lv0
- 查看 ll /dev/vgname/lv0
- 删除lv, umount , lvremove /dev/vgname/lv0

5. 扩大lv

![image](http://img1.51cto.com/attachment/201101/4/847418_1294140902VZf5.png)
- lvextend -L 2G /dev/vgname/lv0

6 缩小lv

- 卸载文件系统 umount /lv0
- 检查文件系统 e2fsck -f /dev/vgname/lv0
- 缩小文件系统 在缩小LV之前首先减小文件系统，否则将可能导致数据丢失
> resize2fs /dev/vg0/lv0  500M
- 缩小lv 
> vreduce -L 500M /dev/vg0/lv0
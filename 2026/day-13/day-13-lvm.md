#  Task 1: Check Current Storage
---
Run: 
-` lsblk`
![alt text](lsblk.png)
- `pvs`
-  `vgs`
-  `lvs`
-  `df -h`
-  
![alt text](df-h.png)

# Task 2: Create Physical Volume
```bash
pvcreate /dev/sdb   # or your loop device
pvs
```
![alt text](pvs.png)
# Task 3: Create Volume Group
```bash
vgcreate gurudeen_vg /dev/sdb /dev/sdc
vgs
```
![alt text](lvs.png)
# Task 4: Create Logical Volume
```
lvcreate -L 500M -n aman gurudeen_vg
lvs
```
![alt text](lvs.png)
# Task 5: Format and Mount
```bash
mkfs.ext4 /dev/gurudeen/aman 
mkdir -p /mnt/my_lvm
mount /dev/gurudeen/aman /mnt/my_lvm
df -h /mnt/my_lvm

```
# Task 6: Extend the Volume
```bash 
lvextend -L +200M /dev/gurudeen/aman
df -h /mnt/my_lvm
```

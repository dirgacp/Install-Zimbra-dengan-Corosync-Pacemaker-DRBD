# Install Zimbra dengan Corosync, Pacemaker dan DRBD
Setting hostname pada kedua node
  ```bash
root@node1 ~]# vi /etc/hostname
node1.localdomain.com

[root@node2 ~]# vi /etc/hostname
node2.localdomain.com
  ```
Edit SElinux menjadi permissive pada file /etc/selinux/config di kedua node
  ```bash
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=permissive
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
  ```
Edit file /etc/hosts, tambahkan IP dan hostname pada kedua node
```bash
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.7.60.182 node1.localdomain.com node1
10.7.60.184 node2.localdomain.com node2
10.7.60.186 mail.localdomain.com mail
```
## Konfigurasi DRBD
Download repositori ELRepo dan import kunci paket ELRepo pada kedua node
```bash
# rpm -ivh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
# rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org
```
Install modul dan utilitas kernel DRBD di kedua node
```bash
# yum install -y kmod-drbd84 drbd84-utils
```
Load DRBD modul pada kedua node
```bash
# modprobe drbd
```
Cek apakah modul sudah berhasil di load atau tidak
```bash
# lsmod | grep drbd
drbd                  397041  1
libcrc32c              12644  4 xfs,drbd,nf_nat,nf_conntrack
```
Tambahkan port DRBD di firewall untuk mensinkronisasi data antara dua node
```bash
# firewall-cmd --permanent --add-port=7788/tcp
# firewall-cmd --reload
```
Sekarang kita harus menyiapkan area penyimpanan berukuran sama pada kedua node. Ini bisa berupa partisi hard drive (atau hard drive fisik lengkap), perangkat RAID perangkat lunak, LVM Logical Volume atau tipe perangkat blok lain apa pun yang ditemukan di sistem Anda. Buat disk LVM pada kedua node
```bash
# pvcreate /dev/sda3
    Physical volume "/dev/sdb" successfully created
# vgcreate vg_drbd /dev/sda3
    Volume group "vg_drbd" successfully created
# lvcreate -n lv_drbd -l +100%FREE vg_drbd
    Logical volume "lv_drbd" created

NOTE:untuk /dev/sda3 tidak sama seprti contoh yang diatas tergantung penomoran partisi yang dibuat
```
Setelah menyaipkan area penyimpanan langkah selanjutnya Buat sumber daya(resource) pada kedua node dengan nama file /etc/drbd.d/datadrbd.res
```bash
resource datadrbd {
   Protocol C;
   meta-disk internal;
   device /dev/drbd1;
   syncer {
    verify-alg sha1;
   }
   on node1.localdomain {
    disk /dev/vg_drbd/lv_drbd;
    address 10.7.60.182:7788;
   }
   on node2.localdomain {
    disk /dev/vg_drbd/lv_drbd;
   address 10.7.60.182:7788;
   }
}
```
Penjelasasn:

Hallo, hallo

<b>resource datadrbd  : </b>nama folder resourcenya

<b>meta-disk  internal: </b>menyimpan metadata DRBD di internal disk

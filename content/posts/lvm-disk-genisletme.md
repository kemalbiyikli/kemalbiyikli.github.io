---
title: Linux Sunucuda Disk Genişletme
date: 2025-02-28T23:22:00
draft: false
tags:
  - Linux
  - LVM
  - Storage
slug: lvm-ile-linux-sunucuda-disk-genisletme
---
Linux sunucular, özellikle disk yönetimi söz konusu olduğunda oldukça esnek bir yapı sunar. Bu yapı karmaşık gibi görünse de aslında basit bir mantığın üzerine inşa edilmiştir. Linux sistemlerde yaygın bir şekilde kullanılan **LVM (Logical Volume Manager)** sayesinde, sunucunuzun disk alanını dinamik olarak yönetebilir ve ihtiyaç duyduğunuzda belirli bölümleri kolayca genişletebilirsiniz. Bu yazıda, **RedHat** tabanlı bir sunucuda `/home` dizinini **30GB** artırma işlemini adım adım anlatacağım.

### **1-Disk Durumunun Kontrolü**

Sunucuda diskimizin durumu aşağıdaki gibidir.

```text
[root@hometst ~]# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0  100G  0  disk  
├─sda1        8:1    0    1G  0  part  /boot/efi
├─sda2        8:2    0    1G  0  part  /boot
└─sda3        8:3    0   98G  0  part  
  ├─rhel-root 253:0  0   89G  0  lvm   /
  ├─rhel-swap 253:1  0    8G  0  lvm   [SWAP]
  └─rhel-home 253:2  0   21G  0  lvm   /home
sdb           8:16   0  100G  0  disk  
└─rhel-home  253:2  0   21G  0  lvm   /home
sr0          11:0    1 1024M  0  rom  
```

Yukarıdaki çıktıya göre, **/home** dizini 21GB alana sahip. Genişletme işlemi için disk boyutunun artırılması gerekiyor. Sunucumuzda diskler **vCenter** üzerinden yönetiliyor, bu yüzden genişleteceğimiz diskin hangi fiziksel diske karşılık geldiğini tespit etmemiz gerekiyor.

Bu bilgiyi aşağıdaki komutla edinebiliriz:

```
lsblk -o NAME,HCTL,SIZE,MOUNTPOINT
```

**HCTL** sütunu, fiziksel diskin bağlı olduğu controller ve disk numarasını gösterir. Böylece vCenter üzerinde hangi iSCSI bağlantısına sahip olduğunu görebiliriz. Gerekli diski tespit ettikten sonra vCenter üzerinden genişletme işlemini gerçekleştireceğiz.

### **2: vCenter’da Diski Genişletme**

**vCenter**‘a giriş yapıp diskini artıracağımız **VM**‘i buluyoruz.

Sanal makinenizde **Edit Settings** altında tespit ettiğimiz diski buluyoruz.

**sdb** diskine karşılık gelen diski **en az 30 GB** artırın.

### **3-Physical Volume’u Genişletme**

Disk boyutunu artırdıktan sonra, sunucuyu yeniden başlatmadan diskin büyüdüğünü tanıtmak için şu komutu çalıştırın:

```
echo 1 > /sys/block/sdb/device/rescan
```

Diski genişlettikten sonra, Physical Volume’ü yeniden boyutlandırmak için şu komutu çalıştırın.

```
pvresize /dev/sdb
```

Yeni disk boyutunu kontrol etmek için `lsblk` komutunu kullanın. `sdb` diskinin artık 130GB olduğunu görmelisiniz.

### **4-Logical Volume’u Genişletme**

`sdb` diskini 130GB olarak gördükten sonra, `vgs` komutunu çalıştırarak volume group’ta kullanılabilir boş alanı kontrol edebilirsiniz.

`/home` dizini için ayrılan volume group’u genişletmek amacıyla aşağıdaki komutu kullanın. Bu komut, `**rhel-home**` dizinini 30GB büyütecektir:

```
lvextend -L+30G /dev/mapper/rhel-home
```

Alternatif olarak, kullanılabilir tüm boş alanı kullanmak için şu komutu da tercih edebilirsiniz:

```
lvextend -l +100%FREE /dev/mapper/rhel-home
```

Bu komutu çalıştırdıktan sonra `lsblk` ile kontrol edin. `/home` dizininin 51GB olduğunu görmelisiniz.

```text
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0  100G  0  disk  
├─sda1        8:1    0    1G  0  part  /boot/efi
├─sda2        8:2    0    1G  0  part  /boot
└─sda3        8:3    0   98G  0  part  
  ├─rhel-root 253:0  0   89G  0  lvm   /
  ├─rhel-swap 253:1  0    8G  0  lvm   [SWAP]
  └─rhel-home 253:2  0   51G  0  lvm   /home
sdb           8:16   0  130G  0  disk  
└─rhel-home  253:2  0   51G  0  lvm   /home
sr0          11:0    1 1024M  0  rom  
```

### **5-Dosya Sistemini Genişletme**

Ancak, `df -h` komutu ile kontrol ettiğinizde `/home` dizini hala 21GB olarak görebilirsiniz. `/home` dizininin yeni ayrılan alanı kullanabilmesi için dosya sistemini de genişletmeniz gerekiyor. Öncelikle dosya sisteminin türünü kontrol etmek için şu komutu çalıştırın:

```
df -T /home
```

Bu komutun çıktısında dosya sisteminin **xfs** ya da **ext4** olduğunu göreceksiniz.

**XFS Dosya Sistemi İçin:**

Eğer dosya sisteminiz **XFS** ise, aşağıdaki komutla genişletme işlemini gerçekleştirebilirsiniz:

```
xfs_growfs /home
```

**EXT4 Dosya Sistemi İçin:**

Eğer dosya sisteminiz **ext4** ise, aşağıdaki komutu kullanarak dosya sistemini genişletebilirsiniz:

```
resize2fs /dev/mapper/rhel-home
```

Dosya sistemini genişlettikten sonra `df -h` komutunu tekrar çalıştırın. Artık `/home` dizininin 51GB olduğunu görmelisiniz.

```text
[root@hometst ~]# df -h
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda           8:0    0  100G  0  disk  
├─sda1        8:1    0    1G  0  part  /boot/efi
├─sda2        8:2    0    1G  0  part  /boot
└─sda3        8:3    0   98G  0  part  
  ├─rhel-root 253:0  0   89G  0  lvm   /
  ├─rhel-swap 253:1  0    8G  0  lvm   [SWAP]
  └─rhel-home 253:2  0   51G  0  lvm   /home
sdb           8:16   0  130G  0  disk  
└─rhel-home  253:2  0   51G  0  lvm   /home
sr0          11:0    1 1024M  0  rom  
```

Yukarıdaki adımları takip ederek, Linux sunucularda **LVM** dizinlerini herhangi bir restart veya kesinti gerekmeksizin genişletebilirsiniz.
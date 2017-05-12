# software-defined-storage

Benvenuti al corso su Software defined storage!
--------------------------------
Usatelo pure per prendere appunti, fare domande

## Parte 1

[slides](https://docs.google.com/presentation/d/1QDVo2PxECeWevDDTidjdsRqvUW5qPSTgf3sWZhC7zfo/edit?usp=sharing)

## Parte 2

### dashboard
        -        Lanciare 4 template -> Images - shared with me - count 4
        -        Creare 2 volumi STD 50GB
        -        Rinominare le VM da Horizon come admin,mon1,osd1,osd2
        -        Dare IP pubblico ad admin
        -        Dentro a admin
                        -        hostname admin
                        -        echo "admin" > /etc/hostname
                        -        ssh-keygen

### sistemare hosts - prendere ip reali da dashboard
echo "192.168.0.x mon1" >> /etc/hosts
echo "192.168.0.x osd1" >> /etc/hosts
echo "192.168.0.x osd2" >> /etc/hosts
echo "192.168.0.x admin" >> /etc/hosts

### copiare chiavi
ssh-keygen
ssh-copy-id root@mon1
ssh-copy-id root@osd1
ssh-copy-id root@osd2

### settare hostname
ssh mon1 'hostname mon1'
ssh mon1 'echo "mon1" > /etc/hostname'        
ssh osd1 'hostname osd1'
ssh osd1 'echo "osd1" > /etc/hostname'
ssh osd2 'hostname osd2'
ssh osd2 'echo "osd2" > /etc/hostname'

### copiare hosts
scp /etc/hosts root@mon1:/etc/hosts
scp /etc/hosts root@osd1:/etc/hosts
scp /etc/hosts root@osd2:/etc/hosts

### installare pacchetti
apt update
apt install -y ceph-deploy ntp
ssh mon1 'apt install -y python ntp'
ssh osd1 'apt install -y python ntp'
ssh osd2 'apt install -y python ntp'

### installare ceph
mkdir loged-cluster
cd loged-cluster
ceph-deploy new mon1
echo "osd pool default size = 2" >> ceph.conf
ceph-deploy install admin mon1 osd1 osd2
ceph-deploy mon create-initial
ceph-deploy admin admin mon1 osd1 osd2
ceph-deploy osd prepare osd1:/dev/vdb osd2:/dev/vdb
ceph -w
ceph -s

### espansione cluster HA
_lanciare 3 template - rinominarne 2 mon2,mon3_
echo "192.168.0.x mon2" >> /etc/hosts
echo "192.168.0.x mon3" >> /etc/hosts
ssh-copy-id root@mon2
ssh-copy-id root@mon3
ssh mon2 'hostname mon2'
ssh mon2 'echo "mon2" > /etc/hostname'        
ssh mon2 reboot
ssh mon3 'hostname mon3'
ssh mon3 'echo "mon3" > /etc/hostname'        
ssh mon3 reboot
scp /etc/hosts root@mon1:/etc/hosts
scp /etc/hosts root@mon2:/etc/hosts
scp /etc/hosts root@mon3:/etc/hosts
scp /etc/hosts root@osd1:/etc/hosts
scp /etc/hosts root@osd2:/etc/hosts
ssh mon2 'apt install -y python ntp'
ssh mon3 'apt install -y python ntp'
ceph-deploy install mon2 mon3
ceph-deploy mon add mon2
ceph-deploy mon add mon3
ceph -s
ceph mon dump

### rinominare terzo template osd3
echo "192.168.0.x osd3" >> /etc/hosts
ssh-copy-id root@osd3
ssh osd3 'hostname osd3'
ssh osd3 'echo "osd3" > /etc/hostname'        
ssh osd3 reboot
scp /etc/hosts root@mon1:/etc/hosts
scp /etc/hosts root@mon2:/etc/hosts
scp /etc/hosts root@mon3:/etc/hosts
scp /etc/hosts root@osd1:/etc/hosts
scp /etc/hosts root@osd2:/etc/hosts
scp /etc/hosts root@osd3:/etc/hosts
ssh osd3 'apt install -y python ntp'
ceph-deploy install osd3
ceph-deploy osd prepare osd3:/dev/vdb
ceph osd tree
ceph osd dump
ceph osd pool ls
ceph osd pool get rbd min_size
ceph osd pool get rbd size
ceph -w

### aggangiare volume a VM
rbd -p rbd create loged --size 10G  --image-feature layering
rbd -p rbd info loged
rbd map -p rbd loged
rbd showmapped
fdisk /dev/rbd0
mkfs.xfs /dev/rbd0p1
mount /dev/rbd0p1 /mnt
cd /mnt
touch loged

### testare ridondanza 
ssh osd3 shutdown -h now
ceph -w
ssh mon3 shutdown -h now
ceph -w
ceph -s

## Parte 3
[slides](https://docs.google.com/presentation/d/1ETZ0vVr_ss4V4bcWJkjbjs_bdJrB9BbF0hk_KKLm-jc/edit?usp=sharing)

### dashboard
        -        Lanciare 2 istanze e3standard.x4 -> Images ubuntu 16.04
        -        Creare 2 volumi STD 50GB
        -        Rinominare le VM da Horizon come glusterfs-1 e glusterfs-2

### sistemare hosts - prendere ip reali da dashboard
echo "192.168.0.x glusterfs-1" >> /etc/hosts
echo "192.168.0.x glusterfs-2" >> /etc/hosts

### copiare chiavi
ssh-keygen
ssh-copy-id root@glusterfs-1
ssh-copy-id root@glusterfs-2

### settare hostname
ssh glusterfs-1 'hostname glusterfs-1'
ssh glusterfs-1 'echo "glusterfs-1" > /etc/hostname'
ssh glusterfs-1 'hostname glusterfs-2'
ssh glusterfs-1 'echo "glusterfs-2" > /etc/hostname'

### copiare hosts
scp /etc/hosts root@glusterfs-2:/etc/hosts

### preparare dischi
parted -s /dev/vdb mklabel gpt
parted --align optimal -s /dev/vdb mkpart primary xfs 0% 100%
mkfs.xfs /dev/vdb1
mkdir /mnt/storage-brick
mount /dev/vdb1 /mnt/storage-brick

### install
apt install glusterfs-server -y 

### peer
gluster peer probe glusterfs-1
gluster peer probe glusterfs-2

### create volume
gluster volume create volume0 replica 2 transport tcp glusterfs-1:/mnt/storage-brick/volume0 glusterfs-2:/mnt/storage-brick/volume0

### opzioni mount da fstab
glusterfs-1:/volume0 /mnt/cloudstorage glusterfs acl,backupvolfile-server=glusterfs-2,fetch-attempts=2,log-level=WARNING,log-file=/var/log/gluster.log 0 0

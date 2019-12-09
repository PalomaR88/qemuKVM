**Crea 2 máquinas virtuales, debian Buster, con un disco en fichero, utilizando aprovisionamiento ligero, una conectada a una red NAT (10.0.0.0/24) y la otra a un bridge externo.**

Driver virtio en todo lo que se pueda.

Creación de qcow2:
~~~
qemu-img create -f qcow2 "nombre".qcow2 "tamaño"G
~~~

~~~
paloma@coatlicue:~/DISCO2/CICLO II/CLOUD COMPUTING$ qemu-img create -f qcow2 paloma.qcow2 10G
Formatting 'paloma.qcow2', fmt=qcow2 size=4294967296 cluster_size=65536 lazy_refcounts=off refcount_bits=16
~~~


Y se instala:
~~~
kvm -m 1024 -hda paloma.qcow2 \
-cdrom /home/paloma/Descargas/debian-10.2.0-amd64-netinst.iso 
~~~

lo siguiente que hay que hacer:
qemu-img info paloma.qcow2

qemu-img create -b paloma1.qcow2 -f qcow2 paloma.qcow2




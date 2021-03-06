# Migración de una máquina virtual con libvirt
Realiza el proceso de migración de una máquina virtual que está ejecutando un servicio PostgreSQL desde tu propio hipervisor al situado en la dirección 172.22.200.10, estando en todo momento los datos del servidor PostgreSQL en un volumen independiente.

Cuanto mayor sea el nivel de automatismo de la tarea más alta será la calificación.

Los paquetes necesarios son:
~~~
sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system
~~~

FASE 1:
### Crear una MV en libvirt/KVM de tu equipo conectado a una red interna que tenga salida por NAT usando la imagen de buster del NAS y con aprovisionamiento ligero (MV1).
Se crea un fichero de imagen que contendrá el disco duro:
~~~
paloma@coatlicue:~/qemuKVM$ qemu-img create -f qcow2 mv1.qcow2 10G
Formatting 'mv1.qcow2', fmt=qcow2 size=10737418240 cluster_size=65536 lazy_refcounts=off refcount_bits=16
~~~

Se configura la RAM y la imagen del sistema operativo que se va a usar y se completa la instalación de este:
~~~
paloma@coatlicue:~/qemuKVM$ kvm -m 1024 -hda mv1.qcow2 -cdrom /home/paloma/DISCO2/ISO/debian-10.2.0-amd64-netinst.iso 
qemu-system-x86_64: -hda mv1.qcow2: drive with bus=0, unit=0 (index=0) exists
~~~

Tras la instalación, se actalizan los paquetes para comenzar a crear el aprovisionamiento ligero:
~~~
paloma@coatlicue:~/qemuKVM$ qemu-img create -b mv1.qcow2 -f qcow2 mv1-1.qcow2
Formatting 'mv1-1.qcow2', fmt=qcow2 size=10737418240 backing_file=mv1.qcow2 cluster_size=65536 lazy_refcounts=off refcount_bits=16
~~~

Para la configuración de la red interna se crea un fichero xml con la siguiente configuración:
~~~
<network>
 <name>red_int</name>
    <forward mode="nat"/>
    <ip address="10.0.0.1" netmask="255.255.255.0">
        <dhcp>
          <range start="10.0.0.1" end="10.0.0.30"/>
        </dhcp>
    </ip>
</network>
~~~

Para implementar la nueva red descrita en el fichero:
~~~
paloma@coatlicue:~/qemuKVM$ virsh -c qemu:///system net-define nat.xml
Network red_int defined from nat.xml
paloma@coatlicue:~/qemuKVM$ virsh -c qemu:///system net-autostart red_int
Network red_int marked as autostarted
paloma@coatlicue:~/qemuKVM$ virsh -c qemu:///system net-start red_int
Network red_int started
~~~

Y se comprueba que la red se ha creado correctamente:
~~~
paloma@coatlicue:~/qemuKVM$ virsh -c qemu:///system net-list
 Name      State    Autostart   Persistent
--------------------------------------------
 red_int   active   yes         yes
~~~


### Configurar la RAM disponible a 500 MiB
Para completar la configuración de la máquina e indicar la memoria RAM que va a tener disponible y añadir la red que se ha configurado anteriormente, entre otras cosas:
~~~
virt-install --connect qemu:///system 
             --name buster 
             --memory 500 
             --disk path=/home/paloma/qemuKVM/mv1-1.qcow2 
             --vcpus=1 
             --boot hd 
             --vnc 
             --os-type linux 
             --os-variant=debian10 
             --network network=red_int 
             --noautoconsole 
             --hvm 
             --keymap es
~~~

La instalación se realiza como se indica a continuación:
~~~
paloma@coatlicue:~/qemuKVM$ virt-install --connect qemu:///system --name buster1 --memory 500 --disk path=/home/paloma/qemuKVM/mv1.qcow2 --vcpus=1 --boot hd --vnc --os-type linux --os-variant=debian10 --network network=red_int --noautoconsole --hvm --keymap es

Empezando la instalación...
Creación de dominio completada.
~~~

En la máquina hay que añadir una interfaz de red para la red interna añadiendo las siguientes líneas a **/etc/network/interfaces**:
~~~
allow-hotpolug enp1s0
iface enp1s0 inet dhcp
~~~

### Crear un fichero adicional de 200 MiB y conectarlo a MV1 a través de libvirt
Creación del fichero adicional:
~~~
paloma@coatlicue:~/qemuKVM$ dd if=/dev/zero of=200.img bs=1M count=200
200+0 registros leídos
200+0 registros escritos
209715200 bytes (210 MB, 200 MiB) copied, 0,383341 s, 547 MB/s
~~~

Y se añade a la máquina:
~~~
paloma@coatlicue:~/qemuKVM$ virsh -c qemu:///system attach-disk buster1 --source /home/paloma/qemuKVM/200.img --target vdb --persistent
Disk attached successfully
~~~

En la máquina aparece como **vdb**:
~~~
debian@debian:~$ su
Contraseña: 
root@debian:/home/debian# lsblk -f
NAME FSTYPE LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINT
vda                                                                   
├─vda1
│    ext4         decbcae4-83e1-4aaf-bdab-8beb7723fbd7    7,1G    14% /
├─vda2
│                                                                     
└─vda5
     swap         c86bf5c7-3367-4ec3-a62a-d9100477a8c2                [SWAP]
vdb 
~~~

Se crea una partición:
~~~
root@debian:~# fdisk /dev/vdb

Bienvenido a fdisk (util-linux 2.33.1).
Los cambios solo permanecerán en la memoria, hasta que decida escribirlos.
Tenga cuidado antes de utilizar la orden de escritura.

El dispositivo no contiene una tabla de particiones reconocida.
Se ha creado una nueva etiqueta de disco DOS con el identificador de disco 0x0aee6d89.

Orden (m para obtener ayuda): n
Tipo de partición
   p   primaria (0 primaria(s), 0 extendida(s), 4 libre(s))
   e   extendida (contenedor para particiones lógicas)
Seleccionar (valor predeterminado p): p
Número de partición (1-4, valor predeterminado 1): 
Primer sector (2048-409599, valor predeterminado 2048): 
Último sector, +/-sectores o +/-tamaño{K,M,G,T,P} (2048-409599, valor predeterminado 409599): 

Crea una nueva partición 1 de tipo 'Linux' y de tamaño 199 MiB.

Orden (m para obtener ayuda): w
Se ha modificado la tabla de particiones.
Llamando a ioctl() para volver a leer la tabla de particiones.
Se están sincronizando los discos.
~~~

Y se formatea:
~~~
root@debian:~# mkfs.ext4 /dev/vdb1
mke2fs 1.44.5 (15-Dec-2018)
Creating filesystem with 203776 1k blocks and 51000 inodes
Filesystem UUID: 32659830-ef58-4297-a5ef-8cba27537da7
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done 
~~~

### Instalar PostgreSQL y ubicar el directorio /var/lib/postgresql en el volumen asociado al fichero adicional (muy importante comprobar permisos y propietarios)
Instalación de los paquetes de postgresql:
~~~
root@debian:~# apt install postgresql postgresql-contrib postgresql-client
~~~

Y se monta el directorio en el volumen, para ello se comprueban los permisos y usuarios que tiene /var/lib/postgresql, que son:
~~~
drwxr-xr-x 3 postgres postgres 4096 ene 14 21:02 postgresql
~~~

Y se copia el fichero:
~~~
root@debian:~# cp -r /var/lib/postgresql/* /var/lib/postgresql-copia/
~~~

Se añade la siguiente línea en **/etc/fstab**:
~~~
UUID=32659830-ef58-4297-a5ef-8cba27537da7 /var/lib/postgresql ext4 defaults     0       0
~~~

Se realiza el montaje y comprueba:
~~~
root@debian:~# mount -a
root@debian:~# lsblk -f
NAME   FSTYPE LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINT
vda                                                                     
├─vda1 ext4         decbcae4-83e1-4aaf-bdab-8beb7723fbd7    7,1G    14% /
├─vda2                                                                  
└─vda5 swap         c86bf5c7-3367-4ec3-a62a-d9100477a8c2                [SWAP]
vdb                                                                     
└─vdb1 ext4         32659830-ef58-4297-a5ef-8cba27537da7  173,3M     1% /var/lib/postgresql
~~~

Con el mantaje, se copia el fichero y se ha cambiado el propietario de /var/lib/postgrsql:
~~~
root@debian:~# ls -l /var/lib/
...
drwxr-xr-x 3 root root 1024 ene 14 21:09 postgresql
...
root@debian:~# chown -R postgres:postgres /var/lib/postgresql/
root@debian:~# ls -l /var/lib/
...
drwxr-xr-x 3 postgres postgres 1024 ene 14 21:09 11
...
~~~


### Poblar la base de datos
Se va a crear un usuario y una base de datos:
~~~
postgres=# create user debian with password 'debian';
CREATE ROLE
postgres=# create database restaurante;
CREATE DATABASE
postgres=# grant all privileges on database restaurante to debian;
GRANT

~~~

Y se añaden algunas tablas:
~~~
postgres@debian:/$ psql -U debian -d restaurante -h localhost
Contraseña para usuario debian: 
psql (11.5 (Debian 11.5-1+deb10u1))
conexión SSL (protocolo: TLSv1.3, cifrado: TLS_AES_256_GCM_SHA384, bits: 256, compresión: desactivado)
Digite «help» para obtener ayuda.

restaurante=> \d
                 Listado de relaciones
 Esquema |           Nombre           | Tipo  | Dueño  
---------+----------------------------+-------+--------
 public  | aspectos                   | tabla | paloma
 public  | catadores                  | tabla | paloma
 public  | colaboraciones             | tabla | paloma
 public  | composicion_ing_preparados | tabla | paloma
 public  | experimentos               | tabla | paloma
 public  | ingredientes               | tabla | paloma
 public  | ingredientes_por_version   | tabla | paloma
 public  | investigadores             | tabla | paloma
 public  | puntuaciones               | tabla | paloma
 public  | versiones                  | tabla | paloma
(10 filas)

restaurante=> select * from aspectos;
 codigo | descripcion  | importancia 
--------+--------------+-------------
 COL    | Color        | Baja
 TEX    | Textura      | Alta
 VOL    | Volumen      | Media
 CAN    | Cantidad     | Alta
 PRE    | Presentacion | Alta
 TEC    | Tecnica      | Media
 ORI    | Originalidad | media
(7 filas)

~~~

### Crear una regla de iptables que redirija las peticiones al puerto 5432/tcp que se realicen desde fuera a MV1 para que la base de datos sea accesible desde el exterior.
Configuración de postgresql para que sea accesible:
- /etc/postgresql/11/main/pg_hba.conf
~~~
host    all             all             0.0.0.0/0            md5
~~~

- /etc/postgresql/11/main/postgresql.conf
~~~ 
listen_addresses = '*'          
~~~

Se reinicia los servicios:
~~~
root@debian:~# systemctl status postgresql@11-main.service
root@debian:~# systemctl status postgresql.service
~~~

Se añaden las siguientes reglas de iptable:
~~~
sudo iptables -t nat -I PREROUTING -p tcp --dport 5432 -i enp1s0 -j DNAT --to 10.10.10.1
sudo iptables -t nat -I POSTROUTING -p tcp --sport 5432 -s 10.10.10.0/24 -j MASQUERADE
~~~



### Crear un registro en el DNS para el servicio

FASE 2:

### Monitorizar el uso de RAM de MV1 de manera que comience la migración en el momento que el uso de RAM supere el 90%

### Crear una MV en libvirt/KVM remoto conectado a la red interna 10.0.1.0/24 que tiene salida por NAT usando la imagen de buster del NAS y con aprovisionamiento ligero (MV2).

### Configurar la RAM disponible para MV2 a 1 GiB

### Instalar PostgreSQL en MV2
- Crear una regla de iptables que redirija las peticiones al puerto 5432/tcp que se realicen desde fuera a MV2 para que la base de datos sea accesible desde el exterior.
- Parar el servicio PostgreSQL en MV1
- Desconectar el volumen adicional de MV1, redimensionarlo a 400 MiB (también el sistema de ficheros), copiarlo y conectarlo a MV2.
- Montar el volumen adicional en /var/lib/postgresql y reiniciar el servicio PostgreSQL en KVM verificando permisos y propietarios.
- Actualizar el registro DNS para que el servicio lo preste ahora MV2
- Comprobar el funcionamiento

FASE 3:

- Monitorizar MV2 y cuando el uso de RAM llegue al 90%, subir la RAM asignada a 2 GiB en vivo.

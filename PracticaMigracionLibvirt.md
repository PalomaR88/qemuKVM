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
          <range start="10.0.0.1" end="10.0.0.20"/>
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
paloma@coatlicue:~/qemuKVM$ virt-install --connect qemu:///system --name buster --memory 500 --disk path=/home/paloma/qemuKVM/mv1-1.qcow2 --vcpus=1 --boot hd --vnc --os-type linux --os-variant=debian10 --network network=red_int --noautoconsole --hvm --keymap es

Empezando la instalación...
Creación de dominio completada.
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
paloma@coatlicue:~/qemuKVM$ virsh -c qemu:///system attach-disk buster --source /home/paloma/qemuKVM/200.img --target vdb --persistent
Disk attached successfully
~~~

En la máquina anfitriona hay que añadir una interfaz de red para la red interna añadiendo las siguientes líneas a **/etc/network/interfaces**:
~~~
allow-hotpolug enp1s0
iface enp1s0 inet dhcp
~~~


### Instalar PostgreSQL y ubicar el directorio /var/lib/postgresql en el volumen asociado al fichero adicional (muy importante comprobar permisos y propietarios)

### Poblar la base de datos

### Crear una regla de iptables que redirija las peticiones al puerto 5432/tcp que se realicen desde fuera a MV1 para que la base de datos sea accesible desde el exterior.

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

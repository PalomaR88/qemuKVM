# DISPOSITIVOS VIRTUALES
- CPU (vCPU)
- RAM - puede ser dinámica
- I/O 
	1. Almacenamiento
		+ Bus
		+ Unidad de almacenamiento
	2. Red. 
		+ Dispositivo de red. 
	3. Video
		+ Suele ser una Cirrus

# DISPOSITIVOS REALES
Disp PCI - PCI passthrough
Esto es que un hardware que está conectado a la anfitriona pasa automáticamente a la virtual, desapareciendo de la anfitriona. 


# ALMACENAMIENTO
El almacenamiento pasa por el kernel de la anfitriona salvo que utilice paravistualización.

~~~
kvm -m 512 -hda hd2.qcow2 -cdrom debian.iso
~~~
Con el comando anterior, se configura una máquina virtual y todos los parámetros que no se indican se establecen por defecto. 

Estos parámetros, no se guardan en un fichero de configuración, sino que se ejecutan desde la línea de comando. 

Libvirt: es una API, programable, para utilizar una sintaxis común para diferentes sistemas de virtualización (Xen, KVM, etc).
Esta es la aplicación que crea el fichero de configuración de las másquinas, y se guardan en .xml. 

Pero además, no nos comunicamos directamente con libvirt, que podríamos y configurar manualmente el fichero de configuración, pero por lo general, usamos otra cosa. Hay muchas, algunos ejemplos:
- virsh: shell
- virt-manager: interfaz gráfica. Pero es algo simple. 
- oVirt: la aplicacion web. Pero consume muchos recursos.
- Openstack: es en realidad un frontal de libvirt.

Almacenamiento:
- raw
- qcow2 - aprovisionamiento ligero
- LVM - esto no es un sistema de ficheros como el anterior, sino que es un bloque, y los bloques son usados por el root. Luego, no se puede añadir directamente. Primero se tiene que añadir al usuario al grupo disk, porque los bloques están en el grupo disk. Y reiniciar, mejor con newgrp disk, que cambia el grupo principal del usuario en esta sesión. 


# REDES VIRTUALES DE MÁQUINAS VIRTUALES
Redes virtuales, redes locales dentro de un nodo.
Virtualización de la red, conectividad entre nodos que se encuentran en diferentes nodos. Esto no lo vamos a hacer. 

Tipos de redes que podemos crear:
- Red aislada: red interna donde se conectan mis máquinas virtuales. 
- Puente o bridge con interfaz externa.
- Router: configurar una tabla como router:
* Activar el bit de forward.
* Y las reglas de encaminamiento.


# Opciones
/etc/libvirt/qemu/maquina.xml ---> aquí se guardan los fichero sxml de las máquinas que se han creado. 

Se puede copiar un fichero xml de una máquina ya creada para hacer otra máquina igual. Hay que modificar ciertos parámetros:
- name
- uuid --> cat /proc/sys/kernel/random/uuid esto es para que el sistema cree un uid aleatorio. 
- source --> cambiar el nombre
- mac
- model type --> esto por defecto pone un tipo que no vale porque no se aprobechan bien los recursos de debian. 

Ese fichero, que hemos copiado y modificado para crear una nueva, se carga en virtmanager para terminar de crear la máquina. 

virsh -c qemu://system define debian10.xml --> para levantar la máquina con la red.

virsh -> buscar esto con detenimiento






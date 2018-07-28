# Alta disponibilidad (H.A) de Issabel (voip-PBX basada en Asterisk) sobre CentOS7 con Corosync Pacemaker y PCS


## Ambiente:

Virtualizador -VirtualBox

Maquinas virtuales: 2

CPU: 2Ghz 

RAM: 1Gb

Network: 1 interfaz de red para cada server (Eth0)


* Primero se debe ganarantizar que los hostname de los servidores son diferentes, para ello el nodo 1 lo denominamos issabel-1 y al nodo 2 lo denominaremos issabel-2 donde cada uno tendrá su ip estática 10.10.1.20 (issabel-1) y 10.10.1.30(issabel-2)

Editamos el archivo /etc/hosts de ambos servidores agregando:


10.10.1.20  issabel-1
10.10.1.30  issabel-2

Para asi garantizar que ambos servidores puedan resovler sus FQDN's entre ellos mismos sin la necesidad de un servidor DNS

ahora si procedemos con la instalación paso a paso

# INSTALLATION

 
```bash
[root@issabel-1]# yum -y update
[root@issabel-2]# yum -y update
```
* Instalamos corosync, pcs y pacemaker en ambos nodos

```bash
[root@issabel-1]# yum -y install corosync pcs pacemaker
[root@issabel-2]# yum -y install corosync pcs pacemaker
```

# START PCSD

```bash
[root@issabel-1]# systemctl start pcsd
[root@issabel-2]# systemctl start pcsd
```

# SET PASSWORD

```bash
[root@issabel-1]# passwd hacluster
[root@issabel-2]# passwd hacluster
```
```bash
[root@issabel-1]# pcs cluster auth issabel-1 issabel-2
```
# En esta parte pedira usuario y contraseña para lo cual agregamos
# user: hacluster
# pass:abcd1234
# (no mporta si indica que la contraseña es incorrecta porque no cumple los parámetros de seguridad, es un cluster de pruebas)

# START CLUSTER

 
```bash
[root@issabel-1]# pcs cluster setup --name cluster-asterisk issabel-1 issabel-2
[root@issabel-1]# systemctl start pacemaker; systemctl start corosync
[root@issabel-2]# systemctl start pacemaker; systemctl start corosync
[root@issabel-1]# pcs cluster start --all
```
 

# Disable STONITH

```bash
[root@issabel-1]# pcs property set stonith-enabled=false
```
# (si esto no se deshabilita, la ip virtual jamás levantara por patrones de seguridad)

```bash
[root@issabel-1]# pcs property set no-quorum-policy=ignore
``` 

# CONFIGURE CLUSTER

```bash
[root@issabel-1]# pcs resource create virtual_ip ocf:heartbeat:IPaddr2 \
ip=10.10.1.40 cidr_netmask=32 nic=eth0 op monitor interval=30s on-fail=restart
```
 
```bash
[root@issabel-1]# yum -y install uuid-c++ uuid-c++-devel libuuid-devel \
jansson-devel gmime gmime-devel gsm gsm-devel ilbc ilbc-devel speex \
speex-devel libogg libogg-devel libvorbis libsrtp libsrtp-devel \
libvorbis-devel
```

## (se reinstalan las librerías necesarias de asterisk solo por seguridad y estar sobre seguros)

```bash
[root@issabel-1]# cd /usr/lib/ocf/resource.d/heartbeat; \
wget https://raw.githubusercontent.com/ClusterLabs/resource-agents\
/master/heartbeat/asterisk; \

chmod 755 asterisk
```
 
```bash
[root@issabel-2]# yum -y install uuid-c++ uuid-c++-devel libuuid-devel \
jansson-devel gmime gmime-devel gsm gsm-devel ilbc ilbc-devel speex \
speex-devel libogg libogg-devel libvorbis libsrtp libsrtp-devel \
libvorbis-devel
```
 
## (se reinstala de la misma manera en el nodo 2)

```bash
[root@issabel-2]# cd /usr/lib/ocf/resource.d/heartbeat; \
wget https://raw.githubusercontent.com/ClusterLabs/resource-agents/\
master/heartbeat/asterisk; \

chmod 755 asterisk
```
 
## Creamos a Asterisk como recurso propio del cluster

```bash
[root@issabel-1]# pcs resource create asterisk ocf:heartbeat:asterisk \
user="root" group="asterisk" op monitor timeout="30"
```
```bash 
[root@issabel-1]# pcs constraint colocation add asterisk with virtual_ip \
score=INFINITY
```
 
```bash
[root@issabel-1]# pcs constraint order virtual_ip then asterisk
```
 

# GRACIAS A ESTE PARAMETRO PODREMOS MOVER LOS RECURSOS ENTRE LOS NODOS DE MANERA AUTOMÁTICA

```bash
[root@issabel-1]# pcs resource defaults resource-stickiness="0"
```
 

# !!!! POR NADA DEL MUNDO SE TE PUEDE OLVIDAR DESHABILITAR LA AUTORECARGA DE ASTERISK CON SYSTEMD EN AMBOS NODOS,

# DE LO CONTRARIO, ASTERISK NO PODRÁ SER MANEJADO POR PACEMAKER Y PCS

 

# SETEAMOS EL AUTOSTART
```bash
[root@issabel-1]# systemctl enable pcsd; systemctl enable corosync; \
systemctl enable pacemaker
```
```bash
[root@issabel-2]# systemctl enable pcsd; systemctl enable corosync; \
systemctl enable pacemaker
```


# Y LISTO, YA PODEMOS ESTAR TRANQUILOS CON NUESTRA ALTA DISPONIBILIDAD Y CONTINUIDAD DE SERVICIO

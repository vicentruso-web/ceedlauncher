# Proyecto FP: lab GNS3, FortiGate, Proxmox y cabina

Nov 1, 2026 · @Virt

## Resumen

El lab corre en un PC con Windows 11 Pro y Hyper-V. Dentro de la GNS3 VM 2.2.61 están el **FortiGate FGT-1**, el router **Street-Data** (FRR) y los switches Dell OS10. Fuera de GNS3, como VM de Hyper-V, están **Promox-02** y la cabina **Cabina-01**, que llegan al FortiGate (port3) por la red Proyecto, 172.29.0.0/24. El FortiGate hace NAT hacia la red del Default Switch para salir a internet.

| Equipo | Interfaz | IP | Función |
| --- | --- | --- | --- |
| Street-Data (FRR 8.2.2) | eth0 | 172.29.1.1/24 | Router; ruta por defecto hacia 172.29.1.2 |
| FortiGate FGT-1 (7.6.7) | port1 | 172.29.1.2/24 (manual) | Lado interno, hacia Street-Data |
| FortiGate FGT-1 | port2 | 172.29.164.150/20 (DHCP) | Salida a internet; puerta de enlace 172.29.160.1 |
| FortiGate FGT-1 | port3 | 172.29.0.1/24 (manual) | Red Proyecto: Proxmox y sus VM |
| GNS3 VM | eth0 | 172.29.2.10/24 (fija, switch GNS3-NAT) | Servidor GNS3, Web UI en el puerto 80 |
| GNS3 VM | eth1 | Sin IP (switch Proyecto) | La usa el nodo Cloud de GNS3 hacia port3 |
| Windows | vEthernet (Proyecto) | 172.29.0.10/24 | Acceso del PC a la red Proyecto |
| Promox-02 | vmbr0 (eth0) | 172.29.0.22/24 | Proxmox VE, web en https://172.29.0.22:8006 |
| Cabina-01 | eth0 | 172.29.0.23/24 (prevista) | Cabina iSCSI con Cockpit (en instalación) |

&#91;embedded content: topología del lab · router, firewall y salida\]

La red del Default Switch (172.29.160.0/20) la asigna Hyper-V y puede cambiar al reiniciar el PC.

### Plan de direccionamiento

| Red | Uso |
| --- | --- |
| 172.29.0.0/24 | Red Proyecto (switch interno de Hyper-V): FortiGate port3 .1, Windows .10, Promox-02 .22, Cabina-01 .23 |
| 172.29.1.0/24 | Tránsito entre el FortiGate y los routers: FortiGate .2, Street-Data .1, siguientes routers .3, .4… |
| 172.29.2.0/24 | Gestión de la GNS3 VM (PC .1, GNS3 VM .10). No usar dentro del lab |
| 172.29.8.0/21 | VLAN 10 – Semáforos (puerta de enlace 172.29.8.1) |
| 172.29.16.0/21 | VLAN 20 – Cámaras (puerta de enlace 172.29.16.1) |
| 172.29.24.0/21 | VLAN 30 – Ambiente (puerta de enlace 172.29.24.1) |
| 172.29.254.0/24 | Enlaces punto a punto /30 entre switches |

### Estado actual del lab (18-11-2026)

Cómo está montado hoy. FW-01 está en el centro con una tarjeta en cada switch de Hyper-V y es el único camino entre las redes; sale a internet por el Default Switch. Lo discontinuo está pendiente.

&#91;embedded content: FW-01 · puertos, switches de Hyper-V y equipos de cada red\]

Street-Data vive dentro de la GNS3 VM: la red STREET-DATA de Hyper-V entra por la eth1 de la GNS3 VM, un nodo Cloud la lleva al puerto Te1/1/6 de Street-data-01, y este se une a los otros dos switches por port-channel enrutados.

&#91;embedded content: Street-Data en GNS3 · tres switches con OSPF\]

Máquinas de Hyper-V: cada máquina con sus tarjetas de red y el cable de cada una hasta el puerto de FW-01 de su red. Las NIC 2 y NIC 3 de Promox-02 llevarán los bridges vmbr2 (DMZ) y vmbr1 (INTERNAL), sin IP; la eth1 de la GNS3 VM la usa el Cloud para conectar los routers de GNS3.

&#91;embedded content: Máquinas de Hyper-V · tarjetas de red y conexión a FW-01\]

### Diseño objetivo de redes (pendiente de aplicar)

Criterio: FW-01 es el firewall central y todo el tráfico entre redes pasa por él. **GESTION** es solo para administrar; **INTERNAL** es para las VM de servicio de Proxmox; la DMZ va aparte; Street-Data (OT/IoT) son tres switches LiteSwitch con OSPF, conectados a FW-01 por un enlace de tránsito. Cada switch tiene su propio bloque 172.30.x.0/24 dividido en /26 por VLAN, y los enlaces entre routers usan 192.168.100.0 en adelante. Desde GESTION se administra todo; hacia GESTION no se permite nada. El iSCSI va por gestión (al disco solo se conecta Proxmox) y no pasa por el firewall. El FortiGate ya no está en el lab.

```
                    EXTERNAL (Default Switch, DHCP)
                                  |
                           FW-01 (OpenWrt)
       +----------------+---------+--------+-----------------------+
    GESTION          INTERNAL             DMZ                 STREET-DATA
 172.29.0.0/23    172.29.4.0/23     172.29.2.0/23          192.168.100.0/24
 Windows .0.10    Promox vmbr1      Promox vmbr2                  |
 Promox-02 .0.22  VM de servicio    VM de la DMZ           Street-data-01 (.2)
 Cabina-01 .0.23                                         Po1 /          \ Po2
 GNS3 VM .0.40                                  Street-data-02 -- Street-data-03
                                                              Po2
  Street-data-01/02/03: 172.30.1.0/24, 172.30.2.0/24, 172.30.3.0/24 (VLAN 10/20/30 en /26)
  Gestión de los switches: 172.29.7.1 / .2 / .3
```

**Plan de direccionamiento objetivo:**

| Red | Switch Hyper-V / dónde | Uso |
| --- | --- | --- |
| 172.29.0.0/23 | GESTION | Solo administración: FW-01 .0.1, Windows .0.10, Promox-02 .0.22 (web e iSCSI), Cabina-01 .0.23, GNS3 VM .0.40 |
| 172.29.2.0/23 | DMZ | Servicios publicados: FW-01 .2.1, VM desde .2.10 |
| 172.29.4.0/23 | INTERNAL | VM de servicio de Proxmox: FW-01 .4.1, VM desde .4.10 |
| 172.29.6.0/24 | — | Libre |
| 172.29.7.0/24 | dentro de los switches (Vlan99) | Gestión de los switches: Street-data-01 .7.1, 02 .7.2, 03 .7.3 (cada una /32) |
| 172.29.8.0 en adelante | — | Libre |
| 172.30.1.0/24 | Street-data-01 | VLAN 10 semáforos .1.0/26 (gw .1), VLAN 20 cámaras .1.64/26 (gw .65), VLAN 30 sensores .1.128/26 (gw .129), .1.192/26 libre |
| 172.30.2.0/24 | Street-data-02 | Mismo reparto: .2.0/26, .2.64/26, .2.128/26, .2.192/26 libre |
| 172.30.3.0/24 | Street-data-03 | Mismo reparto: .3.0/26, .3.64/26, .3.128/26, .3.192/26 libre |
| 192.168.100.0/24 | STREET-DATA (Hyper-V) | Tránsito FW-01 .1 ↔ Street-data-01 Te1/1/6 .2 |
| 192.168.101.0/24 | Po1 | Street-data-01 .1 ↔ Street-data-02 .2 |
| 192.168.102.0/24 | Po2 | Street-data-01 .1 ↔ Street-data-03 .2 |
| 192.168.103.0/24 | Po2/Po2 | Street-data-02 .1 ↔ Street-data-03 .2 |

El switch GNS3-NAT y el NAT de Windows desaparecen: la GNS3 VM pasa a la red de gestión y sale a internet por FW-01.

**Tarjetas de cada equipo:**

| Equipo | Tarjeta | Red | IP |
| --- | --- | --- | --- |
| FW-01 | eth0 (br-lan), interfaz `gestion` | GESTION | 172.29.0.1/23 |
| FW-01 | eth1, interfaz `external` | Default Switch | DHCP (internet) |
| FW-01 | eth2, interfaz `dmz` | DMZ | 172.29.2.1/23 |
| FW-01 | eth3, interfaz `streetdata` | STREET-DATA | 192.168.100.1/24 |
| FW-01 | eth4 (MAC 00155D290003), interfaz `internal` | INTERNAL | 172.29.4.1/23 |
| Promox-02 | vmbr0 | GESTION | 172.29.0.22/23, puerta de enlace 172.29.0.1 (web e iSCSI) |
| Promox-02 | vmbr1 (tarjeta …640D) | INTERNAL | sin IP (puente de las VM de servicio) |
| Promox-02 | vmbr2 (tarjeta …640E) | DMZ | sin IP (puente de las VM de la DMZ) |
| Cabina-01 | eth0 | GESTION | 172.29.0.23/23, puerta de enlace 172.29.0.1 (Cockpit e iSCSI) |
| GNS3 VM | eth0 (…6404) | GESTION | 172.29.0.40/23, puerta de enlace 172.29.0.1 (web de GNS3) |
| GNS3 VM | eth1 (…640A) | STREET-DATA | sin IP (nodo Cloud STREET-DATA) |
| GNS3 VM | eth2 (…640C) | GESTION | sin IP, sin uso de momento |
| Street-data-01 | Te1/1/6 (enrutado) | STREET-DATA | 192.168.100.2/24, ruta por defecto 192.168.100.1 |
| Windows | vEthernet | GESTION | 172.29.0.10/23 (sin puerta de enlace en esta red) |

**Gestión de los switches:** los LiteSwitch no tienen puerto de gestión separado ni interfaces Loopback, así que cada uno lleva su IP de gestión en una `interface Vlan99` sin puertos, con máscara /32 (172.29.7.1, .2 y .3), anunciada por OSPF. Desde GESTION se llega a través de FW-01. Limitación de los LiteSwitch: las interfaces Vlan solo están activas si el switch tiene al menos un puerto de acceso con equipo conectado, por eso cada switch lleva un VPCS.

**Enrutamiento y NAT:** FW-01 tiene dos rutas estáticas por Street-data-01 (192.168.100.2): 172.30.0.0/16 y 172.29.7.0/24, y es el único que hace NAT a internet. Street-data-01 tiene la ruta por defecto hacia FW-01 y la reparte por OSPF (`default-information originate`) a los otros dos switches; entre los switches todo es OSPF área 0 por los port-channel enrutados. FW-01 no habla OSPF. Los equipos de gestión con puerta de enlace 172.29.0.1 (Proxmox, cabina, GNS3 VM) llegan a Street-Data sin rutas añadidas; Windows es el anfitrión y sale a internet por su propia tarjeta, así que las pruebas se hacen desde la GNS3 VM o Proxmox.

**Políticas (lo no permitido se deniega):**

| Origen → destino | Permitido |
| --- | --- |
| GESTION → internet | Navegación, DNS, NTP (NAT) |
| GESTION → INTERNAL / DMZ / Street-Data | Administración: SSH, HTTPS, SNMP, Zabbix |
| Cualquier red → GESTION | Denegado |
| INTERNAL → internet | Web, DNS, NTP (NAT) |
| DMZ → internet | Web, DNS (NAT) |
| DMZ → INTERNAL | Denegado |
| Street-Data → INTERNAL | Solo los servicios necesarios (MQTT, Zabbix) |
| Street-Data → internet | Solo NTP |
| Street-Data → FW-01 | Denegado (zona streetdata con input REJECT) |

**Cambios para llegar a este diseño (de uno en uno):**

1. Hyper-V: renombrar el switch a GESTION. **Hecho.**
2. FW-01: gestión a /23, eth2 172.29.2.1/23, eth3 192.168.100.1/24 y zona streetdata. **Hecho.**
3. Windows: máscara /23 en GESTION. **Hecho.**
4. GNS3 VM: 172.29.0.40/23 en GESTION, eth1 a STREET-DATA, eth2 nueva; switch GNS3-NAT y NAT de Windows quitados. **Hecho.**
5. Hyper-V: switch INTERNAL, tarjeta de FW-01 en INTERNAL y dos tarjetas nuevas en Promox-02 (INTERNAL y DMZ), con *MAC spoofing*. **Hecho.**
6. FW-01: interfaz `gestion` (sin DHCP) y nueva `internal` en eth4. **Hecho.**
7. FW-01: reglas predeterminadas sustituidas por Allow-DHCP-Renew y forwardings desde gestión. **Hecho.**
8. FW-01: rutas 172.30.0.0/16 y 172.29.7.0/24 vía 192.168.100.2 (la ruta hacia el antiguo FortiGate se quita). **Hecho.**
9. GNS3: Cloud STREET-DATA (eth1) ↔ Street-data-01 Te1/1/6 y nodos renombrados. **Hecho.**
10. Street-data-01: configuración con OSPF y prueba con PC1. **Hecho.**
11. Street-data-02 y Street-data-03, con un VPCS en cada uno.
12. Promox-02: máscara /23, vmbr1 (INTERNAL) y vmbr2 (DMZ).
13. Cabina-01: máscara /23.
14. FW-01: políticas (fwpolicy).

## GNS3

La GNS3 VM empezó en VirtualBox, pero allí el FortiGate no podía arrancar por falta de virtualización anidada. Se pasó a Hyper-V, que sí la permite.

### 1. Activar Hyper-V

En PowerShell como administrador:

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All -All -NoRestart
Restart-Computer
```

Comprobación tras el reinicio:

```powershell
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All | Select-Object FeatureName, State
Get-VMSwitch
```

Se usa el switch **Default Switch** que crea Hyper-V.

### 2. Crear la GNS3 VM en Hyper-V

Los dos discos `.vhdx` de la GNS3 VM 2.2.61 (descargados de GitHub en la misma versión que el cliente GNS3) están en `C:\HyperV\GNS3VM`.

```powershell
$dir   = "C:\HyperV\GNS3VM"
$disks = Get-ChildItem $dir -Recurse -Filter *.vhdx | Sort-Object Name
New-VM -Name "GNS3 VM" -Generation 1 -MemoryStartupBytes 8GB -VHDPath $disks[0].FullName -SwitchName "Default Switch" -Path $dir
Add-VMHardDiskDrive -VMName "GNS3 VM" -ControllerType IDE -ControllerNumber 0 -ControllerLocation 1 -Path $disks[1].FullName
Set-VM -Name "GNS3 VM" -ProcessorCount 4 -StaticMemory -CheckpointType Disabled
Set-VMProcessor -VMName "GNS3 VM" -ExposeVirtualizationExtensions $true
Start-VM "GNS3 VM"
```

- `ExposeVirtualizationExtensions` es lo que permite a la VM ejecutar el FortiGate con KVM.
- Con virtualización anidada Hyper-V no admite memoria dinámica: la RAM se reserva entera al arrancar. Con 8 GB en el PC hay que ajustar `-MemoryStartupBytes` a la memoria libre.
- Al principio la GNS3 VM cogía IP del Default Switch (172.29.164.225), que cambiaba al reiniciar. Ahora tiene IP fija **172.29.2.10** (apartado 5). La Web UI y la API están en `http://172.29.2.10` (puerto 80).

### 3. Plantilla del FortiGate

Plantilla QEMU **FortiGate-7.6.8** con la imagen KVM `fortios.qcow2`:

| Apartado | Valor |
| --- | --- |
| Default name format | `FGT-{0}` |
| Category | Security devices |
| Adapters | 10 |
| Name format (red) | `port{port1}` |
| Tipo de adaptador | Paravirtualized Network I/O (virtio-net-pci) |
| HDA | `fortios.qcow2`, interfaz `virtio` |

### 4. Proyecto

- Proyecto **Proyecto FP** (ID `f1a214bd-1578-430f-9b5a-276287390824`), con los nodos **FGT-1** y **Street-Data**.
- Para que no se cierre solo al desconectar el navegador, se desactivó `auto_close` y se activó `auto_open` por la API v2:

```powershell
$s="http://172.29.2.10/v2"
irm "$s/projects" | ft name,project_id,status
irm "$s/projects/f1a214bd-1578-430f-9b5a-276287390824" -Method Put -ContentType "application/json" -Body '{"auto_close":false,"auto_open":true}' | fl auto_close,auto_open
```

### 5. IP fija de la GNS3 VM (switch GNS3-NAT)

Con el Default Switch la IP de la GNS3 VM cambiaba en cada reinicio del PC. Se creó un switch interno de Hyper-V con NAT de Windows y se fijó la IP dentro de la VM.

| Elemento | Valor |
| --- | --- |
| Switch Hyper-V | GNS3-NAT (interno) |
| Red | 172.29.2.0/24 |
| PC, vEthernet (GNS3-NAT) | 172.29.2.1 (puerta de enlace y NAT) |
| GNS3 VM, eth0 | 172.29.2.10 |
| Web UI | http://172.29.2.10 |
| RAM de la GNS3 VM | 7 GB |
| CPU de la GNS3 VM | 8 vCPU (portátil i7-13620H, 16 procesadores lógicos) |

En el PC, PowerShell como administrador:

```powershell
New-VMSwitch -Name "GNS3-NAT" -SwitchType Internal
New-NetIPAddress -IPAddress 172.29.2.1 -PrefixLength 24 -InterfaceAlias "vEthernet (GNS3-NAT)"
New-NetNat -Name "GNS3-NAT" -InternalIPInterfaceAddressPrefix 172.29.2.0/24
Stop-VM "GNS3 VM"
Connect-VMNetworkAdapter -VMName "GNS3 VM" -SwitchName "GNS3-NAT"
Start-VM "GNS3 VM"
```

En la GNS3 VM: menú azul → **Network**, que abre `/etc/netplan/90_gns3vm_static_netcfg.yaml`. Contenido final (sin `#` delante de las líneas de configuración):

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 172.29.2.10/24
      routes:
        - to: default
          via: 172.29.2.1
      nameservers:
          addresses: [8.8.8.8, 8.8.4.4]
```

Se aplica desde **Shell** y se comprueba:

```bash
sudo netplan apply
ip -br a
```

En `eth0` debe salir `172.29.2.10/24`. Desde el PC: `ping 172.29.2.10`.

## Firewall FortiGate (FGT-1)

El FortiGate recibe el tráfico de Street-Data por port1 y lo saca a internet por port2 con NAT. La consola se abre desde la Web UI de GNS3 (botón derecho sobre FGT-1 → **Web console in new tab**), usuario `admin`.

**Licencia de evaluación** (FortiOS 7.6.7, modelo `EVAL`, número de serie `FGVMEV…`): máximo 1 CPU y 2 GB de RAM, y como mucho **3 interfaces, 3 políticas y 3 rutas** para todo el equipo ([documentación de Fortinet](https://docs.fortinet.com/document/fortigate/7.4.2/administration-guide/441460)). Se comprueba con `get system status` y `diagnose debug vm-print-license`.

### 1. Interfaces

```
config system interface
    edit port1
        set mode static
        set ip 172.29.1.2 255.255.255.0
        set allowaccess ping https ssh http
    next
    edit port2
        set mode dhcp
    next
    edit port3
        set mode static
        set ip 172.29.0.1 255.255.255.0
    next
end
```

- **port1**: 172.29.1.2/24, hacia Street-Data.
- **port2**: por DHCP del Default Switch, obtuvo 172.29.164.150/20. Opcional: fijarla con `set mode static` y `set ip 172.29.164.150 255.255.240.0` para que no cambie.
- **port3**: 172.29.0.1/24, puerta de enlace de la red Proyecto (Proxmox y la cabina). Se comprueba con `show system interface port3`.
- Con port3 se llega al límite de 3 interfaces de la licencia de evaluación.

### 2. Ruta de salida

La ruta por defecto llega por el DHCP de port2: `0.0.0.0/0 via 172.29.160.1, port2`. Si no apareciera, se añade a mano:

```
config router static
    edit 0
        set gateway 172.29.160.1
        set device port2
    next
end
```

No hace falta ruta hacia Street-Data: la red 172.29.1.0/24 está conectada en port1. La ruta estática `172.29.1.1/32 via 172.29.1.1` que se creó en la web sobra y conviene borrarla.

### 3. Objeto de dirección del router

```
config firewall address
    edit "RT-Street-Data"
        set subnet 172.29.1.1 255.255.255.255
    next
end
```

### 4. Política de salida con NAT

```
config firewall policy
    edit 1
        set name "Street-data a External"
        set srcintf "port1"
        set dstintf "port2"
        set action accept
        set srcaddr "RT-Street-Data"
        set dstaddr "all"
        set schedule "always"
        set service "PING" "HTTP" "HTTPS"
        set logtraffic all
        set nat enable
    next
end
```

Con NAT activado el tráfico sale con la IP de port2, así que la respuesta vuelve al FortiGate sin necesidad de rutas extra en el Default Switch.

### 5. Comprobaciones

```
execute ping 172.29.1.1
execute ping 8.8.8.8
get router info routing-table details 8.8.8.8
show firewall policy
show firewall address RT-Street-Data
```

Para ver qué hace el firewall con un paquete concreto:

```
diagnose debug flow filter addr 8.8.8.8
diagnose debug flow trace start 10
diagnose debug enable
```

Se lanza el ping desde Street-Data y se lee la salida; se para con `diagnose debug disable`.

## Router Street-Data (FRR)

Street-Data es un router FRR 8.2.2 con eth0 conectado al port1 del FortiGate y una ruta por defecto hacia él. Se configura desde su consola en GNS3 (`vtysh`, prompt `frr#` o `Street-Data#`).

### 1. Configuración

```
configure terminal
hostname Street-Data
interface eth0
ip address 172.29.1.1/24
no shutdown
exit
ip route 0.0.0.0/0 172.29.1.2
exit
write memory
```

- Si eth0 tenía otra IP antes, se quita con `no ip address <ip>/<máscara>` dentro de `interface eth0`.
- `wr` es la abreviatura de `write memory`. Sin guardar, la configuración se pierde al reiniciar el nodo.

Configuración resultante (`show running-config`):

```
hostname Street-Data
ip route 0.0.0.0/0 172.29.1.2
!
interface eth0
 ip address 172.29.1.1/24
exit
```

### 2. Comprobaciones

```
show interface brief
show ip route
ping 172.29.1.2
ping 8.8.8.8
```

En `show ip route` deben salir estas dos líneas:

```
S>* 0.0.0.0/0 [1/0] via 172.29.1.2, eth0
C>* 172.29.1.0/24 is directly connected, eth0
```

## Switches Street-Data (Dell OS10)

Los switches Dell OS10 tardan varios minutos en arrancar. Solo se pueden configurar cuando el prompt es `OS10#`; con `loading-OS10#` el sistema todavía está cargando y `configure terminal` no existe.

### 1. Arranque

1. Encender los switches de uno en uno y esperar a que cada uno llegue a `OS10#` antes del siguiente.
2. Entrar con el usuario `admin`, que abre directamente el CLI de OS10. Con `linuxadmin` se entra al shell de Linux (`linuxadmin@OS10:~$`) y hay que lanzar `clish`.
3. Si al entrar sale `loading-OS10#`, esperar unos minutos, salir con `exit` y volver a entrar: la sesión abierta durante la carga se queda con ese prompt aunque el sistema ya haya terminado.

Para ver qué está cargando, desde el shell de `linuxadmin` (fuera de `clish`):

```bash
systemctl list-jobs
systemctl --failed
sudo journalctl -f
```

`No jobs running.` en `list-jobs` indica que el arranque ha terminado.

### 2. Cambio de contraseñas

```
configure terminal
username admin password <contraseña> role sysadmin
system-user linuxadmin password <contraseña>
end
write memory
```

- `username admin` cambia la contraseña del CLI y `system-user linuxadmin` la del shell de Linux.
- OS10 exige contraseñas robustas: al menos 9 caracteres con mayúsculas, minúsculas, números y símbolos.
- Al volver a entrar ya no deben salir los avisos «Default password … should be changed».
- Se repite en Street-data-01, 02 y 03.

### 3. Equipos finales para pruebas

Para probar las VLAN (semáforos, cámaras, ambiente) se usan nodos **VPCS**: gastan unos pocos MB cada uno, frente a unos 4 GB de un switch OS10 o 2 GB del FortiGate.

## Proxmox (Promox-02) en Hyper-V

Promox-02 corre como VM de Hyper-V en el mismo PC que la GNS3 VM, con la IP 172.29.0.22/24. La red 172.29.0.0/24 es solo para el FortiGate (port3), los Proxmox y sus VM. Funciona: responde a ping desde Windows y su web está en https://172.29.0.22:8006.

| Elemento | Valor |
| --- | --- |
| Switch Hyper-V | Proyecto (interno) |
| Red | 172.29.0.0/24, puerta de enlace 172.29.0.1 (FortiGate port3) |
| Windows, vEthernet (Proyecto) | 172.29.0.10 |
| GNS3 VM, segunda tarjeta (eth1) | Switch Proyecto, sin IP: la usa el nodo Cloud de GNS3 |
| Promox-02 | 172.29.0.22/24, tarjeta eth0 en el bridge vmbr0 |
| Web de Proxmox | https://172.29.0.22:8006, usuario root |
| SSH | ssh root@172.29.0.22 |

### 1. Switch Proyecto en Hyper-V

En el PC, PowerShell como administrador:

```powershell
New-VMSwitch -Name "Proyecto" -SwitchType Internal
New-NetIPAddress -InterfaceAlias "vEthernet (Proyecto)" -IPAddress 172.29.0.10 -PrefixLength 24
Stop-VM -Name "GNS3 VM"
Add-VMNetworkAdapter -VMName "GNS3 VM" -SwitchName "Proyecto" -Name "Proyecto"
Set-VMNetworkAdapter -VMName "GNS3 VM" -Name "Proyecto" -MacAddressSpoofing On
Start-VM -Name "GNS3 VM"
Connect-VMNetworkAdapter -VMName "Promox-02" -SwitchName "Proyecto"
Set-VMNetworkAdapter -VMName "Promox-02" -MacAddressSpoofing On
```

Comprobación: la GNS3 VM sale con dos tarjetas (GNS3-NAT y Proyecto) y Promox-02 con Proyecto.

```powershell
Get-VMNetworkAdapter -All | Select VMName, Name, SwitchName
```

La IP de la GNS3 VM no cambia: eth0 sigue en 172.29.2.10 y eth1 se queda sin IP.

### 2. Red de Promox-02

Desde la consola de la VM, como root. La tarjeta se llama eth0 (se comprueba con `ip -br link`). Contenido de `/etc/network/interfaces`:

&#91;image: ip -br link en Proxmox: eth0 y vmbr0\]

```
auto lo
iface lo inet loopback

iface eth0 inet manual

auto vmbr0
iface vmbr0 inet static
        address 172.29.0.22/24
        gateway 172.29.0.1
        bridge-ports eth0
        bridge-stp off
        bridge-fd 0

source /etc/network/interfaces.d/*
```

&#91;image: /etc/network/interfaces de Promox-02\]

En `/etc/hosts`, la línea del propio Proxmox:

```
172.29.0.22 pve.mshome.net pve
```

&#91;image: /etc/hosts de Promox-02\]

Se aplica y se comprueba:

```bash
ifreload -a
ip -br a
ping -c 3 172.29.0.10
```

En `ip -br a`, eth0 debe salir UP y vmbr0 con 172.29.0.22/24.

&#91;image: ip -br a: vmbr0 con 172.29.0.22/24\]

## Crear una máquina en Proxmox

Orden para crear una VM en Promox-02: primero bajar la imagen del sistema, después las redes (bridges), comprobar que Proxmox puede arrancar VMs (virtualización anidada), y por último crear la VM y configurarla. Ejemplo: **Zabbix-01** (VM 100) en la red INTERNAL.

### 1. Bajar la imagen cloud de Ubuntu

Para las VM de Proxmox se usa la **imagen cloud oficial de Ubuntu** en lugar de la ISO. Es Ubuntu Server ya instalado en un disco virtual, publicado por Canonical en `cloud-images.ubuntu.com`: los mismos paquetes, kernel y actualizaciones que la ISO, pero sin instalador. No trae usuario ni red configurados; los pone **cloud-init** en el primer arranque con los datos que se le dan a Proxmox (IP, puerta de enlace, DNS, usuario y contraseña).

|  | ISO (instalador) | Imagen cloud + cloud-init |
| --- | --- | --- |
| Instalación | 15–20 min de pantallas en noVNC | No hay, ya viene instalada |
| IP, DNS y usuario | A mano en el instalador | Con `qm set` antes de arrancar |
| Disco | 5–6 GB | Unos 3 GB, se amplía al tamaño deseado |
| Rehacer la VM | Volver a instalar | Repetir los mismos comandos |

**DNS de Proxmox.** Para descargar la imagen, Proxmox tiene que resolver nombres. Tenía como DNS **172.26.208.1** (el Default Switch de Windows, de cuando estaba en esa red) y daba `Could not resolve host`. Se cambia a FW-01:

```bash
cp /etc/resolv.conf /etc/resolv.conf.bak-$(date +%F-%H%M)
sed -i 's/^nameserver .*/nameserver 172.29.0.1/' /etc/resolv.conf
cat /etc/resolv.conf
```

**Nombre del fichero.** La carpeta `releases/26.04/release/` contesta `302 Found` y redirige a `releases/resolute/release/` (resolute = nombre en clave de Ubuntu 26.04). Con `curl -s` sin `-L` la lista sale vacía; con `-L` sigue la redirección:

```bash
curl -sI https://cloud-images.ubuntu.com/releases/26.04/release/ | grep -i location
curl -sL https://cloud-images.ubuntu.com/releases/26.04/release/ | grep -o 'ubuntu-26.04[^"]*amd64.img' | sort -u
```

**Descarga** en la carpeta de imágenes de Proxmox (va por el puerto 443, permitido desde GESTION):

```bash
cd /var/lib/vz/template/iso
wget https://cloud-images.ubuntu.com/releases/resolute/release/ubuntu-26.04-server-cloudimg-amd64.img
ls -lh /var/lib/vz/template/iso/
```

Resultado: `ubuntu-26.04-server-cloudimg-amd64.img` de **822 MB**, junto a la ISO `ubuntu-26.04.1-live-server-amd64.iso`.

### 2. Redes (bridges)

Cada red de Hyper-V a la que llega Promox-02 necesita un bridge en Proxmox. El bridge **no lleva IP** (Proxmox solo hace de switch para las VM), salvo vmbr0, que es la gestión de Proxmox.

| Bridge | Red | Tarjeta | IP en Proxmox |
| --- | --- | --- | --- |
| vmbr0 | GESTION | eth0 (NIC 1) | 172.29.0.22 |
| vmbr1 | INTERNAL | eth1 (NIC 3, MAC …:64:0d) | sin IP |
| vmbr2 | DMZ | NIC 2 (MAC …:64:0e) | sin IP (pendiente) |

Para saber qué interfaz es cada tarjeta se busca por la MAC que tiene en Hyper-V:

```bash
ip -br link | grep -i 64:0d
```

En la web: **pve → System → Network → Create → Linux Bridge**, con Name `vmbr1`, IPv4 y Gateway vacíos, Autostart marcado, VLAN aware sin marcar, Bridge ports `eth1` y Comment `INTERNAL`. Después **Apply Configuration** (sin esto el bridge no se activa). Comprobación: `ip -br link | grep vmbr1` tiene que salir **UP**.

### 3. Virtualización anidada (importante)

Promox-02 es una VM de Hyper-V. Para que Proxmox pueda arrancar VMs con KVM, Hyper-V tiene que pasarle al procesador virtual las extensiones de virtualización (Intel VT-x). Por defecto **no las pasa**, y al arrancar cualquier VM en Proxmox sale este error:

```
qm start 100
KVM virtualisation configured, but not available. Either disable in VM configuration or enable in BIOS.
```

**Comprobación** en PowerShell (Windows, como administrador). Si sale `False`, está desactivada:

```powershell
Get-VMProcessor -VMName "Promox-02" | Select-Object ExposeVirtualizationExtensions
Get-VMMemory -VMName "Promox-02" | Select-Object DynamicMemoryEnabled, Startup
```

**Solución**: con Promox-02 apagada, activar la virtualización anidada y quitar la memoria dinámica (Hyper-V no la admite con virtualización anidada; la VM se queda fija con la memoria de inicio, que tiene que llegar para Proxmox y sus VM):

```powershell
Stop-VM -Name "Promox-02"
Set-VMProcessor -VMName "Promox-02" -ExposeVirtualizationExtensions $true
Set-VMMemory -VMName "Promox-02" -DynamicMemoryEnabled $false
Start-VM -Name "Promox-02"
```

**Comprobación** en la shell de Proxmox: el primer comando tiene que dar un número mayor que 0 y la VM tiene que arrancar sin error.

```bash
grep -c -E 'vmx|svm' /proc/cpuinfo
qm start 100
```

### 4. Crear la VM

En la web: **Create VM**. Valores usados para Zabbix-01:

| Pestaña | Campo | Valor |
| --- | --- | --- |
| General | VM ID / Name | 100 / Zabbix-01 |
| System | Qemu Agent | marcado (el resto por defecto: i440fx, SeaBIOS, VirtIO SCSI single) |
| Disks | Bus / Storage / Size | SCSI 0 / cabina-01-lvm / 10 GB, IO thread marcado, Discard sin marcar (LVM normal) |
| CPU | Sockets / Cores / Type | 1 / 2 / por defecto (x86-64-v2-AES) |
| Memory | Memory | 2048 MiB |
| Network | Bridge / Model / Firewall | vmbr1 / VirtIO / sin marcar (el firewall es FW-01) |
| Confirm | Start after created | sin marcar |

Si el bridge no aparece al crear la VM, se cambia después en **Hardware → Network Device → Edit**. Comprobación: `qm config 100 | grep net0` tiene que mostrar `bridge=vmbr1`.

### 5. Disco de la VM: importar la imagen cloud

**Importante:** la imagen cloud es un **disco ya instalado**, no un CD. No se monta en CD/DVD Drive (así no arranca): se **importa como disco** con `qm disk import`. El disco vacío que crea el asistente se borra y se sustituye por la imagen importada.

**Comprobar el tamaño real de la imagen** (el disco importado debe tener ese tamaño antes de ampliarlo):

```bash
qemu-img info /var/lib/vz/template/iso/ubuntu-26.04-server-cloudimg-amd64.img | grep 'virtual size'
```

Sale `virtual size: 3.5 GiB`. Si el disco scsi0 de la VM tiene otro tamaño (por ejemplo 596M), **no es la imagen** y hay que importarla.

**Comandos**, en la Shell de pve y **de uno en uno** (si la consola web añade `^[[200~` al pegar, ejecutar antes `bind 'set enable-bracketed-paste off'`):

| Paso | Comando | Qué hace |
| --- | --- | --- |
| 1 | `qm set 100 --delete ide2` | Quita la ISO o la imagen del CD/DVD |
| 2 | `qm disk unlink 100 --idlist scsi0 --force` | Borra el disco vacío del asistente |
| 3 | `qm disk import 100 /var/lib/vz/template/iso/ubuntu-26.04-server-cloudimg-amd64.img cabina-01-lvm` | Importa la imagen como disco en la cabina; al final dice `unused0: successfully imported disk 'cabina-01-lvm:vm-100-disk-N'` |
| 4 | `qm set 100 --scsi0 cabina-01-lvm:vm-100-disk-N,iothread=1` | Conecta el disco importado como scsi0 (N = el número que dio el paso 3) |
| 5 | `qm disk resize 100 scsi0 10G` | Amplía el disco de 3,5 GB a 10 GB (Ubuntu agranda la partición solo en el primer arranque) |
| 6 | `qm set 100 --boot order=scsi0` | Arranca desde ese disco |

Comprobación: `qm config 100` → scsi0 con `size=10G`, sin ide2, `boot: order=scsi0`.

### 6. Cloud-init

Unidad de cloud-init: **Hardware → Add → CloudInit Drive**, storage `cabina-01-lvm` (queda en ide0). Después, en la pestaña **Cloud-Init** de la VM:

| Campo | Valor | Ojo |
| --- | --- | --- |
| User | `administrador` | En minúscula |
| Password | la contraseña |  |
| DNS domain | vacío (use host settings) | Es el dominio de búsqueda, **no** la IP del DNS |
| DNS servers | `172.29.4.1` | FW-01 en INTERNAL. En la misma ventana que DNS domain: no confundir los campos |
| Upgrade packages | No | Con Yes hace `apt upgrade` por el puerto 80 en el primer arranque |
| IP Config (net0) | `ip=172.29.4.10/23,gw=172.29.4.1` | La puerta de enlace es la de la red de la VM (INTERNAL), **no** la de Proxmox (172.29.0.1) |

Al terminar: **Regenerate Image**.

**Proxmox no cambia:** su IP (172.29.0.22) y su puerta de enlace (172.29.0.1) siguen en GESTION; en vmbr1 no tiene IP. Lo que se pone en Cloud-Init lo escribe cloud-init **dentro de la VM**.

**Tarjeta de red:** el firewall de Proxmox va desactivado (el firewall es FW-01): `qm set 100 --net0 virtio=<MAC>,bridge=vmbr1` (sin `firewall=1`).

### 7. Clave SSH para entrar en la VM

La imagen cloud de Ubuntu **no deja entrar por SSH con contraseña** (viene así de serie): solo con **clave SSH**. Sin clave, al conectar sale `Permission denied (publickey)`. La contraseña de Cloud-Init sirve para `sudo` y para la Console de Proxmox.

**1. Generar la clave en el portátil** (PowerShell):

```powershell
ssh-keygen -t ed25519
```

- *Enter file in which to save the key*: **Enter** (ruta por defecto `C:\Users\<usuario>\.ssh\id_ed25519`).
- *Enter passphrase*: una frase que se recuerde (recomendado). Cifra la clave privada en el disco: si alguien copia el fichero, sin la frase no le sirve.
- Si pregunta `Overwrite (y/n)?`, ya había una clave: **n** y se usa esa.

Se crean dos ficheros: `id_ed25519` (clave **privada**, no sale nunca del portátil) e `id_ed25519.pub` (clave **pública**, la que se pone en los servidores).

&#91;image: ssh-keygen: clave ED25519 creada en el portátil\]

**2. Copiar la clave pública:**

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

Se copia la línea entera (`ssh-ed25519 AAAA… bitacora@BITACORA-03`).

**3. Ponerla en la VM:** Proxmox → VM → **Cloud-Init → SSH public key → Edit**, pegar la línea → **OK** → **Regenerate Image**. Si la VM ya había arrancado, reiniciarla para que cloud-init añada la clave: `qm reboot 100`.

**4. No escribir la frase en cada conexión (opcional):** el servicio **ssh-agent** de Windows la guarda en memoria mientras dura la sesión. En PowerShell como administrador:

```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
```

Y en el PowerShell normal (pide la frase una vez):

```powershell
ssh-add $env:USERPROFILE\.ssh\id_ed25519
```

**5. Entrar:** `ssh administrador@172.29.4.10`. La primera vez pregunta si se confía en el servidor: **yes**.

### 8. Primer arranque

Pendiente: `qm start 100` y comprobar `ping 172.29.4.10`. Se documenta cuando funcione.

## Monitorización con Zabbix

Zabbix corre en contenedores Podman dentro de **Zabbix-01** (172.29.4.10, INTERNAL). Proxmox se monitoriza **sin agente**, con la plantilla **Proxmox VE by HTTP**, que consulta la API de Proxmox con un usuario de solo lectura y un token.

### 1. Usuario de solo lectura en Proxmox

| Elemento | Valor | Para qué |
| --- | --- | --- |
| Usuario | `zabbix@pve` | Usuario propio de Proxmox (realm pve), sin acceso a la consola de Linux |
| Rol | **PVEAuditor** en `/` | Solo lectura: ve nodos, VMs, almacenamiento y estado; no puede cambiar nada |
| Token | `zabbix@pve!monitoring` | Lo que usa Zabbix para autenticarse, en lugar de una contraseña |
| Privilegios del token | separados (`--privsep 1`) + PVEAuditor | El token solo tiene los permisos que se le asignan a él |

En la Shell de pve, de uno en uno:

```bash
pveum user add zabbix@pve --comment "Zabbix - solo lectura"
pveum acl modify / --users zabbix@pve --roles PVEAuditor
pveum user token add zabbix@pve monitoring --privsep 1 --comment "Token Zabbix"
pveum acl modify / --tokens 'zabbix@pve!monitoring' --roles PVEAuditor
```

**Importante:** `pveum user token add` muestra el **secreto del token** (`value`) **una sola vez**; hay que copiarlo en ese momento. Si se pierde, se borra el token (`pveum user token remove zabbix@pve monitoring`) y se crea otro. En este laboratorio el secreto queda documentado; en un entorno real se guardaría en un gestor de contraseñas y no en la documentación.

Secreto del token del laboratorio: `27da4f52-9c72-4cd7-b690-3f5cd6a39dac`

&#91;image: pveum: usuario zabbix@pve y token monitoring creados, con su secreto\]

**Comprobación** (solo deben salir permisos `*.Audit`, sin `Modify` ni `PowerMgmt`):

```bash
pveum user permissions 'zabbix@pve!monitoring' --path /
```

Resultado: el token solo tiene permisos de lectura (`Datastore.Audit`, `Mapping.Audit`, `Pool.Audit`, `SDN.Audit`, `Sys.Audit`, `VM.Audit`, `VM.GuestAgent.Audit`), todos con propagación `(*)` a todo lo que cuelga de `/`.

&#91;image: pveum: permisos del token zabbix@pve!monitoring, solo Audit\]

**Datos para la plantilla de Zabbix:**

| Macro | Valor |
| --- | --- |
| `{$PVE.URL.HOST}` | 172.29.0.22 |
| `{$PVE.URL.PORT}` | 8006 |
| `{$PVE.TOKEN.ID}` | `zabbix@pve!monitoring` |
| `{$PVE.TOKEN.SECRET}` | el secreto guardado |

Pendiente: regla en FW-01 para que Zabbix-01 (172.29.4.10) llegue a Proxmox (172.29.0.22) por 8006/tcp, porque INTERNAL no puede entrar en GESTION.

## Cabina de almacenamiento (Cabina-01)

Cabina-01 hace de cabina para Proxmox: una VM de Hyper-V con Debian 13 que compartirá un disco por iSCSI (LIO con `targetcli`, de serie en el kernel) y se gestionará por web con Cockpit. Proxmox la usará como almacenamiento compartido.

| Elemento | Valor |
| --- | --- |
| VM de Hyper-V | Cabina-01, generación 2, arranque seguro desactivado |
| Recursos | 2 vCPU, 2 GB de RAM fija |
| Discos | Sistema: sistema.vhdx, 16 GB (sda tras la instalación). Datos: datos.vhdx, 30 GB dinámico, para iSCSI (sdb). En D:\\HyperV\\Cabina. Las letras sda/sdb pueden cambiar entre arranques: el iSCSI usa /dev/disk/by-id. |
| Red | Default Switch durante la instalación; después switch Proyecto con 172.29.0.23/24 |
| Sistema | Debian 13.7 netinst, solo «servidor SSH» y «utilidades estándar» |

### Cómo funciona: una cabina simulada con iSCSI

En un CPD real, los servidores de virtualización no guardan las VM en sus discos internos: usan una **cabina de almacenamiento** que les presta discos por la red. Aquí se simula igual: una VM Debian (Cabina-01) presta un disco por **iSCSI** (SCSI sobre TCP/IP, puerto 3260) y Proxmox lo usa como si fuera un disco propio.

```
Windows (Hyper-V)            Cabina-01 (Debian 13)                       Promox-02
datos.vhdx (30 GB) ──▶ sdb ──▶ targetcli: backstore "datos"
                               destino iqn.2026-11.lab.cabina01:datos
                               lun0 ── red Proyecto, TCP 3260 ──▶ cabina-01-iscsi (disco 30 GB)
                                                                   └─ cabina-01-lvm (vg_cabina)
                                                                       └─ discos de las VM
```

Vocabulario iSCSI y su equivalente en una cabina real:

| Término | Qué es | En el lab |
| --- | --- | --- |
| Target (destino) | El equipo que presta los discos: la cabina | Cabina-01 con targetcli (LIO, incluido en el kernel de Linux) |
| Initiator (iniciador) | El equipo que usa los discos: el servidor | Promox-02 (open-iscsi, incluido en Proxmox) |
| IQN | Nombre único de un destino o de un iniciador | iqn.2026-11.lab.cabina01:datos |
| Backstore | El almacenamiento real que hay detrás | Disco sdb de 30 GB (backstore block «datos») |
| LUN | Cada disco que la cabina publica en un destino | lun0 = el disco de 30 GB |
| Portal | IP y puerto donde escucha la cabina | 172.29.0.23:3260 |
| LVM compartido | Reparto del LUN en volúmenes, visible desde varios servidores | cabina-01-lvm (grupo vg\_cabina) |

Resumen de los pasos (detalle en los apartados siguientes):

1. **Hyper-V:** VM Cabina-01 con dos discos: sistema (16 GB) y datos (30 GB). El de datos es el que se presta.
2. **Debian:** instalación mínima en el disco de sistema, sin tocar el de datos.
3. **Paquetes:** targetcli-fb (la cabina iSCSI) y cockpit (web de gestión).
4. **Red:** IP fija 172.29.0.23/24 en el switch Proyecto, la misma red que Proxmox.
5. **targetcli:** el disco de datos se publica como LUN en un destino iSCSI.
6. **Proxmox:** se conecta al destino (cabina-01-iscsi) y crea encima un LVM compartido (cabina-01-lvm) para los discos de las VM.

### 1. Instalación de Debian

Se arranca la ISO netinst de Debian 13.7 (amd64). Durante la instalación la VM va en el **Default Switch**, que da IP por DHCP y salida a internet para descargar los paquetes.

**Arranque.** En el menú del instalador se elige **Install** (modo texto).

&#91;image: Menú del instalador de Debian 13\]

**Idioma, país y teclado:** español, España.

**Red.** Nombre de la máquina `cabina-01`, dominio vacío. Si falla la configuración por DHCP, la VM no está en el Default Switch: se conecta desde PowerShell y se reintenta.

```powershell
Connect-VMNetworkAdapter -VMName "Cabina-01" -SwitchName "Default Switch"
```

**Usuarios.** Contraseña de root y un usuario normal.

&#91;image: Pantalla de usuarios y contraseñas\]

**Método de particionado:** «Guiado - utilizar todo el disco».

&#91;image: Pantalla de método de particionado\]

**Disco.** Hay que elegir **sdb, 17.2 GB** (sistema). El instalador marca por defecto **sda, 107.4 GB**, que es el disco de datos y no se debe tocar.

&#91;image: Elección de disco: sda 107.4 GB y sdb 17.2 GB\]

**Esquema:** «Todos los ficheros en una partición».

&#91;image: Debian: esquema de particionado en sdb\]

**Resumen.** En sdb quedan ESP (925.9 MB), `/` en ext4 (15.4 GB) e intercambio (899.7 MB). sda queda sin tocar para el iSCSI.

&#91;image: Debian: resumen de particiones\]

**Escribir cambios.** Solo se formatea sdb → **Sí**.

&#91;image: Debian: escribir cambios en sdb\]

**Gestor de paquetes.** ¿Analizar otros medios de instalación? → **No**.

&#91;image: Debian: no analizar medios adicionales\]

**Réplica:** `deb.debian.org`; el proxy se deja vacío.

&#91;image: Debian: réplica deb.debian.org\]

**Encuesta de popularidad:** **No**.

&#91;image: Debian: encuesta de popularidad, No\]

**Programas.** Solo **SSH server** y **Utilidades estándar del sistema**; sin entorno de escritorio. Se marcan y desmarcan con Espacio y se pasa a Continuar con Tab.

&#91;image: Debian: selección de programas\]

**Fin de la instalación.** Antes de pulsar Continuar se quita la ISO desde PowerShell (se puede con la VM encendida); si no, vuelve a arrancar el instalador:

```powershell
Set-VMDvdDrive -VMName "Cabina-01" -Path $null
```

&#91;image: Debian: instalación completada\]

**Primer arranque.** Debian arranca desde el disco (`cabina-01 login:`). Para trabajar cómodo se entra por SSH desde PowerShell, porque la consola de Hyper-V no admite pegar texto: `ssh vicente@<IP>` y después `su -` para pasar a root.

### 2. Paquetes

Con la VM aún en el Default Switch (con internet):

```bash
apt update
apt install -y targetcli-fb cockpit
```

Cockpit queda en **https://172.29.0.23:9090** (usuario vicente o root).

### 3. Red: IP fija en el switch Proyecto

Como root, `/etc/network/interfaces`:

```
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
        address 172.29.0.23/24
        gateway 172.29.0.1
```

DNS: `echo "nameserver 8.8.8.8" > /etc/resolv.conf`. Después, en Windows:

```powershell
Connect-VMNetworkAdapter -VMName "Cabina-01" -SwitchName "Proyecto"
```

Y en la cabina `systemctl restart networking`. Comprobación: `ping 172.29.0.10` (Windows) y `ping 172.29.0.22` (Proxmox) responden.

### 4. Disco de datos de la cabina (Hyper-V)

El disco que la cabina presta por iSCSI es un segundo disco virtual de la VM, separado del sistema. Es **dinámico**: en el disco D: solo ocupa lo que se escribe, aunque la VM lo vea de 30 GB.

| Disco | Fichero | Tamaño | Posición SCSI | En Debian |
| --- | --- | --- | --- | --- |
| Sistema | D:/HyperV/Cabina/sistema.vhdx | 16 GB | 0 | sda |
| Datos (iSCSI) | D:/HyperV/Cabina/datos.vhdx | 30 GB | 1 | sdb |

**Ver los discos de la VM** (tamaño y lo que ocupan de verdad), en PowerShell como administrador:

```powershell
Get-VMHardDiskDrive -VMName "Cabina-01" | ForEach-Object {
    $v = Get-VHD -Path $_.Path
    [PSCustomObject]@{ Posicion = $_.ControllerLocation; Tamano_GB = [math]::Round($v.Size/1GB,1); Ocupa_GB = [math]::Round($v.FileSize/1GB,2); Ruta = $_.Path }
} | Format-Table -AutoSize
```

**Añadir (o cambiar) el disco de datos.** La VM tiene que estar apagada. En el lab se creó primero de 100 GB y se cambió por uno de 30 GB, todavía vacío:

```powershell
Stop-VM -Name "Cabina-01"
# quitar el disco anterior (solo si existe) y borrar su fichero
Remove-VMHardDiskDrive -VMName "Cabina-01" -ControllerType SCSI -ControllerNumber 0 -ControllerLocation 1
Remove-Item "D:\HyperV\Cabina\datos.vhdx"
# crear el disco nuevo, dinámico, y conectarlo a la VM
New-VHD -Path "D:\HyperV\Cabina\datos.vhdx" -SizeBytes 30GB -Dynamic
Add-VMHardDiskDrive -VMName "Cabina-01" -Path "D:\HyperV\Cabina\datos.vhdx"
Start-VM -Name "Cabina-01"
```

**Comprobarlo en la cabina** (como root):

```bash
lsblk
ls -l /dev/disk/by-id/ | grep -v part
```

- `lsblk` muestra sda (16 GB, con las particiones del sistema) y sdb (30 GB, sin particiones). El disco de datos no se formatea ni se particiona: se presta entero.
- Las letras sda/sdb **pueden cambiar entre arranques** (en el instalador el de datos era sda). Por eso el iSCSI apunta al nombre estable de `/dev/disk/by-id`, que es fijo para cada disco: aquí `wwn-0x60022480bbd6534fe5ed340af1ce0c0e -> ../../sdb`.
- Si se cambia el disco por otro, el identificador cambia y hay que actualizarlo en targetcli.

### 5. Publicar el disco por iSCSI (targetcli)

En la cabina, como root. Se usan dos variables para no repetir el nombre del disco y del destino:

```bash
D=/dev/disk/by-id/wwn-0x60022480bbd6534fe5ed340af1ce0c0e
IQN=iqn.2026-11.lab.cabina01:datos
ls -l $D        # debe apuntar a ../../sdb (el disco de 30 GB)
```

Paso a paso:

```bash
# 1. Backstore: registrar el disco de 30 GB como almacenamiento llamado "datos"
targetcli /backstores/block create name=datos dev=$D

# 2. Destino iSCSI: crea el IQN, su grupo tpg1 y el portal 0.0.0.0:3260
targetcli /iscsi create $IQN

# 3. LUN: publicar el disco "datos" dentro del destino (queda como lun0)
targetcli /iscsi/$IQN/tpg1/luns create /backstores/block/datos

# 4. Acceso: sin usuario/contraseña y sin lista de iniciadores (modo laboratorio)
targetcli /iscsi/$IQN/tpg1 set attribute authentication=0 generate_node_acls=1 demo_mode_write_protect=0 cache_dynamic_acls=1

# 5. Guardar y arrancar el servicio en cada inicio
targetcli saveconfig
systemctl enable --now rtslib-fb-targetctl
```

| Atributo | Valor | Significado |
| --- | --- | --- |
| authentication | 0 | Sin CHAP (usuario y contraseña iSCSI) |
| generate\_node\_acls | 1 | Cualquier iniciador puede conectarse, sin darlo de alta |
| demo\_mode\_write\_protect | 0 | Los iniciadores pueden escribir |
| cache\_dynamic\_acls | 1 | Recuerda los iniciadores que se han conectado |

En una cabina de producción se limitaría el acceso a la IQN de cada servidor (ACL) y se añadiría CHAP; para el lab basta con estar en la red Proyecto.

**Comprobaciones en la cabina:**

```bash
targetcli ls                             # disco datos (30 GiB), lun0 y portal 0.0.0.0:3260 [OK]
ss -ltn | grep 3260                      # puerto 3260 escuchando
systemctl status rtslib-fb-targetctl     # servicio habilitado (la configuración se carga al arrancar)
```

La configuración queda guardada en `/etc/rtslib-fb-target/saveconfig.json`.

&#91;image: targetcli ls: disco datos de 30 GiB publicado como lun0\]

### 6. Conexión con Proxmox (iSCSI + LVM compartido)

En Promox-02 hay dos almacenamientos:

| Nombre | Tipo | Para qué |
| --- | --- | --- |
| cabina-01-iscsi | iSCSI | Conexión con la cabina (disco de 30 GB). No se usa directamente. |
| cabina-01-lvm | LVM compartido | Discos de las VM. Es el que se elige al crear una VM (pestaña Disks → Storage). |

Por SSH como root en Proxmox:

```bash
pvesm add iscsi cabina-01-iscsi --portal 172.29.0.23 --target iqn.2026-11.lab.cabina01:datos --content none
BASE=$(pvesm list cabina-01-iscsi | awk 'NR==2{print $1}')
pvesm add lvm cabina-01-lvm --vgname vg_cabina --base $BASE --shared 1 --content images,rootdir
pvesm status
vgs
```

- La primera vez salen avisos `iscsiadm: No records found`: son normales, Proxmox aún no conocía el destino.
- Proxmox ve el disco de la cabina como `cabina-01-iscsi:0.0.0.scsi-3600140598fea027d69845a087c607e38` (30 GB).
- **LVM** reparte ese disco en volúmenes, uno por cada disco de VM (grupo `vg_cabina`, 29,99 GB).
- **Compartido** (`--shared 1`): otro Proxmox conectado a la misma cabina vería los mismos volúmenes y se podrían mover VM entre ellos sin copiar discos.
- Los almacenamientos de Proxmox no se pueden renombrar: para cambiar el nombre se quitan con `pvesm remove` (no borra datos) y se crean de nuevo.

&#91;image: pvesm: conexión iSCSI activa y disco de 30 GB\]

&#91;image: pvesm status y vgs: LVM activo y vg\_cabina de 29,99 GB\]

Resultado en la web de Proxmox:

&#91;image: Proxmox: cabina-01-iscsi y cabina-01-lvm en el nodo pve\]

### 7. Comprobaciones y uso

**En Proxmox:**

```bash
iscsiadm -m session          # sesión abierta con 172.29.0.23:3260 e iqn.2026-11.lab.cabina01:datos
pvesm status                 # cabina-01-iscsi y cabina-01-lvm: active
vgs                          # vg_cabina con unos 30 GB
lsblk                        # el LUN aparece como un disco más de 30 GB
```

**Crear una VM en la cabina:** en la web de Proxmox, *Create VM* → pestaña **Disks** → **Storage: cabina-01-lvm**. El disco de la VM se crea como un volumen dentro de vg\_cabina (`vm-<id>-disk-0`), es decir, se guarda en la cabina y no en el disco del propio Proxmox.

**Orden de arranque del lab:** primero Cabina-01 y después Promox-02, igual que en un CPD (primero el almacenamiento, luego los servidores). Si Proxmox arranca antes, los dos almacenamientos aparecen inactivos hasta que la cabina responde; con la cabina ya encendida, `pvesm status` los vuelve a activar.

## Firewall FW-01 (OpenWrt)

FW-01 es una VM de Hyper-V (generación 1) con **OpenWrt 25.12.6** (imagen x86-64 *generic-ext4-combined*, disco `openwrt.vhdx`). Hace de firewall entre las redes del proyecto. Se gestiona por SSH (`ssh root@172.29.0.1`) y por la web LuCI (`http://172.29.0.1`); el usuario es `root` y la contraseña es la misma para las dos.

| Interfaz (OpenWrt) | Tarjeta | MAC (Hyper-V) | Switch de Hyper-V | Zona | IP |
| --- | --- | --- | --- | --- | --- |
| gestion | eth0 (br-lan) | 00155D290001 | GESTION | lan | 172.29.0.1/23 |
| external | eth1 | 00155D290002 | Default Switch | wan | DHCP |
| dmz | eth2 | 00155D290004 | DMZ | dmz | 172.29.2.1/23 |
| streetdata | eth3 | 00155D290005 | STREET-DATA | streetdata | 192.168.100.1/24 |
| internal | eth4 | 00155D290003 | INTERNAL | internal | 172.29.4.1/23 |

La consola de Hyper-V usa teclado americano; para pegar comandos es más cómodo entrar por SSH.

### 1. Recuperar la contraseña de root (PowerShell + WSL)

Windows no puede leer el disco de OpenWrt (ext4), así que se monta en WSL2 (Ubuntu) con la VM apagada y se deja `root` sin contraseña. En PowerShell como administrador:

```powershell
Stop-VM -Name FW-01 -TurnOff
$vhd = (Get-VMHardDiskDrive -VMName FW-01).Path
wsl --mount --vhd $vhd --partition 2 --name fw01
wsl -u root -- cp /mnt/wsl/fw01/etc/shadow /mnt/wsl/fw01/etc/shadow.bak
wsl -u root -- sed -i 's/^root:[^:]*:/root::/' /mnt/wsl/fw01/etc/shadow
wsl -u root -- grep ^root /mnt/wsl/fw01/etc/shadow
wsl --unmount $vhd
Start-VM -Name FW-01
```

- La partición 2 es la del sistema de OpenWrt; la 1 es la de arranque.
- El `grep` debe mostrar una línea que empieza por `root::` (sin contraseña).
- Antes de modificarlo se guarda una copia en `/etc/shadow.bak`.
- Después se entra con `ssh root@172.29.0.1` sin contraseña y se pone una nueva con `passwd`.

Requisito: WSL2 con una distribución instalada (`wsl -l -v`). Sin WSL, la alternativa es arrancar en la entrada **OpenWrt (failsafe)** de GRUB y ejecutar `mount_root` y `passwd`.

### 2. Web «Policy & Objects» estilo FortiGate (fwpolicy)

**Estado: pendiente de instalar en FW-01.** Probado fuera de FW-01; esta parte se completará con las capturas cuando esté instalado.

Paquete propio `fwpolicy-1.0.tar.gz` que añade a LuCI el menú **Policy & Objects**, con la estructura y los términos de FortiGate y el aspecto normal de LuCI (sin logos ni colores de Fortinet):

| Página | Qué hace |
| --- | --- |
| Firewall Policy | Políticas con ID, Name, From, To, Source, Destination, Service, Action (ACCEPT/DENY), NAT, Log y Bytes. Se evalúan de arriba abajo, gana la primera que coincide y al final está la Implicit Deny (ID 0). |
| Addresses | Objetos Subnet, IP Range e Interface Subnet, y grupos de direcciones. |
| Services | Servicios predefinidos (HTTP, HTTPS, DNS, SSH, RDP, PING, ISCSI, PROXMOX, ZABBIX…), propios (TCP/UDP/SCTP, ICMP, IP) y grupos. |
| Virtual IPs | VIP (DNAT) para publicar un servidor; solo deja pasar tráfico si una política usa la VIP como Destination. |
| Forward Traffic Log | Sesiones de las políticas con Log activado: fecha, origen, destino, servicio, interfaces, acción y política. |

**Cómo funciona.** La configuración se guarda en `/etc/config/fwpolicy`. Al pulsar *Save & Apply*, `/usr/share/fwpolicy/generate.uc` convierte las políticas en reglas nftables (`/etc/fwpolicy/gen/*.nft`) que el firewall de OpenWrt (fw4) carga mediante 4 *includes* en `/etc/config/firewall`.

- Las políticas solo controlan el tráfico que **atraviesa** el firewall. El acceso a FW-01 (web y SSH) sigue en *Network → Firewall*, así que una política mal hecha no deja fuera del equipo.
- El NAT lo decide cada política (casilla NAT = IP de la interfaz de salida).
- Al instalar, las forwardings actuales se convierten en políticas (internal → external y dmz → external, con NAT), para que todo siga funcionando igual.

**Instalación (por consola).** Desde PowerShell en la carpeta del paquete:

```powershell
scp -O fwpolicy-1.0.tar.gz root@172.29.0.1:/tmp/
ssh root@172.29.0.1
```

`-O` es necesario porque OpenWrt no trae servidor SFTP. Dentro de FW-01:

```bash
cd /tmp && tar xzf fwpolicy-1.0.tar.gz && cd fwpolicy && sh install.sh
```

El instalador guarda una copia de `/etc/config/firewall` en `/etc/fwpolicy/backup-FECHA/`, comprueba las reglas con `fw4 check` y, si fallan, deshace los cambios sin aplicar nada. Después se abre la web y se pulsa Ctrl+F5.

**Comprobaciones:**

```bash
cat /etc/fwpolicy/gen/summary.json
nft list chain inet fw4 fwp_forward
nft list counters table inet fw4
logread -e FWP:
```

**Desinstalar:** `sh /tmp/fwpolicy/uninstall.sh` quita los includes, los ficheros y el servicio, y el firewall vuelve a funcionar con las zonas. Las políticas se guardan en `/etc/fwpolicy/fwpolicy.saved-FECHA`. Para desactivarlo sin desinstalar: *Firewall Policy → Policy engine settings → Policy engine enabled*.

**Pruebas hechas antes de instalar:** las reglas generadas se cargaron en el firewall oficial de OpenWrt (fw4) y se pasó tráfico con equipos simulados (interno, DMZ e internet): salida a internet con NAT, bloqueo DMZ → servidores, VIP 8080 → DMZ:80 e Implicit Deny funcionaron como se esperaba. Las 5 páginas se probaron en el navegador, y el instalador y el desinstalador en un sistema simulado.

**Limitaciones:** solo IPv4; sin objetos FQDN, perfiles de seguridad ni horarios; el log sale del log del sistema y se pierde al reiniciar.

### 3. Red según el diseño objetivo

Por SSH. Primero se preparan los cambios y se revisan con `uci changes` (no se aplica nada hasta el `commit`):

```bash
uci -q delete network.internal.netmask
uci set network.internal.ipaddr='172.29.0.1/23'
uci -q delete network.dmz.netmask
uci set network.dmz.ipaddr='172.29.2.1/23'
uci -q delete network.streetdata.netmask
uci set network.streetdata.ipaddr='192.168.100.1/24'

uci set network.ruta_streetdata=route
uci set network.ruta_streetdata.interface='streetdata'
uci set network.ruta_streetdata.target='172.29.8.0/21'
uci set network.ruta_streetdata.gateway='192.168.100.2'
uci set network.ruta_transito=route
uci set network.ruta_transito.interface='streetdata'
uci set network.ruta_transito.target='192.168.101.0/24'
uci set network.ruta_transito.gateway='192.168.100.2'

uci set firewall.streetdata=zone
uci set firewall.streetdata.name='streetdata'
uci add_list firewall.streetdata.network='streetdata'
uci set firewall.streetdata.input='REJECT'
uci set firewall.streetdata.output='ACCEPT'
uci set firewall.streetdata.forward='REJECT'

uci changes
```

Se aplica y se comprueba:

```bash
uci commit network
uci commit firewall
/etc/init.d/network reload
fw4 reload
ip -o -4 addr | grep -E "br-lan|eth2|eth3"
```

- OpenWrt no trae `nohup`; `network reload` solo da la orden a netifd y vuelve enseguida, así que se ejecuta directamente.
- Las zonas de firewall se llaman `lan` (internal), `wan` (external), `dmz` y `streetdata`. El forwarding de la DMZ apuntaba a una zona `external` que no existe y fw4 lo ignoraba; se corrigió con `uci set firewall.@forwarding[1].dest='wan'`.
- Las rutas hacia Street-Data aparecen en `ip route` cuando el FortiGate esté en 192.168.100.2.

### 4. Gestión e internal

La interfaz de br-lan pasa a llamarse `gestion` (solo administración, sin DHCP: todo lleva IP fija) y se crea `internal` en la tarjeta nueva eth4 para las VM de servicio. Primero se añade la tarjeta en Hyper-V (ver el cambio 5 del diseño objetivo). Después, por SSH:

```bash
uci rename network.internal='gestion'
uci del_list firewall.@zone[0].network='internal'
uci add_list firewall.@zone[0].network='gestion'
for s in $(uci show dhcp | grep "\.interface='internal'" | cut -d. -f2); do uci set dhcp.$s.interface='gestion'; uci set dhcp.$s.ignore='1'; done
for s in $(uci show dropbear | grep "\.Interface='internal'" | cut -d. -f2); do uci set dropbear.$s.Interface='gestion'; done

uci set network.internal=interface
uci set network.internal.device='eth4'
uci set network.internal.proto='static'
uci set network.internal.ipaddr='172.29.4.1/23'

uci set firewall.internal=zone
uci set firewall.internal.name='internal'
uci add_list firewall.internal.network='internal'
uci set firewall.internal.input='REJECT'
uci set firewall.internal.output='ACCEPT'
uci set firewall.internal.forward='REJECT'
uci set firewall.fwd_internal_wan=forwarding
uci set firewall.fwd_internal_wan.src='internal'
uci set firewall.fwd_internal_wan.dest='wan'
uci changes
```

Se aplica y se comprueba:

```bash
uci commit dhcp
uci commit network
uci commit firewall
/etc/init.d/network reload
fw4 reload
/etc/init.d/dnsmasq restart
ip -o -4 addr | grep -E "br-lan|eth2|eth3|eth4"
```

- La zona `lan` es la de gestión; en `uci changes` aparece con su nombre interno (`cfg02dc81`).
- El SSH (dropbear) no estaba atado a ninguna interfaz, así que el cambio no corta el acceso.
- Resultado: br-lan 172.29.0.1/23, eth2 172.29.2.1/23, eth3 192.168.100.1/24, eth4 172.29.4.1/23.

### 5. Reglas y forwardings: gestión delimitada

Se borran todas las reglas que trae OpenWrt de serie y solo se crea la necesaria. Las reglas IPSec-ESP e ISAKMP abrían tráfico desde internet hacia `lan` (gestión), y el resto (ping desde internet, IGMP, IPv6) no se usa en el lab. El acceso a FW-01 desde gestión (web y SSH) no depende de estas reglas: lo da la zona `lan`.

```bash
while uci -q delete firewall.@rule[0]; do :; done

uci set firewall.dhcp_renew=rule
uci set firewall.dhcp_renew.name='Allow-DHCP-Renew'
uci set firewall.dhcp_renew.src='wan'
uci set firewall.dhcp_renew.proto='udp'
uci set firewall.dhcp_renew.dest_port='68'
uci set firewall.dhcp_renew.family='ipv4'
uci set firewall.dhcp_renew.target='ACCEPT'

uci set firewall.fwd_gestion_internal=forwarding
uci set firewall.fwd_gestion_internal.src='lan'
uci set firewall.fwd_gestion_internal.dest='internal'
uci set firewall.fwd_gestion_dmz=forwarding
uci set firewall.fwd_gestion_dmz.src='lan'
uci set firewall.fwd_gestion_dmz.dest='dmz'
uci set firewall.fwd_gestion_streetdata=forwarding
uci set firewall.fwd_gestion_streetdata.src='lan'
uci set firewall.fwd_gestion_streetdata.dest='streetdata'

uci changes
uci commit firewall
fw4 reload
uci show firewall | grep -E "=rule|=forwarding|\.src=|\.dest="
ping -c 2 8.8.8.8
```

Resultado:

| Elemento | Origen → destino | Uso |
| --- | --- | --- |
| Regla Allow-DHCP-Renew | wan → FW-01 (UDP 68) | Renovar la IP de eth1 por DHCP |
| Forwarding | lan → wan | Gestión a internet |
| Forwarding | dmz → wan | DMZ a internet |
| Forwarding | internal → wan | VM de servicio a internet |
| Forwarding | lan → internal | Gestión a las VM |
| Forwarding | lan → dmz | Gestión a la DMZ |
| Forwarding | lan → streetdata | Gestión a Street-Data |

Ningún forwarding tiene `lan` como destino, así que desde ninguna otra red se puede iniciar tráfico hacia gestión (las respuestas a lo que sale de gestión sí vuelven). Las zonas dmz, internal, streetdata y wan tienen `input REJECT`: FW-01 solo se administra desde gestión.

### 6. Reglas finales y copias de seguridad

Los forwardings abren todo el tráfico de una zona a otra, así que se quitaron y se sustituyeron por reglas con servicios concretos. En OpenWrt una regla tiene una sola zona de origen y una de destino; para unificar se usa la zona «cualquiera» (`*`) y se acota por redes (IP). Estado final:

| Regla | De | A | Servicios |
| --- | --- | --- | --- |
| Allow-DHCP-Renew | wan | FW-01 | UDP 68 |
| Gestion-Admin | zona gestion | redes DMZ, INTERNAL, Street-Data, gestión de switches y tránsito | Ping, TCP/UDP 22, 80, 443, 161 |
| Internet-HTTPS | redes gestión, DMZ e INTERNAL | wan | TCP 443 |
| DNS-FW01 | redes DMZ e INTERNAL | FW-01 | TCP/UDP 53 |

No queda ningún forwarding: todo lo que no está en estas reglas se rechaza, y desde Street-Data no se puede iniciar nada. Los equipos de gestión, DMZ e INTERNAL deben usar FW-01 como DNS, porque el puerto 53 hacia internet está cerrado.

Una regla con TCP, UDP e ICMP genera una línea por protocolo (`tcp dport {…}`, `udp dport {…}`, `icmp type 8`), así que el ping cabe en la misma regla que los puertos.

El orden en que fw4 evalúa una conexión nueva que atraviesa FW-01: respuestas de conexiones ya permitidas → reglas, de arriba abajo → forwardings de la zona → política `forward` de la zona (REJECT).

**Copia antes de cada cambio y cómo volver atrás:**

```bash
cp /etc/config/firewall /etc/config/firewall.bak-$(date +%Y%m%d-%H%M)
ls /etc/config/firewall.bak-*

# Volver a una copia
cp /etc/config/firewall.bak-FECHA /etc/config/firewall
fw4 reload

# Deshacer cambios todavía sin commit
uci revert firewall
```

Si se borran las zonas (por ejemplo desde la web), las reglas quedan apuntando a zonas que no existen: fw4 avisa con «specifies invalid value» y lo rechaza todo, incluida la web de FW-01 (`ERR_CONNECTION_REFUSED`). Se recuperan creando de nuevo las zonas con los mismos nombres: `gestion` (network gestion, input/output/forward ACCEPT), `wan` (network external, input REJECT, masq y mtu\_fix) y `dmz`, `streetdata` e `internal` (input REJECT, output ACCEPT, forward REJECT).

## Street-Data (switches LiteSwitch en GNS3)

Tres LiteSwitch en triángulo, con dos cables entre cada pareja agrupados en port-channel LACP. Los enlaces entre switches son **enrutados** (`no switchport`) y el enrutamiento es OSPF área 0. Street-data-01 es el core y se conecta a FW-01 por el puerto Te1/1/6.

### 1. Nodos y cables en GNS3 (por la API, desde PowerShell)

La API de la web de GNS3 está en `http://172.29.0.40/v2` y el proyecto Proyecto FP es `f1a214bd-1578-430f-9b5a-276287390824`. En la API los puertos de un LiteSwitch se cuentan desde 0: Gi1/0/1–10 son los adaptadores 0–9 y Te1/1/1–6 los 10–15.

| Cable | Puertos |
| --- | --- |
| Street-data-01 ↔ Street-data-02 (Po1) | Te1/1/1–2 ↔ Te1/1/1–2 |
| Street-data-01 ↔ Street-data-03 (Po2 / Po1) | Te1/1/3–4 ↔ Te1/1/1–2 |
| Street-data-02 ↔ Street-data-03 (Po2) | Te1/1/3–4 ↔ Te1/1/3–4 |
| Cloud STREET-DATA (eth1 de la GNS3 VM) ↔ Street-data-01 | eth1 ↔ Te1/1/6 |
| PC1 (VPCS) ↔ Street-data-01 | Ethernet0 ↔ Gi1/0/1 (VLAN 10) |

Comandos usados (ver nodos y cables, mapear el Cloud a eth1 y crear un cable):

```powershell
$s = "http://172.29.0.40/v2"
$p = "f1a214bd-1578-430f-9b5a-276287390824"
$nodes = irm "$s/projects/$p/nodes"
$nodes | ft name,node_type,status,console
foreach ($l in (irm "$s/projects/$p/links")) { ($l.nodes | % { $id=$_.node_id; "$(($nodes | ? node_id -eq $id).name) $($_.adapter_number)/$($_.port_number)" }) -join "  <->  " }

$cloud = $nodes | ? name -eq "STREET-DATA"
$body = @{ properties=@{ ports_mapping=@(@{ name="eth1"; interface="eth1"; port_number=0; type="ethernet" }) } } | ConvertTo-Json -Depth 5
irm "$s/projects/$p/nodes/$($cloud.node_id)" -Method Put -ContentType "application/json" -Body $body

$sw1 = $nodes | ? name -eq "Street-data-01"
$link = @{ nodes=@( @{node_id=$cloud.node_id; adapter_number=0; port_number=0}, @{node_id=$sw1.node_id; adapter_number=15; port_number=0} ) } | ConvertTo-Json -Depth 5
irm "$s/projects/$p/links" -Method Post -ContentType "application/json" -Body $link
```

- Un cable conectado a un LiteSwitch ya encendido no se detecta hasta reiniciarlo (`.../nodes/<id>/reload`): el puerto sale `NO-CARRIER` en `linux ip -br link`.
- Para entrar a la consola de un nodo sin la web: `ssh gns3@172.29.0.40` y `telnet localhost <puerto de consola>`; se sale con Ctrl+\] y `quit`.
- La IP del VPCS se puede dejar en su fichero de arranque por la API: `irm "$s/projects/$p/nodes/<id>/files/startup.vpc" -Method Post -ContentType "text/plain" -Body "ip 172.30.1.10 255.255.255.192 172.30.1.1"`.

### 2. Configuración de Street-data-01

```
enable
configure terminal
hostname Street-data-01
ip routing
vlan 10
 name SEMAFOROS
vlan 20
 name CAMARAS
vlan 30
 name SENSORES
vlan 99
 name GESTION-SW
exit
interface range Gi1/0/1-3
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
interface range Gi1/0/4-6
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
interface range Gi1/0/7-9
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
interface range Te1/1/1-2
 channel-group 1 mode active
interface range Te1/1/3-4
 channel-group 2 mode active
interface Port-channel1
 description Hacia-Street-data-02
 no switchport
 ip address 192.168.101.1 255.255.255.0
interface Port-channel2
 description Hacia-Street-data-03
 no switchport
 ip address 192.168.102.1 255.255.255.0
interface Te1/1/6
 description Hacia-FW-01
 no switchport
 ip address 192.168.100.2 255.255.255.0
interface Vlan10
 ip address 172.30.1.1 255.255.255.192
interface Vlan20
 ip address 172.30.1.65 255.255.255.192
interface Vlan30
 ip address 172.30.1.129 255.255.255.192
interface Vlan99
 description Gestion-del-switch
 ip address 172.29.7.1 255.255.255.255
exit
ip route 0.0.0.0 0.0.0.0 192.168.100.1
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20,30,99 priority 4096
router ospf 1
 router-id 172.29.7.1
 network 172.30.1.0 0.0.0.255 area 0
 network 192.168.101.0 0.0.0.255 area 0
 network 192.168.102.0 0.0.0.255 area 0
 network 172.29.7.1 0.0.0.0 area 0
 passive-interface default
 no passive-interface Port-channel1
 no passive-interface Port-channel2
 default-information originate
end
write memory
```

- Los enlaces entre switches son enrutados, así que el spanning-tree no bloquea nada entre ellos (la redundancia la da OSPF); la prioridad 4096 deja a Street-data-01 como raíz de sus VLAN y protege los puertos de acceso.
- `show spanning-tree summary` muestra «0 vlans» mientras no haya equipos conectados a los puertos de acceso.
- `linux <comando>` ejecuta un comando del sistema del switch; `linux ip -br link` muestra el estado real de cada puerto (eth0 = Gi1/0/1 … eth15 = Te1/1/6).

### 3. Prueba

PC1 (VPCS) en Gi1/0/1 con `ip 172.30.1.10 255.255.255.192 172.30.1.1` y `save`.

```bash
# Desde PC1
ping 172.30.1.1
# Desde FW-01
ping -c 3 192.168.100.2
# Desde la GNS3 VM (gestión, puerta de enlace FW-01)
ping -c 3 172.30.1.10
```

Resultado: responde todo; desde la GNS3 VM, TTL 62 (dos saltos: FW-01 y Street-data-01). El ping del PC1 a 192.168.100.1 no responde, como debe (Street-Data no puede entrar a FW-01).

## Problemas encontrados y solución

| Problema | Causa | Solución |
| --- | --- | --- |
| FortiGate no arranca en la GNS3 VM de VirtualBox | Sin virtualización anidada (no existe `/dev/kvm`) | Pasar la GNS3 VM a Hyper-V con `ExposeVirtualizationExtensions` |
| La GNS3 VM no arranca o va justa | El PC tiene 8 GB y Hyper-V reserva toda la RAM de la VM | Cerrar Edge y Chrome (`Stop-Process`); hay un slot libre para ampliar a 16 GB DDR4 SO-DIMM 3200 |
| Zip de la GNS3 VM incompleto | Descarga cortada | Volver a descargarlo de GitHub en la misma versión del cliente |
| «The project is not opened» | El proyecto se cierra al quedarse sin clientes | `auto_close` desactivado por la API (puerto 80, no 3080) |
| Router: «Network unreachable» | eth0 sin la IP de la red 172.29.1.0/24 aplicada | Volver a aplicar la IP en `interface eth0` y guardar con `write memory` |
| Ping del router a 8.8.8.8 sin respuesta; debug: «Denied by forward policy check (policy 0)» | El objeto `RT-Street-Data` apuntaba a 172.29.1.2 (el propio FortiGate) | Cambiarlo a 172.29.1.1/32 |
| La IP de la GNS3 VM cambia al reiniciar el PC | El Default Switch reparte una red distinta en cada arranque | Switch interno GNS3-NAT con NAT de Windows e IP fija 172.29.2.10 por netplan |
| netplan apply: «expected mapping» o «invalid address» | Líneas de configuración aún con #, caracteres sueltos al inicio del fichero y un punto de más en la IP | Quitar los #, dejar 172.29.2.10/24 y volver a aplicar. En la consola de la VM, revisar / y \* antes de pulsar Enter (el teclado los cambia) |
| Los switches Dell OS10 se quedan en «System is loading» y la CPU de la GNS3 VM va al 100 % | La GNS3 VM solo tenía 2 vCPU para el FortiGate y tres switches OS10 | Con la VM apagada: Set-VMProcessor "GNS3 VM" -Count 8. Arrancar los nodos de uno en uno |
| Dell OS10 sigue en «loading-OS10#» aunque ya ha terminado de cargar | La sesión de CLI se abrió mientras el switch cargaba y conserva ese estado | Salir con exit y volver a entrar con admin; el prompt pasa a OS10# |
| Proxmox, ifreload: «bridge port nic0 does not exist» | La tarjeta de la VM se llama eth0, no nic0 | sed -i 's/nic0/eth0/g' /etc/network/interfaces y ifreload -a |
| New-NetIPAddress: «Invalid parameter InterfaceAlias vEthernet (LAB-PVE)» | El switch de Hyper-V todavía no existía | Crear antes el switch Proyecto como interno y usar vEthernet (Proyecto) |
| Texto sin sentido en la consola de Hyper-V del Proxmox | La consola de Hyper-V no admite pegar texto | Escribir a mano o entrar por SSH: ssh root@172.29.0.22 |

### Pendiente

- [ ] Conectar en GNS3 el nodo Cloud (eth1) al port3 del FortiGate y comprobar `ping 172.29.0.1` desde Proxmox.
- [ ] Reglas del FortiGate: port3 → port2 con NAT y port3 → port1. Con la que ya existe (port1 → port2) se llega al máximo de 3 de la licencia.
- [ ] Red de gestión de los switches Dell: por decidir.
- [ ] Revisar las redes de las VLAN de Street-Data en el plan de direccionamiento.
- [ ] Borrar la ruta estática sobrante `172.29.1.1/32` del FortiGate.
- [ ] La política solo permite PING, HTTP y HTTPS: para que Street-Data resuelva nombres hay que añadir el servicio **DNS**.
- [ ] Fijar port2 en estática o tener en cuenta que la red del Default Switch cambia al reiniciar el PC.

## Recuperar la contraseña de un servidor Ubuntu

Si se pierde la contraseña de `administrador` en un servidor Ubuntu con LVM (probado en podman-01 y zabbix-01), se deja en blanco editando `/etc/shadow` con `sed` desde el arranque y se pone una nueva al entrar.

### 1. Arrancar desde GRUB

En el menú de GRUB, `c` abre la consola `grub>`. El kernel está en una partición `/boot` separada, así que las rutas van sin `/boot`. Cada comando va en su propia línea; TAB completa los nombres.

```
ls (hd0,gpt2)/
set root=(hd0,gpt2)
linux /vmlinuz root=/dev/mapper/ubuntu--vg-ubuntu--lv rw init=/bin/bash
initrd /initrd.img
boot
```

El nombre del volumen es `ubuntu--vg-ubuntu--lv`: dos guiones, `vg`, uno, `ubuntu`, dos, `lv`. Con la ruta bien escrita se llega a `root@<servidor>:/#`; con la ruta mal, a la shell de emergencia `dracut:/#`.

### 2a. Si se llega a `root@<servidor>:/#`

```bash
mount -o remount,rw /
passwd administrador
sync
reboot -f
```

### 2b. Si se llega a `dracut:/#`

En dracut no existe `chroot`, así que se usa `sed`:

```bash
lvm vgchange -ay
mkdir -p /sysroot
mount /dev/ubuntu-vg/ubuntu-lv /sysroot
sed -i 's/^administrador:[^:]*:/administrador::/' /sysroot/etc/shadow
grep administrador /sysroot/etc/shadow
sync
umount /sysroot
reboot -f
```

El `grep` debe mostrar la línea empezando por `administrador::`. Después se arranca normal, se entra con `administrador` pulsando solo Enter en la contraseña y se ejecuta `passwd` en ese momento.

### Errores vistos

- **Errata en el `sed`:** escribir `adminstrador` (sin la `i`) en la parte derecha renombra el usuario en `/etc/shadow`. Síntoma: `Login incorrect` y `passwd` responde `user unknown`. Se arregla con `sed -i 's/^adminstrador:/administrador:/' /etc/shadow`.
- **`mount -o remount, rw /`** con espacio tras la coma no monta en escritura: va sin espacio.
- **`initrd` en la misma línea que `linux`:** no se carga; va en una línea aparte.

### Contraseña del Admin de Zabbix

Zabbix corre en Podman sin root, con el usuario `administrador` (pod `zabbix`, base de datos en el contenedor `zbx-db`). Para dejar la contraseña de la web en `zabbix`:

```bash
podman exec -it zbx-db psql -U zabbix -d zabbix
```

```sql
UPDATE users SET passwd = '$2a$10$ZXIvHAEP2ZM.dLXTm6uPHOMVlARXX7cqjbhM6Fn0cANzkCQBWpMrS' WHERE username = 'Admin';
\q
```

Se entra en `http://<IP>:8080` con `Admin` / `zabbix` y se cambia en User settings → Profile → Change password.

## Anexo A. Capturas del laboratorio

Capturas tomadas durante el montaje, ordenadas por apartados y numeradas (Figura A.n). Se incluyen los pasos que muestran cómo se hace cada cosa, el resultado correcto y los errores más importantes con los que se encontró el montaje. Las capturas que ya aparecen dentro de los apartados del manual (instalación de Debian en Cabina-01, red de Proxmox, iSCSI, clave SSH y usuario de Zabbix) no se repiten aquí.

| Apartado | Figuras | Contenido |
| --- | --- | --- |
| A.1 Hyper-V y Windows | A.1 – A.4 | Switches, tarjetas de red de las VM, virtualización anidada, memoria y rutas del portátil. |
| A.2 FW-01 (OpenWrt) | A.5 – A.19 | Instalación de la imagen, recuperación de la contraseña con WSL, interfaces, zonas y conectividad con Street-Data. |
| A.3 GNS3 y Street-Data | A.20 – A.33 | Primeras pruebas con Dell OS10, GNS3 VM, plantilla LiteSwitch, configuración de Street-data-01 y prueba de extremo a extremo. |
| A.4 Cabina-01 (iSCSI) | A.34 – A.37 | Discos, red y target iSCSI de la cabina. |
| A.5 Proxmox y Zabbix-01 | A.38 – A.63 | Bridges, creación de la VM, errores encontrados, imagen cloud, cloud-init y acceso por SSH. |

### A.1 Hyper-V y Windows

**Figura A.1.** Adaptadores de red de cada VM y switches de Hyper-V, con MAC address spoofing

&#91;image: Figura A.1: Adaptadores de red de cada VM y switches de Hyper-V, con MAC address spoofing\]

**Figura A.2.** Virtualización anidada desactivada en Promox-02 (ExposeVirtualizationExtensions = False)

&#91;image: Figura A.2: Virtualización anidada desactivada en Promox-02 (ExposeVirtualizationExtensions = False)\]

**Figura A.3.** Memoria de Promox-02: dinámica desactivada y 2048 MB de inicio (después se amplió)

&#91;image: Figura A.3: Memoria de Promox-02: dinámica desactivada y 2048 MB de inicio (después se amplió)\]

**Figura A.4.** tracert desde Windows sin ruta al lab: el tráfico sale por el Livebox

&#91;image: Figura A.4: tracert desde Windows sin ruta al lab: el tráfico sale por el Livebox\]

### A.2 FW-01 (OpenWrt)

**Figura A.5.** Descarga de la imagen de OpenWrt 25.12.6 (x86-64, ext4 combined)

&#91;image: Figura A.5: Descarga de la imagen de OpenWrt 25.12.6 (x86-64, ext4 combined)\]

**Figura A.6.** Conversión de la imagen a VHDX con qemu-img

&#91;image: Figura A.6: Conversión de la imagen a VHDX con qemu-img\]

**Figura A.7.** Disco y tarjetas de red de FW-01 en Hyper-V

&#91;image: Figura A.7: Disco y tarjetas de red de FW-01 en Hyper-V\]

**Figura A.8.** Error: memoria insuficiente al arrancar FW-01

&#91;image: Figura A.8: Error: memoria insuficiente al arrancar FW-01\]

**Figura A.9.** Primer arranque de OpenWrt 25.12, con el aviso de root sin contraseña

&#91;image: Figura A.9: Primer arranque de OpenWrt 25.12, con el aviso de root sin contraseña\]

**Figura A.10.** WSL disponible para montar el disco de FW-01

&#91;image: Figura A.10: WSL disponible para montar el disco de FW-01\]

**Figura A.11.** Disco de FW-01 montado en WSL (partición 2)

&#91;image: Figura A.11: Disco de FW-01 montado en WSL (partición 2)\]

**Figura A.12.** Contraseña de root borrada en /etc/shadow

&#91;image: Figura A.12: Contraseña de root borrada en /etc/shadow\]

**Figura A.13.** LuCI: interfaces iniciales (lan y wan)

&#91;image: Figura A.13: LuCI: interfaces iniciales (lan y wan)\]

**Figura A.14.** LuCI: interfaces dmz, external, internal y streetdata

&#91;image: Figura A.14: LuCI: interfaces dmz, external, internal y streetdata\]

**Figura A.15.** uci show network: cada interfaz con su tarjeta

&#91;image: Figura A.15: uci show network: cada interfaz con su tarjeta\]

**Figura A.16.** Direcciones IP de las interfaces de FW-01

&#91;image: Figura A.16: Direcciones IP de las interfaces de FW-01\]

**Figura A.17.** LuCI: zonas del firewall

&#91;image: Figura A.17: LuCI: zonas del firewall\]

**Figura A.18.** Error: la web de FW-01 rechaza la conexión tras borrar las zonas

&#91;image: Figura A.18: Error: la web de FW-01 rechaza la conexión tras borrar las zonas\]

**Figura A.19.** FW-01 llega a Street-data-01 (192.168.100.2)

&#91;image: Figura A.19: FW-01 llega a Street-data-01 (192.168.100.2)\]

### A.3 GNS3 y Street-Data

**Figura A.20.** Dell OS10: configure terminal no existe mientras el sistema carga

&#91;image: Figura A.20: Dell OS10: configure terminal no existe mientras el sistema carga\]

**Figura A.21.** Dell OS10: modo configuración disponible

&#91;image: Figura A.21: Dell OS10: modo configuración disponible\]

**Figura A.22.** Menú de la GNS3 VM

&#91;image: Figura A.22: Menú de la GNS3 VM\]

**Figura A.23.** netplan de la GNS3 VM en GESTION (172.29.0.40/23)

&#91;image: Figura A.23: netplan de la GNS3 VM en GESTION (172.29.0.40/23)\]

**Figura A.24.** Importación de la plantilla LiteSwitch

&#91;image: Figura A.24: Importación de la plantilla LiteSwitch\]

**Figura A.25.** Plantilla LiteSwitch: 16 interfaces, Te1/1/1–6 en los adaptadores 10–15

&#91;image: Figura A.25: Plantilla LiteSwitch: 16 interfaces, Te1/1/1–6 en los adaptadores 10–15\]

**Figura A.26.** Enlaces del proyecto: LACP entre switches, Cloud y PC1

&#91;image: Figura A.26: Enlaces del proyecto: LACP entre switches, Cloud y PC1\]

**Figura A.27.** Spanning-tree rapid-pvst con prioridad 4096 en Street-data-01

&#91;image: Figura A.27: Spanning-tree rapid-pvst con prioridad 4096 en Street-data-01\]

**Figura A.28.** show ip interface brief de Street-data-01

&#91;image: Figura A.28: show ip interface brief de Street-data-01\]

**Figura A.29.** show vlan brief: VLAN 10, 20, 30 y 99

&#91;image: Figura A.29: show vlan brief: VLAN 10, 20, 30 y 99\]

**Figura A.30.** linux ip -br link: puertos Te dentro de los port-channel

&#91;image: Figura A.30: linux ip -br link: puertos Te dentro de los port-channel\]

**Figura A.31.** VPCS PC1 con 172.30.1.10/26

&#91;image: Figura A.31: VPCS PC1 con 172.30.1.10/26\]

**Figura A.32.** PC1 llega a su puerta de enlace

&#91;image: Figura A.32: PC1 llega a su puerta de enlace\]

**Figura A.33.** GNS3 VM → PC1 a través de FW-01 y Street-data-01 (TTL 62)

&#91;image: Figura A.33: GNS3 VM → PC1 a través de FW-01 y Street-data-01 (TTL 62)\]

### A.4 Cabina-01 (iSCSI)

**Figura A.34.** Discos de Cabina-01 en Hyper-V (sistema y datos, dinámicos)

&#91;image: Figura A.34: Discos de Cabina-01 en Hyper-V (sistema y datos, dinámicos)\]

**Figura A.35.** Red de Cabina-01 (172.29.0.23) y pings a Windows y Proxmox

&#91;image: Figura A.35: Red de Cabina-01 (172.29.0.23) y pings a Windows y Proxmox\]

**Figura A.36.** lsblk en Cabina-01: disco de datos de 30 GB

&#91;image: Figura A.36: lsblk en Cabina-01: disco de datos de 30 GB\]

**Figura A.37.** IQN del target iSCSI de Cabina-01

&#91;image: Figura A.37: IQN del target iSCSI de Cabina-01\]

### A.5 Proxmox y Zabbix-01

**Figura A.38.** Consola de Proxmox: inicio de sesión e interfaces

&#91;image: Figura A.38: Consola de Proxmox: inicio de sesión e interfaces\]

**Figura A.39.** Almacenamiento cabina-01-lvm en Proxmox (32,20 GB)

&#91;image: Figura A.39: Almacenamiento cabina-01-lvm en Proxmox (32,20 GB)\]

**Figura A.40.** Crear el bridge vmbr1

&#91;image: Figura A.40: Crear el bridge vmbr1\]

**Figura A.41.** vmbr1 activo sobre eth1 (MAC …64:0d)

&#91;image: Figura A.41: vmbr1 activo sobre eth1 (MAC …64:0d)\]

**Figura A.42.** Crear VM: pestaña System

&#91;image: Figura A.42: Crear VM: pestaña System\]

**Figura A.43.** Crear VM: disco en cabina-01-lvm

&#91;image: Figura A.43: Crear VM: disco en cabina-01-lvm\]

**Figura A.44.** Crear VM: CPU

&#91;image: Figura A.44: Crear VM: CPU\]

**Figura A.45.** Crear VM: memoria

&#91;image: Figura A.45: Crear VM: memoria\]

**Figura A.46.** Crear VM: red sin firewall de Proxmox

&#91;image: Figura A.46: Crear VM: red sin firewall de Proxmox\]

**Figura A.47.** Crear VM: resumen antes de crear

&#91;image: Figura A.47: Crear VM: resumen antes de crear\]

**Figura A.48.** Hardware de Zabbix-01 recién creada

&#91;image: Figura A.48: Hardware de Zabbix-01 recién creada\]

**Figura A.49.** Error: KVM no disponible (falta la virtualización anidada)

&#91;image: Figura A.49: Error: KVM no disponible (falta la virtualización anidada)\]

**Figura A.50.** Proxmox con el DNS antiguo (172.26.208.1): no resuelve nombres

&#91;image: Figura A.50: Proxmox con el DNS antiguo (172.26.208.1): no resuelve nombres\]

**Figura A.51.** La carpeta 26.04 redirige a resolute; nombre de la imagen cloud

&#91;image: Figura A.51: La carpeta 26.04 redirige a resolute; nombre de la imagen cloud\]

**Figura A.52.** Imagen cloud descargada (822 MB)

&#91;image: Figura A.52: Imagen cloud descargada (822 MB)\]

**Figura A.53.** Disco de 596 MB y RAM de Promox-02 (5,9 GB)

&#91;image: Figura A.53: Disco de 596 MB y RAM de Promox-02 (5,9 GB)\]

**Figura A.54.** Error: imagen cloud montada como CD y disco de 596 MB

&#91;image: Figura A.54: Error: imagen cloud montada como CD y disco de 596 MB\]

**Figura A.55.** Tamaño real de la imagen cloud: 3,5 GiB

&#91;image: Figura A.55: Tamaño real de la imagen cloud: 3,5 GiB\]

**Figura A.56.** Importación de la imagen como disco de la VM

&#91;image: Figura A.56: Importación de la imagen como disco de la VM\]

**Figura A.57.** Cloud-Init con errores: puerta de enlace y DNS domain

&#91;image: Figura A.57: Cloud-Init con errores: puerta de enlace y DNS domain\]

**Figura A.58.** Cloud-Init correcto

&#91;image: Figura A.58: Cloud-Init correcto\]

**Figura A.59.** qm config de Zabbix-01 tras importar el disco

&#91;image: Figura A.59: qm config de Zabbix-01 tras importar el disco\]

**Figura A.60.** ciupgrade a 0 y cloud-init regenerado

&#91;image: Figura A.60: ciupgrade a 0 y cloud-init regenerado\]

**Figura A.61.** SSH sin clave: Permission denied (publickey)

&#91;image: Figura A.61: SSH sin clave: Permission denied (publickey)\]

**Figura A.62.** Aviso de huella cambiada en known\_hosts

&#91;image: Figura A.62: Aviso de huella cambiada en known\_hosts\]

**Figura A.63.** Comprobaciones dentro de Zabbix-01: red, disco y DNS

&#91;image: Figura A.63: Comprobaciones dentro de Zabbix-01: red, disco y DNS\]

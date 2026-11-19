# Web del panel en la DMZ: LXC en Proxmox + Podman

```
Portátil (GESTION) → FW-01 eth2 → DMZ 172.29.2.0/23 → Web-01 (LXC 101 en Promox-02, vmbr2) 172.29.2.10
                                                       └─ Podman → contenedor nginx (puerto 80)
```

Dos tipos de contenedores: **LXC** (contenedor de sistema, lo gestiona Proxmox) y **Podman** (contenedor de aplicación, se gestiona con Cockpit).

Reglas de FW-01 que ya lo permiten: **Gestion-Admin** (GESTION → DMZ por 22, 80, 443), **Internet-HTTPS** y **DNS-FW01** (salida de la DMZ). No hay forwarding DMZ → INTERNAL, así que Web-01 no llega a Zabbix ni al resto.

## 1. Hyper-V: tarjeta de Promox-02 en la DMZ (PowerShell como administrador)

```powershell
Add-VMNetworkAdapter -VMName Promox-02 -SwitchName DMZ -Name DMZ
Set-VMNetworkAdapter -VMName Promox-02 -Name DMZ -MacAddressSpoofing On
```

`MacAddressSpoofing` es obligatorio por la virtualización anidada: sin él no sale el tráfico de los contenedores.

## 2. Proxmox: crear vmbr2 (SSH a root@172.29.0.22)

```bash
ip -br link
```

Busca la tarjeta nueva (la que no está en ningún bridge, por ejemplo `eth2`) y añade al final de `/etc/network/interfaces`, cambiando `eth2` por su nombre real:

```
auto vmbr2
iface vmbr2 inet manual
        bridge-ports eth2
        bridge-stp off
        bridge-fd 0
#DMZ
```

```bash
ifreload -a
```

vmbr2 va sin IP: Proxmox no queda expuesto en la DMZ.

## 3. Crear el LXC Web-01

```bash
pveam update && T=$(pveam available --section system | awk '/debian-13-standard/{print $2}' | tail -1) && pveam download local $T && echo $T
pct create 101 local:vztmpl/$T --hostname web-01 --cores 1 --memory 1024 --rootfs cabina-01-lvm:8 \
  --net0 name=eth0,bridge=vmbr2,ip=172.29.2.10/23,gw=172.29.2.1 --nameserver 172.29.2.1 \
  --unprivileged 1 --features nesting=1,keyctl=1 --onboot 1 --password
pct start 101 && pct enter 101
```

`nesting` y `keyctl` son necesarios para que Podman funcione dentro del LXC.

## 4. Dentro de Web-01: Podman, Cockpit y nginx

La DMZ solo sale a internet por HTTPS, así que primero se pasan los repositorios de Debian a https:

```bash
sed -i 's|http://|https://|g' /etc/apt/sources.list.d/*.sources /etc/apt/sources.list 2>/dev/null
apt update && apt install -y podman cockpit cockpit-podman curl sudo
adduser administrador && usermod -aG sudo administrador
mkdir -p /srv/web && curl -fsSL -o /srv/web/index.html https://raw.githubusercontent.com/vicentruso-web/ceedlauncher/proyecto-fp/proyecto-fp/panel-laboratorio.html
podman run -d --name web --restart=always -p 80:80 -v /srv/web:/usr/share/nginx/html:ro docker.io/library/nginx:stable-alpine
systemctl enable --now podman-restart.service
```

- `administrador`: Cockpit en Debian no deja entrar como root.
- `podman-restart.service`: el contenedor `web` arranca solo al reiniciar Web-01.
- Actualizar la web: repetir el `curl`. El contenedor lee `/srv/web`, no hay que tocarlo.

## 5. Probar

- Desde el portátil: **http://172.29.2.10** → sale el panel.
- Si no sale: en Proxmox `pct exec 101 -- curl -sI localhost`, y dentro de Web-01 `ping 172.29.2.1`.

## 6. Gestionar

| Qué | Dónde |
| --- | --- |
| El LXC (arrancar, consola, CPU/RAM/disco, copias, snapshots) | Web de Proxmox: https://172.29.0.22:8006 → CT 101 |
| El contenedor nginx (imagen, logs, reiniciar) | Cockpit de Cabina-01: https://172.29.0.23:9090 → Añadir anfitrión nuevo → `172.29.2.10`, usuario `administrador` → Contenedores de Podman |

Cockpit de Cabina-01 entra por SSH (22), que Gestion-Admin ya permite: no hay que abrir el 9090 en FW-01. Desde un Cockpit en INTERNAL no funcionaría (no hay paso INTERNAL → DMZ).

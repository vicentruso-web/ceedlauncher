# Migración: panel y Zabbix en máquinas separadas

Nov 2, 2026 · @Virt

## Qué vamos a hacer

- **Máquina Zabbix**: Zabbix completo (servidor, web y su base de datos) y Grafana. Se queda con la IP de ahora, **172.16.1.91**, para no tocar nada de lo que ya apunta ahí (FortiNAC, vCenter, Proxmox, permisos SNMP de los equipos).
- **Máquina del panel**: la aplicación (contenedor python3.12) y el nginx (frontal). IP nueva.
- **Base de datos del panel aparte**, en una de estas dos formas:
  - **A**: en su propia máquina (IP nueva).
  - **B**: en la máquina del panel, pero en un segundo disco solo para ella.
- La máquina vieja **no se borra**: se apaga y queda como vuelta atrás.

No se crea nada desde cero: cada contenedor se vuelve a crear con el mismo comando con el que se creó (Podman lo guarda), y los datos se copian tal cual. Las imágenes son las mismas oficiales de ahora.

**Antes de empezar**: las máquinas nuevas con Debian 13 (o Ubuntu), el usuario **administrador** y una IP provisional, todas en la misma red 172.16.1.x (el panel lee las MAC de la red donde está). Hazlo fuera de horas: Zabbix y el panel estarán parados lo que tarde la copia.

## Paso 1 · En la vieja (zabbix-01), como administrador: apuntar cómo está todo

Guarda el comando de cada contenedor y de cada pod, la red de la máquina y el nombre del volumen de la base del panel.

```
mkdir -p ~/migracion/contenedores ~/migracion/volumenes && chmod 700 ~/migracion && cd ~/migracion
leer='import json,sys,shlex; d=json.load(sys.stdin); d=d[0] if isinstance(d,list) else d; c=d.get("CreateCommand") or d.get("Config",{}).get("CreateCommand"); print(shlex.join(c) if c else "# sin comando guardado")'
for p in $(podman pod ps --format '{{.Name}}'); do podman pod inspect "$p" | podman exec -i python3.12 python3 -c "$leer" > contenedores/pod-$p.sh; done
for c in $(podman ps --format '{{.Names}}' | grep -v -- '-infra$'); do podman inspect "$c" | podman exec -i python3.12 python3 -c "$leer" > contenedores/$c.sh; done
podman inspect panel-db --format '{{range .Mounts}}{{.Name}}{{end}}' > volumen-panel-db.txt
{ podman --version; podman ps --format '{{.Names}}  {{.PodName}}  {{.Image}}'; podman volume ls; ip -4 addr; ip route; cat /etc/resolv.conf; } > lista.txt
ls contenedores; cat volumen-panel-db.txt
```

Debe salir un fichero por contenedor (zbx-db.sh, zbx-server.sh, zbx-web.sh, python3.12.sh, frontal.sh, panel-db.sh, el de Grafana…) y el nombre del volumen de la base del panel. Si alguno sale vacío o falta, para y dímelo.

## Paso 2 · En la vieja: copia de seguridad de las dos bases (por si acaso)

```
cd ~/migracion
podman exec zbx-db sh -c 'pg_dump -U "$POSTGRES_USER" -Fc "${POSTGRES_DB:-$POSTGRES_USER}"' > zabbix.dump
podman exec panel-db pg_dump -p 5433 -U panel -Fc panel > panel.dump
ls -lh *.dump
```

Los dos ficheros tienen que pesar algo (no 0). Además siguen estando las copias diarias del panel en \~/accesos/web/datos/copias-bd.

## Paso 3 · En la vieja: parar todo y copiar los datos

Desde aquí Zabbix y el panel están parados.

```
podman stop -a
cd ~/migracion
for v in $(podman volume ls --format '{{.Name}}'); do podman volume export "$v" -o volumenes/$v.tar; done
podman unshare tar czpf casa.tar.gz --exclude=./.local/share/containers --exclude=./migracion --exclude=./.cache -C ~ .
crontab -l > crontab.txt 2>/dev/null
du -sh ~/migracion
```

casa.tar.gz lleva la carpeta del usuario entera: \~/accesos con web/datos (sobre todo **config/clave-servidor.key**, sin la que no se pueden leer las contraseñas guardadas en el panel), el certificado del frontal y las copias diarias.

## Paso 4 · En cada máquina nueva, con sudo: prepararla

```
sudo apt-get update && sudo apt-get install -y podman
id administrador || sudo adduser administrador
sudo loginctl enable-linger administrador
echo 'net.ipv4.ip_unprivileged_port_start=80' | sudo tee /etc/sysctl.d/99-puertos.conf && sudo sysctl --system
```

Luego entra por SSH directamente como administrador (no con sudo su) y activa que los contenedores vuelvan solos al reiniciar:

```
systemctl --user enable podman-restart.service
```

## Paso 5 · Traer los ficheros (la vieja aún encendida)

**En la máquina Zabbix**, como administrador (se trae todo):

```
scp -r administrador@172.16.1.91:migracion ~/
```

**En la máquina del panel**, como administrador:

```
mkdir -p ~/migracion/volumenes && cd ~/migracion
for f in contenedores casa.tar.gz crontab.txt volumen-panel-db.txt panel.dump; do scp -r administrador@172.16.1.91:migracion/$f . ; done
```

**Donde vaya la base del panel** (la máquina A, o la del panel si es la forma B), como administrador:

```
mkdir -p ~/migracion/volumenes && cd ~/migracion
scp administrador@172.16.1.91:migracion/volumen-panel-db.txt .
V=$(cat volumen-panel-db.txt); scp administrador@172.16.1.91:migracion/volumenes/$V.tar volumenes/
```

## Paso 6 · Cambio de IP

1. En la vieja: `sudo poweroff`
2. En la máquina Zabbix pon la IP **172.16.1.91** con la misma máscara, puerta 172.16.1.193 y DNS que tenía la vieja (están en \~/migracion/lista.txt) y reiníciala.
3. En tu PC, para que SSH no proteste por el cambio de máquina: `ssh-keygen -R 172.16.1.91`

## Paso 7 · Máquina Zabbix, como administrador: levantar Zabbix y Grafana

```
cd ~/migracion
podman unshare tar xzpf casa.tar.gz -C ~ --exclude=./accesos
V=$(cat volumen-panel-db.txt)
for f in volumenes/*.tar; do v=$(basename "$f" .tar); [ "$v" = "$V" ] && continue; podman volume create "$v" && podman volume import "$v" "$f"; done
for f in contenedores/pod-*.sh; do [ "$f" = contenedores/pod-control.sh ] || bash "$f"; done
for f in contenedores/*.sh; do case "$f" in */pod-*|*/panel-db.sh|*/python3.12.sh|*/frontal.sh) continue;; esac; grep -q -- '--pod control' "$f" && continue; bash "$f"; done
sleep 60; podman ps --format '{{.Names}}  {{.Status}}'
podman logs --since 2m zbx-server | tail -20
ss -ltn | grep -E ':(8080|10051|3000) '
```

Tienen que salir zbx-db, zbx-server, zbx-web y Grafana «Up», y los puertos 8080, 10051 y 3000 escuchando. El **10051** es importante: por ahí le manda datos el panel (Veeam, System Center…). Abre la web de Zabbix como siempre y mira que están tus hosts y que llegan datos nuevos.

## Paso 8 · Base de datos del panel

### 8.1 · El disco aparte (con sudo, en la máquina donde vaya la base)

Mira cuál es el disco nuevo y vacío. **Cuidado: el que escribas se borra entero.**

```
lsblk
read -p "Nombre del disco nuevo y vacío (por ejemplo sdb): " D
sudo mkfs.ext4 -L paneldb /dev/$D
sudo mkdir -p /datos && echo 'LABEL=paneldb /datos ext4 defaults 0 2' | sudo tee -a /etc/fstab && sudo mount /datos
sudo mkdir /datos/panel-db && sudo chown administrador: /datos/panel-db && sudo chmod 700 /datos/panel-db
```

### 8.2 · Poner los datos y arrancarla (como administrador)

```
cd ~/migracion; V=$(cat volumen-panel-db.txt)
podman unshare tar xpf volumenes/$V.tar -C /datos/panel-db
```

**Forma A (máquina propia)**, escucha en la red por el 5433:

```
podman run -d --name panel-db --restart=always --network host -v /etc/localtime:/etc/localtime:ro -v /datos/panel-db:/var/lib/postgresql/data docker.io/library/postgres:16-alpine postgres -c port=5433
```

**Forma B (en la máquina del panel)**, solo escucha dentro de la máquina:

```
podman run -d --name panel-db --restart=always --network host -v /etc/localtime:/etc/localtime:ro -v /datos/panel-db:/var/lib/postgresql/data docker.io/library/postgres:16-alpine postgres -c port=5433 -c listen_addresses=127.0.0.1
```

Comprobar (en las dos formas):

```
podman exec panel-db psql -p 5433 -U panel -d panel -c "SELECT count(*) FROM ips_maquinas"
```

Tiene que dar el mismo número de máquinas que tenías. La contraseña del usuario panel es la de siempre: va dentro de los datos copiados.

## Paso 9 · Máquina del panel, como administrador: levantar el panel

```
cd ~/migracion
podman unshare tar xzpf casa.tar.gz -C ~
bash contenedores/pod-control.sh
bash contenedores/python3.12.sh
bash contenedores/frontal.sh
sleep 90; podman ps --format '{{.Names}}  {{.Status}}'
```

La primera vez python3.12 tarda un poco porque instala sus paquetes de Debian.

## Paso 10 · Lo que cambia por estar separados

**En el panel** (entra con tu usuario maestro, que funciona aunque la base aún no conecte):

- **⚙ › Integraciones › Zabbix › Conexión**: si la dirección de Zabbix era 127.0.0.1 o localhost, cámbiala por **172.16.1.91** con el mismo puerto. Si ya ponía 172.16.1.91, no se toca.
- **Base de datos del panel › Conexión**: forma A, servidor = la IP de la máquina de la base (puerto 5433). Forma B, se queda 127.0.0.1.
- Si las gráficas de Grafana no salen: ⚙ › Integraciones › Grafana › «Preparar todo».

**Fuera del panel** (por la IP nueva del panel):

- **Veeam**: Menú › Opciones › Event Forwarding › Syslog servers: cambia la IP por la del panel nuevo.
- **DNS**: el nombre con el que entráis al panel tiene que apuntar a la IP nueva del panel (el certificado comodín sigue valiendo).
- **FortiGate**: las reglas que usaba el panel desde 172.16.1.91 (salida a Sophos Central, System Center, Znuny, SSH a los Linux de Instalaciones) tienen que permitir también la IP nueva del panel. Las de Zabbix no cambian.

## Paso 11 · Comprobación final

- Web de Zabbix: hosts, datos de los últimos minutos, VMware, Proxmox y FortiNAC al día.
- Panel: entra por su nombre con https sin aviso del navegador; la lista de IPs, accesos y racks como antes.
- ⚙ › Integraciones › Zabbix «Conectado»; Veeam recibe eventos tras la siguiente copia; la ficha de un PC trae datos de System Center.
- Base de datos: la copia diaria de las 03:00 aparece al día siguiente en web/datos/copias-bd de la máquina del panel.
- Reinicia cada máquina nueva (`sudo reboot`) y mira que todo vuelve solo (`podman ps`).
- Si crontab.txt no está vacío, dímelo y te digo en qué máquina va.

## Si algo va mal: volver atrás

Apaga las máquinas nuevas (o quítale la IP .91 a la de Zabbix) y enciende la vieja: lo tiene todo tal cual y sus contenedores arrancan solos. Si alguno no arranca: `podman start -a`.

Cuando lleves una semana bien, borra \~/migracion en todas las máquinas (lleva las contraseñas de los contenedores y copias con IPs): `rm -rf ~/migracion`

# Guía: Ubuntu Server, Podman, Cockpit y Zabbix

Oct 9, 2026 · @Virt

Esta guía recoge, en orden, solo los pasos que funcionaron para montar los servidores Ubuntu con Podman, Cockpit y Zabbix. Se irá ampliando con cada paso nuevo.

| Máquina | Papel |
| --- | --- |
| podman-01 | Consola central: Cockpit desde donde se ven y gestionan los contenedores de las demás máquinas |
| Máquina Zabbix | Máquina destino donde corre el pod de Zabbix |

- Todo se instala desde repositorios oficiales: `apt` de Ubuntu e imágenes oficiales con nombre completo (por ejemplo `docker.io/library/...`).
- Los valores entre `<>` son marcadores: cámbialos por los tuyos. Nunca escribas contraseñas reales en ficheros ni en esta guía.

## 1. Red con netplan (en cada servidor)

La IP fija se pone en el fichero que crea el instalador de Ubuntu, `00-installer-config.yaml`.

1. Mira el nombre de la tarjeta de red con `ip a` (por ejemplo `ens18`).
2. Edita el fichero: `sudo nano /etc/netplan/00-installer-config.yaml`
3. Deja este contenido, respetando los espacios (netplan falla si el YAML no está bien alineado):

```yaml
network:
  version: 2
  ethernets:
    <tarjeta>:
      dhcp4: false
      addresses: [<IP>/<prefijo>]
      routes:
        - to: default
          via: <puerta_de_enlace>
          on-link: true
      nameservers:
        addresses: [<DNS1>, <DNS2>]
```

4. Prueba con `sudo netplan try` y, si va bien, `sudo netplan apply`.
5. Comprueba con `ip route`: debe salir `default via <puerta_de_enlace>`.

Notas:

- La puerta de enlace va en `routes` con `via`. No uses `gateway4:`, que está obsoleto.
- `on-link: true` hace falta cuando la puerta de enlace no está en la misma red que la IP. Va alineado con `via`.

## 2. Activar SSH (en cada servidor)

```
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

- Si el cortafuegos está activo: `sudo ufw allow ssh`.
- Desde tu PC te conectas con `ssh <usuario>@<IP_servidor>`.

## 3. Consola central podman-01: Podman y Cockpit

Podman se instala directamente en el servidor Ubuntu, sin contenedores intermedios.

1. Instala Podman y compruébalo:

```
sudo apt update
sudo apt install -y podman
podman --version
podman run --rm docker.io/library/hello-world
```

2. Instala Cockpit con su módulo de Podman:

```
sudo apt install -y cockpit cockpit-podman
sudo systemctl enable --now cockpit.socket
```

3. Activa la opción de añadir otros anfitriones (las versiones nuevas de Cockpit la traen desactivada):

```
printf '[WebService]\nAllowMultiHost=true\n' | sudo tee /etc/cockpit/cockpit.conf
sudo systemctl restart cockpit
cat /etc/cockpit/cockpit.conf
```

El `cat` debe mostrar `[WebService]` y `AllowMultiHost=true`.

4. Abre `https://<IP_podman-01>:9090` y entra con tu usuario de Linux. Acepta el aviso del certificado la primera vez.
5. Pulsa **Acceso administrativo** arriba a la derecha para salir del modo limitado.
6. Tras cambiar `cockpit.conf`, cierra sesión en Cockpit (Sesión → Cerrar sesión) y vuelve a entrar. Recargar la página no basta.

Las imágenes se descargan siempre con el nombre completo de una imagen oficial, en la máquina donde va a correr el contenedor: `podman pull docker.io/library/<imagen>:<etiqueta>`. En Cockpit: recuadro **Imágenes** → ⋮ → **Descargar nueva imagen**.

## 4. Preparar cada máquina destino (por ejemplo, la de Zabbix)

Cada máquina que quieras ver desde podman-01 necesita Podman, SSH, Cockpit y el socket de Podman siempre activo.

1. En la máquina destino, con tu usuario normal (no root):

```
sudo apt update
sudo apt install -y podman cockpit cockpit-podman openssh-server
sudo systemctl enable --now ssh cockpit.socket
systemctl --user enable --now podman.socket
sudo loginctl enable-linger $USER
```

`podman.socket` con `linger` deja Podman escuchando aunque no haya nadie conectado.

2. En el Cockpit de podman-01, pulsa **\<usuario>@podman-01** arriba a la izquierda y luego **Añadir anfitrión nuevo**.
3. Rellena **Anfitrión** con la IP de la máquina destino y **Usuario** con tu usuario de Linux en ella. Acepta la huella SSH y pon su contraseña.
4. Elige esa máquina en el mismo menú y entra en **Contenedores de Podman**.

Si no ves los contenedores, cambia **Propietario** a **Todo**, porque se crearon con tu usuario y no como administrador. Recuerda que Cockpit muestra la máquina que pone arriba a la izquierda.

## 5. Pod de Zabbix (en la máquina Zabbix)

Zabbix son tres contenedores dentro de un pod llamado `zabbix`. El pod comparte red, así que se encuentran entre ellos en `127.0.0.1`.

| Contenedor | Imagen oficial | Función |
| --- | --- | --- |
| zbx-db | `docker.io/library/postgres:16-alpine` | Base de datos PostgreSQL 16 |
| zbx-server | `docker.io/zabbix/zabbix-server-pgsql:alpine-7.0-latest` | Servidor Zabbix 7.0 LTS: recoge métricas y lanza alertas |
| zbx-web | `docker.io/zabbix/zabbix-web-nginx-pgsql:alpine-7.0-latest` | Interfaz web (nginx) |

Ejecútalo por SSH en la máquina Zabbix, todo en la misma sesión de terminal. El primer comando pide la clave de la base de datos sin mostrarla ni guardarla en el historial, y los tres contenedores usan la misma.

```
read -s -p "Clave de la base de datos: " CLAVE; echo

podman pod create --name zabbix -p 8080:8080 -p 10051:10051

podman run -d --pod zabbix --name zbx-db --restart=always \
  -e POSTGRES_USER=zabbix -e POSTGRES_PASSWORD="$CLAVE" -e POSTGRES_DB=zabbix \
  -v zbx-db:/var/lib/postgresql/data \
  docker.io/library/postgres:16-alpine

podman run -d --pod zabbix --name zbx-server --restart=always \
  -e DB_SERVER_HOST=127.0.0.1 -e POSTGRES_USER=zabbix -e POSTGRES_PASSWORD="$CLAVE" -e POSTGRES_DB=zabbix \
  docker.io/zabbix/zabbix-server-pgsql:alpine-7.0-latest

podman run -d --pod zabbix --name zbx-web --restart=always \
  -e ZBX_SERVER_HOST=127.0.0.1 -e DB_SERVER_HOST=127.0.0.1 \
  -e POSTGRES_USER=zabbix -e POSTGRES_PASSWORD="$CLAVE" -e POSTGRES_DB=zabbix \
  -e PHP_TZ=Europe/Madrid \
  docker.io/zabbix/zabbix-web-nginx-pgsql:alpine-7.0-latest
```

- Puerto 8080: la web. Puerto 10051: los agentes hablan con el servidor.
- Comprueba con `podman pod ps` y `podman ps --pod`: los tres deben salir como `Up`.
- Si la web no carga, mira `podman logs zbx-server`.
- Copia de seguridad de la definición del pod: `podman generate kube zabbix > zabbix.yaml`. Se recrea con `podman kube play zabbix.yaml`.

## 6. Primer acceso a la web de Zabbix

1. Abre `http://<IP_zabbix>:8080`.
2. Entra con el usuario por defecto **Admin** (con A mayúscula) y la contraseña **zabbix**.
3. Cámbiala nada más entrar: Usuarios → Admin → Cambiar contraseña.

Si te equivocas 5 veces, Zabbix bloquea el usuario 30 segundos. Espera y vuelve a probar. Si aun así no entras, restablece la contraseña por defecto desde la máquina Zabbix:

```
podman exec -it zbx-db psql -U zabbix -d zabbix -c "update users set passwd=md5('zabbix'), attempt_failed=0 where username='Admin';"
```

Para Cockpit no hay usuario de Zabbix: se entra con el usuario de Linux de cada máquina.

## 7. API de Zabbix con el usuario zabbixapi

La API web (JSON-RPC) está en `http://<IP_zabbix>:8080/api_jsonrpc.php`. Se usa con un usuario propio, `zabbixapi`, que puede ver, crear y borrar hosts solo por la API y no puede entrar en la web. Todo se hace en la web de Zabbix, en este orden.

1. **Rol** (Usuarios → Roles de usuario → Crear rol)
   - Nombre: `api-hosts`. Tipo de usuario: **Admin**.
   - Elementos de la interfaz: todos desmarcados.
   - Acceso a la API: **Activado**.
   - Métodos de API: **Lista de permitidos** con `host.*`, `hostgroup.get`, `template.get`, `item.get`, `problem.get`, `trigger.get`, `history.get`.
   - Acciones: todas desmarcadas.
2. **Grupo** (Usuarios → Grupos de usuarios → Crear grupo)
   - Nombre: `API`.
   - Acceso a la interfaz: **Desactivado**, para que no pueda entrar en la web.
   - Pestaña Permisos de grupos de hosts: todos los grupos con **Lectura-escritura**.
3. **Usuario** (Usuarios → Usuarios → Crear usuario)
   - Usuario: `zabbixapi`. Grupo: `API`. Contraseña: `<contraseña_larga>`.
   - Pestaña Permisos: rol `api-hosts`.
4. **Token** (Usuarios → Tokens de API → Crear token de API)
   - Usuario: `zabbixapi`. Copia el token al crearlo, porque solo se muestra una vez.
5. **Prueba** desde cualquier equipo con acceso a la máquina Zabbix:

```
curl -s -X POST http://<IP_zabbix>:8080/api_jsonrpc.php \
  -H "Content-Type: application/json-rpc" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{"jsonrpc":"2.0","method":"host.get","params":{"output":["hostid","name"]},"id":1}'
```

Debe devolver la lista de hosts en JSON. Guarda el token fuera de scripts y repositorios.

# Notas del chat del proyecto FP

Resumen de lo hablado y decidido en el chat de Claude Code "Continuación del proyecto" (18-20 nov 2026).

## Qué hay en esta carpeta

- `panel-laboratorio.html`: el panel del laboratorio (artifact https://claude.ai/artifact/ExSf5jg6in2hJTgk2A5JhS).
- `lab-gns3-fortigate-street-data.md`, `guia-ubuntu-podman-cockpit-zabbix.md`, `migracion-panel-zabbix.md`: copiados de los documentos del laboratorio.
- `guia-web-dmz-lxc.md`: cómo montar la web del panel en la DMZ.

## Cambios en el panel

- Documentación → Switches: la configuración de cada switch se ve como una consola (`show running-config`). El botón «Copiar configuración» copia solo los comandos.
- Documentación → Máquinas: rutas de los discos de cada VM.
  - Cabina-01: `C:\HyperV\Cabina-01\Virtual Hard Disks\sistema.vhdx` (16 GB) y `datos.vhdx` (30 GB)
  - FW-01: `D:\HyperV\FW-01\openwrt.vhdx`
  - GNS3 VM: `C:\GNS3\VM\gns3vm-disk1.vhdx` (sistema) y `gns3vm-disk2.vhdx` (proyectos e imágenes: el que hay que conservar)
  - Promox-02: 2 discos en `C:\ProgramData\Microsoft\Windows\Virtual Hard Disks\` (falta el nombre completo: `Get-VMHardDiskDrive -VMName Promox-02 | Format-List Path`)
- El secreto del token de Proxmox se mantiene en el panel (es un laboratorio).

## Decisión: contenedores

| Red | Tipo de contenedor | Qué lleva | Se gestiona desde |
| --- | --- | --- | --- |
| INTERNAL | Podman en Zabbix-01 (VM 100) | Zabbix | Cockpit |
| DMZ | LXC de Proxmox, Web-01 (CT 101, 172.29.2.10) | nginx con `apt` y la web del panel | Web de Proxmox |

Así el proyecto muestra cómo se gestionan los dos tipos de contenedores. Solo hay un Podman.

## Dónde nos quedamos

- Creando Web-01 desde la web de Proxmox: General hecho (hostname WEB-01, contraseña y clave `clavessh@laboratoriofp`).
- La lista de plantillas sale vacía: ejecutar `pveam update` en la Shell de Proxmox y descargar `debian-13-standard`.
- Si `pveam update` falla, probablemente es FW-01: GESTION solo sale a internet por el 443 y `download.proxmox.com` usa el 80.
- Pendiente: pasos 1 y 2 de la guía (tarjeta DMZ en Hyper-V y vmbr2) si no están hechos; añadir Web-01 al panel cuando funcione.

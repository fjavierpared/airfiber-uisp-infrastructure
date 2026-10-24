# AirFiber UISP Infrastructure

Migración de la consola UISP de una red airFiber a un servidor dedicado con Ubuntu Server 26.04 LTS y Docker. Incluye instalación del sistema operativo, despliegue de UISP en contenedor, backup de la consola anterior y restauración en el nuevo servidor, configuración posterior a la migración (incluidas las alertas) y stack de monitoreo.

| Responsable | Fecha de implementación | Estado |
| :--- | :--- | :--- |
| Ing. Francisco Paredes | Octubre 2026 | En producción |

## Contexto

La consola UISP anterior tenía limitaciones de almacenamiento que afectaban la estabilidad del servicio y no permitían procesar backups de gran tamaño. Se decidió reemplazarla por un servidor físico dedicado, instalado desde cero, usando el disco completo y manteniendo la configuración existente mediante la restauración del backup.

Resultados:

- Backup de la consola anterior restaurado en el nuevo servidor UISP, sin pérdida de datos (base de datos, usuarios, sitios, dispositivos y configuración).
- Sistema configurado tras la migración: dirección del servidor, dispositivos, detección de caídas y alertas.
- Reconexión automática de todas las antenas airFiber al nuevo servidor.
- Disco de 465 GB asignado en su totalidad al sistema (10 % de uso tras la migración).
- Servicios con arranque automático, verificado con reinicio del servidor.

## Arquitectura

```
                     +-----------------------------------------------+
                     |  uispairfiber (servidor físico)               |
                     |  Ubuntu Server 26.04 LTS, kernel 7.0          |
                     |  10.0.0.2/22, gateway 10.0.0.1                |
                     |  Disco 465 GB                                 |
                     |                                               |
                     |  Docker - red bridge "monitoring"             |
                     |  +----------+  +-------------+  +---------+   |
 Antenas airFiber ---+->|   unms   |  | uptime-kuma |  | grafana |   |
                     |  |  :8443   |  |    :3001    |  |  :3000  |   |
                     |  +----+-----+  +-------------+  +---------+   |
                     |       |                                       |
                     |  /opt/unms-data (datos persistentes)          |
                     +-----------------------------------------------+
```

| Componente | Detalle |
| :--- | :--- |
| Hostname | `uispairfiber` |
| Sistema operativo | Ubuntu Server 26.04 LTS, instalación nativa |
| Kernel | 7.0.0-38-generic |
| Red | `10.0.0.2/22`, gateway `10.0.0.1` |
| Almacenamiento | 465 GB, disco completo asignado al sistema |
| Contenedores | Docker Engine 29.9.2, Docker Compose |

| Contenedor | Imagen | Puertos |
| :--- | :--- | :--- |
| `unms` | `nico640/docker-unms:latest` (UISP 3.1.65, PHP 8.3.32) | 8443/tcp → 443, 2055/udp |
| `uptime-kuma` | `louislam/uptime-kuma:1` | 3001/tcp |
| `grafana` | `grafana/grafana:latest` | 3000/tcp |

## Procedimiento

### 1. Instalación del sistema operativo

ISO oficial de Ubuntu Server 26.04 LTS (https://ubuntu.com/download/server), grabada en USB con Rufus. Instalación limpia con los siguientes parámetros:

| Parámetro | Valor |
| :--- | :--- |
| Hostname | `uispairfiber` |
| Usuario | `airfiber` |
| Red | IP estática `10.0.0.2/22`, gateway `10.0.0.1` |
| Almacenamiento | Disco completo (465 GB) asignado al sistema, sin particiones adicionales |

Actualización del sistema y limpieza de kernels antiguos:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt autoremove --purge -y
```

### 2. Docker

Instalación desde el repositorio oficial de Docker:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

Red interna y directorio de datos de UISP:

```bash
sudo docker network create monitoring
sudo mkdir -p /opt/unms-data
```

### 3. Despliegue de UISP

```bash
sudo docker run -d \
  --name unms \
  --restart unless-stopped \
  -p 8443:443 \
  -p 2055:2055/udp \
  -v /opt/unms-data:/config \
  --network monitoring \
  nico640/docker-unms:latest
```

| Parámetro | Función |
| :--- | :--- |
| `--restart unless-stopped` | Arranque automático tras reinicio del host |
| `-p 8443:443` | Interfaz web y conexión de dispositivos |
| `-p 2055:2055/udp` | Recepción de NetFlow |
| `-v /opt/unms-data:/config` | Persistencia de base de datos y configuración |
| `--network monitoring` | Red compartida con los servicios de monitoreo |

La imagen utilizada incluye PHP 8.3.32 con los parches de seguridad vigentes.

### 4. Migración: backup y restauración

1. **Backup de la consola anterior.** Se generó y descargó el backup completo desde la consola UISP en producción (*Settings → Backups*). Tamaño: 1.5 GB.
2. **Restauración en el nuevo servidor.** Desde la interfaz web del nuevo servidor (`https://10.0.0.2:8443`) se cargó el archivo con la opción *Restore from file*. La versión de UISP del servidor nuevo es igual o superior a la de origen, requisito para aplicar el backup.
3. **Validación.** Se restauraron la base de datos, los usuarios, los sitios, los dispositivos y la configuración completa de la consola anterior.

Métricas observadas durante la restauración:

| Métrica | Valor |
| :--- | :--- |
| Escritura en disco | > 218 GB |
| Pico de CPU del contenedor | 198 % |
| Interrupciones del servicio | Ninguna |

### 5. Configuración posterior a la migración

Inicio de sesión con las credenciales restauradas de la consola anterior y ajuste de la configuración al nuevo entorno:

| Área | Configuración |
| :--- | :--- |
| Dirección del servidor | Hostname/IP actualizado a `10.0.0.2` (el backup restaura la dirección de la consola de origen) |
| Dispositivos | Verificación de la conexión de todas las antenas airFiber al nuevo servidor |
| Detección de caídas | Umbrales de caída de dispositivos (*Settings → Devices → Outage Settings*) |
| Alertas | Notificaciones de caída de dispositivos habilitadas para los usuarios administradores (*Settings → Users*) |

Las antenas airFiber se reconectaron automáticamente al nuevo servidor y comenzaron a reportar, sin intervención manual en los equipos. Se verificó el arranque automático de UISP reiniciando el servidor.

### 6. Monitoreo

```bash
sudo docker run -d \
  --name uptime-kuma \
  --restart unless-stopped \
  -p 3001:3001 \
  -v uptime-kuma:/app/data \
  --network monitoring \
  louislam/uptime-kuma:1

sudo docker run -d \
  --name grafana \
  --restart unless-stopped \
  -p 3000:3000 \
  -v grafana-data:/var/lib/grafana \
  --network monitoring \
  grafana/grafana:latest
```

- **Uptime Kuma:** monitoreo de disponibilidad del servidor y de las antenas.
- **Grafana:** visualización de métricas.

## Estado de los servicios

| Servicio | Acceso | Estado |
| :--- | :--- | :--- |
| UISP | https://10.0.0.2:8443 | En producción, antenas conectadas, alertas configuradas |
| Uptime Kuma | http://10.0.0.2:3001 | Operativo |
| Grafana | http://10.0.0.2:3000 | Operativo |
| Almacenamiento | — | 465 GB, 10 % en uso |

## Operación

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'   # estado de contenedores
sudo docker stats --no-stream                                         # consumo de recursos
sudo docker logs --tail 100 unms                                      # logs de UISP
df -h /                                                               # espacio en disco
```

# Aurkitu - Infraestructura y Aprovisionamiento

Este documento detalla los pasos necesarios para preparar el servidor de producción antes del despliegue de la aplicación. Se prioriza la estabilidad del sistema mediante la separación de datos y el control estricto de permisos.

## 1. Aprovisionamiento de Almacenamiento

Debido al gran tamaño de las imágenes Docker, los volúmenes de base de datos y los ficheros generados por los usuarios, **no es viable utilizar la partición raíz (`/`) del sistema operativo**, ya que su llenado provocaría la caída del servidor.

Se requiere una partición dedicada montada en `/docker_data`.
```bash
sudo mkdir /docker_data
```
### Montaje Persistente
La partición debe configurarse en `/etc/fstab` para montarse automáticamente al arranque del sistema.

```bash
# Ejemplo de configuración en /etc/fstab
# <file system>    <mount point>   <type>  <options>       <dump>  <pass>
UUID=tu-uuid-disco /docker_data    ext4    defaults,nofail        0       2
```

**NOTA:** se puede obtener el UUID con los siguientes comandos

```bash
# Para ver particiones no montadas con espacio disponible
lsblk -o NAME,SIZE,TYPE, MOUNTPOINT
# Sabiendo su nombre, vemos su formato, uuid, etc
blkid (dev/vda4)
```
---
## 2. Instalación y Configuración de Docker

### 2.1. Instalación de Docker Engine
Se recomienda seguir estrictamente la guía oficial para Ubuntu
* [Guía oficial de instalación Docker en Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
### 2.2. Configuración de Almacenamiento (`data-root`)
**Importante:** Por defecto, Docker almacena imágenes y contenedores en `/var/lib/docker` (partición raíz). Dado el tamaño de las imágenes y el crecimiento de los logs, esto puede llenar rápidamente el disco de sistema y bloquear el servidor.

Para evitarlo, se configura Docker para usar la partición dedicada `/docker_data`.

1.  Crear el directorio:
    ```bash
    sudo mkdir -p /docker_data/docker-storage
    ```
2.  Editar/Crear el archivo `/etc/docker/daemon.json`:
    ```json
    {
      "data-root": "/docker_data/docker-storage",
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "10m",
        "max-file": "3"
      }
    }
    ```
3.  Reiniciar Docker:
    ```bash
    sudo systemctl restart docker
    ```
---
## 3. Usuarios y Permisos

Es crítico respetar los UIDs y GIDs específicos para que coincidan con los usuarios internos de los contenedores y los agentes de despliegue. El usuario `deploy-runner` (UID 999) coincide intencionalmente con el usuario `postgres` dentro del contenedor oficial de PostgreSQL.

El usuario `deploy-runner` debe ser también miembro del grupo `docker`

| **Usuario**         | **UID** | **GID** | **Rol**                                              |
| ------------------- | ------- | ------- | ---------------------------------------------------- |
| **`birt`**          | 1000    | 1000    | Propietario de la aplicación, fotos y documentos.    |
| **`deploy-runner`** | 999     | 987     | Ejecuta CI/CD, contenedores y mueve APKs, sin shell  |

---

## 4. Estructura de Directorios

La infraestructura distingue entre **código/configuración** (en `/opt`) que es el lugar de la estructura de archivos de Linux para aplicaciones de terceros que se instalan manualmente y **datos persistentes** (en `/docker_data`)

### 4.1. Código y Configuración (`/opt`)

El repositorio se clona desde este **repositorio de despliegue** [aurkitu-deploy](https://github.com/xarrondobirt/aurkitu-deploy)

```bash
cd /opt
sudo mkdir aurkitu
sudo chown deploy-runner:deploy-runner aurkitu
sudo -u deploy-runner git clone https://github.com/xarrondobirt/aurkitu-back.git aurkitu
```
```bash

/opt/aurkitu/
├─- .env                  # (Fichero de secretos - NO versionado)
├── compose.yaml          # (Orquestación Docker)
├── db/                   # (Scripts de inicialización de la base de datos)
└── nginx/                # (Configuración del Proxy)
```
#### Inicialización de la base de datos
**NOTA:** es necesario crear manualmente la carpeta `db` y colocar en ella el `schema.sql` obtenido desde el **repositorio de backend**: [aurkitu-back](https://github.com/xarrondobirt/aurkitu-deploy)
```bash
cd /opt/aurkitu 
sudo -u deploy-runner mkdir db
```
#### Variables de entorno .env
Se debe crear manualmente el fichero `.env`a partir de la plantilla `.env.example`del repositorio.
```bash
cd /opt/aurkitu
sudo -u deploy-runner cp .env.example .env
sudo -u deploy-runner nano .env
```
```
### 4.2. Datos y Runners (`/d
ocker_data`)

Esta es la estructura real en producción. Observar los permisos especiales para la carpeta `mobile`, ya que es el runner quien deposita allí los APKs, mientras que `fotos` las gestiona el backend (usuario birt).

```bash 
/docker_data/
├── actions-runner/          # [deploy-runner:deploy-runner] Agente CI/CD Backend
├── actions-runner-front/    # [deploy-runner:deploy-runner] Agente CI/CD Frontend
├── docker-storage/          # [root:root] Imágenes y contenedores Docker (data-root)
├── postgresql/              # [deploy-runner:systemd-journal] Persistencia BBDD (UID 999)
└── uploads/                 # [birt:birt] (SetGID activado drwxr-sr-x)
    ├── docs/                # [birt:birt] Documentos subidos por usuarios
    ├── fotos/               # [birt:birt] Imágenes subidas por usuarios
    └── mobile/              # [deploy-runner:deploy-runner] APKs generadas por CI/CD
```

### 4.3. Comandos para replicar permisos

Si se migra el servidor, ejecutar:
```bash 
# Directorio base
sudo mkdir -p /docker_data/postgresql /docker_data/uploads

# Base de datos (UID 999)
sudo chown -R 999:999 /docker_data/postgresql
sudo chmod 700 /docker_data/postgresql

# Uploads generales (birt)
sudo chown -R 1000:1000 /docker_data/uploads
sudo chmod 2775 /docker_data/uploads # SetGID para herencia de grupo

# Carpeta Mobile (Excepción: Propiedad del Runner)
sudo mkdir -p /docker_data/uploads/mobile
sudo chown -R 999:999 /docker_data/uploads/mobile
```

---

## Inicialización de certificados SSL

Tal y como está definido el fichero de configuración de nginx `default.conf.template` el servidor no arrancará porque no existen los certificados.

Para la primera instalación:

1. **Modo HTTP**: En `nginx/default.conf.template`, comentar las líneas SSL y el bloque `listen 443`.
2. **Arrancar**: `sudo -u deploy-runner docker compose up -d nginx certbot`
3. **Generar los certificados:**
    ```bash
    sudo -u deploy-runner docker compose exec certbot certbot certonly --webroot -w /var/www/certbot --agree-tos --no-eff-email --email usuario@birt.eus -d dominio-proyecto.birt.eus
   ```    
4. **Activar HTTPS**: Descomentar la configuración SSL en el template de Nginx porque los certificados ya estarían disponibles.
5. **Reiniciar** servicio: `sudo -u deploy-runner docker compose restart nginx`

---

## Gestión y mantenimiento

### Arrancar y parar escenario
```bash
sudo -u deploy-runner sh -c "cd /opt/aurkitu && docker compose up -d"
sudo -u deploy-runner sh -c "cd /opt/aurkitu && docker compose down"
```
### Reiniciar servicio específico:
```bash
sudo -u deploy-runner docker compose restart backend
```
### Consultar logs
```bash
cd /opt/aurkitu
# Ver los logs de todos los contenedores en tiempo real
sudo -u deploy-runner docker compose logs -f 
# Ver los log de un contenedor específico en tiempo real
sudo -u deploy-runner docker compose logs -f backend"
```
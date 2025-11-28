Aurkitu - Infraestructura de Despliegue

Este repositorio contiene la orquestación de contenedores (Docker Compose) y la configuración del Proxy Inverso (Nginx) para el proyecto Aurkitu.

1. Prerrequisitos del Servidor

Antes de realizar el despliegue, el servidor debe cumplir con los siguientes requisitos de usuarios y estructura de directorios.

Usuarios de Sistema

Usuario deploy-runner (UID 999)

Este usuario ejecuta los procesos de Docker y la Base de Datos.

Debe pertenecer al grupo docker.

Shell: /bin/false (sin acceso interactivo directo).

Home: /home/deploy-runner.

Usuario birt (UID 1000)

Usuario estándar del sistema.

Propietario de los ficheros subidos por la aplicación (uploads).

GitHub Runner

Debe existir un agente Self-Hosted Runner instalado y configurado para automatizar el CD.

Ubicación: /opt/actions-runner

Propietario: deploy-runner:deploy-runner

Persistencia de Datos (Bind Mounts)

La aplicación requiere una estructura de carpetas específica en el host para persistir datos fuera de los contenedores.

# Estructura requerida en /docker_data
/docker_data/
├── postgresql/        # UID 999 (deploy-runner)
└── uploads/           # UID 1000 (birt) - Backend escribe aquí
├── fotos/
└── docs/


Comandos para generar la estructura:

sudo mkdir -p /docker_data/postgresql
sudo mkdir -p /docker_data/uploads/fotos
sudo mkdir -p /docker_data/uploads/docs

# Permisos Base de Datos (Postgres requiere 700 y su propio UID)
sudo chown -R 999:999 /docker_data/postgresql
sudo chmod 700 /docker_data/postgresql

# Permisos Uploads (Backend corre como UID 1000)
sudo chown -R 1000:1000 /docker_data/uploads


2. Instalación y Configuración (Provisioning)

Sigue estos pasos para preparar el entorno en un servidor nuevo.

1. Clonar el Repositorio

La aplicación se aloja en /opt/aurkitu.

cd /opt
sudo mkdir aurkitu
sudo chown deploy-runner:deploy-runner aurkitu

# Clonar repo (hacerlo como deploy-runner o corregir permisos después)
sudo -u deploy-runner git clone <URL_REPO_DEPLOY> aurkitu


2. Variables de Entorno

Copia la plantilla y establece los secretos de producción. Nunca subir el .env real al repositorio.

cd /opt/aurkitu
sudo -u deploy-runner cp .env.example .env
sudo -u deploy-runner nano .env


Nota: Asegúrate de configurar UPLOADS_ROOT_PATH, UPLOADS_FOTO_DIR y UPLOADS_DOC_DIR coincidiendo con la estructura creada en /docker_data.

3. Base de Datos Inicial

El archivo db/schema.sql no se versiona en este repositorio para evitar duplicidades. Debe copiarse manualmente desde el repositorio de backend solo para la instalación inicial.

# Copiar schema.sql del repo aurkitu-back a:
/opt/aurkitu/db/schema.sql


Este script solo se ejecutará si la carpeta /docker_data/postgresql está vacía.

3. Estructura de Ficheros Esperada

Tras la instalación, el directorio /opt/aurkitu debería verse así:

/opt/aurkitu/
├── .env                  (Fichero de secretos - NO versionado)
├── .env.example          (Plantilla)
├── compose.yaml          (Orquestación Docker)
├── db/
│   └── schema.sql        (Copiado manualmente del Back - NO versionado)
├── nginx/
│   └── default.conf.template
└── README.md


Propietario recursivo: deploy-runner:deploy-runner (UID 999).

4. Gestión de Certificados SSL (Primer Arranque)

Nginx fallará al iniciar si la configuración SSL está activa pero los certificados no existen. Sigue este procedimiento para la primera vez:

Desactivar SSL temporalmente:
Edita nginx/default.conf.template. Comenta las líneas relacionadas con listen 443 y ssl_certificate. Deja solo activo el puerto 80.

Arrancar servicios:

sudo -u deploy-runner docker compose up -d nginx certbot


Solicitar Certificado:
Ejecuta Certbot a través del contenedor.

sudo -u deploy-runner sh -c "cd /opt/aurkitu && docker compose exec certbot sh -c 'certbot certonly --webroot -w /var/www/certbot --agree-tos --no-eff-email --email TU_EMAIL@birt.eus -d TU_DOMINIO.birt.eus'"


Activar SSL:
Edita de nuevo nginx/default.conf.template, descomenta la configuración SSL y el puerto 443.

Recargar Nginx:

sudo -u deploy-runner sh -c "cd /opt/aurkitu && docker compose exec nginx nginx -s reload"

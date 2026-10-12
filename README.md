# Conceptos Fundamentales de Docker

## Comandos de Docker

- `docker run`: Sirve para crear y arrancar un contenedor nuevo partiendo de una imágen.
- `-d`: Arranca el contenedor en segundo plano, liberando la terminal para poder seguir ejecutando comandos
- `-t`: Abre una pseudo-terminal. En los apuntes se menciona que si quitamos el -d y dejamos -t vemos los logs en directo durante el arranque para depurar errores.
- `-v <host>:<contenedor>`: Mapea o monta carpetas de nuestra máquina dentro del contenedor, permitiendo persistencia de ficheros o meter codigo fuente.
- `-p <puerto_host>:<puerto_contenedor>`: Redirige puertos de la máquina anfitriona hacia dentro del contenedor. Con -p 8069:8069 podemos abrir Odoo directamente desde nuestro navegador en localhost:8069
- `-e`: Define variables de entorno para configurar el software
- `--name`: Le da un nombre reconocible al contenedor en vez de asignarle un hash aleatorio.
- `--link`: forma antigua de conectar contenedores en red para que se vean por nombre de host. Actualmente Docker Compose lo hace de forma automatica mediante redes internas.
- `--user`: Fuerza a que los procesos internos se ejecuten bajo un usuario concreto, por ejemplo root.
- `docker compose up -d`: Lee el fichero de configuración (compose.yml) y levanta todos los servicios, redes y volumenes a la vez en segundo plano.
- `docker compose down`: Detiene y elimina los contenedores y redes creados por el compose, manteniendo los volumenes para no perder datos
- `docker pull`: Descarga una imagen desde el docker hub a nuestra maquina local sin ejecutarla inmediatamente.

---

## ¿Qué quieren decir los bloques de compose.yml (docker-compose.yml)?

El archivo compose.yml permite definir toda una infraestructura multi-contenedor de forma declarativa. Sus bloques principales son:

- `services`: Es el bloque raíz donde declaramos cada componente o contenedor de la aplicación (en nuestro caso, el servicio web y la base de datos db)
- `image`: Especifica la imágen base a descargar de Docker Hub, indicando la versión (odoo:19, postgres:18).
- `depends_on`: Marca el orden de inicio de los servicios. Se asegura de que la base de datos arranque antes de que Odoo intente conectarse a ella
- `ports`: Define el mapeo de puertos entre el host y el contenedor ("8069:8069").
- `volumes`: A nivel de servicio asocia rutas locales (bind mounts) o volumenes para persistir datos como addons, filestore o las tablas de PostgreSQL. A nivel raiz, define volumenes con nombre independientes gestionados directamente por el motor de Docker.
- `environment`: Pasa las variables necesarias para la conexión interna entre Odoo y la base de datos (como HOST=db, USER=odoo).
- `command`: Sobrescribe el comando por defecto de la imagen, utilizado en desarrollo para pasar parametros como --dev=all.
- `networks`: Configura redes virtuales internas para aislar la comunicacion entre contenedores.

---

## Conceptos Clave: Imagen, Contenedor y Volumen

- **¿Qué es una imagen?**: Una imagen es una plantilla inmutable de solo lectura que contiene el sistema operativo base, las librerías, dependencias y el código de la aplicación necesarios para funcionar. Se construyen por capas y sirven como molde para crear contenedores.
- **¿Qué es un contenedor?**: Es una instancia en ejecución de una imagen. Funciona como una "cajita" aislada del resto del sistema operativo que comparte el kernel de la maquina anfitriona. Los contenedores son efímeros: si se borran sin haber guardado su estado en un volumen, todos los cambios internos se pierden por completo. Por esto siempre deberias usar los volúmenes si necesitas la persistencia en proyecto
- **¿Qué es un volumen?**: Es el mecanismo que proporciona Docker para guardar datos de manera persistente fuera del ciclo de vida del contenedor. Permite que, aunque destruyamos o actualicemos los contenedores de Odoo o PostgreSQL, la base de datos y los ficheros subidos por los usuarios se quedan intactos y no se borren.


---

# Alternativas de Eficiencia

## Eficiencia en la utilización de recursos:

- **Límites de CPU y RAM**: Por defecto un contenedor puede consumir toda la memoria del host. Mediante deploy.resources.limits en Compose podemos definir topes para evitar caídas del sistema
- **Optimización de imágenes y caché**: Usar versiones ligeras o imagenes oficiales limpiando paquetes temporales y capas innecesarias para ahorrar espacio en disco.
- **Workers**: En entornos con pocos recursos, ajustar la configuracion de procesos concurrentes de Odoo para evitar que se sature la CPU de la maquina virtual o del servidor.

## Gestión segura de variables de entorno y secretos:

- **Archivos .env**: En lugar de dejar usuarios y contraseñas expuestos directamente en el codigo de compose.yml, se deben extraer a un archivo .env local. Este archivo debe añadirse siempre al .gitignore para no subir credenciales sensibles al repositorio de Git
- **Docker Secrets**: Es la alternativa mas robusta. Permite inyectar contraseñas mediante archivos montados en memoria en lugar de variables de entorno visibles mediante comandos como docker inspect.


---

# Configuración de Odoo

## Comparativa: odoo.conf vs. variables en compose.yml

### ¿Qué diferencias hay entre configurar en un sitio u otro?
- **compose.yml (environment)**: Se usa para la infraestructura y conectar contenedores entre si (definir el host de la base de datos HOST=db, usuario, contraseña y puertos).
- **odoo.conf**: Es el archivo interno de Odoo para configurar la aplicacion en sí. Aquí defines las rutas de los módulos (addons_path), la contraseña maestra, los procesos (workers) o filtros de base de datos.

### ¿Qué opción es más segura?
Es más seguro el odoo.conf. Si pones contraseñas en las variables del compose.yml cualquiera con acceso a Docker puede verlas en texto plano haciendo un simple docker inspect al contenedor. Con odoo.conf, le puedes dar permisos restringidos en Linux (chmod 600) y montarlo como solo lectura en Docker con :ro (que significa read only), evitando que se modifique o se filtre facilmente.

### ¿Y para desarrollar en local, qué es más práctico?
Es más práctico usar odoo.conf. Si estás programando y metes módulos nuevos, solo tienes que cambiar la linea de addons_path en el archivo desde VSCode sin tener que editar el Compose ni reiniciar toda la infraestructura. El Compose se reserva mas para la conexion de red y levantar el servicio.


---

# Producción vs. Desarrollo y cómo configurarlo en Compose

La forma más limpia sin duplicar código es usar ficheros override separando la configuración de ambos entornos:

- **`compose.yml` (Base común)**: Contiene lo compartido por ambos entornos, como las imagenes de Odoo y Postgres, la red interna y los volumenes de persistencia de base de datos
- **`compose.override.yml` (Desarrollo)**: Docker Compose lo lee y aplica solo al hacer `docker compose up -d`. Aquí metemos el mapeo de puertos directos ("8069:8069"), las carpetas locales montadas (addons, filestore, sessions), `workers = 0` para poder depurar sin bloqueos y el comando `--dev=all` para que Odoo recargue los cambios de código Python y vistas XML automáticamente sin reiniciar el contenedor a cada rato.
- **`compose.prod.yml` (Producción)**: Se usa para el despliegue real en el servidor. Desactiva el modo dev y los datos demo, activa multiples procesos (workers) segun la CPU del servidor, añade politica de reinicio `restart: always` y limites de recursos (CPU/RAM). Ademas no expone el puerto 8069 directo a internet sino que se protege detras de un proxy inverso (Nginx) con SSL.


---

# Gestión y seguridad de directorios de Odoo en el repositorio

A la hora de gestionar el proyecto con Git, debemos separar el código fuente de los datos dinámicos generados por la aplicación:

- **addons (Módulos personalizados) SÍ se sube al repositorio**:
  - **Por qué**: Aquí se guarda el código fuente de los módulos y desarrollos propios que creamos o adaptamos. Debe estar versionado para poder colaborar, revisar cambios y desplegarlo en otros entornos o en producción.

- **filestore (/var/lib/odoo/filestore) NO se sube al repositorio**:
  - **Por qué**: Contiene los archivos binarios que suben los usuarios. Subirlo llenaria el repositorio de archivos pesados innecesariamente y ademas es un fallo grave de privacidad y seguridad, ya que expondria datos sensibles y de clientes protegidos por normativas como el RGPD o la LOPDGDD. Estos datos se gestionan mediante copias de seguridad de la máquina o volumenes.

- **sessions (/var/lib/odoo/sessions) NO se sube al repositorio**:
  - **Por qué**: Guarda los ficheros temporales con las sesiones y tokens de autenticación de los usuarios que inician sesión. Si se sube a Git, cualquiera con acceso al repositorio podría robar esas sesiones y entrar como administrador sin contraseña. Son archivos efímeros que pertenecen al contenedor, no al código.

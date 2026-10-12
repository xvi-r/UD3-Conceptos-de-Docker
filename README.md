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

# Manual de instalación

Para poder ejecutar la aplicación en un entorno local y acceder a toda la funcionalidad que ofrece, son necesarios realizar una serie de pasos. Su dispositivo debe contar con los siguientes requisitos:

- Sistema operativo: Ubuntu 20.04, OpenSuse Tumbleweed.
- Dependencias:
  - Python 3.11
  - Node v20.15.1
  - Docker 26.1.5-ce
  - Kubectl v1.31.0
  - Minikube v1.34.0

A continuación se presenta la guía de instalación para el entorno de desarrollo, y la previsualización del entorno de producción.

Como paso preliminar, deben descargarse las imágenes de Docker que se utilizarán con Kubernetes. Para ello, es preciso introducir los siguientes comandos.

```
docker image pull goncammej/vividus_back:0.0.1
```
```
docker image pull goncammej/vividus_front:0.0.1
```
```
docker image pull goncammej/inference_process:0.0.6 docker image pull goncammej/features_process:0.0.1
```


Como paso preliminar, deben crearse las imágenes de Docker que se utilizarán con Kubernetes. Para ello, es preciso realizar los siguientes pasos:

- **Extractor de features:** Para crear esta imagen, se debe acceder al directorio **"extractor"** y ejecutar el siguiente comando

```
docker build
```


Posteriormente, se deben guardar todas las imágenes como archivos 'tar'. A continuación se muestra un ejemplo para la primera imagen.

```
docker image save -o vividus-back.img goncammej/vividus_back:0.0.1
```


## Entorno de desarrollo

Para la instalación de la aplicación y su despliegue local en un entorno de desarrollo, se deben seguir los siguientes pasos:

1. Una vez se tiene acceso al código, se debe acceder al directorio **'backend'**, copiar el archivo *`.env.example`* y renombrarlo a *`.env`*. El fichero debe contener lo siguiente:

   ![09_env_file](https://github.com/user-attachments/assets/ca619440-7b56-4269-bf83-f525a00f6c03)

   En entornos de producción, sería indispensable cambiar los valores del par de claves *JWT_SECRET_KEY* y *JWT_REFRESH_SECRET_KEY*, así como las credenciales del superusuario (las entradas con el prefijo *SUPER_USER*), y los parámetros de conexión con la base de datos y el bróker de tareas (las entradas con prefijo *PSQL* y *REDIS* respectivamente). Sin embargo, para este ejemplo, se recomienda mantener los valores por defecto, pues cambiarlos podría dificultar el seguimiento del manual.

2. Una vez realizado lo anterior, se debe instalar la herramienta Pipenv [^1] para poder descargar las dependencias necesarias. Basta con ejecutar el comando:

```
pip install pipenv
```


Se recomienda encarecidamente crear un entorno virtual para la gestión de las dependencias del proyecto. Para ello, deben ejecutarse los comandos:

```
python -m venv .venv
```

```
source .venv/bin/activate
```


Este último comando se deberá ejecutar cada vez que se quiera lanzar el *backend* de la aplicación.

3. Ahora, se deberá instalar las dependencias del proyecto. Para ello, sólo será necesario ejecutar el comando:

```
pipenv install
```


Este comando deberá ejecutarse dentro del directorio **'backend'**, donde se encuentran los archivos *Pipfile* y *Pipfile.lock*.

1. Una vez instaladas las dependencias, se debe acceder al directorio **'devcontainers'** dentro de la carpeta **'backend'**, y ejecutar el siguiente comando:
```
docker compose up
```
Se debe mantener el proceso activo hasta finalizar la ejecución de la aplicación. Este comando lanzará dos contenedores de Docker, uno para la base de datos, y otro para el servicio de Redis (en caso de haber cambiado los valores de los parámetros de conexión de estos servicios en el paso 2, se debe hacer lo propio en el fichero **'docker-compose.yaml'**).

2. Lanzada la base de datos, se debe proceder a realizar la migración para crear las tablas. Para ello, se deben ejecutar los comandos:
```
alembic revision --autogenerate
```
```
alembic upgrade head
```

3. Realizadas las migraciones, se procede a crear el superusuario. Para ello, se debe ejecutar el comando:
```
python -m src.initial_data
```


Que creará un usuario administrador con las credenciales indicadas en el fichero *'.env'*.

4. Ahora, se lanzan los *workers* de Python RQ. A efectos de esta prueba, se ejecutará una única vez el siguiente comando:
```
rq worker
```

Se debe mantener el proceso activo hasta finalizar la ejecución de la aplicación.

5. Realizados los pasos anteriores, se debe volver al directorio raíz del proyecto, y acceder al directorio **'kubernetes'**. Posteriormente, se ejecutará el siguiente comando para iniciar Minikube si no se ha iniciado todavía:

```
minikube start
```

6. Una vez iniciado, es preciso cargar las imágenes de los procesos que se ejecutarán dentro del clúster. Se deben ejecutar los siguientes comandos:
```
minikube load features_process.img
```
```
minikube load inference_process.img
```

Los comandos anteriores se deben ejecutar en el directorio en el que se guardaron las imágenes.

7. Completado lo anterior, se montarán los directorios necesarios para el funcionamiento del sistema. Se deben crear tres directorios en el equipo local, dentro de la carpeta **'backend'**:
- Un directorio **'infer_dir/'** que se montará con el comando:

  ```
  minikube mount {ruta del proyecto backend}/infer_dir:/infer_dir
  ```

- Un directorio **'tmp_dir/'** que se montará con el comando:

  ```
  minikube mount {ruta del proyecto backend}/tmp_dir:/tmp_dir
  ```

- Un directorio **'datasets/'** que se montará con el comando:

  ```
  minikube mount {ruta del proyecto backend}/datasets:/datasets
  ```

Donde *{ruta del proyecto backend}* es la ruta del directorio **'backend'** en el equipo local.

8. Ahora, se deben ejecutar los siguientes comandos para configurar el administrador de workflows.
  ```
minikube kubectl -- create namespace argo
  ```
  ```
minikube kubectl -- apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.5.7/install.yaml
  ```


9. Por último, se debe ejecutar el comando
```
minikube kubectl -- apply -k overlays/development
  ```


Para concluir con la configuración de Kubernetes.

10. Completados los pasos anteriores, se debe volver al directorio **'backend'** y ejecutar el comando
 ```
python -m src.main
  ```


Este comando, como se indicó en el paso 3, debe ejecutarse dentro del entorno virtual.

12. Para finalizar la configuración, se debe acceder al directorio **'frontend'** en la raíz del proyecto, e instalar sus dependencias con el siguiente comando:
  ```
npm install
  ```

13. Una vez completada la instalación, se debe ejecutar el comando:
  ```
npm run dev
  ```

Que devolverá la URL para acceder al servicio.

## Previsualización del entorno de producción

Para simular el entorno de producción propuesto en un entorno local, se propone la siguiente guía. Para seguirla, es preciso contar con los siguientes requisitos:

- Una base de datos PostgreSQL alojada en la nube.
- Un dominio que apunte a la dirección IP externa del clúster Minikube o modificar el fichero de configuración **/etc/hosts** para asociar dicha IP al nombre de dominio elegido. Para obtener la IP, es necesario ejecutar el siguiente comando una vez iniciado el clúster de Minikube:
 ```
minikube ip
 ```


Los pasos a seguir son los siguientes:

1. Se debe acceder al directorio **'kubernetes'** y ejecutar el siguiente comando para iniciar el clúster si no se ha lanzado todavía:
 ```
minikube start
 ```


2. Una vez iniciado, se deben descargar los *addons* del *ingress*. Para ello, se deben ejecutar los siguientes comandos:
 ```
minikube addons enable ingress
 ```
 ```
minikube addons enable ingress-dns
 ```
14. Realizadas las descargas, se configuran los secretos de la aplicación. A continuación se muestran la secuencia de comandos a ejecutar. En primer lugar, se crea el *namespace* para la aplicación:
 ```
minikube kubectl -- create namespace app
 ```


Posteriormente, se introduce la contraseña del servicio de Redis. El siguiente comando incluye un valor de prueba:
 ```
minikube kubectl -- create secret generic redis-password
--from-literal=password=eYVX7EwVmmxKPCDmwMtyKVge8oLd2t81 -n app
 ```


En último lugar, se introducen las claves JWT. El siguiente comando incluye dos claves de prueba:
 ```
minikube kubectl -- create secret generic jwt-secrets
--from-literal=JWT_SECRET_KEY=a7e76d632967f7417d1794d8b78cb5e816060fa740913c3b09529b2740dd4e0f
--from-literal=JWT_REFRESH_SECRET_KEY=7d0e556702dadd613c79f4b61de92c7bca7bb48d98b14eb1326f3b39352cee5c -n app
 ```


15. Una vez terminado el paso anterior, se cargan las imágenes previamente descargadas:
 ```
minikube image load vividus_back
 ```
 ```
minikube image load vividus_front
 ```
 ```
minikube image load features_process
 ```
 ```
minikube image load inference_process
 ```

Los comandos anteriores deben ejecutarse en el mismo directorio en el que se guardaron las imágenes.

16. Cargadas las imágenes, se configura el entorno del administrador de *workflows*.
 ```
minikube kubectl -- create namespace argo
 ```
 ```
minikube kubectl -- apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.5.7/install.yaml
 ```

17. Configurado lo anterior, se crean y se montan los volúmenes necesarios para el funcionamiento de la aplicación.

Se debe crear un volumen para el almacenamiento del código fuente de los modelos:
 ```
minikube mount {ruta del directorio de modelos}:/infer_models
 ```

Se debe crear un volumen para los archivos temporales:

 ```
minikube mount {ruta del directorio de archivos temporales}:/tmp_inference
 ```

18. Se debe crear un volumen para los datasets:
 ```
minikube mount {ruta del directorio de datasets}:/datasets
 ```


Las instrucciones anteriores deben permanecer activas hasta finalizar la ejecución de la aplicación.

19. Ahora, se debe introducir el dominio como valor de las variables **SITE** y **PROD_URL** del archivo **base/production/configmap.yaml**, y en las instrucciones **host** del archivo **base/production/ingress.yaml**.

20. Por último, se cargan los despliegues de *backend* y *frontend*, así como el *ingress* y las variables de entorno:
 ```
minikube kubectl -- apply -k overlays/production
 ```

Al seguir los pasos anteriores, la aplicación debe ejecutarse al introducir el dominio en un navegador.

Para subir datasets a la plataforma, se deben copiar los contenidos del directorio **'datasets'** de la ruta principal del proyecto, en el directorio local asociado al volumen con el mismo nombre. Una vez hecho eso, se debe acceder a la base de datos, a través de una aplicación de administración de bases de datos, y añadir las entradas correspondientes a cada uno de los datasets incorporados. La única condición para que el sistema los detecte, es que el campo *accessor* en la base de datos, coincida con el nombre de la carpeta asociada a ese *dataset*.

Para subir modelos a la plataforma, es necesario que el código fuente de cada uno de ellos esté alojado en un repositorio público de GitHub. El punto de entrada de los modelos debe ser un archivo llamado **'infer.py'**, que cuente con los siguientes parámetros de ejecución:

- **--rgb-list**, que proporciona la ruta del fichero de texto que contiene las rutas de las características de RGB.
- **--audio-list**, que proporciona la ruta del fichero de texto que contiene las rutas de las características de audio.
- **--output-path**, que proporciona la ruta en la que se debe escribir un archivo **'results.npy'** con el *array* ordenado de las predicciones realizadas.

## Vídeo con la demo de VIVIDUS
[Demo VIVIDUS](https://uses0-my.sharepoint.com/:v:/g/personal/alemeddur_alum_us_es/EfGTRFuXGYlBv5E5C3WYndMBP6F0trxZ3Sr6XTUcm9BErA?e=uKApfx&nav=eyJyZWZlcnJhbEluZm8iOnsicmVmZXJyYWxBcHAiOiJTdHJlYW1XZWJBcHAiLCJyZWZlcnJhbFZpZXciOiJTaGFyZURpYWxvZy1MaW5rIiwicmVmZXJyYWxBcHBQbGF0Zm9ybSI6IldlYiIsInJlZmVycmFsTW9kZSI6InZpZXcifX0%3D)

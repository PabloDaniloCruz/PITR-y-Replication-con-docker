# PITR (point-in-time-recovery)


## Introducción
PITR constituye una estrategia robusta para recuperar una base de datos hasta un punto temporal específico, garantizando la consistencia de los datos, a diferencia de otras herramientas como las copias de seguridad ('dumps').


Para su implementación, se combina una copia de seguridad a nivel del sistema de archivos con una copia de los archivos WAL (Write-Ahead Logging). En caso de ser necesaria una recuperación, se procede a restaurar la copia de seguridad del sistema de archivos y, posteriormente, se reproducen los archivos WAL copiados para que el sistema retorne a su estado deseado.


Anteriormente se hizo mención a los archivos WAL, los cuales resultan fundamentales para la implementación del PITR. En su operación habitual, los archivos WAL se almacenan en la carpeta pg_wal, pero este almacenamiento es temporal. Para lograr la persistencia de estos archivos, se requiere realizar modificaciones que permitan su almacenamiento en un directorio externo. Para ello, se deben ajustar las configuraciones de almacenamiento de los WAL para que la base de datos réplica pueda acceder a ellos. Adicionalmente, se debe modificar la configuración de la base de datos principal para que copie los WAL a la carpeta externa y, finalmente, modificar la configuración de la base de datos réplica para que, en caso de requerir los WAL, los busque en el directorio externo. Estos y otros detalles serán explicados con mayor profundidad a continuación.


## Preparación
Creamos dos volúmenes administrados por Docker para almacenar los datos de la base de datos en forma permanente.
```
docker volume create pgdata wals
```
## Arquitectura del Proyecto
La estructura elegida para el proyecto es la siguiente:
```
    srv
      |-pitr-master
            |- config
                |- postgresql.conf
      |-pitr-copy
            |- config
                |- postgresql.conf
```
Asignar los permisos al usuario postgres:
```
sudo chown -R 999:999 /srv
```
> En algunos casos puede no ser necesario dar permisos (Windows).


## Write-Ahead Logging (WAL)
Los wals son el método estándar de PostgreSQL para garantizar la integridad de los datos. Las alteraciones en la base de datos no se escriben directamente en disco si no que antes son escritas en un registro de escritura anticipada en donde cuando estos estén seguros recién se almacenarán en disco. Esto permite a la base de datos recuperarse ante un fallo del sistema ya que va a poder recuperarse hasta el último instante confirmado.


Por defecto, los archivos WAL (segmentos de 16MB) se reciclan o eliminan una vez que ya no son necesarios para la consistencia inmediata. Sin embargo, para estrategias de recuperación avanzadas, configuramos el parámetro archive_mode = on. Esto instruye a PostgreSQL a copiar cada segmento WAL completado a una ubicación externa segura  antes de reciclarlo. Esto crea un historial continuo de todas las transacciones ocurridas en  la base de datos de forma persistente.

**Configurar el archivado continuo de wals**

En el archivo `postgresql.conf` del nodo principal debemos definir las siguientes directivas:

```
# Define el directorio para el almacenado de la BD
data_directory = '/data'

# Habilita nivel de réplica y archivado de wals
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /mnt/server/wals/%f && cp %p /mnt/server/wals/%f'
```

En el archivo `postgresql.conf` del nodo copia debemos definir las siguientes directivas:

```
# Define el directorio para el almacenado de la BD
data_directory = '/data'

# Habilita nivel de réplica y archivado de wals
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /mnt/server/wals/%f && cp %p /mnt/server/wals/%f'
restore_command = 'cp /mnt/server/wals/%f %p'
```

## Ejecución


Creamos el contenedor nodo master con tres volúmenes los mismos enlazan carpetas del contenedor con carpetas locales almacenadas fuera del contenedor con esto se logra **persistencia de los datos** en dichas carpetas.


**Para crear el contenedor es importante ubicarnos en el directorio `/pitr-master`**


```
docker run -it -d --name postgres-pitr \
-e POSTGRES_USER=postgres \
-e POSTGRES_PASSWORD=postgres \
-e POSTGRES_DB=dellstore \
-v ${PWD}/config:/config \
-v pgdata:/data:rw \
-v wals:/mnt/server/wals:rw \
-p 5500:5432 \
postgres:17 -c 'config_file=/config/postgresql.conf'
```
> Se puede usar el ${PWD}, aunque también se puede usar ./ para hacer referencia a la carpeta sobre la que estamos parados.


## Conexión a la base de datos
Nos conectamos a la base de datos dellstore y cargamos su dump.
```
psql -U postgres -p 5500 -h localhost -d dellstore < dellstore_dump.sql
```

## Conexión a la terminal del contenedor:
```
docker exec -it postgres-pitr bash
```
#### Cambiamos los permisos del /mnt/server/wals
```
chown -R 999:999 /mnt
```


> De no hacerse este cambio, los wals no se guardan.


#### Guardamos el full backup en el volumen pgdata
Utilizamos la utilidad `pg_basebackup` que nos permite automatizar todo el proceso de backup.
```
pg_basebackup -h localhost -p 5432 -U postgres -D /data/ -Fp -Xs -P
```
*Explicación de directivas de `pg_basebackup`*
* `-Fp` el formato de la copia es a nivel de filesystem (copia tal cual los archivos).
* `-D` directorio donde se guardará copia.
* `-Xs` permite la escritura de wals por streaming.
* `-P` muestra reporte del progreso


> Seguimos dentro de la consola del contenedor.


#### Creamos la bandera dentro de la carpeta /data
```
touch recovery.signal
```
> Cuando el nodo se inicie, este lo hará en modo recuperación, solo la primera que se inicia.


## Datos de prueba
_Insertamos datos en la tabla creada anteriormente, para ello nos conectamos a la base de datos dellstore._
```
INSERT INTO id_prueba(id,texto) values (1,'Datos Valiosos');
```
#### Forzamos que se guarde el archivo wal actual
```
SELECT pg_switch_wal();
```
## Restauración total
Se necesita comentar la siguiente línea dentro del archivo postgresql.conf:
```
#time_target_recovery = 'formatofecha'
```


## Restauración a un punto en el tiempo
Necesitamos saber el tiempo actual o el tiempo al que queremos volver
```
    SELECT now();


    Salida:
    2025-11-16 21:27:01.444901+00
```
Modificar el postgresql.conf del pitr-copy para que se restaure a una hora dada.    
Se necesita descomentar la siguiente línea dentro del archivo postgresql.conf:
```
time_target_recovery = '2025-11-16 21:27:01.444901+00'
```
Podemos agregar más datos en la tabla, para ello nos conectamos a la base de datos dellstore:
```
INSERT INTO id_prueba(id,texto) values (2,'Datos Inútiles');
```
> Estos datos no serán restaurados
## Levantar el servidor copia
El nodo copia se va a levantar en modo recuperación utilizando como checkpoint el full backup y ejecutando sobre este los wals hasta un determinado punto en el tiempo o en su totalidad.


**Nos posicionamos sobre la carpeta pitr-copy y ejecutamos:**
```
docker run -it -d --name postgres-pitr-copy \
-e POSTGRES_USER=postgres \
-e POSTGRES_PASSWORD=postgres \
-e POSTGRES_DB=dellstore \
-e PGDATA="/data" \
-v ${PWD}/config:/config \
-v wals:/mnt/server/wals:rw \
-v pgdata:/data:rw \
-p 5501:5432 \
postgres:17 -c 'config_file=/config/postgresql.conf'
```


# Streaming Replication
## Introducción
En el entorno de bases de datos, streaming replication es una técnica que se utiliza para garantizar que la información esté segura y disponible todo el tiempo utilizando una replicación en streaming la cual mantiene los datos sincronizados casi en tiempo real. Esto lo consigue enviando transacciones desde una base de datos principal hacia una copia que replicará estos cambios en su base de datos tan pronto como sea posible.


Para lograr streaming replication se cuenta con un nodo que llamamos esclavo o réplica, que combina una copia de seguridad a nivel de sistema de archivos con una copia de los archivos WAL (Write-Ahead Logging). Similar a lo que obtuvimos con PITR, solo que además, la base de datos principal va a enviar transacciones (WALS) a través de procesos wal senders los cuales serán recibidos por la base de datos réplica por un proceso wal receiver y luego serán escritos en la misma.


## Crear la red
Creamos una red de Docker que nos será útil para que el contenedor del nodo master (maestro), se pueda comunicar con el contenedor del nodo slave (esclavo/standby).
```
docker network create postgres
```
## Arquitectura del Proyecto
Para realizar el proyecto definiremos la siguiente estructura de archivos:
```
    master
        |- config
            |- pg_hba.conf
            |- postgresql.conf
        |- pgdata (no hace falta crearla)
        |- wals (no hace falta crearla)
    slave
        |- config
            |- pg_hba.conf
            |- postgresql.conf
        |- pgdata (no hace falta crearla)
        |- wals (no hace falta crearla)
           
```


## Archivos de configuración
Necesitamos definir archivos de configuración para que tanto el nodo master como el nodo slave se comporten como queremos.


Como se ve en la estructura anterior debemos crear dos archivos de configuración por cada nodo.


El archivo `postgresql.conf` del nodo master como del nodo slave tendrá las siguientes directivas:
```
# Define el directorio para el almacenado de la BD como el archivo de permisos de acceso
data_directory = '/data'
hba_file = '/config/pg_hba.conf'


# Habilita nivel de réplica y archivado de wals
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /mnt/server/wals/%f && cp %p /mnt/server/wals/%f'
max_wal_senders = 3
```
Por otra parte el archivo `pg_hba.conf` del nodo master como del nodo slave tendrá las siguientes directivas:
```
# Define el usuario para la replicación
host     replication     replicationUser         0.0.0.0/0        md5
```


## Nodo master (inicialización)


Creamos el contenedor nodo master con tres volúmenes bind mounts los mismos enlazan carpetas del contenedor con carpetas locales almacenadas fuera del contenedor con esto se logra **persistencia de los datos** en dichas carpetas.


**Para crear el contenedor es importante ubicarnos en el directorio `/master`**
```
docker run -it -d --name postgres-master `
--net postgres `
-e POSTGRES_USER=postgres `
-e POSTGRES_PASSWORD=postgres `
-e POSTGRES_DB=dellstore `
-e PGDATA="/data" `
-v ${PWD}/pgdata:/data `
-v ${PWD}/config:/config `
-v ${PWD}/wals:/mnt/server/wals `
-p 5400:5432 `
postgres:17 -c 'config_file=/config/postgresql.conf'
```
En el comando anterior definimos un usuario para la administración de base de datos, una base de datos dellstore, enlazamos los volúmenes, y por último definimos que postgres utilice el archivo de configuración que definimos anteriormente.




## Creación del usuario para replicación
Crearemos un usuario con los permisos necesarios para realizar la replicación.


Para ello nos conectamos a la terminal bash del contenedor del nodo maestro para ejecutar el comando de creación.


```
docker exec -it postgres-master bash
```
Utilizamos el siguiente comando para crear un usuario para la replicación, utilizamos el mismo que hemos definido anteriormente en el archivo `pg_hba.conf`
```
createuser -U postgres -P -c 5 --replication replicationUser
```


## Realizar un full-backup
Ya teniendo el nodo master funcionando lo que resta hacer es un full-backup que sirva de base al momento de creación del nodo slave.


Utilizaremos un contenedor efímero que tenga conexión con la red que definimos antes, y que monte la carpeta /pgdata donde se realizará el backup.


**Para realizar los siguientes pasos es importante ubicarse en la carpeta `/slave`**


Lanzamos el contenedor efímero:
```
docker run -it --rm `
--net postgres `
-v ${PWD}/pgdata:/data `
--entrypoint /bin/bash postgres:17
```
Ejecutamos el comando para hacer el full-backup:


>Usamos como `host -h` el nombre del contenedor nodo master, ya que estamos en una red, Docker sabe como traducir la dirección para que la conexión sea efectiva.
```
pg_basebackup -h postgres-master -p 5432 -U replicationUser -D /data/ -Fp -Xs -R
```
## Nodo slave (inicialización)
Solo falta inicializar el nodo slave, gracias a la utilidad `pg_basebackup` ya tenemos preparados los archivos necesarios para que el nodo recupere el full-backup, e inicie en modo standby.


>La utilidad `pg_basebackup` coloca el archivo `standby.signal` en el directorio de datos `/pgdata` automáticamente al realizar el backup.


**Para realizar los siguientes pasos es importante ubicarse en la carpeta `/slave`**
```
docker run -it -d --name postgres-slave `
--net postgres `
-e POSTGRES_USER=postgres `
-e POSTGRES_PASSWORD=postgres `
-e POSTGRES_DB=dellstore `
-e PGDATA="/data" `
-v ${PWD}/pgdata:/data `
-v ${PWD}/config:/config `
-v ${PWD}/wals:/mnt/server/wals `
-p 5401:5432 `
postgres:17 -c 'config_file=/config/postgresql.conf'
```
## Failover (fallo)
En una situación de fallo total o catástrofe del nodo maestro PostgreSQL no tiene una función que permita recuperarse de la caída y promover al nodo esclavo como nodo principal o maestro. Para ello utilizaremos la utilidad `pg_ctl` que convertirá el nodo esclavo en maestro.


Ejecutamos la consola del contenedor del nodo esclavo
```
docker exec -it postgres-slave bash
```
Ejecutamos el comando para promoción como usuario `postgres` (no root)
```
runuser -u postgres -- pg_ctl promote
```


## Monitoreo de la replicación
Para asegurarnos que la replicación esté funcionando correctamente, además de realizar pruebas de modificaciones y comprobaciones dentro de las tablas, hacemos el uso de 3 comandos.


**Verificar conexión de standby's**


 Desde el nodo maestro podemos saber qué nodos standby  están conectados en ese momento utilizando el comando:
```
SELECT * FROM pg_stat_replication;
```
**Verificar modo recuperacion del standby**


Desde el nodo slave podemos saber si el standby está en recuperación con el comando:
```
SELECT pg_is_in_recovery();
```
La consulta devuelve true(t) o false(f) indicando si está en recuperación.


## Conexión con dos computadoras (Red LAN)
Para realizar la conexión con dos computadoras que simulan el nodo maestro y nodo esclavo podemos instalar Docker en cada PC con un contenedor de PostgreSQL, y seguir los pasos anteriores menos el paso de realizar la full-backup el cual debe ser modificado de la siguiente manera:


```
pg_basebackup -h X.X.X.X -p 5432 -U replicationUser -D /data/ -Fp -Xs -R
```
Siendo `X.X.X.X` la IP privada de la PC principal o nodo master.


# Plan de contingencia


Las frecuencias de los full dependen del tamaño de la base de datos. Si se trabaja con una base de datos grande como la de un E-Commerce que contiene un alto volumen de transacciones diarias. Esto requerirá un full backup diario que se haga durante la madrugada para evitar el alto tráfico, así también tener un servidor de respaldo que almacene los WALs (Streaming Replication).


Mientras que si se trabaja con un local, con pocas ventas (pocas transacciones) por minuto, se utilizará un full backup semanal junto con una copia incremental continua. En este caso, se mantendrá registro de dos full backup, el de la semana actual, y el de la semana anterior por si falla el actual. Cuando se elimina un full backup, también se deben eliminar los WALs que están entre dos full backups, debido a que sin el punto de control, estos no tienen sobre qué aplicarse. Como las transacciones son pocas, puede llevar tiempo llenar los 16 MB de un WAL, por lo que se puede configurar el cierre de un archivo cada hora.




| Característica              | E-Commerce (Alto Tráfico)                                   | Local Pequeño (Bajo Tráfico)                                          |
|-----------------------------|--------------------------------------------------------------|------------------------------------------------------------------------|
| Volumen de Transacciones    | Alto/Crítico. Miles de operaciones diarias.                 | Bajo. Pocas ventas por minuto u hora.                                 |
| Frecuencia Full Backup      | Diario.                                                      | Semanal.                                                               |
| Objetivo del Full Backup    | Minimizar el tiempo de restauración (RTO).                  | Ahorrar recursos de disco y CPU.                                      |
| Estrategia de WALs          | Streaming Replication                                        | Continuous Archiving                                                   |
| Configuración Timeout       | Default o Bajo. Los archivos de 16MB se llenan rápido.      | 1 Hora. Se fuerza el cierre del WAL para asegurar los datos.          |
| Política de Retención       | Generalmente corta en días pero con réplica en tiempo real. | 2 Semanas. Se mantiene la semana actual y la anterior (N y N-1).       |
| Lógica de Limpieza          | Rotación continua en servidor réplica.                      | En Cascada. Al borrar el Full Backup viejo, se borran sus WALs huérfanos. |


# Errores


#### 1. El Desafío de los Permisos (UID/GID)


* **El Problema:** El contenedor no podía escribir en las carpetas del host (`./pgdata`, `./wals`). Salía `Permission denied`.
* **La Causa:** El usuario de Linux Mint tiene UID **1000**, pero el usuario `postgres` dentro del contenedor (imagen `postgres:17`) tiene UID **999** (al principio pensamos que era 70, pero confirmamos 999).
* **La Solución:**
    * Identificamos el UID correcto (**999**).
    * Usamos `sudo chown -R 999:999 ./wals` en el host para ceder la propiedad al contenedor.


#### 2. El Bloqueo de Seguridad (AppArmor)


* **El Problema:** Incluso con los permisos `999:999` correctos, el `archive_command` fallaba al escribir en `./wals`.
* **La Causa:** **AppArmor** (el sistema de seguridad de Linux Mint/Ubuntu) bloqueaba al contenedor por intentar escribir en carpetas "sensibles" del usuario (`~/Documentos/...`) o del sistema.
* **La Solución:**
    1.  Movimos el proyecto a una ruta estándar de servicios: **/srv/pitr-master**.
    2.  Añadimos `security_opt: ["apparmor=unconfined"]` al `docker-compose.yml` para permitir que el contenedor escriba libremente en los *bind mounts*.


#### 3. Conflicto de Inicialización (initdb)


* **El Problema:** El contenedor se cerraba inmediatamente (`Exited (1)`) o fallaba al arrancar.
* **La Causa:**
    1.  Pasamos los comandos de configuración como una lista en el archivo `.yml`.
    2.  Intentábamos iniciar un contenedor nuevo sobre un directorio de datos "sucio" o mal restaurado.
* **La Solución:**
    1.  Convertimos el `command` a una sola cadena de texto larga.
    2.  Aprendimos a limpiar siempre: `docker rm` (contenedor) y `docker volume rm` (volumen) antes de intentar una restauración.


#### 4. La Arquitectura de Almacenamiento (Híbrida)


* **El Problema:** Intentar usar *Bind Mounts* para todo (PGDATA y WALs) era inestable por los permisos.
* **La Solución:**
    * Usamos dos **Volúmenes Administrados** (`pgdata` y `wals`). Docker gestiona los permisos internos.


#### 5. Backup y Restauración (pg_basebackup y tar)


* **El Problema:** `pg_basebackup` creaba un directorio en lugar de un archivo, y `tar` fallaba al descomprimir (`Is a directory` o `not found`).
* **La Causa:** `pg_basebackup` crea la estructura de directorios definida en `-D`. Además, usamos flags incorrectos en `tar` (`--strip-components`) que corrompía la estructura al restaurar.
* **La Solución:**
    * Entendimos la estructura: `./backups/backup_nuevo/base.tar.gz`.
    * Corregimos el comando de restauración para apuntar al archivo exacto y quitamos los flags innecesarios.


#### 6. Configuración de PITR (postgresql.conf)


* **El Problema:** Errores en `postgresql.conf` que impedían arrancar la recuperación.
* **La Causa:** Usar `echo` con malas comillas rompió el archivo de configuración.
* **La Solución:** Simplemente enlazar un archivo postgresql.conf externo con el interno.

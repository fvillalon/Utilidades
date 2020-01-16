## REPLICACION MASTER-SLAVE [APPSERV 9.3](https://www.appserv.org/en/)

* instalar appserv sin mysql, en vez de usar localhost usa tu IP 10.8.71.XX
* instalar mysql en C con el workbench opcional
* debe tener el binlog aparece en configuracion o agregar en log_bin en my.ini
* cuando escojas nombre del servicio usar algo como MySQL-DEV
* crear usuario replica como replication slave (opcional, puede hacerce mas adelante)
* NO ARRANCAR MYSQL
***
modificar archivo my.ini en 
````
C:\programdata\msyql-server...
````

agregar las bases de datos a replicar :

````ini
binlog-do-db=basedatos1
binlog-do-db=basedatosX
````

copiar la carpeta  
````
C:\programdata\msyql-server... 
````
 a
````
D:\AppServ\  [RUTA APPSERV]
````
###### (se puede cambiar el nombre se sugiere MySQL-DEV-RPL)

editar el archivo my.ini en nueva ruta D:\AppServ\MySQL-DEV-RPL\bin

````
server-id=1 por 2
````

***

````ini
binlog-do-db=basedatos1
binlog-do-db=basedatosX

por 

replicate-do-db=basedatos1
replicate-do-db=basedatosX
````

* cambiar puertos 3306 por 3307
* cambiar rutas de carpetas apuntando a C: por D:\appserv\mysql ...
* habilitar #basedir

````ini
basedir="D:/AppServ/MySQL-DEV-RPL/"
````
cambiar (no se utilizara otro metodo)

````ini
default_authentication_plugin=mysql_native_password
````

arrancar mysql desde servicios  MySQL-DEV o desde comando
````bash
net start MySQL-DEV
````

probar entrando en phpmyadmin http://10.8.71.XX/phpmyadmin

si no entra el phpmyadmin se debe por caching_sha2_... 
entrar via commando o workbench y ejecutar

~~~
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'SUPER-CLAVE-QUE-PUSISTE';
~~~

en cmd como administrador ejecutar comando para añadir un servicio

````
mysqld --install MySQL-DEV-RPL --defaults-file="D:\AppServ\MySQL-DEV-RPL\my.ini"
````

## INICIALIZACION data

### forma 1

copiar carpeta Data
~~~
C:\ProgramData\MySQL\MySQL Server 8.0\Data

a

D:\AppServ\MySQL-DEV-RPL
~~~

### forma 2

inicializar la carpeta data

````
mysqld --initialize-insecure
````
entrar sin password y cambiarlo con alter user ...
````
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'SUPER-CLAVE-QUE-PUSISTE';
````

***

## DEPLOY

arrancar mysql desde servicios  MySQL-DEV-RPL o desde comando
````
net start MySQL-DEV-RPL
````
pueba con
````
mysql -u root -p -P3307
````
primero importar la bd en la 3306 via cmd
````
mysql -u root -p -P3306 basedatos1 < basedatos1.sql
````
o 
````
mysqldump -u root --all-databases --master-data=2 > /root/dbdump.db
````

###### *SOLO SI NO CARGA,CREAR LA BASE DE DATOS PRIMERO EN AMBOS SERVIDORES
***
### CONFIGURAR MASTER

Crear usuario replica (sólo si no lo creaste al inicio)
````
create user 'replica'@'localhost' identified with mysql_native_password by 'SUPER-CLAVE';
grant replication slave on *.* TO 'replica'@'localhost';
````
###### deberias revisar en el phpmyadmin como se va realizando la replica antes de levantar el slave

***
PARA REVISAR EN LA REPLICA en phpmyadmin
***
cambiar en archivo 
~~~
D:\AppServ\www\phpMyAdmin\libraries\config.default.php
~~~

````
$cfg['Servers'][$i]['port'] = ''
````
por 
````
$cfg['Servers'][$i]['port'] = '3307'
````

sin reniciar, para volver al master dejarlo blanco o 3306

para comprobar  al lado del LOGO phpmyadmin , arriba de pestaña base de datos aparece >
~~~
 Servidor:localhost:3307
~~~

***

##SLAVE

configurar SLAVE en mysql ingresar via CMD
~~~bash
mysql -u root -p -P3307
~~~

Una vez ingresado
~~~
change master to master_host='localhost',
master_port=3306,master_user='replica',
master_password='SUPER-CLAVE-DE-MASTER';
~~~
~~~
start slave;
~~~
~~~
show slave status\G;
~~~
# PROFIT

## CARGAR NUEVA BASE

La mejor alternativa es eliminar las bases que estan replicadas y re-cagarlas usando en slave;

~~~
STOP SLAVE;
~~~

cargar la base en servidor slave
~~~
mysqldump .... nuevabase < nuevabase.sql;
~~~

detener servicios mysql de slave  y master;
agregar en my.ini  (de master y replica)


MASTER 
~~~
binlog-do-db=nuevabase

~~~
 
SLAVE
~~~
replicate-do-db=nuevabase
~~~

levantar servicios mysql
y luego levantar slave via

~~~bash
mysql -u root -p -P3307
~~~

~~~
start slave;
~~~

Para verificar  
~~~bash
show slave status\G;
~~~


# ERRORES

En el peor de los casos, para comenzar la re-sincronizacion ejecutar:

### SLAVE
~~~
stop slave;
~~~

### MASTER
~~~
reset master;
~~~


### SLAVE
~~~
reset slave;
~~~
~~~
start slave;
~~~


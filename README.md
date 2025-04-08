# Taller-RUNDECK-2025
Taller de Rundeck con PostgreSQL y proxies inversos.


<details>
  <summary>Lab</summary>

  
| Hostname               | Laboratorio | IP           | IP VPN         | User       | Password   |
|------------------------|-------------|--------------|----------------|------------|------------|
| Taller-Rundeck-Alondra | Alondra     | 10.0.0.183   | 100.75.34.113  | demorundec | redhat2025 |
| Taller-Rundeck-Omar    | Omar        | 10.0.0.185   | 100.115.190.104| demorundec | redhat2025 |
| Taller-Rundeck-Angel   | Angel       | 10.0.0.200   | 100.109.208.48 | demorundec | redhat2025 |
| Taller-Rundeck-Brandon | Brandon     | 10.0.0.181   | 100.116.65.103 | demorundec | redhat2025 |
| Taller-Rundeck-Hecto   | Hecto       | 10.0.0.221   | 100.81.37.68   | demorundec | redhat2025 |
| Taller-Rundeck-Javier  | Javier      | 10.0.0.192   | 100.82.186.128 | demorundec | redhat2025 |
| Taller-Rundeck-Mike    | Mike        | 10.0.0.190   | 100.102.173.14 | demorundec | redhat2025 |
| Taller-Rundeck-Olive   | Olivel      | 10.0.0.195   | 100.114.185.46 | demorundec | redhat2025 |

</details>
<details>
  <summary>clientes</summary>

| User     | Password     | Address     | Root Password |
|----------|--------------|-------------|----------------|
| mike     | mike2025     | 10.0.0.191  | taller2025     |
| javi     | javi2025     | 10.0.0.193  | taller2025     |
| oliver   | oliver2025   | 10.0.0.194  | taller2025     |
| antonio  | antonio2025  | 10.0.0.196  | taller2025     |
| angel    | angel2025    | 10.0.0.199  | taller2025     |
| brandom  | brandom2025  | 10.0.0.180  | taller2025     |
| alondra  | alondra2025  | 10.0.0.182  | taller2024     |
| omar     | omar2025     | 10.0.0.184  | taller2025     |
| jatzy    | jatzy2025    | 10.0.0.186  | taller2025     |
| sergio   | sergio2025   | 10.0.0.189  | taller2025     |
| hector   | hector2025   | 10.0.0.220  | taller2025     |
| miguel   | miguel2025   | 10.0.0.222  | taller2025     |

</details>



## Paso 1. Instalación de Rundeck Community

### 1.1. JDK Java 11
```bash
sudo yum install java-11-openjdk-devel
```

### 1.2. Instalación automática vía curl
```bash
curl https://raw.githubusercontent.com/rundeck/packaging/main/scripts/rpm-setup.sh 2> /dev/null | sudo bash -s rundeck
```

### 1.3. Instalación de Rundeck
```bash
sudo yum install rundeck
```

### 1.4. Activación de Puerto TCP para Rundeck
```bash
sudo firewall-cmd --permanent --add-port=4440/tcp
sudo firewall-cmd --reload
sudo netstat -tuln | grep 4440
```

### 1.5. Instalación de Python3
```bash
sudo yum install python3-gps
```
## 1.6. Reiniciar urndeck 
```bash
sudo systemctl restart rundeckd

#comprobar si se esta ejecutando 
sudo systemctl status rundeckd
```
### 1.7. Ejecución de Log para ver posibles problemas
```bash
tail -f /var/log/rundeck/service.log
```
> [!IMPORTANT]
> **Conectividad en red para Rundeck**
>
> En este paso, Rundeck solo funcionará para las personas que estén en la misma red local.  
> Si lo estás ejecutando de forma local en tu equipo, funcionará correctamente.  
> 
> Sin embargo, si lo estás ejecutando desde un servidor, será necesario que tú y los demás usuarios estén conectados a la misma red.
>
> ---
>
> En caso de que estés utilizando una VPN para acceder al servidor, será necesario configurar un **proxy inverso** para que Rundeck sea accesible desde fuera de la red local.

## Paso 2. Proxy inverso con Apache httpd

### 2.1. Instalación de Apache
```bash
sudo yum install httpd -y
```

### 2.2. Habilitar Módulos Necesarios
```bash
# Habilitar e iniciar el servicio
sudo systemctl enable httpd
sudo systemctl start httpd
sudo yum install httpd mod_ssl -y

#Habitiliatar modulos necesarios
sudo tee /etc/httpd/conf.modules.d/00-proxy.conf <<'EOL'
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
LoadModule proxy_wstunnel_module modules/mod_proxy_wstunnel.so
LoadModule rewrite_module modules/mod_rewrite.so
LoadModule ssl_module modules/mod_ssl.so
LoadModule headers_module modules/mod_headers.so
EOL
# Verificar estado
sudo systemctl status httpd

# Reiniciar Apache
sudo systemctl restart httpd


```

### 2.3. Creación de Archivo de Configuración
```bash
sudo vim /etc/httpd/conf.d/rundeck.conf
```
```text
VirtualHost *:80>
    ServerName 10.0.0.186
    ServerAdmin admin@yourdomain.com

    # Configuración esencial del proxy
    ProxyRequests Off
    ProxyPreserveHost On
    ProxyPass / http://localhost:4440/ timeout=600
    ProxyPassReverse / http://localhost:4440/

    # Cabeceras CRÍTICAS que faltaban
    RequestHeader set X-Forwarded-Host "10.0.0.186"  
    RequestHeader set X-Forwarded-Proto "http"
    RequestHeader set X-Forwarded-Port "80"
    RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}e"
    RequestHeader set X-Forwarded-Server "%{SERVER_NAME}e"

    # WebSockets optimizado
    RewriteEngine On
    RewriteCond %{HTTP:Upgrade} =websocket [NC]
    RewriteCond %{HTTP:Connection} upgrade [NC]
    RewriteRule /(.*) ws://localhost:4440/$1 [P,L,NE]

    # Configuración de logs
    ErrorLog /var/log/httpd/rundeck_error.log
    CustomLog /var/log/httpd/rundeck_access.log combined
</VirtualHost>
```

### 2.4. Reiniciar Servicios
```bash
sudo systemctl restart httpd
```

### 2.5. Configuración de Extras para el Proxy
```bash
sudo vim /etc/rundeck/rundeck-config.properties
```
```text
grails.serverURL=http://100.76.213.65:4440
server.useForwardHeaders=true

# tailscale
rundeck.security.authorization.preauthenticated.redirectUserToServer=true
rundeck.security.authorization.preauthenticated.redirectLogout=true
rundeck.security.authorization.preauthenticated.userNameHeader=X-Forwarded-User
rundeck.security.authorization.preauthenticated.userRolesHeader=X-Forwarded-Roles
```

### 2.6. Activación de Puertos TCP para Apache
```bash
sudo firewall-cmd --permanent --add-port=80/tcp

# Reinicio de Firewall
sudo firewall-cmd --reload

# Reinicio de Servicios Rundeck y httpd
sudo systemctl restart httpd rundeckd

# Ejecución de los logs de Rundeck
tail -f /var/log/rundeck/service.log /var/log/httpd/error_log
```

## Paso 3. Conexión con PostgreSQL

### 3.1. Instalación de PostgreSQL
```bash
sudo yum install postgresql-server -y

# Estado de PostgreSQL
sudo systemctl status postgresql

# Inicializar un nuevo clúster de base de datos PostgreSQL
sudo postgresql-setup --initdb

# Iniciar PostgreSQL
sudo systemctl start postgresql

# Habilitar el inicio automático del servicio PostgreSQL cuando el sistema operativo arranque.
sudo systemctl enable postgresql
```

### 3.2. Ingreso a PostgreSQL
```bash
# Acceso a PostgreSQL
sudo -i -u postgres

# Acceder al prompt de PostgreSQL
psql
```

### 3.3. Creación de la Base de Datos
```bash
create database rundeckdb;
```

### 3.4. Creación de Usuario y Permisos
```bash
# Crear el usuario
create user rundeckuser with password 'pass personalizado';

# Asignar permisos al usuario creado
grant ALL privileges on database rundeckdb to rundeckuser;
```

### 3.5. Configuración del archivo de propiedades de Rundeck
```bash
sudo vim /etc/rundeck/rundeck-config.properties
```
```text
dataSource.driverClassName = org.postgresql.Driver
dataSource.url = jdbc:postgresql://localhost/rundeck
dataSource.username = rundeckuser
dataSource.password = passwordForRundeck
```
> [!NOTE]
> Comentar las siguientes líneas
> ```
> #dataSource.dbCreate =
> #dataSource.url =
> ```

### 3.6. Configuración de pg_hba.conf para IPv4 y IPv6
```bash
sudo vim /var/lib/pgsql/data/pg_hba.conf
```
```bash
# IPv4 local connections:
host    all             rundeckuser     127.0.0.1/32            md5
# IPv6 local connections:
host    all             rundeckuser     ::1/128                 md5
```
> [!TIP]
> En este archivo podrás asignar roles a los usuarios.
> ```bash
> sudo vim /etc/rundeck/realm.properties
> ```
> **Ejemplo de usuarios con rol:**
> ```properties
> rundeck_user:(password),user            # Rol básico
> rundeck_architect:(password),architect  # Creador de jobs
> rundeck_deploy:(password),deploy        # Especialista en despliegues
> rundeck_build:(password),build          # Integración continua
> ```

### 3.7. Reinicio de Servicios de RunDeck
```bash
sudo systemctl restart rundeckd

# Ejecución de logs de Rundeck
tail -f /var/log/rundeck/service.log
```

## 4 Nodos con Aechivos XLM

## 4.1 Permisos Necesarios para el Usuario rundeck
```bash
sudo chown rundeck:rundeck /var/lib/rundeck/.ssh

#comando para verificar permisos
ls -ld .ssh
```
## 4.2 Creacion de llaves RSA
```bash
sudo -u rundeck ssh-keygen -t rsa -b 4096 -f /var/lib/rundeck/.ssh/id_rsa_pruebanodo1
```
## 4.3 Jecucion para hacer la comunicacion 
```bash
sudo -u rundeck ssh-copy-id -i /var/lib/rundeck/.ssh/id_rsa_pruebanodo1 cliente03rk@10.0.0.182
```

## 4.3 Script 
```bash
#!/bin/bash

# Imprimir el nombre del servidor
echo "Nombre del servidor:"
hostname
echo ""

# Mostrar usuarios conectados actualmente
echo "Usuarios conectados actualmente:"
who
echo ""

# Mostrar el uso de la memoria
echo "Memoria RAM - Información:"
cat /proc/meminfo | grep -E "MemTotal|MemFree"
echo ""

# Mostrar el uso del disco en todos los puntos de montaje
echo "Uso del disco por puntos de montaje:"
df -h
echo ""

# Mostrar los últimos usuarios conectados (últimos 10 días)
echo "Registro de últimos accesos de usuarios:"
lastlog -t 10
echo ""
```


## 5 Conexion con equipos windows Winrm

## 5.1 instalcion de python3
```bash
sudo yum install python3-pip
```
## 5.2 Instalacion de Winrm
```bash
pip install pywinrm
```



## 6 Instalacion de Ansible 

## 6.1 Intalcion de Repositorios 
```bash
sudo yum install epel-release –y
```

## 6.2 Instalacion de Ansible 
```bash
sudo yum install ansible –y
```
## 6.3 verificacion de ansible 
```bash
ansible --version
```
## 6.4 Instalcion de Plugins de Ansible
```bash
curl -L -o ansible-plugin-4.0.7.jar https://github.com/rundeck-plugins/ansible-plugin/releases/download/v4.0.7/ansible-plugin-4.0.7.jar 
```
## 6.5 Mover archivos del plugin a la carpeta libext
```bash
sudo mv ansible-plugin-4.0.7.jar /var/lib/rundeck/libext/

#Reinicio del sitema rundeck
sudo systemctl restart rundeckd 
```

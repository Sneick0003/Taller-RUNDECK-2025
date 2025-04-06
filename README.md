# Taller-RUNDECK-2025
Taller de Rundeck con PostgreSQL y proxies inversos.

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

### 1.6. Ejecución de Log para ver posibles problemas
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

# Verificar estado
sudo systemctl status httpd

# Reiniciar Apache
sudo systemctl restart httpd
```

### 2.3. Creación de Archivo de Configuración
```bash
sudo nano /etc/apache2/sites-available/rundeck.conf
```
```text
<VirtualHost *:80>
    ServerName 100.76.213.65

    ProxyPreserveHost On
    ProxyPass / http://localhost:4440/
    ProxyPassReverse / http://localhost:4440/

    # Cabeceras para Tailscale
    RequestHeader set X-Forwarded-Proto "http"
    RequestHeader set X-Forwarded-Port "80"
    RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}e"

    # WebSockets
    RewriteEngine On
    RewriteCond %{HTTP:Upgrade} websocket [NC]
    RewriteCond %{HTTP:Connection} upgrade [NC]
    RewriteRule /(.*) ws://localhost:4440/$1 [P,L]
</VirtualHost>
```

### 2.4. Reiniciar Servicios
```bash
sudo systemctl restart httpd
```

### 2.5. Configuración de Extras para el Proxy
```bash
sudo nano /etc/rundeck/rundeck-config.properties
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
sudo nano /etc/rundeck/rundeck-config.properties
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
sudo nano /var/lib/pgsql/data/pg_hba.conf
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
> sudo nano /etc/rundeck/realm.properties
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

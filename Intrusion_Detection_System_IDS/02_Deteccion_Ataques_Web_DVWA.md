# 02: Detección de Ataques Web (SQLi y XSS) con DVWA

Este documento detalla la configuración de un laboratorio para simular ataques web contra *Damn Vulnerable Web Application* (DVWA) y validar la detección mediante la integración de Wazuh y Suricata descrita en el documento anterior.

## Infraestructura del Laboratorio

* **Servidor Wazuh (VM):** `192.168.1.136`
* **Monitor (Agente + Suricata):** PC anfitrión con Ubuntu (`192.168.1.133`), actuando como TAP server
* **Servidor Víctima (VM):** Debian 12 con DVWA (`192.168.1.135`)
* **Atacante:** Equipo Windows en la misma red

---

## 1. Instalación del Servidor Víctima (DVWA)

### 1.1 Corrección de la Configuración de Red

1. Apagar la VM de Debian.
2. En VirtualBox, abrir **Configuración → Red**.
3. Cambiar "Conectado a:" de `NAT` a **Adaptador Puente**.
4. Iniciar la VM y comprobar con `ip a` que obtiene una IP de la red local (ej. `192.168.1.135`).

### 1.2 Instalación de Dependencias

Instalar Apache, MariaDB, PHP y `git`:

```bash
sudo apt -y install apache2 mariadb-server php php-mysqli php-gd libapache2-mod-php git
```

### 1.3 Configuración de la Base de Datos

1. Ejecutar la instalación segura de MariaDB:

    ```bash
    sudo mariadb-secure-installation
    ```

    *Establecer una contraseña para el usuario root cuando se solicite.*

2. Conectarse como root a MariaDB:

    ```bash
    sudo mysql -u root -p
    ```

3. Crear el usuario y conceder privilegios sobre la base de datos `dvwa`:

    ```sql
    CREATE USER 'dvwa'@'localhost' IDENTIFIED BY 'password';
    GRANT ALL PRIVILEGES ON dvwa.* TO 'dvwa'@'localhost' IDENTIFIED BY 'password';
    FLUSH PRIVILEGES;
    ```

4. Salir de la consola de MariaDB:

    ```sql
    exit
    ```

### 1.4 Instalación de DVWA

1. Posicionarse en el directorio web y clonar el repositorio oficial:

    ```bash
    cd /var/www/html
    sudo git clone https://github.com/digininja/DVWA.git
    sudo chown -R www-data:www-data /var/www/html/*
    ```

2. Copiar el fichero de configuración por defecto:

    ```bash
    sudo cp /var/www/html/DVWA/config/config.inc.php.dist /var/www/html/DVWA/config/config.inc.php
    ```

3. Verificar (opcional) que los valores por defecto `db_user='dvwa'` y `db_password='password'` en `config.inc.php` son correctos.

### 1.5 Configuración de PHP y Lanzamiento de DVWA

1. Iniciar el servicio de base de datos:

    ```bash
    sudo systemctl start mariadb
    ```

2. Editar el fichero `php.ini` para permitir `allow_url_include`:

    ```bash
    sudo nano /etc/php/8.2/apache2/php.ini
    ```

    Buscar la directiva y dejarla en:

    ```ini
    allow_url_include = On
    ```

3. Reiniciar Apache para aplicar los cambios:

    ```bash
    sudo systemctl restart apache2
    ```

4. En la VM Debian, abrir `http://localhost/DVWA/setup.php`, pulsar **Create / Reset Database** y acceder con `admin / password`.

---

## 2. Ajuste Crítico: Configuración de Suricata

La HOME_NET original no incluía la IP del servidor DVWA, evitando la generación de alertas.

1. En el monitor Ubuntu, editar `/etc/suricata/suricata.yaml`:

    ```bash
    sudo nano /etc/suricata/suricata.yaml
    ```

2. Ajustar las direcciones monitorizadas:

    ```yaml
    vars:
      address-groups:
        HOME_NET: "[192.168.1.135]"
        EXTERNAL_NET: "any"
    ```

3. Reiniciar Suricata para aplicar la configuración:

    ```bash
    sudo systemctl restart suricata
    ```

---

## 3. Simulación de Ataque: SQL Injection (SQLi)

1. Desde el equipo atacante, acceder a `http://192.168.1.135/DVWA/`.
2. Autenticarse con `admin / password`.
3. En **DVWA Security**, fijar el nivel en *low* y pulsar **Submit**.
4. Abrir la sección **SQL Injection**.
5. Lanzar el payload indicado en el libro:

    ```text
    http://192.168.1.135/DVWA/vulnerabilities/sqli/?id=a' UNION SELECT "Hello","Hello Again";-- -&Submit=Submit
    ```

6. Confirmar que la página muestra "First name: Hello" y "Surname: Hello Again".

### Visualización de la Alerta (SQLi)

En el dashboard de Wazuh, filtrar por `rule.groups: suricata` y verificar los campos relevantes:

* `data.alert.signature`: `ET WEB_SERVER Possible SQL Injection Attempt UNION SELECT in HTTP URI`
* `data.dest_ip`: `192.168.1.135`
* `data.src_ip`: IP del equipo atacante



---

## 4. Simulación de Ataque: Reflected XSS

1. Mantener DVWA con seguridad *low*.
2. Abrir la sección **XSS (Reflected)**.
3. Enviar el siguiente payload:

    ```html
    <script>alert("Hello");</script>
    ```

4. Validar que el navegador muestra un *popup* con el texto "Hello".

### Visualización de la Alerta (XSS)

En Wazuh se genera una nueva alerta de Suricata:

* `rule.description`: `Suricata: Alert - ET WEB_SERVER Script tag in URI Cross Site Scripting Attempt`
* `data.dest_ip`: `192.168.1.135`
* `data.src_ip`: IP del equipo atacante
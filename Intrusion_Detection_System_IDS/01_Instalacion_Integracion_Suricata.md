# 01: Instalación del Servidor, Agente e Integración con Suricata

Este documento describe el proceso de instalación de un entorno de laboratorio de Wazuh y su integración con el NIDS Suricata, basado en el "Capítulo 1: Intrusion Detection System (IDS) Using Wazuh" y la documentación oficial de Wazuh.

## Infraestructura del Laboratorio

* **Servidor Wazuh:** Máquina Virtual (OVA) oficial de Wazuh.
  * **IP:** `192.168.1.136`
* **Endpoint (Agente):** PC anfitrión con Ubuntu.
  * **IP:** `192.168.1.133`
  * **Interfaz de Red:** `enp1s0`
* **Atacante:** Un PC con Windows en la misma red.

---

## 1. Instalación del Servidor Wazuh (OVA)

Para el entorno de laboratorio, se utiliza la máquina virtual (OVA) proporcionada por Wazuh. Esta opción unifica todos los componentes centrales: servidor Wazuh, indexador y dashboard.

1. **Importación:** Se importa el fichero OVA en el software de virtualización (por ejemplo, VirtualBox).
2. **Configuración de Red:** Se configura la red de la VM en modo "Adaptador Puente" (Bridge) para que obtenga una IP de la misma red que el PC anfitrión.
3. **Credenciales de Acceso:** Una vez iniciada la VM, se accede a la interfaz web (`https://192.168.1.136`).
    * **Usuario:** `admin`
    * **Contraseña:** `admin`

---

## 2. Instalación del Agente Wazuh (Endpoint Ubuntu)

El agente se instala en el endpoint que se desea monitorear (el PC anfitrión Ubuntu).

1. **Acceder al Dashboard:** En la interfaz web de Wazuh, navegar a **Agents** y hacer clic en **Deploy an agent**.
2. **Generar Comando:**
    * Se selecciona el sistema operativo (Linux) y la arquitectura (DEB amd64).
    * Se introduce la dirección IP del servidor: `192.168.1.136`.
    * Se asigna un nombre al agente (ej. `ubu-serv`) y se selecciona un grupo.
3. **Instalación:** Se copia el comando `curl` proporcionado y se ejecuta en el terminal del PC Ubuntu anfitrión. Este comando descarga el paquete, configura la IP del servidor e inicia el servicio del agente.

---

## 3. Instalación de Suricata (Endpoint Ubuntu)

Suricata se instala en el mismo endpoint donde reside el agente de Wazuh.

1. **Añadir Repositorio:**

    ```bash
    sudo add-apt-repository ppa:oisf/suricata-stable
    ```

2. **Actualizar e Instalar:**

    ```bash
    sudo apt-get update
    sudo apt-get install suricata -y
    ```

---

## 4. Configuración de Suricata

### 4.1. Descarga de Reglas (Emerging Threats)

Se utiliza el conjunto de reglas de Emerging Threats (ET).

```bash
cd /tmp/ && curl -LO https://rules.emergingthreats.net/open/suricata-6.0.8/emerging.rules.tar.gz
sudo tar -xvzf emerging.rules.tar.gz && sudo mkdir /etc/suricata/rules && sudo mv rules/*.rules /etc/suricata/rules/
sudo chmod 777 /etc/suricata/rules/*.rules
```

### 4.2. Modificación del archivo de configuración

Editar el archivo `/etc/suricata/suricata.yaml` para ajustar la configuración:

```yaml
vars:
  address-groups:
    HOME_NET: "[192.168.1.0/24]"
    EXTERNAL_NET: "any"

default-rule-path: /etc/suricata/rules
rule-files:
  - "*.rules"

af-packet:
  - interface: enp1s0
```

### 4.3. Reiniciar el servicio de Suricata

```bash
sudo systemctl restart suricata
sudo systemctl enable suricata
```

---

## 5. Integración con Wazuh

Para que el agente de Wazuh pueda monitorear los logs de Suricata, se debe configurar el archivo `/var/ossec/etc/ossec.conf`:

```xml
<ossec_config>
  <localfile>
    <log_format>json</log_format>
    <location>/var/log/suricata/eve.json</location>
  </localfile>
</ossec_config>
```

Reiniciar el agente de Wazuh:

```bash
sudo systemctl restart wazuh-agent
```

---

## 6. Sintaxis de las Reglas de Suricata

### 6.1. Estructura Básica de una Regla

Una regla de Suricata sigue el siguiente formato:

```text
acción protocolo ip_origen puerto_origen -> ip_destino puerto_destino (opciones;)
```

**Ejemplo básico:**

```text
alert tcp any any -> 192.168.1.0/24 22 (msg:"SSH connection detected"; sid:1000001;)
```

### 6.2. Componentes de una Regla

#### **Acción (Action)**

Define qué hacer cuando se activa la regla:

* `alert` - Genera una alerta
* `drop` - Descarta el paquete (modo IPS)
* `reject` - Rechaza la conexión
* `pass` - Permite el tráfico

#### **Protocolo (Protocol)**

Tipo de protocolo de red:

* `tcp` - Protocolo TCP
* `udp` - Protocolo UDP
* `icmp` - Protocolo ICMP
* `ip` - Cualquier protocolo IP

#### **Direcciones IP y Puertos**

* `any` - Cualquier dirección o puerto

* `$HOME_NET` - Variable para red local
* `!$HOME_NET` - Negación (cualquier IP excepto HOME_NET)
* `192.168.1.0/24` - Rango de red específico
* `[80,443,8080]` - Lista de puertos específicos
* `1:1024` - Rango de puertos

### 6.3. Opciones Principales

#### **msg (Mensaje)**

Descripción de la alerta:

```text
msg:"Descripción del ataque detectado";
```

#### **sid (Signature ID)**

Identificador único de la regla:

```text
sid:1000001;
```

#### **content (Contenido)**

Busca contenido específico en el payload:

```text
content:"GET"; content:"/admin";
```

#### **flow**

Dirección del flujo de tráfico:

```text
flow:to_server,established;  # Tráfico hacia el servidor en conexión establecida
flow:to_client;              # Tráfico hacia el cliente
flow:from_server;            # Tráfico desde el servidor
```

#### **classtype**

Categoriza el tipo de ataque:

```text
classtype:web-application-attack;
classtype:trojan-activity;
classtype:policy-violation;
```

### 6.4. Ejemplos de Reglas Comunes

#### **Detección de Escaneo de Puertos**

```text
alert tcp any any -> $HOME_NET any (msg:"Potential port scan detected"; flags:S; threshold: type both, track by_src, count 10, seconds 60; sid:1000002;)
```

#### **Detección de SQL Injection**

```text
alert tcp any any -> $HOME_NET [80,443,8080] (msg:"SQL Injection attempt detected"; content:"UNION SELECT"; nocase; classtype:web-application-attack; sid:1000003;)
```

#### **Detección de XSS**

```text
alert tcp any any -> $HOME_NET [80,443] (msg:"XSS attempt detected"; content:"<script>"; nocase; classtype:web-application-attack; sid:1000004;)
```

#### **Detección de Conexión SSH**

```text
alert tcp $EXTERNAL_NET any -> $HOME_NET 22 (msg:"SSH connection from external network"; flow:to_server,established; content:"SSH-2.0"; sid:1000005;)
```

#### **Detección de DNS Tunneling**

```text
alert dns any any -> any any (msg:"Suspicious DNS query - potential tunneling"; dns_query; content:".onion"; nocase; sid:1000006;)
```

### 6.5. Modificadores de Contenido

* `nocase` - Ignora mayúsculas/minúsculas
* `depth:10` - Busca solo en los primeros 10 bytes
* `offset:5` - Comienza la búsqueda después de 5 bytes
* `distance:5` - Distancia desde el match anterior
* `within:10` - Dentro de los próximos 10 bytes

### 6.6. Variables de Red Personalizadas

En el archivo `suricata.yaml` se pueden definir variables:

```yaml
vars:
  address-groups:
    HOME_NET: "[192.168.1.0/24,10.0.0.0/8]"
    EXTERNAL_NET: "!$HOME_NET"
    DMZ_NET: "172.16.1.0/24"
    WEB_SERVERS: "[192.168.1.10,192.168.1.11]"
  port-groups:
    HTTP_PORTS: "[80,81,300,311,383,591,593,631,901,1220,1414,1741,1830,2301,2381,2809,3037,3128,3702,4343,4848,5250,6988,7000,7001,7144,7145,7510,7777,7779,8000,8008,8014,8028,8080,8085,8088,8090,8118,8123,8180,8181,8243,8280,8300,8800,8888,8899,9000,9060,9080,9090,9091,9443,9999,11371,34443,34444,41080,50002,55555]"
```

### 6.7. Buenas Prácticas

1. **SID únicos:** Usar rangos específicos para reglas personalizadas (ej: 1000000-1999999)
2. **Mensajes descriptivos:** Incluir información relevante sobre el tipo de amenaza
3. **Clasificación:** Usar `classtype` para categorizar las reglas
4. **Optimización:** Usar `fast_pattern` para mejorar el rendimiento
5. **Testing:** Probar las reglas en un entorno controlado antes del despliegue

---

## 7. Verificación de la Instalación

### 7.1. Verificar el estado de Suricata

```bash
sudo systemctl status suricata
sudo tail -f /var/log/suricata/suricata.log
```

### 7.2. Verificar la integración con Wazuh

```bash
sudo tail -f /var/log/suricata/eve.json
sudo tail -f /var/ossec/logs/ossec.log
```

### 7.3. Generar tráfico de prueba

Para verificar que la detección funciona, se pueden ejecutar comandos simples:

```bash
# Escaneo de puertos
nmap -sS 192.168.1.136

# Consulta DNS sospechosa (simulada)
curl "http://192.168.1.136/test.php?id=1' UNION SELECT * FROM users--"
```

Los eventos deberían aparecer en el dashboard de Wazuh bajo "Security Events" con el filtro `rule.groups: suricata`.

# Laboratorio Wazuh: Bloqueo de IP Maliciosa (Setup Adaptado)

Este documento resume los pasos para completar el laboratorio "Blocking a known malicious actor" de Wazuh, utilizando un setup doméstico específico.

## Infraestructura del Laboratorio

* Servidor Wazuh: Una Máquina Virtual (VM) corriendo en el PC Ubuntu.
* Endpoint Víctima: El PC anfitrión Ubuntu (192.168.1.130), que tiene el agente Wazuh instalado.
* Atacante: Un PC Windows en la misma red (192.168.1.133).

---

## 1. Configurar la Víctima (PC Ubuntu Anfitrión)

El objetivo es instalar un servicio para "atacar" y decirle al agente de Wazuh que vigile sus logs.

### Instalar Apache (Servidor Web)

En el PC Ubuntu anfitrión, instala Apache para simular un servicio web.

```bash
sudo apt update
sudo apt install apache2
```

### Configurar el Agente Wazuh

Edita el fichero de configuración del agente en el PC Ubuntu para que monitorice los logs de acceso de Apache.

Edita /var/ossec/etc/ossec.conf y añade este bloque:

```xml
<localfile>
  <log_format>apache</log_format>
  <location>/var/log/apache2/access.log</location>
</localfile>
```

### Reiniciar el Agente

Aplica los cambios reiniciando el servicio del agente.

```bash
sudo systemctl restart wazuh-agent
```

---

## 2. Configurar el Servidor (Wazuh Server VM)

Aquí es donde creamos la lógica de detección y respuesta. (Estos pasos se ejecutan como root dentro de la VM).

### Obtener Utilidades y Lista de IPs

Descargamos las herramientas necesarias y la lista de reputación de AlienVault.

```bash
# Descargar script de conversión
curl -so /tmp/iplist-to-cdblist.py https://raw.githubusercontent.com/wazuh/wazuh/4.7/src/active-response/iplist-to-cdblist.py

# Descargar lista de reputación
curl -so /var/ossec/etc/lists/alienvault_reputation.ipset https://reputation.alienvault.com/reputation.generic
```

### Adaptación Clave: Añadir IP del Atacante

La IP de nuestro PC Windows no está en la lista de AlienVault, así que la añadimos manualmente.

```bash
# (Reemplaza la IP si es diferente)
echo "192.168.1.133" >> /var/ossec/etc/lists/alienvault_reputation.ipset
```

### Convertir la Lista a Formato CDB

Convertimos el archivo de texto (.ipset) a un formato binario (.cdb) que Wazuh pueda leer rápidamente.

```bash
python3 /tmp/iplist-to-cdblist.py /var/ossec/etc/lists/alienvault_reputation.ipset /var/ossec/etc/lists/blacklist-alienvault
```

### Asignar Permisos Correctos

Cambiamos el propietario del archivo a wazuh:wazuh para que los servicios de Wazuh (que no corren como root) tengan permiso para leerlo.

```bash
chown wazuh:wazuh /var/ossec/etc/lists/blacklist-alienvault
```

### Crear Regla de Detección

Añadimos una regla personalizada en /var/ossec/etc/rules/local_rules.xml que se disparará si una IP del log coincide con nuestra lista.

```xml
<group name="attack,">
  <rule id="100100" level="10">
    <if_group>web|attack|attacks</if_group>
    <list field="srcip" lookup="address_match_key">etc/lists/blacklist-alienvault</list>
    <description>IP address found in AlienVault reputation database.</description>
  </rule>
</group>
```

### Configurar Servidor y Respuesta Activa

Editamos el archivo principal /var/ossec/etc/ossec.conf en el servidor para que:

A) Cargue nuestra nueva lista de IPs.

B) Defina la acción a tomar (bloquear la IP) cuando la regla 100100 se dispare.

```xml
<ossec_config>

  <ruleset>
    <!-- ... otras listas ... -->
    <!-- A) Añadimos nuestra lista -->
    <list>etc/lists/blacklist-alienvault</list>
  </ruleset>

  <!-- ... resto de la configuración ... -->

  <!-- B) Añadimos el bloque de respuesta activa -->
  <active-response>
    <disabled>no</disabled>
    <command>firewall-drop</command>
    <location>local</location>
    <rules_id>100100</rules_id>
    <timeout>60</timeout>
  </active-response>

</ossec_config>
```

### Reiniciar el Servidor

Aplica todos los cambios reiniciando el manager de Wazuh.

```bash
systemctl restart wazuh-manager
```

---

## 3. Simulación y Verificación

### Ejecutar el "Ataque"

Desde el PC Windows (Atacante), intenta acceder al servidor web del PC Ubuntu.

```bash
# Puedes usar un navegador o la terminal (CMD/PowerShell)
curl http://192.168.1.130
```

### Verificar el Resultado

* Primer Intento: Debería funcionar. Verás la página de Apache.
* Segundo Intento (inmediato): Debería fallar (timeout / "la página no carga"). El bloqueo está activo.
* Tercer Intento (tras 60 segundos): Debería volver a funcionar, ya que el timeout de la respuesta activa ha expirado.

### Confirmación Final (Dashboard)

En el dashboard de Wazuh, verás la alerta de nivel 10 con la descripción "IP address found in AlienVault reputation database." y la regla 100100, confirmando que la IP 192.168.1.133 fue detectada.

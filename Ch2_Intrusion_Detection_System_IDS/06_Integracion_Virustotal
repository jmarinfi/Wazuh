# 06: Integración de Wazuh con VirusTotal para Detección de Malware

Este documento detalla la configuración y prueba de la integración de Wazuh con VirusTotal, enriqueciendo las alertas de FIM enviando automáticamente los hashes de archivos nuevos o modificados a VirusTotal para su análisis.

Laboratorio basado en la sección "VirusTotal integration" del libro, con ajustes prácticos y depuración.

## Infraestructura del Laboratorio

* **Servidor Wazuh:** Máquina Virtual (OVA) oficial de Wazuh.  
  **IP:** `192.168.1.136`
* **Endpoint (Agente):** PC anfitrión con Ubuntu (ubu-serv).  
  **IP:** `192.168.1.133`

---

## 1. Obtención de la Clave API de VirusTotal

1. Crear una cuenta gratuita en [VirusTotal.com](http://virustotal.com/).
2. Acceder al perfil y copiar la clave API.

---

## 2. Configuración del Servidor Wazuh (Manager)

### 2.1. Habilitar la Integración en `ossec.conf`

Editar `/var/ossec/etc/ossec.conf` y añadir:

```xml
<ossec_config>
  ...
  <integration>
    <name>virustotal</name>
    <api_key>TU_CLAVE_API_DE_VIRUSTOTAL</api_key>
    <rule_id>100200,100201</rule_id>
    <alert_format>json</alert_format>
  </integration>
  ...
</ossec_config>
```

> **Nota:** Se usa `<rule_id>` para limitar las llamadas a la API a eventos específicos.

### 2.2. Creación de Reglas Personalizadas

En `/var/ossec/etc/rules/local_rules.xml`:

```xml
<group name="syscheck,pci_dss_11.5,nist_800_53_SI.7,">
  <rule id="100200" level="7">
    <if_sid>550</if_sid>
    <field name="file" type="pcre2">^/root/</field>
    <description>File modified in /root directory. Triggering VirusTotal Scan.</description>
  </rule>
  <rule id="100201" level="7">
    <if_sid>554</if_sid>
    <field name="file" type="pcre2">^/root/</field>
    <description>File added to /root directory. Triggering VirusTotal Scan.</description>
  </rule>
</group>
```

> **Corrección:** Usar `type="pcre2"` y la expresión `^/root/` para que coincida con cualquier fichero en `/root`.

### 2.3. Reiniciar el Manager

```bash
sudo systemctl restart wazuh-manager
```

---

## 3. Configuración del Agente (ubu-serv)

Asegurarse de que FIM monitoriza `/root`. Editar `/var/ossec/etc/ossec.conf`:

```xml
<ossec_config>
  <syscheck>
    <disabled>no</disabled>
    ...
    <directories check_all="yes" report_changes="yes" realtime="yes">/root</directories>
    ...
  </syscheck>
</ossec_config>
```

Reiniciar el agente:

```bash
sudo systemctl restart wazuh-agent
```

---

## 4. Simulación, Depuración y Detección

### 4.1. Creación del Fichero de Prueba (EICAR)

En el agente, crear el fichero EICAR en `/root`:

```bash
touch /root/eicar
# Opcional: descargar EICAR
# curl -L -o /root/eicar https://secure.eicar.org/eicar.com
```

### 4.2. Problema de Interferencia de Reglas

Si solo aparece la alerta de la lista CDB (Regla 110002, Nivel 13) y no la esperada 100201 (Nivel 7), la integración de VirusTotal no se ejecuta.

> **Causa:** Ambas reglas dependen de la misma regla padre (554). Wazuh solo genera la alerta con el nivel más alto.

### 4.3. Solución y Verificación Final

Comentar temporalmente el grupo de reglas de la CDB list en `/var/ossec/etc/rules/local_rules.xml` del manager. Reiniciar el manager y repetir la prueba.

Alertas esperadas en el dashboard:

```text
rule.id: 100201
rule.level: 7
rule.description: File added to /root directory. Triggering VirusTotal Scan.
syscheck.path: /root/eicar
```

Alerta de integración VirusTotal (generada por el script tras el análisis):

```text
rule.id: 87105 (puede variar)
rule.level: 12 (si detecta malware)
rule.description: VirusTotal: Alert - /root/eicar - 65 engines detected this file
data.integration: virustotal
data.virustotal.positives: 65
```

---

## Conclusión

La integración con VirusTotal funciona correctamente tras dos ajustes clave:

1. Corregir la sintaxis del campo `field` en las reglas personalizadas para usar `pcre2` y asegurar coincidencia con la ruta completa.
2. Gestionar la interferencia entre reglas: reglas de mayor nivel pueden ocultar alertas de menor nivel si dependen del mismo evento padre. Ajustar niveles o condiciones según la lógica deseada.

Este laboratorio demuestra la capacidad de Wazuh para interactuar con servicios externos como VirusTotal, añadiendo inteligencia de amenazas al análisis de integridad de ficheros.

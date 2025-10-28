# 03: Pruebas de Detección NIDS con tmNIDS

Este documento describe la prueba final de la integración de Suricata utilizando el framework `tmNIDS`, como se detalla en la sección "Testing NIDS with tmNIDS" del libro. El objetivo es simular una variedad de ataques y tráfico malicioso para verificar que las reglas de Suricata se activan y las alertas se visualizan correctamente en el dashboard de Wazuh.

## Infraestructura del Laboratorio

A diferencia del laboratorio anterior con DVWA, esta prueba simplifica la configuración a dos dispositivos, como se muestra en la Figura 1.28 del libro:

* **Servidor Wazuh (VM):** `192.168.1.136`. Su rol es únicamente recibir y mostrar las alertas.
* **Monitor (Agente + Suricata):** `192.168.1.133`. Esta máquina (nuestro PC Ubuntu) ejecuta el agente, Suricata, y también el script de ataque `tmNIDS`.

En esta configuración, el script `tmNIDS` genera tráfico de ataque local, que es interceptado y analizado por el servicio local de Suricata.

---

## 1. Instalación de tmNIDS

`tmNIDS` es un script diseñado para probar las capacidades de detección de un NIDS.

1. **Ejecutar en el Monitor (PC Ubuntu):** Se descarga y ejecuta el script en el PC Ubuntu (`.133`).

2. **Corrección del Comando:** El comando de instalación provisto en el libro contiene errores tipográficos (un guion largo `–` en lugar de un guion corto `-`, y un símbolo `>` extra al final de la URL).

    El comando corregido y funcional es:

    ```bash
    curl -sSL [https://raw.githubusercontent.com/3CORESec/testmynids.org/master/tmNIDS](https://raw.githubusercontent.com/3CORESec/testmynids.org/master/tmNIDS) -o /tmp/tmNIDS && chmod +x /tmp/tmNIDS && /tmp/tmNIDS
    ```

3. **Menú Actualizado:** Al ejecutar el script, se observa un menú de opciones más actualizado que el que se muestra en el libro. En nuestra versión, hay 17 opciones, siendo la 16 "CHAOS! RUN ALL!".

---

## 2. Ejecución de Pruebas

Se ejecutan varias pruebas individuales para verificar la detección.

### Prueba 3: HTTP Malware User-Agent

Esta prueba simula una conexión HTTP con una cadena de `User-Agent` conocida por estar asociada a malware.

* **Resultado Esperado (Libro):** El libro muestra una alerta `ET POLICY GNU/LINUX APT User-Agent...`.
* **Resultado Obtenido:** La alerta recibida en el dashboard de Wazuh fue:
  * **rule.description:** `Suricata: Alert - ET INFO Delphi JEDI Visual Component Library User-Agent (JEDI-VCL)`

Esta discrepancia es normal. Las reglas de Emerging Threats y el script `tmNIDS` se actualizan. La prueba sigue siendo un éxito, ya que Suricata detectó correctamente un `User-Agent` anómalo y generó una alerta.

### Prueba 5: Tor .onion DNS response

Esta prueba simula una consulta DNS para un dominio `.onion` y una conexión a una IP conocida de un nodo Tor.

* **Resultado Obtenido:** La prueba fue un éxito y generó las dos alertas esperadas en el dashboard de Wazuh:
  * `Suricata: Alert - ET MALWARE Cryptowall .onion Proxy Domain`
  * `Suricata: Alert - ET POLICY DNS Query for TOR Hidden Domain .onion Accessible Via TOR`

### Prueba 16: "CHAOS! RUN ALL!"

Finalmente, se ejecuta la opción 16 para lanzar todas las pruebas de `tmNIDS` simultáneamente.

* **Resultado Obtenido:** El dashboard de Wazuh recibió una gran cantidad de alertas, confirmando que el ruleset de Suricata es efectivo. Algunas de las alertas detectadas incluyen:
  * `Suricata: Alert - ET MALWARE W32/LetsGo.APT Sleep Cnc Beacon`
  * `Suricata: Alert - ET ADWARE_PUP yupsearch.com Spyware Install`
  * `Suricata: Alert - ET ADWARE_PUP 180solutions (Zango) Spyware`
  * `Suricata: Alert - ET GAMES Second Life setup download`
  * `Suricata: Alert - ET INFO URL Shortener Domain in DNS Lookup (lk.tc)`
  * `Suricata: Alert - ET INFO URL Shortener Domain in DNS Lookup (cutt.ly)`
  * `Suricata: Alert - ET POLICY External IP Lookup ip-api.com`
  * `Suricata: Alert - ET INFO External IP Lookup Domain in DNS Lookup (ifconfig.io)`

## Conclusión

Todas las pruebas se completaron con éxito. La integración de Suricata con el agente de Wazuh funciona correctamente, y las alertas de red son centralizadas y visualizadas en el dashboard de Wazuh como se esperaba.

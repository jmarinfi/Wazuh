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
cd /tmp/ && curl -LO [https://rules.emergingthreats.net/open/suricata-6.0.8/emerging.rules.tar.gz](https://rules.emergingthreats.net/open/suricata-6.0.8/emerging.rules.tar.gz)
sudo tar -xvzf emerging.rules.tar.gz && sudo mkdir /etc/suricata/rules && sudo mv rules/*.rules /etc/suricata/rules/
sudo chmod 777 /etc/suricata/rules/*.rules

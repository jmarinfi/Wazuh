# Laboratorio de Casos de Uso de Wazuh

## Propósito del Repositorio

El objetivo principal de este repositorio es servir como una bitácora de aprendizaje y guía práctica para la plataforma de seguridad **Wazuh**.

La documentación se basa en los laboratorios y casos de uso del libro **"Security Monitoring with Wazuh"**, contrastados y complementados con la **documentación oficial de Wazuh** y las guías de integración de herramientas como **Suricata**.

La función de este espacio es documentar de forma detallada:

* El proceso de **instalación** de los componentes de Wazuh (Servidor, Indexer, Dashboard y Agentes).
* La **configuración** de servicios, reglas, decodificadores e integraciones.
* La exploración y puesta en marcha de distintos **casos de uso** para la detección de amenazas, cumplimiento y respuesta a incidentes.

## Contenido

A medida que se exploren nuevas funcionalidades, se añadirán nuevos documentos a este repositorio, cubriendo temas como:

* Integración con NIDS (Suricata).
* Monitorización de Integridad de Ficheros (FIM).
* Detección de malware y rootkits.
* Análisis de logs y respuesta activa.
* Evaluación de vulnerabilidades.

## Guía de Pruebas

* [01: Instalación del Servidor, Agente e Integración con Suricata](Intrusion_Detection_System_IDS/01_Instalacion_Integracion_Suricata.md)
* [02: Detección de Ataques Web (SQLi y XSS) con DVWA](Intrusion_Detection_System_IDS/02_Deteccion_Ataques_Web_DVWA.md)
* [03: Detección con tmNIDS y Wazuh](Intrusion_Detection_System_IDS/03_Deteccion_NIDS_con_tmNIDS.md)

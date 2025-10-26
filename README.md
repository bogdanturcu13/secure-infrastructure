# Laboratorio SOC: Monitorización y SIEM con Zabbix, Wazuh, Suricata y Grafana

Este proyecto despliega un entorno de laboratorio simulado para practicar tareas de Analista SOC (Security Operations Center), integrando herramientas líderes en monitorización de rendimiento (Zabbix), SIEM/HIDS (Wazuh), NIDS (Suricata) y visualización centralizada (Grafana). El despliegue se automatiza parcialmente mediante Ansible.

## 🎯 Objetivo

Crear un "Panel de Control Único" (Single Pane of Glass) en Grafana que muestre tanto métricas de rendimiento y disponibilidad de sistemas (procedentes de Zabbix) como alertas de seguridad relevantes (procedentes de Wazuh, que a su vez integra alertas de Suricata). Esto permite correlacionar eventos de rendimiento con posibles incidentes de seguridad en una sola vista.

## 🏗️ Arquitectura

El laboratorio consta de las siguientes máquinas virtuales (VMs), configuradas manualmente o mediante un método alternativo:

1.  **`ansible-automation` (192.168.56.101):** Nodo de control donde se ejecutan los playbooks de Ansible. También se utiliza como máquina "atacante" para simular tráfico malicioso (`nmap`).
2.  **`monitor-server` (192.168.56.102):** Servidor central que aloja:
    * **Zabbix Server:** Recolecta métricas de rendimiento.
    * **Wazuh Manager, Indexer & Dashboard:** Recolecta y analiza logs de seguridad (SIEM/HIDS).
    * **Suricata:** Sistema de Detección de Intrusiones en Red (NIDS) que analiza el tráfico de `enp0s3`.
    * **Grafana:** Plataforma de visualización que integra datos de Zabbix y Wazuh.
3.  **`web-server` (192.168.56.103):** Servidor simulado (Ubuntu base), monitorizado por Zabbix Agent y Wazuh Agent. *(Nota: Nginx no fue instalado en este paso)*.
4.  **`db-server` (192.168.56.104):** Servidor simulado (Ubuntu base), monitorizado por Zabbix Agent y Wazuh Agent. *(Nota: MySQL no fue instalado en este paso)*.


## 🛠️ Prerrequisitos

* Un entorno de virtualización (yo lo hice con VirtualBox) con las 4 VMs creadas y configuradas en la red `192.168.56.0/24`.
* Ansible instalado en la máquina `ansible-automation`.
* Conectividad SSH configurada desde `ansible-automation` a los otros nodos.

## 🚀 Despliegue Automatizado (Parcial)

1.  **Clonar Repositorio:**
    ```bash
    git clone https://github.com/bogdanturcu13/secure-infrastructure
    cd secure-infrastructure
    ```
2.  **Verificar Inventario:** Asegúrate de que el fichero `inventory.ini` contiene las IPs correctas de tus VMs.
3.  **Ejecutar Playbooks de Ansible:**
    * Ejecuta los playbooks en el orden adecuado desde `ansible-automation`:
        ```bash
        # Instalar componentes principales en monitor-server
        ansible-playbook playbooks/zabbix_install.yml
        ansible-playbook playbooks/wazuh_install.yml # (Este puede tardar bastante)
        ansible-playbook playbooks/suricata_install.yml
        ansible-playbook playbooks/grafana_install.yml

        # Instalar agentes Zabbix y Wazuh en nodos monitorizados
        
        ```

## ⚙️ Configuración Post-Instalación

1.  **Grafana:**
    * Accede a Grafana: `http://192.168.56.102:3000` (Usuario: `admin`, Contraseña: `admin`, cambiar al primer login).
    * **Añadir Fuente de Datos Zabbix:**
        * Tipo: Zabbix.
        * URL: `http://localhost/zabbix/api_jsonrpc.php` (o `http://192.168.56.102/zabbix/api_jsonrpc.php`).
        * Credenciales Zabbix API: (Usuario: `Admin`, Contraseña: `zabbix` - o la que hayas configurado).
        * Activar "Trends".
    * **Añadir Fuente de Datos Wazuh (Elasticsearch):**
        * Tipo: Elasticsearch.
        * URL: `https://192.168.56.102:9200` (o `https://localhost:9200`).
        * Autenticación Básica: `admin` / `<contraseñaWazuh>`. *(Recuperar con `sudo tar -xOf /home/ansible/wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt | grep -A 1 "admin"` en monitor-server)*.
        * Versión: `OpenSearch 2.x`.
        * Nombre del Índice: `wazuh-alerts-*`
        * Campo de Tiempo: `@timestamp`.
        * Desmarcar/Saltar verificación TLS (Skip TLS Verify).
    * **Importar Dashboard Zabbix:**
        * Ir a Dashboards -> Import.
        * Subir el fichero `grafana_dashboards/zabbix_dashboard.json`.
        * Seleccionar la fuente de datos Zabbix creada.
2.  **Añadir Panel Wazuh al Dashboard Zabbix:**
    * Abrir el dashboard de Zabbix importado.
    * Entrar en modo Edición.
    * Añadir un nuevo panel ("Visualization").
    * Seleccionar la fuente de datos **Elasticsearch (Wazuh)**.
    * Consulta Lucene: `rule.level: >= 3`.
    * Tipo de Visualización: **Table**.
    * Configurar Título (ej: "Alertas Wazuh/Suricata (Nivel 3+)") y opciones de tabla si se desea.
    * Aplicar y **Guardar** el dashboard.

## 🔗 Detalles de Integración

* **Zabbix:** Los Zabbix Agents en `web-server` y `db-server` envían métricas al Zabbix Server en `monitor-server`. Grafana consulta la API de Zabbix Server.
* **Wazuh:** Los Wazuh Agents en `web-server` y `db-server` envían logs al Wazuh Manager en `monitor-server`. El Manager analiza los logs, genera alertas y las almacena en Wazuh Indexer (OpenSearch).
* **Suricata:** Se ejecuta en `monitor-server`, inspeccionando el tráfico de la interfaz `enp0s3`. Escribe sus logs/alertas en `/var/log/suricata/eve.json`.
* **Suricata -> Wazuh:** El Wazuh Manager (configurado en `/var/ossec/etc/ossec.conf`) lee el fichero `/var/log/suricata/eve.json`, decodifica los eventos JSON y genera alertas Wazuh basadas en las reglas de Suricata.
* **Wazuh -> Grafana:** Grafana consulta directamente el Wazuh Indexer (vía la fuente de datos Elasticsearch/OpenSearch) para obtener las alertas `wazuh-alerts-*`.

## 🐛 Troubleshooting Destacado

Durante la configuración, se encontraron y resolvieron varios problemas críticos con Suricata y la visualización de alertas:

1.  **Suricata no generaba alertas:**
    * **Causa 1:** Configuración incorrecta de interfaz y `HOME_NET` en `suricata.yaml`.
    * **Solución 1:** Editar `suricata.yaml`, especificar `interface: enp0s3` en `af-packet`, comentar otras interfaces, definir `HOME_NET: "[192.168.56.0/24]"`.
    * **Causa 2:** Errores de sintaxis YAML durante ediciones.
    * **Solución 2:** Usar `journalctl -u suricata` para depurar y corregir `suricata.yaml`.
    * **Causa 3:** No cargaba reglas por falta de fuentes habilitadas en `suricata-update` y ausencia del enlace simbólico `/etc/suricata/rules/suricata.rules`.
    * **Solución 3:** Habilitar fuente `et/open`, actualizar reglas, crear directorio y enlace simbólico.
    * **Causa 4:** Permisos incorrectos en `/var/run/suricata/`.
    * **Solución 4:** Crear directorio y ajustar propietario (`chown suricata:suricata`).
    * **Causa 5:** Suricata no capturaba paquetes de la interfaz `enp0s3` a pesar de configuración y reglas correctas.
    * **Solución 5:** Habilitar **Modo Promiscuo ("Permitir Todo")** en la configuración de red de la VM `monitor-server` (VirtualBox/VMware) y reiniciar la VM. Confirmado con `tcpdump`.
2.  **Alertas de Suricata no aparecían en Wazuh/Grafana:**
    * **Causa 1:** Wazuh Manager no leía `/var/log/suricata/eve.json`.
    * **Solución 1:** Añadir bloque `<localfile>` en `/var/ossec/etc/ossec.conf` y reiniciar `wazuh-manager`.
    * **Causa 2 (Final):** Alertas de Suricata generadas (nivel 3) eran filtradas por los dashboards (`>= 5` o `>= 11`).
    * **Solución 2:** Ajustar consulta en Grafana a `rule.level: >= 3`.

## ✅ Simulación y Verificación

Para probar el flujo completo de alertas NIDS:

1.  **Instalar `nmap`** en `ansible-automation`:
    ```bash
    ssh ansible@192.168.56.101
    sudo apt update && sudo apt install nmap -y
    ```
2.  **Lanzar Escaneo:**
    ```bash
    nmap -sS -A 192.168.56.103
    ```
3.  **Observar Grafana:** Verificar que las alertas correspondientes (ej., "ET SCAN Nmap FIN Scan", "SURICATA ICMPv4 unknown code") aparecen en el panel "Alertas Wazuh/Suricata" del dashboard (con filtro `>= 3`).

## 🖥️ Uso

* **Dashboard Centralizado (Grafana):** `http://192.168.56.102:3000`
* **Dashboard Wazuh:** `https://192.168.56.102` (Usuario: `admin`, Contraseña: `<contraseña-wazuh-recuperada>`)
* **Interfaz Zabbix:** `http://192.168.56.102/zabbix` (Usuario: `Admin`, Contraseña: `zabbix`)

## 💡 Posibles Mejoras

* Crear playbooks específicos para instalar solo los agentes Zabbix/Wazuh.
* Configurar Wazuh para monitorizar logs específicos (ej., logs de sistema, `auth.log`).
* Afinar reglas de FIM en Wazuh.
* Crear reglas personalizadas en Wazuh.
* Integrar con Splunk u otro SIEM externo.
* Desarrollar roles de Ansible para una mejor organización.

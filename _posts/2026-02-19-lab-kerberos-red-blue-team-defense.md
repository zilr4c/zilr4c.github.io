---
title: "(3/3) Lab de Kerberos: Red Team vs Blue Team - Detección y Análisis de Logs"
date: 2026-02-22 08:00:00 +0100
categories: [Cybersecurity, Lab Setup - Purple Team]
tags: [active directory, sysmon, kerberoasting, as-rep roasting, windows server 2022, homelab, kerberos, john, impacket]
image:
  path: /assets/img/posts/kerberos-lab-defense/blue_team_defense_p.png
  alt: "Ataques Kerberos en Active Directory"
published: false
---

**ARQUITECTURA DEL ENTORNO**
  - **Endpoint (Víctima):** Windows Server 2022 (Controlador de Dominio).
  - **SIEM (Cerebro):** Ubuntu Server corriendo Splunk Enterprise.
  - **Transporte:** Splunk Universal Forwarder (UF) instalado en el Windows Server.

## Fase 1: Despliegue de la Infraestructura (Ubuntu SIEM)

### 1 SIEM (Ubuntu Server)
**El objetivo:** Mostrar cómo se levanta el "cerebro" de la defensa.
- **Lo que muestras:** Un par de capturas rápidas instalando **Ubuntu Server** (sin interfaz gráfica, para ser más profesional).
    
- **Configuración de Red:** Muestra la asignación de una **IP Estática**.
    - _Por qué es clave:_ "En un entorno real, el SIEM debe tener una dirección fija para que todos los agentes (Forwarders) sepan siempre a dónde enviar los logs".
        
- **Firewall (UFW):** Captura abriendo solo el puerto de Splunk (`9997/tcp`) y el de gestión (`8000/tcp`).
    - _Mensaje:_ "Aplicando el principio de mínimo privilegio desde la red".

### 2 Endpoint (Windows Server)
Aquí es donde instalas el **Splunk Universal Forwarder**.
- **Acción:** Instalación del agente (MSI) en el Domain Controller.
- **Punto clave:** Especificar que este agente es "ligero" y su único trabajo es enviar logs sin consumir apenas recursos del servidor.
- **Imagen sugerida:** Captura del instalador de Splunk UF o del servicio `splunkd` corriendo en Windows.

### 3 INGESTA DE DATOS: CONFIGURACIÓN DEL PIPELINE
Para que los eventos viajen desde el **Windows Server (Emisor)** hasta el **Ubuntu (Receptor)**, debemos configurar la "tubería" en ambos extremos.
#### A. En el Windows Server (Configurar el Universal Forwarder)
Debemos editar dos archivos en la ruta: `C:\Program Files\SplunkUniversalForwarder\etc\system\local\`

1. **Archivo `inputs.conf` (¿Qué recolectamos?):**
    Este archivo le dice al agente qué logs debe leer del sistema.
    ```powershell
    [WinEventLog://Microsoft-Windows-Sysmon/Operational]
    disabled = 0
    renderXml = 1
    start_from = oldest  # CLAVE: Lee el historial para la Fase 1 (Forensics)

    [WinEventLog://Security]
    disabled = 0
    start_from = oldest
    ```
    
1. **Archivo `outputs.conf` (¿A dónde lo enviamos?):**
    Este archivo establece la conexión con la máquina Ubuntu.
    ```powershell
    [tcpout]
    defaultGroup = grupo_ubuntu
    
    [tcpout:grupo_ubuntu]
    server = IP_DE_TU_UBUNTU:9997  # <--- Sustituye por la IP real de tu SIEM
    ```


#### B. En la máquina Ubuntu (Configurar Splunk Indexer)
Splunk viene cerrado por defecto. Debes abrir el puerto de escucha (Listening Port).

1. **Habilitar recepción vía Web:**
    - Ve a **Settings** > **Forwarding and receiving**.
    - En **Receive data**, haz clic en **+ Add new**.
    - Introduce el puerto `9997` y guarda.
        
2. **Abrir el Firewall de Ubuntu (UFW):**
    Ejecuta en la terminal para permitir la entrada de logs:

```bash
sudo ufw allow 9997/tcp
```
    

#### C. Verificación de la conexión
Una vez reinicies el servicio de Splunk en Windows, puedes verificar que la "tubería" está conectada ejecutando este comando en la PowerShell de Windows:

```powershell
netstat -ano | findstr 9997
```
> **Resultado esperado:** Si aparece una línea con **ESTABLISHED**, los datos están fluyendo hacia el SIEM en tiempo real.

### 4 Normalización: Uso de Add-ons específicos para parsear los eventos y prepararlos para la detección.

## Fase 2: ANÁLISIS FORENSE (Investigación Retrospectiva)
Demostrar que sabes encontrar la aguja en el pajar aunque el ataque ya haya pasado.

- **Query de búsqueda en Splunk (SPL):**
Fragmento de código
```bash
index=windows EventCode=4769 Ticket_Encryption_Type=0x17 Service_Name!=*$ | table _time, User, Client_Address, Service_Name, Ticket_Encryption_Type
```
    
- **Objetivo:** Identificar qué usuario pidió el ticket, desde qué IP y a qué hora exacta.
    
- **Argumento:** "Gracias a la ingesta de históricos, podemos realizar un análisis post-mortem para determinar el alcance de un compromiso previo a la instalación del SIEM."


## Fase 3: MONITOREO Y SOC (Detección en Tiempo Real)

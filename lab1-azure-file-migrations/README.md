# Lab 01: Azure File Sync Migration
## Hybrid Migration Lab Documentation

Proyecto de migración y modernización de servidores de archivos locales (*on-premises*), donde se establece una sincronización híbrida entre un servidor local (Windows Server) y la nube de Microsoft Azure. En este caso, se centralizan los ficheros mediante un **Azure File Share** conectado a través de **Azure File Sync**, logrando que los datos se repliquen de forma bidireccional. La infraestructura utiliza almacenamiento optimizado para garantizar la conectividad y el intercambio de ficheros, manteniendo la experiencia tradicional de acceso SMB para los usuarios. *(Nota arquitectónica: Las capacidades de Cloud Tiering se contemplaron a nivel de diseño teórico, enfocando la práctica en la validación de la sincronización bidireccional y la gestión de dependencias).*

## Objetive proyect
- Modernizar servidores de archivos locales heredados migrando los recursos compartidos a Microsoft Azure mediante **Azure File Sync**.
- Validar arquitecturas de sincronización bidireccional y conectividad híbrida en entornos de nube.
- Mantener la experiencia tradicional de uso compartido de archivos SMB y los permisos NTFS para los usuarios finales sin interrupción del servicio ni downtime.
- Demostrar experiencia práctica en entornos híbridos, administración de sistemas Windows Server y resolución de dependencias complejas mediante Azure PowerShell.

## Architecture
Componentes principales:
- **Windows Server (Entorno local simulado):** Actúa como el servidor de ficheros local (*edge file server*) de la oficina.
- **Azure File Share (`companydata`):** Almacenamiento centralizado en la nube con rendimiento optimizado para transacciones (*Transaction Optimized*).
- **Storage Sync Service & Sync Group (`companydata`):** Orquestan y gestionan la sincronización bidireccional entre el servidor local y Azure.
- **Recovery Services Vault (`vault-mwaqjjm4`):** Almacén utilizado para gestionar los elementos de respaldo y protección de los datos.

**Flujo:**  
Modificación/Creación de ficheros en servidor local (Windows Server) ⇄ Sincronización bidireccional mediante Azure File Sync ⇄ Almacenamiento seguro en Azure File Share.
<img width="674" height="330" alt="image" src="https://github.com/user-attachments/assets/5e390523-ce0e-4111-a2c3-e5052807885f" />

## Azure services used
- **Azure Storage Account:** Cuenta de propósito general v2 (Transaction Optimized) para alojar el recurso compartido (`stfilesynclab`).
- **Azure File Share:** Recurso compartido de archivos en la nube (`companydata`).
- **Storage Sync Service:** Servicio de Azure que coordina los grupos de sincronización (`sync-filesync`).
- **Azure Virtual Machines (Windows Server 2022 Datacenter):** Máquina virtual desplegada para simular el servidor físico de la empresa.
- **Recovery Services Vault:** Almacén de recuperación (`vault-mwaqjjm4`).

## Implementation steps
**step 1: creation of resource group and storage account**
1. create resource group (`rg-filesync`) en la región de *France Central*.
2. create storage account (`stfilesynclab`) con redundancia LRS y optimización para transacciones.
<img width="582" height="360" alt="image" src="https://github.com/user-attachments/assets/b2ece950-2e57-4606-aeed-72530b2d17fb" />
3. create Azure File Share (`companydata`) dentro de la cuenta de almacenamiento.
<img width="602" height="76" alt="image" src="https://github.com/user-attachments/assets/e6077842-18f0-4d96-9e81-8b2b32e31671" />

**step 2: deployment of local file server and synchronization infrastructure**
1. deploy Windows Server 2022 Virtual Machine para simular el entorno *on-premises*.
2. install Azure File Sync Agent en el servidor local.
<img width="442" height="372" alt="image" src="https://github.com/user-attachments/assets/de1fdcb2-175f-4d8c-93a1-4fb889bbff8a" />
3. register server conectando la máquina virtual con el **Storage Sync Service** en Azure.
<img width="598" height="393" alt="image" src="https://github.com/user-attachments/assets/50c8e420-dac0-4eed-8d85-76d2f7e6b3b2" />

**step 3: configuration of sync group and bidirectional sync**
1. create sync group (`companydata`) para asociar el recurso compartido de la nube.
<img width="547" height="452" alt="image" src="https://github.com/user-attachments/assets/4eaff656-99f6-4747-aad0-631102887ac2" />
2. configure Cloud Endpoint y Server Endpoint para establecer la ruta de sincronización de ficheros.
<img width="536" height="133" alt="image" src="https://github.com/user-attachments/assets/b9fdec23-cb0e-48f8-a23a-87819b940be4" />
3. validate bidirectional sync asegurando el flujo correcto de datos entre el servidor local y el recurso compartido de Azure.
<img width="535" height="352" alt="image" src="https://github.com/user-attachments/assets/edcc2587-8a1a-4569-9c6d-3e40a28d0502" />
<img width="523" height="247" alt="image" src="https://github.com/user-attachments/assets/eca97bf7-bfa3-4a1e-bf56-e7697bed9117" />

## Challenges and Solutions

| Reto / Problema Encontrado | Causa Raíz | Solución Aplicada |
| :--- | :--- | :--- |
| **Problemas de conexión remota a la VM** | Restricciones de reglas de red, puertos de acceso remoto cerrados o validación inicial de credenciales en el Windows Server desplegado. | Verificación de los grupos de seguridad de red (NSG) en Azure, validación de reglas de firewall y reintento de conexión remota con las credenciales correctas del sistema. |
| **Fallos durante la instalación del Agente / Configuración** | Restricciones de seguridad del navegador o políticas de ejecución en el servidor local que bloqueaban la descarga o el registro inicial. | Configuración de las opciones de seguridad del navegador en el servidor local y ejecución de los prerrequisitos del agente de Azure File Sync paso a paso. |
| **Errores de sintaxis y rutas en PowerShell** | Diferencias en los nombres de variables o rutas de recursos al ejecutar cmdlets de Azure File Sync en Cloud Shell. | Ajuste de variables de entorno (`$resourceGroup`, `$syncService`, `$syncGroup`) y validación de la existencia de la jerarquía antes de invocar los comandos de búsqueda de endpoints. |
| **Bloqueo en la eliminación del Recovery Services Vault** | Presencia de contenedores de respaldo registrados (`stfilesynclab`), directivas de *Soft Delete* y falta de contexto en la consola. | 1. Asignación del contexto con `Set-AzRecoveryServicesVaultContext -Vault $vault`.<br>2. Purgado de datos de respaldo con `Disable-AzRecoveryServicesBackupProtection -RemoveRecoveryPoints -Force`.<br>3. Desregistro del contenedor con `Unregister-AzRecoveryServicesBackupContainer -Force`. |

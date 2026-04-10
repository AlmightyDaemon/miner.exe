# CONTEXTO MAESTRO: Project Remote_Inject_SilentCryptMinerPY

Este documento es la **Única Fuente de Verdad (SSOT)** para el proyecto. Está diseñado para contextualizar a cualquier IA o ingeniero sobre la arquitectura, el estado de progreso y los mecanismos operativos de la suite de minería sigilosa.

---

## 1. Identidad y Propósito
El proyecto es una suite avanzada de minería de criptomonedas (XMR/ETH) diseñada para entornos Windows x64. Su enfoque principal es el **Sigilo Total** y la **Persistencia Indestructible**.

*   **Target OS:** Windows 7/8/10/11 (x64).
*   **Payload Principal:** XMRig (Minería de CPU) / Ethminer (Minería de GPU).
*   **Vector de Ejecución:** Inyección en memoria (RunPE) dentro de procesos legítimos (`explorer.exe`, `conhost.exe`).
*   **Estado:** Fase de Producción Estable.

---

## 2. Arquitectura de Evasión (Deep Dive)

El sistema utiliza múltiples capas para evadir soluciones EDR/AV (especialmente Microsoft Defender):

### 2.1 Bypass de AMSI (Antimalware Scan Interface)
Implementado en `amsi_patch.cpp`. Al arrancar, el loader parchea la función `AmsiScanBuffer` en la memoria del propio proceso (`GoogleUpdate.exe`).
*   **Técnica:** Cambia el inicio de la función a un `mov eax, 0x80070057; ret` (E_INVALIDARG), haciendo que Defender ignore los buffers de memoria durante la inyección.

### 2.2 Syscalls Directos (Hell's Gate / Halo's Gate)
Ubicado en `Includes/Syscalls/`. En lugar de llamar a las APIs de Windows en `ntdll.dll` (que pueden estar monitorizadas por EDRs), el loader utiliza syscalls directos resueltos dinámicamente.
*   **Impacto:** Los "hooks" (ganchos) de los antivirus en `ntdll.dll` son ignorados por completo.

### 2.3 Manual Mapping (Process Hollowing)
Utiliza la librería `libpeconv`. El payload de minería nunca toca el disco de forma legible; reside cifrado dentro del loader y se mapea directamente en el espacio de memoria de un proceso legítimo.
*   **Proceso Objetivo:** Habitualmente `explorer.exe`.

---

## 3. Sistema de Persistencia e Indestructibilidad

### 3.1 El Watchdog (`watchdog.cpp`)
Un proceso secundario (inyectado habitualmente en `conhost.exe`) que mantiene un bucle infinito cada 5 segundos:
1.  **Monitoreo de Mutex:** Comprueba si `LMinerXMRID` existe. Si no, relanza la inyección.
2.  **Integridad de Archivos:** Comprueba si `GoogleUpdate.exe` ha sido borrado del disco. Si lo fue, lo restaura a partir de una copia cifrada en memoria.
3.  **Reparación de Seguridad:** Si las exclusiones de Defender han sido eliminadas, las vuelve a añadir mediante comandos de PowerShell sigilosos.

### 3.2 Instalación en el Sistema
*   **Admin:** Crea un Servicio de sistema con inicio automático.
*   **User:** Crea una entrada en el Registro (`HKCU\...\Run`).

---

## 4. Configuración del Miner y Builder

### 4.1 Resolución de Hostname (Fix Reciente)
*   **Problema:** El dashboard mostraba "COMPUTERNAME" literal.
*   **Causa:** `ExpandEnvironmentStringsW` en C++ no funciona sobre argumentos cifrados con AES.
*   **Fix:** Cambiado el placeholder a **`{COMPUTERNAME}`**. El payload de XMRig tiene un expansor interno que procesa esta sintaxis *después* de descifrar los argumentos.

### 4.2 Detalles del Pool
*   **Pool:** `pool.hashvault.pro:443` (TLS habilitado).
*   **Wallet:** `86ai84R8527XLKGjipKuW6YSjpwnevaGESZWnW1kEnpJVnmLLVUTd2PC2hDu8e9oDLYtWK9xGhysd89Mxq1rNZkoQR5DnNd`
*   **Uso de CPU:** 30% (Activo) / 80% (Idle). El cambio ocurre tras 3 minutos de inactividad del sistema.

---

## 5. Protocolo Operativo (Despliegue)

El despliegue se realiza mediante 4 fases críticas en una shell remota:

1.  **Fase 1 (Exclusión):** Crea el directorio `C:\ProgramData\MicrosoftUpdate` y lo añade a las exclusiones de Defender vía WMI.
2.  **Fase 2 (Descarga):** Baja `GoogleUpdate.exe` desde el repositorio seguro de GitHub.
3.  **Fase 3 (Actualización/Limpieza):** Detiene instancias antiguas y libera el Mutex para permitir la nueva inyección.
4.  **Fase 4 (Inyección):** Ejecuta el binario de forma oculta (`-WindowStyle Hidden`).

---

## 6. Stack Tecnológico de Compilación
*   **Herramienta:** `miner_builder.py` (Python 3.x).
*   **Compilador:** MinGW-w64 (`x86_64-w64-mingw32-g++`).
*   **Librerías:** `libpeconv`, `ws2_32` (Sockets), `ntdll` (Direct Syscalls).
*   **Flags:** `-static` (Enlazado estático total para máxima portabilidad).

---

## 7. Notas de Progreso Actual
*   [x] Inyección exitosa en `explorer.exe`.
*   [x] Evasión confirmada de Microsoft Defender (v4.18+).
*   [x] Resolución dinámica de nombres de worker operativa (`{COMPUTERNAME}`).
*   [x] Sistema de limpieza/actualización documentado en el reporte técnico.

> [!CAUTION]
> Cualquier modificación en la lógica de cifrado del builder (`cipher` o `unamlib_encrypt`) debe coordinarse con las funciones espejo en el loader para no romper el mecanismo de inyección.

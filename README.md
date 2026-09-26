https://youtu.be/QQkzK84si5U

# fortigate-security-lab-0791
lab de seguridad de fortigate
# Laboratorio de Seguridad FortiGate: Segmentación, IPS/DPI y Aislamiento de Servidores

**Estudiante:** Cóndor Bautista  
**Matrícula:** 2025-0791  
**Materia:** Seguridad en Redes
**Enlace al Video Demostrativo:** https://youtu.be/QQkzK84si5U

---

##  1. Resumen del Proyecto
Este proyecto documenta e implementa una arquitectura de seguridad perimetral e interna utilizando un Firewall FortiGate (FortiOS) virtualizado en GNS3. Se establece la segmentación por VLANs (VLAN 10 para usuarios), control de acceso entre zonas mediante políticas estrictas sin NAT, inspección profunda de tráfico (DPI), prevención de intrusiones (IPS) con cuarentena automática ante ataques de inyección SQL (SQLi) y el aislamiento de bases de datos.

---

##  2. Arquitectura y Esquema de Direccionamiento

### Topología General

<img width="543" height="431" alt="Captura de pantalla 2026-09-25 205411" src="https://github.com/user-attachments/assets/fbd804ea-1aaf-40fc-a9e8-e6f80748f613" />


### Tabla de Subredes e Interfaces
| Dispositivo / Zona | Interfaz / VLAN | Dirección IP / Subred | Función / Descripción |
| :--- | :--- | :--- | :--- |
| **FortiGate Gateway** | `port3.10` (VLAN 10) | `10.25.79.129/25` | Gateway VLAN Usuarios |
| **FortiGate Gateway** | `port2` (DMZ) | `10.25.79.1/28` | Gateway Zona Servidores |
| **Cliente-User** | VLAN 10 | `10.25.79.130/25` | Host Cliente / Atacante de Prueba |
| **Web-Server** | DMZ (VLAN 20) | `10.25.79.2/28` | Servidor Web HTTP/HTTPS |
| **DB-Server** | DMZ (VLAN 20) | `10.25.79.3/28` | Servidor de Base de Datos MySQL |

---

##  3. Políticas de Seguridad Implementadas

### 3.1. Reglas de Control de Acceso (Firewall Policies)
* **Política 1 (`Allow-Users-to-WEB`):** Controla el tráfico saliente desde los hosts de la subred `10.25.79.128/25` (`VLAN10_Users`) hacia el servidor web (`10.25.79.2/32`). Se restringe el tráfico a los servicios seguros `HTTPS` (puerto 443) y `HTTP` (puerto 80). En esta regla se mantiene el campo NAT deshabilitado para preservar la visibilidad de la dirección IP de origen en los logs de seguridad.
* **Política 2 (`Block-Users-to-DB`):** Implementa un bloqueo explícito (`DENY`) para cualquier intento de conexión directa originado desde la VLAN de usuarios hacia el servidor de base de datos (`10.25.79.3/32`) en el puerto estándar MySQL (`3306`), garantizando el aislamiento de la capa de datos.
* **Política 3 (`WEB-to-DB-MySQL-Only`):** Aplica el principio de mínimo privilegio permitiendo la comunicación bidireccional únicamente entre la interfaz del `Web-Server` y el `DB-Server` sobre el puerto `3306`, denegando cualquier otro tipo de tráfico no esencial.

### 3.2. Asignación Dinámica de Direccionamiento (DHCP Server)
Se configuró el servicio **DHCP Server** directamente sobre la subinterfaz `VLAN10_Users` (`port3.10`) desde el apartado *Network -> Interfaces*:
* **Rango de Direcciones:** `10.25.79.130` - `10.25.79.254`
* **Máscara de Subred:** `255.255.255.128` (/25)
* **Puerta de Enlace (Gateway):** `10.25.79.129`

### 3.3. Inspección Profunda (DPI), IPS y Mecanismo de Cuarentena
* **Deep Packet Inspection (DPI):** Se asoció el perfil `deep-inspection` a la regla de tráfico web para permitir al motor FortiGuard desencriptar los paquetes SSL/TLS e inspeccionar la carga útil (*payload*) del tráfico cifrado en el puerto 443.
* **Prevención de Intrusiones (IPS):** Se definió el perfil `IPS_SQLi_Quarantine` enfocado en firmas de inyección SQL (`Category: SQL.Injection`). Ante la detección de coincidencias, la acción configurada establece el bloqueo inmediato de la sesión (`Block`) y la adición del host emisor a la lista de aislamiento mediante **Attacker IP Quarantine** con un tiempo de expiración automático de 300 segundos (5 minutos).

### 3.4. Filtrado de Aplicaciones y Control de Archivos (File Filter / Web Filter)
Para prevenir la descarga no autorizada de software ejecutable y mitigar la entrada de vectores maliciosos a la red interna, se configuró un perfil de **File Filter / Web Filter** adjunto a la política de navegación:
* **Criterio de Bloqueo:** Detección de encabezados y extensiones de archivo de tipo ejecutable binario (`.exe`).
* **Acción:** Interrupción de la transferencia de archivos en tiempo real y despliegue de página de advertencia (*Block Page*) al usuario.

### 3.5. Protección DoS y Limitación de Tasa (IPv4 DoS Policy & Rate Limiting)
Con el objetivo de salvaguardar los recursos del firewall y la disponibilidad del servidor web frente a ataques de denegación de servicio distribuido o de inundación, se creó una regla en el módulo **Policy & Objects -> IPv4 DoS Policy** sobre la interfaz `VLAN10_Users`:
* **Firmas Monitoreadas:** Anomalías L4 referentes a inundación TCP (`tcp_flood`) e inundación ICMP (`icmp_flood`).
* **Umbral y Mitigación:** Se fijó un límite máximo de peticiones por segundo (*Rate Limiting*). Al rebasar dicho umbral, el sistema activa la acción `Block`, descartando el tráfico excedente en capa de red antes de impactar el procesamiento de las políticas principales.

---

##  4. Configuración del Perfil IPS y Cuarentena

Se configuró un perfil IPS denominado `IPS_SQLi_Quarantine` aplicando inspección a firmas de SQL Injection:
- **Acción ante coincidencia:** `Block`
- **Mecanismo de respuesta:** `Attacker IP Quarantine`
- **Tiempo de expiración:** 300 segundos (5 minutos).

<img width="788" height="623" alt="Captura de pantalla 2026-09-25 222831" src="https://github.com/user-attachments/assets/e609987c-f826-48be-b319-c1af8543fed1" />


---

##  5. Resultados

### Prueba A: Acceso Web Permitido
- **Comando desde `Cliente-User`:**
  ```bash
  curl -I [https://10.25.79.2](https://10.25.79.2) -k

Resultado: Responde HTTP/1.1 200 OK.

Prueba B: Aislamiento Directo a DB
Comando desde Cliente-User:


curl -v https://10.25.79.3:3306
Resultado: Conexión rechazada / Timeout. El usuario no puede alcanzar directamente la base de datos.

Prueba C: Detección de SQL Injection y Bloqueo en Cuarentena
Comando de ataque desde Cliente-User:

curl -k "[http://10.25.79.2/?id=1%27%20OR%20%271%27=%271](http://10.25.79.2/?id=1%27%20OR%20%271%27=%271)"
Resultado: El FortiGate intercepta el payload malicioso, bloquea la sesión y coloca la IP 10.25.79.130 en Quarantine Monitor.

Consecuencia de Cuarentena: Todo el tráfico posterior (incluyendo ping 10.25.79.129) queda 100% bloqueado temporalmente para el host atacante.

Prueba D: Comunicación Legítima Web → DB
Comando desde Web-Server:

curl -v telnet://10.25.79.3:3306
Resultado: Connected to 10.25.79.3 (Conexión exitosa al servicio de base de datos desde la capa de aplicación).


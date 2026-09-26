# fortigate-security-lab-0791
lab de seguridad de fortigate
# Laboratorio de Seguridad FortiGate: Segmentación, IPS/DPI y Aislamiento de Servidores

**Estudiante:** Cóndor Bautista  
**Matrícula:** 2025-0791  
**Materia:** Seguridad en Redes
**Enlace al Video Demostrativo:** https://youtu.be/QQkzK84si5U

---

## 📌 1. Resumen del Proyecto
Este proyecto documenta e implementa una arquitectura de seguridad perimetral e interna utilizando un Firewall FortiGate (FortiOS) virtualizado en GNS3. Se establece la segmentación por VLANs (VLAN 10 para usuarios), control de acceso entre zonas mediante políticas estrictas sin NAT, inspección profunda de tráfico (DPI), prevención de intrusiones (IPS) con cuarentena automática ante ataques de inyección SQL (SQLi) y el aislamiento de bases de datos.

---

## 📐 2. Arquitectura y Esquema de Direccionamiento

### Topología General
![Topología GNS3](images/topologia.png)

### Tabla de Subredes e Interfaces
| Dispositivo / Zona | Interfaz / VLAN | Dirección IP / Subred | Función / Descripción |
| :--- | :--- | :--- | :--- |
| **FortiGate Gateway** | `port3.10` (VLAN 10) | `10.25.79.129/25` | Gateway VLAN Usuarios |
| **FortiGate Gateway** | `port2` (DMZ) | `10.25.79.1/28` | Gateway Zona Servidores |
| **Cliente-User** | VLAN 10 | `10.25.79.130/25` | Host Cliente / Atacante de Prueba |
| **Web-Server** | DMZ (VLAN 20) | `10.25.79.2/28` | Servidor Web HTTP/HTTPS |
| **DB-Server** | DMZ (VLAN 20) | `10.25.79.3/28` | Servidor de Base de Datos MySQL |

---

## 🔒 3. Políticas de Seguridad Implementadas

1. **Permitir Acceso Web (Users $\rightarrow$ Web-Server):**
   - **Origen:** `10.25.79.128/25` (`VLAN10_Users`)
   - **Destino:** `10.25.79.2/32` (`Web-Server`)
   - **Servicios:** HTTP, HTTPS, PING
   - **NAT:** Desactivado (mantiene visibilidad de la IP real del cliente)
   - **Seguridad:** Perfil IPS con Cuarentena y SSL Deep Inspection activado.

2. **Aislamiento Directo a Base de Datos (Users $\rightarrow$ DB-Server):**
   - **Origen:** `10.25.79.128/25`
   - **Destino:** `10.25.79.3/32` (`DB-Server`)
   - **Acción:** `DENY` (Bloqueo explícito del acceso directo desde clientes).

3. **Acceso Aplicativo Interno (Web-Server $\rightarrow$ DB-Server):**
   - **Origen:** `10.25.79.2/32`
   - **Destino:** `10.25.79.3/32`
   - **Servicio:** MySQL (`3306`)
   - **Acción:** `ACCEPT`.

---

## 🛡️ 4. Configuración del Perfil IPS y Cuarentena

Se configuró un perfil IPS denominado `IPS_SQLi_Quarantine` aplicando inspección a firmas de SQL Injection:
- **Acción ante coincidencia:** `Block`
- **Mecanismo de respuesta:** `Attacker IP Quarantine`
- **Tiempo de expiración:** 300 segundos (5 minutos).

![Perfil IPS](images/ips_profile.png)

---

## 🧪 5. Batería de Pruebas y Resultados

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

📦 6. Archivos Adjuntos
Backup de Configuración de FortiGate: 



---

### 5. Archivo de Entrega `.txt`

Recuerda crear el archivo `CondorBautista_20250791_P1.txt` para subir a la plataforma institucional con este texto básico:

```text
Nombre: Cóndor Bautista
Matrícula: 2025-0791
Materia: Seguridad en Redes

Enlace al Repositorio de GitHub:
https://github.com/TU_USUARIO/fortigate-security-lab-0791

Enlace al Video Demostrativo en YouTube:
https://youtu.be/TU_CODIGO_DE_VIDEO


  

# Guía Técnica: Configuración de VPN Site-to-Site con pfSense en Entorno Virtualizado

## 1. Introducción

El presente documento describe el procedimiento técnico para configurar una **VPN Site-to-Site** utilizando **pfSense** en un entorno virtualizado.  
El objetivo es establecer una conexión segura entre dos redes privadas simuladas, representadas como **Sede ACME A** y **Sede ACME B**, garantizando el cifrado y la autenticación mediante el protocolo **OpenVPN en modo SSL/TLS**.

Esta práctica permite comprender los principios de interconexión segura entre sedes, la gestión de certificados digitales, las reglas de firewall asociadas y la verificación del tráfico entre segmentos de red aislados.
---
## 2. Requisitos del Entorno

### 2.1 Software Necesario

| Herramienta       | Versión recomendada | Función |
|-------------------|--------------------|----------|
| pfSense           | 2.7.x o superior   | Firewall y servidor VPN |
| VirtualBox / VMware | Última versión | Virtualización de red |
| OpenVPN           | Integrado en pfSense | Servicio VPN |
| Navegador Web     | Cualquiera         | Acceso a la interfaz web de pfSense |
| Servicio DDNS (DuckDNS, No-IP, DynDNS) | — | Resolución dinámica del servidor VPN |

---

### 2.2 Configuración de Máquinas Virtuales

Cada sede contará con una máquina virtual pfSense configurada con **dos adaptadores de red**:

| Interfaz | Tipo de Adaptador | Función |
|-----------|-------------------|----------|
| **WAN**   | Red NAT o Bridge | Conexión simulada a Internet |
| **LAN**   | Red interna o host-only | Red local interna de la sede |

**Sede ACME A:**
- WAN: 10.0.10.2 (gateway 10.0.10.1)
- LAN: 192.168.10.1/24

**Sede ACME B:**
- WAN: 10.0.20.2 (gateway 10.0.20.1)
- LAN: 192.168.20.1/24

---

## 3. Configuración de DDNS en la Sede ACME A (Servidor VPN)

1. Ir a **Services → Dynamic DNS → Add**.
2. Seleccionar el proveedor (por ejemplo, **DuckDNS** o **No-IP**).
3. Configurar:
   - **Service Type:** DuckDNS (o el proveedor elegido)
   - **Interface to monitor:** WAN
   - **Hostname:** acmevpn.duckdns.org
   - **Token o clave:** (proporcionado por el servicio DDNS)
4. Guardar y aplicar los cambios.
5. Verificar que el estado indique **Updated successfully**.

> **Importante:**  
> El DDNS debe apuntar correctamente a la IP WAN del pfSense servidor, y el certificado SSL debe emitirse con ese mismo nombre de dominio.

---

## 4. Configuración en la Sede ACME A (Servidor OpenVPN)

### 4.1 Creación de la Autoridad Certificadora (CA)
1. Ir a **System → Cert. Manager → CAs → Add**.
2. Configurar:
   - **Descriptive Name:** ACME_CA
   - **Method:** Create an internal Certificate Authority
   - **Key Length:** 2048 bits
   - **Digest Algorithm:** SHA256
   - **Lifetime:** 3650 días
3. Guardar los cambios.

---

### 4.2 Creación del Certificado del Servidor
1. En **System → Cert. Manager → Certificates → Add/Sign**:
   - **Method:** Create an internal certificate
   - **Descriptive Name:** ACME_Server
   - **Certificate Authority:** ACME_CA
   - **Type:** Server Certificate
   - **Common Name (CN):** acmevpn.duckdns.org
2. Guardar.

---

### 4.3 Configuración del Servidor OpenVPN
1. Ir a **VPN → OpenVPN → Servers → Add**.
2. Parámetros:
   - **Server Mode:** Peer to Peer (SSL/TLS)
   - **Protocol:** UDP
   - **Port:** 1194
   - **Tunnel Network:** 172.16.100.0/30
   - **Local Network:** 192.168.10.0/24
   - **TLS Authentication:** Enabled
   - **Server Certificate:** ACME_Server
3. Guardar y aplicar.

---

## 5. Configuración de la Sede ACME B (Cliente OpenVPN)

### 5.1 Certificados de Cliente
1. En la Sede A, ir a **System → Cert. Manager → Certificates → Add/Sign**:
   - **Descriptive Name:** ACME_Client_B
   - **Certificate Authority:** ACME_CA
   - **Type:** Client Certificate
2. Exportar la CA y los certificados (formato `.crt` y `.key`).
3. En la Sede B, importar en:
   - **System → Cert. Manager → CAs → Import**
   - **System → Cert. Manager → Certificates → Import**

---

### 5.2 Configuración del Cliente OpenVPN
1. En **VPN → OpenVPN → Clients → Add**:
   - **Server host or address:** acmevpn.duckdns.org
   - **Port:** 1194
   - **Protocol:** UDP
   - **Server Mode:** Peer to Peer (SSL/TLS)
   - **Peer Certificate Authority:** ACME_CA
   - **Client Certificate:** ACME_Client_B
   - **Tunnel Network:** 172.16.100.0/30
   - **Remote Network:** 192.168.10.0/24
   - **Local Network:** 192.168.20.0/24
2. Guardar y habilitar el cliente.

---

## 6. Reglas de Firewall y Ruteo

En ambas sedes:
1. Ir a **Firewall → Rules → OpenVPN → Add**.
2. Configurar:
   - **Action:** Pass  
   - **Protocol:** Any  
   - **Source:** Any  
   - **Destination:** Any  
   - **Description:** Permitir tráfico a través del túnel  
3. Aplicar los cambios.

En **Firewall → Rules → LAN**, agregar reglas que permitan tráfico hacia la red remota correspondiente (192.168.10.0 ↔ 192.168.20.0).

---

## 7. Verificación de Conectividad

### 7.1 Estado del túnel
- En **Status → OpenVPN**, ambos lados deben mostrarse como **up**.

### 7.2 Prueba de comunicación entre sedes
Desde pfSense A:
```bash
ping 192.168.20.1
````

Desde pfSense B:

```bash
ping 192.168.10.1
```

Si la respuesta es exitosa, el túnel VPN está operativo.

---

## 8. Errores Comunes

| Error                      | Causa probable                                | Solución                                                            |
| -------------------------- | --------------------------------------------- | ------------------------------------------------------------------- |
| El cliente no se conecta   | IP o nombre DDNS incorrecto                   | Verificar el nombre de host y la actualización DDNS                 |
| Error de certificado       | El CN del certificado no coincide con el DDNS | Regenerar el certificado con el nombre correcto                     |
| Sin respuesta de ping      | Falta de reglas en OpenVPN o LAN              | Revisar reglas de firewall                                          |
| Estado “Connection failed” | Puertos bloqueados o error de NAT             | Verificar reenvío del puerto 1194 en el router físico o NAT virtual |

---

```

¿Deseas que le agregue una sección al final con **capturas esperadas** o **comandos de diagnóstico (como netstat, traceroute, etc.)** para completar el informe?
```

# Guía Técnica: Configuración de VPN Site-to-Site con pfSense en Entorno Virtualizado

## 1. Introducción

El presente documento describe el procedimiento técnico para configurar una **VPN Site-to-Site** utilizando **pfSense** en un entorno virtualizado.  
El objetivo es establecer una conexión segura entre dos redes privadas simuladas, representadas como **Sede ACME A** y **Sede ACME B**, garantizando el cifrado y la autenticación mediante el protocolo **OpenVPN en modo SSL/TLS**.

### Teoría
Una **VPN site-to-site** permite la comunicación segura entre dos redes remotas a través de Internet, cifrando los datos transmitidos entre ellas.  
El uso de **SSL/TLS (Secure Sockets Layer / Transport Layer Security)** garantiza que los datos viajen cifrados y autenticados mediante certificados digitales, evitando accesos no autorizados o interceptaciones.  
pfSense es una solución de firewall y enrutador basada en FreeBSD, ampliamente utilizada para implementar VPNs de manera segura y administrable.

---

## 2. Requisitos del Entorno

### Teoría
Antes de la implementación, es necesario contar con un entorno virtualizado que simule dos redes independientes conectadas mediante Internet. Cada red representará una sede distinta, con su propio firewall pfSense.

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

### Teoría
El **Dynamic DNS (DDNS)** es un servicio que asocia una dirección IP dinámica a un nombre de dominio. Esto permite acceder al servidor VPN utilizando un dominio (por ejemplo, `acmevpn.duckdns.org`), incluso si la IP pública cambia.  
Solo **una de las sedes** —la que actúa como servidor VPN— necesita configurar DDNS, ya que el cliente se conectará a ese nombre.

> **Nota:**  
> Si se usa OpenVPN con SSL/TLS, el certificado del servidor debe coincidir con el nombre del DDNS configurado. De lo contrario, el cliente no podrá autenticar el servidor correctamente.

### Procedimiento
1. Ir a **Services → Dynamic DNS → Add**.
2. Seleccionar el proveedor (por ejemplo, **DuckDNS** o **No-IP**).
3. Configurar:
   - **Service Type:** DuckDNS (o el proveedor elegido)
   - **Interface to monitor:** WAN
   - **Hostname:** acmevpn.duckdns.org
   - **Token o clave:** (proporcionado por el servicio DDNS)
4. Guardar y aplicar los cambios.
5. Verificar que el estado indique **Updated successfully**.

---

## 4. Configuración en la Sede ACME A (Servidor OpenVPN)

### Teoría
El servidor OpenVPN será el encargado de aceptar las conexiones entrantes del cliente remoto (Sede B).  
Se utilizará una **Autoridad Certificadora (CA)** interna para emitir certificados SSL/TLS, garantizando la identidad de ambos extremos de la VPN.

---

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

### Teoría
El cliente OpenVPN establece la conexión hacia el servidor usando el dominio del DDNS configurado.  
Debe importar los certificados generados por la CA del servidor para autenticar la conexión SSL/TLS.

---

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

### Teoría
El firewall de pfSense controla todo el tráfico que pasa por el túnel VPN.  
Se deben crear reglas específicas en las interfaces **OpenVPN** y **LAN** para permitir la comunicación entre las redes locales de ambas sedes.

---

### Procedimiento
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

### Teoría
Una vez configurado el túnel, se deben realizar pruebas de conectividad para confirmar que las redes pueden comunicarse de forma cifrada y estable.

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

# Configuración del Túnel VPN Site-to-Site entre Sede ACME A y Sede ACME B

## 1. Descripción General

El objetivo de esta configuración es establecer una conexión segura entre dos sedes virtuales de la empresa **ACME** mediante un túnel **VPN site-to-site** implementado con **pfSense**.  
El túnel se configura bajo el modo **Peer-to-Peer (SSL/TLS)** utilizando **OpenVPN**, lo que permite el cifrado y autenticación mediante certificados digitales, garantizando la integridad y confidencialidad de las comunicaciones entre ambas redes.

Esta interconexión asegura que los dispositivos pertenecientes a las subredes locales de cada sede puedan comunicarse de manera segura y controlada, respetando las políticas de segmentación y aislamiento definidas por VLAN.

---

## 2. Parámetros de Red

| Elemento        | Sede ACME A              | Sede ACME B              |
|-----------------|--------------------------|--------------------------|
| pfSense         | 10.0.20.27               | 10.0.30.27               |
| Red local       | 10.0.20.0/24             | 10.0.30.0/24             |
| Tipo de VPN     | OpenVPN (Peer to Peer - SSL/TLS) | OpenVPN (Peer to Peer - SSL/TLS) |
| Protocolo       | UDP                      | UDP                      |
| Puerto          | 1194                     | 1194                     |
| Red del túnel   | 172.16.1.0/30            | 172.16.1.0/30            |

---

## 3. Configuración en pfSense ACME A (Servidor OpenVPN)

1. Ingresar a **VPN → OpenVPN → Wizards**.  
2. Crear una nueva **CA (Certificate Authority)** denominada `ACME_CA`.  
3. Crear un **Certificado del Servidor** llamado `ACME_A_SERVER_CERT`.  
4. Configurar el **Servidor OpenVPN** con los siguientes parámetros:
   - **Server Mode:** Peer to Peer (SSL/TLS)
   - **Protocol:** UDP
   - **Port:** 1194
   - **Tunnel Network:** 172.16.1.0/30
   - **Local Network:** 10.0.20.0/24
   - **Remote Network:** 10.0.30.0/24
   - **Cryptographic Settings:** AES-256-CBC, SHA256
   - **TLS Key:** generar automáticamente
5. Descargar los certificados necesarios (CA y Server Cert).
6. Aplicar los cambios y verificar el estado del servicio en **Status → OpenVPN**.

---

## 4. Configuración en pfSense ACME B (Cliente OpenVPN)

1. Acceder a **VPN → OpenVPN → Clients → Add**.  
2. Configurar los siguientes valores:
   - **Server host or address:** 10.0.20.27
   - **Port:** 1194
   - **Protocol:** UDP
   - **Server Mode:** Peer to Peer (SSL/TLS)
   - **Certificate Authority:** importar `ACME_CA`
   - **Client Certificate:** importar `ACME_B_CLIENT_CERT`
   - **Cryptographic Settings:** AES-256-CBC, SHA256
   - **Tunnel Network:** 172.16.1.0/30
   - **Local Network:** 10.0.30.0/24
   - **Remote Network:** 10.0.20.0/24
3. Guardar los cambios y habilitar la conexión.  
4. Confirmar el estado del túnel en **Status → OpenVPN → Status**, donde debe mostrarse como *up*.

---

## 5. Reglas de Firewall

En ambos pfSense, se deben crear reglas de firewall para permitir el tráfico a través del túnel VPN:

1. Ir a **Firewall → Rules → OpenVPN**.  
2. Crear una regla con los siguientes parámetros:
   - **Action:** Pass  
   - **Protocol:** Any  
   - **Source:** Any  
   - **Destination:** Any  
   - **Description:** Permitir tráfico VPN site-to-site  
3. Aplicar los cambios.  
4. Verificar conectividad entre las redes **10.0.20.0/24** y **10.0.30.0/24** a través del túnel.

---

## 6. Pruebas de Conectividad

Para validar la correcta implementación del túnel VPN SSL/TLS:

**Desde la sede ACME A:**
```bash
ping 10.0.30.10
````

*(Reemplazar `10.0.30.10` por la IP de un host dentro de la red LAN de la sede B.)*

**Desde la sede ACME B:**

```bash
ping 10.0.20.10
```

*(Reemplazar `10.0.20.10` por la IP de un host dentro de la red LAN de la sede A.)*

**Resultados esperados:**

* Las pruebas `ping` deben tener respuesta exitosa.
* En **Status → OpenVPN**, ambos túneles deben mostrarse como *up*.
* El tráfico debe observarse en la interfaz `ovpn` de cada pfSense.

**Prueba adicional con traceroute:**

```bash
traceroute 10.0.30.10
```

El resultado debe mostrar el paso del tráfico a través del túnel VPN (`172.16.1.0/30`).

---

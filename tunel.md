# Configuración del Túnel VPN Site-to-Site entre Sede ACME A y Sede ACME B

## 1. Descripción General

El objetivo de esta configuración es establecer una conexión segura entre dos sedes virtuales de la empresa ACME mediante un túnel **VPN site-to-site** implementado con **pfSense**.  
El túnel se configura bajo el modo **Peer-to-Peer (Shared Key)** utilizando **OpenVPN**, lo que permite el cifrado del tráfico y la comunicación directa entre las redes internas de ambas sedes.

Esta interconexión asegura que los dispositivos pertenecientes a las subredes locales de cada sede puedan comunicarse de manera segura y controlada, respetando las políticas de segmentación y aislamiento definidas por VLAN.

---

## 2. Parámetros de Red

| Elemento        | Sede ACME A              | Sede ACME B              |
|-----------------|--------------------------|--------------------------|
| pfSense         | 10.0.20.27               | 10.0.30.27               |
| Red local       | 10.0.20.0/24             | 10.0.30.0/24             |
| Tipo de VPN     | OpenVPN (Peer to Peer - Shared Key) | OpenVPN (Peer to Peer - Shared Key) |
| Protocolo       | UDP                      | UDP                      |
| Puerto          | 1194                     | 1194                     |
| Red del túnel   | 172.16.1.0/30            | 172.16.1.0/30            |

---

## 3. Configuración en pfSense ACME A (Servidor OpenVPN)

1. Ingresar a **VPN → OpenVPN → Wizards**.
2. Crear una nueva **CA (Certificate Authority)** denominada `ACME_CA`.
3. Crear un **servidor OpenVPN** con los siguientes parámetros:
   - **Server Mode:** Peer to Peer (Shared Key)
   - **Protocol:** UDP
   - **Port:** 1194
   - **Tunnel Network:** 172.16.1.0/30
   - **Local Network:** 10.0.20.0/24
   - **Remote Network:** 10.0.30.0/24
4. Generar y guardar la **Shared Key**, que será utilizada por el cliente (pfSense de la sede B).
5. Aplicar los cambios y verificar que el servicio aparezca activo en **Status → OpenVPN**.

---

## 4. Configuración en pfSense ACME B (Cliente OpenVPN)

1. Acceder a **VPN → OpenVPN → Clients → Add**.
2. Configurar los siguientes valores:
   - **Server host or address:** 10.0.20.27
   - **Port:** 1194
   - **Protocol:** UDP
   - **Shared Key:** (pegar la clave generada en ACME A)
   - **Tunnel Network:** 172.16.1.2
   - **Remote Network:** 10.0.20.0/24
   - **Local Network:** 10.0.30.0/24
3. Guardar los cambios y habilitar la conexión.
4. Confirmar el estado de la conexión en **Status → OpenVPN → Status**, donde debe mostrarse como *up*.

---

## 5. Reglas de Firewall

En ambos pfSense, se deben definir reglas de firewall que permitan el tráfico a través del túnel VPN:

1. Ir a **Firewall → Rules → OpenVPN**.
2. Crear una regla con los siguientes parámetros:
   - **Action:** Pass
   - **Protocol:** Any
   - **Source:** Any
   - **Destination:** Any
   - **Description:** Permitir tráfico VPN site-to-site
3. Aplicar los cambios y verificar que el tráfico entre las redes 10.0.20.0/24 y 10.0.30.0/24 esté permitido a través del túnel.

---

## 6. Pruebas de Conectividad

Para validar la correcta implementación del túnel VPN:

**Desde la sede ACME A:**
```bash
ping 10.0.30.27

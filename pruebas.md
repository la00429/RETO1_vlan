# Verificación práctica de aislamiento y acceso controlado entre segmentos

## 1. Objetivo

El propósito de esta sección es comprobar la correcta segmentación de la red y la aplicación de las políticas de seguridad mediante el túnel **OpenVPN SSL/TLS**.  
Se busca verificar que los departamentos o VLANs permanezcan aislados entre sí, salvo aquellos casos en los que las reglas del firewall permitan la comunicación.

---

## 2. Escenario de Prueba

En cada sede (**ACME A** y **ACME B**) se cuenta con al menos dos VLAN configuradas:

| VLAN | Departamento | Rango de Red |
|------|---------------|--------------|
| VLAN10 | Administración | 10.0.20.0/24 |
| VLAN20 | Ventas | 10.0.30.0/24 |

El túnel VPN SSL/TLS conecta ambas sedes, permitiendo que los equipos autorizados se comuniquen a través de la red cifrada **172.16.1.0/30**.

---

## 3. Pruebas de Verificación

### 3.1. Prueba de conectividad entre sedes (túnel activo)

**Desde un host en VLAN10 (ACME A):**
```bash
ping 10.0.30.10
````

**Desde un host en VLAN20 (ACME B):**

```bash
ping 10.0.20.10
```

**Resultado esperado:**
Las respuestas `ping` deben ser exitosas. Esto confirma que el túnel **OpenVPN SSL/TLS** está estable y que las rutas entre redes se aplican correctamente.

---

### 3.2. Prueba de aislamiento entre VLANs dentro de una misma sede

**Desde un host en VLAN10 (Administración):**

```bash
ping 10.0.20.50
```

*(IP perteneciente a VLAN20 - Ventas)*

**Resultado esperado:**
Sin respuesta (`Request timed out`).
Esto confirma que las VLAN están correctamente aisladas por las reglas de firewall y no se permite comunicación directa entre segmentos.

---

### 3.3. Prueba de control de acceso permitido

**Caso:** El administrador permite acceso de la VLAN de TI a todos los segmentos.

**Desde un host en VLAN30 (TI):**

```bash
ping 10.0.20.10
ping 10.0.30.10
```

**Resultado esperado:**
Respuestas exitosas, demostrando que las reglas del firewall permiten comunicación controlada desde el segmento TI hacia los demás.

---

### 3.4. Prueba de intento de acceso no autorizado

**Desde un host en VLAN20 (Ventas):**

```bash
ping 10.0.30.10
```

*(IP de otro departamento o sede no autorizado)*

**Resultado esperado:**
Sin respuesta.
El firewall bloquea la conexión, cumpliendo con la política de restricción de tráfico inter-VLAN.

---

## 4. Monitoreo de tráfico en pfSense

Durante las pruebas se puede usar la herramienta integrada **Diagnostics → Packet Capture**:

* Interfaz: `ovpn`
* Filtro: dirección IP del host remoto (por ejemplo, `10.0.30.10`)
* Duración: 10 segundos

**Resultado esperado:**

* Paquetes ICMP visibles solo cuando la política lo permite.
* No hay paquetes en tránsito entre VLANs aisladas.

---

## 5. Posibles Errores y Solución

| Problema detectado                               | Causa probable                                      | Solución                                                                                |
| ------------------------------------------------ | --------------------------------------------------- | --------------------------------------------------------------------------------------- |
| No hay respuesta al `ping` entre sedes           | Certificados SSL no coinciden o túnel inactivo      | Revisar el estado del túnel en **Status → OpenVPN** y verificar logs.                   |
| Los segmentos pueden comunicarse sin restricción | Reglas de firewall demasiado permisivas             | Limitar el tráfico entre VLANs en **Firewall → Rules → VLANX**.                         |
| No se observa tráfico en la interfaz `ovpn`      | Túnel caído o sin tráfico permitido                 | Confirmar conectividad y revisar políticas de enrutamiento.                             |
| La VLAN no accede a internet                     | Gateway incorrecto o DNS no configurado             | Verificar gateway y reglas de NAT en **Firewall → NAT → Outbound**.                     |
| El túnel se cae intermitentemente                | Diferencia de cifrados o problemas con certificados | Asegurar que ambos pfSense usen las mismas suites criptográficas (AES-256-CBC, SHA256). |



````markdown
# Verificación Práctica de Aislamiento y Acceso Controlado entre Segmentos

Este apartado tiene como propósito validar que la segmentación mediante VLANs y las reglas del firewall en pfSense cumplen correctamente con las políticas de acceso establecidas, garantizando el aislamiento entre departamentos y la comunicación segura cuando es requerida.

---

## 1. Objetivo de las pruebas
Verificar la correcta implementación de:
- Aislamiento entre VLANs de diferentes departamentos.
- Accesibilidad controlada entre segmentos permitidos.
- Conectividad interna e intersede mediante VPN site-to-site.
- Aplicación efectiva de políticas de seguridad (bloqueos y permisos específicos).

---

## 2. Equipos y segmentos de red

| Departamento     | VLAN | Red/Subred         | Ejemplo de IP asignada | pfSense asociado |
|------------------|------|--------------------|------------------------|------------------|
| Administración   | 10   | 10.0.10.0/24 (A) / 10.1.10.0/24 (B) | 10.0.10.101 / 10.1.10.101 | pfSense A / B |
| Ventas           | 20   | 10.0.20.0/24 (A) / 10.1.20.0/24 (B) | 10.0.20.101 / 10.1.20.101 | pfSense A / B |
| Tecnología (TI)  | 30   | 10.0.30.0/24 (A) / 10.1.30.0/24 (B) | 10.0.30.101 / 10.1.30.101 | pfSense A / B |

---

## 3. Escenarios de prueba

### 3.1 Prueba 1 – Conectividad básica dentro de la misma VLAN
**Objetivo:** Verificar que los equipos de un mismo departamento pueden comunicarse entre sí.

**Procedimiento:**
```bash
# Desde PC-Admin-A
ping 10.0.10.101   # Hacia PC-Admin-B a través del túnel VPN
````

**Resultado esperado:**

* El ping debe responder correctamente.
* La latencia debe ser baja y constante.
* El tráfico debe enrutarse por la VPN site-to-site (verificable con `traceroute` o `tracert`).

---

### 3.2 Prueba 2 – Intento de acceso entre VLANs no autorizadas

**Objetivo:** Confirmar que las reglas del firewall bloquean el acceso desde la VLAN Ventas hacia la VLAN TI.

**Procedimiento:**

```bash
# Desde PC-Ventas-A (10.0.20.x)
ping 10.0.30.101   # Intento hacia PC-TI-A
```

**Resultado esperado:**

* El ping no debe responder.
* Wireshark o los logs del firewall deben registrar el bloqueo (ICMP bloqueado).
* Esto demuestra que la segmentación entre VLANs funciona correctamente.

---

### 3.3 Prueba 3 – Acceso controlado desde VLAN TI

**Objetivo:** Verificar que el departamento de TI tiene acceso administrativo a todas las VLANs según las políticas configuradas.

**Procedimiento:**

```bash
# Desde PC-TI-A (10.0.30.x)
ping 10.0.10.101   # Hacia PC-Admin-A
ping 10.0.20.101   # Hacia PC-Ventas-A
```

**Resultado esperado:**

* Ambos pings deben responder exitosamente.
* Los paquetes deben cruzar el firewall con permiso explícito para la VLAN30 (TI).
* La regla de firewall correspondiente debe reflejar "Pass" en los logs.

---

### 3.4 Prueba 4 – Conectividad VPN site-to-site

**Objetivo:** Confirmar que las VLAN equivalentes en ambas sedes se comunican correctamente por la VPN cifrada.

**Procedimiento:**

```bash
# Desde PC-TI-A
ping 10.1.30.101   # Hacia PC-TI-B
```

**Resultado esperado:**

* Comunicación estable y respuesta satisfactoria.
* El túnel VPN debe estar activo (estado “up”) en ambas pfSense.
* En pfSense A y B, el registro de OpenVPN debe mostrar tráfico cifrado.

---

### 3.5 Prueba 5 – Acceso a Internet desde VLAN autorizadas

**Objetivo:** Validar que cada VLAN puede acceder a Internet según las políticas definidas.

**Procedimiento:**

```bash
# Desde PC-Ventas-A
ping 8.8.8.8       # Comprobación de salida a Internet
```

**Resultado esperado:**

* Solo las VLAN con permisos de salida (por ejemplo, Ventas y Administración) deben tener acceso.
* VLANs restringidas (si alguna lo está) no deben obtener respuesta.

---

## 4. Validación adicional

1. **Capturas en pfSense**:
   Usar *Diagnostics → Packet Capture* para observar si el tráfico bloqueado es interceptado correctamente.

2. **Logs de firewall:**
   Revisar *Status → System Logs → Firewall* para confirmar la acción “Block” o “Pass” según las pruebas.

3. **Trazado de ruta:**
   Ejecutar `traceroute` para confirmar la trayectoria del tráfico por la VPN y no por rutas externas.

---

## 5. Resultados esperados finales

| Prueba                                | Resultado esperado   | Cumple |
| ------------------------------------- | -------------------- | ------ |
| 1. Comunicación intra-VLAN            | Comunicación exitosa | ✅      |
| 2. Bloqueo entre VLANs no autorizadas | Bloqueo confirmado   | ✅      |
| 3. Acceso TI a todas las VLANs        | Permitido            | ✅      |
| 4. Comunicación entre sedes vía VPN   | Correcta             | ✅      |
| 5. Acceso controlado a Internet       | Conforme a políticas | ✅      |

---


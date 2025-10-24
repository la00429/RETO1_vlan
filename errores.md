# Posibles Errores durante la Verificación del Aislamiento y Acceso Controlado

Durante la validación del entorno de red, pueden presentarse diferentes fallos relacionados con la configuración de VLANs, reglas de firewall o el túnel VPN. A continuación, se detallan los posibles errores más comunes, sus causas probables y las acciones recomendadas para su corrección.

---

## 1. Errores de Conectividad entre VLANs

### 1.1 Las VLANs no entregan direcciones IP
**Causa probable:**
- El servicio DHCP no está habilitado o no está asociado a la VLAN correspondiente.
- La interfaz VLAN no tiene asignada una dirección IP.
- La interfaz VLAN no está activada.

**Solución:**
1. Ir a `Services → DHCP Server` y verificar que el servicio esté habilitado para cada VLAN.  
2. Revisar en `Interfaces → Assignments` que las VLANs tengan IP configuradas.  
3. Confirmar que las interfaces VLAN estén habilitadas (casilla *Enable Interface* marcada).

---

### 1.2 No hay comunicación entre equipos del mismo departamento
**Causa probable:**
- Los equipos no están conectados al mismo switch virtual (red interna de VirtualBox).  
- El firewall de pfSense bloquea el tráfico dentro de la misma VLAN.  

**Solución:**
1. Verificar en VirtualBox que ambos equipos estén en la misma red interna correspondiente.  
2. En pfSense, ir a `Firewall → Rules → VLANx` y agregar una regla **“Allow any to any”** temporal para pruebas.

---

## 2. Errores en el Túnel VPN Site-to-Site

### 2.1 El túnel VPN no se establece
**Causa probable:**
- Clave compartida (Shared Key) diferente entre las sedes.  
- IP de destino incorrecta en el cliente.  
- Puerto o protocolo erróneo (UDP/TCP 1194).  
- Falta de coincidencia en la red local o remota declarada.

**Solución:**
1. Verificar la configuración de OpenVPN en ambas pfSense (`VPN → OpenVPN`).  
2. Confirmar que la “Shared Key” sea idéntica en ambas partes.  
3. Revisar los campos **Local Network** y **Remote Network** (deben coincidir con los rangos 10.0.x.0/24 y 10.1.x.0/24).  
4. Comprobar conectividad entre las interfaces WAN de ambas pfSense (`ping 192.168.100.2` ↔ `192.168.200.2`).

---

### 2.2 El túnel se establece, pero no hay comunicación entre VLANs equivalentes
**Causa probable:**
- Falta de ruta estática hacia la red remota.  
- Las reglas del firewall de OpenVPN bloquean el tráfico.  

**Solución:**
1. Agregar una regla en `Firewall → Rules → OpenVPN` para permitir tráfico entre redes 10.0.0.0/16 y 10.1.0.0/16.  
2. En `System → Routing`, revisar que existan rutas activas hacia la red remota.  
3. Verificar con `traceroute` que el tráfico pase por el túnel.

---

## 3. Errores en las Reglas de Firewall

### 3.1 Tráfico bloqueado inesperadamente
**Causa probable:**
- Regla de “Block” mal posicionada (pfSense evalúa reglas de arriba hacia abajo).  
- Las reglas se aplican a la interfaz equivocada.

**Solución:**
1. Revisar el orden de las reglas en `Firewall → Rules`.  
2. Asegurar que las reglas de bloqueo estén debajo de las de permiso específicas.  
3. Confirmar en los *logs* del firewall (Status → System Logs → Firewall) qué regla generó el bloqueo.

---

### 3.2 No hay acceso a Internet desde una VLAN con permisos
**Causa probable:**
- Falta de regla de salida (NAT) en `Firewall → NAT → Outbound`.  
- DNS mal configurado o bloqueado por el firewall.  

**Solución:**
1. Habilitar el modo “Automatic Outbound NAT” en pfSense o agregar manualmente reglas NAT por VLAN.  
2. Verificar que las VLAN tengan configurado un DNS funcional (`8.8.8.8` o el de pfSense).  
3. Probar resolución de nombres con:
   ```bash
   nslookup google.com
   ```

## 4. Errores de Configuración en VirtualBox

### 4.1 Los equipos no obtienen red o no hacen ping

**Causa probable:**

* Adaptadores mal asignados o redes internas incorrectas.
* pfSense no tiene asociadas todas las redes necesarias.

**Solución:**

1. Comprobar en **Configuración → Red** de cada máquina que los adaptadores estén conectados a las redes internas correctas.
2. Verificar que las interfaces de pfSense estén en modo **“Internal Network”** con los nombres exactos (`vlan10_admin`, `vlan20_sales`, etc.).

---

## 5. Errores de Segmentación y Aislamiento

### 5.1 Acceso no autorizado entre VLANs

**Causa probable:**

* Regla de firewall demasiado permisiva (por ejemplo, “allow any to any”).
* Tráfico inter-VLAN permitido por defecto en pfSense.

**Solución:**

1. Restringir las reglas por VLAN según política:

   * **VLAN10 (Admin):** acceso completo.
   * **VLAN20 (Ventas):** solo salida a Internet.
   * **VLAN30 (TI):** acceso a todas las VLANs.
2. Comprobar el tráfico con *Diagnostics → Packet Capture* para verificar que los paquetes bloqueados no cruzan interfaces.

---

## 6. Recomendaciones de Diagnóstico

| Comando / Herramienta          | Uso principal                  | Comentario                                                   |
| ------------------------------ | ------------------------------ | ------------------------------------------------------------ |
| `ping <IP>`                    | Verificar conectividad básica  | Ideal para pruebas de aislamiento                            |
| `traceroute <IP>`              | Confirmar la ruta del tráfico  | Muestra si el túnel VPN se usa correctamente                 |
| `tcpdump -i <interfaz>`        | Capturar paquetes en pfSense   | Permite observar si el tráfico sale por la VLAN o por la VPN |
| `Diagnostics → Packet Capture` | Herramienta gráfica de pfSense | Identifica intentos de conexión bloqueados                   |
| `Status → OpenVPN`             | Estado del túnel VPN           | Verifica si está activo y con tráfico                        |

---

## 7. Conclusión

La mayoría de los errores durante la verificación de aislamiento y control de acceso se deben a configuraciones incorrectas en:

* El direccionamiento IP o las rutas.
* La posición de las reglas de firewall.
* La correspondencia de VLANs con las redes internas en VirtualBox.

Una revisión sistemática de las interfaces, servicios DHCP, reglas y rutas garantiza el restablecimiento correcto de la conectividad y el cumplimiento de las políticas de segmentación establecidas.

```
```

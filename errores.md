# Errores específicos en la configuración y operación de la VPN SSL/TLS entre sedes ACME

## 1. Errores comunes durante la configuración

| Error | Causa probable | Solución recomendada |
|--------|----------------|----------------------|
| **“TLS Error: TLS handshake failed”** | Desfase en certificados entre servidor y cliente. El certificado del cliente no pertenece a la CA configurada en el servidor. | Regenerar certificados desde la misma CA en pfSense ACME A y reimportarlos correctamente en ACME B. Verificar que el campo **Peer Certificate Authority** sea el mismo en ambos. |
| **“Inactivity timeout (--ping-restart)”** | El cliente pierde comunicación con el servidor por una regla de firewall o problema de enrutamiento. | Revisar reglas en **Firewall → Rules → OpenVPN** y permitir tráfico bidireccional. Confirmar que el túnel (`ovpn`) esté activo en ambas sedes. |
| **“TLS key negotiation failed to occur within 60 seconds”** | El servidor no escucha en el puerto configurado o el cliente no puede alcanzarlo. | Verificar que el puerto **UDP 1194** esté abierto en pfSense A. Comprobar direcciones IP y conectividad previa entre WANs. |
| **“No valid SSL certificate found”** | No se ha asignado el certificado correcto en el servidor o cliente. | En **VPN → OpenVPN → Server/Client**, revisar los campos **Server Certificate** y **Client Certificate**. Reasignar los generados por la CA “ACME_CA”. |
| **“Route addition failed using service”** | El túnel no logra agregar rutas automáticas a las subredes remotas. | Revisar que las redes locales/remotas estén correctamente declaradas en cada lado (por ejemplo, `10.0.20.0/24` ↔ `10.0.30.0/24`). Agregar rutas estáticas si es necesario. |
| **El cliente no logra resolver nombres (DNS)** | El servidor no propaga DNS o los clientes usan servidores externos no accesibles. | En **OpenVPN Server → Advanced Configuration**, agregar: `push "dhcp-option DNS 10.0.20.27"`. Reiniciar el servicio. |

---

## 2. Errores de firewall y segmentación

| Error | Causa probable | Solución recomendada |
|--------|----------------|----------------------|
| **Las VLANs pueden comunicarse entre sí sin restricción** | Reglas de firewall demasiado permisivas o colocadas en orden incorrecto. | Reorganizar reglas en **Firewall → Rules**: primero las de bloqueo, luego las de permiso. Asegurar que el tráfico inter-VLAN esté explícitamente bloqueado. |
| **Tráfico legítimo bloqueado entre sedes** | Regla de bloqueo genérica aplicada sobre la interfaz VPN. | Mover la regla de bloqueo debajo de la regla que permite el túnel o crear una excepción específica para las subredes 10.0.20.0/24 ↔ 10.0.30.0/24. |
| **El túnel funciona, pero no hay conectividad entre VLANs de la misma sede** | Falta de rutas internas o NAT incorrecto. | Verificar que las VLANs estén correctamente asignadas y que cada interfaz tenga gateway configurado. Revisar la tabla de rutas en **Diagnostics → Routes**. |

---

## 3. Errores operativos y de rendimiento

| Error | Causa probable | Solución recomendada |
|--------|----------------|----------------------|
| **Latencia alta o pérdida de paquetes en el túnel** | Saturación del adaptador virtual, fragmentación de MTU o sobrecarga del cifrado. | Reducir MTU en la interfaz VPN a 1400 (`ifconfig-push mtu 1400`). Revisar carga del sistema y CPU. |
| **El túnel se desconecta después de reiniciar pfSense** | El servicio OpenVPN no inicia automáticamente o depende del orden de carga de interfaces. | Ir a **Status → Services** y habilitar **OpenVPN → Start on boot**. Verificar dependencias con interfaces VLAN. |
| **Las pruebas de ping funcionan, pero no hay acceso HTTP/HTTPS entre sedes** | Bloqueo de puertos TCP en reglas de firewall. | Agregar reglas específicas para puertos 80 y 443 o habilitar tráfico completo entre redes autorizadas en la interfaz OpenVPN. |

---

## 4. Errores en certificados y autenticación

| Error | Causa probable | Solución recomendada |
|--------|----------------|----------------------|
| **“Certificate expired or not yet valid”** | Certificados vencidos o con fecha incorrecta en el sistema. | Regenerar certificados en **System → Cert Manager → Certificates**. Sincronizar fecha/hora en ambos pfSense mediante **System → General Setup → NTP Server**. |
| **“CRL verification failed”** | Lista de revocación (CRL) no disponible o corrompida. | En **System → Cert Manager → Certificate Revocation**, eliminar la CRL y volver a generarla desde la CA. |
| **Error al importar certificado en cliente** | Archivo en formato incorrecto o con encabezados incompletos (`-----BEGIN CERTIFICATE-----`). | Exportar nuevamente el certificado desde pfSense A y asegurarse de copiarlo completo, incluyendo encabezado y pie. |

---

## 5. Errores en monitoreo y diagnóstico

| Error | Causa probable | Solución recomendada |
|--------|----------------|----------------------|
| **No se visualiza tráfico en la interfaz ovpn** | Filtros incorrectos en captura o túnel sin actividad. | Realizar ping sostenido y capturar tráfico en **Diagnostics → Packet Capture → Interface: ovpn**. |
| **El log de OpenVPN no muestra actividad** | Nivel de registro bajo. | Aumentar nivel de log en **VPN → OpenVPN → Advanced Configuration:** `verb 4`. Luego revisar **Status → System Logs → OpenVPN**. |
| **IPs incorrectas en la captura de tráfico** | Tráfico enmascarado por NAT o doble enrutamiento. | Verificar configuración de NAT en **Firewall → NAT → Outbound** y asegurarse de que no se aplique NAT sobre la interfaz VPN. |

---

# Verificación práctica de aislamiento y acceso controlado entre segmentos

## 1. Objetivo
Comprobar el correcto funcionamiento del aislamiento entre las VLAN configuradas y la segmentación del tráfico mediante políticas de firewall en pfSense, garantizando que cada departamento solo tenga acceso a los recursos permitidos y que la VPN site-to-site funcione correctamente entre las sedes ACME A y ACME B.

---

## 2. Preparación del entorno

**Infraestructura de red utilizada:**
- **Sede ACME A:** Red 10.0.20.0/24  
- **Sede ACME B:** Red 10.0.30.0/24  
- **IP del firewall pfSense en ACME A:** 10.0.20.27  
- **IP del firewall pfSense en ACME B:** 10.0.30.27  
- **VLANs configuradas:**
  - VLAN 20: Ventas  
  - VLAN 30: Soporte técnico  

Cada VLAN cuenta con su puerta de enlace configurada en pfSense y un rango de direcciones IP asignado. Se han definido reglas de firewall diferenciadas para restringir el tráfico entre VLANs.

---

## 3. Procedimiento de verificación

### 3.1. Prueba de conectividad interna por VLAN
1. Acceder a una estación dentro de la VLAN 10 (Administración).
2. Ejecutar el comando `ping` hacia otra estación dentro de la misma VLAN.
   ```bash
   ping 10.0.10.15

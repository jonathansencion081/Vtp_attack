# Ataque VTP — Inyección y borrado de VLANs

**Autor:** Jonathan Sención  
**Matrícula:** 20250851  
**Materia:** Seguridad de Redes — ITLA (Instituto Tecnológico de las Américas)

---

## Objetivo del laboratorio
Demostrar cómo un atacante puede manipular las VLANs de toda una red abusando del protocolo **VTP (VLAN Trunking Protocol)** de Cisco —agregando y borrando VLANs— y documentar la contramedida correspondiente.

## Objetivo del ataque
Introducir en la red un switch **"rogue"** configurado como **servidor VTP** del dominio `ITLA`. Al unirse al dominio, el atacante puede **agregar** la VLAN 666 y **borrar** la VLAN 20; ambos cambios se propagan automáticamente a todos los switches del dominio vía VTP.

## Topología
| Equipo | Rol | Interfaz |
|--------|-----|----------|
| SW1 | VTP **Server**, dominio `ITLA` | Et0/0 (troncal a SW2), Et0/3 (troncal al ROGUE) |
| SW2 | VTP **Client** | Et0/0 (troncal a SW1) |
| ROGUE | Switch atacante, VTP **Server**, dominio `ITLA` | Et0/0 (troncal a SW1) |

- **VLANs del dominio:** 10 (USUARIOS), 20 (SERVIDORES), 99 (GESTION)
- **Dominio VTP:** `ITLA` (sin contraseña — condición vulnerable)

## Funcionamiento del ataque
En VTP, **cualquier switch que entre al dominio como servidor puede modificar la base de datos de VLANs de toda la red**, y el switch con el **número de revisión más alto** impone su configuración.

1. Se configura el switch ROGUE como servidor VTP del dominio `ITLA`.
2. Se conecta por un enlace troncal a SW1; el ROGUE sincroniza la base de datos del dominio.
3. Desde el ROGUE se **agrega** la VLAN 666 y se **borra** la VLAN 20.
4. Ambos cambios se propagan vía VTP a SW1 y SW2 **sin tocar esos switches**.

Archivo del ataque: [`rogue_vtp.cfg`](rogue_vtp.cfg)

## Requisitos
- Un switch adicional Cisco IOL L2 como atacante (ROGUE), o Kali con herramientas VTP.
- Acceso físico/lógico a un puerto **troncal** del dominio VTP.
- Dominio VTP **sin contraseña**.

## Parámetros utilizados
- Dominio VTP: `ITLA`
- Modo del ROGUE: `server`
- VLAN agregada: `666` (nombre `HACKED`)
- VLAN borrada: `20` (`SERVIDORES`)

## Verificación (en SW1 y SW2)
```
show vlan brief    -> aparece la VLAN 666 (HACKED), desaparece la VLAN 20
show vtp status    -> la Configuration Revision aumenta y cambia "modified by"
```

## Nota sobre Yersinia / Scapy
Se incluye `vtp_poc_scapy.py` como prueba de concepto que construye los anuncios VTP (Summary + Subset). Sin embargo, las imágenes IOS/IOL que validan estrictamente el **digest MD5** del anuncio rechazan las tramas generadas por Yersinia 0.8.2 o por Scapy sin la implementación exacta del MD5. Por ello el ataque se **demuestra con el método del switch rogue**, que utiliza VTP real de Cisco (MD5 válido) y constituye el ataque VTP clásico documentado.

## Contramedidas
1. **Contraseña VTP** en los switches legítimos (autenticación del dominio):
   ```
   SW1(config)# vtp password ClaveSegura123
   SW2(config)# vtp password ClaveSegura123
   ```
   Sin la contraseña correcta, el ROGUE no se autentica y sus anuncios se **ignoran**.
2. **Modo transparente u off:** `vtp mode transparent` (no participa en VTP) o `vtp mode off`.
3. **Limitar troncales:** en puertos de usuario, `switchport mode access` + `switchport nonegotiate` para que no se formen troncales no autorizados por donde viaja VTP.

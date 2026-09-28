# Documentación Unificada de Configuración de Red

## 1. Configuración del Router

```text
en
configure terminal
interface gigabitethernet 0/0
no shutdown
exit

! Subinterfaz VLAN 10 (Administración)
interface gigabitethernet 0/0.10
encapsulation dot1Q 10
ip address 192.168.10.1 255.255.255.0

! Subinterfaz VLAN 20 (Profesorado)
interface gigabitethernet 0/0.20
encapsulation dot1Q 20
ip address 192.168.20.1 255.255.255.0

! Subinterfaz VLAN 30 (Alumnos)
interface gigabitethernet 0/0.30
encapsulation dot1Q 30
ip address 192.168.30.1 255.255.255.0

! Subinterfaz VLAN 999 (Nativa de gestión del tronco)
interface gigabitethernet 0/0.999
encapsulation dot1Q 999 native
ip address 192.168.999.1 255.255.255.0
end
write
```

---

## 2. Configuración del Switch Central (Switch 1)

```text
Switch Central:
en
configure terminal

! Creación de todas las VLANs en la red
vlan 10
 name Administracion
vlan 20
 name Profesorado
vlan 30
 name Alumnos
vlan 666
 name Blackhole_Sinkhole
vlan 999
 name VLAN_Nativa

! Puerto hacia el Router (Troncal con VLAN nativa 999)
interface gigabitethernet 0/1
switchport mode trunk
switchport trunk native vlan 999

! Puertos hacia Switch0 y Switch2 (También troncales)
interface range fastethernet 0/1 - 2
switchport mode trunk
switchport trunk native vlan 999

! Puertos locales para los PCs de Profesorado (PC3, PC4, PC5 en puertos Fa0/3 a Fa0/5)
interface range fastethernet 0/3 - 5
switchport mode access
switchport access vlan 20

! Puertos libres restantes mandados al Blackhole y apagados
interface range fastethernet 0/6 - 24
switchport mode access
switchport access vlan 666
shutdown
end
write
```

---

## 3. Configuración del Switch 0

```text
en
configure terminal

! Creación de VLANs necesarias
vlan 10
 name Administracion
vlan 666
 name Blackhole_Sinkhole
vlan 999
 name VLAN_Nativa

! Puerto troncal que viene del Switch1 (Fa0/1)
interface fastethernet 0/1
switchport mode trunk
switchport trunk native vlan 999

! Puertos para los PCs de Administración (PC0, PC1, PC2 en Fa0/2 a Fa0/4)
interface range fastethernet 0/2 - 4
switchport mode access
switchport access vlan 10

! Puertos libres al Blackhole y apagados
interface range fastethernet 0/5 - 24
switchport mode access
switchport access vlan 666
shutdown
end
write
```

---

## 4. Configuración del Switch 2

```text
en
configure terminal

! Creación de VLANs necesarias
vlan 30
 name Alumnos
vlan 666
 name Blackhole_Sinkhole
vlan 999
 name VLAN_Nativa

! Puerto troncal que viene del Switch1 (Fa0/1)
interface fastethernet 0/1
switchport mode trunk
switchport trunk native vlan 999

! Puertos para los PCs de Alumnos (PC6, PC7, PC8 en Fa0/2 a Fa0/4)
interface range fastethernet 0/2 - 4
switchport mode access
switchport access vlan 30

! Puertos libres al Blackhole y apagados
interface range fastethernet 0/5 - 24
switchport mode access
switchport access vlan 666
shutdown
end
write
```

---

## 5. Configuración IP de los Dispositivos

```text
6. Configuración IP de los PCs (En la pestaña "Desktop > IP Configuration" de cada PC)
Switch0 (Administración - VLAN 10):

PC0: IP 192.168.10.10, Máscara 255.255.255.0, Gateway 192.168.10.1

PC1: IP 192.168.10.11, Máscara 255.255.255.0, Gateway 192.168.10.1

PC2: IP 192.168.10.12, Máscara 255.255.255.0, Gateway 192.168.10.1

Switch1 (Profesorado - VLAN 20):

PC3: IP 192.168.20.10, Máscara 255.255.255.0, Gateway 192.168.20.1

PC4: IP 192.168.20.11, Máscara 255.255.255.0, Gateway 192.168.20.1

PC5: IP 192.168.20.12, Máscara 255.255.255.0, Gateway 192.168.20.1

Switch2 (Alumnos - VLAN 30):

PC6: IP 192.168.30.10, Máscara 255.255.255.0, Gateway 192.168.30.1

PC7: IP 192.168.30.11, Máscara 255.255.255.0, Gateway 192.168.30.1

PC8: IP 192.168.30.12, Máscara 255.255.255.0, Gateway 192.168.30.1

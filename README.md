# Documentacion Practica Final

# INFORME DE IMPLEMENTACIÓN TÉCNICA: INFRAESTRUCTURA SEGURA Y EVASIÓN DE FIREWALL MEDIANTE VPN

**Institución:** Instituto Tecnológico de las Américas (ITLA)
**Estudiante:** Cesar Valdez Elibo
**Matrícula:** 2024-2372
**Materia:** Seguridad de Redes
**Profesor:** Jonathan Esteban Rondon Corniel
**Fecha:** 08/04/2026

**Link YT**: [Click Aqui](https://youtu.be/V-VnICnS9Wo)

---

## 1. Resumen Ejecutivo y Objetivo del Proyecto

El presente documento detalla la implementación desde cero de una arquitectura de red corporativa que aplica el principio de **Defensa en Profundidad (Defense in Depth)**.

El proyecto simula un entorno donde el acceso a un servidor interno está restringido por un Firewall perimetral (Fortinet). Para proveer acceso seguro y autorizado a este recurso, se desplegó un servidor VPN L2TP. Simultáneamente, la infraestructura base fue sometida a un proceso de *Hardening* (endurecimiento) en las Capas 2 y 3 del modelo OSI, aplicando control de acceso granular para garantizar que solo usuarios legítimos puedan establecer el túnel VPN, previniendo así amenazas de suplantación de identidad y movimientos laterales.

---

## 2. Topología y Esquema de Direccionamiento IP

La red se diseñó y segmentó desde cero utilizando la plataforma PNETLab, estableciendo zonas de confianza claras. El esquema de direccionamiento se basó en los requerimientos del entorno de laboratorio:
<img width="675" height="393" alt="image" src="https://github.com/user-attachments/assets/a0c14a03-e066-4bc0-916c-f7157fe430cb" />
<img width="942" height="747" alt="image" src="https://github.com/user-attachments/assets/ab161be8-b418-4f5b-856b-1edca4516c7d" />

## 3. Fases de Implementación y Configuración (Desde Cero)

La red se construyó siguiendo un modelo de implementación por capas, asegurando cada nivel antes de proceder al siguiente.

### Fase I: Ruteo Base y Segmentación (Router 2)

El primer paso consistió en levantar el núcleo de la red interna. El Router 2 fue configurado como el Gateway principal para las redes de usuarios, manejando el enrutamiento Inter-VLAN (Router-on-a-Stick).

- Se crearon las subinterfaces `e0/1.30` y `e0/1.40` con encapsulamiento 802.1Q.
- Se configuraron los servicios DHCP para asignar dinámicamente direcciones a las máquinas Windows, excluyendo las IPs de los gateways.
- Se estableció una ruta por defecto `0.0.0.0 0.0.0.0` apuntando a la interfaz interna del FortiGate (`172.16.1.2`).

<img width="1032" height="262" alt="image" src="https://github.com/user-attachments/assets/897433a0-6ba0-41e3-8204-2169dd80b52d" />

### Fase II: Hardening de Capa 2 (Switches de Acceso)

Para evitar que la red fuera comprometida desde la capa de enlace de datos, se aplicaron configuraciones de seguridad estrictas en los Switches 1 y 2:

- **Prevención de Suplantación:** Se habilitó **DHCP Snooping** globalmente y se declararon los puertos hacia los routers como confiables (`trust`). Esto se complementó con **Dynamic ARP Inspection (DAI)** para validar que cada paquete ARP correspondiera a una asignación DHCP legítima, mitigando ataques *Man-in-the-Middle*.
- **Seguridad de Puertos:** En las interfaces de acceso (conectadas a las PCs y al Servidor), se habilitó **Port-Security** con un máximo de 1 dirección MAC por puerto, usando el método `sticky` para aprendizaje dinámico y la acción `violation shutdown` para aislar intrusos automáticamente.
- **Protección de Topología:** Se configuró **BPDU Guard** y **Portfast** en los puertos de usuario para prevenir ataques de denegación de servicio (DoS) dirigidos al protocolo Spanning-Tree.
<img width="569" height="298" alt="image" src="https://github.com/user-attachments/assets/514966cb-2d81-452a-9304-150eaeea7e9e" />

<img width="876" height="467" alt="image" src="https://github.com/user-attachments/assets/6ea21d91-e8d0-4f3a-b74b-36816e6469df" />

### Fase III: Políticas Perimetrales y Firewall (FortiGate)

El FortiGate se configuró como el punto central de inspección y ruteo de borde.

- **Enrutamiento Estático:** Se le enseñó al firewall cómo alcanzar las VLANs internas (a través del Router 2) y cómo llegar a la red virtual de la VPN (a través del Router 3).
- **Bloqueo Selectivo (El Reto):** Se creó una política de firewall que **deniega** explícitamente todo el tráfico HTTP (puerto 80) originado desde la red de usuarios hacia el Servidor Web. Esta regla es la que hace necesaria la evasión.
- **Regla de Evasión:** Se configuró una política que permite el tráfico total hacia el servidor, pero **únicamente** si proviene de la subred del túnel VPN (`192.23.72.0/24`).

<img width="1856" height="386" alt="image" src="https://github.com/user-attachments/assets/5a6e8720-32a7-4935-931f-de6dcecd5ce5" />

### Fase IV: Implementación del Servidor VPN y Control de Acceso Granular

La última fase del desarrollo fue levantar los servicios criptográficos y restringir su uso.

- **Configuración del LNS (Router 3):** Se habilitó el servicio VPDN (Virtual Private Dialup Network) para aceptar conexiones L2TP. Se exigió el uso de protocolos de autenticación fuertes (`CHAP`, `MS-CHAP-v2`) en la interfaz Virtual-Template y se creó la cuenta de usuario local con la matrícula `2024-2372`.
- **Control de Identidad (Router 2):** Para cumplir con el requerimiento de seguridad de impedir el acceso a la VLAN 40, se diseñó una **ACL Extendida**. Esta lista se aplicó de entrada en la subinterfaz `.40`, bloqueando proactivamente los paquetes con destino al puerto UDP 1701 (L2TP).

<img width="744" height="220" alt="image" src="https://github.com/user-attachments/assets/1a3a77d4-d2f6-4c96-aab3-1e2d9c0befe2" />
<img width="884" height="163" alt="image" src="https://github.com/user-attachments/assets/6d26c660-0621-43d1-94d4-4245dff0414e" />

## 4. Resultados y Validación de la Infraestructura

Una vez construida la topología, el comportamiento de la red valida el éxito de la implementación:

1. **Aislamiento de Seguridad Efectivo:** Al intentar establecer la conexión VPN desde un host de la VLAN 40, la conexión es rechazada a nivel de red (Capa 3). La ACL del Router 2 descarta el tráfico antes de que alcance el servidor VPN, demostrando un control de acceso granular exitoso.
<img width="545" height="372" alt="image" src="https://github.com/user-attachments/assets/1ed58c31-4078-4f9b-b86e-10d43e955654" />
2. **Cumplimiento de Políticas de Firewall:** Cualquier intento de navegación directa mediante HTTP desde la VLAN 30 hacia el Servidor Web resulta en un "Time Out". El Fortinet inspecciona y bloquea el tráfico de acuerdo con la política establecida.
3. **Evasión Exitosa y Ruteo Tunelizado:** Al autenticarse satisfactoriamente en la VPN desde la VLAN 30, el cliente recibe una dirección IP del pool `192.23.72.x`. Al reintentar la conexión web, el servidor responde entregando la página con el identificador del estudiante. El tráfico, al viajar encapsulado dentro del túnel UDP/L2TP, pasa de forma transparente (evade) la regla de bloqueo HTTP del firewall perimetral.
<img width="541" height="140" alt="image" src="https://github.com/user-attachments/assets/4ea5d209-ea99-4f8b-9f4c-6bb597d027e5" />
<img width="1052" height="292" alt="image" src="https://github.com/user-attachments/assets/12a385ad-0a6e-414c-96de-2ef621e6c214" />
<img width="578" height="160" alt="image" src="https://github.com/user-attachments/assets/7b45fb3c-47a2-4ade-bef5-e2f7463542ef" />

# Seguridad de Redes - Implementación y Validación de Túnel IPsec Site-to-Site Heterogéneo (Cisco & FortiGate)
# ENLACE HACIA VIDEO: https://youtu.be/1OI2it4toN4
## 📌 Datos Generales

* **Autor:** Manuel Alejandro Cruz Messón
* **Matrícula:** 2025-0689
* **Fecha:** 2 de Octubre 2026
* **Plataforma:** Entorno virtualizado (PNetLab / GNS3 / Cisco IOS & FortiOS 7.0.3)

---

## 🗺️ Topología de Red
![Topología de Red](https://raw.githubusercontent.com/labcruzmesson-cyber/FORTIGATE-INFRAESTRUCTURA-2/refs/heads/main/IMAGES/Screenshot%202026-10-02%20144425.png)

## 1. Diseño de Direccionamiento IP y VLSM

Para optimizar el direccionamiento y garantizar el aislamiento estructural entre los hosts clientes y los servidores de producción, se implementó un esquema de direccionamiento con máscaras de subred de longitud variable (**VLSM**) derivado de la matrícula del autor (**2025-0689**), segmentando a partir de los identificadores **25**, **06** y **89**.

### 1.1 Requisitos de Capacidad por Segmento

1. **Subred 1 (VLAN 10 - Red de Clientes/Usuarios):** Puestos de trabajo dimensionados con prefijo `/25`, gestionados dinámicamente mediante DHCP implementado en el Router Cisco.
2. **Subred 2 (LAN Servidor Web - DMZ Remota):** Segmento acotado con prefijo `/28` destinado exclusivamente a servidores de producción que publican servicios HTTP/HTTPS.
3. **Segmento WAN Interconexión ISP:** Bloque `192.168.145.0/24` para enlaces de tránsito público y peering perimetral entre el Router Cisco y el firewall FortiGate.

### 1.2 Tabla de Direccionamiento

| Segmento | ID VLAN | Red de Subred | Máscara de Red | Rango Útil | Puerta de Enlace | Dispositivos Clave Asignados |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **VLAN 10 (Usuarios)** | 10 | `10.25.6.0/25` | `255.255.255.128` | `10.25.6.1 - 10.25.6.126` | `10.25.6.1` | Tiny Core Linux (`10.25.6.10` vía DHCP) |
| **LAN Servidor Web** | N/A | `172.25.6.0/28` | `255.255.255.240` | `172.25.6.1 - 172.25.6.14` | `172.25.6.1` | Web Server Ubuntu (`172.25.6.2`), Nginx |
| **WAN ISP - Cisco** | N/A | `192.168.145.0/24` | `255.255.255.0` | `192.168.145.1 - 192.168.145.254` | `192.168.145.1` | Cisco Router `Gi0/1` (`192.168.145.150`) |
| **WAN ISP - FortiGate** | N/A | `192.168.145.0/24` | `255.255.255.0` | `192.168.145.1 - 192.168.145.254` | `192.168.145.1` | FortiGate WAN `port1` (`192.168.145.148`) |

---

## 2. Configuración en Router Cisco (CLI)

En el router Cisco se implementó el esquema de subinterfaces IEEE 802.1Q (Router-on-a-Stick), pool DHCP para los clientes de la VLAN 10, exclusión de NAT y configuración criptográfica IPsec.

### 2.1 Subinterfaces y Servidor DHCP

```text
Router# show ip interface brief
Interface                  IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0         unassigned      YES unset  up                    up      
GigabitEthernet0/0.10      10.25.6.1       YES manual up                    up      
GigabitEthernet0/1         192.168.145.150 YES manual up                    up      
GigabitEthernet0/2         unassigned      YES unset  administratively down down    
GigabitEthernet0/3         unassigned      YES unset  administratively down down    
```

* **DHCP Pool Status (`VLAN10_USERS`):**
  * Rango de IPs: `10.25.6.1 - 10.25.6.126`
  * Direcciones arrendadas: Asignada a host cliente TinyCore (`10.25.6.10`).

### 2.2 Exclusión de NAT para Tráfico VPN (Tráfico Interesante)

Para evitar la alteración de cabeceras privadas en el túnel IPsec, se deniega explícitamente el tráfico inter-sitio en la ACL de NAT:

```cisco
ip access-list extended ACL_NAT
 deny ip 10.25.6.0 0.0.0.127 172.25.6.0 0.0.0.15
 permit ip 10.25.6.0 0.0.0.127 any
```

### 2.3 Criptografía ISAKMP / IPsec (Cisco IOS)

* **Fase 1 (IKEv1):** `crypto isakmp policy 10` con cifrado `DES`, autenticación `SHA` (SHA1), Diffie-Hellman `Group 14` y clave precompartida `ClaveSecreta2025` asociada al peer remoto `192.168.145.148`.
* **Fase 2 (IPsec):** Transform-set `esp-des esp-sha-hmac` en modo túnel vinculado a la crypto map `CMAP_VPN` sobre la interfaz `GigabitEthernet0/1`.

---

## 3. Configuración en FortiGate (GUI 7.0.3)

Toda la configuración en FortiGate fue construida mediante el entorno gráfico web de FortiOS.

### 3.1 Enrutamiento y Conectividad WAN

* **Ruta por Defecto:**
  * Destino: `0.0.0.0/0`
  * Gateway: `192.168.145.2` (o provisto por el upstream ISP)
  * Interfaz: `port1` (WAN)
* **Ruta Estática hacia Red Remota VPN:**
  * Destino: `10.25.6.0/25`
  * Gateway: `192.168.145.150`
  * Interfaz: `VPN-CISCO` (Interfaz virtual del túnel)

### 3.2 Parámetros del Túnel IPsec Site-to-Site

* **Ruta GUI:** `VPN` > `IPsec Tunnels` > `Create New` > `IPsec Tunnel (Custom)`
* **Fase 1:**
  * Remote Gateway: `192.168.145.150` (IP WAN Cisco)
  * Outgoing Interface: `port1` (`192.168.145.148`)
  * Pre-shared Key: `ClaveSecreta2025`
  * IKE Mode: IKEv1 / Main Mode
  * Propuesta: Encryption `DES`, Authentication `SHA1`, Diffie-Hellman `Group 14`, Key Lifetime `86400` s.
* **Fase 2 (Selectores de Tráfico):**
  * Local Address: `172.25.6.0/255.255.255.240` (`172.25.6.0/28`)
  * Remote Address: `10.25.6.0/255.255.255.128` (`10.25.6.0/25`)
  * Propuesta: Encryption `DES`, Authentication `SHA1`, PFS: Desactivado.

### 3.3 Matriz de Políticas de Firewall en FortiOS

| Nombre de Política | Interfaz Origen | Interfaz Destino | Origen | Destino | Servicio | Acción | NAT |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **VPN-A-SERVER** | `VPN-CISCO` | `port2` (LAN Servidor) | `LAN_Usuarios_VLAN10` (`10.25.6.0/25`) | `LAN_Servidor_Web` (`172.25.6.0/28`) | HTTP, HTTPS, ALL ICMP (o ALL) | ACCEPT | Disabled |
| **SERVER-A-VPN** | `port2` (LAN Servidor) | `VPN-CISCO` | `LAN_Servidor_Web` (`172.25.6.0/28`) | `LAN_Usuarios_VLAN10` (`10.25.6.0/25`) | ALL | ACCEPT | Disabled |
| **Implicit Deny** | `any` | `any` | `all` | `all` | ALL | DENY | N/A |

---

## 4. Resultados de Verificación y Pruebas Funcionales

### 4.1 Matriz de Comprobaciones Técnicas

| # | Prueba | Acción Ejecutada | Resultado en Terminal | Evidencia | Diagnóstico |
| :-: | :--- | :--- | :--- | :--- | :--- |
| **1** | Asignación DHCP en VLAN 10 | `ip -br addr show` en TinyCore | `eth0 UP 10.25.6.10/25` | Concesión de red y gateway `10.25.6.1` correctos | **Éxito** |
| **2** | Asociación IPsec Fase 1 y 2 | `show crypto isakmp sa`<br>`show crypto ipsec sa` | `QM_IDLE / ACTIVE`<br>`pkts encaps / decaps > 0` | Logs FortiOS: `tunnel-up`, `phase2-up`, `install_sa` exitosos | **Éxito** |
| **3** | Acceso Web Seguro (HTTP) | `curl -i http://172.25.6.2` | `HTTP/1.1 200 OK` | Conexión establecida; entrega de código `<h1>Mi server funciona</h1>` | **Éxito** |
| **4** | Trazado de Ruta por Túnel | `traceroute -I -n 172.25.6.2` | 3 saltos limpios:<br>1: `10.25.6.1`<br>2: `192.168.145.148`<br>3: `172.25.6.2` | Tráfico canalizado sin pasar por direccionamiento público del ISP | **Éxito** |
| **5** | Exclusividad (Caída de VPN) | Desactivación del túnel (*Bring Down* en GUI) | `traceroute`: se frena en `10.25.6.1 * * *`<br>`curl`: `(28) Connection timed out` | Paquetes descartados; el flujo no se fuga por Internet | **Éxito** |
| **6** | Restauración del Enlace | Activación del túnel (*Bring Up* en GUI) | `HTTP/1.1 200 OK`<br>`traceroute`: 3 saltos completados | Restablecimiento instantáneo de la sesión cifrada | **Éxito** |

### 4.2 Evidencia de Consola

#### Traza de Red hacia el Servidor Web

```bash
root@box:/home/gns3# traceroute -I -n 172.25.6.2
traceroute to 172.25.6.2 (172.25.6.2), 30 hops max, 38 byte packets
 1  10.25.6.1 (10.25.6.1)  2.333 ms  1.911 ms  1.456 ms
 2  192.168.145.148 (192.168.145.148)  2.906 ms  2.906 ms  2.895 ms
 3  172.25.6.2 (172.25.6.2)  3.038 ms  3.347 ms  2.936 ms
```

#### Solicitud HTTP con Encabezados

```bash
root@box:/home/gns3# curl -i http://172.25.6.2
HTTP/1.1 200 OK
Server: nginx/1.14.0 (Ubuntu)
Date: Fri, 02 Oct 2026 18:28:59 GMT
Content-Type: text/html
Content-Length: 28
Last-Modified: Fri, 02 Oct 2026 16:53:04 GMT
Connection: keep-alive
ETag: "6abfe170-1c"
Accept-Ranges: bytes

<h1>Mi server funciona</h1>
```

---

## 5. Justificación Técnica y Análisis de Comportamiento

1. **Selección de Suites Criptográficas en Entornos Virtuales:**
   * Las imágenes de evaluación virtual de FortiGate imponen restricciones criptográficas en el asistente GUI, bloqueando combinaciones de algoritmos como DES con SHA256. FortiOS requiere que cifrados simétricos tradicionales como DES se asocien con hashing simétrico clásico (SHA1/MD5). La solución fue homologar de forma idéntica en ambos extremos: `DES` + `SHA1` + `DH Group 14`.
2. **Comportamiento de Traceroute en Linux (Sondas UDP vs. ICMP):**
   * El binario estándar de `traceroute` en sistemas Unix utiliza datagramas UDP dirigidos a puertos efímeros altos ($>33434$). La política inicial de FortiGate inspeccionaba únicamente TCP (HTTP/HTTPS) e ICMP. Al emplear sondas ICMP (`traceroute -I -n`) o configurar el servicio en `ALL` en la política, los paquetes superaron la inspección de estado del kernel de FortiOS.
3. **Preservación de Cabeceras IP y Exclusión de NAT:**
   * La regla `deny` previa en la access-list de NAT en Cisco y el parámetro `nat disable` en la política de FortiGate preservan las cabeceras IP privadas originales (`10.25.6.10` hacia `172.25.6.2`), viajando encapsuladas en ESP (IPsec modo túnel) sin fuga ni traducción a través del ISP.

---

## 6. Conclusiones

* **Interoperabilidad Heterogénea:** Se validó satisfactoriamente el peering VPN IPsec Site-to-Site entre dos sistemas operativos y fabricantes distintos (Cisco IOS y Fortinet FortiOS), coordinando la negociación IKEv1 y Phase 2 SAs sin conflictos.
* **Aislamiento y Exclusividad del Enlace:** La comunicación depende en su totalidad del túnel cifrado; al forzar la caída de la VPN, el tráfico privado no se redirige por la ruta default ni se expone al ISP upstream.
* **Cumplimiento Integral de Directivas:** El despliegue cumplió con el dimensionamiento VLSM por matrícula, el aprovisionamiento gráfico (GUI) en FortiOS, la exclusión estricta de NAT y la validación de servicios web de extremo a extremo.

---

## 7. Declaración sobre el Uso de Inteligencia Artificial (IA)

En el desarrollo de este informe técnico/laboratorio se utilizaron herramientas de Inteligencia Artificial Generativa exclusivamente como apoyo para la redacción, estructuración del documento, depuración de sintaxis en scripts y consulta conceptual sobre configuraciones de red.

Todo el diseño topológico, el cálculo de direccionamiento IP (VLSM), la configuración en los equipos virtuales (FortiOS/Cisco) y la validación empírica de las pruebas de conectividad fueron implementados, ejecutados y supervisados directamente por el autor. El autor asume la responsabilidad total de la veracidad, precisión técnica, integridad y conclusiones presentadas en este documento.

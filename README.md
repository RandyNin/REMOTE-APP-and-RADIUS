# RemoteApp y RD Web Client con IIS y Autenticación Centralizada AAA (NPS/RADIUS)

**Windows Server 2022 + Cisco IOS | GNS3**

Laboratorio que publica aplicaciones remotas con **RDS RemoteApp** (entregadas tanto por el portal clásico **RD Web Access** como por el cliente HTML5 **RD Web Client**), sirve una página personalizada de **IIS** y la publica como RemoteApp, y controla el **acceso administrativo** a un router Cisco mediante **AAA** validado por un servidor **NPS/RADIUS** de Windows, con dos niveles de privilegio según el grupo de dominio del usuario.

**Autor:** Randy Nin &nbsp;|&nbsp; **Matrícula:** 2025-0660

- **Video:** [https://youtu.be/1kudbGeEuM4](https://youtu.be/1kudbGeEuM4)
---

## Índice

- [Resumen](#resumen)
- [Entorno del laboratorio](#entorno-del-laboratorio)
- [Baseline (preexistente)](#baseline-preexistente)
- [Despliegue RDS / RemoteApp](#despliegue-rds--remoteapp)
- [Certificado del servicio](#certificado-del-servicio)
- [RD Web Client (HTML5)](#rd-web-client-html5)
- [Página personalizada de IIS y su publicación como RemoteApp](#página-personalizada-de-iis-y-su-publicación-como-remoteapp)
- [NPS como servidor RADIUS](#nps-como-servidor-radius)
- [Router R-CORE: AAA, RADIUS y hardening](#router-r-core-aaa-radius-y-hardening)
- [Verificación](#verificación)
- [Referencia CLI del router](#referencia-cli-del-router)
- [Conclusión](#conclusión)

---

## Resumen

La idea central de una **RemoteApp** es que la aplicación se ejecuta íntegramente en el servidor y al cliente solo le llega su ventana. El usuario ve lo que parece un programa local, pero cada clic, cada cálculo y cada conexión de red ocurren en el servidor. Este laboratorio entrega esa RemoteApp por dos vías: el portal clásico **RD Web Access** (que descarga un archivo `.rdp` abierto por el cliente de Escritorio Remoto) y el **RD Web Client** (un cliente HTML5 que ejecuta la app dentro del navegador).

El acceso administrativo al router se centraliza con **AAA** (Authentication, Authorization, Accounting) delegado en un servidor **RADIUS**. El rol **NPS** de Windows valida las credenciales contra Active Directory y, según el grupo del usuario, devuelve al router el nivel de privilegio que debe otorgar mediante el atributo de proveedor de Cisco, el par `shell:priv-lvl`. El router no conoce los grupos del dominio: solo obedece ese atributo.

**Objetivos:** desplegar RDS/RemoteApp y el RD Web Client, servir una página personalizada de IIS y publicarla como RemoteApp, levantar NPS/RADIUS con dos niveles de acceso (privilegio 15 y privilegio 1) atados a grupos del dominio, y configurar el router con usuario local, SSH y AAA sobre RADIUS con registro.

---

## Entorno del laboratorio

Un router central (R-CORE) conecta tres dominios de red: internet a través del ISP (nube NAT de GNS3, con DHCP en gig0/0), el segmento de servidores y el de clientes. Un único Windows Server 2022 concentra todos los roles del lado servidor, y una VM Windows 11 y una Kali Linux actúan como clientes.

![Topología](./IMG/topology.png)

El direccionamiento deriva de la matrícula **2025-0660** (ambos segmentos usan el patrón `.6.60`, clave RADIUS `Radius0660`).

| Dispositivo | Interfaz | IP / Máscara | Función |
|:---|:---|:---|:---|
| R-CORE | gig0/0 | DHCP (desde el ISP) | Internet, NAT outside |
| R-CORE | gig0/4 | 172.6.60.1/24 | Gateway de servidores, NAT inside, origen RADIUS |
| R-CORE | gig0/5 | 192.6.60.1/24 | Gateway de clientes, NAT inside, relay DHCP |
| DC-RANDYNIN | Ethernet | 172.6.60.10/24 | AD DS, DNS, DHCP, NPS, RDS, IIS |
| Windows 11 | Ethernet | 192.6.60.x/24 (DHCP) | Cliente de pruebas |
| Kali Linux | eth0 | 192.6.60.5/24 (DHCP) | Cliente de pruebas (SSH) |

> **Nota:** Solo la LAN de servidores tiene PAT (overload) hacia el ISP. La LAN de clientes **no** tiene NAT a propósito, por lo que no tiene salida directa a internet. Esto se usa luego para demostrar que una RemoteApp realmente corre en el servidor (ver [Verificación](#verificación)).

---

## Baseline (preexistente)

Active Directory (dominio `randy.nin`), DNS y DHCP ya estaban configurados antes de esta práctica. Lo relevante para RADIUS son los dos grupos de TI: **NET-ADMINS** (privilegio 15) y **NET-SUPPORT** (privilegio 1), ambos bajo la OU TI.

![Estructura de OUs y contenido de TI](./IMG/ad-ou-ti.png)

**NET-ADMINS** contiene al administrador de red Mark Grayson (privilegio 15 en el router).

![Miembros de NET-ADMINS](./IMG/group-net-admins.png)

**NET-SUPPORT** contiene al técnico de soporte Tobio Kageyama (privilegio 1, solo lectura).

![Miembros de NET-SUPPORT](./IMG/group-net-support.png)

Un tercer usuario, Walter White, está en la OU HHRR y no pertenece a ningún grupo de red. Se usa luego para demostrar que un usuario válido del dominio sin autorización de red es rechazado.

![OU HHRR y Walter White](./IMG/ad-ou-hhrr.png)

| Usuario | Login | OU | Grupo | Acceso al router |
|:---|:---|:---|:---|:---|
| Mark Grayson | mgrayson | TI | NET-ADMINS | Privilegio 15 (total) |
| Tobio Kageyama | tkageyama | TI | NET-SUPPORT | Privilegio 1 (solo lectura) |
| Walter White | wwhite | HHRR | HHRR | Denegado |

El DHCP sirve al segmento de clientes. Como el servidor DHCP (172.6.60.10) está en un segmento distinto al de los clientes, el router relaya las peticiones con `ip helper-address 172.6.60.10`.

![Ámbito DHCP](./IMG/dhcp-scope.png)

El servidor usa direccionamiento estático.

![ipconfig del servidor](./IMG/server-ipconfig.png)

---

## Despliegue RDS / RemoteApp

RDS se desplegó con la instalación dedicada de **Remote Desktop Services** en modo **Quick Start**, escenario **Session-based** (no como roles sueltos). Esto instala y enlaza tres roles en el mismo servidor: **RD Web Access** (el portal `/RDWeb`), **RD Connection Broker** (reparte y administra las sesiones) y **RD Session Host** (donde realmente corren las apps).

El flujo importa: el usuario nunca se conecta directo al Session Host. Entra por el portal, el broker lo autentica y lo dirige, la app se ejecuta en el Session Host, y al cliente solo le llega la ventana de la aplicación.

![Despliegue RDS](./IMG/rds-deployment.png)

Las RemoteApp se publicaron en la colección `QuickSessionCollection`, asignada al grupo de usuarios del dominio.

![RemoteApps publicadas en la colección](./IMG/rds-collection-remoteapps.png)

`RNIN_Page` es la RemoteApp usada para publicar la página de IIS (más abajo).

---

## Certificado del servicio

El certificado que RDS genera solo **no** servía en los navegadores modernos. El problema no era de confianza, sino del **Key Usage** del certificado: le faltaba el atributo **DigitalSignature**. Los cifrados TLS modernos basados en `ECDHE_RSA` (los que negocian Chrome y Edge por defecto) requieren que el certificado del servidor lleve el bit `digitalSignature`, y el certificado autogenerado solo traía `keyEncipherment`. Por eso Chrome y Edge 115+ rechazan el handshake con **`ERR_SSL_KEY_USAGE_INCOMPATIBLE`**.

Un detalle revelador: la RemoteApp abierta por el cliente de Escritorio Remoto (`mstsc`, el archivo `.rdp`) sí funcionaba con ese certificado, porque el cliente RDP no valida ese atributo. Solo el navegador lo hace. Así que el certificado defectuoso rompía las vías por navegador (la página de IIS y el RD Web Client), pero no la vía `.rdp`.

La solución fue generar por PowerShell un certificado nuevo con los Key Usages correctos (**DigitalSignature** y **KeyEncipherment**), asignarlo a los tres roles del despliegue RDS y al binding 443 de IIS, e importarlo como certificado de confianza en el cliente para que el candado saliera limpio.

![Certificado en PowerShell](./IMG/certificate-powershell.png)

![Certificado asignado a los roles RDS](./IMG/rds-deployment-certificate.png)

La consola del despliegue muestra el nivel como `Untrusted` simplemente porque no encadena con una CA pública (normal en un entorno interno); ese lado se resolvió importando el certificado en el almacén de confianza del cliente. La parte crítica que rompía los navegadores era el Key Usage, resuelta al regenerar el certificado.

---

## RD Web Client (HTML5)

La segunda vía de entrega es el **RD Web Client**, un cliente HTML5 que ejecuta las RemoteApp dentro del navegador sin descargar ningún `.rdp`. A diferencia del resto de la configuración, no tiene interfaz gráfica: se instala y publica solo por PowerShell con el módulo `RDWebClientManagement`, un complemento del PowerShell Gallery.

Surgió un obstáculo: las versiones de fábrica de `PowerShellGet` y `PackageManagement` no podían leer el módulo por su formato, así que hubo que actualizarlos primero. Una vez instalado, se importó el certificado del broker y se publicó en producción.

![Paquete del RD Web Client](./IMG/rdwebclient-package.png)

Ambas vías sirven la misma RemoteApp: el portal clásico en `/RDWeb` (descarga un `.rdp`) y el cliente web en `/RDWeb/webclient/` (renderiza la app en HTML5).

---

## Página personalizada de IIS y su publicación como RemoteApp

IIS ya estaba presente porque RD Web Access corre sobre él. Se colocó una página de portada con el nombre y la matrícula en `wwwroot`, servida en la raíz `https://randy.nin` con el mismo certificado.

![Página personalizada de IIS](./IMG/iis-custom-page.png)

El binding 443 se configuró con el certificado `RDS-RandyNin`.

![Binding HTTPS de IIS](./IMG/iis-https-binding.png)

Para publicar la página como RemoteApp se publicó Microsoft Edge (`msedge.exe`) con el parámetro `--app=https://randy.nin`, que abre Edge en modo aplicación (una sola ventana, sin barra ni pestañas) mostrando la página. Esa RemoteApp es `RNIN_Page`.

![Parámetro de RNIN_Page](./IMG/remoteapp-iis-param.png)

El mismo mecanismo (Edge corriendo en el servidor) habilita la demo de internet más abajo: ese Edge tiene salida a internet aunque el cliente no.

---

## NPS como servidor RADIUS

El rol **NPS** convierte al servidor en un RADIUS. Se registró en Active Directory para poder leer usuarios y grupos del dominio. Escucha en `172.6.60.10`.

El router se registra como cliente RADIUS, identificado por su IP y un secreto compartido que debe coincidir con el del router.

![Cliente RADIUS R-CORE](./IMG/nps-radius-client.png)

**Política de privilegio 15 (NET-ADMINS):**

![Política NPS para NET-ADMINS](./IMG/nps-policy-priv15.png)

**Política de privilegio 1 (NET-SUPPORT):**

![Política NPS para NET-SUPPORT](./IMG/nps-policy-priv1.png)

Tres detalles son determinantes en ambas políticas:

1. El control clave es el atributo **Cisco-AV-Pair** `shell:priv-lvl` (15 o 1). Le indica al router qué privilegio dar según el grupo de AD. El router no sabe de grupos; solo obedece el atributo.
2. Ambas habilitan autenticación **PAP**, porque el IOS envía las credenciales al RADIUS usando PAP. Si el NPS no acepta PAP, rechaza todo por incompatibilidad de método.
3. El `Service-Type` se fija en `Login` y se retira el atributo `Framed-Protocol`, dejando solo lo pertinente a una sesión de administración.

El orden importa: el NPS evalúa de arriba hacia abajo y aplica la primera política que coincide, por eso la de privilegio 15 va encima. Un usuario de NET-ADMINS coincide con la primera (privilegio 15); uno de NET-SUPPORT solo con la segunda (privilegio 1); uno que no esté en ningún grupo no coincide con ninguna política de concesión y es rechazado.

---

## Router R-CORE: AAA, RADIUS y hardening

Toda la configuración de seguridad se agregó de forma **aditiva** sobre la base que el router ya tenía (interfaces y NAT), sin tocarla.

### Usuario local y enable secret

Un usuario local `randy` (privilegio 15) y un `enable secret` protegen el modo privilegiado/configuración. El usuario local cumple el requisito y sirve de respaldo anti-bloqueo si el RADIUS falla (las listas de métodos caen a `local`). Ambos secretos se guardan como hash.

![Usuario local y enable secret](./IMG/router-user-enable.png)

### SSH

Se define el nombre de dominio (`ip domain name randy.nin`), se generan las llaves RSA de 2048 bits, se fuerza SSH versión 2 y se ajustan timeout e intentos. El dominio **debe** ir antes de generar las llaves, porque IOS construye la etiqueta de la llave a partir del hostname + dominio (`R-CORE.randy.nin`); sin dominio, la generación falla.

![show ip ssh](./IMG/router-ssh.png)

### AAA y RADIUS

Se activa AAA (`aaa new-model`), el servidor RADIUS apunta a `172.6.60.10` (puertos 1812/1813, mismo secreto), y se agrupa en `GRP-RADIUS` con `ip radius source-interface GigabitEthernet0/4` para que los paquetes RADIUS salgan con origen `172.6.60.1`, la IP registrada como cliente en NPS. Se habilitan los atributos de proveedor con `radius-server vsa send authentication` y `accounting`.

![show run section aaa](./IMG/router-aaa.png)

![show run section radius](./IMG/router-radius.png)

Dos piezas hacen efectiva la diferenciación de privilegios:

- **Envío de VSA:** sin `radius-server vsa send authentication`, el atributo `shell:priv-lvl` llega pero el router lo ignora, y el usuario cae siempre en privilegio 1.
- **Autorización de exec** (`aaa authorization exec default group GRP-RADIUS local if-authenticated`): la autenticación solo verifica identidad; es la autorización de exec la que lee el privilegio devuelto y abre la sesión en ese nivel. Sin esa línea, el usuario autentica pero cae siempre en privilegio 1.

### Listas de métodos, líneas y registro

La autenticación de login por defecto usa el grupo RADIUS con respaldo local, la autorización de exec usa el grupo RADIUS con `if-authenticated`, y el accounting de exec es start-stop. La consola tiene su propia lista de autenticación **solo local**, para que nunca dependa del RADIUS y sirva de puerta de emergencia. Las VTY solo permiten `transport input ssh` (sin Telnet).

![show run section line](./IMG/router-lines.png)

El registro usa timestamps con fecha y hora en debug y log, un buffer de logging, y el registro automático `login on-success` / `login on-failure`.

---

## Verificación

### 1. Página por RD Web Access (portal clásico)

![Login en RD Web Access](./IMG/rdweb-login.png)

![RemoteApps en RD Web Access](./IMG/rdweb-apps.png)

Al lanzar una RemoteApp se descarga un `.rdp`; se muestra el publicador antes de conectar.

![Descarga del .rdp](./IMG/rdweb-rdp-download.png)

![Publicador de la RemoteApp](./IMG/remoteapp-publisher.png)

![Credenciales de la RemoteApp](./IMG/remoteapp-credentials.png)

![RemoteApp conectada](./IMG/remoteapp-connected.png)

### 2. Página por RD Web Client (HTML5)

La segunda vía ejecuta las apps dentro del navegador, sin descarga.

![Login en RD Web Client](./IMG/webclient-login.png)

![RemoteApps en RD Web Client](./IMG/webclient-apps.png)

Al lanzar `RNIN_Page` se abre la página de IIS dentro de la pestaña del navegador.

![Página IIS por RD Web Client](./IMG/webclient-iis-page.png)

Es la misma RemoteApp entregada de dos formas distintas.

### 3. Demostración de la RemoteApp por el acceso a internet

Como la LAN de clientes no tiene PAT, la Windows 11 no tiene internet directo: el navegador no carga Google y el `ping 8.8.8.8` falla.

![El cliente no tiene internet por navegador](./IMG/client-no-internet-browser.png)

![El ping del cliente falla](./IMG/client-no-internet-ping.png)

Sin embargo, la RemoteApp de Edge **sí** llega a Google, porque ese Edge corre en el servidor (que sí tiene internet), no en el cliente. Al cliente solo le llega la ventana. Esto demuestra visualmente que la app corre en el servidor.

![La RemoteApp de Edge llega a internet](./IMG/remoteapp-edge-internet.png)

### 4. SSH con privilegio 15 (mgrayson, NET-ADMINS)

`mgrayson` es autorizado con privilegio 15, así que entra directo a modo privilegiado (`R-CORE#`) sin escribir `enable` y puede acceder a configuración.

![SSH privilegio 15](./IMG/ssh-priv15-mgrayson.png)

### 5. SSH con privilegio 1 (tkageyama, NET-SUPPORT)

`tkageyama` autentica pero el NPS le asigna privilegio 1. La sesión abre en modo usuario (`R-CORE>`) y `configure terminal` es negado.

![SSH privilegio 1](./IMG/ssh-priv1-tkageyama.png)

### 6. Usuario denegado (wwhite, HHRR)

`wwhite` es un usuario válido del dominio pero no pertenece a ningún grupo de TI. El NPS no encuentra política de concesión y rechaza el acceso; el router cierra la conexión. No basta con ser usuario del dominio: hay que estar en un grupo autorizado.

![SSH denegado](./IMG/ssh-denied-wwhite.png)

![Logs de login exitoso del router](./IMG/router-log-login-success.png)

![Logs de login fallido del router](./IMG/router-log-login-failed.png)

> **Nota sobre el comando SSH:** desde Kali (y OpenSSH moderno en general) hay que forzar algoritmos heredados con `-o KexAlgorithms=+diffie-hellman-group14-sha1 -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa`, porque OpenSSH moderno deshabilitó los algoritmos basados en SHA-1, que son los únicos que ofrece el IOSv 15.6. En producción la solución correcta es actualizar el IOS del equipo para que ofrezca SHA-2, no debilitar el cliente; aquí es una limitación de la imagen de laboratorio.

### 7. Diagnóstico de AAA en el router

`test aaa` valida la cadena RADIUS sin abrir una sesión SSH, confirmando la autenticación y el privilegio devuelto.

![test aaa usuarios autorizados](./IMG/test-aaa-authorized.png)

![test aaa usuario rechazado](./IMG/test-aaa-rejected.png)

Se activa la depuración de autenticación, autorización y RADIUS para trazar el flujo completo.

![Depuración activada](./IMG/debug-enable.png)

![Traza completa de AAA](./IMG/debug-aaa-flow.png)

![show aaa sessions](./IMG/show-aaa-sessions.png)

![show aaa servers](./IMG/show-aaa-servers.png)

### 8. Logs del NPS (lado servidor)

El Event Viewer registra cada decisión del NPS.

![Evento NPS: acceso concedido](./IMG/nps-event-granted.png)

![Evento NPS: acceso denegado](./IMG/nps-event-denied.png)

La correspondencia es completa: el NPS concede a tkageyama (evento 6272) y niega a wwhite (evento 6273), exactamente igual que el router y las pruebas SSH.

---

## Referencia CLI del router

Los secretos aparecen como marcadores; en el dispositivo se guardan cifrados.

**Base preexistente (interfaces, NAT, relay DHCP):**

```
hostname R-CORE
no ip domain-lookup

interface GigabitEthernet0/0
 ip address dhcp
 ip nat outside
 no shutdown

interface GigabitEthernet0/4
 ip address 172.6.60.1 255.255.255.0
 ip nat inside
 no shutdown

interface GigabitEthernet0/5
 ip address 192.6.60.1 255.255.255.0
 ip helper-address 172.6.60.10
 ip nat inside
 no shutdown

access-list 10 permit 172.6.60.0 0.0.0.255
ip nat inside source list 10 interface GigabitEthernet0/0 overload
```

**Usuario local, enable secret y SSH:**

```
username randy privilege 15 secret <PASSWORD>
enable secret <ENABLE_SECRET>

ip domain name randy.nin
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
```

**AAA y RADIUS:**

```
aaa new-model

radius server NPS
 address ipv4 172.6.60.10 auth-port 1812 acct-port 1813
 key <RADIUS_KEY>

aaa group server radius GRP-RADIUS
 server name NPS
 ip radius source-interface GigabitEthernet0/4

radius-server vsa send authentication
radius-server vsa send accounting

aaa authentication login CONSOLE local
aaa authentication login default group GRP-RADIUS local
aaa authorization exec default group GRP-RADIUS local if-authenticated
aaa accounting exec default start-stop group GRP-RADIUS
aaa session-id common
```

**Líneas y registro:**

```
line console 0
 login authentication CONSOLE
line vty 0 4
 transport input ssh
 login authentication default
 authorization exec default
 exec-timeout 10 0

service timestamps debug datetime msec
service timestamps log datetime msec
logging buffered 16384 debugging
login on-failure log
login on-success log
```

**Comandos de verificación:**

```
test aaa group GRP-RADIUS <usuario> <password> new-code
debug aaa authentication
debug aaa authorization
debug radius
show aaa servers
show aaa sessions
show logging
show ip ssh
```

**Comando SSH desde el cliente (algoritmos heredados para IOSv 15.6):**

```
ssh -o KexAlgorithms=+diffie-hellman-group14-sha1 -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa <usuario>@192.6.60.1
```

---

## Conclusión

El laboratorio publica RemoteApps por dos vías (RD Web Access con `.rdp` y el RD Web Client en HTML5), resuelve el certificado TLS regenerándolo con los Key Usages correctos para que los navegadores modernos lo acepten, sirve una página personalizada de IIS por HTTPS y la expone como RemoteApp mediante Edge en modo aplicación, y centraliza el acceso al router con NPS/RADIUS. El router delega la autenticación y la autorización en RADIUS y aplica el privilegio devuelto en el atributo `shell:priv-lvl`, siendo el envío de VSA y la autorización de exec las dos piezas que hacen efectiva la diferenciación.

Las pruebas confirman cada objetivo de extremo a extremo: la página es accesible por las dos vías de RemoteApp, la RemoteApp de Edge entrega internet que el cliente por sí solo no tiene (demostrando que la app corre en el servidor), y el acceso SSH se comporta según la identidad: mgrayson con privilegio 15, tkageyama con privilegio 1 de solo lectura, y wwhite rechazado. La evidencia del router (`test aaa`, trazas de debug, `show aaa servers`/`sessions`) y los eventos del NPS (6272 concedido, 6273 denegado) coinciden por completo.

*Randy Nin | Matrícula 2025-0660*

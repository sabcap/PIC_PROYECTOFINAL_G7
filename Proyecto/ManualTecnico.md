# Manual Técnico — Proyecto Final Fase 1
## Infraestructura de Red Empresarial: AD DS, DNS, DHCP y File Server

> **Carátula**
>
> **Universidad de San Carlos de Guatemala — Escuela de Ciencias y Sistemas**  
> **Asignatura:** Prácticas Iniciales "C"
> **Proyecto Final — Fase 1 (24/09/2026):** Numerales 1.1, 1.2, 1.3 y 1.4  
> **Grupo:** 7  
> **Estudiantes:**
> - Josué Javier Carrera Soyós
> - Alvaro Javier Paredes Sulá
>
> **Dominio del laboratorio:** `servidor1.com`  
> **Servidor:** Windows Server 2022 (WS22) — IP `192.168.0.20`  
> **Clientes:** Windows 10 y Ubuntu Desktop 24.04 LTS

---

## Índice

- [Introducción](#introducción)
- [Resumen técnico del laboratorio](#resumen-técnico-del-laboratorio)
- [1.1 Servicio de Virtualización](#11-servicio-de-virtualización)
- [1.2 Servidor de Identidad y Autenticación (Active Directory)](#12-servidor-de-identidad-y-autenticación-active-directory)
  - [1.2.1 Servicio DNS](#121-servicio-dns-instalación-y-verificación)
  - [1.2.2 Servicio Active Directory Domain Services (AD DS)](#122-servicio-active-directory-domain-services-ad-ds)
  - [1.2.3 Cuentas de usuario del dominio](#123-cuentas-de-usuario-del-dominio)
  - [1.2.4 Unión de clientes al dominio](#124-unión-de-clientes-al-dominio-windows-y-linux)
  - [1.2.5 Validación de autenticación](#125-validación-de-autenticación)
  - [1.2.6 GPO — Fondo de pantalla corporativo](#126-gpo--fondo-de-pantalla-corporativo-obligatorio)
  - [1.2.7 GPO organizacional — Navegador (Edge / Chrome)](#127-gpo-organizacional--navegador-microsoft-edge--google-chrome)
- [1.3 Servidor de Asignación Dinámica de IP (DHCP)](#13-servidor-de-asignación-dinámica-de-ip-dhcp)
- [1.4 Servidor de Archivos y Recursos Compartidos (File Server)](#14-servidor-de-archivos-y-recursos-compartidos-file-server)
- [Conclusiones de Fase 1](#conclusiones-de-fase-1)
- [Anexo A — Tabla resumen de direccionamiento y cuentas](#anexo-a--tabla-resumen-de-direccionamiento-y-cuentas)
- [Anexo B — Comandos de referencia rápida](#anexo-b--comandos-de-referencia-rápida)
- [Anexo C — Referencias y documentación oficial](#anexo-c--referencias-y-documentación-oficial)

---

## Introducción

En toda organización moderna, los servicios de red constituyen la base sobre la que operan los usuarios, los equipos y las aplicaciones. La gestión centralizada de identidades, la resolución de nombres, la asignación ordenada de direcciones IP y el control de acceso a la información son requisitos mínimos para garantizar seguridad, trazabilidad y continuidad operativa. Este manual nace con el propósito de documentar cómo se construye dicha base, paso a paso y con evidencia verificable, utilizando herramientas empresariales reales en un entorno de laboratorio virtualizado.

El proyecto se desarrolla sobre dos computadoras físicas (hosts) con Oracle VirtualBox, donde se virtualizan un servidor Windows Server 2022 y clientes Windows 10 y Ubuntu Desktop 24.04 LTS, todos integrados al dominio `servidor1.com`. A lo largo del documento se explica no solo *cómo* se configuró cada servicio, sino *para qué sirve*, *qué requisitos exige* y *cómo se comprueba que funciona*, manteniendo un lenguaje técnico y profesional pero de fácil comprensión para estudiantes y evaluadores.

### Objetivo general

Diseñar, configurar e integrar una infraestructura de red empresarial funcional en un entorno virtualizado, que provea identificación centralizada, resolución de nombres, direccionamiento dinámico y recursos compartidos controlados por permisos.

### Objetivos específicos de la Fase 1

1. Virtualizar el servidor y los clientes con conectividad en modo puente dentro del mismo segmento de red.
2. Implementar DNS y Active Directory Domain Services (AD DS) para autenticación centralizada del dominio `servidor1.com`.
3. Automatizar el direccionamiento IPv4 mediante DHCP, con exclusiones, concesiones y reservas por MAC.
4. Centralizar archivos con SMB/CIFS aplicando permisos compartidos y NTFS diferenciados (recurso público y privado).
5. Estandarizar la experiencia de usuario mediante directivas de grupo (GPO) de fondo de pantalla y navegador.

Este manual documenta la **Fase 1** del proyecto, correspondiente a los numerales **1.1 a 1.4** de las especificaciones:

1. **1.1** Virtualización
2. **1.2** Identidad y autenticación (AD DS + DNS + GPO)
3. **1.3** Asignación dinámica de IP (DHCP)
4. **1.4** Archivos y recursos compartidos (File Server SMB/CIFS)
### Enfoque del documento

Cada sección sigue la misma estructura exigida en los entregables:

1. **¿Para qué sirve?** — propósito técnico del servicio.
2. **Requisitos previos.**
3. **¿Cómo se configura?** — procedimiento paso a paso tal como se ejecutó en el laboratorio.
4. **¿Cómo se verifica?** — evidencia con capturas.
5. **Notas técnicas** — comportamiento observado, limitaciones y buenas prácticas.

Explicaciones accesibles para un lector con conocimientos básicos de redes y sistemas operativos.

---

## Resumen técnico del laboratorio

| Elemento | Valor configurado |
|---|---|
| Dominio AD / DNS | `servidor1.com` |
| Controlador de dominio | `WS22` — Windows Server 2022, IP estática `192.168.0.20/24` |
| Hipervisor | Oracle VirtualBox, red en modo **Bridged Adapter** |
| Cliente Windows | Windows 10, IP reservada DHCP `192.168.0.51` |
| Cliente Linux | Ubuntu Desktop 24.04 LTS, IP reservada DHCP `192.168.0.50`, unido al dominio con `realmd/sssd` |
| Rango DHCP | `192.168.0.1` a `192.168.0.100`, máscara `255.255.255.0 (/24)` |
| Exclusiones DHCP | `192.168.0.1` (gateway/router) y `192.168.0.20` (servidor-dominio) |
| Lease time | 8 días (valor predeterminado de Windows Server) |
| Usuarios del dominio | `usuario1@servidor1.com` / `usuario2@servidor1.com`, contraseña `Password1` |
| Recurso público | `\\192.168.0.20\Publica` — acceso total para usuarios autenticados (`Everyone`) |
| Recurso privado | `\\192.168.0.20\Privada` — acceso restringido (en laboratorio: `usuario1`; ver nota sobre especificación `privado/privado`) |
| GPO 1 | Fondo de escritorio corporativo obligatorio (logo USAC) |
| GPO 2 | Página de inicio institucional en Microsoft Edge / Google Chrome vía plantillas ADMX / políticas JSON |

---

## 1.1 Servicio de Virtualización

### 1.1.1 ¿Para qué sirve?

La virtualización permite ejecutar múltiples máquinas virtuales (VM) aisladas sobre un mismo hardware físico (*host*), mediante un hipervisor. En este proyecto se utiliza para simular una red empresarial completa sin requerir servidores físicos dedicados:

- **1 Servidor Virtual:** Windows Server 2022 (controlador de dominio, DNS, DHCP, archivos).
- **1 Cliente Virtual:** Ubuntu Desktop 24.04 LTS o Windows 10/11.
### 1.1.2 Implementación en este proyecto

Se optó por **Oracle VirtualBox** como hipervisor, por su disponibilidad, soporte de red en puente y compatibilidad con Windows Server y Linux.

Inventario de VM en el host (ver Figura 1):

- `WS22` — Windows Server 2022.
- `W10` — Windows 10 (cliente de dominio y validación de GPO).
- `Ubuntu24` — Ubuntu 24.04 LTS (cliente Linux unido a AD).

**Figura 1 — Ventana principal de VirtualBox con las máquinas virtuales del laboratorio.**

![Screenshot 20260923 195042](Attachments/Screenshot_20260923_195042.png)
### 1.1.3 Configuración de red: Bridged Adapter

Para la comunicación se configuró la red en modo **Bridge Adapter (adaptador puente)**.

**¿Qué es?**

- En modo puente, la VM se conecta directamente a la red física del host como si fuera un equipo independiente. Obtiene su propia dirección IP del mismo segmento (en este caso `192.168.0.0/24`) y es visible para el router, el servidor y las demás VM.
- Diferencia con NAT: en NAT la VM queda oculta tras el host y no es accesible desde la LAN; en red interna o *host-only* no sale a la red física. El modo puente es el adecuado cuando se requiere que el servidor sea alcanzable por nombre DNS (`servidor1.com`) desde todos los clientes.

**Referencia:** [VirtualBox — Bridged networking](https://www.virtualbox.org/manual/ch06.html#network_bridged)

---

## 1.2 Servidor de Identidad y Autenticación (Active Directory)

> **Objetivo de la sección:** Implementar Active Directory Domain Services (AD DS) en Windows Server 2022 para la gestión centralizada de usuarios en el dominio `servidor1.com`, con DNS integrado, cuentas de dominio y directivas de grupo (GPO).

### 1.2.1 Servicio DNS — Instalación y verificación

**¿Para qué sirve?**

El Sistema de Nombres de Dominio (DNS) traduce nombres legibles (`servidor1.com`) a direcciones IP (`192.168.0.20`). Sin DNS, los clientes no podrían localizar el controlador de dominio, unirse al dominio ni resolver recursos SMB (`\\servidor1.com\Publica`). En este diseño, el propio controlador de dominio es el **servidor DNS principal** del laboratorio.

**Requisito previo:** IP estática en el servidor (`192.168.0.20`, máscara `255.255.255.0`, DNS preferido `127.0.0.1` o su propia IP).

**Procedimiento ejecutado:**

1. Abrir **Server Manager > Dashboard > (2) Add roles and features**. Esta opción inicia el asistente para agregar roles y características al servidor.

**Figura 2 — Server Manager. La opción “2 Add roles and features” permite agregar AD DS, DNS y DHCP.**

![Screenshot 20260923 194218](Attachments/Screenshot_20260923_194218.png)

2. En *Server Roles* seleccionar **Active Directory Domain Services**, **DNS Server** y **DHCP Server** (DHCP se utilizará en la sección 1.3, pero se instala en el mismo paso para optimizar).

3. Completar el asistente con los valores predeterminados hasta finalizar la instalación. El proceso es guiado; basta conocer qué roles requiere cada servicio.

4. Verificar en el Dashboard que los roles aparecen como instalados y operativos (barra verde de *Manageability*).

**Figura 3 — Roles AD DS, DHCP y DNS instalados y activos.**

![Screenshot 20260923 162514](Attachments/Screenshot_20260923_162514.png)

**Alternativa por PowerShell (equivalente, útil para auditoría y Fase 2):**

```powershell
# Instalación de roles AD DS y DNS con herramientas de administración
Install-WindowsFeature -Name AD-Domain-Services, DNS -IncludeManagementTools

# Promoción del servidor como nuevo bosque (crea el dominio servidor1.com con DNS integrado)
Install-ADDSForest -DomainName "servidor1.com" -InstallDns:$true
```

Después de la promoción el servidor se reinicia y queda como controlador de dominio del bosque raíz `servidor1.com`.

**Verificación DNS:**

Desde el cliente Linux se ejecutó `ping` hacia el nombre de dominio para comprobar resolución y conectividad:

```bash
ping servidor1.com
```

Una respuesta sin pérdida de paquetes confirma que el nombre resuelve a `192.168.0.20` y que existe conectividad de nivel 3.

**Figura 4 — Ping a `servidor1.com` desde Linux. Responde con la IP `192.168.0.20` asignada al servidor.**

![Screenshot 20260923 155954 1](Attachments/Screenshot_20260923_155954%201.png)

> **Nota:** Toda prueba y navegación del proyecto debe realizarse a través del dominio `servidor1.com` (resolución DNS), no mediante IP directa, según las restricciones técnicas del enunciado).

**Referencias:**
- [AD DS Overview — Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview)
- [DNS Overview — Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/networking/dns/dns-top)

### 1.2.2 Servicio Active Directory Domain Services (AD DS)

**¿Para qué sirve?**

AD DS es el servicio de directorio de Microsoft. Centraliza la autenticación (quién puede entrar), la autorización (a qué puede acceder) y la aplicación de directivas (qué configuración recibe). Sin AD, cada equipo tendría cuentas locales aisladas; con AD, `usuario1@servidor1.com` es válido en todos los equipos unidos al dominio.

**Configuración del dominio:**

1. Tras instalar el rol AD DS, hacer clic en la notificación (bandera amarilla) **“Promote this server to a domain controller”**.
2. Elegir **Add a new forest > Root domain name: `servidor1.com`**.
3. Definir contraseña de restauración DSRM, aceptar valores de NetBIOS (`SERVIDOR1`), rutas NTDS/SYSVOL y revisar prerrequisitos.
4. Instalar y reiniciar. Al iniciar sesión, el dominio será `SERVIDOR1`.

Para administración diaria se utiliza la consola **Active Directory Users and Computers (`dsa.msc`)**, donde se crean Unidades Organizativas (OU), usuarios y grupos.

> **Buena práctica aplicada:** se creó una **Unidad Organizativa (OU)** dedicada para aplicar las GPO solo a los usuarios del laboratorio, evitando afectar a `Domain Controllers` o cuentas administrativas. Las GPO se enlazan a la OU, no al dominio completo.

### 1.2.3 Cuentas de usuario del dominio

| Usuario | UPN (Principal Name) | Contraseña |
|---|---|---|
| usuario1 | usuario1@servidor1.com | Password1 |
| usuario2 | usuario2@servidor1.com | Password1 |

> **Nota:** La especificación original indica `Password`, pero se utilizó `Password1` (agregando el `1`) por limitaciones de la política de contraseñas del servidor, que exige complejidad mínima (mayúsculas, minúsculas y dígito).

**Procedimiento (`dsa.msc > OU > New > User`):**

1. Crear `usuario1` vinculado al dominio. Las siguientes figuras, tomadas de la actividad de verificación del 17/08/2026, muestran el alta y la asignación de contraseña.

![Creación del usuario1](Attachments/Screenshot_20260917_130447.png)

![Asignación de contraseña al usuario1](Attachments/Screenshot_20260917_130541.png)

2. Repetir para `usuario2`. La consola muestra ambos usuarios ya creados y habilitados.

![Usuarios usuario1 y usuario2 creados](Attachments/Screenshot_20260917_131026.png)

> Las cuentas quedan habilitadas para iniciar sesión en cualquier equipo unido al dominio. 

### 1.2.4 Unión de clientes al dominio (Windows y Linux)

#### Cliente Windows 10

1. En el cliente, configurar como DNS preferido la IP del servidor (`192.168.0.20`).
2. `System > About > Join a domain` o por PowerShell (ejecutado como administrador):

```powershell
Add-Computer -DomainName "servidor1.com" -Credential (Get-Credential) -Restart
```

3. Reiniciar. En la pantalla de inicio de sesión usar **Other user** e ingresar `usuario1@servidor1.com` / `Password1`. El texto `Sign in to: SERVIDOR1` confirma que autentica contra el dominio y no contra una cuenta local.

Para distinguir una cuenta local de una de dominio, en Windows puede usarse `.\usuario_local` (cuenta local) frente a `servidor1\usuario1` o `usuario1@servidor1.com` (cuenta de dominio).

```powershell
# Ver usuarios locales del equipo (no son del dominio)
Get-LocalUser
```

#### Cliente Linux — Ubuntu 24.04 LTS unido a AD con realmd/sssd

**¿Para qué sirve?** Permite que los usuarios de AD inicien sesión en Ubuntu con las mismas credenciales, cumpliendo la restricción de contar con un cliente Linux unido al dominio que demuestre al menos 2 funcionalidades (en este laboratorio: resolución DNS + autenticación AD + acceso SMB + políticas de Chrome).

**Procedimiento ejecutado:**

```bash
# 1. Instalar paquetes de integración
sudo apt update && sudo apt install -y realmd sssd sssd-tools adcli krb5-user

# 2. Descubrir el dominio en la red (debe resolver vía DNS a 192.168.0.20)
realm discover servidor1.com

# 3. Unirse al dominio (solicita credencial de Administrador del dominio)
sudo realm join -U 'Administrador' servidor1.com

# 4. Habilitar creación automática de /home al primer inicio de sesión
sudo pam-auth-update --enable mkhomedir

# 5. Reiniciar el servicio de identidades
sudo systemctl restart sssd
```

El paso 4 corresponde a la Figura 5. Sin este ajuste, el inicio de sesión de un usuario de dominio en Linux falla o queda sin directorio personal.

**Figura 5 — Configuración PAM en Ubuntu. La opción “Create home directory on login” indica a Linux que debe crear automáticamente los directorios `/home/usuario` en el primer inicio de sesión.**

![Screenshot 20260923 144048 1](Attachments/Screenshot_20260923_144048%201.png)

**Referencias:**
- [Ubuntu — Join Active Directory (realmd/sssd)](https://ubuntu.com/server/docs/service-sssd-ad)
- [SSSD Documentation](https://sssd.io/docs/introduction.html)

### 1.2.5 Validación de autenticación

Se realizaron tres niveles de validación (evidencia de la actividad del 17/08/2026):

**a) Validación local en el servidor (`runas`):**

```cmd
runas /user:usuario1@servidor1.com cmd.exe
```

Al ingresar la contraseña se abrió una consola con título `cmd.exe (running as usuario1@servidor1.com)`, lo que confirma que las credenciales son válidas y que la cuenta puede iniciar procesos.

![Validación local con runas](Attachments/Screenshot_20260917_162834.png)

**b) Inicio de sesión en cliente Windows 10:**

Selección de `Other user`, ingreso de credenciales de `usuario1`, indicador `Sign in to: SERVIDOR1`. El escritorio cargó sin errores con el perfil del dominio.

![Inicio de sesión en cliente](Attachments/Screenshot_20260917_170622.png)

**c) Verificación de identidad efectiva (`whoami`):**

```cmd
whoami
:: resultado esperado:
:: servidor1\usuario1
```

Esto demuestra que la sesión activa corresponde al usuario del dominio y no a un usuario local, validando de extremo a extremo la creación de usuarios, la unión al dominio y la autenticación.

![Verificación whoami](Attachments/Screenshot_20260917_170826.png)

### 1.2.6 GPO — Fondo de pantalla corporativo obligatorio

**¿Para qué sirve?** Las Directivas de Grupo (GPO, *Group Policy Objects*) permiten aplicar configuración centralizada a usuarios y equipos. En este caso se impone el logotipo de la Universidad como fondo de escritorio al iniciar sesión, sin que el usuario pueda cambiarlo permanentemente.

**Diseño:** GPO enlazada a la OU de usuarios del laboratorio (ver §1.2.2).

**Procedimiento (`gpmc.msc`):**

1. Crear GPO (ej. `GPO_FondoPantalla`) y enlazarla a la OU.
2. Editar: `User Configuration > Policies > Administrative Templates > Desktop > Desktop > Desktop Wallpaper`.
3. Habilitar (*Enabled*), indicar ruta UNC del fondo (ej. `\\servidor1.com\Publica\FondoUsac.jpg`) y estilo (Fill/Centered).
4. Cerrar sesión y volver a iniciarla en el cliente para aplicarla.

**Figura 6 — OU creada para aplicar las GPO a un grupo de usuarios.**

![Screenshot 20260923 163755 1](Attachments/Screenshot_20260923_163755%201.png)

**Figura 7 — Configuración del fondo en `User Policies > Templates > Desktop > Desktop Wallpaper`.**

![Screenshot 20260923 163951 1](Attachments/Screenshot_20260923_163951%201.png)

**Figura 8 — Verificación en cliente Windows 10: fondo institucional aplicado tras inicio de sesión con usuario de dominio.**

![Screenshot 20260923 162700 1](Attachments/Screenshot_20260923_162700%201.png)

### 1.2.7 GPO organizacional — Navegador (Microsoft Edge / Google Chrome)

**¿Para qué sirve?** Estandarizar el navegador corporativo: establecer la página de inicio institucional, definir comportamiento al abrir el navegador y, opcionalmente, bloquear extensiones no autorizadas o restringir almacenamiento masivo USB. Es la “GPO organizacional” exigida en el enunciado.

**¿Qué son las plantillas ADMX?** Son archivos de definición de directivas (`*.admx` + `*.adml` de idioma) que extienden el Editor de directivas con opciones específicas de un producto (Edge, Chrome). Sin importarlas, esas opciones no aparecen en `gpedit/gpmc`.

**Descarga e instalación (Edge):**

1. Descargar **Microsoft Edge Policy Files** (versión correspondiente al Edge instalado) desde [Microsoft Edge Enterprise — Download policy files](https://www.microsoft.com/en-us/edge/business/download).
2. Copiar `windows/admx/*.admx` a `C:\Windows\PolicyDefinitions\` y `windows/admx/es-ES/*.adml` (o `en-US`) a `C:\Windows\PolicyDefinitions\es-ES\`.
3. Abrir `gpmc.msc`. Aparecerá `User/Computer Configuration > Policies > Administrative Templates > Microsoft Edge > Startup, home page and new tab page`.

**Configuración aplicada en laboratorio:**

- `Action to take on startup` → **Open a list of URLs**.
- `Sites to open when the browser starts` → URL institucional (sitio de la Universidad).

**Figura 9 — Tras instalar las plantillas de Edge: `User Configuration > Startup, home page and new tab page`.**

![Screenshot 20260923 142802 1](Attachments/Screenshot_20260923_142802%201.png)

**Figura 10 — `Action to take on startup`: indicar que debe abrir una lista de vínculos.**

![Screenshot 20260923 142855 1](Attachments/Screenshot_20260923_142855%201.png)

> A continuación se configuró `Sites to open when the browser starts`, agregando el enlace institucional. Al abrir Edge, el navegador redirige en primer lugar al sitio de la institución.

**Figura 11 — Comprobación: al abrir el navegador dirige al sitio institucional.**

![Screenshot 20260923 162733 1](Attachments/Screenshot_20260923_162733%201.png)

**Figura 12 — Verificación en `edge://policy`: aparecen aplicadas las directivas configuradas.**

![Screenshot 20260923 162757 1](Attachments/Screenshot_20260923_162757%201.png)

**Cliente Linux — Google Chrome:**

No existe compatibilidad directa de GPO de Windows Server con Linux. En Ubuntu, Chrome se gestiona mediante **políticas JSON administradas**:

- Ruta: `/etc/opt/chrome/policies/managed/institucional.json`
- Ejemplo:

```json
{
  "HomepageLocation": "https://www.institucion.edu.gt",
  "RestoreOnStartup": 4,
  "RestoreOnStartupURLs": ["https://www.institucion.edu.gt"]
}
```

**Figura 13 — Chrome en cliente Linux con página institucional.**

![Screenshot 20260923 155249 1](Attachments/Screenshot_20260923_155249%201.png)

**Figura 14 — Verificación en `chrome://policy`: políticas aplicadas.**

![Screenshot 20260923 155348 1](Attachments/Screenshot_20260923_155348%201.png)

> **Nota de compatibilidad:** El fondo de pantalla no pudo implementarse en Linux por falta de equivalencia nativa de GPO. Windows 10 sí aplica ambas GPO de forma nativa sin problemas de compatibilidad.

> **Nota operativa:** En Windows 10 las GPO no siempre se aplican instantáneamente. Para forzar la actualización ejecutar como administrador y luego cerrar e iniciar sesión nuevamente.

**Figura 15 — `gpupdate /force` para forzar la aplicación de directivas.**

![Screenshot 20260923 094809 1](Attachments/Screenshot_20260923_094809%201.png)

**Referencias:**
- [Configure Microsoft Edge policy settings — Microsoft Learn](https://learn.microsoft.com/en-us/deployedge/configure-microsoft-edge)
- [Chrome Enterprise Policy List](https://chromeenterprise.google/policies/)
- [Group Policy Overview — Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/group-policy/group-policy-overview)

---

## 1.3 Servidor de Asignación Dinámica de IP (DHCP)

### 1.3.1 ¿Para qué sirve?

El Protocolo de Configuración Dinámica de Host (DHCP) asigna automáticamente direcciones IP, máscara, puerta de enlace y servidores DNS a los clientes. Evita conflictos por configuración manual y centraliza el control del direccionamiento.

- **Rango:** `192.168.0.1` a `192.168.0.100`.
- **Exclusiones, tiempo de concesión (*lease time*) y opciones DHCP (DNS Server y Gateway).**
### 1.3.2 Instalación del rol

Se reutiliza el rol DHCP instalado. Una vez instalado, se accede desde **Server Manager > Tools > DHCP** (consola `dhcpmgmt.msc`).

**Figura 16 — Acceso a la consola DHCP desde `Tools`.**

![Screenshot 20260923 210217](Attachments/Screenshot_20260923_210217.png)

### 1.3.3 Creación del ámbito (Scope)

1. Desplegar el servidor, clic derecho sobre **IPv4 > New Scope** para iniciar el asistente.

**Figura 17 — Selección de `New Scope`.**

![Screenshot 20260923 113256](Attachments/Screenshot_20260923_113256.png)

2. Asignar nombre descriptivo. En este laboratorio: **`Ambito_Red_Local`**.

**Figura 18 — Nombre del ámbito: `Ambito_Red_Local`.**

![Screenshot 20260923 113411](Attachments/Screenshot_20260923_113411.png)

3. Definir el rango `192.168.0.1` – `192.168.0.100`, longitud `24`, máscara `255.255.255.0`.

**Figura 19 — Rango de IP del ámbito.**

![Screenshot 20260923 113519](Attachments/Screenshot_20260923_113519.png)

4. Agregar **exclusiones**: direcciones que DHCP nunca debe entregar porque ya están asignadas estáticamente. Se excluyó `192.168.0.1` (router/gateway) y `192.168.0.20` (servidor-dominio). El asistente permite excluir un rango o direcciones individuales.

**Figura 20 — Exclusiones: `.1` (router) y `.20` (servidor).**

![Screenshot 20260923 113839](Attachments/Screenshot_20260923_113839.png)

5. Configurar **Lease Duration** (duración de la concesión: tiempo durante el cual el cliente conserva la IP). Se mantuvo el valor predeterminado de **8 días**, adecuado para una red estable con equipos fijos.

**Figura 21 — Duración de concesión (8 días).**

![Screenshot 20260923 113953](Attachments/Screenshot_20260923_113953.png)

6. Configurar opciones DHCP. Cuando el asistente pregunta por el **DNS**, indicar la IP del servidor.

**Figura 22 — Configuración de DNS en el asistente.**

![Screenshot 20260923 114049](Attachments/Screenshot_20260923_114049.png)

7. Agregar datos del dominio: nombre `servidor1.com` y dirección `192.168.0.20`. También debe configurarse la opción **003 Router (Gateway)** con la IP del router (`192.168.0.1`).

**Figura 23 — Dominio `servidor1.com` con IP `192.168.0.20`.**

![Screenshot 20260923 114302](Attachments/Screenshot_20260923_114302.png)

8. Finalizar y **activar** el ámbito. A partir de este momento el servidor puede arrendar direcciones.

**Figura 24 — Ámbito creado y finalizado. Desde aquí pueden crearse reservas.**

![Screenshot 20260923 114657](Attachments/Screenshot_20260923_114657.png)

### 1.3.4 Reservas por dirección MAC

Para garantizar que equipos críticos reciban siempre la misma IP (comportamiento tipo estático pero gestionado por DHCP), se crean **reservas** vinculadas a la dirección MAC.

Asignación del laboratorio (dentro del rango permitido):

- Cliente Linux → `192.168.0.50`
- Cliente Windows → `192.168.0.51`

**¿Qué es una dirección MAC?** Es el identificador físico único de 48 bits de cada tarjeta de red (ej. `08:00:27:xx:xx:xx` en VirtualBox). DHCP la utiliza para reconocer al cliente y entregarle la IP reservada.

**¿Cómo obtenerla?**

- **Ubuntu 24.04:**

```bash
ip link show
# o
ip a
# la MAC aparece como link/ether XX:XX:XX:XX:XX:XX
```

- **Windows 10 (CMD):**

```cmd
ipconfig /all
:: Buscar "Dirección física" del adaptador Ethernet
:: Alternativa:
getmac /v
```

**Figura 25 — Reserva configurada (se requiere la MAC de cada cliente).**

![Screenshot 20260923 210345](Attachments/Screenshot_20260923_210345.png)

Procedimiento: `DHCP > Scope > Reservations > New Reservation > Nombre + IP + MAC > Add`. Luego, en cada cliente, renovar la concesión:

```bash
# Linux
sudo dhclient -r && sudo dhclient
ip a
ping servidor1.com
```

```cmd
:: Windows
ipconfig /release
ipconfig /renew
ipconfig /all
```

**Figura 26 — Verificación en Linux con `ip a`: el equipo posee la IP terminada en `.50`.**

![Screenshot 20260923 121339 1](Attachments/Screenshot_20260923_121339%201.png)

> **Nota:** Si la reserva no se aplica, verificar que la MAC ingresada no contenga guiones ni errores de transcripción, que el ámbito esté activado y que no exista una exclusión que cubra la IP reservada.

**Referencias:**
- [DHCP Overview — Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/dhcp-top)
- [Manage DHCP scopes](https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/dhcp-manage-scope)

---

## 1.4 Servidor de Archivos y Recursos Compartidos (File Server)

### 1.4.1 ¿Para qué sirve?

El File Server centraliza documentos y recursos mediante el protocolo **SMB/CIFS** (*Server Message Block*), el estándar de Windows para carpetas compartidas en red. El acceso se controla en dos capas complementarias:

1. **Permisos de recurso compartido (*Share permissions*):** controlan el acceso a través de la red (`Everyone: Full Control` vs. usuarios específicos).
2. **Permisos NTFS (*Security*):** controlan el acceso a nivel de sistema de archivos (lectura, escritura, modificación, control total), incluso en acceso local. Aplican ACL (*Access Control Lists*).

**Regla efectiva:** el permiso real es el más restrictivo entre ambas capas. Por ello deben configurarse de forma coherente.

- **CarpetaPrivada:** acceso restringido. Credenciales exclusivas `privado/privado` con permisos de lectura, modificación y eliminación.
- **CarpetaPública:** abierta a todos los usuarios del dominio, con permisos de lectura, creación, modificación y eliminación.

> **Nota de adecuación del laboratorio:** para demostrar la denegación, la carpeta privada se restringió a `usuario1` (control total) sin conceder acceso a `usuario2`. El modelo es equivalente al especificado (`usuario exclusivo con control total`); si se requiere conformidad literal, crear el usuario `privado` en AD y sustituirlo por `usuario1` en los pasos siguientes. El procedimiento de permisos es idéntico.

### 1.4.2 Estructura base

En la raíz del servidor (ej. `C:\`) se creó el directorio **`ServiciosArchivos`**, contenedor de los dos recursos.

**Figura 27 — Raíz del servidor Windows Server 2022 donde se crea la estructura.**

![Screenshot 20260923 121557](Attachments/Screenshot_20260923_121557.png)

**Figura 28 — Carpetas `Publica` y `Privada` dentro de `ServiciosArchivos`.**

![Screenshot 20260923 121725](Attachments/Screenshot_20260923_121725.png)

Ruta física sugerida:

```text
C:\ServiciosArchivos\Publica
C:\ServiciosArchivos\Privada
```

### 1.4.3 Configuración — Carpeta pública

1. Clic derecho sobre `Publica > Properties > Sharing > Advanced Sharing > Share this folder`. Nombre de recurso: `Publica`.
2. En *Permissions*, conceder a **Everyone: Full Control / Change / Read**. Este directorio no requiere permisos especiales; cualquier usuario autenticado (`usuario1`, `usuario2`) puede acceder.

**Figura 29 — Propiedades de uso compartido de la carpeta pública.**

![Screenshot 20260923 122051](Attachments/Screenshot_20260923_122051.png)

3. En la pestaña *Security*, conceder control total a usuarios autenticados para que NTFS no bloquee lo permitido en *Share*.

**Figura 30 — Pestaña Seguridad de la carpeta pública (control total).**

![Screenshot 20260923 122231](Attachments/Screenshot_20260923_122231.png)

### 1.4.4 Configuración — Carpeta privada

1. Repetir el proceso de compartición para `Privada`, pero en *Share Permissions* otorgar control total únicamente a `usuario1` (o al usuario `privado` si se implementa la variante literal). Eliminar o no incluir a `Everyone` ni a `usuario2`.

**Figura 31 — Propiedades de uso compartido de la carpeta privada.**

![Screenshot 20260923 122419](Attachments/Screenshot_20260923_122419.png)

2. En *Security*, remover permisos generales heredados y otorgar control total solo a los usuarios autorizados (y a `Administrators`/`SYSTEM` para administración). Esto garantiza que, aunque alguien adivine la ruta UNC, NTFS deniegue el acceso.

**Figura 32 — Pestaña Seguridad de la carpeta privada (solo usuarios específicos).**

![Screenshot 20260923 122613](Attachments/Screenshot_20260923_122613.png)

> **Advertencia:** No basta con configurar solo *Share*; si NTFS conserva `Everyone`, el recurso queda expuesto en acceso local o ante reconfiguraciones. Siempre deben revisarse ambas pestañas.

### 1.4.5 Verificación desde clientes

#### Cliente Windows (con `usuario2`)

Desde el Explorador de archivos acceder a la IP del dominio:

```text
\\192.168.0.20\
```

**Figura 33 — En la ruta se observan `Publica` y `Privada`.**

![Screenshot 20260923 123110](Attachments/Screenshot_20260923_123110.png)

- Al abrir `Publica` no hay restricción; se visualiza su contenido.
- Al abrir `Privada` con `usuario2` (no autorizado) el sistema muestra **Acceso denegado**, lo que confirma que la ACL funciona.

**Figura 34 — Acceso denegado a la carpeta privada para `usuario2`.**

![Screenshot 20260923 123309](Attachments/Screenshot_20260923_123309.png)

#### Cliente Linux

Desde el gestor de archivos (Nautilus) acceder vía SMB:

```text
smb://192.168.0.20/Publica
smb://192.168.0.20/Privada
```

Autenticarse con credenciales de dominio cuando se solicite (`usuario2` para pública, `usuario1` para privada).

**Figura 35 — Carpeta pública con archivos disponibles (acceso con `usuario2`).**

![Screenshot 20260923 155720](Attachments/Screenshot_20260923_155720.png)

**Figura 36 — Carpeta privada con su contenido (acceso con `usuario1`).**

![Screenshot 20260923 162237](Attachments/Screenshot_20260923_162237.png)

Alternativamente puede montarse por terminal:

```bash
# Acceso temporal con gio
gio mount smb://usuario1@servidor1.com/Privada

# Montaje persistente (requiere cifs-utils)
sudo apt install -y cifs-utils
sudo mount -t cifs //192.168.0.20/Publica /mnt/publica -o username=usuario2,domain=servidor1
```

#### Nota sobre acceso por nombre DNS

También es posible acceder mediante el nombre `servidor1.com` en lugar de la IP:

```text
\\servidor1.com\Publica
smb://servidor1.com/Publica
```

**Figura 37 — Acceso utilizando `servidor1.com` en lugar de la IP.**

![Screenshot 20260923 162932](Attachments/Screenshot_20260923_162932.png)

Comportamiento observado:

- **En Linux** funciona correctamente.
- **En el cliente Windows del laboratorio** la ruta abre y muestra los directorios, pero no lista los archivos almacenados (comportamiento anómalo específico del entorno, posiblemente por caché de credenciales, resolución LLMNR o perfil de red).

**Figura 38 — Ruta por nombre: se visualizan los directorios, pero no su contenido (limitación observada en Windows).**

![Screenshot 20260923 163008](Attachments/Screenshot_20260923_163008.png)

> **Recomendación para la defensa:** demostrar primero `ping servidor1.com` (prueba DNS), luego acceso SMB por IP y finalmente por nombre, explicando la limitación detectada. Esto evidencia dominio técnico y honestidad en los resultados.

**Referencias:**
- [SMB in Windows Server — Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/storage/file-server/file-server-smb-overview)
- [Share and NTFS permissions](https://learn.microsoft.com/en-us/windows-server/storage/file-server/ntfs-overview)
- [Samba / CIFS (Ubuntu)](https://ubuntu.com/server/docs/samba-introduction)

---

## Conclusiones de Fase 1

1. La virtualización con VirtualBox y red en puente proporciona una base estable para simular una red empresarial con un único host físico.
2. AD DS + DNS constituyen la columna vertebral: sin resolución de `servidor1.com` ningún otro servicio (unión al dominio, DHCP con actualización DNS, SMB) opera de forma fiable.
3. La validación `runas` + inicio de sesión en cliente + `whoami = servidor1\usuario1` demuestra autenticación de extremo a extremo.
4. DHCP con ámbito `Ambito_Red_Local`, exclusiones y reservas por MAC garantiza direccionamiento ordenado y predecible (`.20` servidor, `.50` Linux, `.51` Windows).
5. El File Server con doble capa Share + NTFS cumple el principio de mínimo privilegio: recurso público abierto y recurso privado denegado correctamente a usuarios no autorizados.
6. Las GPO de fondo y navegador centralizan la imagen institucional; la limitación en Linux (sin GPO nativa, políticas JSON para Chrome, sin fondo).

El entorno queda listo como prerrequisito para la Fase 2 (correo, IIS, FTP), que reutilizará usuarios, DNS y DHCP aquí configurados.

---

## Anexo A — Tabla resumen de direccionamiento y cuentas

| Host | IP / Máscara | Gateway | DNS | Observación |
|---|---|---|---|---|
| WS22 (DC, DNS, DHCP, Files) | 192.168.0.20 /24 | 192.168.0.1 | 127.0.0.1 (sí mismo) | Excluida del DHCP |
| Router/Gateway | 192.168.0.1 | — | — | Excluida del DHCP |
| Ubuntu24 | 192.168.0.50 (reserva) | 192.168.0.1 | 192.168.0.20 | MAC registrada en DHCP |
| W10 | 192.168.0.51 (reserva) | 192.168.0.1 | 192.168.0.20 | MAC registrada en DHCP |
| Pool dinámico restante | 192.168.0.2–.100 (menos excluidas/reservadas) | — | — | Lease 8 días |

| Cuenta AD | UPN | Uso en Fase 1 |
|---|---|---|
| usuario1 | usuario1@servidor1.com | Acceso a Publica + Privada, validación whoami |
| usuario2 | usuario2@servidor1.com | Acceso solo a Publica (denegado en Privada para demostrar ACL) |
| Administrador | Administrador@servidor1.com | Unión de equipos al dominio, gestión GPO/DHCP |

---

## Anexo B — Comandos de referencia rápida

```powershell
# --- Windows Server (PowerShell admin) ---
Install-WindowsFeature -Name AD-Domain-Services, DNS, DHCP -IncludeManagementTools
Install-ADDSForest -DomainName "servidor1.com" -InstallDns:$true
Add-Computer -DomainName "servidor1.com" -Credential (Get-Credential) -Restart
Get-LocalUser
gpupdate /force
whoami
runas /user:usuario1@servidor1.com cmd.exe
ipconfig /all
ipconfig /release; ipconfig /renew
```

```bash
# --- Ubuntu 24.04 ---
sudo apt update && sudo apt install -y realmd sssd sssd-tools adcli krb5-user cifs-utils
realm discover servidor1.com
sudo realm join -U 'Administrador' servidor1.com
sudo pam-auth-update --enable mkhomedir
sudo systemctl restart sssd
ip link show; ip a
sudo dhclient -r && sudo dhclient
ping servidor1.com
gio mount smb://usuario1@servidor1.com/Privada
cat /etc/opt/chrome/policies/managed/institucional.json
```

---

## Anexo C — Referencias y documentación oficial

- VirtualBox Networking — Bridged: https://www.virtualbox.org/manual/ch06.html#network_bridged
- AD DS Overview: https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview
- DNS Overview: https://learn.microsoft.com/en-us/windows-server/networking/dns/dns-top
- DHCP Overview: https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/dhcp-top
- Group Policy Overview: https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/group-policy/group-policy-overview
- Microsoft Edge — Download / Configure policies: https://www.microsoft.com/en-us/edge/business/download — https://learn.microsoft.com/en-us/deployedge/configure-microsoft-edge
- Chrome Enterprise Policy List: https://chromeenterprise.google/policies/
- Ubuntu — SSSD + Active Directory: https://ubuntu.com/server/docs/service-sssd-ad
- SMB / File Server: https://learn.microsoft.com/en-us/windows-server/storage/file-server/file-server-smb-overview
- NTFS / Share permissions: https://learn.microsoft.com/en-us/windows-server/storage/file-server/ntfs-overview

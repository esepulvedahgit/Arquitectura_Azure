# Lab Azure: Moodle LMS + IAM con Entra ID
> **Certificaciones objetivo:** AZ-104 · SC-300  
> **Dominio:** `cloudlab.cl` · **Tenant:** Microsoft Entra ID  
> **Región:** West US 3

---

## Índice

1. [Descripción del laboratorio](#1-descripción-del-laboratorio)
2. [Arquitectura](#2-arquitectura)
3. [Infraestructura Azure](#3-infraestructura-azure)
4. [Identidad y Control de Acceso (IAM)](#4-identidad-y-control-de-acceso-iam)
5. [Seguridad Externa](#5-seguridad-externa)
6. [Flujo de tráfico](#6-flujo-de-tráfico)
7. [Competencias certificación cubiertas](#7-competencias-certificación-cubiertas)

---

## 1. Descripción del laboratorio

Laboratorio práctico que despliega una instancia de **Moodle 5.0.2** sobre infraestructura Azure, integrando:

- Gestión de identidades con **Microsoft Entra ID** y sincronización desde AD on-premises
- Control de acceso basado en roles (**Azure RBAC**) mediante grupos de seguridad
- Almacenamiento de objetos en **Azure Blob Storage** con Managed Identity
- Base de datos **MySQL Flexible Server** en red privada
- Seguridad perimetral con **Cloudflare WAF** y **CrowdSec**

---

## 2. Arquitectura

![Diagrama de arquitectura](./arquitectura_azure.png)



---

## 3. Infraestructura Azure

### 3.1 Grupo de Recursos

| Parámetro | Valor |
|---|---|
| Nombre | `Moodle` |
| Región | `West US 3` |
| Suscripción | `Suscripción de Azure 1` |

### 3.2 Red Virtual

| Recurso | Tipo | CIDR |
|---|---|---|
| `moodle` | Virtual Network | `10.0.0.0/16` |
| `vm-moodle` | Subnet | `10.0.0.0/24` |
| `db-moodle` | Subnet | `10.0.1.0/24` |
| `vm-moodle-nsg` | Network Security Group | Asociado a subnet `vm-moodle` |

### 3.3 Máquina Virtual

| Parámetro | Valor |
|---|---|
| Nombre | `vm-moodle` |
| SO | Ubuntu 24.04 LTS |
| Subnet | `vm-moodle` (10.0.0.0/24) |
| IP Pública | `vm-moodle-ip` (puerto SSH 22) |
| Stack web | Apache 2.4 + PHP 8.3 |
| Aplicación | Moodle 5.0.2 (Git clone) |
| Identidad | `identity-moodle` (User-Assigned Managed Identity) |

### 3.4 Base de Datos

| Parámetro | Valor |
|---|---|
| Nombre | `dbmoodle` |
| Tipo | Azure MySQL Flexible Server |
| Subnet | `db-moodle` (10.0.1.0/24) |
| IP Privada | `10.0.1.4` |
| Acceso público | Deshabilitado |
| DNS Privado | `dbmoodle.private.mysql.database.azure.com` |

> **Workaround aplicado:** entrada manual en `/etc/hosts` de `vm-moodle` para resolución DNS privada.

### 3.5 Almacenamiento

| Parámetro | Valor |
|---|---|
| Cuenta | `moodlestorageesh` |
| Contenedor | `moodle-data` (acceso privado) |
| Autenticación | SAS Token |
| Rol asignado | `Storage Blob Data Contributor` → `identity-moodle` |
| Plugin Moodle | `tool_objectfs` + `local_azureblobstorage` |

> **Importante:** Shared Key debe estar habilitado para que los tokens SAS funcionen.

---

## 4. Identidad y Control de Acceso (IAM)

### 4.1 Diagrama de identidades

```
On-premises Active Directory
    └── Usuarios: usertest01–usertest20
          │
          ▼ Entra Connect (Password Hash Sync)
Microsoft Entra ID (cloudlab.cl)
    └── Grupos cloud-only: grp-azure-admins
    └──                    grp-azure-engineers
    └──                    grp-azure-readers
          │  Membresía manual: usuarios sincronizados asignados a grupos cloud
          ▼ Azure RBAC
Suscripción / Resource Group Moodle
```

### 4.2 Grupos de seguridad

| Grupo | Origen | Rol ARM | Scope |
|---|---|---|---|
| `grp-azure-admins` | **Cloud-only (Entra ID)** | `Colaborador` | Suscripción |
| `grp-azure-engineers` | **Cloud-only (Entra ID)** | `Colaborador` | RG `Moodle` |
| `grp-azure-readers` | **Cloud-only (Entra ID)** | `Lector` | Suscripción |

> **Nota:** Los grupos se crean y gestionan directamente en Entra ID. Los usuarios sincronizados desde AD on-prem se agregan manualmente como miembros de estos grupos cloud-only.

### 4.3 Managed Identity

| Recurso | Tipo | Rol | Scope |
|---|---|---|---|
| `identity-moodle` | User-Assigned | `Storage Blob Data Contributor` | `moodlestorageesh` |

> **Principio aplicado:** Least Privilege — la identidad solo tiene acceso al recurso que necesita.

### 4.4 Entra Connect

| Parámetro | Valor |
|---|---|
| Método de sincronización | Password Hash Sync (PHS) |
| UPN suffix sincronizado | `@cloudlab.cl` |
| Usuarios sincronizados | `usertest01` – `usertest20` |
| Estado | Activo |

### 4.5 Referencia de roles ARM (AZ-104)

| Rol | Gestiona recursos | Asigna roles | Scope típico |
|---|---|---|---|
| `Propietario` | ✓ | ✓ | Break-glass / Admin |
| `Colaborador` | ✓ | ✗ | Grupos operacionales |
| `Administrador de acceso de usuario` | ✗ | ✓ | Delegación IAM |
| `Lector` | Solo lectura | ✗ | Auditoría / Monitoring |

### 4.6 Gaps identificados (roadmap SC-300)

- [ ] Configurar **PIM** (Privileged Identity Management) para activación just-in-time
- [ ] Crear **Administrative Units** para segmentar `usertest*` por grupo
- [ ] Implementar **Conditional Access** policies
- [ ] Reemplazar cuenta de sync Global Admin por cuenta `Sync_xxxxxxxx` con rol `Directory Synchronization Accounts`
- [ ] Eliminar asignación huérfana `Colaborador de base de datos MySQL ClearDB` en RG `Moodle`
- [ ] Mover `Felipe Cerda` de asignación directa al grupo `grp-azure-readers`

---

## 5. Seguridad Externa

### 5.1 Cloudflare

| Configuración | Valor |
|---|---|
| Dominio | `aula.cloudlab.cl` |
| Modo SSL | Strict (Origen: certificado Cloudflare en `vm-moodle`) |
| Proxy | Activo (naranja) |
| WAF | OWASP Core Ruleset habilitado |
| Cache Rule | Bypass para `/admin/*` y sesiones autenticadas |

### 5.2 CrowdSec

| Parámetro | Detalle |
|---|---|
| Instalación | Agente en `vm-moodle` |
| Bouncer | **Firewall Bouncer (iptables/nftables)** |
| Caso de uso | Protección contra fuerza bruta SSH |
| Mecanismo | Bloqueo de IP a nivel firewall tras intentos fallidos de autenticación SSH |

---

## 6. Flujo de tráfico

```
Usuario Internet
    │
    ▼
Cloudflare (WAF + Proxy HTTPS)
    │  SSL Strict — certificado Origin
    ▼
vm-moodle-ip (IP Pública Azure)
    │  Apache 2.4 + CrowdSec Bouncer
    ▼
Moodle 5.0.2 (PHP 8.3)
    ├──► MySQL Flexible Server (red privada 10.0.1.4)
    │       DNS: dbmoodle.private.mysql.database.azure.com
    └──► Azure Blob Storage (SAS Token / Managed Identity)
              moodlestorageesh / moodle-data
```

---

## 7. Competencias certificación cubiertas

### AZ-104
- [x] Despliegue y configuración de máquinas virtuales
- [x] Configuración de redes virtuales y subnets
- [x] Network Security Groups (NSG)
- [x] Azure Storage — Blob, SAS tokens, niveles de acceso
- [x] MySQL Flexible Server — red privada, DNS privado
- [x] Managed Identity — User-Assigned
- [x] Azure RBAC — asignación de roles a grupos en suscripción y RG

### SC-300
- [x] Microsoft Entra ID — gestión de usuarios y grupos
- [x] Entra Connect — sincronización desde AD on-premises
- [x] RBAC con grupos de seguridad (least privilege)
- [x] Managed Identity como identidad de carga de trabajo
- [ ] PIM — activación just-in-time (pendiente)
- [ ] Conditional Access (pendiente)
- [ ] Administrative Units (pendiente)

---

## Recursos relacionados

- [Documentación Azure RBAC](https://learn.microsoft.com/es-es/azure/role-based-access-control/)
- [Documentación Entra Connect](https://learn.microsoft.com/es-es/entra/identity/hybrid/connect/whatis-azure-ad-connect)
- [MySQL Flexible Server — Red privada](https://learn.microsoft.com/es-es/azure/mysql/flexible-server/concepts-networking-vnet)
- [Managed Identity](https://learn.microsoft.com/es-es/entra/identity/managed-identities-azure-resources/overview)
- [CrowdSec Apache Bouncer](https://docs.crowdsec.net/docs/bouncers/apache/)
- [Cloudflare WAF](https://developers.cloudflare.com/waf/)

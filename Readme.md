# 🔍 AWS VPC Troubleshooting & Flow Logs Analysis

> **Laboratorio práctico:** Diagnóstico y resolución de fallos en configuración de red en Amazon VPC, con análisis de tráfico mediante VPC Flow Logs.

---

## 📋 Tabla de Contenidos

1. [Contexto del Problema](#contexto-del-problema)
2. [Arquitectura del Entorno](#arquitectura-del-entorno)
3. [Task 1: Conexión al CLI Host](#task-1-conexión-al-cli-host)
4. [Task 2: Habilitación de VPC Flow Logs](#task-2-habilitación-de-vpc-flow-logs)
5. [Task 3: Troubleshooting](#task-3-troubleshooting)
   - [Desafío #1 — Sitio web inaccesible (HTTP)](#desafío-1--sitio-web-inaccesible-http)
   - [Desafío #2 — SSH bloqueado](#desafío-2--ssh-bloqueado)
6. [Task 4: Análisis de Flow Logs](#task-4-análisis-de-flow-logs)
7. [Conclusiones](#conclusiones)

---

## Contexto del Problema

El equipo de **Mom & Pop Café** realizó cambios en la configuración de red de su VPC con intención de mejorar la seguridad. Como resultado no previsto, los clientes dejaron de poder acceder al sitio web y el equipo técnico perdió acceso SSH a la instancia del servidor web.

**Síntomas reportados:**
- `ERR_CONNECTION_TIMED_OUT` al cargar el sitio web via HTTP.
- `Connection timed out` al intentar SSH al Web Server.

---

## Arquitectura del Entorno

| Componente | Detalle |
|---|---|
| **VPC1** | VPC del Web Server — con errores de configuración |
| **VPC2** | VPC del CLI Host — sin errores, acceso a internet normal |
| **CLI Host** | Instancia Amazon Linux 2 en VPC2, usada para ejecutar comandos AWS CLI |
| **Web Server** | Instancia EC2 en subred pública de VPC1 (`10.0.1.92`) |
| **IP Web Server** | `13.223.77.43` |
| **VPC1 ID** | `vpc-011033ee710342cce` |

> VPC1 y VPC2 **no están peered**. El CLI Host accede a VPC1 a través de internet, igual que cualquier usuario externo.

Los valores anteriores fueron extraídos del panel de credenciales del laboratorio:

![Panel de credenciales del laboratorio](./img/Details.png)

---

## Task 1: Conexión al CLI Host

Se estableció conexión SSH al CLI Host usando el par de llaves descargado del panel de credenciales del laboratorio.

```bash
ssh -i labsuser.pem ec2-user@<CliHostIP>
```

**Resultado:** Conexión exitosa. Sistema operativo: Amazon Linux 2.

![Conexión SSH al CLI Host](./img/01.png)

A continuación se identificó la región del entorno mediante el servicio de metadatos de la instancia:

```bash
curl http://169.254.169.254/latest/dynamic/instance-identity/document | grep region
```

**Resultado:** `"region": "us-east-1"`

![Verificación de región](./img/02.png)

Luego se configuraron las credenciales del AWS CLI:

```bash
aws configure
```

| Campo | Valor |
|---|---|
| AWS Access Key ID | `AKIAW3MEC4HBSJVSUMIW` |
| AWS Secret Access Key | *(credencial del lab)* |
| Default region | `us-east-1` |
| Default output format | `json` |

![Configuración AWS CLI](./img/03.png)

---

## Task 2: Habilitación de VPC Flow Logs

### 2.1 Obtener el ID de VPC1

```bash
aws ec2 describe-vpcs \
  --query 'Vpcs[*].[VpcId,Tags[?Key==`Name`].Value,CidrBlock]' \
  --filters "Name=tag:Name,Values='VPC1'"
```

**Resultado:** `vpc-011033ee710342cce` / CIDR `10.0.0.0/16`

![Describe VPCs](./img/04.png)

### 2.2 Crear bucket S3 para los logs

```bash
aws s3api create-bucket --bucket cubetagv0001 --region us-east-1
```

**Resultado:** Bucket `cubetagv0001` creado exitosamente.

![Creación del bucket S3](./img/05.png)

> ⚠️ Para la región `us-east-1` se omite el parámetro `--create-bucket-configuration` ya que es la región por defecto de S3.

### 2.3 Activar VPC Flow Logs sobre VPC1

```bash
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-011033ee710342cce \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::cubetagv0001
```

**Resultado:** Flow Log creado con ID `fl-00f1fb0d536cbda11`.

![Creación del Flow Log](./img/06.png)

### 2.4 Verificar estado del Flow Log

```bash
aws ec2 describe-flow-logs
```

**Resultado:** `FlowLogStatus: ACTIVE` | `DeliverLogsStatus: SUCCESS`

![Verificación del Flow Log](./img/07.png)

---

## Task 3: Troubleshooting

### Investigación inicial del Web Server

```bash
aws ec2 describe-instances \
  --filter "Name=ip-address,Values='13.223.77.43'" \
  --query 'Reservations[*].Instances[*].[State,PrivateIpAddress,InstanceId,SecurityGroups,SubnetId,KeyName]'
```

**Datos clave extraídos:**

| Campo | Valor |
|---|---|
| Estado | `running` |
| IP Privada | `10.0.1.92` |
| Instance ID | `i-0614f8c6393dd148c` |
| Security Group ID | `sg-024470395c3cde28e` |
| Subnet ID | `subnet-0383d6d5b1cfd33c3` |
| Key Pair | `vockey` |

![Describe instances completo](./img/08.png)
![Describe instances con query](./img/09.png)

La instancia está **running** y el par de llaves es el correcto. Sin embargo, el intento de SSH desde la máquina local falla con timeout:

![SSH timeout desde máquina local](./img/10.png)

Al intentar cargar el sitio web desde el navegador, también se obtiene un error de conexión:

![ERR_CONNECTION_TIMED_OUT en el navegador](./img/Error_404.png)

---

### Desafío #1 — Sitio web inaccesible (HTTP)

#### Paso 1: Diagnóstico con nmap

```bash
sudo yum install -y nmap
nmap 13.223.77.43
```

![Instalación de nmap](./img/11.png)

**Resultado:** `Host seems down` — nmap no detecta ningún puerto abierto. Esto descarta que el problema esté en la instancia misma e indica un bloqueo a nivel de red.

![Resultado nmap](./img/12.png)

#### Paso 2: Revisión del Security Group

```bash
aws ec2 describe-security-groups --group-ids sg-024470395c3cde28e
```

**Análisis del resultado:**

- **Egress:** Todo el tráfico permitido hacia `0.0.0.0/0` ✅
- **Ingress:** Puerto `80` (HTTP) permitido desde `0.0.0.0/0` ✅ | Puerto `22` (SSH) permitido desde `0.0.0.0/0` ✅

![Security Group - Egress](./img/13_1.png)
![Security Group - Descripción y tags](./img/13_2.png)
![Security Group - Ingress puertos 80 y 22](./img/13_3.png)

**Conclusión:** El Security Group **no es el problema**. Permite tráfico HTTP y SSH desde cualquier origen.

#### Paso 3: Revisión de la Tabla de Rutas

```bash
aws ec2 describe-route-tables \
  --filter "Name=association.subnet-id,Values='subnet-0383d6d5b1cfd33c3'"
```

![Route Table - Metadatos](./img/14_1.png)
![Route Table - Rutas](./img/14_2.png)

**Diagnóstico — Causa Raíz #1:**

La tabla de rutas `rtb-07e67ef6df59ca951` (VPC1 Public Route Table) solo contiene **una ruta local**:

```
Destino: 10.0.0.0/16  →  GatewayId: local
```

**Falta la ruta por defecto** `0.0.0.0/0` hacia el Internet Gateway. Sin esta ruta, ningún tráfico de internet puede entrar ni salir de la subred pública.

#### Solución #1: Crear la ruta hacia el IGW

```bash
aws ec2 create-route \
  --route-table-id rtb-07e67ef6df59ca951 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id <VPC1-IGW-ID>
```

> El `gateway-id` del IGW de VPC1 se obtiene del panel de credenciales del lab o ejecutando `aws ec2 describe-internet-gateways`.

El comando devuelve `"Return": true`, confirmando que la ruta fue creada exitosamente.

![create-route exitoso](./img/15.png)

**Resultado post-fix:** El sitio web carga correctamente en `http://13.223.77.43` — el servidor responde con `Hello From Your Web Server!` ✅

![Sitio web cargando en el navegador](./img/16.png)

---

### Desafío #2 — SSH bloqueado

Con la ruta corregida, el HTTP funciona pero SSH (`puerto 22`) sigue fallando. El Security Group permite el puerto 22, la ruta existe. El siguiente nivel a investigar es la **Network ACL (NACL)**.

> **Diferencia clave:** Los Security Groups son *stateful* (si permites la entrada, la respuesta sale automáticamente). Las NACLs son *stateless* — hay que permitir entrada **y** salida explícitamente, y las reglas se evalúan en orden ascendente; la primera coincidencia gana.

#### Revisión de la NACL

```bash
aws ec2 describe-network-acls \
  --filter "Name=association.subnet-id,Values='subnet-0383d6d5b1cfd33c3'"
```

La NACL asociada a la subred pública de VPC1 es `acl-0dc2c7a507c8aee9d` ("VPC1 Public Network ACL").

![describe-network-acls metadatos](./img/17.png)

**Diagnóstico — Causa Raíz #2:**

Al inspeccionar las `Entries` de la NACL se identificó la regla bloqueante:

| RuleNumber | Protocol | Port | Egress | Action |
|---|---|---|---|---|
| 40 | TCP (6) | 22 | `false` (ingress) | **deny** |
| 100 | All (-1) | — | `false` (ingress) | allow |
| 100 | All (-1) | — | `true` (egress) | allow |

La regla **40 (ingress DENY)** se evalúa antes que la regla 100 (ingress ALLOW), por lo que todo intento de conexión SSH entrante es rechazado antes de que el ALLOW tenga efecto.

![NACL entries con regla DENY puerto 22](./img/18.png)

#### Solución #2: Eliminar la regla DENY

Primero se obtuvo el NACL ID de forma directa:

```bash
aws ec2 describe-network-acls \
  --filter "Name=association.subnet-id,Values='subnet-0383d6d5b1cfd33c3'" \
  --query 'NetworkAcls[*].NetworkAclId' --output text
# → acl-0dc2c7a507c8aee9d

aws ec2 delete-network-acl-entry \
  --network-acl-id acl-0dc2c7a507c8aee9d \
  --ingress \
  --rule-number 40
```

![delete-network-acl-entry ejecutado](./img/19.png)

**Verificación post-fix:** Al volver a describir las entradas de la NACL, la regla 40 ya no existe. Solo quedan las reglas 100 (allow) y 32767 (deny implícito por defecto) en ambas direcciones — comportamiento correcto.

![NACL entries post-fix sin regla 40](./img/20.png)

**Resultado:** Conexión SSH al Web Server establecida exitosamente. El comando `hostname` confirma conexión al host `web-server` ✅

---

## Task 4: Análisis de Flow Logs

### Descarga y extracción

```bash
mkdir flowlogs && cd flowlogs
aws s3 ls                                        # confirmar nombre del bucket
aws s3 cp s3://cubetagv0001/ . --recursive
cd AWSLogs/818893665666/vpcflowlogs/us-east-1/2026/04/18/
gunzip *.gz
```

Los archivos se descargaron desde `s3://cubetagv0001` y se extrajeron exitosamente. Los logs se ubican bajo la ruta `AWSLogs/<account-id>/vpcflowlogs/us-east-1/2026/04/18/`.

![Descarga de flow logs desde S3](./img/21.png)

### Estructura de un registro

```bash
head <nombre-del-archivo>.log
```

El encabezado revela las columnas:

```
version  account-id  interface-id  srcaddr  dstaddr  srcport  dstport  protocol  packets  bytes  start  end  action  log-status
```

Los registros inmediatamente muestran entradas `REJECT` de IPs externas hacia la instancia, evidenciando el tráfico bloqueado durante la actividad.

![head del log — estructura y primeras entradas](./img/22.png)

### Análisis de intentos SSH rechazados

**Todos los REJECTs registrados:**
```bash
grep -rn REJECT .
```

![grep REJECT — salida de registros rechazados](./img/23.png)

```bash
grep -rn REJECT . | wc -l
# → 179 líneas
```

**REJECTs en el puerto 22:**
```bash
grep -rn ' 22 ' . | grep REJECT
```

El resultado muestra exactamente dos entradas REJECT en el puerto 22: una desde `74.82.47.23` (tráfico externo genérico) y otra desde `65.49.1.233` — correspondiente a los intentos de SSH realizados durante la actividad.

![wc -l total REJECTs y grep puerto 22](./img/24.png)

**Obtención de la IP pública del analista:**

Se usó la consola de AWS (Security Groups → Edit inbound rules → My IP) para obtener la IP pública de la máquina local: `181.52.218.29/32`.

![Consola AWS — obtención de IP pública](./img/25.png)

**Filtrado por IP del analista:**
```bash
grep -rn ' 22 ' . | grep REJECT | grep 181.52.218.29
```

El resultado confirma que no hay entradas REJECT del puerto 22 con esa IP — los intentos de SSH fallidos del analista fueron registrados bajo la IP `65.49.1.233` (IP del CLI Host en VPC2, desde donde se ejecutaron los comandos SSH).

**Verificación de la interfaz de red:**
```bash
aws ec2 describe-network-interfaces \
  --filters "Name=association.public-ip,Values='34.224.22.198'" \
  --query 'NetworkInterfaces[*].[NetworkInterfaceId,Association.PublicIp]'
# → eni-02d54364b1ade7780  /  34.224.22.198
```

El `NetworkInterfaceId` (`eni-02d54364b1ade7780`) coincide con el registrado en los flow logs, confirmando que las entradas REJECT corresponden al tráfico real hacia el Web Server.

![grep por IP + describe-network-interfaces](./img/26.png)

**Conversión de timestamps Unix a formato legible:**
```bash
date -d @1776483941
# → sáb abr 18 03:45:41 UTC 2026

date
# → sáb abr 18 04:01:54 UTC 2026
```

El timestamp convertido corresponde al horario en que se ejecutaron los intentos de conexión SSH durante la actividad, validando la integridad cronológica de los logs.

![Conversión de timestamp con date](./img/27.png)

---

## Conclusiones

| # | Capa OSI | Componente | Problema | Solución |
|---|---|---|---|---|
| 1 | Capa 3 — Red | **Route Table** | Faltaba ruta `0.0.0.0/0` al IGW en la subred pública | `aws ec2 create-route` apuntando al IGW de VPC1 |
| 2 | Capa 3 — Red | **Network ACL** | Regla DENY con número bajo bloqueaba el puerto 22 (SSH) | `aws ec2 delete-network-acl-entry` eliminando la regla bloqueante |

**Lecciones aprendidas:**

- El orden de investigación correcto sigue la pila de red de afuera hacia adentro: **IGW → Route Table → NACL → Security Group → instancia**.
- Las NACLs son *stateless*: un DENY en ingress del puerto 22 es suficiente para romper SSH aunque el Security Group lo permita.
- Los VPC Flow Logs son la herramienta de forensia de red definitiva en AWS: permiten correlacionar intentos de conexión con acciones ACCEPT/REJECT y timestamps verificables.
- `grep` sobre los logs en texto plano es efectivo para análisis rápido; para análisis a escala, Amazon Athena permite ejecutar SQL directamente sobre los logs en S3.

---

*Actividad realizada en entorno AWS Academy | Región: us-east-1 | Fecha: Abril 2026*

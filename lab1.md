# Laboratorios AWS + Azure (9:00–12:00)

**Perfil:** clase en vivo (3 horas) para cubrir:
- **Arquitectura Cloud y Ambientes**
- **Servicios de Aplicaciones Web**
- **Servicios de Bases de Datos**

**Estrategia didáctica (para no “correr”):**
- Se hace **un ambiente “dev” completo** en cada nube (mínimo viable).
- Se agregan **buffers obligatorios** (tiempo de espera/errores/explicación).
- Cada bloque incluye **ejercicios extra** (por si un aprovisionamiento va rápido).
- En Bases de Datos: **Azure PostgreSQL (relacional)** + **AWS DynamoDB (NoSQL)** para asegurar progreso sin esperas largas.

> Nota de costos: usa SKUs/tamaños pequeños. Al final hay sección de **limpieza**.

---

## 0) Requisitos previos (antes de las 9:00)

- Tener listo:
  - **Azure CLI**: `az`
  - **AWS CLI**: `aws`
  - **Python 3** (para instalar EB CLI si se usa Beanstalk)
  - (Opcional) **psql** para conectarte a PostgreSQL desde terminal

---

## 1) Preparación y verificación (9:00–9:15)

### 1.1 Azure login + variables (5–7 min)

```bash
az login
az account show
az account set --subscription "<TU_SUBSCRIPTION_ID_O_NOMBRE>"

export AZ_REGION="eastus"
export AZ_RG="rg-lab-webdb"
export AZ_APP="app$(date +%s)"
export AZ_PLAN="plan-lab"
export AZ_DB_SERVER="pg$(date +%s)"
export AZ_DB_NAME="appdb"
export AZ_DB_ADMIN="dbadminuser"
export AZ_DB_PASS="P@ssw0rd-$(date +%s)"   # cambia si tu política lo requiere
```

**Checkpoint (Azure):**
- `az group exists -n "$AZ_RG"` debe devolver `false` (aún no lo creamos).

### 1.2 AWS identidad + variables (5–7 min)

```bash
aws sts get-caller-identity
aws configure get region

export AWS_REGION="us-east-1"
export AWS_PREFIX="labwebdb-$(date +%s)"
export AWS_VPC_CIDR="10.20.0.0/16"
```

**Checkpoint (AWS):**
- `aws sts get-caller-identity` muestra cuenta/ARN (si falla, credenciales).

### 1.3 Buffer (1–3 min)

Usa este mini-buffer para resolver logins/permisos, o para explicar:
- Suscripción (Azure) / Cuenta e IAM (AWS)
- Región y latencia

---

## 2) Lab 1 — Arquitectura Cloud y Ambientes (9:15–10:10)

**Objetivo:** construir la “base” de arquitectura (red + segmentación + seguridad) y hablar de **ambientes**.

### 2.1 Azure: Resource Group + VNet + Subnet (15–20 min)

1) Crear Resource Group:

```bash
az group create -n "$AZ_RG" -l "$AZ_REGION"
```

2) Crear VNet y Subnet para app:

```bash
az network vnet create -g "$AZ_RG" -n vnet-lab \
  --address-prefix 10.10.0.0/16 \
  --subnet-name snet-app --subnet-prefix 10.10.1.0/24
```

3) Mostrar estado:

```bash
az network vnet show -g "$AZ_RG" -n vnet-lab --query "{name:name,location:location,addressSpace:addressSpace.addressPrefixes}" -o yaml
```

**Puntos para explicar (Azure):**
- `Resource Group` como unidad de **ambiente** (dev/prod) y ciclo de vida.
- VNet/Subnet = aislamiento y segmentación.

### 2.2 AWS: VPC + 2 subnets + Security Group web (20–25 min)

1) Crear VPC:

```bash
VPC_ID=$(aws ec2 create-vpc --cidr-block "$AWS_VPC_CIDR" --query 'Vpc.VpcId' --output text)
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-hostnames "{\"Value\":true}"
aws ec2 create-tags --resources "$VPC_ID" --tags Key=Name,Value="$AWS_PREFIX-vpc"
echo "$VPC_ID"
```

2) Crear 2 subnets (para arquitectura y discusión “multi-AZ”):

```bash
SUBNET_A=$(aws ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.20.1.0/24 --availability-zone "${AWS_REGION}a" --query 'Subnet.SubnetId' --output text)
SUBNET_B=$(aws ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.20.2.0/24 --availability-zone "${AWS_REGION}b" --query 'Subnet.SubnetId' --output text)
echo "$SUBNET_A"
echo "$SUBNET_B"
```

3) Security Group para HTTP:

```bash
SG_WEB=$(aws ec2 create-security-group --group-name "$AWS_PREFIX-web-sg" --description "web sg" --vpc-id "$VPC_ID" --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id "$SG_WEB" --protocol tcp --port 80 --cidr 0.0.0.0/0
echo "$SG_WEB"
```

**Puntos para explicar (AWS):**
- VPC/Subnets y el concepto de **zonas de disponibilidad**.
- SG como firewall “stateful” a nivel de ENI/instancia.
- Ambientes: tags + cuentas/OU (conceptual), o prefijos como `$AWS_PREFIX`.

### 2.3 Buffer + ejercicios extra (10–15 min)

**Ejercicio extra A (Azure):** crear una segunda subnet “db” (solo para visualizar segmentación)

```bash
az network vnet subnet create -g "$AZ_RG" --vnet-name vnet-lab -n snet-db --address-prefixes 10.10.2.0/24
```

**Ejercicio extra B (AWS):** listar recursos creados y “leer” la arquitectura

```bash
aws ec2 describe-vpcs --vpc-ids "$VPC_ID" --query "Vpcs[0].{VpcId:VpcId,CidrBlock:CidrBlock,Tags:Tags}" -o json
aws ec2 describe-subnets --subnet-ids "$SUBNET_A" "$SUBNET_B" --query "Subnets[].{SubnetId:SubnetId,CidrBlock:CidrBlock,AZ:AvailabilityZone}" -o table
```

---

## 3) Lab 2 — Servicios de Aplicaciones Web (10:10–11:20)

**Objetivo:** desplegar un “Hello World” accesible por URL, comparar el servicio web administrado en cada nube.

### 3.1 Azure: App Service (Node) + deploy zip (25–35 min)

1) Plan + Web App:

```bash
az appservice plan create -g "$AZ_RG" -n "$AZ_PLAN" --is-linux --sku B1
az webapp create -g "$AZ_RG" -p "$AZ_PLAN" -n "$AZ_APP" --runtime "NODE:18-lts"
```

2) Crear app mínima y desplegar:

```bash
mkdir -p /tmp/azapp
cat > /tmp/azapp/index.js <<'EOF'
const http = require('http');
const port = process.env.PORT || 8080;
http.createServer((req,res)=>{ res.end("Hola desde Azure App Service\n"); }).listen(port);
EOF

cat > /tmp/azapp/package.json <<'EOF'
{ "name":"azapp","version":"1.0.0","main":"index.js","scripts":{"start":"node index.js"} }
EOF

(cd /tmp/azapp && zip -r app.zip .)
az webapp deployment source config-zip -g "$AZ_RG" -n "$AZ_APP" --src /tmp/azapp/app.zip
```

3) Probar:

```bash
echo "URL Azure:"
echo "https://$AZ_APP.azurewebsites.net"
```

**Ejercicio extra (Azure, 5 min):** ver logs de arranque

```bash
az webapp log config -g "$AZ_RG" -n "$AZ_APP" --application-logging filesystem --level information
az webapp log tail -g "$AZ_RG" -n "$AZ_APP"
```

### 3.2 AWS: Elastic Beanstalk (Node) (25–35 min)

> Si ya lo tienes instalado, salta la instalación. Si no, instálalo.

1) Instalar EB CLI:

```bash
python3 -m pip install --user awsebcli
export PATH="$HOME/.local/bin:$PATH"
eb --version
```

2) Crear app y desplegar:

```bash
mkdir -p /tmp/awsapp && cd /tmp/awsapp

cat > server.js <<'EOF'
const http = require('http');
const port = process.env.PORT || 8081;
http.createServer((req,res)=>{ res.end("Hola desde AWS Elastic Beanstalk\n"); }).listen(port);
EOF

cat > package.json <<'EOF'
{ "name":"awsapp","version":"1.0.0","main":"server.js","scripts":{"start":"node server.js"} }
EOF

eb init "$AWS_PREFIX-ebapp" --platform "Node.js" --region "$AWS_REGION"
eb create "$AWS_PREFIX-env" --single --timeout 20
eb open
```

**Ejercicio extra (AWS, 5–10 min):** revisar eventos si demora

```bash
eb events --follow
```

### 3.3 Buffer + discusión guiada (10–15 min)

Usa este tiempo para comparar:
- PaaS web: App Service vs Beanstalk (qué te administra cada uno).
- Variables de entorno, escalado, logs.
- “Ambientes” en web: `dev`/`prod` (2 apps) y “slots” (conceptual).

---

## 4) Lab 3 — Servicios de Bases de Datos (11:20–11:50)

**Objetivo:** crear una DB administrada y ejecutar al menos una operación real (insert/select o put/get).

### 4.1 Azure: PostgreSQL Flexible Server + DB (15–20 min)

1) Crear servidor (público para demo rápida):

```bash
az postgres flexible-server create -g "$AZ_RG" -n "$AZ_DB_SERVER" -l "$AZ_REGION" \
  --admin-user "$AZ_DB_ADMIN" --admin-password "$AZ_DB_PASS" \
  --sku-name Standard_B1ms --storage-size 32 --version 14 \
  --public-access 0.0.0.0
```

2) Crear base:

```bash
az postgres flexible-server db create -g "$AZ_RG" -s "$AZ_DB_SERVER" -d "$AZ_DB_NAME"
```

3) (Opcional) Conectar con `psql`:

```bash
export PGPASSWORD="$AZ_DB_PASS"
psql "host=$AZ_DB_SERVER.postgres.database.azure.com port=5432 dbname=$AZ_DB_NAME user=$AZ_DB_ADMIN sslmode=require" \
  -c "create table if not exists demo(ts timestamptz default now(), msg text); insert into demo(msg) values('hola azure db'); select * from demo;"
```

**Si NO tienes `psql`:**
- Muestra el endpoint y explica cadena de conexión:

```bash
echo "$AZ_DB_SERVER.postgres.database.azure.com"
```

### 4.2 AWS: DynamoDB (instantáneo) + put/get (10–15 min)

1) Crear tabla:

```bash
aws dynamodb create-table \
  --table-name "$AWS_PREFIX-messages" \
  --attribute-definitions AttributeName=pk,AttributeType=S \
  --key-schema AttributeName=pk,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

2) Insertar y leer:

```bash
aws dynamodb put-item --table-name "$AWS_PREFIX-messages" --item '{"pk":{"S":"1"},"msg":{"S":"hola desde dynamodb"}}'
aws dynamodb get-item --table-name "$AWS_PREFIX-messages" --key '{"pk":{"S":"1"}}'
```

**Puntos para explicar:**
- Relacional (PostgreSQL): esquema, joins, transacciones.
- NoSQL (DynamoDB): clave-partición, acceso por patrones, latencia.
- “Administrado”: backups, parches, alta disponibilidad (conceptual).

### 4.3 Ejercicios extra (si sobra 5–10 min)

**Extra A (Azure):** ver configuración del servidor

```bash
az postgres flexible-server show -g "$AZ_RG" -n "$AZ_DB_SERVER" -o yaml
```

**Extra B (AWS):** escanear tabla (pequeña)

```bash
aws dynamodb scan --table-name "$AWS_PREFIX-messages"
```

---

## 5) Cierre + limpieza (11:50–12:00)

### 5.1 Limpieza Azure (2–4 min)

```bash
az group delete -n "$AZ_RG" --yes --no-wait
```

### 5.2 Limpieza AWS (4–8 min)

**Elastic Beanstalk:**

```bash
cd /tmp/awsapp
eb terminate "$AWS_PREFIX-env" --force
```

**DynamoDB:**

```bash
aws dynamodb delete-table --table-name "$AWS_PREFIX-messages"
```

**VPC/Subnets/SG** (si no quedó atado a otros recursos):

```bash
aws ec2 delete-subnet --subnet-id "$SUBNET_A"
aws ec2 delete-subnet --subnet-id "$SUBNET_B"
aws ec2 delete-security-group --group-id "$SG_WEB"
aws ec2 delete-vpc --vpc-id "$VPC_ID"
```

---

## Apéndice — Plan B si Beanstalk demora o falla (para no quedarte sin ejercicio)

Si EB CLI se complica (permisos, cuotas, instalación), reemplaza AWS Lab 2 por **S3 Static Website** (rápido y demostrable):

1) Crear bucket + subir `index.html`:

```bash
export AWS_BUCKET="$AWS_PREFIX-site"
aws s3 mb "s3://$AWS_BUCKET" --region "$AWS_REGION"
cat > /tmp/index.html <<'EOF'
<html><body><h1>Hola desde AWS S3 Static Website</h1></body></html>
EOF
aws s3 cp /tmp/index.html "s3://$AWS_BUCKET/index.html"
```

2) Habilitar website (requiere política/ACL según tu configuración de cuenta):
- Si tu cuenta bloquea acceso público por default, úsalo como demo de **seguridad por defecto** y deja la publicación pública como “tarea”.

Limpieza:

```bash
aws s3 rm "s3://$AWS_BUCKET/" --recursive
aws s3 rb "s3://$AWS_BUCKET"
```


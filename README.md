# 💧 Chatbot Asistente Para Lavandería Ecológica

Widget de chat flotante integrado en una landing page estática, con backend serverless en AWS Lambda, Amazon Bedrock Agent y Knowledge Base RAG.

---

## Demo

La landing page muestra la sucursal, servicios y beneficios ecológicos de la lavandería. El widget flotante en la esquina inferior derecha conecta al usuario con el asistente de IA.

---

## Arquitectura

```
Usuarios
   │
   ▼
CloudFront CDN
   ├── /          → S3 Bucket  (index.html + chatbot-wiget.js)
   └── /chat      → API Gateway HTTP → Lambda (Flask + Mangum)
                                           │
                                           ▼
                                  Amazon Bedrock Agent
                                  (Nova-Lite + Knowledge Base RAG)
```

| Capa | Tecnología |
|---|---|
| Frontend | HTML5 / CSS3 / JavaScript vanilla |
| Widget | JS autocontenido (`chatbot-wiget.js`) |
| Backend | Python 3.11 · Flask · Mangum |
| IA | Amazon Bedrock Agent · Nova-Lite v1 |
| Infra | AWS Lambda · API Gateway HTTP · S3 · CloudFront |

---

## Estructura del proyecto

```
.
├── index.html              # Landing page con widget integrado
├── chatbot-wiget.js        # Widget flotante del chatbot (autocontenido)
├── img/                    # Imágenes de la sucursal
└── tintorerias-chatbot/
    ├── app.py              # Flask app — endpoint POST /chat
    ├── lambda_handler.py   # Handler WSGI nativo para AWS Lambda
    ├── bedrock_client.py   # Cliente multi-contexto para Bedrock Agent / KB
    ├── config.py           # Variables de entorno
    ├── .env.example        # Plantilla de variables locales
    └── package/            # Dependencias empaquetadas para Lambda
```

---

## Requisitos previos

- Python 3.11+
- AWS CLI configurado (`aws configure`)
- Bedrock Agent creado con alias activo **o** Knowledge Base con PDFs indexados
- Modelo `amazon.nova-lite-v1:0` habilitado en tu región

---

## Desarrollo local

```bash
cd tintorerias-chatbot

# Crear entorno virtual
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

# Instalar dependencias
pip install flask flask-cors boto3 python-dotenv

# Configurar variables
cp .env.example .env
# Edita .env con tus valores reales

# Iniciar servidor
python app.py
```

El servidor queda disponible en `http://localhost:3000`.

Para probar el widget localmente, abre `index.html` en un servidor estático (p. ej. `python -m http.server 8080`) y asegúrate de que `apiUrl` en `chatbot-wiget.js` apunte a `http://localhost:3000/chat`.

---

## Variables de entorno

| Variable | Descripción | Ejemplo |
|---|---|---|
| `KB_ID` | ID de la Knowledge Base en Bedrock | `WJ76LJMDXT` |
| `MODEL_ARN` | ARN o ID del modelo | `amazon.nova-lite-v1:0` |
| `AWS_REGION` | Región AWS | `us-east-1` |
| `PORT` | Puerto del servidor Flask | `3000` |
| `MAX_HISTORY_TURNS` | Turnos de historial enviados a Bedrock | `10` |

> **Nunca subas `.env` a git.** Está incluido en `.gitignore`.

---

## Despliegue en AWS

### 1 — Frontend en S3 + CloudFront

```bash
BUCKET_NAME="tintorerias-max-chatbot-frontend"
REGION="us-east-1"

# Crear bucket con acceso público bloqueado
aws s3api create-bucket --bucket $BUCKET_NAME --region $REGION
aws s3api put-public-access-block \
  --bucket $BUCKET_NAME \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Subir archivos
aws s3 cp index.html        s3://$BUCKET_NAME/
aws s3 cp chatbot-wiget.js  s3://$BUCKET_NAME/
aws s3 cp img/              s3://$BUCKET_NAME/img/ --recursive
```

Crea una distribución CloudFront con el bucket como origen (OAC) y `index.html` como objeto raíz.

---

### 2 — Backend en AWS Lambda

```bash
cd tintorerias-chatbot

# Empaquetar dependencias + código
pip install -r requirements.txt -t package/
cp app.py bedrock_client.py config.py lambda_handler.py package/
cd package && zip -r ../lambda_deployment.zip . && cd ..
```

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Crear rol IAM
aws iam create-role \
  --role-name tintorerias-max-lambda-role \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }'

aws iam attach-role-policy \
  --role-name tintorerias-max-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam put-role-policy \
  --role-name tintorerias-max-lambda-role \
  --policy-name BedrockAccess \
  --policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Action":["bedrock:InvokeAgent","bedrock:RetrieveAndGenerate","bedrock:Retrieve","bedrock:InvokeModel"],"Resource":"*"}]
  }'

# Crear función Lambda
aws lambda create-function \
  --function-name tintorerias-max-chatbot \
  --runtime python3.11 \
  --role arn:aws:iam::${ACCOUNT_ID}:role/tintorerias-max-lambda-role \
  --handler lambda_handler.handler \
  --zip-file fileb://lambda_deployment.zip \
  --timeout 30 \
  --memory-size 256 \
  --environment Variables="{KB_ID=TU_KB_ID,MODEL_ARN=amazon.nova-lite-v1:0,AWS_REGION=us-east-1,PORT=3000,MAX_HISTORY_TURNS=10}"
```

---

### 3 — API Gateway HTTP

```bash
API_ID=$(aws apigatewayv2 create-api \
  --name "tintorerias-max-chat-api" \
  --protocol-type HTTP \
  --query 'ApiId' --output text)

LAMBDA_ARN="arn:aws:lambda:us-east-1:${ACCOUNT_ID}:function:tintorerias-max-chatbot"

INTEGRATION_ID=$(aws apigatewayv2 create-integration \
  --api-id $API_ID \
  --integration-type AWS_PROXY \
  --integration-uri $LAMBDA_ARN \
  --payload-format-version "2.0" \
  --query 'IntegrationId' --output text)

aws apigatewayv2 create-route \
  --api-id $API_ID \
  --route-key "POST /chat" \
  --target "integrations/$INTEGRATION_ID"

aws apigatewayv2 create-stage \
  --api-id $API_ID \
  --stage-name prod \
  --auto-deploy

aws lambda add-permission \
  --function-name tintorerias-max-chatbot \
  --statement-id apigateway-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:${ACCOUNT_ID}:${API_ID}/*/*/chat"

echo "Endpoint: $(aws apigatewayv2 get-api --api-id $API_ID --query 'ApiEndpoint' --output text)/prod/chat"
```

---

### 4 — Conectar CloudFront con el backend

En la consola de CloudFront, agrega un segundo origen apuntando al API Gateway y crea un behavior para el path `/chat`:

- Allowed methods: `GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE`
- Cache policy: `CachingDisabled`
- Origin request policy: `AllViewer`

Luego actualiza `chatbot-wiget.js`:

```javascript
const CONFIG = {
  apiUrl: 'https://TU_DOMINIO.cloudfront.net/chat',
  requestTimeout: 30000,
};
```

Sube el archivo actualizado e invalida la caché:

```bash
aws s3 cp chatbot-wiget.js s3://$BUCKET_NAME/
aws cloudfront create-invalidation --distribution-id TU_DISTRIBUTION_ID --paths "/*"
```

---

### Verificar el despliegue

```bash
curl -X POST https://TU_DOMINIO.cloudfront.net/chat \
  -H "Content-Type: application/json" \
  -d '{"sessionId":"test-1","message":"¿Cuáles son sus horarios?"}'
```

---

## Actualizaciones

### Frontend

```bash
aws s3 sync . s3://$BUCKET_NAME/ --exclude "*" --include "*.html" --include "*.js"
aws cloudfront create-invalidation --distribution-id $DISTRIBUTION_ID --paths "/*"
```

### Backend (Lambda)

```bash
cd tintorerias-chatbot
pip install -r requirements.txt -t package/
cp app.py bedrock_client.py config.py lambda_handler.py package/
cd package && zip -r ../lambda_deployment.zip . && cd ..
aws lambda update-function-code \
  --function-name tintorerias-max-chatbot \
  --zip-file fileb://lambda_deployment.zip
```

---

## Licencia

Proyecto privado — todos los derechos reservados.

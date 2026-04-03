# Pepe Pagos - Guía de Implementación y Despliegue

## 🚀 Visión General del Producto
Pepe Pagos es una plataforma B2B de infraestructura de pagos y movimiento de dinero para fintechs, bancos, comercios y marketplaces. Unifica en una sola API cobros, payouts, transferencias locales e internacionales, conciliación, smart routing, antifraude, compliance y wallets. Con un enfoque developer-first, permite orquestar múltiples rieles de pago tradicionales y digitales garantizando escalabilidad, trazabilidad y automatización operativa.

## 🏗️ Diagrama de Arquitectura (ASCII)

```text
+-------------------+       +-----------------------+       +-------------------+
|                   |       |                       |       |                   |
|  Clientes B2B /   | REST  |   Cloud Run (GCP)     |  SQL  | Cloud SQL (GCP)   |
|  Sistemas Externos| ----> |   [Pepe Pagos API &   | ----> | [BD: pepe_pagos]  |
|                   |       |    MCP Server]        |       |                   |
+-------------------+       +-----------------------+       +-------------------+
                                      |   ^
                             Webhooks |   | API Calls
                                      v   |
                            +-----------------------+
                            |                       |
                            |      Stripe API       |
                            |  (Billing & Payments) |
                            |                       |
                            +-----------------------+
```

## ⚙️ Variables de Entorno
Para ejecutar este proyecto, necesitas configurar las siguientes variables de entorno:

* `DB_USER`: `svc_pepe_pagos`
* `DB_PASS`: (Obtenido desde Secret Manager `pepe_pagos-db-password`)
* `DB_NAME`: `pepe_pagos`
* `DB_HOST`: Conexión a través de Cloud SQL Auth Proxy (`test-agents-ai-app:us-central1:quien-prod`)
* `STRIPE_SECRET_KEY`: Tu clave secreta de Stripe.
* `STRIPE_WEBHOOK_SECRET`: El secreto del webhook proporcionado por Stripe.
* `ENVIRONMENT`: `production` o `development`

## 🛠️ Instrucciones de Despliegue
*Nota: El despliegue automático mediante Cloud Build falló por falta de permisos (API 403). Se requiere realizar el despliegue manual.*

1. Autentícate en Google Cloud CLI:
   ```bash
   gcloud auth login
   ```
2. Configura el proyecto:
   ```bash
   gcloud config set project [ID_DEL_PROYECTO]
   ```
3. Construye la imagen (requiere habilitar la API y asignar permisos):
   ```bash
   gcloud builds submit --tag gcr.io/[ID_DEL_PROYECTO]/pepe-pagos-studio
   ```
4. Despliega en Cloud Run:
   ```bash
   gcloud run deploy pepe-pagos-studio --image gcr.io/[ID_DEL_PROYECTO]/pepe-pagos-studio --region us-central1 --add-cloudsql-instances test-agents-ai-app:us-central1:quien-prod
   ```

## 🌐 Configuración DNS
Para enrutar el tráfico al dominio `mvpfactory.studio/p/pepe-pagos`:
1. Ve al panel de control de tu proveedor de dominio (DNS).
2. Crea un registro para tu subdominio o asocia un Custom Domain en Cloud Run.
3. Apunta el registro CNAME o A hacia la IP proporcionada por el balanceador de carga de Google Cloud o Firebase Hosting, según tu configuración de enrutamiento.

## 💳 Configuración de Webhooks de Stripe
*Nota: La configuración automática de Stripe no se pudo completar. Se ha dejado un archivo `stripe-config.json` como plantilla en el repositorio.*

1. Ve a tu [Stripe Dashboard](https://dashboard.stripe.com/).
2. Dirígete a **Developers > Webhooks** y añade un nuevo endpoint.
3. Introduce la URL: `https://mvpfactory.studio/p/pepe-pagos/api/stripe/webhook` (o la ruta correspondiente).
4. Selecciona los eventos necesarios (ej. `payment_intent.succeeded`, `transfer.created`).
5. Copia el **Signing secret** y guárdalo como la variable de entorno `STRIPE_WEBHOOK_SECRET`.
6. Actualiza manualmente el archivo `stripe-config.json` en este repositorio con los IDs de tu Producto y Precio de Stripe.

## 📊 Monitorización y Resolución de Problemas
* **Logs del Sistema:** Accede a **Google Cloud Logging** y filtra por `Cloud Run Revision` para ver en tiempo real peticiones y errores de la API.
* **Métricas de Base de Datos:** Ve a la instancia `quien-prod` en Cloud SQL para revisar CPU, conexiones activas y almacenamiento.
* **Alertas de Pagos:** Utiliza el panel de Webhooks de Stripe para confirmar que los eventos envían códigos HTTP 200. En caso de fallos, revisa los logs de aplicación buscando el endpoint del webhook.
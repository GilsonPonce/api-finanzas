# Arquitectura de Datos y Especificación de API REST para Finanzas Personales

Este documento define el modelo de datos relacional, las especificaciones de la API REST JSON, las consideraciones técnicas de diseño y la hoja de ruta de futuras mejoras para un sistema de finanzas personales basado en el principio de contabilidad por partida doble (Debe y Haber).

---

## 1. Arquitectura del Modelo de Datos (ERD / Relacional)

### Diagrama de Entidad-Relación (Conceptual)

```text
 [Users] 1 ─── N [Accounts] 1 ─── N [Transactions] N ─── 1 [Categories]
                    │                     │
                    └─── N [Transfers] ───┘
```

---

### Tablas y Campos

#### A. `users` (Usuarios)
Almacena los perfiles del sistema y sus configuraciones globales.

| Campo | Tipo de Dato | Restricciones | Descripción |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | Primary Key, Default: `gen_random_uuid()` | Identificador único del usuario. |
| `email` | `VARCHAR(255)` | Unique, Not Null | Correo electrónico principal. |
| `password_hash` | `VARCHAR(255)` | Not Null | Hash de la contraseña. |
| `currency` | `VARCHAR(3)` | Default: `'USD'` | Moneda base predeterminada (ISO 4217). |
| `created_at` | `TIMESTAMPTZ` | Default: `NOW()` | Fecha de creación del registro. |
| `updated_at` | `TIMESTAMPTZ` | Default: `NOW()` | Fecha de última actualización. |

#### B. `accounts` (Cuentas)
Representa los lugares físicos o virtuales donde reside el dinero (Activos y Pasivos).

| Campo | Tipo de Dato | Restricciones | Descripción |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | Primary Key, Default: `gen_random_uuid()` | Identificador único de la cuenta. |
| `user_id` | `UUID` | FK -> `users(id)`, On Delete Cascade | Propietario de la cuenta. |
| `name` | `VARCHAR(100)` | Not Null | Nombre asignado (ej. "Banco Principal", "Efectivo"). |
| `type` | `VARCHAR(20)` | Enum: `checking`, `savings`, `credit_card`, `cash`, `investment` | Naturaleza de la cuenta. |
| `initial_balance` | `DECIMAL(12,2)` | Default: `0.00` | Saldo inicial registrado. |
| `current_balance` | `DECIMAL(12,2)` | Default: `0.00` | Saldo calculated en tiempo real. |
| `is_active` | `BOOLEAN` | Default: `TRUE` | Estado de la cuenta. |
| `created_at` | `TIMESTAMPTZ` | Default: `NOW()` | Fecha de creación. |

#### C. `categories` (Categorías)
Define la clasificación de los ingresos y egresos.

| Campo | Tipo de Dato | Restricciones | Descripción |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | Primary Key, Default: `gen_random_uuid()` | Identificador único de la categoría. |
| `user_id` | `UUID` | FK -> `users(id)`, Nullable | Propietario (NULL si es categoría global del sistema). |
| `name` | `VARCHAR(100)` | Not Null | Nombre de la categoría (ej. "Supermercado", "Sueldo"). |
| `type` | `VARCHAR(20)` | Enum: `income`, `expense` | Define si afecta al Haber (ingreso) o Debe (gasto). |
| `icon` | `VARCHAR(50)` | Nullable | Identificador del icono para frontend. |
| `parent_id` | `UUID` | FK -> `categories(id)`, Nullable | Para jerarquías de subcategorías. |

#### D. `transactions` (Transacciones)
Registro continuo de los movimientos individuales de ingreso o gasto (Libro Diario).

| Campo | Tipo de Dato | Restricciones | Descripción |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | Primary Key, Default: `gen_random_uuid()` | Identificador único de la transacción. |
| `user_id` | `UUID` | FK -> `users(id)`, On Delete Cascade | Usuario propietario. |
| `account_id` | `UUID` | FK -> `accounts(id)`, On Delete Restrict | Cuenta de origen o destino. |
| `category_id` | `UUID` | FK -> `categories(id)`, On Delete Restrict | Categoría asociada. |
| `type` | `VARCHAR(20)` | Enum: `expense`, `income` | Tipo de transacción. |
| `amount` | `DECIMAL(12,2)` | Not Null, Check: `amount > 0` | Valor monetario (siempre positivo). |
| `currency` | `VARCHAR(3)` | Default: `'USD'` | Moneda de la transacción. |
| `date` | `TIMESTAMPTZ` | Not Null | Fecha y hora en que ocurrió el movimiento. |
| `description` | `TEXT` | Nullable | Detalle u observaciones de la transacción. |
| `created_at` | `TIMESTAMPTZ` | Default: `NOW()` | Fecha de registro en base de datos. |

#### E. `transfers` (Transferencias entre Cuentas)
Movilidad de fondos entre cuentas del propio usuario sin afectar ingresos o gastos globales.

| Campo | Tipo de Dato | Restricciones | Descripción |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | Primary Key, Default: `gen_random_uuid()` | Identificador único de la transferencia. |
| `user_id` | `UUID` | FK -> `users(id)`, On Delete Cascade | Usuario propietario. |
| `from_account_id` | `UUID` | FK -> `accounts(id)`, On Delete Restrict | Cuenta de origen (Abono). |
| `to_account_id` | `UUID` | FK -> `accounts(id)`, On Delete Restrict | Cuenta de destino (Cargo). |
| `amount` | `DECIMAL(12,2)` | Not Null, Check: `amount > 0` | Monto transferido. |
| `fee` | `DECIMAL(12,2)` | Default: `0.00` | Comisión bancaria asociada. |
| `date` | `TIMESTAMPTZ` | Not Null | Fecha y hora de ejecución. |
| `description` | `TEXT` | Nullable | Detalle de la transferencia. |

---

## 2. Estructura JSON de Endpoints REST Principales

### A. Registrar una Transacción (`POST /api/v1/transactions`)

#### Request Body
```json
{
  "account_id": "b3c91d4e-5a6b-7c8d-9e0f-1a2b3c4d5e6f",
  "category_id": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
  "type": "expense",
  "amount": 24.50,
  "currency": "USD",
  "date": "2026-08-07T14:30:00Z",
  "description": "Compra de supermercado",
  "tags": ["comida", "hogar"]
}
```

#### Response Payload (`201 Created`)
```json
{
  "status": "success",
  "data": {
    "transaction": {
      "id": "f8e7d6c5-b4a3-2f1e-0d9c-8b7a6f5e4d3c",
      "account_id": "b3c91d4e-5a6b-7c8d-9e0f-1a2b3c4d5e6f",
      "category": {
        "id": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
        "name": "Supermercado",
        "type": "expense"
      },
      "type": "expense",
      "amount": 24.50,
      "currency": "USD",
      "date": "2026-08-07T14:30:00Z",
      "description": "Compra de supermercado",
      "tags": ["comida", "hogar"],
      "created_at": "2026-08-07T14:34:00Z"
    },
    "account_summary": {
      "account_id": "b3c91d4e-5a6b-7c8d-9e0f-1a2b3c4d5e6f",
      "account_name": "Banco Principal",
      "previous_balance": 450.00,
      "new_balance": 425.50
    }
  }
}
```

---

### B. Registrar una Transferencia entre Cuentas (`POST /api/v1/transfers`)

#### Request Body
```json
{
  "from_account_id": "b3c91d4e-5a6b-7c8d-9e0f-1a2b3c4d5e6f",
  "to_account_id": "c4d5e6f7-a8b9-0c1d-2e3f-4a5b6c7d8e9f",
  "amount": 50.00,
  "fee": 0.50,
  "date": "2026-08-07T14:35:00Z",
  "description": "Retiro de cajero a efectivo"
}
```

#### Response Payload (`201 Created`)
```json
{
  "status": "success",
  "data": {
    "transfer_id": "e9f8d7c6-b5a4-3f2e-1d0c-9b8a7f6e5d4c",
    "amount": 50.00,
    "fee": 0.50,
    "from_account": {
      "id": "b3c91d4e-5a6b-7c8d-9e0f-1a2b3c4d5e6f",
      "name": "Banco Principal",
      "new_balance": 375.00
    },
    "to_account": {
      "id": "c4d5e6f7-a8b9-0c1d-2e3f-4a5b6c7d8e9f",
      "name": "Billetera Efectivo",
      "new_balance": 80.00
    }
  }
}
```

---

### C. Obtener Saldos y Estado Patrimonial (`GET /api/v1/balances`)

#### Query Parameters
`GET /api/v1/balances?currency=USD&include_inactive=false`

#### Response Payload (`200 OK`)
```json
{
  "status": "success",
  "data": {
    "net_worth": {
      "total_assets": 1250.00,
      "total_liabilities": 200.00,
      "net_balance": 1050.00,
      "currency": "USD"
    },
    "accounts": [
      {
        "id": "b3c91d4e-5a6b-7c8d-9e0f-1a2b3c4d5e6f",
        "name": "Banco Principal",
        "type": "checking",
        "currency": "USD",
        "current_balance": 375.00,
        "is_asset": true
      },
      {
        "id": "c4d5e6f7-a8b9-0c1d-2e3f-4a5b6c7d8e9f",
        "name": "Billetera Efectivo",
        "type": "cash",
        "currency": "USD",
        "current_balance": 80.00,
        "is_asset": true
      },
      {
        "id": "d5e6f7a8-b9c0-1d2e-3f4a-5b6c7d8e9f0a",
        "name": "Tarjeta de Crédito",
        "type": "credit_card",
        "currency": "USD",
        "current_balance": -200.00,
        "is_asset": false
      }
    ],
    "monthly_summary": {
      "period": "2026-08",
      "total_income": 1200.00,
      "total_expenses": 350.00,
      "savings_rate_percentage": 70.83
    }
  }
}
```

---

## 3. Consideraciones Técnicas, Notas y Diseño de Arquitectura

### A. Manejo de Precisión Financiera
* **Evitar flotantes en almacenamiento y cómputo:** Nunca almacenes ni proceses dinero usando `FLOAT` o `DOUBLE` debido al error de redondeo IEEE 754.
* **Tipos de datos sugeridos:**
  * En SQL: Usar `DECIMAL(12,2)` o `NUMERIC(12,2)`.
  * En código Backend: Usar tipos específicos de precisión como `Decimal` en Python/C#, `BigDecimal` en Java, o almacenar **centavos como enteros** (`$10.50` -> `1050`) en motores NoSQL o estructuras JSON.

### B. Consistencia e Integridad Transaccional (ACID)
* **Transacciones de Base de Datos:** Las operaciones que alteran el saldo de las cuentas (creación/modificación de transacciones o transferencias) deben envolverse estrictamente en un bloque de transacción SQL (`BEGIN TRANSACTION ... COMMIT`).
* **Actualización del saldo actual (`current_balance`):**
  * Para evitar condiciones de carrera (*race conditions*) en ejecuciones concurrentes, utiliza bloqueos pesimistas (`SELECT ... FOR UPDATE`) o recalculado por agregación mediante triggers/jobs en background.
* **Aislamiento en Transferencias:** Un registro en la tabla `transfers` representa una operación atómica donde se debita de `from_account_id` y se acredita en `to_account_id`. Ambas actualizaciones deben ser exitosas o revertirse completamente (*Rollback*).

### C. Estrategia de Indexación y Rendimiento
Para garantizar consultas rápidas en filtros por rango de fechas y cuentas:
* **Índices Compuestos sugeridos:**
  * `CREATE INDEX idx_transactions_user_date ON transactions (user_id, date DESC);`
  * `CREATE INDEX idx_transactions_account ON transactions (account_id, date DESC);`
  * `CREATE INDEX idx_transfers_from_to ON transfers (user_id, from_account_id, to_account_id);`
* **Paginación obligatoria:** Endpoints de listados de movimientos deberán usar paginación basada en cursor (`cursor-based pagination`) en lugar de `OFFSET/LIMIT` tradicional para evitar degradación de rendimiento a escala.

### D. Idempotencia y Resiliencia
* **Header de Idempotencia:** Para prevenir el registro duplicado de pagos o compras por pérdidas de conexión en dispositivos móviles, los endpoints `POST` deben aceptar el encabezado `Idempotency-Key: <UUID>`.
* **Manejo de zonas horarias:** Los campos de fecha deben almacenarse obligatoriamente con desplazamiento de zona horaria (`TIMESTAMPTZ` o formato ISO 8601 UTC `YYYY-MM-DDTHH:MM:SSZ`).

### E. Seguridad y Multitenancy
* **Aislamiento estricto por usuario:** Todas las consultas SQL (`SELECT`, `UPDATE`, `DELETE`) deben incluir la condición explícita `WHERE user_id = :authenticated_user_id` para prevenir vulnerabilidades de referencia directa e insegura a objetos (IDOR).
* **Autenticación en la API:** Se recomienda implementar **JWT (JSON Web Tokens)** mediante Bearer Tokens en encabezados HTTP o manejo de sesiones con tokens de renovación (*Refresh Tokens*).

---

## 4. Futuras Mejoras y Funcionalidades a Implementar

A medida que el sistema evolucione de un core de registro transaccional hacia un ecosistema financiero integral, se sugieren las siguientes fases de desarrollo:

### Fase 1: Automatización e Ingesta de Datos
* **Parsing e Ingesta de Notificaciones SMS/Push:**
  * Desarrollar un servicio en la app móvil que escuche notificaciones bancarias locales para pre-llenar transacciones automáticamente mediante Expresiones Regulares (Regex) o clasificadores livianos.
* **Procesamiento de Recibos / OCR:**
  * Agregar soporte para adjuntar imágenes de facturas y recibos (`POST /api/v1/transactions/ocr`) con extracción de texto mediante OCR para categorizar y extraer el monto de forma automática.
* **Programación de Transacciones Recurrentes:**
  * Módulo para suscripciones, arriendos o pagos de servicios que programe la creación automática de transacciones en fechas específicas o genere alertas de vencimiento (*CRON Jobs / Task Queues*).

### Fase 2: Planificación Financiera Avanzada y Análisis
* **Módulo de Presupuestación (Presupuesto Base Cero):**
  * Creación de metas de gasto por categoría (`budgets`) con alertas al alcanzar el 80% o 100% del límite mensual.
* **Proyecciones de Flujo de Caja (Cash Flow Forecasting):**
  * Algoritmos predictivos basados en series de tiempo para proyectar saldos futuros considerando recurrencias e ingresos esperados.
* **Simulador de Amortización de Deudas:**
  * Motor de cálculo para amortización de préstamos y tarjetas de crédito con estrategias de pago optimizadas (*Método Bola de Nieve vs. Avalancha*).

### Fase 3: Integraciones y Notificaciones Asíncronas
* **Integración con Bots (Telegram / WhatsApp / Discord):**
  * Exponer webhooks securizados para registrar ingresos y gastos mediante comandos rápidos o mensajes de voz procesados por NLP (ej. *"Gasté $15 en gasolina con tarjeta"*).
* **Alertas y Notificaciones Push (WebSockets / FCM):**
  * Notificaciones de sobregastos en presupuestos, vencimiento de tarjetas de crédito y reporte de resumen semanal/mensual automatizado.

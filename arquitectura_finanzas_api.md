# Arquitectura de Datos y Especificación de API REST para Finanzas Personales

Este documento define el modelo de datos relacional y las especificaciones de la API REST JSON para la gestión de finanzas personales basada en el principio de contabilidad por partida doble (Debe y Haber).

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
| `current_balance` | `DECIMAL(12,2)` | Default: `0.00` | Saldo calculado en tiempo real. |
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

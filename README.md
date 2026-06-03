# FormaPro Academy — n8n Payment Webhook

Solución para la prueba técnica de automatización de **FormaPro Academy**. El objetivo es recibir pagos por webhook, responder rápido, normalizar datos, guardar pagos válidos en Supabase, evitar duplicados, reflejar reembolsos y enviar emails de bienvenida.

---

## Entregables principales

- `formapro_n8n_workflow.json` — workflow de n8n.
- `supabase_operations_payments.sql` — estructura SQL de Supabase.
- `supabase_payments_evidence.csv` / `supabase_payments_evidence.json` — evidencia esperada/final de resultados.
- `test_payments.json` — payloads oficiales de prueba.
- `send_webhook_tests.sh` — script auxiliar para pruebas por `curl`.
- `ui/index.html` — UI sencilla para testear el webhook.
- `MASTERCLASS_FORMAPRO_IMPLEMENTACION.md` — explicación técnica extendida.

---

## Arquitectura del workflow

```text
Webhook - Payment Received
  ↓
Respond - OK Fast
  ↓
Code - Normalize Payment Data
  ↓
Switch - Payment Status
  ├── completed + valid
  │     ↓
  │   Supabase - Insert Completed Idempotent
  │     ↓
  │   Code - Keep Only Inserted Rows
  │     ↓
  │   IF - Inserted Row Has Valid Email
  │     ↓
  │   Email - Welcome HTML
  │
  └── refunded + valid
        ↓
      Supabase - Mark Refunded
```

El webhook responde rápido con:

```json
{ "ok": true }
```

La respuesta rápida funciona como acuse de recibo. El procesamiento de Supabase y email ocurre después.

---

## Variables necesarias en n8n

Para no hardcodear secretos en el workflow exportado, se usan variables de n8n:

| Variable | Ejemplo |
|---|---|
| `SUPABASE_URL` | `https://tu-proyecto.supabase.co` |
| `SUPABASE_SERVICE_ROLE_KEY` | Service role key de Supabase |
| `SUPABASE_SCHEMA` | `operations` |

En los nodos HTTP de Supabase se usan así:

```text
URL: ={{$vars.SUPABASE_URL}}/rest/v1/payments?on_conflict=id_pago
apikey: ={{$vars.SUPABASE_SERVICE_ROLE_KEY}}
Authorization: =Bearer {{$vars.SUPABASE_SERVICE_ROLE_KEY}}
Content-Profile: ={{$vars.SUPABASE_SCHEMA}}
Accept-Profile: ={{$vars.SUPABASE_SCHEMA}}
```

> Nota: para producción real usaría un gestor de secretos o credenciales cifradas. En n8n Cloud, algunas funciones avanzadas de secretos pueden requerir plan pago, por eso se usaron variables para esta prueba.

---

## Estructura de Supabase

```sql
create schema if not exists operations;

create table if not exists operations.payments (
  id_pago TEXT NOT NULL,
  email TEXT,
  nombre TEXT NOT NULL,
  curso TEXT NOT NULL,
  importe NUMERIC NOT NULL,
  moneda TEXT NOT NULL,
  estado TEXT NOT NULL,
  fecha timestamptz NOT NULL,
  refunded_at timestamptz,
  created_at timestamptz default now(),
  updated_at timestamptz default now(),
  CONSTRAINT payments_pk PRIMARY KEY (id_pago)
);

grant usage on schema operations to service_role;
grant all on all tables in schema operations to service_role;
alter default privileges in schema operations grant all on tables to service_role;
```

La clave de idempotencia es:

```sql
constraint payments_pk primary key (id_pago)
```

Campos `not null` usados:

| Campo | Motivo |
|---|---|
| `id_pago` | Necesario para idempotencia. |
| `email` | Regla estricta: no se guardan pagos `completed` sin email válido. |
| `curso` | Campo requerido del pago completado. |
| `importe` | Campo requerido y validado como número `> 0`. |
| `moneda` | Campo requerido y validado como `cop` o `usd`. |
| `estado` | Necesario para saber si el pago está `completed` o `refunded`. |
| `fecha` | Fecha del evento de pago. |
| `created_at` / `updated_at` | Timestamps operativos siempre presentes. |

`nombre` queda nullable porque no bloquea el pago; si viene vacío, el email usa el fallback `cliente`. `refunded_at` queda nullable porque solo existe cuando el pago fue reembolsado.

---

## Criterios de aceptación

### 1. Webhook y respuesta rápida

El flujo recibe pagos con un nodo `Webhook` y responde inmediatamente:

```json
{ "ok": true }
```

### 2. Guardar pagos completados

Solo los pagos `completed` que pasan validaciones críticas se insertan en Supabase. Con la regla actual, un `completed` debe tener email válido para guardarse.

### 3. No guardar `failed` ni `refunded` como pagos nuevos

- `failed`: se ignora y no se guarda.
- `refunded`: no se inserta como fila nueva.

Si llega un `refunded` de un pago previamente guardado, se refleja mediante `PATCH`, marcando el pago como `refunded` y guardando `refunded_at`.

Esta decisión cumple el enunciado porque permite **marcar o quitar** el pago reembolsado. Se eligió marcar para conservar trazabilidad.

### 4. Idempotencia

Para evitar duplicados:

- `id_pago` es primary key en Supabase.
- El insert usa `on_conflict=id_pago`.
- El header `Prefer` usa `resolution=ignore-duplicates,return=representation`.

Así, aunque el mismo pago llegue repetido o concurrente, queda una sola fila.

### 5. Email de bienvenida

El email HTML se envía solo si:

1. El pago es `completed`.
2. Fue insertado realmente.
3. Tiene email válido.
4. Si `nombre` viene vacío, usa fallback `cliente` para evitar saludos vacíos.

Además, como el email ahora es obligatorio para guardar pagos `completed`, un pago sin email válido no se inserta ni dispara email.

Esto evita emails duplicados para pagos repetidos y evita emails con saludo incompleto.

### 6. Datos sucios

El nodo de código normaliza y valida:

| Campo | Manejo |
|---|---|
| `id_pago` | `trim`, uppercase y whitelist `^[A-Z0-9-]+$` |
| `email` | `trim`; conserva local-part y normaliza solo el dominio; obligatorio y válido para guardar `completed` |
| `nombre` | `trim`; no es crítico para guardar; en email usa fallback `cliente` si viene vacío |
| `curso` | `trim`, obligatorio para `completed` |
| `importe` | convierte texto a número y valida `> 0` |
| `moneda` | `trim`, lowercase, solo `cop` o `usd` |
| `estado` | `trim`, lowercase; acepta `completed`, `failed`, `refunded` |
| `fecha` | parseo a fecha válida ISO |

Ejemplos:

```text
"  ANA.Lopez@GMAIL.com " → "ANA.Lopez@gmail.com"
"75000" → 75000
"COP" → "cop"
```

---

## Resultado esperado con los payloads oficiales

Después de enviar los 9 pagos del enunciado, Supabase debe quedar con 5 filas bajo la regla estricta de email válido:

| id_pago | email | moneda | estado | observación |
|---|---|---|---|---|
| `PAY-001` | `ana@gmail.com` | `cop` | `completed` | Duplicado controlado |
| `PAY-003` | `marta@gmail.com` | `cop` | `refunded` | Reembolso reflejado |
| `PAY-005` | `ANA.Lopez@gmail.com` | `cop` | `completed` | Email/moneda normalizados |
| `PAY-006` | `diego@gmail.com` | `cop` | `completed` | Importe texto convertido a número |
| `PAY-007` | `juan@gmail.com` | `cop` | `completed` | Pago válido |

No debe existir:

| id_pago | motivo |
|---|---|
| `PAY-002` | Pago `failed` |
| `PAY-004` | Email vacío; con la regla estricta no se guarda |
| `PAY-EMAIL-MISSING` | Email faltante |
| `PAY-EMAIL-INVALID` | Email con formato inválido |

---

## Consultas de verificación

```sql
select id_pago, email, nombre, curso, importe, moneda, estado, fecha, refunded_at
from operations.payments
order by id_pago;
```

```sql
select count(*) as total_rows
from operations.payments;
```

Resultado esperado:

```text
5
```

```sql
select id_pago
from operations.payments
where id_pago in (
  'PAY-002',
  'PAY-004',
  'PAY-BAD-CURRENCY',
  'PAY-BAD-AMOUNT',
  'PAY-008;DROP',
  'PAY-BAD-DATE',
  'PAY-EMAIL-MISSING',
  'PAY-EMAIL-INVALID'
);
```

Resultado esperado:

```text
0 filas
```

---

## UI de testeo

Este repositorio incluye una UI estática en:

```text
ui/index.html
```

Sirve para facilitar pruebas manuales del webhook:

- Enviar payload actual.
- Enviar los 9 pagos oficiales.
- Duplicar el payload actual N veces.
- Enviar casos inválidos.
- Navegar el historial de payloads enviados.
- Copiar comandos `curl`.
- Ver el resultado esperado y SQL de verificación.

Más detalles en:

```text
ui/README.md
```

---

## Respuestas sugeridas del cuestionario

### Duplicados: ¿cómo evitaste guardar el mismo pago dos veces?

Usé `id_pago` como clave primaria en Supabase. Además, el insert se hace con `on_conflict=id_pago` y `resolution=ignore-duplicates`, así que si el mismo pago llega repetido o casi al mismo tiempo, Postgres garantiza que solo exista una fila. No dependí solo de una validación en n8n, sino de una restricción real en base de datos.

### Velocidad: ¿cómo preparaste el flujo para muchos pagos por segundo?

El webhook responde inmediatamente `{ "ok": true }` con un nodo `Respond to Webhook`, antes de ejecutar Supabase o enviar emails. Así la pasarela recibe respuesta rápida aunque después el flujo siga procesando. También delegué la idempotencia a la base de datos para soportar pagos repetidos o concurrentes.

### Reembolso: ¿qué hiciste con el `refunded` de un pago ya guardado?

Los pagos `refunded` no se insertan como filas nuevas. Si llega un `refunded` de un pago ya guardado, hago un `PATCH` sobre ese `id_pago` y marco la fila como `estado = refunded`, guardando también `refunded_at`. Elegí marcarlo en lugar de eliminarlo para conservar trazabilidad.

### Datos sucios: ¿cómo trataste espacios, mayúsculas e importe como texto?

Agregué un nodo de código para normalizar antes de guardar. Quito espacios, normalizo `estado` y `moneda`, convierto `importe` a número aunque venga como texto y valido fechas. Para email, hago `trim`, conservo la parte antes del `@`, normalizo el dominio a minúsculas y exijo que sea válido para guardar pagos `completed`. Además, valido moneda, importe e `id_pago`.

### Uso de IA

Usé IA como apoyo para diseñar la arquitectura, revisar la estrategia de idempotencia, escribir el SQL, estructurar el workflow, reforzar validaciones, crear la UI de pruebas y preparar documentación. Las decisiones clave fueron validadas contra los requisitos de la prueba.

---

## Nota de seguridad

No se recomienda entregar claves reales hardcodeadas dentro del JSON del workflow. Para la prueba se recomienda usar variables de n8n y rotar la `service_role key` después de finalizar la revisión.

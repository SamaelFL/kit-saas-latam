---
name: pagos-mx
description: Cobrar en un SaaS en México con Stripe y Mercado Pago sobre React, Supabase y Vercel. Paquetes de créditos, Checkout, webhooks con firma verificada, idempotencia, pagos en OXXO que llegan después, reembolsos y acreditación atómica en Postgres. Úsala cuando se hable de Stripe, Mercado Pago, checkout, webhooks de pago, créditos o tokens, OXXO, reembolsos o cualquier flujo de cobro.
---

# Pagos en México: Stripe y Mercado Pago

Cobrar bien es sobre todo no regalar créditos: que nadie se los dé a sí mismo,
que un webhook repetido no acredite dos veces y que un pago pendiente no cuente
como pagado.

## Cuál usar

| | Stripe | Mercado Pago |
|---|---|---|
| Tarjetas internacionales y cobros en USD | Mejor opción | Limitado |
| Tarjetas mexicanas, meses sin intereses, saldo de MP | Bien | Mejor opción |
| Pago en efectivo (OXXO) | Sí, llega horas o días después | Sí, llega horas o días después |
| Integración | Checkout alojado o elementos embebidos | Checkout Pro (redirige) o Bricks |

Muchos SaaS en México ofrecen los dos: Stripe como principal y Mercado Pago para
quien no tiene tarjeta o prefiere pagar con su cuenta.

## Las cinco reglas

1. **El precio y los créditos los decide el servidor.** El frontend solo manda qué
   paquete quiere; el monto sale de un catálogo en el backend, nunca del body.
2. **El usuario sale del token**, no de un `userId` que mande el cliente.
3. **Solo el webhook acredita**, nunca la pantalla de "pago exitoso".
4. **Firma verificada e idempotencia** en cada webhook: los proveedores reenvían.
5. **Acreditar es una sola operación en la base de datos**: registrar el pago y sumar
   créditos en la misma transacción, con el id del pago como llave única.

## Base de datos (Supabase)

```sql
create table public.credits (
  user_id uuid primary key references auth.users on delete cascade,
  balance integer not null default 0 check (balance >= 0),
  updated_at timestamptz not null default now()
);

create table public.payments (
  provider text not null check (provider in ('stripe', 'mercadopago')),
  provider_payment_id text not null,
  user_id uuid not null references auth.users on delete cascade,
  credits integer not null,
  amount_mxn numeric(10, 2) not null,
  status text not null default 'paid',
  created_at timestamptz not null default now(),
  primary key (provider, provider_payment_id)
);

alter table public.credits enable row level security;
alter table public.payments enable row level security;

-- El usuario puede VER su saldo y sus pagos. Nadie los modifica desde el cliente.
create policy "ver mi saldo" on public.credits
  for select to authenticated using ((select auth.uid()) = user_id);
create policy "ver mis pagos" on public.payments
  for select to authenticated using ((select auth.uid()) = user_id);
```

### Acreditar un pago una sola vez

```sql
create or replace function public.grant_payment(
  p_provider text, p_payment_id text, p_user_id uuid, p_credits integer, p_amount numeric
) returns boolean            -- true si acreditó, false si ese pago ya estaba
language plpgsql security definer set search_path = '' as $$
begin
  insert into public.payments (provider, provider_payment_id, user_id, credits, amount_mxn)
  values (p_provider, p_payment_id, p_user_id, p_credits, p_amount)
  on conflict do nothing;
  if not found then
    return false;            -- webhook repetido: no se acredita otra vez
  end if;

  insert into public.credits (user_id, balance) values (p_user_id, p_credits)
  on conflict (user_id) do update
    set balance = public.credits.balance + excluded.balance, updated_at = now();
  return true;
end $$;

-- Una función SECURITY DEFINER en public se puede llamar por RPC desde el navegador
-- si no se revoca. Sin esto, cualquiera se regala créditos.
revoke execute on function public.grant_payment(text, text, uuid, integer, numeric)
  from public, anon, authenticated;
```

### Gastar un crédito sin carreras

```sql
create or replace function public.consume_credit(p_user_id uuid)
returns boolean language sql security definer set search_path = '' as $$
  with upd as (
    update public.credits set balance = balance - 1, updated_at = now()
     where user_id = p_user_id and balance > 0
    returning 1
  )
  select exists (select 1 from upd);
$$;

create or replace function public.refund_credit(p_user_id uuid)
returns void language sql security definer set search_path = '' as $$
  update public.credits set balance = balance + 1, updated_at = now() where user_id = p_user_id;
$$;

revoke execute on function public.consume_credit(uuid) from public, anon, authenticated;
revoke execute on function public.refund_credit(uuid) from public, anon, authenticated;
```

Descuenta **antes** de hacer el trabajo caro (llamar al modelo) y regresa el crédito
si falla. Verificar y luego descontar deja pasar diez requests en paralelo con un
solo crédito.

## Catálogo en el servidor

```js
// api/lib/paquetes.js
export const PAQUETES = {
  basico: { creditos: 2, precio_mxn: 99, stripe_price: process.env.STRIPE_PRICE_BASICO },
  pro:    { creditos: 10, precio_mxn: 399, stripe_price: process.env.STRIPE_PRICE_PRO },
};
```

`requireUser` y el cliente `admin` (llave secreta de Supabase) vienen de la skill
`seguridad-saas`.

## Stripe

### Crear el checkout

```js
// api/checkout-stripe.js (runtime Node)
import Stripe from 'stripe';
import { PAQUETES } from './lib/paquetes.js';
import { requireUser } from './lib/auth.js';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);

export async function POST(request) {
  const user = await requireUser(request);
  if (!user) return Response.json({ error: 'No autenticado' }, { status: 401 });

  const { paquete } = await request.json().catch(() => ({}));
  const p = PAQUETES[paquete];
  if (!p) return Response.json({ error: 'Paquete inválido' }, { status: 400 });

  const session = await stripe.checkout.sessions.create({
    mode: 'payment',
    line_items: [{ price: p.stripe_price, quantity: 1 }],
    client_reference_id: user.id,          // lo pone el servidor, viene del token
    metadata: { paquete },
    success_url: `${process.env.APP_URL}/pago?estado=ok`,
    cancel_url: `${process.env.APP_URL}/pago?estado=cancelado`,
  });
  return Response.json({ url: session.url });
}
```

Crea el precio en el Dashboard en MXN con IVA incluido. Los métodos de pago (tarjeta,
OXXO) se activan en Settings → Payment methods. Crear dos sesiones por doble clic no
cobra dos veces: la idempotencia que importa es la del webhook.

### Webhook

```js
// api/webhook-stripe.js (runtime Node)
import Stripe from 'stripe';
import { PAQUETES } from './lib/paquetes.js';
import { admin } from './lib/auth.js';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);

export async function POST(request) {
  const raw = await request.text(); // cuerpo crudo: si lo parseas antes, la firma falla
  let event;
  try {
    event = stripe.webhooks.constructEvent(
      raw, request.headers.get('stripe-signature'), process.env.STRIPE_WEBHOOK_SECRET);
  } catch {
    return new Response('Firma inválida', { status: 400 });
  }

  const pagado =
    (event.type === 'checkout.session.completed' && event.data.object.payment_status === 'paid') ||
    event.type === 'checkout.session.async_payment_succeeded'; // OXXO, días después

  if (pagado) {
    const s = event.data.object;
    const p = PAQUETES[s.metadata?.paquete];
    if (!p || s.currency !== 'mxn' || s.amount_total !== p.precio_mxn * 100) {
      console.error('[stripe] monto o paquete no coincide', s.id);
      return new Response('ok'); // no reintentar, pero tampoco acreditar
    }
    const { error } = await admin.rpc('grant_payment', {
      p_provider: 'stripe', p_payment_id: s.id, p_user_id: s.client_reference_id,
      p_credits: p.creditos, p_amount: p.precio_mxn,
    });
    if (error) return new Response('Error', { status: 500 }); // Stripe reintenta
  }
  return new Response('ok');
}
```

Eventos a suscribir: `checkout.session.completed`,
`checkout.session.async_payment_succeeded`, `checkout.session.async_payment_failed`
y `charge.refunded`. En local: `stripe listen --forward-to localhost:3000/api/webhook-stripe`.

## Mercado Pago

### Crear la preferencia

```js
// api/checkout-mp.js
import { MercadoPagoConfig, Preference } from 'mercadopago';
import { PAQUETES } from './lib/paquetes.js';
import { requireUser } from './lib/auth.js';

const mp = new MercadoPagoConfig({ accessToken: process.env.MP_ACCESS_TOKEN });

export async function POST(request) {
  const user = await requireUser(request);
  if (!user) return Response.json({ error: 'No autenticado' }, { status: 401 });

  const { paquete } = await request.json().catch(() => ({}));
  const p = PAQUETES[paquete];
  if (!p) return Response.json({ error: 'Paquete inválido' }, { status: 400 });

  const pref = await new Preference(mp).create({
    body: {
      items: [{ id: paquete, title: `${p.creditos} créditos`, quantity: 1,
                unit_price: p.precio_mxn, currency_id: 'MXN' }],
      external_reference: user.id,         // lo pone el servidor
      metadata: { paquete },
      back_urls: {
        success: `${process.env.APP_URL}/pago?estado=ok`,
        pending: `${process.env.APP_URL}/pago?estado=pendiente`,
        failure: `${process.env.APP_URL}/pago?estado=error`,
      },
      auto_return: 'approved',
      notification_url: `${process.env.APP_URL}/api/webhook-mp`,
    },
  });
  return Response.json({ url: pref.init_point });
}
```

### Webhook con firma verificada

Configura Webhooks (no IPN) en Tus integraciones → tu aplicación → Webhooks, con el
evento **Pagos**, y copia la clave secreta a `MP_WEBHOOK_SECRET`.

```js
// api/webhook-mp.js (runtime Node)
import crypto from 'node:crypto';
import { MercadoPagoConfig, Payment } from 'mercadopago';
import { PAQUETES } from './lib/paquetes.js';
import { admin } from './lib/auth.js';

const mp = new MercadoPagoConfig({ accessToken: process.env.MP_ACCESS_TOKEN });

function firmaValida(request, dataId) {
  const header = request.headers.get('x-signature') ?? '';
  const partes = Object.fromEntries(header.split(',').map((kv) => kv.trim().split('=')));
  const { ts, v1 } = partes;
  if (!ts || !v1) return false;
  const requestId = request.headers.get('x-request-id');
  let manifest = '';
  if (dataId) manifest += `id:${dataId.toLowerCase()};`;
  if (requestId) manifest += `request-id:${requestId};`;
  manifest += `ts:${ts};`;
  const esperado = crypto.createHmac('sha256', process.env.MP_WEBHOOK_SECRET)
    .update(manifest).digest('hex');
  return v1.length === esperado.length &&
    crypto.timingSafeEqual(Buffer.from(v1), Buffer.from(esperado));
}

export async function POST(request) {
  const url = new URL(request.url);
  const dataId = url.searchParams.get('data.id');   // el id firmado viene en la URL
  if (!firmaValida(request, dataId)) return new Response('Firma inválida', { status: 401 });

  const body = await request.json().catch(() => ({}));
  if (body.type !== 'payment' || !dataId) return new Response('ok');

  // Nunca confíes en el body: consulta el pago a la API de Mercado Pago
  const pago = await new Payment(mp).get({ id: dataId });
  if (pago.status !== 'approved') return new Response('ok'); // pending (OXXO) llega después

  const p = PAQUETES[pago.metadata?.paquete];
  if (!p || pago.currency_id !== 'MXN' || Number(pago.transaction_amount) !== p.precio_mxn) {
    console.error('[mp] monto o paquete no coincide', pago.id);
    return new Response('ok');
  }
  const { error } = await admin.rpc('grant_payment', {
    p_provider: 'mercadopago', p_payment_id: String(pago.id), p_user_id: pago.external_reference,
    p_credits: p.creditos, p_amount: p.precio_mxn,
  });
  if (error) return new Response('Error', { status: 500 }); // MP reintenta
  return new Response('ok');
}
```

Detalles de Mercado Pago que muerden:
- Manda la misma notificación varias veces y a veces fuera de orden: por eso la llave
  única en `payments`.
- Un pago en OXXO llega primero como `pending` y días después como `approved`; acredita
  solo con `approved`.
- Las claves del `metadata` regresan en minúsculas y con guiones bajos; usa nombres así
  desde el inicio (`paquete`, no `Paquete` ni `paqueteId`).
- Las cuentas y tarjetas de prueba son distintas a las de producción. Crea usuarios de
  prueba desde el panel de desarrolladores.
- Responde rápido con 200; si tardas o fallas, reintenta.

## Reembolsos y contracargos

- Stripe: `charge.refunded` y `charge.dispute.created`. Mercado Pago: el pago cambia a
  `refunded` o `charged_back`.
- Marca el pago como `refunded` y descuenta los créditos que no se hayan usado. Si ya
  se gastaron, decide una regla (saldo negativo bloqueado, por ejemplo) y escríbela en
  tus términos.

## Errores comunes

| Síntoma | Causa | Arreglo |
|---|---|---|
| La firma de Stripe nunca valida | El body se parseó antes | Leer `request.text()` y pasar el texto crudo |
| La firma de MP nunca valida | `data.id` con mayúsculas o sacado del body | Tomarlo de la URL y pasarlo a minúsculas |
| Créditos dobles | Webhook reenviado sin llave única | `grant_payment` con `on conflict do nothing` |
| Créditos gratis desde la consola del navegador | Función `security definer` sin `revoke` | `revoke execute ... from public, anon, authenticated` |
| El usuario edita su saldo | Política `for all` en `credits` | Solo `for select` para el usuario |
| Pago en OXXO nunca se acredita | Solo se escucha `checkout.session.completed` | Agregar `async_payment_succeeded` |

## Checklist antes de cobrar en producción

- [ ] Precio y créditos salen del catálogo del servidor
- [ ] `grant_payment`, `consume_credit` y `refund_credit` revocadas para `anon` y `authenticated`
- [ ] El usuario solo puede leer `credits` y `payments`
- [ ] Webhooks de Stripe y de Mercado Pago con firma verificada
- [ ] Monto y moneda comparados contra el catálogo antes de acreditar
- [ ] Pagos pendientes (OXXO) acreditados solo al confirmarse
- [ ] Reembolsos y contracargos manejados
- [ ] Llaves de producción solo en variables de entorno del servidor
- [ ] Probado de punta a punta con cuentas de prueba de ambos proveedores

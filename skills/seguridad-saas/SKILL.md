---
name: seguridad-saas
description: Seguridad para un SaaS con React, Supabase y Vercel, mapeada a OWASP Top 10:2025. Rate limiting que funciona en serverless, créditos de IA sin doble gasto, llaves nuevas de Supabase, validación con schema, CSP y headers, CORS, webhooks firmados, bots y supply chain de npm. Úsala cuando se hable de seguridad, rate limit, API keys, XSS, CSRF, CSP, inyección, bots, dependencias de npm o antes de lanzar, y de forma proactiva si ves llaves expuestas, endpoints sin límite o cobros no atómicos.
---

# Seguridad para un SaaS con React, Supabase y Vercel

Proteger la app, a los usuarios y el dinero. Revisada en octubre de 2026.

Si un endpoint manda texto de usuarios o de terceros a un LLM que tiene herramientas
o acceso a datos, además de esto necesitas defensas contra prompt injection.

## Amenazas reales para este stack

```
Amenaza                                → Impacto                             OWASP 2025
──────────────────────────────────────────────────────────────────────────────────────
Llave de Claude o sb_secret_ expuesta  → Queman tus créditos / leen toda la BD   A02, A04
Endpoint de IA sin límite real         → Factura de API, DoS                     A06 (+LLM10 consumo)
Cobro de crédito no atómico            → Generaciones gratis con requests en paralelo  A06
RLS mal puesto o función DEFINER abierta → Ver/editar datos de otros usuarios    A01
Webhook sin firma                      → Pagos falsos, créditos gratis           A08
Dependencia npm comprometida           → Robo de tokens, persistencia en tu equipo A03
Input sin validar / HTML sin limpiar   → XSS, robo de sesión                     A05
Errores que filtran stack o datos      → Info para el atacante                   A10
```

OWASP Top 10:2025: A01 Broken Access Control, A02 Security Misconfiguration,
A03 Software Supply Chain Failures, A04 Cryptographic Failures, A05 Injection,
A06 Insecure Design, A07 Authentication Failures, A08 Software or Data Integrity
Failures, A09 Security Logging and Alerting Failures, A10 Mishandling of
Exceptional Conditions. Usa estos IDs al reportar.

## Rate limiting (prioridad #1)

**No uses un `Map` en memoria en Vercel.** Cada instancia de la función tiene su
propia memoria, se reciclan y escalan en paralelo: el contador se reinicia y el
límite real es N veces el que escribiste. Sirve solo en desarrollo local.

### Opción A — Vercel WAF + `@vercel/firewall` (recomendada si tu plan la incluye)

1. Dashboard → proyecto → Firewall → Configure → New Rule. Condición `@vercel/firewall`,
   Rate limit ID por ejemplo `ia-generate`. Publicar.
2. En el código (Vercel Function con el formato Web `Request`):

```javascript
// api/generate.js
import { checkRateLimit } from '@vercel/firewall';

export async function POST(request) {
  const user = await requireUser(request); // ver "Auth en API routes"
  if (!user) return json({ error: 'No autenticado' }, 401);

  // Bucket por usuario. Por defecto la llave es la IP del cliente.
  const { rateLimited } = await checkRateLimit('ia-generate', {
    request,
    rateLimitKey: user.id,
    timeout: 1000, // @vercel/firewall >= 1.2.6
  });
  if (rateLimited) return json({ error: 'Demasiadas solicitudes, intenta en un momento.' }, 429);

  // ... lógica
}

const json = (body, status = 200) =>
  new Response(JSON.stringify(body), { status, headers: { 'Content-Type': 'application/json' } });
```

Notas de la doc de Vercel: los contadores son por región (tráfico desde varias
regiones puede pasar el límite de una sola), un `rateLimitKey` constante vuelve la
regla global, y para que corra en previews hay que habilitar Protection Bypass for
Automation. Revisa precio y límites por plan en la doc de WAF Rate Limiting antes de
depender de esto.

### Opción B — Función atómica en Supabase (gratis, persistente)

La versión anterior de esta skill hacía SELECT y luego UPDATE: con requests en
paralelo dos pasan el chequeo a la vez. Esta lo hace en una sola sentencia:

```sql
create table public.rate_limits (
  key text primary key,
  count integer not null,
  window_start timestamptz not null
);
alter table public.rate_limits enable row level security; -- sin policies: nadie desde el cliente

create or replace function public.check_rate_limit(
  p_key text, p_max integer, p_window_seconds integer
) returns boolean            -- true = limitado
language plpgsql
security definer
set search_path = ''
as $$
declare v_count integer;
begin
  insert into public.rate_limits as r (key, count, window_start)
  values (p_key, 1, now())
  on conflict (key) do update set
    count = case when r.window_start < now() - make_interval(secs => p_window_seconds)
                 then 1 else r.count + 1 end,
    window_start = case when r.window_start < now() - make_interval(secs => p_window_seconds)
                 then now() else r.window_start end
  returning count into v_count;
  return v_count > p_max;
end $$;

-- Una función SECURITY DEFINER en public se puede llamar por RPC con la llave
-- publishable si no se revoca. Que solo la llame tu servidor:
revoke execute on function public.check_rate_limit(text, integer, integer)
  from public, anon, authenticated;
```

Desde la API route, con el cliente de servidor (llave secreta):
`const { data: limited } = await admin.rpc('check_rate_limit', { p_key: \`gen:${user.id}\`, p_max: 10, p_window_seconds: 60 })`.

Limpieza: un cron (pg_cron o tarea programada) que borre filas con `window_start`
de más de un día.

### Límites de partida

```
Endpoint               Requests   Ventana   Llave        Razón
───────────────────────────────────────────────────────────────────
/api/generate (IA)     10         1 min     user.id      Créditos de Claude
/api/create-checkout   5          1 min     user.id      Spam de pagos
signup                 3          5 min     IP           Cuentas masivas
login                  5          1 min     IP + email   Fuerza bruta
/api/* general         60         1 min     IP           Límite global
```

Prefiere `user.id` en todo lo autenticado: la IP la comparten oficinas y redes
móviles en México (CGNAT), y un atacante rota IPs fácil.

## Llaves y secretos

### Llaves de Supabase (cambio importante)

Supabase reemplaza las llaves JWT legacy por llaves nuevas y **deprecia `anon` y
`service_role` a fines de 2026**:

| Nueva | Reemplaza a | Dónde va | Rol de Postgres |
|---|---|---|---|
| `sb_publishable_...` | `anon` | Frontend, está bien que sea pública | `anon` / `authenticated` |
| `sb_secret_...` | `service_role` | Solo servidor (API routes, Edge Functions, n8n) | `service_role`, se salta RLS |

- La llave secreta devuelve 401 si se usa desde un navegador (Supabase revisa el
  `User-Agent`), pero desde curl o un script funciona: si se filtra, bórrala.
- Puedes tener una llave secreta por componente (API de Vercel, n8n, script de
  migración) y rotar solo la que se filtró. Borrar una llave secreta no se puede
  deshacer; desactivar las legacy sí, así que primero migra todo y luego desactiva.
- Las llaves nuevas no son JWT: van en el header `apikey`, nunca en
  `Authorization: Bearer`. `Authorization` es solo para el JWT del usuario.
- Edge Functions: `verify_jwt` acepta también las llaves nuevas por compatibilidad,
  así que pasar ese chequeo **no autentica al que llama**. Autoriza en el código. Para
  webhooks externos, `verify_jwt = false` en `supabase/config.toml` y verifica la firma.
- Migración: crea las llaves nuevas, reemplaza en Vercel/n8n/apps, confirma que nada
  usa las legacy, y desactívalas en Settings → API Keys.

### Reglas absolutas

- Ninguna llave privada en el frontend ni en variables `VITE_*` (Vite las mete al
  bundle). En el cliente solo la URL de Supabase y la publishable.
- Llave de Claude, `sb_secret_`, Stripe/Mercado Pago secret: solo en env vars de
  Vercel, marcadas como Sensitive.
- `.env*` en `.gitignore`, y escaneo de secretos en el repo (GitHub secret scanning
  o un escáner como gitleaks).
- Ponle límite de gasto mensual a la llave de Claude en la consola de Anthropic y
  una llave distinta por producto, para que un abuso en una app no tumbe las otras.

## Endpoint de IA sin doble gasto

El patrón viejo verificaba créditos, llamaba a Claude y luego descontaba. Diez
requests en paralelo con 1 crédito pasaban todos el chequeo. Reserva el crédito de
forma atómica ANTES de llamar al modelo y devuélvelo si falla (misma tabla `credits`
que la skill `pagos-mx`):

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

```javascript
// api/generate.js (continúa)
const MAX_INPUT_CHARS = 10000;

export async function POST(request) {
  const user = await requireUser(request);
  if (!user) return json({ error: 'No autenticado' }, 401);
  // ... rate limit (arriba)

  const parsed = GenerateInput.safeParse(await request.json().catch(() => null));
  if (!parsed.success) return json({ error: 'Datos inválidos' }, 400);
  if (JSON.stringify(parsed.data).length > MAX_INPUT_CHARS) return json({ error: 'Texto muy largo' }, 413);

  const { data: ok } = await admin.rpc('consume_credit', { p_user_id: user.id });
  if (!ok) return json({ error: 'Sin créditos' }, 402);

  try {
    const result = await callClaude({
      system: SYSTEM_PROMPT,            // fijo en el servidor
      messages: buildMessages(parsed.data),
      max_tokens: 1500,                 // fijo, nunca del frontend
      // modelo fijo en el servidor, nunca del frontend
    });
    return json(result);
  } catch (err) {
    await admin.rpc('refund_credit', { p_user_id: user.id });
    console.error('[generate]', err?.status, err?.message); // log interno
    return json({ error: 'No se pudo generar, tu crédito se regresó.' }, 502);
  }
}
```

Además: nunca aceptes del cliente `model`, `max_tokens`, `system` ni `tools`; limita
generaciones por usuario por día; y alerta si el gasto diario de la llave sube fuera
de lo normal.

## Auth en API routes

```javascript
// api/lib/auth.js
import { createClient } from '@supabase/supabase-js';

export const admin = createClient(process.env.SUPABASE_URL, process.env.SUPABASE_SECRET_KEY, {
  auth: { persistSession: false, autoRefreshToken: false },
});

export async function requireUser(request) {
  const token = request.headers.get('authorization')?.replace(/^Bearer /, '');
  if (!token) return null;
  const { data, error } = await admin.auth.getUser(token); // valida contra Supabase Auth
  return error ? null : data.user;
}
```

- Lo que el usuario puede ver o tocar lo decide RLS o tu código con `user.id` del
  token, **nunca** un `userId` que venga en el body (IDOR, A01).
- Para leer datos del propio usuario, mejor un cliente con su JWT (RLS aplica) que el
  cliente admin: si te equivocas en un filtro, RLS te salva.
- `ProtectedRoute` en React es UX, no seguridad.
- supabase-js guarda la sesión en `localStorage` por defecto: un XSS puede robarla.
  Por eso CSP y cero `dangerouslySetInnerHTML` sin limpiar no son opcionales. Si usas
  SSR, `@supabase/ssr` con cookies `HttpOnly`, `Secure`, `SameSite=Lax`.

## Validación e inyección (A05)

- Valida con un schema en el servidor (zod o similar): tipo, longitud, formato,
  enums. Rechaza lo que no cumpla; no "limpies" lo que no entiendes.
- Quitar tags con regex (`str.replace(/<[^>]*>/g, '')`) **no es sanitización**: se
  evade con HTML mal formado. Escapa al renderizar (React ya lo hace en JSX) y usa
  DOMPurify si de verdad tienes que meter HTML (por ejemplo markdown renderizado).
- Output de un LLM es input no confiable: no lo metas a `dangerouslySetInnerHTML`
  sin DOMPurify, y no renderices imágenes remotas que el modelo genere (canal de
  exfiltración).
- SQL: siempre por supabase-js o RPC con parámetros, nunca strings concatenados en
  funciones con `execute format(...)` sin `%L`/`%I`.

```javascript
import { z } from 'zod';
export const GenerateInput = z.object({
  name: z.string().trim().min(1).max(100),
  email: z.string().email().max(254),
  jobTitle: z.string().trim().max(120),
  experience: z.string().max(5000),
}).strict(); // rechaza campos extra (mass assignment)
```

## Secure headers y CSP

En `vercel.json`. Cambios contra la versión anterior: se quita `X-XSS-Protection`
(MDN lo marca deprecado y puede crear XSS en sitios que estaban bien; la protección
real es CSP), `frame-ancestors` reemplaza a `X-Frame-Options`, e `img-src` deja de
aceptar cualquier `https:`.

```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" },
        { "key": "Permissions-Policy", "value": "camera=(), microphone=(), geolocation=()" },
        { "key": "Strict-Transport-Security", "value": "max-age=63072000; includeSubDomains; preload" },
        { "key": "X-Frame-Options", "value": "DENY" },
        {
          "key": "Content-Security-Policy-Report-Only",
          "value": "default-src 'self'; script-src 'self' https://js.stripe.com; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src 'self' https://fonts.gstatic.com; img-src 'self' data: blob: https://TU_PROYECTO.supabase.co; connect-src 'self' https://TU_PROYECTO.supabase.co wss://TU_PROYECTO.supabase.co https://api.stripe.com; frame-src https://js.stripe.com https://hooks.stripe.com; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'"
        }
      ]
    }
  ]
}
```

- Arranca con `Content-Security-Policy-Report-Only`, revisa la consola del navegador
  en todas las pantallas, agrega los dominios que falten (Mercado Pago, analytics,
  PostHog, Sentry) y luego cambia el nombre del header a `Content-Security-Policy`.
- Un build de Vite normalmente no necesita `'unsafe-inline'` en `script-src`. Si algo
  lo pide, busca el script inline y muévelo a un archivo antes de abrir la puerta.
- `X-Frame-Options` se queda solo para navegadores viejos; manda `frame-ancestors`.
- HSTS con `preload` es difícil de revertir: ponlo cuando todos los subdominios ya
  sirvan HTTPS.

## CORS y CSRF

```javascript
const ALLOWED_ORIGINS = new Set([
  'https://tudominio.com',
  'https://www.tudominio.com',
  ...(process.env.VERCEL_ENV !== 'production' ? ['http://localhost:5173'] : []),
]);

export function corsHeaders(request) {
  const origin = request.headers.get('origin');
  return ALLOWED_ORIGINS.has(origin)
    ? { 'Access-Control-Allow-Origin': origin, 'Vary': 'Origin',
        'Access-Control-Allow-Methods': 'POST, OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type, Authorization' }
    : {};
}
```

- CORS no es autenticación: curl y los scripts lo ignoran. Solo evita que otro sitio
  lea tus respuestas desde un navegador.
- CSRF: si el token viaja en `Authorization: Bearer` (lo normal con supabase-js en
  SPA), un sitio ajeno no lo puede mandar y CSRF no aplica. Si usas cookies de sesión
  (SSR), necesitas `SameSite=Lax` o `Strict` y revisar el header `Origin` en todo
  POST/PUT/DELETE.
- Nunca `Access-Control-Allow-Origin: *` junto con credenciales.

## Webhooks (A08)

```javascript
// api/stripe-webhook.js — runtime Node
import Stripe from 'stripe';
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);

export async function POST(request) {
  const raw = await request.text(); // cuerpo crudo: si lo parseas antes, la firma falla
  let event;
  try {
    event = stripe.webhooks.constructEvent(raw, request.headers.get('stripe-signature'),
      process.env.STRIPE_WEBHOOK_SECRET);
  } catch {
    return new Response('Firma inválida', { status: 400 });
  }
  // Idempotencia: tabla con event.id UNIQUE; si ya existe, responde 200 y no acredites otra vez.
  // El monto y el usuario salen del objeto de Stripe/MP (o de tu BD), nunca del metadata que mandó el cliente sin validar.
  return new Response('ok');
}
```

Mercado Pago: valida el header `x-signature` con tu secreto y vuelve a consultar el
pago a su API antes de acreditar. Código completo en la skill `pagos-mx`.

## Bots y abuso

- Honeypot: el campo oculto se revisa **en el servidor**. Revisarlo solo en React no
  sirve, el bot llama a tu API directo.
- Rate limit de signup por IP y verificación de email antes de dar créditos gratis.
- Vercel BotID para endpoints caros (signup, generación, checkout); revisa la guía
  "Deploying and testing BotID" de Vercel para instalarlo y probarlo.
- Créditos de bienvenida: uno por email verificado, y vigila dominios desechables.

## Supply chain de npm (A03)

En 2026 los gusanos de npm (familia Shai-Hulud: ChainDrop en más de 400 paquetes en
agosto, SANDWORM_MODE) roban tokens de npm, GitHub y nube, se republican solos en los
paquetes de la víctima y además escriben en `.claude/settings.json`, configs de MCP
de Claude Desktop/Cursor/VS Code y hooks de git para volver a correr. Instalar un
paquete puede comprometer tu equipo y tu agente.

`.npmrc` del proyecto (y de tu usuario):

```ini
ignore-scripts=true
min-release-age=3
allow-git=none
```

- `min-release-age` (días): no instala versiones publicadas hace menos de N días; la
  mayoría de las versiones maliciosas se detectan y retiran en ese lapso. Requiere un
  npm reciente: si tu npm no la reconoce, actualízalo.
- `ignore-scripts=true` bloquea `preinstall`/`postinstall`, que es por donde entran
  estos gusanos. Si un paquete de verdad necesita su script, revísalo y córrelo a mano.
- En CI: `npm ci` con lockfile, nada de `npm install` suelto; acciones de GitHub
  fijadas por SHA; tokens de npm con el mínimo scope; `npm audit signatures`.
- Si instalaste una versión comprometida: limpia caché de npm, rota credenciales
  desde una máquina limpia, revisa `~/.claude/settings.json`, configs MCP y
  `git config --global --get init.templateDir` (debe salir vacío si tú no lo pusiste).
- React2Shell (CVE-2025-55182, dic 2025, RCE sin auth) afecta React Server Components
  (`react-server-dom-*` 19.0 a 19.2.0, Next.js, `@vitejs/plugin-rsc`). Una SPA de Vite
  sin RSC queda fuera de ese alcance, pero si algún día usas Next.js o RSC, verifica
  versiones corregidas (19.0.1, 19.1.2, 19.2.1 o mayores).

## Errores y logs (A09, A10)

- Al cliente, mensajes genéricos; el stack y los detalles van a tus logs (Sentry o
  logs de Vercel). Nunca devuelvas el error crudo de Supabase o de Anthropic.
- Falla cerrado: si el rate limit, la verificación de token o el chequeo de créditos
  truenan, la respuesta es error, no "dejar pasar".
- Loguea: logins fallidos, 429, webhooks con firma inválida, consumos de crédito y
  cambios de rol. Sin contraseñas, tokens ni CVs completos en los logs.

## Checklist pre-launch

**Llaves (A02, A04)**
- [ ] Migrado a `sb_publishable_` / `sb_secret_` y legacy desactivadas (antes de fin de 2026)
- [ ] Ninguna llave privada en el frontend ni en `VITE_*`
- [ ] Llave de Claude con límite de gasto y una por producto
- [ ] `.env*` en `.gitignore` y escaneo de secretos activo

**Endpoints (A01, A06)**
- [ ] Rate limit persistente (WAF o función atómica), no un `Map` en memoria
- [ ] Crédito reservado de forma atómica antes de llamar al modelo, con reembolso si falla
- [ ] Validación con schema en el servidor y `.strict()`
- [ ] `user.id` sale del token, nunca del body
- [ ] `model`, `max_tokens`, `system` y `tools` fijos en el servidor

**Datos (A01)**
- [ ] RLS activo en todas las tablas; Security Advisor de Supabase sin alertas
- [ ] Funciones `SECURITY DEFINER` con `set search_path = ''` y `revoke execute` a `anon`/`authenticated` si son solo del servidor
- [ ] Webhooks con firma verificada e idempotencia por `event.id`

**Headers (A02)**
- [ ] CSP probada en Report-Only y luego activa, sin `'unsafe-inline'` en scripts
- [ ] Sin `X-XSS-Protection`; `frame-ancestors 'none'`; HSTS
- [ ] CORS con lista de orígenes, nunca `*` con credenciales

**Supply chain (A03)**
- [ ] `.npmrc` con `ignore-scripts`, `min-release-age` y `allow-git`
- [ ] `npm ci` en CI, acciones fijadas por SHA, lockfile en el repo

**Auth (A07)**
- [ ] Rate limit en login y signup, verificación de email antes de créditos
- [ ] Contraseña mínima de 8 caracteres y protección de contraseñas filtradas si tu plan de Supabase la tiene

**IA**
- [ ] Si el endpoint mete contenido de terceros a un LLM con tools o datos: tiene defensas contra prompt injection
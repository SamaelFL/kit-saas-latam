---
name: supabase-rls
description: Autenticación y Row Level Security en Supabase para un SaaS con React y Vercel. Llaves nuevas (publishable y secret), políticas por tabla, funciones SECURITY DEFINER sin hoyos, acceso desde API routes, Storage y cómo depurar una política que bloquea o que deja ver de más. Úsala al crear tablas, al tocar login u OAuth, o cuando alguien diga que una query no devuelve nada o que un usuario ve datos de otro.
---

# Supabase: Auth y RLS

Con Supabase el frontend habla directo con la base de datos. Lo único que separa
los datos de un usuario de los de otro es RLS. Una tabla sin RLS se puede leer
entera con la llave pública.

## Las llaves

| Llave | Reemplaza a | Dónde va | Qué puede |
|---|---|---|---|
| `sb_publishable_...` | `anon` | Frontend; puede ser pública | Lo que permitan las políticas de `anon` y `authenticated` |
| `sb_secret_...` | `service_role` | Solo servidor (API routes, Edge Functions, n8n) | Todo; se salta RLS |

- Supabase está retirando las llaves legacy `anon` y `service_role`. En proyectos
  nuevos usa las nuevas desde el inicio; en los viejos, migra y luego desactiva las
  legacy en Settings → API Keys.
- Las llaves nuevas van en el header `apikey`, no en `Authorization`. `Authorization:
  Bearer` es solo para el JWT del usuario.
- Nunca una llave secreta en el frontend ni en variables `VITE_*` o `NEXT_PUBLIC_*`:
  terminan en el bundle.

## Auth en el frontend

```js
import { createClient } from '@supabase/supabase-js';
export const supabase = createClient(
  import.meta.env.VITE_SUPABASE_URL,
  import.meta.env.VITE_SUPABASE_PUBLISHABLE_KEY,
);

// Registro e inicio de sesión
await supabase.auth.signUp({ email, password });
await supabase.auth.signInWithPassword({ email, password });
await supabase.auth.signInWithOAuth({
  provider: 'google',
  options: { redirectTo: `${window.location.origin}/auth/callback` },
});
await supabase.auth.signOut();

// Cambios de sesión en React
useEffect(() => {
  const { data: { subscription } } = supabase.auth.onAuthStateChange((_e, session) => {
    setUser(session?.user ?? null);
  });
  return () => subscription.unsubscribe();
}, []);
```

Una ruta protegida en React es experiencia de usuario, no seguridad: quien llama a
la API directo se la salta. La seguridad vive en RLS y en el servidor.

## RLS: los patrones

Activa RLS al crear cada tabla, no después:

```sql
alter table public.documentos enable row level security;
```

### Cada quien lo suyo

```sql
create policy "leer los míos" on public.documentos
  for select to authenticated using ((select auth.uid()) = user_id);

create policy "crear como yo" on public.documentos
  for insert to authenticated with check ((select auth.uid()) = user_id);

create policy "editar los míos" on public.documentos
  for update to authenticated
  using ((select auth.uid()) = user_id)
  with check ((select auth.uid()) = user_id);   -- sin esto, se puede "regalar" la fila a otro

create policy "borrar los míos" on public.documentos
  for delete to authenticated using ((select auth.uid()) = user_id);

create index on public.documentos (user_id);
```

- `(select auth.uid())` en lugar de `auth.uid()` hace que Postgres lo evalúe una vez
  por consulta y no una por fila; en tablas grandes se nota mucho.
- `to authenticated` evita que la política se evalúe para visitantes sin sesión.
- El índice en `user_id` es lo que hace rápida la política.

### Datos que el usuario ve pero no toca

Saldo de créditos, pagos, plan contratado: solo `select`. Los cambios los hace el
servidor con la llave secreta o una función revocada.

```sql
create policy "ver mi saldo" on public.credits
  for select to authenticated using ((select auth.uid()) = user_id);
-- sin políticas de insert, update ni delete: desde el cliente no se puede escribir
```

Un error típico es `for all using (auth.uid() = user_id)` en una tabla de saldo: deja
al usuario editar su propio saldo desde la consola del navegador.

### Catálogo público de solo lectura

```sql
create policy "lectura pública" on public.planes for select to anon, authenticated using (true);
```

### Roles de administrador

No pongas el rol en `user_metadata`: el usuario lo puede editar. Usa una tabla que
solo el servidor escribe:

```sql
create table public.admins (user_id uuid primary key references auth.users);
alter table public.admins enable row level security; -- sin políticas: nadie desde el cliente

create or replace function public.es_admin() returns boolean
language sql stable security definer set search_path = '' as $$
  select exists (select 1 from public.admins where user_id = (select auth.uid()));
$$;

create policy "admin ve todo" on public.documentos
  for select to authenticated using ((select public.es_admin()));
```

## Funciones SECURITY DEFINER

Corren con los permisos de quien las creó y se saltan RLS. Si viven en `public`, el
cliente las puede llamar con `supabase.rpc()`. Reglas:

1. Siempre `set search_path = ''` y nombres con esquema (`public.tabla`).
2. Si solo debe llamarlas tu servidor:
   `revoke execute on function public.f(...) from public, anon, authenticated;`
3. Si el cliente sí las llama, que usen `auth.uid()` adentro y **nunca** reciban el
   id del usuario como parámetro: si lo reciben, cualquiera actúa por otro.
4. Revisa el Security Advisor del dashboard después de cada migración.

## Desde una API route (Vercel)

```js
import { createClient } from '@supabase/supabase-js';

// Cliente de servidor: se salta RLS. Solo para lo que el usuario no puede hacer solo
// (acreditar un pago, consumir un crédito).
export const admin = createClient(process.env.SUPABASE_URL, process.env.SUPABASE_SECRET_KEY, {
  auth: { persistSession: false, autoRefreshToken: false },
});

// Cliente con el JWT del usuario: RLS aplica como si fuera él.
export function clienteDelUsuario(jwt) {
  return createClient(process.env.SUPABASE_URL, process.env.SUPABASE_PUBLISHABLE_KEY, {
    global: { headers: { Authorization: `Bearer ${jwt}` } },
    auth: { persistSession: false, autoRefreshToken: false },
  });
}
```

Para leer o escribir datos del propio usuario, prefiere el cliente con su JWT: si te
equivocas en un filtro, RLS te salva. El id del usuario sale del token validado
(`admin.auth.getUser(jwt)`), nunca del body.

## Storage

Los buckets también usan RLS, sobre `storage.objects`. Patrón: una carpeta por usuario.

```sql
create policy "subir a mi carpeta" on storage.objects
  for insert to authenticated
  with check (bucket_id = 'archivos' and (storage.foldername(name))[1] = (select auth.uid())::text);

create policy "leer mi carpeta" on storage.objects
  for select to authenticated
  using (bucket_id = 'archivos' and (storage.foldername(name))[1] = (select auth.uid())::text);
```

Sube con la ruta `${user.id}/archivo.pdf`. Los buckets privados sirven archivos con
URLs firmadas de vida corta.

## Depurar

**La consulta regresa `[]` sin error.** RLS está filtrando. Revisa:
1. ¿Hay política de `select` para ese rol? Dashboard → Authentication → Policies.
2. ¿Hay sesión? `await supabase.auth.getUser()`.
3. ¿`user_id` de las filas coincide con el usuario?

**"new row violates row-level security policy".** Falta la política de `insert` o el
`with check` no se cumple, casi siempre porque el `user_id` del insert no es el del
usuario con sesión.

**Probar una política como un usuario concreto** en el SQL Editor:

```sql
begin;
set local role authenticated;
select set_config('request.jwt.claims', '{"sub":"UUID-DEL-USUARIO","role":"authenticated"}', true);
select * from public.documentos;   -- solo debe salir lo de ese usuario
rollback;
```

**Probar que nadie ve lo de otro**: con dos cuentas de prueba, intenta leer, editar y
borrar las filas de la otra desde la consola del navegador. Debe fallar todo.

## Checklist

- [ ] RLS activo en todas las tablas de `public` (Security Advisor sin alertas)
- [ ] Políticas con `to authenticated` y `(select auth.uid())`, índice en `user_id`
- [ ] `update` con `using` y `with check`
- [ ] Saldos, pagos y planes: el usuario solo lee
- [ ] Roles de admin fuera de `user_metadata`
- [ ] Funciones `security definer` con `search_path = ''` y revocadas si son del servidor
- [ ] Llaves nuevas; la secreta solo en el servidor
- [ ] Políticas de Storage por carpeta de usuario
- [ ] Prueba con dos cuentas de que una no ve ni toca lo de la otra

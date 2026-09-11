# Racha de Flexiones

Contador diario de flexiones: registro rápido por series, meta diaria opcional, racha de días consecutivos, calendario mensual (con récord marcado) e historial en gráfico por semana/mes/año.

Es una única página HTML sin dependencias de build. Cada persona se registra con correo y contraseña; su progreso se guarda en Supabase (Postgres + Auth), así que no se pierde al cambiar de móvil o navegador, y cada cuenta ve solo su propio historial.

## Configurar Supabase (una sola vez)

1. Crea una cuenta gratuita en [supabase.com](https://supabase.com) y un proyecto nuevo.
2. En **SQL Editor**, ejecuta este script para crear las tablas y las políticas de seguridad:

   ```sql
   create table public.profiles (
     id uuid primary key references auth.users(id) on delete cascade,
     display_name text not null,
     created_at timestamptz not null default now()
   );
   alter table public.profiles enable row level security;
   create policy "read own profile" on public.profiles for select using (auth.uid() = id);
   create policy "insert own profile" on public.profiles for insert with check (auth.uid() = id);
   create policy "update own profile" on public.profiles for update using (auth.uid() = id);

   create table public.settings (
     user_id uuid primary key references auth.users(id) on delete cascade,
     daily_goal integer not null default 50,
     goal_enabled boolean not null default false
   );
   alter table public.settings enable row level security;
   create policy "manage own settings" on public.settings for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

   create table public.days (
     user_id uuid not null references auth.users(id) on delete cascade,
     date date not null,
     total integer not null default 0,
     sets jsonb not null default '[]'::jsonb,
     updated_at timestamptz not null default now(),
     primary key (user_id, date)
   );
   alter table public.days enable row level security;
   create policy "manage own days" on public.days for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

   create or replace function public.handle_new_user()
   returns trigger
   language plpgsql
   security definer set search_path = public
   as $$
   begin
     insert into public.profiles (id, display_name)
     values (new.id, coalesce(new.raw_user_meta_data->>'display_name', split_part(new.email, '@', 1)));
     insert into public.settings (user_id) values (new.id);
     return new;
   end;
   $$;

   create trigger on_auth_user_created
     after insert on auth.users
     for each row execute procedure public.handle_new_user();
   ```

3. En **Project Settings → API**, copia la **Project URL** y la clave **anon public**.
4. Abre `index.html` y reemplaza las dos constantes al inicio del `<script>`:

   ```js
   var SUPABASE_URL = 'https://TU-PROYECTO.supabase.co';
   var SUPABASE_ANON_KEY = 'TU-CLAVE-ANON-PUBLIC';
   ```

   (La clave `anon public` está pensada para usarse en el navegador; no es secreta, pero la seguridad real la dan las políticas RLS del script SQL de arriba, que hacen que cada persona solo pueda leer y escribir sus propios datos.)
5. Opcional: en **Authentication → Settings**, puedes desactivar "Confirm email" si quieres que cada persona pueda entrar justo después de crear su cuenta, sin confirmar el correo primero.
6. Sube el cambio (`git add`, `git commit`, `git push`) para que la versión publicada en GitHub Pages quede conectada a tu proyecto de Supabase.

## Uso en el móvil

1. Abre la URL de GitHub Pages del repositorio en el navegador del móvil.
2. Crea tu cuenta (nombre, correo y contraseña) o inicia sesión si ya la tienes.
3. En el menú del navegador, elige "Añadir a pantalla de inicio" (Safari/Chrome) para tener un acceso directo como si fuera una app.

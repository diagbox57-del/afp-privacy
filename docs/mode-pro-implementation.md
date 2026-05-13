# AFP Garage — Implémentation « Mode Pro »

> **À donner à la session Claude Code locale qui ouvrira le dossier `afp garage` du bureau.**
> Ce document est la source unique de vérité pour ajouter la fonctionnalité Mode Pro à l'app AFP Garage avant resoumission à l'App Store (rejet Apple directive 4.3(a) — Spam).

---

## 0. Contexte

**Pourquoi cette fonctionnalité ?**
Apple a rejeté la v1.0 d'AFP Garage sous la directive **4.3(a) (Spam / similarité)**. La stratégie de réponse est d'ajouter une fonctionnalité **vraiment originale** qu'aucune app de carnet d'entretien automobile ne propose : permettre au propriétaire de **partager temporairement** l'historique d'entretien d'un véhicule avec un professionnel (garagiste, expert, acheteur lors d'une revente…) via un **lien sécurisé + PIN à 6 chiffres**. Le pro peut **consulter** l'historique et **ajouter une intervention** signée et horodatée. Tout est révocable et auditable.

**Politique de confidentialité** : déjà mise à jour dans le repo `diagbox57-del/afp-privacy`, branche `claude/review-afp-garage-feedback-S1ele`. Voir notamment la section 4.5.

**Périmètre v1.1** : Mode Pro uniquement. Le « stéthoscope sonore » est repoussé à v1.2.

---

## 1. Architecture cible

```
┌─────────────────────┐         ┌──────────────────────┐
│  App RN (utilisateur)│         │  Site Pro web         │
│  - Bouton "Partager" │         │  pro.afpgarage.app    │
│  - Modal QR + PIN    │         │  (React + Vite)       │
│  - Liste accès actifs│         │  - Saisie PIN         │
│  - Journal d'audit   │         │  - Dashboard véhicule │
│  - Notif locales     │         │  - Ajout intervention │
└──────────┬───────────┘         └───────────┬──────────┘
           │                                 │
           │   Supabase Auth (utilisateur)   │   (anonyme, JWT custom)
           ▼                                 ▼
┌──────────────────────────────────────────────────────┐
│              Edge Function `mode-pro`                 │
│  /validate · /vehicle · /entry · /revoke              │
└──────────┬───────────────────────────────────┬───────┘
           │           service_role            │
           ▼                                   ▼
┌─────────────────────────────────────────────────────┐
│   Supabase PostgreSQL (Frankfurt, eu-central-1)     │
│   vehicle_shares · pro_access_log · maintenance_*    │
│   + RLS strictes + trigger immuabilité + 2 RPC      │
└─────────────────────────────────────────────────────┘
```

---

## 2. Migration SQL Supabase

À placer dans `supabase/migrations/20260513000001_mode_pro.sql` (adapter le timestamp).

```sql
-- =============================================================================
-- AFP Garage — Mode Pro
-- Partage temporaire d'un véhicule avec un professionnel
-- =============================================================================
create extension if not exists pgcrypto;

-- 1. Liens de partage ---------------------------------------------------------
create table public.vehicle_shares (
  id               uuid primary key default gen_random_uuid(),
  vehicle_id       uuid not null references public.vehicles(id) on delete cascade,
  owner_id         uuid not null references auth.users(id)      on delete cascade,

  share_token      text not null unique,            -- 32 octets aléatoires, base64url
  pin_hash         text not null,                   -- crypt(pin, gen_salt('bf', 10))

  pro_display_name text,                            -- saisi par le pro à la 1ère ouverture
  expires_at       timestamptz not null
                   default (now() + interval '24 hours')
                   check (expires_at <= created_at + interval '24 hours'),
  revoked_at       timestamptz,
  first_used_at    timestamptz,
  created_at       timestamptz not null default now()
);

create index vehicle_shares_vehicle_idx on public.vehicle_shares (vehicle_id);
create index vehicle_shares_owner_idx   on public.vehicle_shares (owner_id);
create index vehicle_shares_active_idx
  on public.vehicle_shares (expires_at)
  where revoked_at is null;


-- 2. Journal d'audit ----------------------------------------------------------
create table public.pro_access_log (
  id                uuid primary key default gen_random_uuid(),
  share_id          uuid not null references public.vehicle_shares(id) on delete cascade,
  vehicle_id        uuid not null references public.vehicles(id)       on delete cascade,
  owner_id          uuid not null references auth.users(id)            on delete cascade,
  action            text not null check (action in (
                      'link_opened','pin_validated','pin_invalid',
                      'viewed_vehicle','viewed_history',
                      'entry_added',
                      'revoked_by_owner','expired'
                    )),
  pro_display_name  text,
  target_entry_id   uuid,
  ip_address        inet,
  user_agent        text,
  created_at        timestamptz not null default now()
);

create index pro_access_log_share_idx   on public.pro_access_log (share_id);
create index pro_access_log_owner_idx   on public.pro_access_log (owner_id);
create index pro_access_log_created_idx on public.pro_access_log (created_at desc);


-- 3. Marquage des entrées ajoutées par un pro ---------------------------------
alter table public.maintenance_entries
  add column if not exists added_by_pro_share_id uuid
    references public.vehicle_shares(id) on delete set null,
  add column if not exists added_by_pro_name text;

create index if not exists maintenance_entries_by_pro_idx
  on public.maintenance_entries (added_by_pro_share_id)
  where added_by_pro_share_id is not null;


-- 4. RLS — vehicle_shares -----------------------------------------------------
alter table public.vehicle_shares enable row level security;

create policy "owners_select_shares"
  on public.vehicle_shares for select
  using (auth.uid() = owner_id);

create policy "owners_create_share"
  on public.vehicle_shares for insert
  with check (
    auth.uid() = owner_id
    and exists (
      select 1 from public.vehicles
      where id = vehicle_id and user_id = auth.uid()
    )
  );

create policy "owners_revoke_share"
  on public.vehicle_shares for update
  using (auth.uid() = owner_id);


-- 5. RLS — pro_access_log -----------------------------------------------------
alter table public.pro_access_log enable row level security;

create policy "owners_select_log"
  on public.pro_access_log for select
  using (auth.uid() = owner_id);

-- Aucune policy INSERT/UPDATE/DELETE : seul service_role écrit dans le log.


-- 6. Trigger d'immuabilité ----------------------------------------------------
create or replace function public.vehicle_shares_block_mutations()
returns trigger language plpgsql as $$
begin
  if old.id          is distinct from new.id
  or old.vehicle_id  is distinct from new.vehicle_id
  or old.owner_id    is distinct from new.owner_id
  or old.share_token is distinct from new.share_token
  or old.pin_hash    is distinct from new.pin_hash
  or old.expires_at  is distinct from new.expires_at
  or old.created_at  is distinct from new.created_at then
    raise exception 'Immutable column changed on vehicle_shares';
  end if;

  if old.revoked_at is not null
     and new.revoked_at is distinct from old.revoked_at then
    raise exception 'Share already revoked';
  end if;

  if old.first_used_at is not null
     and new.first_used_at is distinct from old.first_used_at then
    raise exception 'first_used_at already set';
  end if;

  if old.pro_display_name is not null
     and new.pro_display_name is distinct from old.pro_display_name then
    raise exception 'pro_display_name already set';
  end if;

  return new;
end;
$$;

create trigger vehicle_shares_block_mutations_trg
  before update on public.vehicle_shares
  for each row execute function public.vehicle_shares_block_mutations();


-- 7. RPC : créer un partage (appelé par le propriétaire) ----------------------
create or replace function public.create_pro_share(
  p_vehicle_id uuid,
  p_share_token text,
  p_pin text
) returns uuid
language plpgsql security definer
set search_path = public, extensions
as $$
declare
  v_id uuid;
begin
  if auth.uid() is null then
    raise exception 'unauthorized';
  end if;

  if not exists (
    select 1 from public.vehicles
    where id = p_vehicle_id and user_id = auth.uid()
  ) then
    raise exception 'not_owner';
  end if;

  if length(p_share_token) < 32 then
    raise exception 'invalid_token';
  end if;

  if p_pin !~ '^\d{6}$' then
    raise exception 'invalid_pin';
  end if;

  insert into public.vehicle_shares (vehicle_id, owner_id, share_token, pin_hash)
  values (
    p_vehicle_id, auth.uid(), p_share_token,
    crypt(p_pin, gen_salt('bf', 10))
  )
  returning id into v_id;

  return v_id;
end;
$$;

revoke all on function public.create_pro_share(uuid, text, text) from public;
grant execute on function public.create_pro_share(uuid, text, text) to authenticated;


-- 8. RPC : valider PIN + premier usage (appelé par l'edge function) -----------
create or replace function public.consume_pro_share(
  p_token text,
  p_pin text,
  p_pro_name text
) returns table (
  ok boolean,
  share_id uuid,
  vehicle_id uuid,
  owner_id uuid,
  pro_display_name text,
  was_first_use boolean
)
language plpgsql security definer
set search_path = public, extensions
as $$
declare
  v_share public.vehicle_shares%rowtype;
  v_name  text;
begin
  select * into v_share
  from public.vehicle_shares
  where share_token = p_token
  for update;

  if not found
     or v_share.revoked_at is not null
     or v_share.expires_at <= now() then
    return query select false, null::uuid, null::uuid, null::uuid, null::text, false;
    return;
  end if;

  if v_share.pin_hash <> crypt(p_pin, v_share.pin_hash) then
    return query select false, v_share.id, v_share.vehicle_id, v_share.owner_id,
                        v_share.pro_display_name, false;
    return;
  end if;

  if v_share.first_used_at is null then
    v_name := coalesce(nullif(trim(p_pro_name), ''), 'Professionnel non identifié');
    update public.vehicle_shares
       set first_used_at = now(), pro_display_name = v_name
     where id = v_share.id;
    return query select true, v_share.id, v_share.vehicle_id, v_share.owner_id, v_name, true;
  else
    return query select true, v_share.id, v_share.vehicle_id, v_share.owner_id,
                        v_share.pro_display_name, false;
  end if;
end;
$$;

revoke all on function public.consume_pro_share(text, text, text) from public;
grant execute on function public.consume_pro_share(text, text, text) to service_role;
```

**Appliquer :**
```bash
supabase db push                       # depuis le dossier de l'app
# ou via le SQL Editor du dashboard Supabase pour le staging
```

---

## 3. Edge Function `mode-pro`

Créer `supabase/functions/mode-pro/index.ts` :

```typescript
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";
import { create, verify, getNumericDate } from "https://deno.land/x/djwt@v3.0.2/mod.ts";

const SUPABASE_URL   = Deno.env.get("SUPABASE_URL")!;
const SERVICE_KEY    = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
const ANON_KEY       = Deno.env.get("SUPABASE_ANON_KEY")!;
const JWT_SECRET     = Deno.env.get("MODE_PRO_JWT_SECRET")!;
const JWT_TTL_SEC    = 15 * 60;
const RATE_WINDOW_M  = 15;
const RATE_MAX_FAILS = 5;

const sb = createClient(SUPABASE_URL, SERVICE_KEY, {
  auth: { persistSession: false, autoRefreshToken: false },
});

const jwtKey = await crypto.subtle.importKey(
  "raw", new TextEncoder().encode(JWT_SECRET),
  { name: "HMAC", hash: "SHA-256" }, false, ["sign", "verify"],
);

const CORS = {
  "Access-Control-Allow-Origin":  "*",
  "Access-Control-Allow-Headers": "authorization, content-type",
  "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
};
const J = (b: unknown, s = 200) =>
  new Response(JSON.stringify(b), { status: s, headers: { "content-type": "application/json", ...CORS } });

const ipOf = (r: Request) => r.headers.get("x-forwarded-for")?.split(",")[0]?.trim() ?? null;
const uaOf = (r: Request) => r.headers.get("user-agent");

type Claims = { share_id: string; vehicle_id: string; owner_id: string; exp: number };

async function logAccess(o: {
  share_id: string; vehicle_id: string; owner_id: string;
  action: string;
  pro_display_name?: string | null;
  target_entry_id?: string | null;
  ip?: string | null; ua?: string | null;
}) {
  await sb.from("pro_access_log").insert({
    share_id: o.share_id, vehicle_id: o.vehicle_id, owner_id: o.owner_id,
    action: o.action,
    pro_display_name: o.pro_display_name ?? null,
    target_entry_id:  o.target_entry_id  ?? null,
    ip_address:       o.ip               ?? null,
    user_agent:       o.ua               ?? null,
  });
}

async function readProJWT(req: Request): Promise<Claims | null> {
  const auth = req.headers.get("authorization");
  if (!auth?.startsWith("Bearer ")) return null;
  try {
    const c = await verify(auth.slice(7), jwtKey) as Claims;
    if (!c.share_id || !c.vehicle_id || !c.owner_id) return null;
    const { data } = await sb.from("vehicle_shares")
      .select("revoked_at, expires_at").eq("id", c.share_id).maybeSingle();
    if (!data || data.revoked_at || new Date(data.expires_at).getTime() <= Date.now()) return null;
    return c;
  } catch { return null; }
}

async function handleValidate(req: Request) {
  const { token, pin, pro_name } = await req.json().catch(() => ({}));
  if (typeof token !== "string" || !/^\d{6}$/.test(pin ?? "")) {
    return J({ error: "invalid_input" }, 400);
  }

  const { data: head } = await sb.from("vehicle_shares")
    .select("id").eq("share_token", token).maybeSingle();
  if (!head) return J({ error: "invalid_link" }, 404);

  const since = new Date(Date.now() - RATE_WINDOW_M * 60_000).toISOString();
  const { count } = await sb.from("pro_access_log")
    .select("*", { head: true, count: "exact" })
    .eq("share_id", head.id).eq("action", "pin_invalid")
    .gte("created_at", since);
  if ((count ?? 0) >= RATE_MAX_FAILS) {
    return J({ error: "too_many_attempts", retry_after_minutes: RATE_WINDOW_M }, 429);
  }

  const { data: rows, error } = await sb.rpc("consume_pro_share", {
    p_token: token, p_pin: pin, p_pro_name: pro_name ?? "",
  });
  if (error) return J({ error: "server_error" }, 500);
  const r = (rows ?? [])[0];

  if (!r?.ok) {
    if (r?.share_id) {
      await logAccess({
        share_id: r.share_id, vehicle_id: r.vehicle_id, owner_id: r.owner_id,
        action: "pin_invalid", ip: ipOf(req), ua: uaOf(req),
      });
    }
    return J({ error: "invalid_pin" }, 401);
  }

  if (r.was_first_use) {
    await logAccess({
      share_id: r.share_id, vehicle_id: r.vehicle_id, owner_id: r.owner_id,
      action: "link_opened", pro_display_name: r.pro_display_name,
      ip: ipOf(req), ua: uaOf(req),
    });
  }
  await logAccess({
    share_id: r.share_id, vehicle_id: r.vehicle_id, owner_id: r.owner_id,
    action: "pin_validated", pro_display_name: r.pro_display_name,
    ip: ipOf(req), ua: uaOf(req),
  });

  const jwt = await create(
    { alg: "HS256", typ: "JWT" },
    { share_id: r.share_id, vehicle_id: r.vehicle_id, owner_id: r.owner_id,
      exp: getNumericDate(JWT_TTL_SEC) },
    jwtKey,
  );
  return J({ jwt, pro_display_name: r.pro_display_name, expires_in: JWT_TTL_SEC });
}

async function handleGetVehicle(req: Request) {
  const c = await readProJWT(req);
  if (!c) return J({ error: "unauthorized" }, 401);

  const [{ data: vehicle }, { data: history }] = await Promise.all([
    sb.from("vehicles").select("*").eq("id", c.vehicle_id).single(),
    sb.from("maintenance_entries")
      .select("*").eq("vehicle_id", c.vehicle_id)
      .order("date", { ascending: false }),
  ]);
  if (!vehicle) return J({ error: "not_found" }, 404);

  await logAccess({
    share_id: c.share_id, vehicle_id: c.vehicle_id, owner_id: c.owner_id,
    action: "viewed_vehicle", ip: ipOf(req), ua: uaOf(req),
  });
  return J({ vehicle, history: history ?? [] });
}

async function handleAddEntry(req: Request) {
  const c = await readProJWT(req);
  if (!c) return J({ error: "unauthorized" }, 401);

  const b = await req.json().catch(() => ({}));
  if (!b.date || !b.type) return J({ error: "invalid_input" }, 400);

  const { data: share } = await sb.from("vehicle_shares")
    .select("pro_display_name").eq("id", c.share_id).single();

  const payload = {
    user_id:               c.owner_id,
    vehicle_id:            c.vehicle_id,
    date:                  b.date,
    type:                  b.type,
    title:                 b.title             ?? null,
    mileage_km:            b.mileage_km        ?? null,
    cost_eur:              b.cost_eur          ?? null,
    garage_name:           b.garage_name       ?? share?.pro_display_name ?? null,
    notes:                 b.notes             ?? null,
    invoice_image_url:     b.invoice_image_url ?? null,
    added_by_pro_share_id: c.share_id,
    added_by_pro_name:     share?.pro_display_name ?? null,
  };

  const { data: entry, error } = await sb.from("maintenance_entries")
    .insert(payload).select().single();
  if (error) return J({ error: "insert_failed", detail: error.message }, 500);

  await logAccess({
    share_id: c.share_id, vehicle_id: c.vehicle_id, owner_id: c.owner_id,
    action: "entry_added", target_entry_id: entry.id,
    pro_display_name: payload.added_by_pro_name,
    ip: ipOf(req), ua: uaOf(req),
  });
  return J({ entry });
}

async function handleRevoke(req: Request) {
  const auth = req.headers.get("authorization");
  if (!auth) return J({ error: "unauthorized" }, 401);

  const userClient = createClient(SUPABASE_URL, ANON_KEY, {
    global: { headers: { Authorization: auth } },
    auth: { persistSession: false, autoRefreshToken: false },
  });
  const { data: u } = await userClient.auth.getUser();
  if (!u.user) return J({ error: "unauthorized" }, 401);

  const { share_id } = await req.json().catch(() => ({}));
  if (typeof share_id !== "string") return J({ error: "invalid_input" }, 400);

  const { data: share } = await userClient.from("vehicle_shares")
    .update({ revoked_at: new Date().toISOString() })
    .eq("id", share_id).is("revoked_at", null)
    .select("id, vehicle_id, owner_id").maybeSingle();
  if (!share) return J({ error: "not_found_or_already_revoked" }, 404);

  await logAccess({
    share_id: share.id, vehicle_id: share.vehicle_id, owner_id: share.owner_id,
    action: "revoked_by_owner", ip: ipOf(req), ua: uaOf(req),
  });
  return J({ ok: true });
}

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") return new Response(null, { headers: CORS });
  const path = new URL(req.url).pathname.replace(/^\/mode-pro/, "");
  try {
    if (req.method === "POST" && path === "/validate") return await handleValidate(req);
    if (req.method === "GET"  && path === "/vehicle")  return await handleGetVehicle(req);
    if (req.method === "POST" && path === "/entry")    return await handleAddEntry(req);
    if (req.method === "POST" && path === "/revoke")   return await handleRevoke(req);
  } catch (e) {
    console.error(e);
    return J({ error: "server_error" }, 500);
  }
  return J({ error: "not_found" }, 404);
});
```

**Déployer :**
```bash
supabase secrets set MODE_PRO_JWT_SECRET="$(openssl rand -base64 48)"
supabase functions deploy mode-pro --no-verify-jwt
```

`--no-verify-jwt` est nécessaire car les routes pro ne s'authentifient pas via Supabase Auth — la sécurité est dans le code de la fonction.

---

## 4. App React Native — 4 écrans côté propriétaire

### 4.1 Dépendances à ajouter

```bash
npx expo install react-native-qrcode-svg react-native-svg
# expo-clipboard, expo-sharing, expo-notifications sont déjà censés être présents
```

### 4.2 Helpers `lib/proShare.ts`

```typescript
import { supabase } from './supabase'; // votre client existant

export type ShareInfo = { id: string; token: string; pin: string };

function randomToken(bytes = 32): string {
  const arr = new Uint8Array(bytes);
  crypto.getRandomValues(arr);
  return btoa(String.fromCharCode(...arr))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

function randomPin6(): string {
  const arr = new Uint32Array(1);
  crypto.getRandomValues(arr);
  return String(100000 + (arr[0] % 900000));
}

export async function createProShare(vehicleId: string): Promise<ShareInfo> {
  const token = randomToken();
  const pin = randomPin6();
  const { data, error } = await supabase.rpc('create_pro_share', {
    p_vehicle_id: vehicleId, p_share_token: token, p_pin: pin,
  });
  if (error) throw error;
  return { id: data as string, token, pin };
}

export function proShareUrl(token: string): string {
  // Adapter au domaine retenu
  return `https://pro.afpgarage.app/?t=${token}`;
}

export async function revokeProShare(shareId: string) {
  const { error } = await supabase
    .from('vehicle_shares')
    .update({ revoked_at: new Date().toISOString() })
    .eq('id', shareId)
    .is('revoked_at', null);
  if (error) throw error;
}

export async function listActiveShares(vehicleId: string) {
  const { data, error } = await supabase
    .from('vehicle_shares')
    .select('*')
    .eq('vehicle_id', vehicleId)
    .is('revoked_at', null)
    .gt('expires_at', new Date().toISOString())
    .order('created_at', { ascending: false });
  if (error) throw error;
  return data ?? [];
}

export async function listAccessLog(vehicleId: string, limit = 100) {
  const { data, error } = await supabase
    .from('pro_access_log')
    .select('*')
    .eq('vehicle_id', vehicleId)
    .order('created_at', { ascending: false })
    .limit(limit);
  if (error) throw error;
  return data ?? [];
}
```

### 4.3 Écran « Partager avec un pro » — `screens/ProShareModal.tsx`

```tsx
import { useEffect, useState } from 'react';
import { View, Text, ScrollView, Pressable, Alert } from 'react-native';
import QRCode from 'react-native-qrcode-svg';
import * as Clipboard from 'expo-clipboard';
import * as Sharing from 'expo-sharing';
import { createProShare, revokeProShare, proShareUrl } from '../lib/proShare';

export function ProShareModal({ vehicleId, vehicleLabel, onClose }: {
  vehicleId: string; vehicleLabel: string; onClose: () => void;
}) {
  const [share, setShare] = useState<{ id: string; token: string; pin: string } | null>(null);
  const [remainingMin, setRemainingMin] = useState(24 * 60);

  useEffect(() => {
    createProShare(vehicleId)
      .then(setShare)
      .catch(err => { Alert.alert('Erreur', err.message); onClose(); });
  }, [vehicleId]);

  useEffect(() => {
    if (!share) return;
    const start = Date.now();
    const t = setInterval(() => {
      const elapsedMin = Math.floor((Date.now() - start) / 60000);
      setRemainingMin(24 * 60 - elapsedMin);
    }, 60000);
    return () => clearInterval(t);
  }, [share]);

  if (!share) return null;

  const url = proShareUrl(share.token);
  const pinPretty = `${share.pin.slice(0, 3)} ${share.pin.slice(3)}`;
  const hours = Math.floor(remainingMin / 60);
  const mins = remainingMin % 60;

  const onShare = async () => {
    const msg = `Voici le lien d'accès à mon carnet d'entretien AFP Garage pour ${vehicleLabel} :\n${url}\nCode PIN : ${pinPretty}\n(Valable 24 h)`;
    if (await Sharing.isAvailableAsync()) {
      await Clipboard.setStringAsync(msg);
      Alert.alert('Lien copié', 'Collez-le dans SMS, WhatsApp ou e-mail.');
    }
  };

  const onCancel = async () => {
    Alert.alert(
      'Annuler le partage ?',
      'L\'accès sera révoqué immédiatement.',
      [
        { text: 'Non', style: 'cancel' },
        { text: 'Oui, annuler', style: 'destructive',
          onPress: async () => { await revokeProShare(share.id); onClose(); } },
      ],
    );
  };

  return (
    <ScrollView contentContainerStyle={{ padding: 24, alignItems: 'center', gap: 16 }}>
      <Text style={{ fontSize: 22, fontWeight: '700', textAlign: 'center' }}>
        Partagez avec votre pro
      </Text>
      <Text style={{ color: '#666', textAlign: 'center' }}>
        Votre garagiste pourra consulter votre carnet et ajouter une intervention.
        L'accès expire dans <Text style={{ fontWeight: '700' }}>{hours} h {mins} min</Text>.
      </Text>

      <View style={{ backgroundColor: '#fff', padding: 16, borderRadius: 16 }}>
        <QRCode value={url} size={220} />
      </View>

      <Text style={{ fontSize: 14, color: '#666' }}>Code PIN à 6 chiffres</Text>
      <Text style={{ fontSize: 42, fontWeight: '800', letterSpacing: 6 }}>
        {pinPretty}
      </Text>
      <Text style={{ color: '#666', textAlign: 'center', fontSize: 13 }}>
        Communiquez ce code à votre garagiste en main propre.
      </Text>

      <Pressable onPress={onShare} style={btnSecondary}>
        <Text style={{ color: '#fff', fontWeight: '600' }}>Copier le lien + PIN</Text>
      </Pressable>

      <Pressable onPress={onCancel} style={btnDanger}>
        <Text style={{ color: '#fff', fontWeight: '600' }}>Annuler le partage</Text>
      </Pressable>
      <Pressable onPress={onClose} style={{ marginTop: 8 }}>
        <Text style={{ color: '#666' }}>Fermer</Text>
      </Pressable>
    </ScrollView>
  );
}

const btnSecondary = { backgroundColor: '#3B82F6', paddingHorizontal: 24, paddingVertical: 12, borderRadius: 12, marginTop: 8 };
const btnDanger    = { backgroundColor: '#DC2626', paddingHorizontal: 24, paddingVertical: 12, borderRadius: 12, marginTop: 8 };
```

À ouvrir depuis l'écran fiche véhicule existant :

```tsx
// Dans VehicleDetailScreen.tsx, ajouter :
<Pressable onPress={() => navigation.navigate('ProShareModal', { vehicleId: vehicle.id, vehicleLabel: vehicle.label })}>
  <Text>👨‍🔧 Partager avec un pro</Text>
</Pressable>
```

### 4.4 Écran « Accès Pro actifs » — `screens/ProAccessListScreen.tsx`

```tsx
import { useEffect, useState, useCallback } from 'react';
import { View, Text, FlatList, Pressable, RefreshControl, Alert } from 'react-native';
import { listActiveShares, revokeProShare } from '../lib/proShare';

export function ProAccessListScreen({ route, navigation }: any) {
  const { vehicleId } = route.params;
  const [shares, setShares] = useState<any[]>([]);
  const [refreshing, setRefreshing] = useState(false);

  const load = useCallback(async () => {
    setRefreshing(true);
    try { setShares(await listActiveShares(vehicleId)); } finally { setRefreshing(false); }
  }, [vehicleId]);

  useEffect(() => { load(); }, [load]);

  const onRevoke = (id: string) => {
    Alert.alert('Révoquer cet accès ?', 'Effet immédiat.', [
      { text: 'Non', style: 'cancel' },
      { text: 'Révoquer', style: 'destructive', onPress: async () => { await revokeProShare(id); load(); } },
    ]);
  };

  return (
    <FlatList
      data={shares}
      keyExtractor={s => s.id}
      refreshControl={<RefreshControl refreshing={refreshing} onRefresh={load} />}
      ListEmptyComponent={<Text style={{ padding: 32, textAlign: 'center', color: '#666' }}>Aucun accès Pro actif.</Text>}
      renderItem={({ item }) => {
        const expMin = Math.floor((new Date(item.expires_at).getTime() - Date.now()) / 60000);
        const h = Math.floor(expMin / 60), m = expMin % 60;
        return (
          <View style={{ padding: 16, borderBottomWidth: 1, borderColor: '#eee' }}>
            <Text style={{ fontWeight: '700' }}>{item.pro_display_name ?? 'En attente d\'ouverture'}</Text>
            <Text style={{ color: '#666' }}>
              {item.first_used_at ? `Ouvert le ${new Date(item.first_used_at).toLocaleString('fr-FR')}` : 'Pas encore ouvert'}
            </Text>
            <Text style={{ color: '#666' }}>Expire dans {h} h {m} min</Text>
            <Pressable onPress={() => onRevoke(item.id)} style={{ marginTop: 8 }}>
              <Text style={{ color: '#DC2626', fontWeight: '600' }}>Révoquer l'accès</Text>
            </Pressable>
          </View>
        );
      }}
    />
  );
}
```

### 4.5 Écran « Journal d'audit » — `screens/ProAccessLogScreen.tsx`

```tsx
import { useEffect, useState, useCallback } from 'react';
import { View, Text, FlatList, RefreshControl } from 'react-native';
import { listAccessLog } from '../lib/proShare';

const ICONS: Record<string, string> = {
  link_opened:      '🔓',
  pin_validated:    '✅',
  pin_invalid:      '⚠️',
  viewed_vehicle:   '👁',
  viewed_history:   '👁',
  entry_added:      '➕',
  revoked_by_owner: '⛔',
  expired:          '⏰',
};
const LABELS: Record<string, string> = {
  link_opened:      'Lien ouvert',
  pin_validated:    'PIN validé',
  pin_invalid:      'PIN invalide',
  viewed_vehicle:   'Véhicule consulté',
  viewed_history:   'Historique consulté',
  entry_added:      'Intervention ajoutée',
  revoked_by_owner: 'Accès révoqué par vous',
  expired:          'Accès expiré',
};

export function ProAccessLogScreen({ route }: any) {
  const { vehicleId } = route.params;
  const [log, setLog] = useState<any[]>([]);
  const [refreshing, setRefreshing] = useState(false);

  const load = useCallback(async () => {
    setRefreshing(true);
    try { setLog(await listAccessLog(vehicleId)); } finally { setRefreshing(false); }
  }, [vehicleId]);

  useEffect(() => { load(); }, [load]);

  return (
    <FlatList
      data={log}
      keyExtractor={x => x.id}
      refreshControl={<RefreshControl refreshing={refreshing} onRefresh={load} />}
      renderItem={({ item }) => (
        <View style={{ flexDirection: 'row', gap: 12, padding: 16, borderBottomWidth: 1, borderColor: '#eee' }}>
          <Text style={{ fontSize: 22 }}>{ICONS[item.action] ?? '•'}</Text>
          <View style={{ flex: 1 }}>
            <Text style={{ fontWeight: '600' }}>{LABELS[item.action] ?? item.action}</Text>
            <Text style={{ color: '#666' }}>
              {new Date(item.created_at).toLocaleString('fr-FR')}
              {item.pro_display_name ? ` · ${item.pro_display_name}` : ''}
            </Text>
            {item.ip_address ? <Text style={{ color: '#999', fontSize: 12 }}>IP : {item.ip_address}</Text> : null}
          </View>
        </View>
      )}
    />
  );
}
```

### 4.6 Notifications locales sur action du pro

À ajouter au démarrage de l'app (après login) :

```typescript
import * as Notifications from 'expo-notifications';
import { supabase } from './lib/supabase';

export function subscribeProAccessNotifications(userId: string) {
  return supabase
    .channel('pro_access_log_user_' + userId)
    .on('postgres_changes', {
      event: 'INSERT', schema: 'public', table: 'pro_access_log',
      filter: `owner_id=eq.${userId}`,
    }, async (payload) => {
      const action = payload.new.action;
      const who = payload.new.pro_display_name ?? 'Un professionnel';
      const titles: Record<string, string> = {
        link_opened:   `${who} a ouvert votre carnet`,
        entry_added:   `${who} a ajouté une intervention`,
        viewed_vehicle: `${who} consulte votre carnet`,
      };
      if (!titles[action]) return;
      await Notifications.scheduleNotificationAsync({
        content: { title: 'AFP Garage', body: titles[action] },
        trigger: null,
      });
    })
    .subscribe();
}
```

### 4.7 Navigation à câbler

Dans votre stack navigator existant :

```tsx
<Stack.Screen name="ProShareModal"    component={ProShareModal}      options={{ presentation: 'modal' }} />
<Stack.Screen name="ProAccessList"    component={ProAccessListScreen} options={{ title: 'Accès Pro actifs' }} />
<Stack.Screen name="ProAccessLog"     component={ProAccessLogScreen}  options={{ title: 'Journal d\'accès Pro' }} />
```

---

## 5. Site Pro web (`pro.afpgarage.app`)

Nouveau repo, **React + Vite + TypeScript + Tailwind**. Mobile-first, hébergement Vercel.

### 5.1 Initialiser

```bash
npm create vite@latest afp-pro -- --template react-ts
cd afp-pro
npm i
npm i -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

Ajouter dans `tailwind.config.js` → `content: ["./index.html","./src/**/*.{ts,tsx}"]`. Dans `src/index.css` :
```css
@tailwind base; @tailwind components; @tailwind utilities;
```

Variables d'env (`.env`) :
```
VITE_EDGE_URL=https://<project-ref>.supabase.co/functions/v1/mode-pro
```

### 5.2 `src/api.ts`

```typescript
const BASE = import.meta.env.VITE_EDGE_URL as string;

export type Vehicle = any;
export type Entry = any;

let jwt: string | null = sessionStorage.getItem('jwt');
export function setJwt(v: string | null) {
  jwt = v;
  if (v) sessionStorage.setItem('jwt', v); else sessionStorage.removeItem('jwt');
}
export const hasJwt = () => !!jwt;

async function call(path: string, init: RequestInit = {}) {
  const r = await fetch(BASE + path, {
    ...init,
    headers: {
      'content-type': 'application/json',
      ...(jwt ? { authorization: `Bearer ${jwt}` } : {}),
      ...(init.headers ?? {}),
    },
  });
  const body = await r.json().catch(() => ({}));
  if (!r.ok) throw Object.assign(new Error(body.error ?? r.statusText), { status: r.status, body });
  return body;
}

export async function validate(token: string, pin: string, proName: string) {
  const r = await call('/validate', { method: 'POST', body: JSON.stringify({ token, pin, pro_name: proName }) });
  setJwt(r.jwt);
  return r;
}
export async function getVehicle(): Promise<{ vehicle: Vehicle; history: Entry[] }> {
  return await call('/vehicle');
}
export async function addEntry(e: any) {
  return (await call('/entry', { method: 'POST', body: JSON.stringify(e) })).entry;
}
```

### 5.3 `src/App.tsx`

```tsx
import { useState, useEffect } from 'react';
import { validate, getVehicle, addEntry, hasJwt, setJwt } from './api';

type Stage = 'pin' | 'dashboard' | 'add' | 'done' | 'error';

export default function App() {
  const params = new URLSearchParams(location.search);
  const token = params.get('t') ?? '';

  const [stage, setStage] = useState<Stage>(hasJwt() ? 'dashboard' : 'pin');
  const [pin, setPin] = useState('');
  const [proName, setProName] = useState('');
  const [err, setErr] = useState('');
  const [vehicle, setVehicle] = useState<any>(null);
  const [history, setHistory] = useState<any[]>([]);

  if (!token) return <Screen><h1 className="text-2xl">Lien invalide</h1></Screen>;

  async function onValidate() {
    setErr('');
    try {
      await validate(token, pin, proName);
      const v = await getVehicle();
      setVehicle(v.vehicle); setHistory(v.history);
      setStage('dashboard');
    } catch (e: any) {
      setErr(e.body?.error === 'invalid_pin' ? 'Code PIN incorrect.' :
             e.body?.error === 'link_expired' ? 'Ce lien a expiré.' :
             e.body?.error === 'too_many_attempts' ? 'Trop de tentatives. Réessayez dans 15 min.' :
             'Erreur. Réessayez.');
    }
  }

  useEffect(() => {
    if (stage === 'dashboard' && !vehicle) {
      getVehicle().then(v => { setVehicle(v.vehicle); setHistory(v.history); })
        .catch(() => { setJwt(null); setStage('pin'); });
    }
  }, [stage]);

  if (stage === 'pin') return (
    <Screen>
      <h1 className="text-2xl font-bold">AFP Garage · Espace Pro</h1>
      <p className="text-slate-600">Votre client vous a partagé son carnet d'entretien.</p>
      <label className="block mt-6 text-sm">Code PIN à 6 chiffres</label>
      <input value={pin} onChange={e => setPin(e.target.value.replace(/\D/g, '').slice(0,6))}
             inputMode="numeric" maxLength={6}
             className="w-full text-4xl tracking-widest text-center border rounded-xl py-3 mt-1" />
      <label className="block mt-4 text-sm">Votre nom ou celui de votre établissement</label>
      <input value={proName} onChange={e => setProName(e.target.value)}
             placeholder="Auto France Performance"
             className="w-full border rounded-xl px-3 py-2 mt-1" />
      <p className="text-xs text-slate-500 mt-2">
        Ce nom apparaîtra dans le carnet du client et ne pourra plus être modifié.
      </p>
      {err && <p className="text-red-600 mt-3">{err}</p>}
      <button onClick={onValidate} disabled={pin.length !== 6 || proName.trim().length < 2}
              className="w-full mt-6 bg-blue-600 disabled:bg-slate-300 text-white font-semibold py-3 rounded-xl">
        Accéder au carnet
      </button>
    </Screen>
  );

  if (stage === 'dashboard' && vehicle) return (
    <Screen>
      <h1 className="text-2xl font-bold">{vehicle.brand} {vehicle.model}</h1>
      <p className="text-slate-600">{vehicle.license_plate} · {vehicle.year} · {vehicle.fuel_type}</p>
      <p className="text-slate-600">VIN : <span className="font-mono">{vehicle.vin}</span></p>
      <p className="text-slate-600">Kilométrage : {vehicle.current_mileage_km} km</p>

      <h2 className="text-lg font-semibold mt-6 mb-2">Historique</h2>
      <div className="space-y-2">
        {history.map(h => (
          <div key={h.id} className="border rounded-xl p-3">
            <div className="flex justify-between">
              <span className="font-semibold">{h.title ?? h.type}</span>
              <span className="text-slate-500">{new Date(h.date).toLocaleDateString('fr-FR')}</span>
            </div>
            <div className="text-sm text-slate-600">
              {h.mileage_km ? `${h.mileage_km} km · ` : ''}{h.cost_eur ? `${h.cost_eur} €` : ''}
              {h.garage_name ? ` · ${h.garage_name}` : ''}
            </div>
            {h.added_by_pro_name && (
              <div className="text-xs text-emerald-700 mt-1">✓ signée {h.added_by_pro_name}</div>
            )}
          </div>
        ))}
        {!history.length && <p className="text-slate-500">Aucune intervention enregistrée.</p>}
      </div>

      <button onClick={() => setStage('add')}
              className="fixed bottom-6 right-6 bg-blue-600 text-white font-semibold py-3 px-5 rounded-full shadow-lg">
        ➕ Ajouter
      </button>
    </Screen>
  );

  if (stage === 'add') return <AddEntry onDone={() => setStage('done')} onCancel={() => setStage('dashboard')} />;

  if (stage === 'done') return (
    <Screen>
      <div className="text-center mt-12">
        <div className="text-6xl mb-4">✅</div>
        <h1 className="text-2xl font-bold">Intervention enregistrée</h1>
        <p className="text-slate-600 mt-2">Le client a été notifié.</p>
        <button onClick={() => { setStage('add'); }} className="mt-8 bg-blue-600 text-white font-semibold py-3 px-6 rounded-xl">
          Ajouter une autre
        </button>
        <button onClick={() => { setJwt(null); location.reload(); }} className="block mx-auto mt-4 text-slate-600">
          Terminer la session
        </button>
      </div>
    </Screen>
  );

  return null;
}

function AddEntry({ onDone, onCancel }: { onDone: () => void; onCancel: () => void }) {
  const today = new Date().toISOString().slice(0,10);
  const [date, setDate] = useState(today);
  const [type, setType] = useState('vidange');
  const [title, setTitle] = useState('');
  const [km, setKm] = useState('');
  const [cost, setCost] = useState('');
  const [garage, setGarage] = useState('');
  const [notes, setNotes] = useState('');
  const [busy, setBusy] = useState(false);
  const [err, setErr] = useState('');

  async function submit() {
    setBusy(true); setErr('');
    try {
      await addEntry({
        date, type, title: title || null,
        mileage_km: km ? Number(km) : null,
        cost_eur: cost ? Number(cost) : null,
        garage_name: garage || null,
        notes: notes || null,
      });
      onDone();
    } catch (e: any) {
      setErr('Erreur : ' + (e.body?.detail ?? e.message));
    } finally { setBusy(false); }
  }

  return (
    <Screen>
      <button onClick={onCancel} className="text-slate-600 mb-2">← Annuler</button>
      <h1 className="text-2xl font-bold">Nouvelle intervention</h1>

      <Field label="Date *">
        <input type="date" value={date} onChange={e => setDate(e.target.value)} className="border rounded-xl px-3 py-2 w-full" />
      </Field>
      <Field label="Type *">
        <select value={type} onChange={e => setType(e.target.value)} className="border rounded-xl px-3 py-2 w-full">
          <option value="vidange">Vidange</option>
          <option value="freinage">Freinage</option>
          <option value="distribution">Distribution</option>
          <option value="pneus">Pneus</option>
          <option value="controle_technique">Contrôle technique</option>
          <option value="autre">Autre</option>
        </select>
      </Field>
      <Field label="Libellé"><input value={title} onChange={e => setTitle(e.target.value)} className="border rounded-xl px-3 py-2 w-full" /></Field>
      <Field label="Kilométrage relevé"><input inputMode="numeric" value={km} onChange={e => setKm(e.target.value.replace(/\D/g,''))} className="border rounded-xl px-3 py-2 w-full" /></Field>
      <Field label="Coût TTC (€)"><input inputMode="decimal" value={cost} onChange={e => setCost(e.target.value)} className="border rounded-xl px-3 py-2 w-full" /></Field>
      <Field label="Garage / Mécanicien"><input value={garage} onChange={e => setGarage(e.target.value)} className="border rounded-xl px-3 py-2 w-full" /></Field>
      <Field label="Notes"><textarea value={notes} onChange={e => setNotes(e.target.value)} className="border rounded-xl px-3 py-2 w-full" rows={3} /></Field>

      <p className="text-xs text-amber-700 mt-4">
        ⚠️ Une fois validée, cette intervention ne pourra plus être modifiée ni supprimée par vous.
        Le client recevra une notification.
      </p>
      {err && <p className="text-red-600 mt-2">{err}</p>}
      <button onClick={submit} disabled={busy}
              className="w-full mt-4 bg-blue-600 disabled:bg-slate-300 text-white font-semibold py-3 rounded-xl">
        {busy ? '...' : 'Valider l\'entrée'}
      </button>
    </Screen>
  );
}

function Field({ label, children }: any) {
  return <label className="block mt-3"><span className="block text-sm mb-1">{label}</span>{children}</label>;
}
function Screen({ children }: any) {
  return (
    <div className="max-w-md mx-auto px-4 py-6">
      <div className="text-xs text-slate-500 mb-3">AFP Garage · Espace Pro</div>
      {children}
    </div>
  );
}
```

### 5.4 Déploiement sur Vercel

```bash
npm i -g vercel
vercel
vercel --prod
vercel env add VITE_EDGE_URL          # mettre l'URL de la fonction Supabase
```

Configurer le domaine `pro.afpgarage.app` (ou `pro.autofranceperformance.fr` si vous voulez rester sur votre domaine existant) dans le dashboard Vercel.

---

## 6. Variables d'environnement à renseigner

| Service | Variable | Valeur |
|---|---|---|
| Supabase (secrets) | `MODE_PRO_JWT_SECRET` | `openssl rand -base64 48` |
| Supabase (fournis auto) | `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_ANON_KEY` | déjà disponibles |
| App RN (`.env`) | `EXPO_PUBLIC_PRO_BASE_URL` | `https://pro.afpgarage.app` |
| Site Pro web | `VITE_EDGE_URL` | `https://<ref>.supabase.co/functions/v1/mode-pro` |

---

## 7. Plan de test bout-en-bout

1. **Migration** : appliquer en staging, vérifier que les tables et RPC existent.
2. **Edge function** : `curl -X POST` sur `/validate` avec faux token → 404. Avec vrai token + mauvais PIN → 401 + entrée dans `pro_access_log`.
3. **App RN** : sur un véhicule existant, créer un partage → vérifier que `vehicle_shares` contient une row avec `pin_hash` (jamais le PIN clair).
4. **Site Pro web** : ouvrir le QR sur un autre téléphone → saisir PIN + nom → voir le véhicule + l'historique.
5. **Ajout d'entrée par le pro** : vérifier qu'elle apparaît dans l'app du propriétaire avec `added_by_pro_name = "Auto France Performance"` et le badge « ✓ vérifié ».
6. **Notification locale** : le propriétaire reçoit la notif dans les secondes qui suivent.
7. **Révocation** : depuis l'app → le JWT du pro est invalidé immédiatement (la prochaine action renvoie 401).
8. **Expiration** : forcer `expires_at = now() - 1h` en base → vérifier que la session pro renvoie 401.
9. **Rate limit** : 5 PIN faux → 6ᵉ tentative renvoie 429.

---

## 8. Pour la resoumission App Store

Une fois tout testé et déployé en prod :

- [ ] Build TestFlight, vérifier sur **iPad Air 11 pouces (M3)** (l'appareil de test d'Apple)
- [ ] Mettre à jour la **description App Store** (Mode Pro mis en avant)
- [ ] Refaire les **screenshots** : inclure la modal de partage (QR + PIN), l'écran « accès Pro actifs », une entrée signée par un pro
- [ ] Bumper la version : 1.0 (19) → 1.1 (20)
- [ ] Soumettre
- [ ] Répondre dans Resolution Center avec un message factuel mentionnant la fonctionnalité Mode Pro et l'URL de la politique de confidentialité mise à jour

---

## 9. Fichiers à créer côté repo app

```
afp garage/
├── supabase/
│   ├── migrations/
│   │   └── 20260513000001_mode_pro.sql        ← § 2
│   └── functions/
│       └── mode-pro/
│           └── index.ts                        ← § 3
├── src/
│   ├── lib/
│   │   └── proShare.ts                         ← § 4.2
│   ├── screens/
│   │   ├── ProShareModal.tsx                   ← § 4.3
│   │   ├── ProAccessListScreen.tsx             ← § 4.4
│   │   └── ProAccessLogScreen.tsx              ← § 4.5
│   └── notifications/
│       └── proAccessNotifications.ts           ← § 4.6
└── package.json                                ← ajouter react-native-qrcode-svg, react-native-svg
```

Et un nouveau dépôt séparé pour le site Pro :

```
afp-pro/
├── src/
│   ├── api.ts                                  ← § 5.2
│   └── App.tsx                                 ← § 5.3
├── tailwind.config.js
└── .env (VITE_EDGE_URL)
```

---

**Fin du document.** En cas de doute, demandez à Claude Code local en lui montrant le bloc concerné de ce fichier. La politique de confidentialité (`afp-privacy` repo, branche `claude/review-afp-garage-feedback-S1ele`) est déjà alignée et n'a pas besoin d'être modifiée pour cette implémentation.

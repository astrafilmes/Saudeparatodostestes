# SAÚDE PARA TODOS — ENTERPRISE ARCHITECTURE CONSTITUTION v2.0
> Documento-mestre — **Single Source of Truth (SSoT)** — Junho/2026
> Substitui: `saude-para-todos-MASTER-ARCHITECTURE.md` (v1.0)
> Audiência: Engenharia, Produto, Segurança, DPO, SRE, QA, Design System.
> Classificação: Confidencial — Uso Interno.
> Status: **MANDATÓRIO** — qualquer desvio exige RFC aprovado pelo Tech Lead + Security Officer.

---

## SUMÁRIO EXECUTIVO

A plataforma **Saúde para Todos** é um sistema multi-perfil (Cidadão, ACS, Profissional de Saúde, Gestor, Admin) de gestão de saúde pública e cuidado contínuo, construído sobre **TanStack Start v1 + React 19 + Supabase (Lovable Cloud) + Cloudflare Workers**.

Esta v2.0 incorpora a **Análise Crítica de Falhas Arquiteturais** (Estado Cliente, Offline-First, Hard Gates de CI/CD, Workflow de Migrações, Telemetria e Gestão de Dependências) e converte recomendações em **bloqueios de compilação** e **invariantes de runtime**.

**Princípios fundadores (não-negociáveis):**
1. **Zero-Trust por padrão** — RLS sempre ligado, `service_role` nunca cruza fronteira cliente.
2. **Zero-Crash Policy** — toda rota tem `errorComponent` + `notFoundComponent`; toda exceção é reportada ao Sentry antes do fallback.
3. **Offline-First** — ACS opera em campo sem rede; UI nunca depende de conectividade contínua.
4. **Anti-Hallucination = Hard Gate** — design tokens, RLS e tipagem são validados em pre-commit e CI, não por revisão humana.
5. **LGPD by Design** — PII mascarada por padrão, `audit_logs` imutável, consentimento versionado.
6. **CLS = 0.00** — toda transição usa `useTransition` + Skeletons dimensionados.

---

## ÍNDICE

1. Visão Geral e Personas
2. Stack Oficial e Travamento de Versões
3. Estrutura de Pastas Canônica
4. Design System Constitucional (tokens, geometria, tipografia)
5. App Shell e Layout Nativo (h-[100dvh])
6. Roteamento (TanStack Router + RBAC)
7. **Gestão de Estado — Servidor (TanStack Query) vs Cliente (Zustand/XState)**
8. **Arquitetura Offline-First (PWA + PGlite + Sync Queue)**
9. Camada de Dados Supabase (clientes, RLS, GRANTs)
10. Autenticação e Autorização (RBAC + Permissões granulares)
11. Workflow de Banco de Dados (CLI, Migrations, Tipagem)
12. **Hard Gates: Husky, lint-staged, ESLint Customizado**
13. **Pipeline CI/CD (GitHub Actions, Vitest 80%, Playwright)**
14. **Telemetria, Observabilidade e Sentry**
15. Gestão de Dependências (pnpm exclusivo, engine-strict)
16. Segurança Aplicacional (CSP, headers, secrets)
17. LGPD, Auditoria e Mascaramento de PII
18. Módulos Clínicos (Saúde, Nutrição, Imuniza, Mental, Família, Gestante, Cuidado)
19. Módulos de Gestão (Pro, Gestão, Território)
20. Componentes UI-Kit (catálogo e contratos)
21. Performance Budget e Web Vitals
22. Acessibilidade (WCAG 2.2 AA)
23. Internacionalização e Conteúdo
24. Runbooks de Incidentes
25. Anti-Hallucination Protocol (18 pontos verificáveis)
26. Glossário e Referências

---

## 1. VISÃO GERAL E PERSONAS

### 1.1 Os 5 Pilares
| Perfil | Rota raiz | Responsabilidade primária |
|---|---|---|
| Cidadão | `/` | Prontuário pessoal, agendamentos, módulos clínicos |
| ACS (Agente Comunitário) | `/territorio/*` | Busca ativa, mapa, cadastro em campo (offline) |
| Profissional de Saúde | `/pro/*` | Atendimento, prescrição, aplicação de vacinas |
| Gestor | `/gestao/*` | Relatórios, equipe, agendamentos, feature flags |
| Admin | `/gestao/usuarios`, `/gestao/templates-permissao` | RBAC, auditoria, diagnóstico |

### 1.2 Domínios de Negócio
- **Identidade**: `profiles`, `user_roles`, `patient_profiles`, `permissions`, `role_permissions`, `user_permissions`, `permission_templates`.
- **Cuidado**: `appointments`, `vaccination_records`, `chronic_medications`, `chronic_medication_intakes`, `mental_health_logs`, `nutritional_plans`, `pregnancy_records`, `health_care_logs`, `chronic_care_logs`.
- **Plataforma**: `audit_logs`, `feature_flags`, `home_widgets`, `health_campaigns`, `ubs`, `specialties`, `team_types`, `acs_agents`, `family_dependents`, `medicines_catalog`, `vaccine_catalog`, `vaccine_schedule`.

---

## 2. STACK OFICIAL E TRAVAMENTO DE VERSÕES

### 2.1 Stack
```
runtime:         Cloudflare Workers (nodejs_compat) + Bun (dev)
framework:       TanStack Start v1.168.x
ui:              React 19.x
build:           Vite 7.x
estilo:          Tailwind v4 (CSS-first via @import)
db/auth/storage: Supabase (Lovable Cloud)
estado-servidor: @tanstack/react-query v5
estado-cliente:  Zustand 4.x  |  XState 5.x (fluxos)
formulários:     react-hook-form + Zod
offline:         vite-plugin-pwa + PGlite (IndexedDB-backed) + dexie (queue)
telemetria:      @sentry/react + @sentry/tanstackstart-react
testes:          Vitest 2.x + @testing-library/react + Playwright 1.4x
qualidade:       ESLint (flat) + Prettier + Husky + lint-staged + commitlint
pacotes:         pnpm 9.x (exclusivo, engine-strict)
```

### 2.2 Travamento (`package.json`)
```jsonc
{
  "packageManager": "pnpm@9.12.0",
  "engines": {
    "node": ">=20.11 <21",
    "pnpm": ">=9.12 <10"
  },
  "engineStrict": true
}
```
`.npmrc`:
```
engine-strict=true
package-manager-strict=true
strict-peer-dependencies=true
auto-install-peers=false
```

### 2.3 Proibições
- `npm install` / `yarn` / `bun add` em produção → bloqueado por pre-commit (`.husky/pre-commit` valida lockfile).
- Versões `^` ou `~` em dependências críticas (`react`, `@tanstack/*`, `@supabase/*`, `vite`) → pinadas exatas.

---

## 3. ESTRUTURA DE PASTAS CANÔNICA

```
saude-para-todos/
├─ .github/workflows/         # CI: lint, typecheck, test, e2e, build
├─ .husky/                    # pre-commit, commit-msg, pre-push
├─ public/                    # assets estáticos, manifest, sw kill-switch
├─ src/
│  ├─ assets/                 # imagens importadas por bundler
│  ├─ components/
│  │  ├─ ui/                  # shadcn primitives (não editar libs)
│  │  ├─ ui-kit/              # primitives próprios (StatusPill, etc.)
│  │  ├─ layouts/             # MobileLayout, DesktopLayout
│  │  └─ <feature>/           # componentes por feature
│  ├─ features/               # casos de uso (Zustand stores + XState machines)
│  │  ├─ agendamento/
│  │  │  ├─ machine.ts        # XState máquina do fluxo
│  │  │  ├─ store.ts          # Zustand UI store
│  │  │  └─ components/
│  │  └─ ...
│  ├─ hooks/                  # hooks compartilhados
│  ├─ integrations/supabase/  # AUTO-GERADO — não editar
│  ├─ lib/                    # helpers puros + .functions.ts
│  ├─ offline/                # PGlite schema, sync queue, conflict resolver
│  │  ├─ db.ts                # instância PGlite
│  │  ├─ schema.sql           # espelho do schema crítico
│  │  └─ sync.ts              # background sync queue
│  ├─ routes/                 # file-based routing
│  ├─ stores/                 # Zustand stores globais (theme, modais, network)
│  ├─ telemetry/              # Sentry init, breadcrumbs, masking
│  ├─ styles.css              # Tailwind v4 tokens
│  └─ test/                   # setup Vitest, mocks, fixtures
├─ supabase/
│  ├─ migrations/             # IMUTÁVEL — append-only
│  └─ seed.sql
├─ tests/
│  └─ e2e/                    # Playwright specs
└─ pnpm-lock.yaml             # commit obrigatório
```

**Regra de ouro:** nunca crie `src/pages/`, `app/`, ou layouts Next/Remix. O TanStack Router usa `src/routes/`.

---

## 4. DESIGN SYSTEM CONSTITUTIONAL

### 4.1 Tokens Semânticos (CSS — `src/styles.css`)
**PROIBIDO** usar hex literal em componentes. Todo cor passa por token.

```css
@import "tailwindcss";

@theme {
  /* Paleta base — Saúde para Todos */
  --color-app-indigo:   oklch(36% 0.22 277);   /* #4338c4 */
  --color-app-lime:     oklch(96% 0.22 117);   /* #dfff2f */
  --color-app-lavender: oklch(97% 0.02 295);   /* #f5f3ff */
  --color-app-pink:     oklch(76% 0.13 10);    /* #ef7b9b */
  --color-app-mint:     oklch(94% 0.04 175);   /* #e0f6f1 */
  --color-app-orange:   oklch(72% 0.21 35);    /* #ff704f */

  /* Semânticos */
  --color-background: var(--color-app-lavender);
  --color-foreground: oklch(18% 0.04 277);
  --color-primary:    var(--color-app-indigo);
  --color-primary-foreground: oklch(99% 0 0);
  --color-muted:      oklch(96% 0.005 277);
  --color-muted-foreground: oklch(48% 0.02 277);
  --color-border:     oklch(92% 0.01 277);
  --color-ring:       color-mix(in oklch, var(--color-primary) 35%, transparent);

  /* Status (semânticos — NUNCA usar verde/vermelho cru) */
  --color-status-success: oklch(72% 0.16 155);
  --color-status-warn:    oklch(82% 0.17 85);
  --color-status-danger:  oklch(62% 0.22 25);
  --color-status-info:    oklch(70% 0.13 230);
  --color-status-process: oklch(66% 0.15 265);

  /* Geometria */
  --radius-card:  32px;
  --radius-bento: 24px;
  --radius-pill:  9999px;

  /* Elevações */
  --shadow-soft:    0 4px 16px -8px color-mix(in oklch, var(--color-primary) 18%, transparent);
  --shadow-floating: 0 24px 48px -24px color-mix(in oklch, var(--color-primary) 30%, transparent);

  /* Tipografia */
  --font-display: "Sora", "ui-sans-serif", system-ui;
  --font-body:    "Manrope", "ui-sans-serif", system-ui;
}

@media (prefers-color-scheme: dark) {
  @theme {
    --color-background: oklch(12% 0.03 270);    /* #0B1120 */
    --color-foreground: oklch(96% 0.01 277);
    --color-muted:      oklch(20% 0.03 270);
    --color-border:     oklch(28% 0.03 270);
  }
}
```

### 4.2 Regras Estritas (bloqueadas no ESLint)
- ❌ `text-[#…]`, `bg-[#…]`, `border-[#…]`, `shadow-[…]` (arbitrários cromáticos)
- ❌ Emoji em UI funcional (apenas conteúdo de usuário)
- ❌ Ícones de outras libs — somente `lucide-react` com `strokeWidth={1.5}`
- ❌ `rounded-md/lg/xl` em cards principais — usar `rounded-[var(--radius-card)]`
- ✅ Touch targets ≥ 44px (h-11 mínimo em interativos)

---

## 5. APP SHELL — LAYOUT NATIVO

```tsx
// src/components/layouts/MobileLayout.tsx
export function AppShell({ children }: PropsWithChildren) {
  return (
    <div className="fixed inset-0 w-full h-[100dvh] bg-background flex flex-col overflow-hidden">
      <Header />
      <main className="flex-1 overflow-y-auto overscroll-none hide-scrollbar relative">
        <div className="w-full max-w-3xl mx-auto px-4 py-6 pb-safe-offset-32">
          {children}
        </div>
      </main>
      <BottomBar />
    </div>
  );
}
```

**Invariantes:**
- Root sempre `h-[100dvh]` + `overflow-hidden`.
- Scroll isolado em `<main>` com `overscroll-none` (impede pull-to-refresh quebrar contexto).
- `pb-safe-offset-32` reserva área da BottomBar + safe-area iOS.
- `hide-scrollbar` é utility custom em `styles.css`.

---

## 6. ROTEAMENTO

### 6.1 Convenção
- `src/routes/_authenticated/` → protegido (gate gerenciado, `ssr:false`).
- `src/routes/*.tsx` na raiz → público (login, esqueci-senha, reset-senha).
- Rotas dinâmicas: `$param` (nunca `:param`).
- Layouts: arquivo pai com `<Outlet />`, filho em `*.index.tsx`.

### 6.2 RBAC nas Rotas
```ts
// src/lib/route-guards.ts (já existente)
beforeLoad: ({ context }) => requireManagementRoute(context)
```
Rotas unificadas para citizen + manager: a UI injeta controles condicionalmente via `useAuth().role`, nunca duplicando rota.

---

## 7. GESTÃO DE ESTADO — REGRA DE OURO

> **Servidor = TanStack Query. Cliente efêmero = Zustand. Fluxo multi-etapa = XState. Formulário = react-hook-form + Zod.**

### 7.1 Matriz de Decisão
| Tipo de estado | Ferramenta | Exemplo |
|---|---|---|
| Dados remotos (lista, detalhe, mutação) | **TanStack Query** | `useSuspenseQuery(appointmentsQueryOptions)` |
| UI global leve (tema, drawer aberto, toast queue, status de rede) | **Zustand** | `useUIStore(s => s.openCommandPalette)` |
| Fluxo multi-etapa com estados nomeados (agendamento, anamnese, onboarding) | **XState** | `agendamentoMachine` |
| Formulário (validação, dirty, submit) | **react-hook-form + Zod** | `useForm({ resolver: zodResolver(schema) })` |
| Estado local de 1 componente | `useState` / `useReducer` | toggle de acordeon |

### 7.2 Zustand — Padrão
```ts
// src/stores/ui-store.ts
import { create } from "zustand";
import { devtools, persist } from "zustand/middleware";

interface UIState {
  commandPaletteOpen: boolean;
  activeModal: string | null;
  setCommandPalette: (open: boolean) => void;
  openModal: (id: string) => void;
  closeModal: () => void;
}

export const useUIStore = create<UIState>()(
  devtools(
    persist(
      (set) => ({
        commandPaletteOpen: false,
        activeModal: null,
        setCommandPalette: (open) => set({ commandPaletteOpen: open }),
        openModal: (id) => set({ activeModal: id }),
        closeModal: () => set({ activeModal: null }),
      }),
      { name: "spt-ui", partialize: (s) => ({ /* nada persistente sensível */ }) },
    ),
  ),
);
```
**Regras Zustand:**
- 1 store por domínio de UI; nunca um "god store".
- Seletores sempre via callback: `useUIStore(s => s.x)` (evita re-render global).
- Proibido armazenar dados de servidor.

### 7.3 XState — Fluxo de Agendamento (canônico)
```ts
// src/features/agendamento/machine.ts
import { setup, assign } from "xstate";

export const agendamentoMachine = setup({
  types: {
    context: {} as {
      especialidade?: string;
      ubs?: string;
      profissional?: string;
      slot?: string;
      erro?: string;
    },
    events: {} as
      | { type: "SELECIONAR_ESPECIALIDADE"; value: string }
      | { type: "SELECIONAR_UBS"; value: string }
      | { type: "SELECIONAR_PROFISSIONAL"; value: string }
      | { type: "SELECIONAR_SLOT"; value: string }
      | { type: "VOLTAR" }
      | { type: "CONFIRMAR" }
      | { type: "RESETAR" },
  },
  actions: {
    setEspecialidade: assign({ especialidade: ({ event }) => (event as any).value }),
    /* … */
  },
  guards: {
    podeConfirmar: ({ context }) =>
      !!(context.especialidade && context.ubs && context.profissional && context.slot),
  },
}).createMachine({
  id: "agendamento",
  initial: "especialidade",
  context: {},
  states: {
    especialidade: { on: { SELECIONAR_ESPECIALIDADE: { target: "ubs", actions: "setEspecialidade" } } },
    ubs:           { on: { SELECIONAR_UBS: "profissional", VOLTAR: "especialidade" } },
    profissional:  { on: { SELECIONAR_PROFISSIONAL: "slot", VOLTAR: "ubs" } },
    slot:          { on: { SELECIONAR_SLOT: "revisao", VOLTAR: "profissional" } },
    revisao:       { on: { CONFIRMAR: { target: "submetendo", guard: "podeConfirmar" }, VOLTAR: "slot" } },
    submetendo:    { invoke: { src: "criarAgendamento", onDone: "sucesso", onError: "erro" } },
    sucesso:       { type: "final" },
    erro:          { on: { RESETAR: "especialidade" } },
  },
});
```
Componentes consomem via `@xstate/react`'s `useMachine`. **Proibido** simular fluxo multi-etapa com `useState` aninhado.

### 7.4 react-hook-form + Zod
```ts
const schema = z.object({
  cpf: z.string().refine(isCPF, "CPF inválido"),
  nascimento: z.coerce.date(),
});
type Form = z.infer<typeof schema>;

const form = useForm<Form>({ resolver: zodResolver(schema), mode: "onBlur" });
```

---

## 8. ARQUITETURA OFFLINE-FIRST

### 8.1 Princípio
Toda leitura crítica para ACS (`patient_profiles`, `family_dependents`, `vaccination_records`, `chronic_medications`) deve ter **espelho local em PGlite (IndexedDB)**. Mutações em modo offline entram em uma **outbox queue** (Dexie) e são drenadas quando `navigator.onLine === true`.

### 8.2 Camadas
```
┌─ UI ────────────────────────────────────────┐
│ useSuspenseQuery → queryFn (estratégia)     │
└─────────────┬───────────────────────────────┘
              ▼
   ┌──────────────────────┐
   │  Strategy Selector    │  online? supabase : pglite
   └─────────┬─────────────┘
        │              │
        ▼              ▼
   Supabase REST   PGlite (local)
        │              ▲
        └──► Sync ─────┘
             ▲
   Outbox (Dexie) ──► drainage on online
```

### 8.3 PGlite — Schema Espelho
```ts
// src/offline/db.ts
import { PGlite } from "@electric-sql/pglite";
import { live } from "@electric-sql/pglite/live";

export const pg = new PGlite("idb://spt-offline", { extensions: { live } });

export async function bootstrapLocalSchema() {
  await pg.exec(`
    CREATE TABLE IF NOT EXISTS patient_profiles_cache (
      user_id UUID PRIMARY KEY,
      payload JSONB NOT NULL,
      synced_at TIMESTAMPTZ NOT NULL DEFAULT now()
    );
    CREATE TABLE IF NOT EXISTS vaccination_records_cache (
      id UUID PRIMARY KEY,
      user_id UUID NOT NULL,
      payload JSONB NOT NULL,
      synced_at TIMESTAMPTZ NOT NULL DEFAULT now()
    );
    CREATE INDEX IF NOT EXISTS idx_vax_user ON vaccination_records_cache(user_id);
  `);
}
```

### 8.4 Outbox + Sync
```ts
// src/offline/outbox.ts
import Dexie, { Table } from "dexie";

export interface OutboxEntry {
  id?: number;
  resource: string;          // "vaccination_records"
  op: "insert" | "update" | "delete";
  payload: unknown;
  client_uuid: string;       // idempotência
  created_at: number;
  attempts: number;
  last_error?: string;
}

class OutboxDB extends Dexie {
  entries!: Table<OutboxEntry, number>;
  constructor() {
    super("spt-outbox");
    this.version(1).stores({ entries: "++id, resource, created_at, client_uuid" });
  }
}
export const outbox = new OutboxDB();
```
```ts
// src/offline/sync.ts
export async function drainOutbox() {
  if (!navigator.onLine) return;
  const items = await outbox.entries.orderBy("created_at").toArray();
  for (const e of items) {
    try {
      await applyToSupabase(e);                        // RLS + idempotência via client_uuid
      await outbox.entries.delete(e.id!);
    } catch (err) {
      await outbox.entries.update(e.id!, {
        attempts: e.attempts + 1,
        last_error: (err as Error).message,
      });
      if (e.attempts > 5) await reportToSentry(err, { tag: "outbox-poison", entry: e });
    }
  }
}
window.addEventListener("online", drainOutbox);
setInterval(drainOutbox, 30_000);
```

**Idempotência:** toda tabela alvo de sync ganha coluna `client_uuid UUID UNIQUE` (migration dedicada). `INSERT` usa `ON CONFLICT (client_uuid) DO NOTHING`.

### 8.5 PWA — Shell Cache
- `vite-plugin-pwa` com `generateSW`, `registerType: "autoUpdate"`.
- HTML: `NetworkFirst`. Assets hashed: `CacheFirst`.
- Wrapper de registro guardado conforme `skill/pwa` (não registra em preview, iframe, dev).

### 8.6 Indicador de Rede
Zustand store `useNetworkStore` expõe `online`, `pendingOutboxCount`, `lastSyncAt`. Componente `<NetworkPill />` renderizado no Header.

---

## 9. CAMADA DE DADOS SUPABASE

### 9.1 Clientes
| Cliente | Onde | Auth | RLS |
|---|---|---|---|
| `@/integrations/supabase/client` | browser, hooks, realtime | publishable + sessão | sim |
| `requireSupabaseAuth` middleware | `createServerFn` | bearer do usuário | sim |
| `@/integrations/supabase/client.server` (`supabaseAdmin`) | server-only, trusted | service_role | **bypass** |

**Regra:** `supabaseAdmin` só pode ser importado dentro do `.handler()` via `await import(...)` em arquivos `.functions.ts`.

### 9.2 RLS — Template Obrigatório
```sql
CREATE TABLE public.minha_tabela (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  client_uuid UUID UNIQUE,        -- idempotência offline
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

GRANT SELECT, INSERT, UPDATE, DELETE ON public.minha_tabela TO authenticated;
GRANT ALL ON public.minha_tabela TO service_role;

ALTER TABLE public.minha_tabela ENABLE ROW LEVEL SECURITY;

CREATE POLICY "owner_select" ON public.minha_tabela FOR SELECT TO authenticated
  USING (auth.uid() = user_id);
CREATE POLICY "owner_write" ON public.minha_tabela FOR INSERT TO authenticated
  WITH CHECK (auth.uid() = user_id);
CREATE POLICY "manager_read" ON public.minha_tabela FOR SELECT TO authenticated
  USING (public.is_manager_or_admin(auth.uid()));

CREATE TRIGGER set_updated_at BEFORE UPDATE ON public.minha_tabela
  FOR EACH ROW EXECUTE FUNCTION public.set_updated_at();
```

---

## 10. AUTENTICAÇÃO E AUTORIZAÇÃO

### 10.1 Camadas
1. **Autenticação** (`auth.users` + Google OAuth via broker Lovable).
2. **Role** (`user_roles.role app_role`) — uma de: `cidadao`, `acs`, `profissional_saude`, `gestor`, `admin`.
3. **Permissão** (`has_permission(uid, key)`) — granular, com override por usuário e templates.
4. **Contexto ativo** (`active-context.tsx`) — ACS/profissional pode atuar sobre um cidadão.

### 10.2 Hard Rules
- **NUNCA** armazenar role em `profiles` ou `localStorage`.
- Toda checagem de admin chama `has_role(uid, 'admin')` SECURITY DEFINER.
- Frontend usa `usePermissions().can("permissao.x")` — nunca compara strings hardcoded.

---

## 11. WORKFLOW DE BANCO DE DADOS

### 11.1 Princípio
**Nenhuma mutação direta na Lovable Cloud via dashboard.** Toda alteração nasce como arquivo em `supabase/migrations/`, é aprovada via PR e aplicada via CLI.

### 11.2 Fluxo Diário
```bash
# 1. Subir Supabase local
pnpm db:start                 # supabase start (docker)

# 2. Criar migration
pnpm db:new add_client_uuid_to_vaccination
# → supabase/migrations/20260612120000_add_client_uuid_to_vaccination.sql

# 3. Aplicar local
pnpm db:reset                 # supabase db reset

# 4. Regenerar tipos
pnpm db:types                 # supabase gen types typescript --local > src/integrations/supabase/types.ts

# 5. Commit (lock + types + migration juntos)
git add supabase/migrations src/integrations/supabase/types.ts
```

### 11.3 Migration — Anatomia Obrigatória
1. `CREATE TABLE` ou `ALTER`
2. `GRANT` para `authenticated` + `service_role` (sempre)
3. `ALTER TABLE ... ENABLE ROW LEVEL SECURITY`
4. `CREATE POLICY` (mínimo SELECT owner)
5. Trigger `set_updated_at` se houver `updated_at`
6. Comentário no topo descrevendo intenção e ticket

**Validação automática:** script `scripts/validate-migrations.mjs` (rodado em pre-commit) garante presença de GRANT + RLS em todo `CREATE TABLE public.*`.

### 11.4 Tipagem Sincronizada
- `src/integrations/supabase/types.ts` é **gerado**, nunca editado à mão.
- CI falha se `pnpm db:types --check` detectar drift entre migrations e types comitados.

---

## 12. HARD GATES — HUSKY + LINT-STAGED + ESLINT CUSTOMIZADO

### 12.1 `.husky/pre-commit`
```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

pnpm exec lint-staged
pnpm exec tsc --noEmit
node scripts/validate-migrations.mjs
node scripts/validate-no-raw-hex.mjs
```

### 12.2 `.husky/commit-msg`
```bash
pnpm exec commitlint --edit "$1"
```

### 12.3 `lint-staged.config.js`
```js
export default {
  "*.{ts,tsx}": ["eslint --max-warnings 0 --fix", "prettier --write"],
  "*.{css,md,json}": ["prettier --write"],
  "supabase/migrations/*.sql": ["node scripts/validate-migration-file.mjs"],
};
```

### 12.4 ESLint Customizado (`eslint.config.js`)
Regras **hard-fail**:
```js
import noRestricted from "eslint-plugin-no-restricted-syntax";

export default [
  // Proibir hex em className
  {
    files: ["src/**/*.{ts,tsx}"],
    rules: {
      "no-restricted-syntax": ["error",
        {
          selector: "Literal[value=/(text|bg|border|ring|shadow|from|to|via)-\\[#[0-9a-fA-F]{3,8}/]",
          message: "Hex literal em className é proibido. Use tokens semânticos em styles.css.",
        },
        {
          selector: "Literal[value=/shadow-\\[/]",
          message: "Shadow arbitrário proibido. Use --shadow-soft / --shadow-floating.",
        },
        {
          selector: "TSAnyKeyword",
          message: "Tipo 'any' proibido. Use unknown + narrow, ou tipos do Supabase.",
        },
      ],
      "@typescript-eslint/no-explicit-any": "error",
      "@typescript-eslint/ban-ts-comment": ["error", { "ts-ignore": true, "ts-expect-error": "allow-with-description" }],
    },
  },
  // Proibir client.server fora de paths server-only
  {
    files: ["src/components/**", "src/hooks/**", "src/routes/**/!(*.functions).{ts,tsx}"],
    rules: {
      "no-restricted-imports": ["error", {
        patterns: [{
          group: ["@/integrations/supabase/client.server", "**/*.server"],
          message: "Server-only. Use createServerFn ou requireSupabaseAuth.",
        }],
      }],
    },
  },
];
```

### 12.5 Scripts de Validação
- `scripts/validate-no-raw-hex.mjs` — varre `src/**/*.{ts,tsx}` por `#[0-9a-fA-F]{3,8}` em string literals e bloqueia.
- `scripts/validate-migrations.mjs` — toda migration com `CREATE TABLE public.*` deve conter `GRANT` e `ENABLE ROW LEVEL SECURITY` no mesmo arquivo.
- `scripts/audit-ui-theme.mjs` — já existente; integrar ao pre-push.

---

## 13. PIPELINE CI/CD

### 13.1 `.github/workflows/ci.yml` (esqueleto canônico)
```yaml
name: CI
on:
  pull_request:
  push: { branches: [main] }

jobs:
  install:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9.12.0 }
      - uses: actions/setup-node@v4
        with: { node-version: 20.11, cache: pnpm }
      - run: pnpm install --frozen-lockfile

  lint:
    needs: install
    runs-on: ubuntu-latest
    steps:
      - run: pnpm lint
      - run: node scripts/validate-no-raw-hex.mjs
      - run: node scripts/validate-migrations.mjs

  typecheck:
    needs: install
    runs-on: ubuntu-latest
    steps:
      - run: pnpm exec tsc --noEmit

  test:
    needs: install
    runs-on: ubuntu-latest
    steps:
      - run: pnpm test --coverage
      - uses: codecov/codecov-action@v4
      - name: Enforce coverage ≥ 80%
        run: node scripts/enforce-coverage.mjs 80

  e2e:
    needs: install
    runs-on: ubuntu-latest
    steps:
      - run: pnpm exec playwright install --with-deps
      - run: pnpm build
      - run: pnpm exec playwright test

  db-drift:
    needs: install
    runs-on: ubuntu-latest
    services:
      postgres: { image: supabase/postgres:15, ports: ["5432:5432"], env: { POSTGRES_PASSWORD: postgres } }
    steps:
      - run: pnpm db:migrate:ci
      - run: pnpm db:types:check    # falha se drift

  build:
    needs: [lint, typecheck, test, e2e, db-drift]
    runs-on: ubuntu-latest
    steps:
      - run: pnpm build
```

### 13.2 Branch Protection
- `main` exige todos os jobs verdes + 1 review + lockfile inalterado por bot.
- Merge tipo `squash` apenas; commits seguem Conventional Commits.

### 13.3 Cobertura Mínima
- **Vitest 80%** linhas/branches (helpers, hooks, stores, máquinas XState).
- **Playwright**: jornadas críticas obrigatórias:
  1. Login email/senha + Google.
  2. Esqueci/Reset de senha.
  3. Agendamento end-to-end (XState).
  4. Aplicação de vacina pelo profissional.
  5. ACS busca ativa offline → online sync.
  6. Gestor altera permissão → audit log gravado.

---

## 14. TELEMETRIA E OBSERVABILIDADE

### 14.1 Sentry — Bootstrap
```ts
// src/telemetry/sentry.ts
import * as Sentry from "@sentry/react";

export function initSentry() {
  if (!import.meta.env.PROD) return;
  Sentry.init({
    dsn: import.meta.env.VITE_SENTRY_DSN,
    environment: import.meta.env.VITE_ENV,
    release: import.meta.env.VITE_RELEASE,
    tracesSampleRate: 0.1,
    replaysSessionSampleRate: 0.0,
    replaysOnErrorSampleRate: 1.0,
    integrations: [
      Sentry.browserTracingIntegration(),
      Sentry.replayIntegration({ maskAllText: true, blockAllMedia: true }),
    ],
    beforeSend(event) {
      return maskPII(event);    // remove CPF, CNS, e-mail, telefone
    },
  });
}
```

### 14.2 ErrorBoundary obrigatório em toda rota
```tsx
errorComponent: ({ error, reset }) => {
  Sentry.captureException(error, { tags: { route: "agendamento" } });
  return <ModuleErrorFallback error={error} reset={reset} />;
}
```

### 14.3 Mascaramento de PII (`src/telemetry/mask.ts`)
- Regex CPF `\d{3}\.?\d{3}\.?\d{3}-?\d{2}` → `***.***.***-**`.
- E-mail → `***@dominio`.
- CNS (15 dígitos) → `••• ••• ••• ••••`.
- Aplicado em `beforeSend`, breadcrumbs e replay.

### 14.4 Métricas adicionais
- Web Vitals (CLS, LCP, INP) → Sentry Performance.
- Custom spans: `pglite.query`, `outbox.drain`, `xstate.transition`.

---

## 15. GESTÃO DE DEPENDÊNCIAS

- **pnpm exclusivo**. `preinstall` hook bloqueia npm/yarn:
```json
"scripts": { "preinstall": "npx only-allow pnpm" }
```
- Renovate/Dependabot configurado em grupos: `react`, `tanstack`, `supabase`, `testing`, `tooling`. PRs automáticas, mas merge manual após CI verde.
- Auditoria semanal: `pnpm audit --prod --audit-level=high` no CI.

---

## 16. SEGURANÇA APLICACIONAL

### 16.1 Headers (Cloudflare Workers — `src/server.ts`)
```
Content-Security-Policy: default-src 'self'; img-src 'self' data: https:; script-src 'self' 'wasm-unsafe-eval'; connect-src 'self' https://*.supabase.co https://*.sentry.io; style-src 'self' 'unsafe-inline'; frame-ancestors 'none'
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(self), camera=(), microphone=()
```
Violações CSP → endpoint `/api/public/csp-report` → Sentry.

### 16.2 Secrets
- Frontend: somente `VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY`, `VITE_SENTRY_DSN`, `VITE_ENV`, `VITE_RELEASE`.
- Backend: `SUPABASE_SERVICE_ROLE_KEY` (gerenciado pela Lovable Cloud), `WEBHOOK_SECRET`, `LOVABLE_API_KEY`.
- Nunca `console.log` de secrets. ESLint regra bloqueia `console.log(process.env...)`.

---

## 17. LGPD, AUDITORIA, MASCARAMENTO

### 17.1 `audit_logs` — Imutável
Trigger `prevent_audit_log_mutation` impede UPDATE/DELETE (exceto cascade SET NULL). Toda ação sensível chama `log_audit_event`.

### 17.2 Consentimento
- Tabela `lgpd_consents` (a criar): `user_id, policy_version, accepted_at, ip, user_agent`.
- Rota `/lgpd` força nova aceitação ao subir `policy_version`.

### 17.3 Mascaramento UI
- `formatCPF(cpf, { masked: true })` por padrão. Desmascarar exige `has_permission('pii.view_full')` + log de auditoria `patient_profile_viewed`.

---

## 18-19. MÓDULOS

Cada módulo segue contrato:
1. **Rota** `src/routes/_authenticated/modulos.<nome>.tsx`.
2. **Store XState/Zustand** se houver fluxo.
3. **Hooks de dados** em `src/lib/use-<recurso>.ts` (TanStack Query).
4. **Componentes** em `src/components/<modulo>/`.
5. **Cache offline** se o módulo é usado por ACS.
6. **Testes**: unit (hooks/stores) + e2e (jornada).

Tabela resumo: ver Anexo A (mantida da v1, atualizada com Nutrição completa).

---

## 20. UI-KIT — CONTRATOS

| Componente | Props críticas | Invariante |
|---|---|---|
| `StatusPill` | `tone: success\|warn\|danger\|info\|process` | nunca aceitar cor crua |
| `Sparkline` | `data: number[]`, `width`, `height` | dimensões fixas (CLS) |
| `GoalProgressBar` | `percent: 0..100`, `status` | clamp em range |
| `RulerPicker` | `value`, `min`, `max`, `step`, `onChange` | virtualizado |
| `CircleArrowButton` | `direction`, `onClick`, `aria-label` obrigatório | 44px mínimo |
| `PremiumDarkCard` | `as`, `radius` | rounded-[32px] padrão |
| `AskBar` | `onSubmit` | input controlado |

---

## 21. PERFORMANCE BUDGET

| Métrica | Meta | Bloqueio |
|---|---|---|
| LCP (mobile 4G) | < 2.5s | CI Lighthouse |
| INP | < 200ms | Sentry alerta |
| CLS | **0.00** | Visual regression Playwright |
| JS inicial (gzip) | < 180KB | `size-limit` |
| Largest route chunk | < 80KB | `size-limit` |

---

## 22. ACESSIBILIDADE — WCAG 2.2 AA

- Contraste mínimo 4.5:1 (verificado em `styles.css` via `npm:wcag-contrast`).
- Toda interação por teclado; focus ring usa `--color-ring`.
- `aria-label` obrigatório em botões-ícone (ESLint rule `jsx-a11y/control-has-associated-label`).
- Movimento respeita `prefers-reduced-motion`.

---

## 23. INTERNACIONALIZAÇÃO

- pt-BR como default; `react-intl` ou `lingui` se mais idiomas forem necessários.
- Datas: `date-fns` com `ptBR` locale. **Proibido** `toLocaleDateString` solto.

---

## 24. RUNBOOKS

### 24.1 Outbox poison message
1. Sentry alerta `outbox-poison` (>5 tentativas).
2. SRE inspeciona entry via IndexedDB DevTools.
3. Fix: corrige payload ou marca `dead_letter=true`.

### 24.2 Drift de tipos Supabase
1. CI `db-drift` falha.
2. Dev roda `pnpm db:types` localmente, commita.

### 24.3 Service Worker preso
1. Usuário com tela branca → acessar `?sw=off`.
2. Worker kill-switch desregistra e recarrega.

---

## 25. ANTI-HALLUCINATION PROTOCOL — 18 PONTOS (VERIFICÁVEIS EM CI)

1. ✅ Zero hex literal em `src/**/*.{ts,tsx}` (`validate-no-raw-hex`).
2. ✅ Zero `any` (ESLint).
3. ✅ Toda migration tem GRANT + RLS (`validate-migrations`).
4. ✅ Types Supabase sincronizados (CI `db-drift`).
5. ✅ Toda rota com loader tem `errorComponent` + `notFoundComponent` (regra ESLint custom).
6. ✅ Toda exceção captura Sentry antes do fallback (lint custom).
7. ✅ Touch targets ≥ 44px (Playwright a11y).
8. ✅ CLS = 0 (Playwright visual diff).
9. ✅ `useTransition` em toda troca de tab (lint custom).
10. ✅ Skeletons dimensionados em width+height (lint custom).
11. ✅ Sem `localStorage` para role/auth state (lint custom).
12. ✅ Sem `client.server` em componentes (ESLint `no-restricted-imports`).
13. ✅ Toda tabela offline-eligible tem `client_uuid UNIQUE`.
14. ✅ Cobertura Vitest ≥ 80%.
15. ✅ Jornadas críticas Playwright verdes.
16. ✅ Lockfile imutável por PR (CI `lockfile-lint`).
17. ✅ Sem ícones fora de `lucide-react`.
18. ✅ CSP headers presentes no response (smoke test).

---

## 26. GLOSSÁRIO E REFERÊNCIAS

- **ACS** — Agente Comunitário de Saúde.
- **CLS** — Cumulative Layout Shift.
- **PGlite** — Postgres compilado em WASM, persistido em IndexedDB.
- **Outbox** — fila local de mutações pendentes de sync.
- **SSoT** — Single Source of Truth.
- **RBAC** — Role-Based Access Control.

Referências externas: TanStack Start v1 docs, Supabase RLS guide, Sentry React SDK, WCAG 2.2, LGPD Lei 13.709/2018.

---

## ANEXO A — Catálogo Completo de Tabelas, Colunas-chave e Domínio

(ver `saude-para-todos-documentacao.md` v1 §4 — mantido por referência; este documento adiciona as colunas `client_uuid UUID UNIQUE` requeridas para offline em: `vaccination_records`, `chronic_medication_intakes`, `mental_health_logs`, `health_care_logs`, `chronic_care_logs`, `appointments`, `family_dependents`.)

## ANEXO B — Convenção de Commits

```
feat(agenda): adiciona máquina XState para agendamento multi-etapa
fix(offline): drena outbox ao reconectar
chore(deps): pin react 19.0.3
docs(arch): atualiza constitution v2
test(e2e): cobre jornada vacinação
```

## ANEXO C — Checklist de PR (template `.github/pull_request_template.md`)

- [ ] Sem hex literal novo
- [ ] Sem `any`
- [ ] Migration com GRANT + RLS (se houver)
- [ ] Tipos Supabase regenerados (se houver migration)
- [ ] Testes Vitest adicionados/atualizados
- [ ] Playwright atualizado (se jornada crítica)
- [ ] Sentry breadcrumbs/tags adicionados em novos fluxos
- [ ] Offline path coberto (se módulo ACS)
- [ ] Acessibilidade verificada (teclado + leitor de tela)
- [ ] Performance budget respeitado (`pnpm size`)
- [ ] Documentação atualizada

---

**FIM DO DOCUMENTO — v2.0**
Mudanças desta versão devem abrir RFC em `docs/rfcs/` antes de merge.

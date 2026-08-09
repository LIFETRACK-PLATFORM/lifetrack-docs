# Frontend — LifeTrack OS

> Contexto completo: [README principal](./README.md)

---

## Stack actual

| Tecnología | Uso |
|-----------|-----|
| Next.js 16 (App Router) | App monolítica — auth, onboarding, rehab |
| React 19 + TypeScript | UI |
| Tailwind CSS v4 | Tokens y utilities en `frontend/src/app/globals.css` |
| shadcn/ui (new-york) | Primitives accesibles (Radix) |
| next-themes | Modo dark / light (`attribute="class"`) |
| lucide-react | Iconografía (Nightframe) |
| Framer Motion | (cuando se use) microinteracciones |
| React Hook Form + Zod | Formularios tipados (cuando aplique) |

La arquitectura microfrontend descrita históricamente queda fuera del código actual; el frontend vive en `frontend/` como una sola app Next.

---

## Design system — Nightframe

Fuente de verdad:

- Reglas y tokens: [`frontend/DESIGN.md`](../frontend/DESIGN.md)
- Referencia visual (light + dark): [`frontend/docs/design-system/nightframe-reference.html`](../frontend/docs/design-system/nightframe-reference.html)
- Obligatorio para agentes: [`frontend/AGENTS.md`](../frontend/AGENTS.md)

Nightframe reemplaza al sistema anterior ("Clinical Vitality"). Personalidad: oscura, minimalista y premium; profundidad por contraste de superficies (sin sombras duras); acento único violeta `#7C5CFF`.

### Tokens clave

- Superficies: `background`, `surface-1`…`surface-5`, `border`
- Texto: `text-1`, `text-3`
- Marca: `primary`, `primary-hover`, `primary-active`, `accent-tint`
- Semánticos: `success`, `warning`, `error`
- Tipografía: Space Grotesk (headings), Manrope (body), JetBrains Mono (métricas)
- Tema: light en `:root`, dark bajo `.dark` — toggle con `ThemeToggle` / next-themes

### Componentes UI

Primitivos en `frontend/src/components/ui/` (button, input, card, dialog, badge, calendar, table, sheet, alert-dialog, etc.). Shell: `SideNav`, `BottomNav`, `ThemeToggle`. Iconos vía `shared/ui/Icon` (mapa lucide).

### Loading states — Skeletons

No se usa ninguna librería externa de skeleton/loading (`react-loading-skeleton`, `react-content-loader`, etc.). Alcanza con el primitivo `Skeleton` de shadcn/ui (`frontend/src/components/ui/skeleton.tsx`, `animate-pulse` + tokens Nightframe) — así el shimmer respeta los mismos colores/radios que el resto del design system en light y dark.

Patrón: cuando un hook expone `loading`, el `return` temprano arma un placeholder que **reproduce la forma real del layout** (mismos contenedores, mismo split mobile/web con `md:hidden` / `hidden md:block`) en vez de un texto tipo "Cargando…" o un spinner centrado. Esto evita saltos de layout (CLS) cuando llegan los datos reales.

```tsx
if (loading) {
  return <DashboardSkeleton />; // función local, misma estructura que el JSX real
}
```

Ejemplos: `ProfileView.tsx` (skeleton inline simple), `DashboardView.tsx` y `PlanDetailView.tsx` (componente `*Skeleton` local con variantes mobile/web).

No usar `Skeleton` para acciones puntuales (guardar, enviar un form) — ahí corresponde texto de estado en el botón (`disabled` + `"Guardando…"`) o un ícono `progress_activity` con `animate-spin`, como ya se hace en botones de submit.

---

## Estado — React Query vs Zustand

Cuando se introduzcan:

```
Server state (API)  →  React Query
Client state (UI)   →  Zustand

Nunca usar Zustand para datos del servidor.
```

---

## Auth en el frontend

- Access / refresh en cookies httpOnly (vía auth-service / gateway)
- Rutas públicas: login, register, forgot/reset password, confirm-email, link-account (vinculación OAuth)
- Login social: botones Google/GitHub redirigen al `api-gateway` (`/auth/google`, `/auth/github`); tras callback exitoso el gateway setea cookies y redirige a `/onboarding`; si el email ya existe como cuenta local, redirige a `/link-account` para confirmar con password
- Rutas autenticadas bajo `src/app/(authenticated)/` con guard de sesión

---

## Módulos UI actuales

| Módulo | Rutas / views |
|--------|----------------|
| Auth | Login, Register, Forgot/Reset, ConfirmEmail, LinkAccount (OAuth) |
| Onboarding | Selección de módulos |
| Rehab | Dashboard, PlanDetail + dialogs |

Módulos futuros (tasks, finance, vault, career) deben nacer ya con tokens Nightframe.

---

> Ver también: [Arquitectura](./ARCHITECTURE.md) · [Backend](./BACKEND.md) · [DevOps](./DEVOPS.md) · [CI/CD](./CICD.md)

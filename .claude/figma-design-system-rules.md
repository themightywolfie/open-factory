# OpenFactory Figma Design System Rules

> Rules for translating Figma designs into OpenFactory frontend code.
> Based on the OpenFactory Constitution v1.0.0 (2026-03-04).

## Project Context

OpenFactory is a **multi-tenant B2B SaaS** AI agent orchestrator.
The frontend is an **enterprise dashboard** — not a marketing site.
All UI serves agent orchestration, task monitoring, knowledge
management, and tenant administration workflows.

**Status**: Greenfield project. No existing frontend code.
All patterns defined here are prescriptive for initial build.

---

## 1. Frameworks & Libraries

| Concern | Technology | Notes |
|---------|-----------|-------|
| Framework | **Next.js 14+** (App Router) | React Server Components where possible |
| Agent UI | **CopilotKit** | AG-UI protocol for real-time agent interaction |
| Language | **TypeScript** (strict mode) | No `any` types in production code |
| Styling | **Tailwind CSS 4+** | Utility-first; no CSS-in-JS |
| Components | **shadcn/ui** | Copy-paste components; fully customizable |
| Icons | **Lucide React** | Consistent with shadcn/ui ecosystem |
| Forms | **React Hook Form + Zod** | Pydantic-compatible validation schemas |
| State | **Zustand** (client) / **TanStack Query** (server) | No Redux |
| SSE/Streaming | **CopilotKit hooks** | AG-UI event streams from FastAPI |

### Build & Tooling

```text
frontend/
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── package.json          # managed with pnpm
└── components.json       # shadcn/ui config
```

---

## 2. Project Structure

```text
frontend/
├── src/
│   ├── app/                    # Next.js App Router pages
│   │   ├── (auth)/             # Auth-gated routes (Zitadel OIDC)
│   │   │   ├── dashboard/      # Main dashboard
│   │   │   ├── agents/         # Agent management & dispatch
│   │   │   ├── knowledge/      # Knowledge base / document ingestion
│   │   │   ├── tasks/          # Task monitoring & results
│   │   │   └── settings/       # Tenant settings & admin
│   │   ├── login/              # Zitadel OIDC login flow
│   │   ├── layout.tsx          # Root layout with providers
│   │   └── globals.css         # Tailwind base + custom tokens
│   ├── components/
│   │   ├── ui/                 # shadcn/ui primitives (Button, Card, etc.)
│   │   ├── agents/             # Agent-specific components
│   │   ├── knowledge/          # Document ingestion components
│   │   ├── tasks/              # Task monitoring components
│   │   ├── chat/               # CopilotKit chat interface
│   │   └── layout/             # Shell, sidebar, header, nav
│   ├── lib/                    # Utilities, API client, helpers
│   │   ├── api.ts              # FastAPI client (fetch-based)
│   │   ├── auth.ts             # Zitadel OIDC helpers
│   │   ├── utils.ts            # cn() helper, formatters
│   │   └── constants.ts        # Route paths, API endpoints
│   ├── hooks/                  # Custom React hooks
│   ├── stores/                 # Zustand stores
│   ├── types/                  # TypeScript type definitions
│   └── styles/                 # Additional style files if needed
├── public/
│   ├── icons/                  # Static icon assets (SVG)
│   └── images/                 # Static images
└── tests/                      # Frontend tests (Vitest + Testing Library)
```

---

## 3. Token Definitions

Design tokens are defined as **CSS custom properties** in
`globals.css` following the shadcn/ui convention. Figma variables
MUST map to these tokens.

### Color Tokens

```css
/* frontend/src/app/globals.css */
@layer base {
  :root {
    /* Brand */
    --primary: 222.2 47.4% 11.2%;
    --primary-foreground: 210 40% 98%;

    /* Semantic */
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;
    --muted: 210 40% 96.1%;
    --muted-foreground: 215.4 16.3% 46.9%;
    --accent: 210 40% 96.1%;
    --accent-foreground: 222.2 47.4% 11.2%;
    --border: 214.3 31.8% 91.4%;
    --input: 214.3 31.8% 91.4%;
    --ring: 222.2 84% 4.9%;

    /* Status (agent task states) */
    --status-pending: 38 92% 50%;
    --status-running: 217 91% 60%;
    --status-success: 142 71% 45%;
    --status-failed: 0 84% 60%;
    --status-timeout: 25 95% 53%;

    /* Tenant accent — dynamically set per tenant */
    --tenant-accent: var(--primary);

    /* Spacing scale */
    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    /* ... dark mode equivalents */
  }
}
```

### Figma Variable Mapping

| Figma Variable | CSS Token | Usage |
|---------------|-----------|-------|
| `color/primary` | `--primary` | Primary brand color |
| `color/background` | `--background` | Page backgrounds |
| `color/foreground` | `--foreground` | Body text |
| `color/muted` | `--muted` | Secondary backgrounds |
| `color/border` | `--border` | Borders and dividers |
| `status/pending` | `--status-pending` | Agent task pending |
| `status/running` | `--status-running` | Agent task in progress |
| `status/success` | `--status-success` | Agent task completed |
| `status/failed` | `--status-failed` | Agent task failed |
| `spacing/radius` | `--radius` | Border radius base |

### Typography

```css
/* Use Inter as the primary font (enterprise clarity) */
--font-sans: 'Inter', system-ui, sans-serif;
--font-mono: 'JetBrains Mono', monospace;
```

| Figma Text Style | Tailwind Class | Size |
|-----------------|----------------|------|
| Heading 1 | `text-3xl font-bold tracking-tight` | 30px |
| Heading 2 | `text-2xl font-semibold tracking-tight` | 24px |
| Heading 3 | `text-xl font-semibold` | 20px |
| Body | `text-sm` | 14px |
| Body Large | `text-base` | 16px |
| Caption | `text-xs text-muted-foreground` | 12px |
| Code | `text-sm font-mono` | 14px |

---

## 4. Component Patterns

### When Figma has a component matching shadcn/ui

Use the existing shadcn/ui component. Do NOT recreate it.

```tsx
// CORRECT: use shadcn/ui Button
import { Button } from "@/components/ui/button"

<Button variant="default" size="sm">
  Dispatch Agent
</Button>

// FORBIDDEN: custom button from scratch when shadcn has one
```

### When Figma has a custom component

Create it in the appropriate feature directory under
`components/`. Use `cn()` for conditional class merging.

```tsx
// frontend/src/components/agents/agent-status-badge.tsx
import { cn } from "@/lib/utils"

interface AgentStatusBadgeProps {
  status: "pending" | "running" | "success" | "failed" | "timeout"
}

export function AgentStatusBadge({ status }: AgentStatusBadgeProps) {
  return (
    <span
      className={cn(
        "inline-flex items-center rounded-full px-2.5 py-0.5",
        "text-xs font-medium",
        {
          "bg-[hsl(var(--status-pending)/.1)] text-[hsl(var(--status-pending))]":
            status === "pending",
          "bg-[hsl(var(--status-running)/.1)] text-[hsl(var(--status-running))]":
            status === "running",
          "bg-[hsl(var(--status-success)/.1)] text-[hsl(var(--status-success))]":
            status === "success",
          "bg-[hsl(var(--status-failed)/.1)] text-[hsl(var(--status-failed))]":
            status === "failed",
          "bg-[hsl(var(--status-timeout)/.1)] text-[hsl(var(--status-timeout))]":
            status === "timeout",
        }
      )}
    >
      {status}
    </span>
  )
}
```

### Component Rules

1. **Server Components by default** — Only add `"use client"`
   when the component needs interactivity, hooks, or browser APIs
2. **Props over context** — Prefer explicit props; use context
   only for cross-cutting concerns (auth, tenant, theme)
3. **No inline styles** — Use Tailwind utilities exclusively
4. **Composition over inheritance** — Use children/slots pattern
5. **`tenant_id` awareness** — Components displaying tenant-scoped
   data MUST receive tenant context (from auth, not from URL)

---

## 5. Icon System

Use **Lucide React** icons (bundled with shadcn/ui). Do NOT use
Font Awesome, Heroicons, or custom SVG icon sets unless Lucide
lacks the needed icon.

```tsx
// CORRECT
import { Bot, FileText, Activity, Settings } from "lucide-react"

<Bot className="h-4 w-4" />

// For custom icons not in Lucide: place SVG in public/icons/
// and create a component wrapper
```

### Icon Naming in Figma

| Figma Icon Name | Lucide Equivalent |
|----------------|-------------------|
| `icon/agent` | `<Bot />` |
| `icon/document` | `<FileText />` |
| `icon/task` | `<Activity />` |
| `icon/settings` | `<Settings />` |
| `icon/tenant` | `<Building2 />` |
| `icon/knowledge` | `<BookOpen />` |
| `icon/dispatch` | `<Send />` |
| `icon/status-success` | `<CheckCircle2 />` |
| `icon/status-failed` | `<XCircle />` |
| `icon/status-pending` | `<Clock />` |

---

## 6. Styling Approach

### Tailwind CSS (utility-first)

- NO CSS Modules, NO Styled Components, NO CSS-in-JS
- Use `cn()` utility (from shadcn/ui) for conditional classes
- Use Tailwind `@apply` sparingly — only in `globals.css` for
  base element resets

```tsx
// cn() utility — frontend/src/lib/utils.ts
import { type ClassValue, clsx } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

### Responsive Design

Enterprise dashboard with a collapsible sidebar layout.
Primary breakpoints:

| Breakpoint | Tailwind | Usage |
|-----------|----------|-------|
| Mobile | `sm:` (640px) | Stacked layout, hidden sidebar |
| Tablet | `md:` (768px) | Collapsed sidebar (icons only) |
| Desktop | `lg:` (1024px) | Full sidebar, main content |
| Wide | `xl:` (1280px) | Multi-panel views |

```tsx
// Responsive pattern
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
  {agents.map(agent => (
    <AgentCard key={agent.id} agent={agent} />
  ))}
</div>
```

### Dark Mode

- Support `light` and `dark` themes via `class` strategy
  (Tailwind `darkMode: "class"`)
- Use semantic tokens (`--background`, `--foreground`) not
  raw colors
- Figma designs MUST provide both light and dark variants

---

## 7. Asset Management

```text
frontend/public/
├── icons/           # Custom SVG icons (only if Lucide lacks them)
├── images/
│   ├── logo.svg     # OpenFactory logo
│   └── empty-*.svg  # Empty state illustrations
└── fonts/           # Self-hosted Inter + JetBrains Mono (if needed)
```

- Use Next.js `<Image>` for all raster images (automatic optimization)
- SVG icons imported as React components or via `<img>` tag
- No external CDN dependencies — all assets self-hosted
  (enterprise/air-gap requirement from constitution)

---

## 8. CopilotKit / AG-UI Patterns

The chat and agent interaction UI uses CopilotKit. This is the
most distinctive part of the frontend.

```tsx
// frontend/src/components/chat/agent-chat.tsx
"use client"

import { CopilotKit } from "@copilotkit/react-core"
import { CopilotChat } from "@copilotkit/react-ui"

export function AgentChat() {
  return (
    <CopilotKit runtimeUrl="/api/copilotkit">
      <CopilotChat
        labels={{
          title: "OpenFactory Agent",
          initial: "How can I help you today?",
        }}
        className="h-full"
      />
    </CopilotKit>
  )
}
```

### Figma Design Rules for Agent UI

- Chat messages MUST show `agent_type` badge (which agent
  responded)
- Streaming responses use AG-UI SSE events — design for
  progressive rendering
- Tool call results (MCP server responses) rendered as
  collapsible cards
- Agent task status visible in real-time during execution
- All agent UI components MUST display `tenant_id` context
  in dev mode

---

## 9. Multi-Tenancy UI Rules (Constitution Principle III)

- **No tenant data in URLs** — tenant resolved from JWT claims
- **Tenant branding**: `--tenant-accent` CSS variable set
  dynamically from tenant config
- **Data isolation**: Frontend API client MUST include tenant
  context in every request (set via auth middleware, not by UI)
- **Tenant switcher**: Admin users with multi-tenant access see
  a tenant selector in the header

---

## 10. Auth UI Rules (Constitution Principle II)

- **Zitadel OIDC** login flow — redirect-based, not embedded
- **No credentials stored in frontend** — tokens managed via
  httpOnly cookies or Zitadel SDK
- **JWT token refresh** handled transparently by auth middleware
- **Role-based UI**: Hide UI elements based on Zitadel roles
  (`tenant_admin`, `agent_operator`, `agent_viewer`)

```tsx
// Role-based rendering pattern
import { useAuth } from "@/hooks/use-auth"

export function AdminPanel() {
  const { hasRole } = useAuth()

  if (!hasRole("tenant_admin")) return null

  return <div>Admin content...</div>
}
```

---

## 11. Key UI Patterns for Figma Designs

### Dashboard Layout

```text
┌──────────────────────────────────────────────┐
│  Header: Logo | Tenant Selector | User Menu  │
├──────────┬───────────────────────────────────┤
│          │                                   │
│ Sidebar  │  Main Content Area                │
│          │                                   │
│ - Dash   │  ┌─────────┐ ┌─────────┐         │
│ - Agents │  │ Card    │ │ Card    │         │
│ - Tasks  │  │ Widget  │ │ Widget  │         │
│ - Know.  │  └─────────┘ └─────────┘         │
│ - Config │                                   │
│          │  ┌───────────────────────┐        │
│          │  │ Agent Chat Panel      │        │
│          │  │ (CopilotKit)          │        │
│          │  └───────────────────────┘        │
├──────────┴───────────────────────────────────┤
│  Status Bar: Connection | Agent Activity     │
└──────────────────────────────────────────────┘
```

### Agent Task Card

```text
┌────────────────────────────────────┐
│ [Bot icon] CRM Agent    [RUNNING] │
│ Task: task_abc123                  │
│ Tenant: acme-corp                  │
│ Provider: Azure OpenAI (gpt-4o)   │
│ Duration: 4.5s                     │
│ Tokens: 1,200 in / 350 out        │
│ ──────────────────────────────     │
│ [View Details] [View in Langfuse]  │
└────────────────────────────────────┘
```

### Document Ingestion Card

```text
┌────────────────────────────────────┐
│ [FileText icon] quarterly-report   │
│ Format: PDF | Chunks: 42           │
│ Collection: acme_crm_reports       │
│ Status: [SUCCESS]                  │
│ ──────────────────────────────     │
│ [Re-ingest] [View Chunks]          │
└────────────────────────────────────┘
```

---

## 12. Translation Rules (Figma → Code)

### DO

- Map Figma auto-layout to Tailwind `flex` or `grid`
- Map Figma fill-container to `w-full` or `flex-1`
- Map Figma fixed dimensions to exact Tailwind values
- Map Figma corner radius to `rounded-*` using `--radius` token
- Map Figma shadows to Tailwind `shadow-*` utilities
- Use semantic color tokens, not raw hex values
- Use shadcn/ui components when Figma matches standard patterns

### DO NOT

- Do NOT use absolute positioning unless the design requires
  overlays/modals/tooltips
- Do NOT use pixel values — convert to Tailwind spacing scale
  (4px = 1 unit)
- Do NOT create custom CSS files for component styles
- Do NOT hard-code colors — always use CSS variables
- Do NOT ignore dark mode — every component must work in both
- Do NOT create new components for patterns that shadcn/ui
  already provides (Dialog, Sheet, Popover, DropdownMenu, etc.)

### Figma-to-Tailwind Spacing Map

| Figma (px) | Tailwind | CSS |
|-----------|----------|-----|
| 4 | `p-1` / `gap-1` | 0.25rem |
| 8 | `p-2` / `gap-2` | 0.5rem |
| 12 | `p-3` / `gap-3` | 0.75rem |
| 16 | `p-4` / `gap-4` | 1rem |
| 20 | `p-5` / `gap-5` | 1.25rem |
| 24 | `p-6` / `gap-6` | 1.5rem |
| 32 | `p-8` / `gap-8` | 2rem |
| 48 | `p-12` / `gap-12` | 3rem |

---

## 13. Accessibility Requirements

- All interactive elements MUST have visible focus indicators
- Color contrast MUST meet WCAG 2.1 AA (4.5:1 body, 3:1 large)
- Status information MUST NOT rely solely on color (use icons +
  text alongside status colors)
- All images and icons MUST have appropriate alt text or
  `aria-label`
- Keyboard navigation MUST work for all interactive flows
- shadcn/ui components include accessibility by default — do NOT
  remove `aria-*` attributes

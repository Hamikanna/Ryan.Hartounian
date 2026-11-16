# /claude-ui-dashboard — Dashboard Layout Architect

Turn a list of metrics and data into a proper information hierarchy before writing
any code. AI defaults to 20 identical gray cards in a grid — this skill classifies
each data point by type and importance, applies dashboard layout principles, outputs
an ASCII wireframe with visual weights, then generates the component structure.

---

## Usage

```
/claude-ui-dashboard "<brief>"
```

Examples:
- `/claude-ui-dashboard "SaaS analytics: MRR, churn rate, active users, revenue chart, top customers table"`
- `/claude-ui-dashboard "E-commerce ops: orders today, revenue, conversion rate, recent orders, inventory alerts"`
- `/claude-ui-dashboard "HR portal: headcount, open roles, time-to-hire, department breakdown, recent hires"`
- `/claude-ui-dashboard "DevOps: deploy frequency, error rate, p99 latency, recent deployments, alert feed"`
- `/claude-ui-dashboard "Admin panel: users, subscriptions, support tickets, activity log, system health"`

Options:
- `--library tremor`      — use Tremor v4 components (default if detected in package.json)
- `--library recharts`    — use Recharts for charts
- `--library shadcn`      — shadcn-only (no chart library)
- `--mobile-first`        — prioritize single-column mobile layout over desktop grid
- `--density compact`     — higher information density (smaller cards, tighter spacing)
- `--density comfortable` — more whitespace, executive-style presentation (default)
- `--code`                — generate full component code after the wireframe is approved

---

## Phase 1 — Data Classification

Parse every metric and data source from the brief. Classify each into one of five types.

### Data type taxonomy

| Type | Description | Examples | Visual treatment |
|------|-------------|---------|-----------------|
| **KPI Scalar** | Single number with context | Revenue, Active users, Churn % | Large number + trend arrow + sparkline |
| **Time Series** | Values over time | Revenue over 30 days, DAU trend | Line/area chart |
| **Categorical Breakdown** | Distribution across categories | Revenue by channel, Users by plan | Bar chart or donut |
| **Tabular** | List of records | Recent orders, Top customers, Activity log | Data table with pagination |
| **Status/Alert** | System state or action needed | Error rate threshold, Low inventory, Open tickets | Alert banner or badge list |

### Importance classification

Within each type, assign importance:
- **Primary** — the number the executive looks at first; defines success
- **Secondary** — supporting context; explains the primary
- **Tertiary** — drill-down detail; available but not leading

Example classification for "SaaS analytics dashboard":
```
DATA CLASSIFICATION
===================
Primary KPIs (scalar):
  → MRR              [Primary]   single number, trend from last month
  → Active users     [Primary]   single number, trend
  → Churn rate       [Secondary] single %, context for MRR

Time series:
  → Revenue chart    [Primary]   30-day line chart, most real estate
  → User growth      [Secondary] smaller, below revenue

Categorical:
  → Revenue by plan  [Secondary] donut or horizontal bar

Tabular:
  → Top customers    [Tertiary]  table with ARR, status, last active
  → Recent events    [Tertiary]  activity log, latest 10 rows

Status:
  → (none detected)
```

---

## Phase 2 — Layout Principles

Apply these rules to arrange the classified data:

### Scan path rule
Users scan dashboards in an F-pattern: top row → left column → right details.
- Place Primary KPIs in the top row — first thing seen
- Place primary time series below KPIs — occupies the most horizontal space
- Secondary context fills right column or second row
- Tables go at the bottom — require focused reading, not glancing

### Visual weight hierarchy
```
Size → importance. The most important metric gets the most space.

Primary KPIs:      span 1 column each, large text (3xl+), full height card
Primary chart:     span 2–3 columns, tall (300–400px)
Secondary charts:  span 1–2 columns, medium height (200–250px)
Tables:            full width, variable height
Status/alerts:     full width banner at top (if critical) or inline badge
```

### Grid system
Use a 12-column grid. Map data importance to column spans:
```
4 Primary KPIs:    3 cols each = 12 total (one full row)
1 Primary chart:   8 cols
1 Secondary chart: 4 cols
1 Table:           12 cols (full width)
```

### Responsive collapse strategy
```
Desktop (lg):  12-col grid, full layout
Tablet (md):   6-col grid, KPIs 3 cols each (2 per row), charts full width
Mobile (sm):   1-col stack, KPIs first → charts → tables (tables scroll horizontally)
```

---

## Phase 3 — Layout Wireframe

Output an ASCII wireframe before any code. Show the grid, the column spans, and the visual weight.

```
DASHBOARD WIREFRAME — SaaS Analytics
======================================
Desktop (1280px, 12-col grid)

┌────────────┬────────────┬────────────┬────────────┐
│   MRR      │ Active     │ Churn      │ New MRR    │
│   $48.2k   │ Users      │ Rate       │ This Mo.   │
│   ↑ 12%    │ 2,847 ↑8%  │ 2.1% ↓0.3 │ $6.4k ↑5% │
│   [3 cols] │ [3 cols]   │ [3 cols]   │ [3 cols]   │
├────────────────────────────┬─────────────────────┤
│   Revenue (30 days)        │  Revenue by Plan    │
│                            │                     │
│   [area chart, tall]       │  [donut or h-bar]   │
│                            │                     │
│   [8 cols]                 │  [4 cols]           │
├────────────────────────────┴─────────────────────┤
│   Top Customers                                   │
│   Name | ARR | Plan | Status | Last Active       │
│   ─────────────────────────────────────────────  │
│   [table with pagination, 5 rows]                 │
│   [12 cols]                                       │
└──────────────────────────────────────────────────┘

Mobile (320px, 1-col stack)
┌─────────────────┐
│ MRR: $48.2k ↑   │
├─────────────────┤
│ Active: 2,847 ↑ │
├─────────────────┤
│ Churn: 2.1% ↓   │
├─────────────────┤
│ Revenue chart   │
│ (scroll down)   │
├─────────────────┤
│ Revenue by plan │
├─────────────────┤
│ Top customers   │
│ (h-scroll)      │
└─────────────────┘
```

---

## Phase 4 — Component Specification

Map each layout slot to specific shadcn/Tremor components.

```
COMPONENT SPECS
===============
KPI Cards (top row × 4):
  Component:  shadcn Card + custom KpiCard pattern
  Content:    metric name, value (text-3xl font-semibold), trend (↑ green / ↓ red), sparkline
  Code hint:  <Card className="p-6"><p className="text-sm text-muted-foreground">MRR</p>
              <p className="text-3xl font-semibold">$48.2k</p>
              <p className="text-sm text-green-600">↑ 12% from last month</p></Card>

Primary Chart (8 cols):
  Component:  Tremor AreaChart  OR  Recharts AreaChart + ResponsiveContainer
  Data shape: { date: string, revenue: number }[]
  Props:      data, index="date", categories={["revenue"]}, colors={["blue"]}

Secondary Chart (4 cols):
  Component:  Tremor DonutChart  OR  Recharts PieChart
  Data shape: { name: string, value: number }[]
  
Customers Table (12 cols):
  Component:  shadcn Table  OR  TanStack Table (if project uses it)
  Columns:    name, arr, plan (Badge), status (Badge), lastActive (relative time)
  Features:   sorting, 5-row pagination
```

---

## Phase 5 — Generate Code (with `--code` flag)

When `--code` is specified, generate the full dashboard component:

```tsx
// app/dashboard/page.tsx (Server Component)
import { KpiCard } from "@/components/dashboard/KpiCard"
import { RevenueChart } from "@/components/dashboard/RevenueChart"
import { PlanBreakdown } from "@/components/dashboard/PlanBreakdown"
import { TopCustomersTable } from "@/components/dashboard/TopCustomersTable"

export default async function DashboardPage() {
  // Fetch in parallel — no waterfall
  const [kpis, revenueHistory, planBreakdown, topCustomers] = await Promise.all([
    fetchKpis(),
    fetchRevenueHistory(),
    fetchPlanBreakdown(),
    fetchTopCustomers(),
  ])

  return (
    <div className="flex flex-col gap-6 p-6 max-w-7xl mx-auto">
      
      {/* Row 1: Primary KPIs */}
      <div className="grid grid-cols-2 gap-4 md:grid-cols-4">
        <KpiCard label="MRR" value={kpis.mrr} trend={kpis.mrrTrend} format="currency" />
        <KpiCard label="Active Users" value={kpis.activeUsers} trend={kpis.usersTrend} />
        <KpiCard label="Churn Rate" value={kpis.churnRate} trend={kpis.churnTrend} format="percent" invertTrend />
        <KpiCard label="New MRR" value={kpis.newMrr} trend={kpis.newMrrTrend} format="currency" />
      </div>

      {/* Row 2: Primary chart + breakdown */}
      <div className="grid grid-cols-1 gap-4 lg:grid-cols-3">
        <div className="lg:col-span-2">
          <RevenueChart data={revenueHistory} />
        </div>
        <PlanBreakdown data={planBreakdown} />
      </div>

      {/* Row 3: Detail table */}
      <TopCustomersTable data={topCustomers} />

    </div>
  )
}
```

**KpiCard component** (generated):
```tsx
// components/dashboard/KpiCard.tsx
import { Card, CardContent } from "@/components/ui/card"
import { TrendingUp, TrendingDown, Minus } from "lucide-react"
import { cn } from "@/lib/utils"

interface KpiCardProps {
  label: string
  value: number | string
  trend?: number          // % change, positive = up
  format?: "number" | "currency" | "percent"
  invertTrend?: boolean   // for metrics where down is good (churn, errors)
}

export function KpiCard({ label, value, trend, format = "number", invertTrend }: KpiCardProps) {
  const isPositive = invertTrend ? (trend ?? 0) < 0 : (trend ?? 0) > 0
  const isNeutral = trend === 0 || trend === undefined

  const formatted = format === "currency"
    ? `$${(Number(value) / 1000).toFixed(1)}k`
    : format === "percent"
    ? `${value}%`
    : value.toLocaleString()

  return (
    <Card>
      <CardContent className="pt-6">
        <p className="text-sm text-muted-foreground">{label}</p>
        <p className="mt-1 text-3xl font-semibold tracking-tight">{formatted}</p>
        {trend !== undefined && (
          <p className={cn(
            "mt-1 flex items-center gap-1 text-sm",
            isNeutral && "text-muted-foreground",
            isPositive && "text-green-600 dark:text-green-400",
            !isPositive && !isNeutral && "text-red-600 dark:text-red-400"
          )}>
            {isNeutral ? <Minus className="h-3 w-3" /> :
             isPositive ? <TrendingUp className="h-3 w-3" /> :
             <TrendingDown className="h-3 w-3" />}
            {Math.abs(trend)}% from last period
          </p>
        )}
      </CardContent>
    </Card>
  )
}
```

---

## Layout Patterns by Dashboard Type

Quick-start templates — specify the domain and get the right layout:

### Business / SaaS metrics
```
Row 1: Revenue | Customers | Churn | ARPU          [4 KPIs, 3-col each]
Row 2: Revenue trend (8 cols) | Plan mix (4 cols)
Row 3: Top accounts table (12 cols)
```

### E-commerce operations
```
Row 1: Orders today | Revenue | Conversion | AOV   [4 KPIs]
Row 2: Orders over time (8 cols) | Category split (4 cols)
Row 3: Recent orders table (7 cols) | Inventory alerts (5 cols)
```

### DevOps / Engineering
```
Row 1: Deploy freq | Error rate | p99 latency | Uptime  [4 KPIs, error rate alert-colored if >1%]
Row 2: Error rate over time (12 cols, prominent — this is the primary concern)
Row 3: Recent deploys (6 cols) | Active alerts (6 cols)
```

### HR / People ops
```
Row 1: Headcount | Open roles | TTH avg | Retention   [4 KPIs]
Row 2: Headcount by dept (6 cols) | Hire pipeline (6 cols)
Row 3: Recent hires table (12 cols)
```

### Content / Marketing
```
Row 1: Pageviews | Sessions | Bounce rate | Top source  [4 KPIs]
Row 2: Traffic over time (8 cols) | Channel split (4 cols)
Row 3: Top pages table (12 cols)
```

---

## Hard Rules

1. **Never generate flat card grids** — every dashboard must have at least 2 visual weight levels. If all cards are the same size, the hierarchy is wrong.
2. **KPI scalars always go first** — the number the stakeholder cares about most must be visible without scrolling on desktop.
3. **Tables go last** — they require focused reading. Place at the bottom after the at-a-glance layer.
4. **Charts need titles and axis labels** — never `<AreaChart data={data} />` without a `<CardHeader>` with a title and the time range.
5. **Trend arrows must be colored** — green for positive, red for negative. Use `invertTrend` for metrics where down is good (churn, errors, latency).
6. **Mobile: collapse tables to horizontal scroll, never hide them** — use `overflow-x-auto` on table wrapper, not responsive hiding.
7. **Parallel data fetching** — always `Promise.all()` for multiple fetches. Never waterfall in a dashboard page.

---

## Relationship to Other Skills

```
/claude-ui-dashboard   → THIS SKILL — data → layout architecture → dashboard code
/claude-ui-wireframe   → generic ASCII wireframes for any UI (not data-aware)
/claude-ui-plan        → feature implementation planning (not layout-focused)
/claude-ui-empty       → generate empty states for each dashboard section
/claude-ui-loading     → generate skeleton for the dashboard while data loads
/claude-ui-contrast    → verify KPI number colors pass WCAG (red/green on card backgrounds)
```

### Full dashboard build sequence:
1. `/claude-ui-dashboard "brief"` → layout + wireframe
2. `/claude-ui-dashboard "brief" --code` → generate components
3. `/claude-ui-loading app/dashboard/page.tsx` → generate dashboard skeleton
4. `/claude-ui-empty "analytics dashboard"` → empty state for no-data state
5. `/claude-ui-contrast src/components/dashboard/` → verify trend colors pass WCAG

---

## When to Use

- **Before building any business dashboard** — get the layout right first, then code
- **When a client says "the dashboard needs to be cleaner"** — run the classification phase to find hierarchy issues
- **When adding a new metric to an existing dashboard** — ask "what type is this and where does it go?"
- **After inheriting a poorly structured dashboard** — classify what's there, identify what's buried that should be prominent

## What This Is Not

- Not a charting library — use Tremor, Recharts, or Visx for chart primitives
- Not a real-time data connector — generates structure, not WebSocket subscriptions
- Not a no-code dashboard builder — outputs React + Tailwind, requires a developer
- Not a data analysis tool — does not infer what metrics matter; takes brief at face value

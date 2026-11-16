# /claude-ui-chart — Data Visualization Component Generator

Generate production-ready chart components from data types or descriptions. AI
generating charts has three consistent failures: forgetting `ResponsiveContainer`
(chart renders as 0px), picking the wrong chart type for the data shape, and
hardcoding hex colors instead of design system tokens. This skill reads your data
type, selects the correct chart variant, generates a fully-configured shadcn chart
with responsive container, ChartConfig tokens, formatters, loading skeleton, and
empty state — ready to drop into any dashboard.

---

## Usage

```
/claude-ui-chart "<data description or TypeScript type>"
```

Examples:
- `/claude-ui-chart "monthly revenue over 12 months"`
- `/claude-ui-chart "user signups by channel: organic, paid, referral"`
- `/claude-ui-chart "order status breakdown: pending, processing, shipped, delivered"`
- `/claude-ui-chart "PageView type with date and views count"`
- `/claude-ui-chart "MultiMetric: revenue + expenses over time"`

Options:
- `/claude-ui-chart --type line`       — force line chart
- `/claude-ui-chart --type bar`        — force bar chart (vertical)
- `/claude-ui-chart --type bar-h`      — force horizontal bar chart
- `/claude-ui-chart --type area`       — force area chart (stacked)
- `/claude-ui-chart --type pie`        — force pie/donut chart
- `/claude-ui-chart --type radial`     — force radial bar chart
- `/claude-ui-chart --library recharts` — use Recharts directly (default: shadcn charts)
- `/claude-ui-chart --no-skeleton`     — skip loading skeleton generation
- `/claude-ui-chart --no-empty`        — skip empty state generation

---

## Phase 1 — Chart Type Selection

### Decision tree

```
Is the data a single value changing OVER TIME?
  YES → Line chart (shows trend)
  → AreaChart if you want to show volume/magnitude

Is the data COMPARING DISCRETE CATEGORIES at a point in time?
  YES → Bar chart (vertical if ≤ 8 categories, horizontal if labels are long)

Is the data PARTS OF A WHOLE (percentages that sum to ~100%)?
  YES → Pie or donut chart (limit to ≤ 7 segments — use "Other" for the rest)
  → Donut preferred: more readable at small sizes, room for center label

Are you COMPARING multiple metrics OVER TIME?
  YES → Multi-series line chart (up to 3 lines max without legend confusion)
  → Stacked area chart if the total is meaningful

Is the data a SINGLE KPI with a target?
  YES → Radial bar chart or gauge (shadcn RadialBarChart)

Is the data a DISTRIBUTION (histogram)?
  YES → Bar chart with equal-width bins

Does the data have NEGATIVE VALUES?
  YES → Bar chart with zero line (line charts handle negatives poorly visually)
```

**Anti-patterns to flag before generating:**
- Pie chart with > 7 segments → suggest bar chart or grouped pie with "Other"
- Time-series data → never pie chart
- More than 3 lines on a line chart → suggest table or small multiples
- Bar chart where bars all look the same height → data might need log scale

---

## Phase 2 — ChartConfig (color tokens)

Always generate `ChartConfig` using CSS variable tokens — never hardcode hex values.

```typescript
// Correct — uses shadcn chart CSS variables (adapts to dark mode automatically)
const chartConfig = {
  revenue: {
    label: "Revenue",
    color: "hsl(var(--chart-1))",
  },
  expenses: {
    label: "Expenses",
    color: "hsl(var(--chart-2))",
  },
} satisfies ChartConfig

// Wrong — hardcoded hex, breaks dark mode
const badConfig = {
  revenue: { label: "Revenue", color: "#6366f1" },
}
```

**shadcn chart CSS variables** (defined in `globals.css`):
```css
/* Light mode */
:root {
  --chart-1: 220 70% 50%;   /* blue */
  --chart-2: 160 60% 45%;   /* green */
  --chart-3: 30 80% 55%;    /* orange */
  --chart-4: 280 65% 60%;   /* purple */
  --chart-5: 340 75% 55%;   /* pink */
}

/* Dark mode */
.dark {
  --chart-1: 220 70% 60%;
  --chart-2: 160 60% 55%;
  --chart-3: 30 80% 65%;
  --chart-4: 280 65% 70%;
  --chart-5: 340 75% 65%;
}
```

Add to `globals.css` if not already present. `ChartContainer` reads these automatically.

---

## Phase 3A — Line Chart (time-series trend)

```tsx
// components/RevenueChart.tsx
"use client"

import {
  LineChart, Line, XAxis, YAxis, CartesianGrid,
  ResponsiveContainer,
} from "recharts"
import {
  ChartContainer,
  ChartTooltip,
  ChartTooltipContent,
  ChartLegend,
  ChartLegendContent,
  type ChartConfig,
} from "@/components/ui/chart"
import { Skeleton } from "@/components/ui/skeleton"

interface DataPoint {
  month: string       // "Jan", "Feb", ...
  revenue: number
  expenses: number
}

interface RevenueChartProps {
  data: DataPoint[]
  isLoading?: boolean
}

const chartConfig = {
  revenue: { label: "Revenue",  color: "hsl(var(--chart-1))" },
  expenses: { label: "Expenses", color: "hsl(var(--chart-2))" },
} satisfies ChartConfig

export function RevenueChart({ data, isLoading }: RevenueChartProps) {
  if (isLoading) return <RevenueChartSkeleton />
  if (!data.length) return <RevenueChartEmpty />

  return (
    <div className="flex flex-col gap-4">
      <div>
        <h3 className="text-heading-md font-semibold">Revenue vs Expenses</h3>
        <p className="text-body-sm text-muted-foreground">Monthly overview for the last 12 months</p>
      </div>

      {/* ChartContainer handles: responsive sizing, ChartConfig context, ARIA label */}
      <ChartContainer config={chartConfig} className="h-[300px] w-full">
        {/* ResponsiveContainer is included inside ChartContainer — don't add another one */}
        <LineChart data={data} margin={{ top: 4, right: 12, bottom: 4, left: 12 }}>
          <CartesianGrid
            strokeDasharray="3 3"
            stroke="hsl(var(--border))"
            vertical={false}  // horizontal gridlines only — cleaner
          />
          <XAxis
            dataKey="month"
            tickLine={false}
            axisLine={false}
            tick={{ fill: "hsl(var(--muted-foreground))", fontSize: 12 }}
          />
          <YAxis
            tickLine={false}
            axisLine={false}
            tick={{ fill: "hsl(var(--muted-foreground))", fontSize: 12 }}
            tickFormatter={v => `$${(v / 1000).toFixed(0)}k`}
            width={48}
          />
          <ChartTooltip
            content={
              <ChartTooltipContent
                formatter={(value) => `$${Number(value).toLocaleString()}`}
                labelFormatter={(label) => `${label} 2025`}
              />
            }
          />
          <ChartLegend content={<ChartLegendContent />} />
          <Line
            dataKey="revenue"
            type="monotone"   // smooth curve — use "linear" for step data
            stroke="var(--color-revenue)"
            strokeWidth={2}
            dot={false}       // dots clutter busy time-series; add back for < 7 points
            activeDot={{ r: 4, fill: "var(--color-revenue)" }}
          />
          <Line
            dataKey="expenses"
            type="monotone"
            stroke="var(--color-expenses)"
            strokeWidth={2}
            dot={false}
            activeDot={{ r: 4, fill: "var(--color-expenses)" }}
            strokeDasharray="4 2"  // dashed for secondary series
          />
        </LineChart>
      </ChartContainer>
    </div>
  )
}
```

---

## Phase 3B — Bar Chart (category comparison)

```tsx
// components/SignupsByChannelChart.tsx
"use client"

import { BarChart, Bar, XAxis, YAxis, CartesianGrid, LabelList } from "recharts"
import {
  ChartContainer, ChartTooltip, ChartTooltipContent, type ChartConfig,
} from "@/components/ui/chart"

interface ChannelData {
  channel: string    // "Organic", "Paid", "Referral", "Direct"
  signups: number
}

const chartConfig = {
  signups: { label: "Signups", color: "hsl(var(--chart-1))" },
} satisfies ChartConfig

export function SignupsByChannelChart({ data, isLoading }: { data: ChannelData[]; isLoading?: boolean }) {
  if (isLoading) return <Skeleton className="h-[300px] w-full rounded-lg" />
  if (!data.length) return <EmptyChart label="No signup data" />

  const total = data.reduce((sum, d) => sum + d.signups, 0)

  return (
    <div className="flex flex-col gap-4">
      <div className="flex items-baseline justify-between">
        <h3 className="text-heading-md font-semibold">Signups by Channel</h3>
        <span className="text-heading-lg font-bold">{total.toLocaleString()}</span>
      </div>
      <ChartContainer config={chartConfig} className="h-[300px] w-full">
        <BarChart data={data} margin={{ top: 16, right: 8, bottom: 4, left: 8 }}>
          <CartesianGrid strokeDasharray="3 3" stroke="hsl(var(--border))" vertical={false} />
          <XAxis
            dataKey="channel"
            tickLine={false}
            axisLine={false}
            tick={{ fill: "hsl(var(--muted-foreground))", fontSize: 12 }}
          />
          <YAxis
            tickLine={false}
            axisLine={false}
            tick={{ fill: "hsl(var(--muted-foreground))", fontSize: 12 }}
            width={40}
          />
          <ChartTooltip
            cursor={{ fill: "hsl(var(--muted))" }}  // subtle hover highlight
            content={<ChartTooltipContent />}
          />
          <Bar
            dataKey="signups"
            fill="var(--color-signups)"
            radius={[4, 4, 0, 0]}  // rounded top corners only
            maxBarSize={72}
          >
            {/* Value labels above bars */}
            <LabelList
              dataKey="signups"
              position="top"
              offset={4}
              style={{ fill: "hsl(var(--foreground))", fontSize: 11, fontWeight: 500 }}
            />
          </Bar>
        </BarChart>
      </ChartContainer>
    </div>
  )
}
```

---

## Phase 3C — Donut Chart (parts of a whole)

```tsx
// components/OrderStatusChart.tsx
"use client"

import { PieChart, Pie, Cell } from "recharts"
import {
  ChartContainer, ChartTooltip, ChartTooltipContent,
  ChartLegend, ChartLegendContent, type ChartConfig,
} from "@/components/ui/chart"

interface StatusData {
  status: string    // "Pending", "Processing", "Shipped", "Delivered"
  count: number
}

const chartConfig = {
  pending:    { label: "Pending",    color: "hsl(var(--chart-1))" },
  processing: { label: "Processing", color: "hsl(var(--chart-2))" },
  shipped:    { label: "Shipped",    color: "hsl(var(--chart-3))" },
  delivered:  { label: "Delivered",  color: "hsl(var(--chart-4))" },
} satisfies ChartConfig

export function OrderStatusChart({ data, isLoading }: { data: StatusData[]; isLoading?: boolean }) {
  if (isLoading) return <Skeleton className="h-[280px] w-full rounded-lg" />
  if (!data.length) return <EmptyChart label="No orders" />

  const total = data.reduce((sum, d) => sum + d.count, 0)

  return (
    <div className="flex flex-col gap-4">
      <h3 className="text-heading-md font-semibold">Order Status</h3>
      <ChartContainer config={chartConfig} className="h-[280px] w-full">
        <PieChart>
          <ChartTooltip content={<ChartTooltipContent nameKey="status" />} />
          <Pie
            data={data}
            dataKey="count"
            nameKey="status"
            cx="50%"
            cy="50%"
            innerRadius="55%"   // creates the donut hole
            outerRadius="80%"
            paddingAngle={2}    // small gap between segments
            strokeWidth={0}
          >
            {data.map((entry) => (
              <Cell
                key={entry.status}
                fill={`var(--color-${entry.status.toLowerCase()})`}
              />
            ))}
          </Pie>
          {/* Center label showing total — unique to donut charts */}
          <text x="50%" y="50%" textAnchor="middle" dominantBaseline="middle">
            <tspan
              className="fill-foreground"
              style={{ fontSize: 24, fontWeight: 700 }}
            >
              {total.toLocaleString()}
            </tspan>
            <tspan
              x="50%"
              dy="1.4em"
              className="fill-muted-foreground"
              style={{ fontSize: 12 }}
            >
              Total orders
            </tspan>
          </text>
          <ChartLegend
            content={<ChartLegendContent nameKey="status" />}
            iconSize={8}
            iconType="circle"
            layout="vertical"
            align="right"
            verticalAlign="middle"
          />
        </PieChart>
      </ChartContainer>
    </div>
  )
}
```

---

## Phase 3D — Area Chart (stacked volume over time)

```tsx
// components/TrafficAreaChart.tsx
"use client"

import { AreaChart, Area, XAxis, YAxis, CartesianGrid } from "recharts"
import {
  ChartContainer, ChartTooltip, ChartTooltipContent, type ChartConfig,
} from "@/components/ui/chart"

const chartConfig = {
  mobile:  { label: "Mobile",  color: "hsl(var(--chart-1))" },
  desktop: { label: "Desktop", color: "hsl(var(--chart-2))" },
  tablet:  { label: "Tablet",  color: "hsl(var(--chart-3))" },
} satisfies ChartConfig

export function TrafficAreaChart({ data, isLoading }: { data: any[]; isLoading?: boolean }) {
  if (isLoading) return <Skeleton className="h-[300px] w-full rounded-lg" />
  if (!data.length) return <EmptyChart label="No traffic data" />

  return (
    <ChartContainer config={chartConfig} className="h-[300px] w-full">
      <AreaChart data={data} margin={{ top: 4, right: 12, bottom: 4, left: 12 }}>
        <defs>
          {/* Gradient fill — more visually polished than flat fill */}
          <linearGradient id="colorMobile" x1="0" y1="0" x2="0" y2="1">
            <stop offset="5%"  stopColor="var(--color-mobile)"  stopOpacity={0.3} />
            <stop offset="95%" stopColor="var(--color-mobile)"  stopOpacity={0} />
          </linearGradient>
          <linearGradient id="colorDesktop" x1="0" y1="0" x2="0" y2="1">
            <stop offset="5%"  stopColor="var(--color-desktop)" stopOpacity={0.3} />
            <stop offset="95%" stopColor="var(--color-desktop)" stopOpacity={0} />
          </linearGradient>
        </defs>
        <CartesianGrid strokeDasharray="3 3" stroke="hsl(var(--border))" vertical={false} />
        <XAxis
          dataKey="date"
          tickLine={false} axisLine={false}
          tick={{ fill: "hsl(var(--muted-foreground))", fontSize: 12 }}
        />
        <YAxis tickLine={false} axisLine={false} width={40}
          tick={{ fill: "hsl(var(--muted-foreground))", fontSize: 12 }} />
        <ChartTooltip content={<ChartTooltipContent />} />
        <Area dataKey="mobile"  type="monotone" stroke="var(--color-mobile)"
              fill="url(#colorMobile)"  strokeWidth={2} />
        <Area dataKey="desktop" type="monotone" stroke="var(--color-desktop)"
              fill="url(#colorDesktop)" strokeWidth={2} />
      </AreaChart>
    </ChartContainer>
  )
}
```

---

## Phase 4 — Loading Skeleton (dimension-matched)

Every chart component must have a skeleton that matches the chart's exact dimensions and layout:

```tsx
// Skeleton matched to the chart's height and structural layout
function RevenueChartSkeleton() {
  return (
    <div className="flex flex-col gap-4">
      <div className="flex flex-col gap-1.5">
        <Skeleton className="h-5 w-48 rounded" />    {/* title */}
        <Skeleton className="h-4 w-64 rounded" />    {/* subtitle */}
      </div>
      <div className="h-[300px] w-full relative">
        {/* Y-axis ticks */}
        <div className="absolute left-0 top-0 h-full flex flex-col justify-between py-4">
          {[...Array(5)].map((_, i) => (
            <Skeleton key={i} className="h-3 w-8 rounded" />
          ))}
        </div>
        {/* Chart area */}
        <Skeleton className="absolute inset-0 ml-12 rounded-lg" />
        {/* X-axis ticks */}
        <div className="absolute bottom-0 left-12 right-0 flex justify-between">
          {[...Array(6)].map((_, i) => (
            <Skeleton key={i} className="h-3 w-8 rounded" />
          ))}
        </div>
      </div>
    </div>
  )
}
```

---

## Phase 5 — Empty State

```tsx
function EmptyChart({ label }: { label: string }) {
  return (
    <div className="h-[300px] w-full flex flex-col items-center justify-center gap-3 rounded-lg border border-dashed bg-muted/30">
      <div className="rounded-full bg-muted p-3">
        <BarChart3 className="h-6 w-6 text-muted-foreground" aria-hidden />
      </div>
      <div className="text-center">
        <p className="text-sm font-medium">{label}</p>
        <p className="text-xs text-muted-foreground mt-1">Data will appear here once available</p>
      </div>
    </div>
  )
}
```

---

## Hard Rules

1. **`ChartContainer` wraps all charts** — shadcn's `ChartContainer` handles: responsive sizing via `ResponsiveContainer` internally, `ChartConfig` context, ARIA `role="img"` and `aria-label`. Never add a `ResponsiveContainer` outside `ChartContainer` — it's already inside.
2. **`ChartConfig` uses CSS variable tokens, not hex** — `color: "hsl(var(--chart-1))"` adapts to dark mode. Hardcoded `color: "#6366f1"` fails in dark mode. Always use the `--chart-N` variables.
3. **No fixed `width` on `ChartContainer`** — `className="h-[300px] w-full"`. Fixed width prevents responsive behavior. Height is fixed (charts have a defined aspect ratio); width is always `w-full`.
4. **YAxis `width` must be explicit** — without `width={48}`, the Y-axis labels overlap the chart area on re-render. Always set `width` on `YAxis` to match the widest label.
5. **`vertical={false}` on `CartesianGrid`** — horizontal gridlines only. Vertical gridlines add noise for time-series and category charts. Only add vertical gridlines for scatter plots.
6. **Every chart needs a loading and empty state** — `isLoading ? <Skeleton /> : !data.length ? <EmptyChart /> : <Chart />`. AI skips both; both are required for production quality.
7. **Donut center label uses `<text>` SVG element** — not a React component. The center label in a donut chart renders inside an SVG canvas; use `<text>` and `<tspan>` elements with CSS variable fills.
8. **Limit pie/donut to 7 segments** — more than 7 pie segments become unreadable. Add an "Other" category that aggregates the tail. Flag this during chart type selection.

---

## Relationship to Other Skills

```
/claude-ui-chart      → THIS SKILL — individual chart components with all states
/claude-ui-dashboard  → dashboard layout: F-scan hierarchy, KPI classification, wireframe + full page
/claude-ui-loading    → dimension-matched skeletons (same pattern as chart skeleton above)
/claude-ui-empty      → entity-specific empty states (same pattern as chart empty above)
/claude-ui-tokens     → extract chart colors from existing code → formalize as --chart-N tokens
/claude-ui-darkmode   → verify --chart-N variables are defined for dark mode in globals.css
```

---

## When to Use

- **Building a dashboard** — run `/claude-ui-dashboard` for layout, then this skill for each individual chart
- **Adding a single chart** — directly generate one chart component with all states
- **Replacing a broken chart** — AI-generated charts missing `ResponsiveContainer` (0px width) or with hardcoded colors
- **Adding a new metric** — provide the data type and description; skill selects the correct chart type

## What This Is Not

- Not a full analytics dashboard — for dashboard layout hierarchy and KPI classification use `/claude-ui-dashboard`
- Not a Tremor/Victory/Nivo wrapper — this skill targets shadcn charts (Recharts-based); for other libraries, the pattern is similar but import paths differ
- Not a real-time chart — for live-updating charts (WebSocket data), add a `useEffect` to append data points to the existing array, and set `isAnimationActive={false}` on the chart to prevent re-animation on each update
- Not a data fetching layer — this skill generates the visual component; wire `isLoading` and `data` to your own query (TanStack Query, SWR, Server Component)

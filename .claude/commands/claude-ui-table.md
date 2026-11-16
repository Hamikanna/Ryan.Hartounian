# /claude-ui-table — TanStack Table Generator

Read a TypeScript type and generate a fully-typed, working data table. AI-generated
TanStack Tables always have the same failures: wrong generics, broken sort, pagination
stuck on page 1, no cell renderers for dates/numbers/enums. This skill reads your type
and outputs correct ColumnDef<T>[], proper hook wiring, and shadcn Table markup.

---

## Usage

```
/claude-ui-table <TypeName or FilePath>
```

Examples:
- `/claude-ui-table Order`                          — find Order type in codebase, generate table
- `/claude-ui-table src/types/order.ts`             — read types from file directly
- `/claude-ui-table src/types/order.ts --type Order` — specific type from multi-type file
- `/claude-ui-table "{ id: string; name: string; email: string; role: 'admin' | 'user'; createdAt: Date }"` — inline type

Options:
- `--server`          — generate server-side sort/filter/pagination (for large datasets)
- `--search`          — add global search input (default: on)
- `--export`          — add CSV export button
- `--selectable`      — add checkbox column for multi-row selection
- `--actions`         — add actions column (Edit, Delete) with dropdown
- `--no-pagination`   — disable pagination (for small static datasets)
- `--page-size 25`    — default page size (default: 10)

---

## Phase 1 — Type Analysis

Read the TypeScript type or interface. For each field, determine:

### Field → column type mapping

| TypeScript type | Column type | Cell renderer | Sort |
|-----------------|-------------|---------------|------|
| `string` | text | plain text | alphanumeric |
| `number` | numeric | right-aligned + `toLocaleString()` | numeric |
| `Date` | date | `format(date, 'MMM d, yyyy')` (date-fns) or `toLocaleDateString()` | date |
| `string` (ISO date) | date | detect if field name contains `At`, `Date`, `Time`, `On` → format as date |
| `boolean` | boolean | `<Check>` / `<X>` icon | boolean |
| `string` literal union (enum) | badge | `<Badge variant>` — map values to variants | alphabetic |
| `number` (money) | currency | detect if field name contains `price`, `amount`, `total`, `cost`, `revenue` → `$` formatted |
| `string` (email) | link | `<a href="mailto:...">` | alphanumeric |
| `string` (url) | link | `<a href="..." target="_blank">` | alphanumeric |
| `string[]` | tags | comma-joined badges | alphanumeric |
| `null` / `undefined` variants | nullable | show `—` for null/undefined values | |

### Enum → Badge variant mapping

For string literal union fields (e.g. `status: 'pending' | 'active' | 'cancelled'`):
```
positive values (active, complete, paid, approved, success, published) → variant="default" (green)
neutral values (pending, processing, draft, review)                    → variant="secondary" (gray)
negative values (cancelled, failed, rejected, overdue, inactive)       → variant="destructive" (red)
warning values (expiring, paused, trial)                               → variant="outline" with yellow text
```

### Column display names
- camelCase → Title Case: `createdAt` → "Created At", `totalAmount` → "Total Amount"
- Remove id suffix on display: `userId` → "User" (keep `accessorKey: "userId"`)
- Abbreviate long names: `numberOfActiveSubscriptions` → "Active Subscriptions"

---

## Phase 2 — Mode Decision

**Client-side** (default): all data loaded at once, TanStack handles sort/filter/pagination in memory.
- Use when: dataset ≤ 1000 rows, data fits in a single fetch
- Warning threshold: if the type has a `total: number` field > 1000 in context, recommend `--server`

**Server-side** (`--server`): sort/filter/pagination state sent to server as query params.
- Use when: large datasets, existing API with pagination support
- Generates: `useQuery` with TanStack Query (or SWR fallback) + URL-based state via `searchParams`

---

## Phase 3 — Generated Output

### Example type input:
```typescript
type Order = {
  id: string
  customer: string
  email: string
  total: number
  status: 'pending' | 'processing' | 'shipped' | 'delivered' | 'cancelled'
  createdAt: Date
  updatedAt: Date
}
```

### Generated: `OrdersTable.tsx`

```tsx
"use client"

import {
  ColumnDef,
  ColumnFiltersState,
  SortingState,
  VisibilityState,
  flexRender,
  getCoreRowModel,
  getFilteredRowModel,
  getPaginationRowModel,
  getSortedRowModel,
  useReactTable,
} from "@tanstack/react-table"
import { useState } from "react"
import { ArrowUpDown, ChevronLeft, ChevronRight, MoreHorizontal } from "lucide-react"

import { Badge } from "@/components/ui/badge"
import { Button } from "@/components/ui/button"
import {
  DropdownMenu, DropdownMenuContent, DropdownMenuItem,
  DropdownMenuTrigger
} from "@/components/ui/dropdown-menu"
import { Input } from "@/components/ui/input"
import {
  Table, TableBody, TableCell, TableHead, TableHeader, TableRow
} from "@/components/ui/table"
import { cn } from "@/lib/utils"
import type { Order } from "@/types/order"

// ─── Status badge variant mapping ────────────────────────────────────────────
const statusVariant: Record<Order["status"], "default" | "secondary" | "destructive" | "outline"> = {
  pending:    "secondary",
  processing: "secondary",
  shipped:    "default",
  delivered:  "default",
  cancelled:  "destructive",
}

const statusLabel: Record<Order["status"], string> = {
  pending:    "Pending",
  processing: "Processing",
  shipped:    "Shipped",
  delivered:  "Delivered",
  cancelled:  "Cancelled",
}

// ─── Column definitions ───────────────────────────────────────────────────────
const columns: ColumnDef<Order>[] = [
  {
    accessorKey: "customer",
    header: ({ column }) => (
      <Button
        variant="ghost"
        size="sm"
        className="-ml-3 h-8 data-[state=open]:bg-accent"
        onClick={() => column.toggleSorting(column.getIsSorted() === "asc")}
      >
        Customer
        <ArrowUpDown className="ml-2 h-3.5 w-3.5 text-muted-foreground/70" />
      </Button>
    ),
    cell: ({ row }) => (
      <div className="font-medium">{row.getValue("customer")}</div>
    ),
  },
  {
    accessorKey: "email",
    header: "Email",
    cell: ({ row }) => (
      <a
        href={`mailto:${row.getValue("email")}`}
        className="text-muted-foreground hover:text-foreground transition-colors
                   focus-visible:outline focus-visible:outline-2 focus-visible:outline-ring
                   focus-visible:rounded"
      >
        {row.getValue("email")}
      </a>
    ),
  },
  {
    accessorKey: "total",
    header: ({ column }) => (
      <Button
        variant="ghost"
        size="sm"
        className="-ml-3 h-8 data-[state=open]:bg-accent"
        onClick={() => column.toggleSorting(column.getIsSorted() === "asc")}
      >
        Total
        <ArrowUpDown className="ml-2 h-3.5 w-3.5 text-muted-foreground/70" />
      </Button>
    ),
    cell: ({ row }) => {
      const amount = row.getValue<number>("total")
      return (
        <div className="text-right font-medium tabular-nums">
          {new Intl.NumberFormat("en-US", { style: "currency", currency: "USD" }).format(amount)}
        </div>
      )
    },
  },
  {
    accessorKey: "status",
    header: "Status",
    cell: ({ row }) => {
      const status = row.getValue<Order["status"]>("status")
      return (
        <Badge variant={statusVariant[status]}>
          {statusLabel[status]}
        </Badge>
      )
    },
    filterFn: (row, id, value: string[]) => value.includes(row.getValue(id)),
  },
  {
    accessorKey: "createdAt",
    header: ({ column }) => (
      <Button
        variant="ghost"
        size="sm"
        className="-ml-3 h-8 data-[state=open]:bg-accent"
        onClick={() => column.toggleSorting(column.getIsSorted() === "asc")}
      >
        Created
        <ArrowUpDown className="ml-2 h-3.5 w-3.5 text-muted-foreground/70" />
      </Button>
    ),
    cell: ({ row }) => {
      const date = row.getValue<Date>("createdAt")
      return (
        <time
          dateTime={date instanceof Date ? date.toISOString() : date}
          className="text-muted-foreground tabular-nums"
        >
          {new Date(date).toLocaleDateString("en-US", {
            month: "short", day: "numeric", year: "numeric"
          })}
        </time>
      )
    },
    sortingFn: "datetime",
  },
  {
    id: "actions",
    enableHiding: false,
    cell: ({ row }) => {
      const order = row.original
      return (
        <DropdownMenu>
          <DropdownMenuTrigger asChild>
            <Button
              variant="ghost"
              size="sm"
              className="h-8 w-8 p-0 hover:bg-accent
                         focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2"
            >
              <MoreHorizontal className="h-4 w-4" aria-hidden />
              <span className="sr-only">Open actions for order {order.id}</span>
            </Button>
          </DropdownMenuTrigger>
          <DropdownMenuContent align="end">
            <DropdownMenuItem onClick={() => navigator.clipboard.writeText(order.id)}>
              Copy ID
            </DropdownMenuItem>
            <DropdownMenuItem>View order</DropdownMenuItem>
            <DropdownMenuItem className="text-destructive">Cancel order</DropdownMenuItem>
          </DropdownMenuContent>
        </DropdownMenu>
      )
    },
  },
]

// ─── Table component ──────────────────────────────────────────────────────────
interface OrdersTableProps {
  data: Order[]
}

export function OrdersTable({ data }: OrdersTableProps) {
  const [sorting, setSorting] = useState<SortingState>([])
  const [columnFilters, setColumnFilters] = useState<ColumnFiltersState>([])
  const [columnVisibility, setColumnVisibility] = useState<VisibilityState>({})
  const [globalFilter, setGlobalFilter] = useState("")

  const table = useReactTable({
    data,
    columns,
    state: { sorting, columnFilters, columnVisibility, globalFilter },
    onSortingChange: setSorting,
    onColumnFiltersChange: setColumnFilters,
    onColumnVisibilityChange: setColumnVisibility,
    onGlobalFilterChange: setGlobalFilter,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
    getPaginationRowModel: getPaginationRowModel(),
    initialState: { pagination: { pageSize: 10 } },
  })

  return (
    <div className="flex flex-col gap-4">
      
      {/* Toolbar */}
      <div className="flex items-center gap-3">
        <Input
          placeholder="Search orders..."
          value={globalFilter}
          onChange={(e) => setGlobalFilter(e.target.value)}
          className="max-w-xs"
          aria-label="Search orders"
        />
      </div>

      {/* Table */}
      <div className="rounded-md border">
        <Table>
          <TableHeader>
            {table.getHeaderGroups().map((headerGroup) => (
              <TableRow key={headerGroup.id}>
                {headerGroup.headers.map((header) => (
                  <TableHead key={header.id} colSpan={header.colSpan}>
                    {header.isPlaceholder
                      ? null
                      : flexRender(header.column.columnDef.header, header.getContext())}
                  </TableHead>
                ))}
              </TableRow>
            ))}
          </TableHeader>
          <TableBody>
            {table.getRowModel().rows.length > 0 ? (
              table.getRowModel().rows.map((row) => (
                <TableRow
                  key={row.id}
                  data-state={row.getIsSelected() && "selected"}
                  className="hover:bg-muted/50"
                >
                  {row.getVisibleCells().map((cell) => (
                    <TableCell key={cell.id}>
                      {flexRender(cell.column.columnDef.cell, cell.getContext())}
                    </TableCell>
                  ))}
                </TableRow>
              ))
            ) : (
              <TableRow>
                <TableCell colSpan={columns.length} className="h-24 text-center text-muted-foreground">
                  No orders found.
                </TableCell>
              </TableRow>
            )}
          </TableBody>
        </Table>
      </div>

      {/* Pagination */}
      <div className="flex items-center justify-between">
        <p className="text-sm text-muted-foreground">
          {table.getFilteredRowModel().rows.length} order
          {table.getFilteredRowModel().rows.length !== 1 ? "s" : ""} total
          {globalFilter && ` matching "${globalFilter}"`}
        </p>
        <div className="flex items-center gap-2">
          <Button
            variant="outline"
            size="sm"
            onClick={() => table.previousPage()}
            disabled={!table.getCanPreviousPage()}
            aria-label="Previous page"
            className="gap-1 disabled:opacity-50 disabled:cursor-not-allowed"
          >
            <ChevronLeft className="h-4 w-4" aria-hidden />
            Previous
          </Button>
          <span className="text-sm text-muted-foreground tabular-nums">
            Page {table.getState().pagination.pageIndex + 1} of{" "}
            {table.getPageCount()}
          </span>
          <Button
            variant="outline"
            size="sm"
            onClick={() => table.nextPage()}
            disabled={!table.getCanNextPage()}
            aria-label="Next page"
            className="gap-1 disabled:opacity-50 disabled:cursor-not-allowed"
          >
            Next
            <ChevronRight className="h-4 w-4" aria-hidden />
          </Button>
        </div>
      </div>

    </div>
  )
}
```

---

## Server-Side Variant (`--server`)

When `--server` is specified, generates URL-based state management with TanStack Query:

```tsx
// Key differences from client-side:
// 1. State lives in URL searchParams (shareable, back-button works)
// 2. useQuery fetches with params on every state change
// 3. manualPagination: true — TanStack doesn't paginate locally

"use client"
import { useRouter, useSearchParams, usePathname } from "next/navigation"
import { useQuery } from "@tanstack/react-query"
import { useReactTable, getCoreRowModel, getSortedRowModel } from "@tanstack/react-table"
import { columns } from "./columns"  // same ColumnDef<Order>[] as above
import type { Order } from "@/types/order"

interface OrdersResponse {
  data: Order[]
  total: number
  pageCount: number
}

export function OrdersTableServer() {
  const router = useRouter()
  const pathname = usePathname()
  const searchParams = useSearchParams()

  // Read state from URL
  const page = Number(searchParams.get("page") ?? "1") - 1
  const pageSize = Number(searchParams.get("pageSize") ?? "10")
  const sortField = searchParams.get("sortField") ?? ""
  const sortDir = (searchParams.get("sortDir") ?? "asc") as "asc" | "desc"
  const search = searchParams.get("search") ?? ""

  // Update URL when state changes
  const setParam = (key: string, value: string) => {
    const params = new URLSearchParams(searchParams.toString())
    if (value) params.set(key, value)
    else params.delete(key)
    router.push(`${pathname}?${params.toString()}`)
  }

  const { data, isLoading } = useQuery<OrdersResponse>({
    queryKey: ["orders", { page, pageSize, sortField, sortDir, search }],
    queryFn: () =>
      fetch(`/api/orders?page=${page}&pageSize=${pageSize}&sortField=${sortField}&sortDir=${sortDir}&search=${encodeURIComponent(search)}`)
        .then((r) => r.json()),
  })

  const table = useReactTable({
    data: data?.data ?? [],
    columns,
    pageCount: data?.pageCount ?? -1,
    state: {
      pagination: { pageIndex: page, pageSize },
      sorting: sortField ? [{ id: sortField, desc: sortDir === "desc" }] : [],
    },
    onSortingChange: (updater) => {
      const next = typeof updater === "function"
        ? updater(sortField ? [{ id: sortField, desc: sortDir === "desc" }] : [])
        : updater
      if (next.length > 0) {
        setParam("sortField", next[0].id)
        setParam("sortDir", next[0].desc ? "desc" : "asc")
      } else {
        setParam("sortField", "")
      }
    },
    onPaginationChange: (updater) => {
      const next = typeof updater === "function"
        ? updater({ pageIndex: page, pageSize })
        : updater
      setParam("page", String(next.pageIndex + 1))
    },
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    manualPagination: true,
    manualSorting: true,
    manualFiltering: true,
  })

  // ... same Table JSX as client version, pass isLoading for skeleton
}
```

---

## Selectable Rows (`--selectable`)

Adds checkbox column prepended to `columns`:

```tsx
import { Checkbox } from "@/components/ui/checkbox"

const selectColumn: ColumnDef<Order> = {
  id: "select",
  header: ({ table }) => (
    <Checkbox
      checked={table.getIsAllPageRowsSelected()}
      onCheckedChange={(value) => table.toggleAllPageRowsSelected(!!value)}
      aria-label="Select all"
      className="translate-y-[2px]"
    />
  ),
  cell: ({ row }) => (
    <Checkbox
      checked={row.getIsSelected()}
      onCheckedChange={(value) => row.toggleSelected(!!value)}
      aria-label={`Select row ${row.id}`}
      className="translate-y-[2px]"
    />
  ),
  enableSorting: false,
  enableHiding: false,
}

// Usage: show selection count above table
<p className="text-sm text-muted-foreground">
  {table.getFilteredSelectedRowModel().rows.length} of{" "}
  {table.getFilteredRowModel().rows.length} row(s) selected.
</p>
```

---

## Hard Rules

1. **Always type `ColumnDef<T>[]` correctly** — never `ColumnDef<any>[]`. The generic must match the data type.
2. **Always include all four row models** for a full-featured table: `getCoreRowModel()` + `getSortedRowModel()` + `getFilteredRowModel()` + `getPaginationRowModel()`. Missing any one breaks the corresponding feature silently.
3. **Dates always use `sortingFn: "datetime"`** — default string sort gives wrong order for ISO dates.
4. **Currency fields always use `Intl.NumberFormat`** — not string interpolation (`$${amount}`).
5. **Numbers always right-align** — `text-right` + `tabular-nums` for numeric columns.
6. **Empty state is required** — always include the empty row (`rows.length === 0` branch) with a contextual message.
7. **`sr-only` for actions trigger** — the `MoreHorizontal` button must have a `<span className="sr-only">` describing which row it's for.
8. **Pagination buttons must have `disabled:opacity-50 disabled:cursor-not-allowed`** — disabled state must be visually communicated.
9. **Server-side: state belongs in URL** — never `useState` for sort/filter/page in server-side tables. URL = shareable, back-button-safe, bookmark-friendly.

---

## Relationship to Other Skills

```
/claude-ui-table    → THIS SKILL — type → typed TanStack Table with working sort/filter/pagination
/claude-ui-form     → type → form component (sibling skill — both read TypeScript types)
/claude-ui-empty    → generate empty state for the table's zero-data condition
/claude-ui-loading  → generate skeleton for the table while data loads
/claude-ui-plan     → identify which tables a feature needs before building them
/claude-ui-stories  → generate Storybook stories for the table with fixture data
```

---

## When to Use

- **Any time data needs to be displayed in a list with sort/filter/pagination** — before building any table manually
- **When inheriting a table with broken sort or pagination** — regenerate from the type
- **When adding a new column** — add the field to the type, re-run to get the column definition
- **For admin panels, order management, user management, data exports** — all benefit from this pattern

## What This Is Not

- Not a no-code table builder — generates React code, not a configuration object
- Not a charting tool — for data visualization use `/claude-ui-dashboard`
- Not a grid editor (Excel-like) — generates read-optimized display tables with optional row actions
- Not a form — for create/edit use `/claude-ui-form`

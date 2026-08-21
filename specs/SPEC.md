Sales Analytics Dashboard — Technical Specification
1. Overview
This document describes how to rebuild the Sales Analytics Dashboard from scratch. The dashboard is a single-page React application that:

Pulls live sales data from a published Google Sheet CSV on every page load.
Displays summary KPIs (Total Revenue, Total Profit, Profit Margin).
Lets users filter the entire view by region.
Renders four charts: a monthly trend line chart, a horizontal bar chart of top customers by profit, donut charts for sales by category and region, and a bar chart of profit margin by category.
Uses a green-and-white design system with a deep emerald gradient background, glassmorphism cards, and mobile-responsive layout.
2. Technology Stack
Layer	Technology	Version
Framework	TanStack Start (full-stack React)	^1.168.32
Router	TanStack Router	^1.170.18
Data fetching	TanStack Query	^5.101.1
Server functions	createServerFn from @tanstack/react-start	-
Styling	Tailwind CSS v4	^4.2.1
Charts	Recharts	^2.15.4
Fonts	Google Fonts (Sora, Manrope)	-
Language	TypeScript	^5.8.3
Build tool	Vite	^8.2.0
No database is required. The published Google Sheet acts as the live source of truth.

3. Data Architecture
3.1 Source Data
The dashboard reads from a published Google Sheet CSV URL:

https://docs.google.com/spreadsheets/d/e/2PACX-1vQVg08JAuokldIXtS1SgUAmhz5w3fedyQNsYc4W62UV-XbS9mBfcksE0rmBI-w5obcSyDXpRoOdGGRk/pub?output=csv
A cache-busting query parameter (&cacheBust=${Date.now()}) is appended to the URL on every request so the browser/proxy does not serve stale data.

3.2 Expected CSV Columns
The CSV parser is case-insensitive and uses substring matching for column names.

Column	Header matcher	Data type	Notes
customer	contains customer	string	Defaults to "Unknown" if missing
category	contains category	string	Defaults to "Uncategorized" if missing
item	contains item	string	Defaults to "" if missing
qty	contains quantity	number	Strips non-numeric characters
unitPrice	contains unit price	number	Strips non-numeric characters
total	contains total	number	Strips non-numeric characters
cogs	contains cogs	number	Strips non-numeric characters
profit	contains profit but not %	number	Strips non-numeric characters
date	contains date	string (ISO YYYY-MM-DD)	Input format D/M/YYYY is normalized to ISO
region	contains region	string	Defaults to "Unknown" if missing
3.3 Internal Data Type
export type SaleRow = {
  customer: string;
  category: string;
  item: string;
  qty: number;
  unitPrice: number;
  total: number;
  cogs: number;
  profit: number;
  date: string;   // ISO 8601: YYYY-MM-DD
  region: string;
};
3.4 CSV Parsing Rules
CSV tokenizer: implement a custom parser that handles quoted fields and escaped quotes ("" → ").
Whitespace trim: trim every field after splitting.
Header row: first non-empty line is the header. Headers are lowercased before matching.
Numeric values: strip any character that is not 0-9, ., or -, then convert to Number. Non-finite values become 0.
Date normalization: if the raw date matches D/M/YYYY or D-M-YYYY, rebuild it as YYYY-MM-DD. Otherwise keep the raw string.
Missing columns: use the defaults listed above so the dashboard never crashes on malformed rows.
3.5 Server Function
Expose the data through a createServerFn GET server function.

export const getSales = createServerFn({ method: "GET" }).handler(async () => {
  return await fetchSalesFromSheet();
});
The server function returns:

{
  rows: SaleRow[];
  fetchedAt: string; // ISO timestamp, e.g. 2026-08-20T07:38:00.000Z
}
3.6 Query Client Configuration
Define a TanStack Query options object with the following behavior so the dashboard refreshes on every page load:

export const salesQueryOptions = queryOptions<{ rows: SaleRow[]; fetchedAt: string }>({
  queryKey: ["sales"],
  queryFn: () => getSales(),
  staleTime: 0,
  gcTime: 0,
  refetchOnMount: "always",
});
3.7 Hydration Note
The lastUpdated timestamp is formatted with toLocaleString and rendered only on the client inside a useEffect to avoid hydration mismatches between server and client.

4. Design System
4.1 Color Scheme
The dashboard uses a green and white theme. All colors are defined with oklch in CSS custom properties and registered as Tailwind theme colors.

Token	Value	Usage
--background	oklch(0.25 0.06 150)	Page background (deep green)
--foreground	oklch(0.99 0.005 150)	Primary white text
--card	oklch(0.98 0.01 150 / 72%)	Glass card fill (white, 72% opacity)
--card-foreground	oklch(0.2 0.06 150)	Dark text inside cards
--primary	oklch(0.6 0.18 145)	Emerald primary accent
--primary-foreground	oklch(0.98 0.005 150)	Text on primary buttons
--muted-foreground	oklch(0.45 0.06 150)	Secondary/muted text
--border	oklch(0.6 0.12 145 / 22%)	Card borders, dividers
--popover	oklch(0.97 0.01 150)	Tooltip/popover background
--popover-foreground	oklch(0.2 0.06 150)	Tooltip/popover text
Chart palette (vibrant green tones):

Token	Value
--chart-1	oklch(0.58 0.2 145)
--chart-2	oklch(0.66 0.18 135)
--chart-3	oklch(0.52 0.16 155)
--chart-4	oklch(0.62 0.22 120)
--chart-5	oklch(0.7 0.16 165)
--chart-6	oklch(0.45 0.18 145)
Gradients:

App background: --gradient-app: linear-gradient(150deg, oklch(0.18 0.05 150) 0%, oklch(0.28 0.08 145) 45%, oklch(0.45 0.12 140) 100%) (navy to deep emerald).
Accent text: --gradient-accent: linear-gradient(120deg, oklch(0.6 0.18 145), oklch(0.75 0.2 120)).
Card shadow: --shadow-card: 0 18px 45px -20px oklch(0.1 0.05 140 / 70%).
4.2 Typography
Display/headings: Sora (weights 500, 600, 700).
Body/UI: Manrope (weights 400, 500, 600, 700).
Headings use letter-spacing: -0.02em.
Load the fonts via <link> in the root route head (not via CSS @import):

<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossOrigin="anonymous" />
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Sora:wght@500;600;700&family=Manrope:wght@400;500;600;700&display=swap" />
4.3 Glass Card Utility
Create a reusable Tailwind utility class:

@utility glass-card {
  background: var(--card);
  backdrop-filter: blur(14px);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-2xl);
  box-shadow: var(--shadow-card);
}
And a gradient text utility:

@utility text-gradient {
  background: var(--gradient-accent);
  background-clip: text;
  color: transparent;
}
5. Dashboard Layout
5.1 Page Structure
<main max-w-7xl px-4 py-8 sm:px-8 sm:py-12>
  ├─ Header (eyebrow, title, subtitle, region filter chips)
  ├─ Summary cards (3-col grid on desktop, stacked on mobile)
  └─ Charts grid (2-col on desktop, stacked on mobile)
     ├─ Monthly trend line chart (full width, spans 2 cols)
     ├─ Top 10 customers by profit (horizontal bar)
     ├─ Sales by category (donut)
     ├─ Sales by region (donut)
     └─ Profit margin by category (vertical bar)
5.2 Header
Eyebrow: uppercase label Sales Intelligence with wide letter spacing, primary color.
Live status: Live from Google Sheets · Last updated <time> — formatted client-side.
Title: Revenue & Profit Analytics with the word "Profit" in the gradient text utility.
Subtitle: dynamic description with customer count, category count, and region count; appends — filtered to <region> when a region filter is active.
Region filter chips: horizontal row of rounded buttons. First chip is All, followed by one chip per unique region sorted alphabetically.
5.3 Summary Cards
Three cards, each using the glass-card utility.

Card	Value	Note
Total Revenue	$907,062 (example)	<N> orders
Total Profit	$611,884 (example)	Gross of COGS
Profit Margin	67.5%	Blended average
Value formatting rules:

Values ≥ 1,000: $<n/1k>k with one decimal (e.g. $907.1k).
Values < 1,000: $<n> with no decimals.
6. Charts
All charts are rendered inside ResponsiveContainer with width="100%" height="100%". Container heights are 260px on mobile and 320px on desktop (sm breakpoint ≥ 640px).

6.1 Monthly Revenue & Profit Trend
Type: LineChart (multi-line).
Position: full-width, spans both columns on desktop.
Data key: label for the X-axis (short month name, e.g. Jan).
Lines:
Revenue: dataKey="revenue", stroke var(--chart-1), strokeWidth 3, dots with radius 3.
Profit: dataKey="profit", stroke var(--chart-2), strokeWidth 3, dots with radius 3.
Axes: Y-axis formatted with the money formatter ($ or $k), X-axis shows month labels.
Grid: horizontal Cartesian grid only, stroke var(--border).
Tooltip/legend: custom styling using the popover tokens.
6.2 Top 10 Customers by Profit
Type: BarChart with layout="vertical".
Data: top 10 customers sorted by profit ascending (so the largest bar is at the bottom, readable top-to-bottom).
Bar: dataKey="profit", fill var(--chart-1), corner radius [0, 8, 8, 0].
Y-axis: customer names, category type.
X-axis: numeric, money formatter.
Mobile: reduce left margin and Y-axis width to prevent truncation.
6.3 Sales by Product Category
Type: PieChart rendered as a donut.
Data: byCategory grouped by category, dataKey="revenue".
Inner radius: 38 on mobile, 60 on desktop.
Outer radius: 68 on mobile, 110 on desktop.
Padding angle: 3.
Stroke: none.
Colors: cycle through CHART_COLORS array (--chart-1 … --chart-6).
Legend: shown below the chart.
6.4 Sales by Region
Type: PieChart rendered as a donut with the same sizing and styling as the category donut.
Data: byRegion grouped by region, dataKey="revenue".
Note: if a region filter is active, this donut only shows the selected region (a single full ring).
6.5 Profit Margin by Category
Type: BarChart (vertical).
Data: byCategory with computed marginPct = (profit / revenue) * 100.
X-axis: category names, rotated -15° on desktop and -35° on mobile to prevent overlap.
Y-axis: formatted as percentage (${v}%).
Bars: colored by category, cycling through CHART_COLORS, corner radius [8, 8, 0, 0].
6.6 Shared Tooltip Style
Use a single tooltip configuration object across all charts:

const tooltipStyle = {
  contentStyle: {
    background: "var(--popover)",
    border: "1px solid var(--border)",
    borderRadius: "0.75rem",
    color: "var(--popover-foreground)",
    fontSize: 12,
  },
  itemStyle: { color: "var(--popover-foreground)" },
  labelStyle: { color: "var(--muted-foreground)" },
};
7. Filtering & Interactivity
7.1 Region Filter
State: region (string), default "All".

Available chips: ["All", ...uniqueRegions] where uniqueRegions comes from [...new Set(sales.map((s) => s.region))].sort().

Filtered rows:

const rows = useMemo(
  () => (region === "All" ? sales : sales.filter((s) => s.region === region)),
  [region, sales],
);
All downstream calculations (summary, charts) must be derived from rows.

7.2 Active Chip Styling
Active: border-primary bg-primary text-primary-foreground.
Inactive: border-foreground/30 bg-foreground/10 text-foreground hover:bg-foreground/15.
Minimum height: 44px (accessibility tap target).
Border radius: rounded-full.
8. Responsive Behavior
Use a custom useIsMobile hook that listens to window.innerWidth < 768.

Breakpoints and adaptations:

Element	Mobile (< 640px)	Desktop (≥ 640px)
Page padding	px-4 py-8	px-8 py-12
Title size	text-3xl	text-5xl
Summary cards	1-column stack	3-column grid
Summary value size	text-3xl	text-4xl
Chart card padding	p-4	p-6
Chart container height	260px	320px
Donut inner radius	38	60
Donut outer radius	68	110
Horizontal bar Y-axis width	92	130
Horizontal bar left margin	0	60
Margin bar X-axis rotation	-35°	-15°
There must be no horizontal overflow on a 390px-wide viewport.

9. Route & SEO
The dashboard is the root route /.

9.1 Root Route (__root.tsx)
Provides <QueryClientProvider> around <Outlet />.
Injects global CSS via <link rel="stylesheet" href={appCss} />.
Loads Google Fonts via <link> tags.
Defines viewport meta tag and default OpenGraph/Twitter meta tags.
Provides 404 and error boundary components using the same color tokens.
9.2 Index Route (/) Head Metadata
head: () => ({
  meta: [
    { title: "Sales Analytics Dashboard | Revenue, Profit & Margins" },
    { name: "description", content: "Interactive sales analytics: total revenue, profit by customer, margins by product category, regional split and monthly revenue trends." },
    { property: "og:title", content: "Sales Analytics Dashboard" },
    { property: "og:description", content: "Revenue, profit and margin analytics across customers, categories and regions." },
  ],
}),
10. Data Transformations
Implement a groupSum helper:

function groupSum(rows: SaleRow[], keyFn: (r: SaleRow) => string) {
  const map = new Map<string, { revenue: number; profit: number; cogs: number }>();
  for (const r of rows) {
    const k = keyFn(r);
    const cur = map.get(k) ?? { revenue: 0, profit: 0, cogs: 0 };
    cur.revenue += r.total;
    cur.profit += r.profit;
    cur.cogs += r.cogs;
    map.set(k, cur);
  }
  return [...map.entries()].map(([name, v]) => ({ name, ...v }));
}
Derived datasets:

topCustomers: groupSum(rows, r => r.customer).sort((a, b) => b.profit - a.profit).slice(0, 10).reverse().
byCategory: groupSum(rows, r => r.category).map(c => ({ ...c, marginPct: (c.profit / c.revenue) * 100 })).sort((a, b) => b.revenue - a.revenue).
byRegion: groupSum(rows, r => r.region).sort((a, b) => b.revenue - a.revenue).
monthly: groupSum(rows, r => r.date.slice(0, 7)).sort((a, b) => a.name.localeCompare(b.name)).map(m => ({ ...m, label: new Date(${m.name}-01T00:00:00).toLocaleDateString(undefined, { month: "short" }) })).
Summary calculations:

const totalRevenue = rows.reduce((s, r) => s + r.total, 0);
const totalProfit = rows.reduce((s, r) => s + r.profit, 0);
const margin = totalRevenue ? (totalProfit / totalRevenue) * 100 : 0;
11. File Structure
src/
├── data/
│   └── sales.ts                    # SaleRow type only
├── hooks/
│   └── use-mobile.tsx              # useIsMobile hook
├── lib/
│   ├── sales.server.ts             # CSV fetch + parser
│   ├── sales.functions.ts          # createServerFn getSales
│   └── sales-query.ts              # salesQueryOptions
├── routes/
│   ├── __root.tsx                  # Root layout, fonts, providers
│   └── index.tsx                   # Dashboard page
├── router.tsx                      # createRouter + QueryClient
├── styles.css                      # Tailwind v4 theme + custom utilities
└── routeTree.gen.ts                # Auto-generated (do not edit)
12. Rebuild Checklist
Initialize project with TanStack Start + Tailwind CSS v4 + Recharts.
Add fonts to the root route head via <link> (Sora, Manrope).
Define color tokens in styles.css using the exact oklch values above; create glass-card and text-gradient utilities.
Implement useIsMobile hook.
Create SaleRow type in src/data/sales.ts.
Implement CSV fetch + parser in src/lib/sales.server.ts.
Create the server function getSales in src/lib/sales.functions.ts.
Create salesQueryOptions in src/lib/sales-query.ts with staleTime: 0 and gcTime: 0.
Build the dashboard page in src/routes/index.tsx:
Route loader calls context.queryClient.ensureQueryData(salesQueryOptions).
Component calls useSuspenseQuery(salesQueryOptions).
Add header, region filter, summary cards, and five charts.
Verify responsiveness: test at 390px and 1280px; confirm no horizontal scroll, readable labels, and 44px tap targets.
Verify live data: load the page, confirm the "Last updated" timestamp changes on reload and the numbers match the Google Sheet.
13. Example Data Snapshot
With the current source sheet, the dashboard should display values approximately:

Total Revenue: ~$907,062
Total Profit: ~$611,884
Profit Margin: ~67.5%
(Exact values depend on the current contents of the Google Sheet.)

# CLAUDE.md

Factory Inventory Management System Demo with GitHub integration - Full-stack application with Vue 3 frontend, Python FastAPI backend, and in-memory mock data (no database).

## Critical Tool Usage Rules

### Subagents
Use the Task tool with these specialized subagents for appropriate tasks:

- **vue-expert**: Use for Vue 3 frontend features, UI components, styling, and client-side functionality
  - Examples: Creating components, fixing reactivity issues, performance optimization, complex state management
  - **MANDATORY RULE: ANY time you need to create or significantly modify a .vue file, you MUST delegate to vue-expert**
- **code-reviewer**: Use after writing significant code to review quality and best practices
- **Explore**: Use for understanding codebase structure, searching for patterns, or answering questions about how components work
- **general-purpose**: Use for complex multi-step tasks or when other agents don't fit

### Skills
- **backend-api-test** skill: Use when writing or modifying tests in `tests/backend` directory with pytest and FastAPI TestClient

### MCP Tools
- **ALWAYS use GitHub MCP tools** (`mcp__github__*`) for ALL GitHub operations
  - Exception: Local branches only - use `git checkout -b` instead of `mcp__github__create_branch`
- **ALWAYS use Playwright MCP tools** (`mcp__playwright__*`) for browser testing
  - Test against: `http://localhost:3000` (frontend), `http://localhost:8001` (API)

## Stack
- **Frontend**: Vue 3 + Composition API + Vite (port 3000)
- **Backend**: Python FastAPI (port 8001)
- **Data**: JSON files in `server/data/` loaded via `server/mock_data.py`

## Quick Start

```bash
# Backend
cd server
uv run python main.py

# Frontend
cd client
npm install && npm run dev
```

## Key Patterns

**Filter System**: 4 filters (Time Period, Warehouse, Category, Order Status) apply to all data via query params
**Data Flow**: Vue filters → `client/src/api.js` → FastAPI → In-memory filtering → Pydantic validation → Computed properties
**Reactivity**: Raw data in refs (`allOrders`, `inventoryItems`), derived data in computed properties

## API Endpoints
- `GET /api/inventory` - Filters: warehouse, category
- `GET /api/orders` - Filters: warehouse, category, status, month
- `GET /api/dashboard/summary` - All filters
- `GET /api/demand`, `/api/backlog` - No filters
- `GET /api/spending/*` - Summary, monthly, categories, transactions
- `GET /api/restocking/recommendations` - Budget-based restock recommendations
- `POST /api/restocking/orders` - Place restocking orders
- `GET /api/restocking/orders` - List placed restocking orders
- `GET /api/reports/quarterly`, `/api/reports/monthly-trends` - Filters: warehouse, category, status, month

## Lessons Learned (from past bugs — MUST follow)

These are real issues that were introduced by Claude and required follow-up fixes. Check each one before completing any frontend work.

### 1. v-for keys: NEVER use array index
**Bug**: Used `:key="index"` in v-for loops, causing incorrect DOM reuse when data changes.
**Fix**: Always use a unique domain identifier — `item.sku`, `q.quarter`, `month.month`, `order.id`, etc.
**Commit**: `a1f5eac`

### 2. Always add empty states for data lists
**Bug**: Tables and charts rendered with no content when data arrays were empty — no visual feedback to user.
**Fix**: Every `v-for` over API data must have a sibling `v-if="data.length === 0"` block showing an empty state message.
**Commit**: `a1f5eac`

### 3. CSS must match design system — no freestyling
**Bug**: New components used `border-radius: 12px`, `box-shadow`, and pill badges (`border-radius: 9999px`) instead of the project's design tokens.
**Fix**: Before writing CSS, check existing components for the correct values:
- Cards: `border-radius: 10px; border: 1px solid #e2e8f0;` (no box-shadow)
- Badges: `border-radius: 6px;` (not pill-shaped)
- Modals: `border-radius: 10px;`
- When in doubt, read `App.vue` and an existing view like `Dashboard.vue` for reference.
**Commits**: `a1f5eac`, `e69f0f5`

### 4. i18n: every user-visible string must use t()
**Bug**: Added a "Reports" nav link with a hardcoded English string, forgetting the i18n system.
**Fix**: Every user-visible string must use `{{ t('key') }}` in templates. When adding new strings:
1. Add the key to `client/src/locales/en.js`
2. Add the translation to `client/src/locales/ja.js`
3. Use `t('key')` in the template — never hardcode text
**Commit**: `06d580c`

### 5. Clear stale derived state when inputs change
**Bug**: Restocking page showed negative "Budget Remaining" because old recommendations (fetched for a higher budget) persisted after the budget slider was lowered.
**Fix**: When a user input controls an async fetch, `watch()` that input and clear any stale results immediately. Don't rely on the next fetch to overwrite — the user sees the stale state in between.
```javascript
watch(budget, () => {
  if (recommendations.value.length > 0) {
    recommendations.value = []
    hasSearched.value = false
  }
})
```
**Commit**: `c3c15e4`

## Common Issues
1. Validate dates before `.getMonth()` calls
2. Update Pydantic models when changing JSON data structure
3. Inventory filters don't support month (no time dimension)
4. Revenue goals: $800K/month single, $9.6M YTD all months

## File Locations
- Views: `client/src/views/*.vue` (Dashboard, Demand, Inventory, Orders, Spending, Backlog, Restocking, Reports)
- Components: `client/src/components/*.vue` (modals, FilterBar, ProfileMenu, LanguageSwitcher)
- Composables: `client/src/composables/` (useAuth, useFilters, useI18n)
- Locales: `client/src/locales/` (en.js, ja.js)
- API Client: `client/src/api.js`
- Backend: `server/main.py`, `server/mock_data.py`
- Data: `server/data/*.json`
- Styles: `client/src/App.vue` (global), scoped styles in each view

## Design System
- Cards: `border-radius: 10px; border: 1px solid #e2e8f0;` (no box-shadow)
- Badges: `border-radius: 6px; padding: 0.25rem 0.75rem;`
- Modals: `border-radius: 10px;`
- Colors: Slate/gray (#0f172a, #64748b, #e2e8f0)
- Status: green (#dcfce7/#166534), blue (#dbeafe/#1e40af), yellow (#fef3c7/#92400e), red (#fecaca/#991b1b)
- Charts: Custom SVG, CSS Grid for layouts
- No emojis in UI
- Empty states: centered text, `color: #64748b`, generous padding

## Recent Commit History
- `d3d0d29` Merge PR #3: Backlog page fixes (routing, i18n, CSS, computed props)
- `69c4a60` Merge PR #2: Restocking tab (budget recommendations, order placement)
- `98701b7` Merge PR #1: Reports page bug bounty (v-for keys, empty states, CSS, i18n)

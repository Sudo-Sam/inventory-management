---
name: saas-sidebar-redesign
description: Use this skill when redesigning a Vue 3 application's navigation and layout into a modern SaaS-style interface with a collapsible vertical left sidebar, replacing a horizontal top navigation bar. Applies to any Vue 3 SPA with a root layout component and vue-router config, not just this app. Triggers on requests like "redesign the UI with a sidebar", "convert the top nav to a sidebar", "give this app a modern SaaS layout".
---

# SaaS Sidebar Redesign

## Goal

Transform this Vue 3 application's navigation and layout from a horizontal top nav bar into a modern SaaS product shell featuring a collapsible vertical left sidebar. The redesign focuses purely on layout, navigation structure, and consistent spacing — it does NOT rebrand colors, introduce design tokens, or change the app's business logic or API contracts. The result is a professional, polished SaaS-style interface that preserves all existing functionality and styling conventions.

## Constraints & Non-Goals

- **No CSS custom properties or design tokens.** Reuse the app's existing literal color values (hex codes) exactly as they appear today. Only reposition and reapply them — for example, use the existing brand blue for a sidebar active-state accent instead of a bottom border. Do not introduce `:root` variables or any design-token system.
- **Never touch the backend.** Do not modify `server/`, API contracts, or any backend code.
- **Preserve all existing global CSS classes.** Views and components depend on global classes like `.page-header`, `.stats-grid`, `.stat-card`, `.card`, `.table-container`, `.badge`, `.loading`, and `.error`. Do not rename, remove, or restructure these — only add new sidebar-specific classes alongside them.
- **No emojis in UI.** Match the project's existing constraint (documented in the root `CLAUDE.md` and `vue-expert` agent guidelines).
- **Preserve and normalize i18n usage.** If the app uses `i18n` translation functions for nav labels (e.g. `t('nav.overview')`), keep using them. If one nav item is hardcoded as a plain string among otherwise-translated siblings, fix the hardcoded one to match the i18n convention while it's being touched — add the new locale key if needed.
- **Keep the existing icon/UI convention.** If the app hand-codes inline SVG icons (no external icon library), continue building new icons the same way. Do not introduce a CSS framework, icon library, or UI component kit unless one is already a project dependency.
- **Delegate all `.vue` file creation/modification to the `vue-expert` subagent.** This is a project-wide mandatory rule. The orchestrating agent must never edit `.vue` files directly — only read them for discovery and run read-only investigation steps. Phase 5 handles delegation.

## Phase 1: Discover the App's Current Structure

This phase is a read-only investigation completed directly by the orchestrating agent — no delegation needed.

1. **Locate the root layout component.** Find the Vue component mounted by the app's entrypoint (usually referenced in `main.js` or `main.ts` as the component passed to `app.mount('#app')`). Read it fully. This is typically where the current navigation bar, layout wrapper (`<div class="app">`), modals, and global unscoped `<style>` block all live. Confirm the current nav structure: is it a `<header>` with a horizontal `<nav>` inside? A flex row with `margin-left: auto` right-aligning nav items? What are the nav link labels and target routes?

2. **Locate the router configuration.** Find the `createRouter()` call — it may be inline in `main.js` or in a separate `router/index.js` or `router.ts` file. Extract every registered route: the path (e.g. `/`, `/inventory`), the imported component, and the component name. Create a clean list for Phase 3.

3. **Glob the app's views directory** (commonly `src/views/`, may vary). List every `.vue` file. Cross-reference this list against the routes from step 2 to identify any view file NOT registered in the router. Flag these orphaned views for Phase 3's wire-up step.

4. **Catalog the current nav links.** For every `router-link` (or equivalent nav element) in the root layout, record: (a) the target route/path, (b) the label text or i18n function call (e.g. `{{ t('nav.overview') }}` vs. hardcoded `"Reports"`), (c) how "active" state is determined (e.g. `:class="{ active: $route.path === '/x' }"`) and styled (what CSS class/rule makes an active item visually distinct — color, background, border, etc.?).

5. **Catalog the global stylesheet.** Find the app's global CSS, typically in the root layout's unscoped `<style>` block (or a separate `global.css`/`main.css`/`style.css` imported in the entrypoint). Record exactly:
   - All color values used (hex codes, RGB, or CSS variable references — don't interpret or change them, just capture them exactly).
   - Font stack (font family names, fallbacks).
   - Spacing scale (common margin/padding values, typically a grid like `0.25rem`, `0.5rem`, `1rem`, etc.).
   - Border-radius conventions (typical values for buttons, cards, inputs).
   - Every globally-scoped class name (ones not inside a `<style scoped>` block) — especially `.page-header`, `.card`, `.stats-grid`, `.stat-card`, `.badge`, `.table-container`, `.loading`, `.error`, and any others used across multiple views. These are NOT to be touched.

6. **Find components structurally coupled to the old header.** Search the client source for any component using `position: sticky` or `position: fixed` with a `top` offset (like `top: 70px`) that appears to be anchored to the header's height. Also flag any component rendered in the root layout between the header and the `<router-view>` — these are candidates for re-anchoring in Phase 4. Common examples: filter bars, breadcrumb areas, secondary toolbars.

7. **Check for existing UI dependencies.** Open `package.json` and scan the `dependencies` and `devDependencies` for icon libraries (e.g. Font Awesome, Feather Icons), UI frameworks (Tailwind, Vuetify, PrimeVue, Element Plus), or CSS-in-JS libraries. This determines whether new sidebar icons should be hand-coded inline SVGs (if no icon lib exists) or built using the existing framework.

**Example walkthrough (inventory-management app):**
- Root layout: `client/src/App.vue` — contains a sticky `<header class="top-nav">` (70px tall), a `.nav-container` with logo on the left and a `.nav-tabs` on the right (6 `router-link` elements, right-aligned via `margin-left: auto`). Active state via `:class="{ active: $route.path === '/x' }"` with blue text + bottom border. Below the header is a `<FilterBar />` component, then `<main class="main-content">` with `<router-view />`. Modal overlays and the profile menu are also rendered at the root level.
- Router: inline in `client/src/main.js` — routes for `/`, `/inventory`, `/orders`, `/demand`, `/spending`, `/reports` (6 routes total, each importing a view component).
- Views directory: `client/src/views/` — contains `Dashboard.vue`, `Inventory.vue`, `Orders.vue`, `Demand.vue`, `Spending.vue`, `Reports.vue` (all 6 registered), plus `Backlog.vue` (NOT registered — orphaned).
- Nav labels: all except "Reports" use i18n (`t('nav.overview')`, etc.); "Reports" is hardcoded as a string — inconsistency to fix.
- Global styles: all in `App.vue`'s `<style>` block — color palette with hex values like `#ffffff`, `#f8fafc`, `#e2e8f0`, `#2563eb`, `#eff6ff`, `#64748b`, etc.; font stack starting with `'Inter'`; spacing scale of `0.25rem` multiples; border-radius of `6px`, `8px`, `10px`; global classes: `.page-header`, `.stats-grid`, `.stat-card`, `.card`, `.table-container`, `.badge`, `.loading`, `.error`.
- Header-coupled component: `FilterBar.vue` — positioned with `position: sticky; top: 70px; z-index: 90;` on its `.filters-bar` class. Also has `.filters-container` with `max-width: 1600px; margin: 0 auto; padding: 0 2rem;` (full-viewport centering).
- UI dependencies: `package.json` shows no UI framework, no icon library — all icons are hand-coded inline SVGs.

## Phase 2: Design the Sidebar Shell

Based on Phase 1's discoveries, design the new layout structure and its CSS. This phase is instruction/blueprint; Phase 5 (vue-expert delegation) will execute it.

**Structural change:** The root layout shifts from a vertical flex stack (header on top, content below) to a horizontal flex layout with a fixed-width (or width-transitioning) sidebar on the left and a content column on the right.

```
Old layout (top-nav):
┌─────────────────────────────┐
│         Header (70px)        │
├─────────────────────────────┤
│       FilterBar (sticky)     │
├─────────────────────────────┤
│                              │
│     Content (router-view)   │
│                              │
└─────────────────────────────┘

New layout (sidebar):
┌──────────┬──────────────────┐
│          │                  │
│ Sidebar  │  FilterBar (now  │
│ (240px,  │   at top of      │
│collapse  │  content column) │
│  →64px)  │                  │
│          ├──────────────────┤
│          │                  │
│          │ Content          │
│          │(router-view)     │
│          │                  │
└──────────┴──────────────────┘
```

**Sidebar composition (top to bottom):**

1. **Brand/Logo area** (top, fixed height, ~1.5rem padding). Reuse whatever logo/company-name markup already existed in the header. When collapsed (icon-only rail), reduce to an icon or initials.

2. **Primary navigation list** (middle, flex: 1 to grow). For each route from Phase 1, create a nav item with:
   - An inline SVG icon (20–24px, viewBox "0 0 24 24", line-style stroke or filled, consistent with any icon style already in the codebase).
   - A text label (the i18n-translated or original label from Phase 1).
   - Active state: when `$route.path` matches the item's path, apply a left-edge accent border (e.g. 3px solid) in the app's existing primary blue color, plus a tinted background (the app's existing "active background" color), plus the active text color. This is the vertical equivalent of the old horizontal bottom-border active tab.

3. **Footer area** (bottom, fixed, ~1rem). Contain utility items like a collapse/expand toggle button. Optionally, if the app has `LanguageSwitcher` and `ProfileMenu` components that fit in a vertical orientation (they're typically icon-based dropdowns), they can move here. If they're too horizontal (wide buttons, text labels), consider keeping them in a slim secondary bar within the content column instead.

4. **Collapse toggle button** (within the footer or near it). A simple chevron or arrow icon button that toggles the `sidebarCollapsed` state. When the sidebar is collapsed, this button still occupies the same space so it remains clickable.

**Collapse/expand behavior:**

- **State:** A single reactive boolean, e.g. `const sidebarCollapsed = ref(false)`.
- **Persistence:** Store the collapsed state in `localStorage` so the user's preference survives a page reload (e.g. `localStorage.setItem('sidebarCollapsed', sidebarCollapsed.value)`). On app init, read from `localStorage` to restore the last state.
- **Width transition:** The sidebar width animates between two states: expanded (e.g. `240px`) and collapsed (e.g. `64px`). Use a CSS `transition: width 0.2s ease` for a smooth animation.
- **Label visibility:** Nav item labels are conditionally rendered — visible when expanded, hidden when collapsed (use `v-if="!sidebarCollapsed"` or similar).
- **Icon-only usability:** When collapsed, each nav item becomes icon-only. Add a `title` attribute to each `router-link` (e.g. `:title="sidebarCollapsed ? t(item.label) : null"`) so hovering over an icon shows the label as a browser tooltip. Alternatively, implement a lightweight CSS-only tooltip (`:hover::after { content: ... }`) if a more polished hover experience is desired.

**Active-item styling:** Translate the old horizontal active-state treatment into a vertical one. The app's existing color values must be reused exactly — do not invent new colors. Example: if the old active tab had `color: #2563eb` + `background: #eff6ff` + `border-bottom: 2px solid #2563eb`, the new vertical active state should be `border-left: 3px solid #2563eb` + `background: #eff6ff` + `color: #2563eb`.

**Spacing and polish:** Apply the app's existing spacing scale consistently. Example: if the app uses `0.25rem` increments (0.5rem, 0.75rem, 1rem, 1.25rem, 1.5rem, 2rem) for padding/margin, use the same increments for the sidebar. Nav item padding might be `0.625rem 0.75rem` (matching the app's scale), gap between items `0.25rem` or `0.5rem`, sections separated by `1rem`. Border-radius on nav items should match the app's conventions (likely `6px`).

**Illustrative code sketch** (the following is a Vue 3 SFC with `<script setup>`, template, and CSS; adapt nav items, colors, and assets to the target app's actual structure):

```vue
<script setup>
import { ref, computed, onMounted } from 'vue'
import { useRoute } from 'vue-router'
import LanguageSwitcher from '@/components/LanguageSwitcher.vue'
import ProfileMenu from '@/components/ProfileMenu.vue'
import FilterBar from '@/components/FilterBar.vue'
// ... import any other components needed (modals, etc.) ...

const route = useRoute()
const showProfileDetails = ref(false)
const showTasks = ref(false)

// Collapse state, persisted across sessions via localStorage
const sidebarCollapsed = ref(localStorage.getItem('sidebarCollapsed') === 'true')

function toggleSidebar() {
  sidebarCollapsed.value = !sidebarCollapsed.value
  localStorage.setItem('sidebarCollapsed', sidebarCollapsed.value)
}

// Adapt this list to the app's actual routes from Phase 1.
// Include any newly-wired orphaned views from Phase 3.
// Each entry needs a path (route), label (i18n key), and icon (inline SVG).
const navItems = [
  { path: '/',          label: 'nav.overview',      icon: 'grid' },
  { path: '/inventory', label: 'nav.inventory',     icon: 'box' },
  { path: '/orders',    label: 'nav.orders',        icon: 'clipboard' },
  { path: '/spending',  label: 'nav.finance',       icon: 'dollar' },
  { path: '/demand',    label: 'nav.demandForecast', icon: 'trending-up' },
  { path: '/reports',   label: 'nav.reports',       icon: 'bar-chart' },
  { path: '/backlog',   label: 'nav.backlog',       icon: 'list' } // newly wired from Phase 3
]

// Load task/profile state (example: these may come from api.js or useAuth)
const tasks = ref([])
async function loadTasks() {
  // tasks.value = await api.getTasks()
}
onMounted(() => {
  loadTasks()
})

// Rest of component logic (task CRUD, profile details, etc.) remains unchanged
</script>

<template>
  <div class="app-shell">
    <!-- Left sidebar navigation -->
    <aside class="sidebar" :class="{ collapsed: sidebarCollapsed }">
      <!-- Brand/logo area -->
      <div class="sidebar-brand">
        <h1 v-if="!sidebarCollapsed">{{ companyName }}</h1>
        <!-- When collapsed, show just the first letter or a logo icon -->
        <span v-else class="brand-mark">{{ companyName[0] }}</span>
      </div>

      <!-- Primary navigation -->
      <nav class="sidebar-nav">
        <router-link
          v-for="item in navItems"
          :key="item.path"
          :to="item.path"
          class="sidebar-nav-item"
          :class="{ active: $route.path === item.path }"
          :title="sidebarCollapsed ? $t(item.label) : null"
        >
          <!-- Inline SVG icon (example structure; replace with actual icon paths) -->
          <svg class="nav-icon" viewBox="0 0 24 24" width="20" height="20">
            <!-- Icon content goes here; see examples below -->
            <path v-if="item.icon === 'grid'" d="M3 3h8v8H3V3zm10 0h8v8h-8V3zM3 13h8v8H3v-8zm10 0h8v8h-8v-8z" />
            <!-- ... add paths for other icons: box, clipboard, dollar, trending-up, bar-chart, list ... -->
          </svg>
          <!-- Nav label, conditionally rendered when NOT collapsed -->
          <span v-if="!sidebarCollapsed" class="nav-label">{{ $t(item.label) }}</span>
        </router-link>
      </nav>

      <!-- Footer: utilities and collapse toggle -->
      <div class="sidebar-footer">
        <LanguageSwitcher />
        <ProfileMenu @show-profile-details="showProfileDetails = true" @show-tasks="showTasks = true" />
        <button
          class="sidebar-toggle"
          @click="toggleSidebar"
          :title="sidebarCollapsed ? 'Expand sidebar' : 'Collapse sidebar'"
          aria-label="Toggle sidebar"
        >
          <svg viewBox="0 0 24 24" width="18" height="18" :style="{ transform: sidebarCollapsed ? 'rotate(180deg)' : 'none' }">
            <!-- Chevron/arrow pointing left, rotated 180° when collapsed to point right -->
            <path d="M15 18l-6-6 6-6" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" />
          </svg>
        </button>
      </div>
    </aside>

    <!-- Main content column -->
    <div class="app-content">
      <!-- FilterBar moves here, no longer below a header in a separate stacking context -->
      <FilterBar />

      <!-- Page content -->
      <main class="main-content">
        <router-view />
      </main>
    </div>

    <!-- Modal overlays (ProfileDetailsModal, TasksModal, etc.) remain unchanged -->
    <ProfileDetailsModal :is-open="showProfileDetails" @close="showProfileDetails = false" />
    <TasksModal :is-open="showTasks" @close="showTasks = false" :tasks="tasks" />
  </div>
</template>

<style>
/* 
  IMPORTANT: All existing global classes (.page-header, .stats-grid, .card, .table-container, etc.)
  are preserved UNCHANGED from the app's original <style> block in App.vue.
  
  Only the following new sidebar-shell classes are added.
  Every color value here must be replaced with the app's existing palette (from Phase 1, step 5).
*/

.app-shell {
  display: flex;
  min-height: 100vh;
  background: #f8fafc;
}

.sidebar {
  width: 240px;
  background: #ffffff;
  border-right: 1px solid #e2e8f0;
  display: flex;
  flex-direction: column;
  transition: width 0.2s ease;
  flex-shrink: 0;
  z-index: 50; /* above content but below modals */
}

.sidebar.collapsed {
  width: 64px;
}

.sidebar-brand {
  padding: 1.5rem 1.25rem;
  border-bottom: 1px solid #f1f5f9;
  display: flex;
  align-items: center;
  justify-content: center;
}

.sidebar-brand h1 {
  margin: 0;
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.brand-mark {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}

.sidebar-nav {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  padding: 1rem 0.75rem;
  flex: 1;
  overflow-y: auto;
}

.sidebar-nav-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.625rem 0.75rem;
  border-radius: 6px;
  border-left: 3px solid transparent;
  color: #475569;
  text-decoration: none;
  font-size: 0.875rem;
  font-weight: 500;
  transition: all 0.15s ease;
  cursor: pointer;
}

.sidebar-nav-item:hover {
  background: #f8fafc;
  color: #334155;
}

.sidebar-nav-item.active {
  background: #eff6ff;
  border-left-color: #2563eb;
  color: #2563eb;
  font-weight: 600;
}

.sidebar-nav-item:focus-visible {
  outline: 2px solid #3b82f6;
  outline-offset: -2px;
}

.nav-icon {
  flex-shrink: 0;
  color: currentColor;
}

.nav-label {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* When sidebar is collapsed, nav items center the icon and hide the label */
.sidebar.collapsed .sidebar-nav-item {
  justify-content: center;
  padding: 0.625rem;
}

.sidebar.collapsed .nav-label {
  display: none;
}

.sidebar-footer {
  padding: 1rem 0.75rem;
  border-top: 1px solid #f1f5f9;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.sidebar-toggle {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0.5rem;
  border-radius: 6px;
  border: 1px solid #e2e8f0;
  background: #ffffff;
  color: #64748b;
  cursor: pointer;
  transition: all 0.15s ease;
}

.sidebar-toggle:hover {
  background: #f8fafc;
  border-color: #cbd5e1;
}

.sidebar-toggle:focus-visible {
  outline: 2px solid #3b82f6;
  outline-offset: -2px;
}

.app-content {
  flex: 1;
  min-width: 0; /* prevent flex-child overflow */
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.main-content {
  flex: 1;
  overflow-y: auto;
  padding: 1.5rem 2rem;
}
</style>
```

**Important notes on this sketch:**
- Replace `navItems` with the app's actual routes from Phase 1.
- Replace each inline SVG `<path>` with the actual icon geometry for that item (grid, box, clipboard, etc.). Keep the `viewBox="0 0 24 24"` size and simple stroke/fill styles consistent with any existing icons in the app.
- Replace every hex color value (`#ffffff`, `#f8fafc`, `#e2e8f0`, `#2563eb`, `#eff6ff`, `#64748b`, etc.) with the app's own exact palette from Phase 1.
- The `companyName` variable, `LanguageSwitcher`/`ProfileMenu` components, and modal imports must match the app's actual structure.
- Every existing global CSS class (`.page-header`, `.stats-grid`, `.card`, etc.) must remain untouched in the `<style>` block — those are your responsibility to preserve, not to duplicate or rewrite here.

## Phase 3: Migrate Nav Items and Wire Up Orphaned Views

Based on Phase 1's discoveries, update the router and add sidebar nav items for ALL routes (both existing and newly-wired orphaned views).

1. **For each registered route from Phase 1:** Create a corresponding sidebar nav item in the `navItems` array (from the Phase 2 code sketch). Preserve the route path and use the label from Phase 1 (either the i18n key or the hardcoded string, as currently found). If multiple routes exist, ensure the nav items are in a sensible order (typically top-to-bottom: Overview/Dashboard first, then product/data-focused views, then reporting/analytics, then ancillary/settings).

2. **For each orphaned view flagged in Phase 1:** Determine a sensible route path following the app's existing convention (kebab-case for multi-word paths, e.g. `/demand-forecast`, not `/demandForecast`). Then:
   - Add a new route entry to the router configuration, importing the view component the same way existing routes import theirs (e.g. `import Backlog from './views/Backlog.vue'`).
   - Add a matching sidebar nav item with an icon + label (using the i18n function if the app's convention is i18n'd nav labels). If no i18n key exists for this view, create a new one (e.g. `nav.backlog`) and add it to the app's locale files.
   - Report the orphaned view(s) to the user/context before executing the wire-up, in case one is intentionally a work-in-progress or excluded for a reason — do not silently assume every orphaned file must be exposed.

3. **Fix any i18n inconsistencies:** If some nav labels use `t('nav.x')` and one is hardcoded as a plain string, fix the hardcoded one to match the pattern while it's being touched anyway.

4. **Never delete or rename existing route paths.** This preserves any bookmarks or deep links users may have created.

**Example (inventory-management app):**
- Existing routes: `/`, `/inventory`, `/orders`, `/spending`, `/demand`, `/reports` — create nav items for all 6, reusing their current labels (5 from i18n, 1 hardcoded "Reports" to be fixed to i18n).
- Orphaned view: `Backlog.vue` exists but has no route. Add route `{ path: '/backlog', component: Backlog }` to the router (importing Backlog from `'./views/Backlog.vue'`). Add a nav item for `/backlog` with label `t('nav.backlog')` and an appropriate icon (e.g. "list"). Add the `nav.backlog` key to the locale files if it doesn't exist.

## Phase 4: Re-anchor Elements Coupled to the Old Header

Find and recalculate any sticky/fixed positioning that was anchored to the old header's height, since the header no longer exists in the content column's stacking context.

1. **Grep for position-relative CSS:** Search the entire client source for rules using `position: sticky` or `position: fixed` with a `top` offset (not `top: 0` or `top: auto`). Focus on offsets that numerically match or are derived from the old header's height (e.g., `top: 70px` if the header was 70px tall).

2. **Cross-check each match:** Read the rule in context to confirm it was indeed anchored to the header's height (not a coincidentally matching number). The FilterBar is the primary example — look for anything else in a similar boat.

3. **Recalculate offsets for the new layout:** Once the header disappears and the sidebar takes its own column, the content column's stacking context changes. A sibling that was "below the header" is now typically the topmost sticky element in that column, so `top: 70px` becomes `top: 0`. If the element now sits inside a scrollable area with nothing fixed above it, remove the offset entirely (or set to `0` and rely on scroll behavior to hide it if it's supposed to scroll away).

4. **Re-evaluate "full viewport centering" logic:** Any `max-width` + `margin: 0 auto` rules on header-coupled siblings assumed full-width content. These likely need to become "fill the content column's width" instead. Common pattern: change `max-width: 1600px; margin: 0 auto;` to no max-width rule (or a column-aware width), since the content column is now narrower than the full viewport.

5. **Search broadly:** Don't just fix the flagged components — grep the entire source for any other hardcoded pixel values that might match the header height, in case more than one place references it.

**Example (inventory-management app):**
- FilterBar's `.filters-bar` has `position: sticky; top: 70px; z-index: 90;`. Change to `top: 0` since FilterBar moves to the top of the content column.
- FilterBar's `.filters-container` has `max-width: 1600px; margin: 0 auto; padding: 0 2rem;`. Remove the `max-width` and `margin: 0 auto` centering, or set `max-width` to `100%`, so it expands to fill the content column's width instead of re-centering within it.

## Phase 5: Delegate Execution to vue-expert

All `.vue` file creation and modification MUST be delegated to the `vue-expert` subagent via the Agent tool. This is a project-wide mandatory rule documented in the root `CLAUDE.md`.

1. **State the delegation rule explicitly:** Remind the context that "Any `.vue` file creation or significant modification must be delegated to the `vue-expert` subagent. The orchestrating agent will not edit `.vue` files directly."

2. **Prepare the delegation prompt:** Bundle all of Phases 2, 3, and 4 into a single, scoped instruction to `vue-expert`. The prompt should cover:
   - **Restructure the root layout component** (App.vue or equivalent) into the new sidebar shell, using the Phase 2 code sketch as a template. Preserve every existing global CSS class (`.page-header`, `.stats-grid`, `.card`, etc.) — only add new sidebar-specific classes.
   - **Add collapse/expand state and toggle logic** (the `sidebarCollapsed` ref, `toggleSidebar()` function, localStorage persistence).
   - **Update the router configuration** with any new routes from Phase 3 (orphaned views that are now wired up).
   - **Update flagged sibling components** (FilterBar, etc.) with recalculated `top` offsets and reconsidered width/centering rules from Phase 4.
   - **Fix i18n inconsistencies** (e.g. the hardcoded "Reports" label becoming `t('nav.reports')`).

3. **Include the Constraints & Non-Goals explicitly in the delegation prompt,** since `vue-expert`'s own guidelines don't mention the "no design tokens" constraint for this specific task:
   - Do NOT introduce CSS custom properties or `:root` variables. Use exact hex values.
   - Do NOT touch `server/` or API contracts.
   - Do NOT rename or remove existing global CSS classes.
   - Preserve i18n conventions; fix any hardcoded label.
   - Keep hand-coded inline SVGs for icons (no new libraries).

4. **Note that `vue-expert` already enforces its own design guidelines** (4px/8px spacing rhythm, no emojis, semantic HTML, card styling) — do not duplicate those rules in your delegation prompt, just hand it a precise scoped task and trust its own conventions.

5. **On completion:** `vue-expert` will report the changes made (root layout restructured, router updated, FilterBar re-anchored, etc.). Proceed to Phase 6.

## Phase 6: Verify with Playwright

Run a browser-based verification sequence to confirm the redesign is functional and visually correct. This phase is read-only; run it directly via Playwright MCP tools (no delegation needed).

1. **Start the dev servers** (if not already running). Use the project's existing `start` skill or run the backend (`python main.py` in `server/`) and frontend (`npm run dev` in `client/`) separately. Confirm both are accessible: http://localhost:8001 (backend) and http://localhost:3000 (frontend).

2. **Navigate to the frontend:** `mcp__playwright__browser_navigate` to `http://localhost:3000/`. Allow the app to fully load.

3. **Capture initial state:** `mcp__playwright__browser_take_screenshot` (or `browser_snapshot`) of the expanded sidebar layout. Visually confirm: logo/brand area at the top, nav items with icons + labels, all 7 routes visible (including the newly-wired Backlog), footer with toggles, collapse button at the bottom.

4. **Check for console errors:** `mcp__playwright__browser_console_messages` to confirm the initial load produced no new JavaScript errors or warnings.

5. **Test each nav route:** For every nav item (including newly-added Backlog), perform:
   - `mcp__playwright__browser_click` on the nav item.
   - `mcp__playwright__browser_snapshot` (or `browser_take_screenshot`) to capture the new page.
   - Verify the correct view renders (e.g., `/inventory` shows the inventory table, `/backlog` shows the backlog stats + table, etc.).
   - Confirm the nav item styling shows the active state (left border accent + tinted background + correct color).

6. **Test collapse/expand:** 
   - `mcp__playwright__browser_click` on the collapse toggle button.
   - `mcp__playwright__browser_snapshot` to confirm the sidebar shrinks to a narrow icon-only rail (~64px wide), labels disappear, icons remain centered.
   - Hover over one or two icon-only nav items (use `browser_hover` or manually test in a browser if Playwright hover isn't reliable) and confirm the `title` attribute displays the label as a tooltip.
   - Click the toggle again to confirm the sidebar expands back to full width (~240px), labels reappear.

7. **Test persistence (if implemented):**
   - If `localStorage` was used to persist the collapsed state, `browser_navigate` to reload the page (same URL, `http://localhost:3000/`).
   - Confirm the sidebar remains in its previous state (collapsed or expanded, as it was before the reload).

8. **Test responsive behavior (optional, out of scope if the app has no mobile/responsive requirements):**
   - `mcp__playwright__browser_resize` to a narrower viewport (e.g. 768px wide, tablet width, or 480px for mobile if the app targets mobile).
   - `mcp__playwright__browser_snapshot` to confirm the layout doesn't visually break, sidebar remains functional, and text remains readable.
   - Note in your report if responsive refinement is out of scope for this pass.

9. **Final verification:**
   - `mcp__playwright__browser_console_messages` one more time to confirm no new errors have appeared during testing.
   - `mcp__playwright__browser_network_requests` (optional) to spot any failed API calls or 4xx/5xx errors.
   - If all checks pass, the redesign is verified and ready for Phase 7.

## Phase 7: Code Review Pass

Delegate a code review to the project's `code-reviewer` subagent, per the root `CLAUDE.md`'s standard practice of reviewing after writing significant code.

1. **Initiate the review:** Use the Agent tool to invoke the `code-reviewer` subagent, specifying the modified files:
   - Root layout component (App.vue or equivalent) — full restructure.
   - Router configuration (main.js or router config file) — new routes added.
   - FilterBar.vue (or other header-coupled siblings) — offset/width logic updated.
   - Any new/modified locale files (if i18n keys were added).

2. **Handle findings:** If `code-reviewer` flags issues (style inconsistencies, missing accessibility attributes, CSS-in-JS anti-patterns, etc.), loop back to Phase 5 — re-delegate the fixes to `vue-expert` rather than editing `.vue` files directly.

3. **Close-out:** Once code-reviewer approves or issues are resolved, the redesign is complete. Summarize the changes: old top-nav replaced with collapsible left sidebar, all routes functional, visual consistency maintained, accessibility checks passed, Playwright verification successful.

## Reference Example

For this specific inventory-management app, the skill's execution maps as follows. Use this table to sanity-check Phase 1's discovery against the actual app if you're testing the skill:

| Phase | Step | Inventory-Management App | Notes |
|-------|------|--------------------------|-------|
| 1 | Root layout | `client/src/App.vue` | `.top-nav` header (70px), 6 nav links, `.main-content`, FilterBar, modals. Global styles in `<style>` block. |
| 1 | Router | `client/src/main.js` | 6 routes: `/`, `/inventory`, `/orders`, `/demand`, `/spending`, `/reports`. Backlog route missing. |
| 1 | Views | `client/src/views/` | 7 files: Dashboard, Inventory, Orders, Demand, Spending, Reports, Backlog (orphaned). |
| 1 | Nav labels | App.vue | All use `t('nav.x')` except "Reports" (hardcoded). |
| 1 | Global palette | App.vue `<style>` | Colors: `#ffffff`, `#f8fafc`, `#e2e8f0`, `#2563eb`, `#eff6ff`, `#475569`, `#64748b`, etc. (hardcoded hex). |
| 1 | Header-coupled | `FilterBar.vue` | `position: sticky; top: 70px; z-index: 90;` on `.filters-bar`. `max-width: 1600px; margin: 0 auto;` on `.filters-container`. |
| 1 | UI libs | `package.json` | None. Icons are hand-coded inline SVGs. |
| 2 | Sidebar width | Expanded: 240px, Collapsed: 64px | CSS `transition: width 0.2s ease`. |
| 2 | Colors | Existing blue `#2563eb` + `#eff6ff` | No new colors introduced. |
| 3 | Orphaned | Backlog.vue | Add route `/backlog` + nav item `nav.backlog` (new i18n key). |
| 4 | Re-anchor | FilterBar | `top: 70px` → `top: 0`; remove `max-width: 1600px; margin: 0 auto;`. |
| 5 | Delegate to | vue-expert | App.vue restructure, router update, FilterBar CSS, i18n fix. |
| 6 | Playwright | localhost:3000 | Navigate, click nav items, toggle collapse, reload for persistence, screenshot. |
| 7 | Review | code-reviewer | App.vue, main.js, FilterBar.vue, locale files. |

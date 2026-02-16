# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

CIPP (CyberDrain Improved Partner Portal) is a multi-tenant Microsoft 365 administration portal for MSPs. It's a **frontend-only** Next.js app that statically exports to `./out` and is deployed to Azure Static Web Apps. The backend API lives in a separate repository (CIPP-API).

## Commands

```bash
npm run dev          # Start dev server on 127.0.0.1:3000
npm run build        # Production build (static export to ./out)
npm run lint         # ESLint check
npm run lint-fix     # ESLint auto-fix
npm run start-swa    # Start with Azure SWA emulator (needs API at :7071)
```

**Node.js ^22.13.0 is required.**

There is no test framework configured in this project.

## Architecture

### Stack
- **Next.js 16** with static export (`output: "export"`) — file-based routing under `src/pages/`
- **React 19**, **MUI 7** (Material UI), **Emotion** for styling
- **TanStack React Query 5** for server state, persisted to localStorage
- **Redux Toolkit** for global UI state (toasts only)
- **Formik + Yup** and **React Hook Form** for forms
- **Axios** for HTTP requests

### Key Directories
- `src/pages/` — Next.js routes. Each page uses `Page.getLayout = (page) => <DashboardLayout>{page}</DashboardLayout>` pattern
- `src/api/ApiCall.jsx` — Three hooks: `ApiGetCall`, `ApiPostCall`, `ApiGetCallWithPagination`. All API calls go through these
- `src/components/CippComponents/` — Domain components: `CippTablePage`, `CippFormPage`, `CippPropertyList`, etc.
- `src/components/CippTable/CippDataTable.js` — Main data table component (~28KB)
- `src/contexts/settings-context.js` — Global settings (current tenant, theme, layout prefs)
- `src/hooks/` — Custom hooks: `use-settings`, `use-auth`, `use-permissions`, `use-guid-resolver`
- `src/utils/` — Helpers: `get-cipp-formatting.js`, `get-cipp-error.js`, `permissions.js`
- `src/data/` — Static JSON: licenses, GDAP roles, portals, standards, Graph Explorer presets
- `src/theme/` — MUI theme with dark/light variants
- `src/store/` — Redux store (toast notifications via `showToast` action)

### Data Fetching Pattern
All API calls use the hooks from `src/api/ApiCall.jsx`:
```jsx
// GET with caching
const data = ApiGetCall({
  url: "/api/ListGraphRequest",
  queryKey: `${tenant}-ListUsers`,
  data: { Endpoint: "users", tenantFilter: tenant },
  waiting: true,
});

// POST with query invalidation (supports wildcard patterns)
const mutation = ApiPostCall({ relatedQueryKeys: ["users*"] });
mutation.mutate({ url: "/api/AddUser", data: formData });
```
Query keys use string-based keys (not arrays). Wildcard invalidation is supported via `*` suffix.

### Page Pattern
Most pages follow this structure:
```jsx
import { CippTablePage } from "/src/components/CippComponents/CippTablePage";
import { Layout as DashboardLayout } from "/src/layouts/index";

const Page = () => {
  const tenant = useSettings().currentTenant;
  return (
    <CippTablePage
      title="Users"
      apiUrl="/api/ListGraphRequest"
      apiData={{ Endpoint: "users", tenantFilter: tenant }}
      actions={[...]}
      filters={[...]}
    />
  );
};
Page.getLayout = (page) => <DashboardLayout>{page}</DashboardLayout>;
export default Page;
```

### Permissions
`src/utils/permissions.js` provides wildcard-matching permission checks. Components like `PermissionButton` and `PermissionCheck` gate UI based on user roles.

### Authentication
Uses Azure Static Web Apps authentication (`/.auth/me` endpoint). The `PrivateRoute` component wraps all pages.

### Multi-Tenant Context
Almost every page depends on `useSettings().currentTenant` to scope API requests to the selected tenant. Always pass `tenantFilter` to API calls when required.

## Code Style
- 2-space indentation, LF line endings, UTF-8
- Max line length: 100 characters
- Single quotes for JS/TS (per editorconfig)
- ESLint extends `next/core-web-vitals` with relaxed rules (no-img-element off, display-name off)
- JSX files use `.js` or `.jsx` extensions
- Use MUI `sx` prop for component styling

## PR Workflow
- PRs must target the `dev` branch, not `main` (enforced by CI)
- All contributions transfer copyright to CyberDrain per the CLA

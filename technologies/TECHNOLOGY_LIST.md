# EcoWaveProjectManagement — Technology List

## Languages

| Technology | Version | Purpose |
|---|---|---|
| C# | .NET 10.0 | Backend application language |
| TypeScript | 6.0.2 | Frontend type-safe JavaScript |
| JavaScript | — | Build configurations and tooling |
| SQL | — | Database queries (via Dapper ORM) |

## Frontend

### Core Framework & Build

| Technology | Version | Purpose |
|---|---|---|
| React | 19.2.6 | UI framework |
| Vite | 8.0.12 | Build tool and dev server |
| @vitejs/plugin-react | 6.0.1 | Vite React plugin |
| TypeScript | 6.0.2 | Type checking |

### Styling

| Technology | Version | Purpose |
|---|---|---|
| Tailwind CSS | 4.3.0 | Utility-first CSS framework |
| @tailwindcss/vite | 4.3.0 | Vite plugin for Tailwind |
| tailwind-merge | 3.6.0 | Merge Tailwind CSS classes |
| tailwindcss-animate | 1.0.7 | Animation utilities |
| class-variance-authority | 0.7.0 | CSS utility for component variants |

### UI Components

| Technology | Version | Purpose |
|---|---|---|
| shadcn/ui | — | Component library (via components.json) |
| Radix UI | — | Headless UI primitives (accordion, avatar, checkbox, dialog, dropdown-menu, label, popover, progress, radio-group, select, separator, slider, slot, switch, tabs, toggle-group, tooltip) |
| Lucide React | 1.17.0 | Icon library |
| sonner | 1.5.0 | Toast notifications |

### Internal Libraries

| Technology | Version | Purpose |
|---|---|---|
| @maccion/manager-ui | — | Internal UI library for desktop/manager interfaces |
| @maccion/operativa-ui | — | Internal UI library for factory floor/touch interfaces |

### State Management & Data Fetching

| Technology | Version | Purpose |
|---|---|---|
| Zustand | 5.0.14 | Client-side state management |
| TanStack Query (React Query) | 5.100.14 | Data fetching and caching |
| TanStack Router | 1.170.10 | Client-side routing |

### Utilities

| Technology | Version | Purpose |
|---|---|---|
| clsx | 2.1.1 | Conditional className utility |

### Development & Tooling

| Technology | Version | Purpose |
|---|---|---|
| pnpm | — | Package manager |
| ESLint | 10.3.0 | Linting |
| @eslint/js | 10.0.1 | ESLint JavaScript config |
| typescript-eslint | 8.59.2 | TypeScript ESLint rules |
| eslint-plugin-react-hooks | 7.1.1 | React hooks linting |
| eslint-plugin-react-refresh | 0.5.2 | React Fast Refresh linting |

### Testing

| Technology | Version | Purpose |
|---|---|---|
| Vitest | 4.1.8 | Unit testing framework |
| @testing-library/react | 16.3.2 | React component testing |
| @testing-library/jest-dom | 6.9.1 | Testing DOM assertions |
| @testing-library/user-event | 14.6.1 | User interaction simulation |
| jsdom | 29.1.1 | DOM environment for testing |

## Backend

### Core Framework

| Technology | Version | Purpose |
|---|---|---|
| .NET | 10.0 | Runtime framework |
| ASP.NET Core Web API | 10.0 | Web framework |
| MSBuild | — | .NET build system |

### Data Access

| Technology | Version | Purpose |
|---|---|---|
| Dapper | 2.1.79 | Micro-ORM for SQL queries |
| Microsoft.Data.SqlClient | 7.0.1 | SQL Server driver |

### Architecture

| Pattern | Description |
|---|---|
| Clean Architecture | Modular project structure |
| Module-based Organization | Separated modules: Projectes, Imatges, Dissenys |
| Layer Pattern | Application → Infrastructure → Domain |
| Dependency Injection | Built-in .NET DI container |

### Testing

| Technology | Version | Purpose |
|---|---|---|
| xUnit | 2.9.3 | .NET testing framework |
| xunit.runner.visualstudio | 3.1.4 | Visual Studio test runner |
| NSubstitute | 5.3.0 | Mocking framework |
| Shouldly | 4.3.0 | Assertion library |
| coverlet.collector | 6.0.4 | Code coverage collection |
| Microsoft.NET.Test.Sdk | 17.14.1 | Test SDK |

## Databases

| Technology | Version | Purpose |
|---|---|---|
| SQL Server | — | Primary relational database (local dev) |
| PostgreSQL | 16 | Containerized deployment database |

## DevOps & CI/CD

| Technology | Version | Purpose |
|---|---|---|
| GitHub Actions | — | CI/CD pipeline automation (backend-ci.yml, frontend-ci.yml) |
| Docker | — | Containerization |
| docker-compose | v3 | Multi-container orchestration |
| GitHub Container Registry (GHCR) | — | Docker image hosting |
| Node.js | 20 | CI runtime for frontend |
| dotnet CLI | — | .NET command-line tooling |

## API & Integration

| Technology | Purpose |
|---|---|
| CORS | Cross-Origin Resource Sharing (dev origin: http://localhost:5173) |
| Health Checks | ASP.NET Core health checks endpoint (/healthz) |
| Vite Proxy | Dev server proxies /api to http://localhost:5000 |

## Version Control & Code Quality

| Technology | Purpose |
|---|---|
| Git | Version control |
| Git Hooks (pre-commit) | Branch guard scripts (branch-guard.sh / branch-guard.ps1) |
| ESLint | JavaScript/TypeScript linting |
| Claude Code (.claude/) | AI-assisted development integration |

## Project Structure

```
EcoWaveProjectManagement/
├── frontend/          # React + Vite + TypeScript (pnpm)
├── backend/
│   └── src/
│       ├── Modules/   # Projectes, Imatges, Dissenys
│       └── Shared/    # SharedInfrastructure (DapperContext)
├── docs/              # Markdown guides, ADRs, requirements
├── topologies/
│   └── monorepo/
│       ├── ci/        # GitHub Actions templates
│       └── hooks/     # Git hook scripts
└── .claude/           # Claude Code configuration
```

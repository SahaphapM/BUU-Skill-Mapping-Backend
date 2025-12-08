# Skill Mapping - Copilot Instructions

## Project Overview

**BUU Skill Mapping** is a full-stack educational platform for managing skills, curricula, and student assessments. This monorepo contains:
- **Frontend**: Vue 3 + Quasar Framework (Bun-based, runs on port 5173)
- **Backend**: NestJS + Prisma + MySQL (Node.js, runs on port 3000)

Both projects use TypeScript with strict type checking.

## Architecture Patterns

### Backend (NestJS + Prisma)

**Module Structure**:
- Each entity (users, curriculums, skills, etc.) has a dedicated module in `src/modules/<entity>/`
- Module includes: `.controller.ts`, `.service.ts`, `.module.ts`
- Generated DTOs from Prisma schema via `prisma-generator-nestjs-dto`
- Database layer via PrismaService (injected as `private prisma: PrismaService`)

**Key Characteristics**:
- Prisma schema (`prisma/schema.prisma`) is the source of truth for DB structure
- Auto-generated DTOs in `src/generated/nestjs-dto/` (prefixes: Create, Update)
- Custom DTOs for filters/pagination in `src/dto/filters/`
- Exception handling: PrismaExceptionFilter catches database errors
- Auth: JWT tokens (access + refresh), strategies in `src/auth/strategies/`

**Database Patterns**:
```bash
npm run db:setup              # First-time setup: migrate + seed
npm run db:reset              # Fresh start: reset migrations + re-seed
npm run prisma:migrate:dev    # Create/apply new migrations
npm run prisma:studio         # GUI for database inspection
```

### Frontend (Vue 3 + Quasar + Pinia)

**Service & Store Pattern**:
- **Services**: `src/services/use-*.ts` composables (pure business logic, no side effects)
- **Stores**: `src/stores/*-store.ts` with Pinia (state management + UI state)
- Services are consumed by stores and components via composables
- Both use `api` from `src/boot/axios/index.ts`

**API Integration**:
- Axios configured with request/response interceptors (`src/boot/axios/index.ts`)
- Automatic token refresh on 401: axios queues requests while refreshing
- Auth state stored in Quasar LocalStorage: `LocalStorage.getItem<AuthPayload>('auth')`
- AccessToken in Authorization header, RefreshToken sent in refresh endpoint

**Component Patterns**:
- Use `<script setup>` with `defineProps`, `defineEmits`, `defineExpose`
- Import services: `const coordinatorService = useServiceCoordinator()`
- Import stores: `const store = useCoordinatorStore()`
- Fetch data in `onMounted`, update both service result and store state
- Tables use Quasar's `q-table` with pagination utilities from `src/utils/pagination`
- Dialogs for forms via `$q.dialog({ component: FormDialog })`
- i18n: `const { t } = useI18n()` for Thai/English translations

**Layout Structure**:
- Routes: `src/router/routes.ts` (main), `curriculumRoutes.ts` (curriculum-specific)
- Pages: `src/pages/*.vue` (role-based: Coordinator, Instructor, Student, etc.)
- Components: Organized by domain (curriculum/, student/, coordinator/, etc.)
- Common utilities: `src/utils/` (pagination, skill types, notifications)

## Critical Developer Workflows

### Backend Development

**Start Development**:
```bash
cd skill-mapping-backend
npm install
cp .env.example .env        # Configure DATABASE_URL and other vars
npm run db:setup            # Migrate + seed
npm run start:dev           # Watch mode (auto-reload)
```

**Database Changes**:
1. Update `prisma/schema.prisma`
2. Run `npm run prisma:migrate:dev --name "descriptive_change_name"`
3. Generated DTOs auto-update in `src/generated/nestjs-dto/`
4. Restart dev server

**Adding New API Endpoint**:
1. Create/update module in `src/modules/<entity>/`
2. Define controller method with `@Get/@Post/@Patch/@Delete` decorators
3. Use auto-generated DTOs for request/response types
4. Inject PrismaService for data access
5. Swagger documentation auto-generated from decorators

### Frontend Development

**Start Development** (requires Bun):
```bash
cd skill-mapping-frontend
bun install                 # Only Bun allowed (npm/yarn/pnpm blocked)
bun run dev                 # Quasar dev server on http://localhost:5173
bun run build               # Production build
```

**Generating TypeScript API Types**:
```bash
bun run generate            # Runs openapi-typescript from backend's /api-json
# Outputs to src/generated.d.ts (auto-included in typescript)
```

**Adding New Service Composable**:
1. Create `src/services/use-<entity>-service.ts`
2. Use `const { data, error } = await api.get/post(...)` pattern
3. Handle errors with axios error detection
4. Return object with multiple functions (pagination, CRUD operations)

**Adding New Page**:
1. Create `src/pages/<EntityName>Page.vue`
2. Fetch data via service in `onMounted`
3. Use Pinia store for shared state
4. Handle pagination with `defaultPagination` utility
5. Use dialogs for forms: `$q.dialog({ component: FormDialog })`

## Naming Conventions

- **Backend**:
  - Entities: singular (user, curriculum, skill, not users)
  - Services: `<Entity>Service`
  - Controllers: `<Entity>Controller`
  - DTOs: `Create<Entity>Dto`, `Update<Entity>Dto`
  - Filter DTOs: `<Entity>FilterDto` in `src/dto/filters/`

- **Frontend**:
  - Stores: `<entity>-store.ts`, export `use<Entity>Store`
  - Services: `use-<entity>-service.ts`, export `use<Entity>Service` (composable function)
  - Components: `<PascalCase>.vue`
  - Pages: `<EntityName>Page.vue`
  - Dialog components: `<Entity>FormDialog.vue`

## Authentication & Security

**Backend Auth Flow**:
- Login generates access token (15m) + refresh token (hashed in DB)
- JwtAuthGuard validates access token on protected routes
- RefreshAuthGuard validates refresh token for `/auth/refresh` endpoint
- Sign out clears hashed refresh token from DB

**Frontend Auth Flow**:
1. Login stores `{ accessToken, refreshToken, id }` in LocalStorage
2. Axios adds Authorization header: `Bearer ${accessToken}`
3. On 401 response: axios calls `/auth/refresh` with refresh token
4. If refresh succeeds: retries queued requests with new token
5. If refresh fails: clears LocalStorage and redirects to `/login`
6. Session expiration notification shown via `useNotification.sessionExpired()`

## External Integrations

- **Google OAuth**: Backend redirects to Google, returns JWT token in URL callback to frontend
- **File Uploads**: Handle via `/public/users/` directory structure
- **CSV Data**: Public CSVs in `public/SoftSkill.csv`, `public/SpecificSkill.csv`

## Common Debugging Tips

1. **API Type Generation Missing**: Run `bun run generate` and restart dev server
2. **Database Sync Issues**: Check `npm run prisma:migrate:status` on backend
3. **Token Refresh Loop**: Verify refresh endpoint works and token format is correct
4. **Component Not Updating**: Ensure store state is updated after service calls
5. **Build Fails**: Clear `bun.lockb` / `node_modules` and reinstall

## Testing

- **Frontend E2E**: Playwright tests in `tests/e2e/` (run with `bun test` or `playwright`)
- **Backend Unit Tests**: Jest in `test/` (run with `npm test`)
- Both use TypeScript with strict checking

## Environment Variables

**Backend (.env)**:
- `DATABASE_URL`: MySQL connection string
- `NODE_ENV`: development/production
- `JWT_SECRET`, `JWT_EXPIRATION`: Access token config
- `REFRESH_TOKEN_SECRET`, `REFRESH_TOKEN_EXPIRATION`: Refresh token config
- `FRONTEND_URL`: For Google OAuth callback redirect
- `GRAYLOG_HOST`: Logging service (optional)

**Frontend (process.env accessed via import.meta.env)**:
- `BACKEND_API`: API base URL (default: `http://localhost:3000`)
- Build-time variables configured in `quasar.config.js`

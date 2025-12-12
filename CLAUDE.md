# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Preferencias del Usuario

**MUY IMPORTANTE:**
- **SIEMPRE comunicarse con el usuario en ESPAÑOL** - Todas las explicaciones, mensajes, y respuestas deben estar en español
- **TODO el código debe escribirse en INGLÉS** - Variables, funciones, comentarios en código, nombres de archivos, etc.
- **Si hay dudas sobre tecnologías poco utilizadas (como Convex) o errores difíciles de resolver:** Buscar en internet usando WebSearch para encontrar documentación actualizada o soluciones

## Comandos de Desarrollo

### Desarrollo Local
```bash
npm run dev          # Inicia el servidor de desarrollo en http://localhost:3000
npm run build        # Compila la aplicación para producción
npm run start        # Inicia el servidor de producción
npm run lint         # Ejecuta ESLint
```

### Convex Backend
```bash
npx convex dev       # Inicia Convex en modo desarrollo (sincroniza funciones)
npx convex deploy    # Despliega funciones a producción
```

## Arquitectura del Proyecto

### Stack Tecnológico
- **Frontend:** Next.js 16 (App Router) + React 19
- **Backend:** Convex (serverless backend con capacidades real-time)
- **Autenticación:** @convex-dev/auth (soporta password, GitHub, Google)
- **Estado:** Jotai para estado del cliente (modales, UI)
- **UI:** Radix UI + shadcn/ui con Tailwind CSS v4
- **TypeScript:** Strict mode con path aliases (`@/*`)

### Estructura de Carpetas

```
src/
├── app/                    # Next.js App Router
├── components/            # Componentes globales compartidos
│   ├── ui/               # Componentes shadcn/ui (Radix-based)
│   └── modals.tsx        # Orquestador global de modales
├── features/             # Módulos por característica
│   ├── auth/
│   │   ├── api/         # React hooks para datos (use-current-user.ts)
│   │   ├── components/  # UI components (auth-screen, sign-in-card, etc.)
│   │   └── types.ts
│   └── workspaces/
│       ├── api/         # Hooks de datos (use-create-workspace, use-get-workspaces)
│       ├── components/  # UI components (create-workspace-modal)
│       └── store/       # Estado Jotai (use-create-workspace-modal)
├── lib/
│   └── utils.ts         # Utilidad cn() para clases de Tailwind
└── middleware.ts        # Protección de rutas con auth

convex/
├── _generated/          # Auto-generado por Convex (no editar)
├── schema.ts           # Definición del schema de la base de datos
├── auth.ts             # Configuración de auth (providers)
├── auth.config.ts      # Config adicional de auth
├── http.ts             # Rutas HTTP (OAuth callbacks)
├── users.ts            # Queries/mutations de usuarios
└── workspaces.ts       # Queries/mutations de workspaces
```

### Arquitectura por Capas

**Feature-Driven Architecture:** Cada feature es un contexto delimitado con su propia estructura:
- `api/` - React hooks que encapsulan useQuery/useMutation de Convex
- `components/` - Componentes UI específicos de la feature
- `store/` - Estado del cliente (Jotai atoms)
- `types.ts` - Definiciones de tipos

**Flujo de Datos:**
```
UI Component → API Hook → Convex Query/Mutation → Database
     ↑                                                ↓
     └────────────── Real-time Updates ──────────────┘
```

## Patrones Clave

### 1. API Hooks Pattern

**Para Queries (lectura):**
```typescript
// Patrón: use-get-{resource}.ts
export const useGetWorkspaces = () => {
  const data = useQuery(api.workspaces.get);
  const isLoading = data === undefined;
  return { data, isLoading };
};
```

**Para Mutations (escritura):**
```typescript
// Patrón: use-create-{resource}.ts
type Options = {
  onSuccess?: (id: Id<"workspaces">) => void;
  onError?: (error: Error) => void;
  onSettled?: () => void;
}

export const useCreateWorkspace = () => {
  const mutation = useMutation(api.workspaces.create);

  const mutate = useCallback(async (
    values: { name: string },
    options?: Options
  ) => {
    try {
      const workspaceId = await mutation(values);
      options?.onSuccess?.(workspaceId);
    } catch (error) {
      options?.onError?.(error as Error);
    } finally {
      options?.onSettled?.();
    }
  }, [mutation]);

  return { mutate };
};
```

### 2. Convex Backend Functions

**Queries (auto-subscriben a cambios):**
```typescript
export const get = query({
  args: {},
  handler: async (ctx) => {
    const userId = await auth.getUserId(ctx);
    if (!userId) throw new Error("Unauthorized");

    return await ctx.db
      .query("workspaces")
      .filter((q) => q.eq(q.field("userId"), userId))
      .collect();
  },
});
```

**Mutations:**
```typescript
export const create = mutation({
  args: {
    name: v.string(),
  },
  handler: async (ctx, args) => {
    const userId = await auth.getUserId(ctx);
    if (!userId) throw new Error("Unauthorized");

    const joinCode = generateCode(); // Implementar generador

    const workspaceId = await ctx.db.insert("workspaces", {
      name: args.name,
      userId,
      joinCode,
    });

    return workspaceId;
  },
});
```

**IMPORTANTE:** Siempre verificar autenticación con `await auth.getUserId(ctx)` en mutations y queries protegidas.

### 3. Estado con Jotai

**Para UI state simple (modales, toggles):**
```typescript
import { atom, useAtom } from "jotai";

const modalState = atom(false);

export const useCreateWorkspaceModal = () => {
  return useAtom(modalState);
};
```

**Uso:**
```typescript
const [open, setOpen] = useCreateWorkspaceModal();
```

### 4. Modales Globales

Todos los modales se registran en `src/components/modals.tsx`:

```typescript
export const Modals = () => {
  const [mounted, setMounted] = useState(false);

  useEffect(() => setMounted(true), []);
  if (!mounted) return null; // Previene hydration mismatch

  return (
    <>
      <CreateWorkspaceModal />
      {/* Registrar nuevos modales aquí */}
    </>
  );
};
```

**Beneficio:** Previene problemas de hidratación SSR y centraliza la gestión de modales.

### 5. Componentes UI (shadcn/ui)

**Variantes con CVA:**
```typescript
const buttonVariants = cva(
  "inline-flex items-center justify-center...",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground...",
        destructive: "bg-destructive text-destructive-foreground...",
        // ...
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 px-3",
        // ...
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
);
```

**Uso del helper cn():**
```typescript
import { cn } from "@/lib/utils";

<div className={cn("base-classes", conditional && "extra-classes")} />
```

## Autenticación

### Flujo de Autenticación

1. **Middleware (`src/middleware.ts`):** Protege todas las rutas excepto `/auth`
2. **Providers:** Envuelven la app en `src/app/layout.tsx`
   - `ConvexAuthNextjsServerProvider` (servidor)
   - `ConvexAuthNextjsProvider` (cliente)
3. **Auth Hooks:**
   - `useAuthActions()` - Proporciona `signIn()`, `signOut()`
   - `useCurrentUser()` - Retorna usuario actual o null
4. **Backend:** `convex/users.ts` define query `current` que obtiene usuario actual

### Providers Configurados
- Password personalizado (con campo name)
- GitHub OAuth
- Google OAuth

### Obtener Usuario Actual

```typescript
const { data: user, isLoading } = useCurrentUser();

if (isLoading) return <div>Loading...</div>;
if (!user) return <div>Not authenticated</div>;
```

## Schema de Base de Datos

**Definido en `convex/schema.ts`:**

```typescript
import { authTables } from "@convex-dev/auth/server";
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  ...authTables, // Tablas de auth (users, sessions, etc.)

  workspaces: defineTable({
    name: v.string(),
    userId: v.id("users"),
    joinCode: v.string(),
  }),

  // Agregar nuevas tablas aquí
});
```

**IMPORTANTE:**
- Usar validators de Convex (`v.string()`, `v.id()`, etc.)
- Incluir `authTables` para funcionalidad de autenticación
- Los cambios de schema requieren reiniciar `npx convex dev`

## Convenciones de Código

### Naming Conventions
- **Hooks:** `use-{verb}-{noun}.ts` (ej: `use-create-workspace.ts`)
- **Componentes:** PascalCase (ej: `CreateWorkspaceModal`, `SignInCard`)
- **Archivos:** kebab-case para archivos, PascalCase para componentes
- **Variables/Funciones:** camelCase en inglés

### Imports
- Usar path alias: `@/` para imports desde `src/`
- Convex API: `import { api } from "../../../../convex/_generated/api"`
- Tipos generados: `import { Id } from "../../../../convex/_generated/dataModel"`

### TypeScript
- Modo strict habilitado
- Evitar `any`, usar tipos específicos
- Props de componentes siempre tipadas
- Convex genera tipos automáticamente desde el schema

## Tailwind CSS v4

**Variables CSS personalizadas en `src/app/globals.css`:**
```css
@theme {
  --color-primary: /* ... */;
  --color-background: /* ... */;
  /* etc. */
}
```

**Dark Mode:** Soportado vía clase `.dark`

## Troubleshooting

### Errores Comunes de Convex

1. **"No current deployment"**: Ejecutar `npx convex dev`
2. **Schema changes no reflejados**: Reiniciar `npx convex dev`
3. **Auth errors**: Verificar que `CONVEX_DEPLOYMENT` esté configurado
4. **TypeScript errors en imports de Convex**: Asegurarse de que `convex/_generated/` exista

### Si los Errores Persisten
- **Buscar en internet** usando WebSearch para documentación actualizada de Convex
- Revisar https://docs.convex.dev para la documentación oficial
- Verificar versiones de paquetes en `package.json`

## Estado Actual del Proyecto

**Completado:**
- ✅ Autenticación multi-provider funcional
- ✅ UI components (shadcn/ui) configurados
- ✅ Schema de workspaces definido
- ✅ Estructura de features establecida

**En Progreso:**
- ⚠️ Hook `useCreateWorkspace` - Implementación del callback incompleta
- ⚠️ Generación de join codes - Usar placeholder "123456" actualmente
- ⚠️ Routing a workspaces específicos - Solo hay redirección console.log

**Próximos Pasos:**
- Implementar generación de join codes únicos
- Completar flujo de creación de workspace
- Agregar routing dinámico `/workspace/[workspaceId]`
- Implementar funcionalidad de unirse a workspace con join code

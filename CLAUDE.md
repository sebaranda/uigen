# CLAUDE.md

Este archivo provee orientación a Claude Code (claude.ai/code) para trabajar con el código de este repositorio.

## Descripción del proyecto

UIGen es un generador de componentes React con IA y preview en tiempo real. Los usuarios describen componentes en un chat, Claude genera código React mediante tool calls, y un iframe en sandbox renderiza el resultado en tiempo real.

## Comandos de desarrollo

```bash
npm run setup          # Primera vez: instala deps + prisma generate + migrate
npm run dev            # Servidor de desarrollo con Turbopack
npm run build          # Build de producción
npm run lint           # ESLint
npm test               # Vitest (modo watch)
npm test -- path/to/file.test.tsx  # Archivo de test individual
npm run db:reset       # Resetea la base de datos SQLite (destructivo)
```

Migraciones de Prisma:
```bash
npx prisma migrate dev --name nombre_migracion
npx prisma generate
```

## Arquitectura

### Flujo principal
1. El usuario envía un mensaje → `src/app/api/chat/route.ts` lo recibe junto con el estado actual del sistema de archivos
2. El Vercel AI SDK transmite la respuesta de Claude con tool calls (`streamText`, hasta 40 pasos)
3. Dos herramientas de IA modifican una instancia de `VirtualFileSystem`:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`): crear, ver y editar archivos mediante reemplazo de strings
   - `file_manager` (`src/lib/tools/file-manager.ts`): renombrar y eliminar archivos/directorios
4. El cliente renderiza los resultados de las tools al sistema de archivos virtual y actualiza el preview

### Sistema de archivos virtual
`src/lib/file-system.ts` — Árbol en memoria basado en `Map` de objetos `FileNode`. No se escriben archivos en disco. Se serializa a JSON para persistencia en base de datos.

### Sistema de preview
`src/components/preview/PreviewFrame.tsx` renderiza los componentes en un iframe en sandbox:
- `src/lib/transform/jsx-transformer.ts` convierte JSX a módulos ES via Babel Standalone
- Crea import maps del navegador con blob URLs para cada módulo
- Punto de entrada: `/App.jsx` (o `.tsx`, `/index.jsx`, `/index.tsx`)

### Enrutamiento
- `/` — Usuarios anónimos acceden a `MainContent` directamente; usuarios autenticados son redirigidos al proyecto más reciente
- `/[projectId]` — Página de proyecto autenticado (redirige a `/` si no está logueado)
- `/api/chat` — Endpoint de streaming (duración máxima 120s)

### Gestión de estado
Dos providers de React Context envuelven la app en `MainContent`:
- `FileSystemProvider` (`src/lib/contexts/file-system-context.tsx`) — envuelve VirtualFileSystem
- `ChatProvider` (`src/lib/contexts/chat-context.tsx`) — gestiona mensajes y streaming de IA

### Auth y persistencia
- Sesiones JWT via `jose` (`src/lib/auth.ts`)
- Prisma + SQLite (`prisma/schema.prisma`): modelos `User` y `Project`
- Output del cliente Prisma: `src/generated/prisma` (ubicación no estándar)
- Los usuarios anónimos pueden usar la app sin persistencia
- `src/lib/anon-work-tracker.ts` rastrea trabajo anónimo no guardado

### Mock Provider
Cuando `ANTHROPIC_API_KEY` no está configurada, `MockLanguageModel` en `src/lib/provider.ts` devuelve ejemplos de componentes estáticos (contador, formulario, tarjeta). Útil para desarrollo/testing sin costos de API. El mock usa 4 pasos máximo vs 40 de la API real.

## Convenciones clave

- Todos los imports usan el alias `@/` que apunta a `src/`
- Componentes UI de shadcn/ui (estilo new-york) en `src/components/ui/`, configurados via `components.json`
- Íconos: `lucide-react`
- Los componentes generados por IA deben usar `/App.jsx` como punto de entrada, Tailwind para estilos y exports por defecto
- El system prompt de generación está en `src/lib/prompts/generation.tsx`

## Testing

Vitest con entorno jsdom y React Testing Library. Los tests viven en directorios `__tests__` junto a su fuente:
- `src/components/**/__tests__/*.test.tsx`
- `src/lib/**/__tests__/*.test.ts`

## Stack tecnológico

- Next.js 15 (App Router) + React 19 + TypeScript
- Anthropic Claude (claude-sonnet-4-6) via Vercel AI SDK
- Prisma + SQLite
- Tailwind CSS v4
- Monaco Editor
- Babel Standalone (JSX→ESM)
- Primitivos de Radix UI

## Variables de entorno

```
ANTHROPIC_API_KEY=...   # Opcional; se usa el mock provider si no está presente
JWT_SECRET=...          # Por defecto: 'development-secret-key'
```

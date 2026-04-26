# agents.md — Azar Site

> Instrucciones y contexto para agentes de IA trabajando en este proyecto.

## Descripción del Proyecto

**Azar** es una landing page + suite de herramientas de aleatoriedad, construida como sitio
completamente estático (solo frontend). No existe SSR, backend ni servidor — todo el cómputo
ocurre en el navegador del usuario.

El concepto central: **la entropía del cursor del usuario alimenta el generador de números
aleatorios**. Mientras el usuario mueve el ratón, sus coordenadas, velocidad y timing se mezclan
en un pool de entropía que luego sirve de semilla para un PRNG xorshift32.

---

## Stack Técnico

| Tecnología | Versión | Rol |
|-----------|---------|-----|
| [Astro](https://astro.build) | ^6.1.9 | Framework SSG — compilado a HTML/CSS/JS puro |
| [Tailwind CSS](https://tailwindcss.com) | ^4.2.4 | Estilos — vía `@tailwindcss/vite` |
| TypeScript | (via Astro) | Tipado en scripts `<script>` de componentes |

**Sin** SSR adapters, **sin** Node/Cloudflare/Vercel. El build produce un `dist/` completamente estático.

---

## Comandos

```bash
npm run dev      # servidor de desarrollo en localhost:4321
npm run build    # compilar a dist/
npm run preview  # previsualizar el build
```

---

## Arquitectura

```
src/
├── layouts/
│   ├── Layout.astro          # Layout raíz: entropy engine, cursor, HUD
│   └── ToolPage.astro        # Layout de herramienta individual (nav, título, slot)
├── pages/
│   ├── index.astro           # Landing: Hero + grid de cards que linkean a herramientas
│   ├── sorteo.astro          # /sorteo
│   ├── dados.astro           # /dados
│   ├── cartas.astro          # /cartas
│   ├── numero.astro          # /numero
│   ├── moneda.astro          # /moneda
│   └── ruleta.astro          # /ruleta
├── components/
│   ├── Hero.astro            # Hero + explicación de entropía
│   ├── ToolsSection.astro    # Grid de cards (landing) — cada card linkea a su página
│   └── tools/
│       ├── Sorteo.astro      # 01 — Sorteo de personas
│       ├── Dados.astro       # 02 — Lanzamiento de dados (D4-D100)
│       ├── Cartas.astro      # 03 — Cartas (francesa y española)
│       ├── Numero.astro      # 04 — Número en rango personalizado
│       ├── Moneda.astro      # 05 — Cara o Cruz (con stats visuales)
│       └── Ruleta.astro      # 06 — Ruleta de opciones de texto
└── styles/
    └── global.css            # Tailwind + tokens @theme + animaciones globales
```

---

## Sistema de Diseño

Los tokens de diseño se definen en `src/styles/global.css` usando la directiva `@theme` de
Tailwind v4, haciéndolos disponibles como clases utilitarias (`text-gold`, `bg-card`, etc.):

| Token | Valor | Uso |
|-------|-------|-----|
| `--color-void` | `#080810` | Fondo base más oscuro |
| `--color-card` | `#16162a` | Fondo de tarjetas |
| `--color-gold` | `#d4a843` | Acento principal |
| `--color-amber` | `#f5c842` | Acento hover/resultado |
| `--color-ember` | `#e8732a` | Acento cálido |
| `--color-crimson` | `#c2384a` | Error / palo rojo |
| `--color-violet` | `#7b5ea7` | Gradiente entropía |
| `--font-display` | `Bebas Neue` | Títulos grandes |
| `--font-serif` | `Cormorant Garamond` | Texto descriptivo |
| `--font-mono` | `DM Mono` | Labels, badges, código |

---

## Motor de Entropía

Definido en `src/layouts/Layout.astro` (`<script>`), expone dos globales:

```typescript
window.azarRandom: () => number   // RNG [0, 1) seeded por entropía del cursor
window.azarPool:   Uint32Array    // El pool de entropía crudo (256 uint32)
```

**Todos los componentes de herramientas usan `(window as any).azarRandom()`** — nunca `Math.random()`.

El algoritmo: los eventos `mousemove` XOR-mezclan coordenadas y timing en el pool. Al
solicitar un número, se toman 8 valores del pool, se mezclan con el timestamp actual, y
se aplica xorshift32.

---

## Reglas para Agentes

1. **Nunca usar `Math.random()`** — siempre `(window as any).azarRandom()`.
2. **No agregar SSR** — este sitio es 100% estático. No instalar adapters de Astro.
3. **No instalar librerías externas** sin necesidad clara — el proyecto debe ser liviano.
4. **Usar Tailwind v4** — los tokens están en `@theme {}` en `global.css`. No crear un
   `tailwind.config.js`; eso es Tailwind v3.
5. **Nuevas herramientas** van en `src/components/tools/NombreHerramienta.astro`, una
   página en `src/pages/nombre.astro` (usando `ToolPage.astro`), y una card en
   `ToolsSection.astro`. Las herramientas NO se renderizan en el landing directamente.
6. **El efecto spotlight** en tarjetas requiere la clase CSS `spotlight-card` más el listener
   `mousemove` local en cada componente. El layout lo aplica globalmente a todas las
   `.spotlight-card` presentes en la página.
7. **Animaciones**: usar las clases `animate-float-up`, `animate-fade`, `animate-dice-shake`
   definidas en `global.css`. Agregar delays con `.delay-{100..500}`.
8. **Performance del cursor**: el cursor usa `transform: translate3d()` con `will-change: transform`
   y `contain: strict`. Nunca mover el cursor con `left/top`. Nunca hacer `querySelectorAll`
   en `mousemove` sin throttle con `requestAnimationFrame`.
9. **Entropía móvil**: el motor ya colecta de `touchmove`, `touchstart`, `click` y
   `devicemotion`. No agregar más listeners de entropía en los componentes individuales.

---

## Herramientas Existentes

| # | Archivo | Descripción |
|---|---------|-------------|
| 01 | `Sorteo.astro` | Fisher-Yates shuffle, selección de N ganadores o lista ordenada completa |
| 02 | `Dados.astro` | D4/D6/D8/D10/D12/D20/D100 + rango personalizado, hasta 20 dados |
| 03 | `Cartas.astro` | Baraja francesa (52 cartas) y española (40 cartas), con/sin repetición |
| 04 | `Numero.astro` | Entero aleatorio en rango [min, max] con animación de rodillo |
| 05 | `Moneda.astro` | Cara o Cruz, hasta 20 lanzamientos con estadísticas |
| 06 | `Ruleta.astro` | Texto libre con opciones por línea, animación de barrido |

---

## Ideas para Futuras Herramientas

- **Generador de Contraseña** — caracteres aleatorios configurables (longitud, charset)
- **Generador de UUID/GUID** — v4 usando el pool de entropía
- **Color Aleatorio** — paleta hex con preview visual
- **Nombre Aleatorio** — combinaciones de adjetivo + sustantivo
- **Generador de Equipos** — divide una lista de personas en N equipos balanceados
- **Temporizador de Pomodoro Aleatorio** — duración aleatoria dentro de rango
- **Dado de Texto** — facetas de texto personalizado (tipo: "Sí / No / Quizás / Nunca")

---

## Skills Disponibles

Los skills se encuentran en `.agents/skills/`. Relevantes para este proyecto:

- `astro` — Referencia rápida de CLI, estructura de proyecto y configuración
- `frontend-design` — Guía de diseño para interfaces premium y distintivas
- `tailwind-css-patterns` — Patrones y mejores prácticas de Tailwind v4

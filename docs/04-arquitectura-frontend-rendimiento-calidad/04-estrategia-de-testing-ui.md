# Estrategia de Testing en UI

## Jest vs. Vitest

Ambos son test runners compatibles con una API muy similar (`describe`, `it`, `expect`), así que el conocimiento se transfiere entre uno y otro sin mucho esfuerzo.

- **Jest:** el histórico, usado en la mayoría de proyectos Create React App y muchos backends Node.
- **Vitest:** construido sobre Vite, nativo en ESM, arranca y ejecuta pruebas notablemente más rápido en proyectos que ya usan Vite (o Next.js con configuración moderna) porque reutiliza la misma configuración de bundling en vez de tener su propio pipeline de transformación separado.

Para un proyecto nuevo con Vite, Vitest es la opción por defecto hoy. En un proyecto que ya usa Jest y funciona bien, migrar no es obligatorio — la ganancia de velocidad no siempre justifica el costo de migración.

## La filosofía de React Testing Library (RTL)

La idea central de RTL, resumida en su propio lema: **"cuanto más se parezcan tus tests a la forma en que el software es usado, más confianza te dan."**

Esto tiene una consecuencia práctica muy concreta: RTL evita a propósito que accedas a detalles internos de implementación (estado interno de un componente, nombres de métodos privados). En vez de eso, buscas elementos como lo haría un usuario real:

- `getByRole` (botón, campo de texto, encabezado) — la forma preferida, porque además verifica accesibilidad de forma indirecta (si no tiene un rol accesible, el test no lo encuentra).
- `getByLabelText` — para campos de formulario, buscando por su label asociado.
- `getByText` — para contenido visible.
- `getByTestId` — el último recurso, cuando no hay una forma semántica de identificar el elemento. Abusar de `data-testid` es una señal de que el HTML no es lo suficientemente semántico (conecta directamente con lo que viste en accesibilidad, punto 9).

La razón de fondo: si un test depende de un detalle interno de implementación, refactorizar el componente (sin cambiar su comportamiento) rompe el test aunque nada esté realmente roto. Eso es un test frágil, no un test útil.

## Mockeo de HTTP con MSW (Mock Service Worker)

Al testear un componente que hace fetch a una API, hay dos formas de mockear esa llamada:

- Mockear la función `fetch`/`axios` directamente (`jest.mock`) — funciona, pero acopla el test a *cómo* el componente hace la petición (qué librería usa), no a *qué* devuelve el servidor.
- **MSW:** intercepta la petición a nivel de red (usando un Service Worker en el navegador, o interceptando `http`/`https` en Node para tests). El componente hace un fetch real, MSW lo intercepta antes de que salga a internet y responde con los datos que definiste. Esto significa que el mismo mock sirve tanto para desarrollo local como para tests, y el componente se testea exactamente como se comporta en producción, sin acoplarte a los detalles de qué librería HTTP usa por dentro.

## Testear estados de carga, error y vacío — no solo el "happy path"

Un componente que consume datos de una API típicamente tiene al menos 4 estados posibles, y cada uno merece su propio test:

1. **Cargando:** ¿se muestra un spinner o skeleton mientras espera la respuesta?
2. **Éxito con datos:** el caso feliz — los datos se renderizan correctamente.
3. **Éxito sin datos (vacío):** ¿qué se muestra cuando la API responde una lista vacía? (un mensaje "no hay resultados", no una pantalla en blanco confusa).
4. **Error:** si la API falla o responde con un error, ¿el usuario ve un mensaje claro, o la app se queda colgada/rota?

Testear solo el caso 2 es la razón más común por la que un componente "pasa todos los tests" en CI pero falla en producción la primera vez que la API tarda, devuelve vacío, o cae — exactamente el mismo argumento de cobertura real (no solo cobertura de líneas) que vimos en la estrategia de testing del punto 2.

# Accesibilidad Digital (a11y)

## ¿Qué es accesibilidad y por qué importa?

Accesibilidad significa diseñar productos que las personas con discapacidad o alguna limitación sensorial puedan usar de forma efectiva. Según la OMS, alrededor del **16% de la población mundial** vive con algún tipo de discapacidad — diseñar accesible no es un nicho, es diseñar para una fracción enorme de tus usuarios reales.

Es fácil pensar en accesibilidad como algo que solo afecta a "otras personas", pero las limitaciones también pueden ser temporales o situacionales: leer la pantalla del celular bajo el sol directo, o pausar un video con la voz porque tienes las manos ocupadas cocinando, son formas de discapacidad situacional. El diseño accesible termina mejorando la experiencia para todos, no solo para quien tiene una discapacidad permanente.

Accesibilidad es un subconjunto de usabilidad enfocado específicamente en personas con discapacidad, y debe considerarse en cada etapa del proceso de diseño — no como un checklist que se aplica al final.

## Los 4 principios de la WCAG: POUR

El estándar global de accesibilidad web es el **WCAG** (Web Content Accessibility Guidelines), definido por el W3C. Sus 4 principios se resumen como **POUR**:

| Principio | Qué significa | Ejemplo de implementación |
|---|---|---|
| **Perceivable** (Perceptible) | El contenido debe poder percibirse sin importar qué sentido use el usuario. | Texto alternativo en imágenes, subtítulos en video. |
| **Operable** (Operable) | La interfaz debe poder operarse de distintas formas. | Toda la funcionalidad debe funcionar con teclado, sin límites de tiempo agresivos. |
| **Understandable** (Comprensible) | El contenido y la interfaz deben ser fáciles de entender. | Texto legible, comportamiento predecible, mensajes de error claros. |
| **Robust** (Robusto) | El contenido debe funcionar de forma confiable con tecnologías actuales y futuras. | HTML semántico que interpretan bien los lectores de pantalla, sin importar el dispositivo. |

Estos 4 principios son el marco de referencia contra el que se evalúa cualquier auditoría de accesibilidad.

## Categorías de discapacidad y consideraciones de diseño

Las discapacidades que afectan la accesibilidad web se agrupan en 4 categorías. Además, pueden ser **permanentes** (ceguera), **temporales** (un brazo fracturado) o **situacionales** (ver un video sin sonido en el transporte público).

- **Visuales:** ceguera total, baja visión, daltonismo, sensibilidad a la luz. Se apoyan en lectores de pantalla y magnificadores. Impacto en diseño: contraste de color suficiente (ver [Teoría del Color](./05-teoria-del-color.md) y [Color para UI](./06-color-para-ui.md)), tipografía legible, layouts que soporten zoom sin romperse, nunca depender solo del color para comunicar información.
- **Auditivas:** desde pérdida parcial hasta sordera total. Se apoyan en subtítulos, transcripciones y alertas visuales. Impacto en diseño: todo contenido de audio necesita subtítulo o transcripción, y ninguna interacción crítica debe depender solo del sonido.
- **Motoras:** artritis, Parkinson, lesiones de columna, túnel carpiano, o una lesión temporal. Se apoyan en navegación por teclado, comandos de voz y dispositivos de switch/eye-tracking. Impacto en diseño: toda funcionalidad debe ser operable por teclado, objetivos de toque grandes, sin gestos complejos ni combinaciones simultáneas de teclas.
- **Cognitivas:** dislexia, TDAH, autismo, dificultades de memoria o aprendizaje. Se apoyan en recordatorios externos, pasos simplificados y reducción de distracciones. Impacto en diseño: lenguaje directo, revelar información de forma gradual, patrones de interfaz consistentes, minimizar contenido en auto-reproducción, mensajes de error claros con ruta de recuperación.

## Checklist práctico para diseño de interfaces

**Imágenes y medios**
- El texto alternativo (`alt`) describe el *propósito* de la imagen, no su apariencia — un botón de búsqueda usa `alt="buscar"`, no `alt="lupa"`.
- Las imágenes puramente decorativas usan `alt=""` para que el lector de pantalla las ignore.
- Todo video necesita subtítulos y, si es posible, transcripción y audiodescripción para contenido visual relevante.

**Formularios**
- Cada campo necesita una etiqueta (`label`) visible y asociada — nunca solo un placeholder, porque desaparece al escribir y el lector de pantalla pierde la referencia.
- Los campos obligatorios se marcan con texto ("(requerido)"), no solo con color.
- Los mensajes de error explican qué está mal y cómo corregirlo: "El correo debe incluir @" es útil; "Dato inválido" no lo es.

**Navegación por teclado**
- Toda la interfaz debe poder recorrerse sin mouse, con un indicador de foco visible en todo momento.
- Evitar "keyboard traps" — menús o reproductores donde el foco queda atrapado y no se puede salir con teclado.
- Los "skip links" (enlaces para saltar directo al contenido principal) evitan que un usuario de teclado tenga que recorrer todo el header en cada página.

**Estructura y contenido**
- Jerarquía de encabezados (`h1`, `h2`, `h3`) sin saltos de nivel — los lectores de pantalla la usan como tabla de contenidos.
- Los títulos de página deben describir claramente el contenido, porque el lector de pantalla los anuncia primero.
- El texto debe poder ampliarse hasta 200% sin romper el layout ni forzar scroll horizontal.

**Movimiento y contenido dinámico**
- Todo contenido en auto-reproducción por más de 5 segundos necesita control de pausa.
- El contenido que parpadea no debe superar 3 destellos por segundo (riesgo de convulsiones).

**Objetivos táctiles**
- En móvil, los elementos interactivos deben medir al menos 44×44 píxeles, con espacio suficiente para evitar toques accidentales.

## HTML semántico y ARIA

La base de una página accesible es el **HTML semántico**: usar `<header>`, `<nav>`, `<main>`, `<article>`, `<section>`, `<footer>` en vez de `<div>` genéricos para todo. Un `<div>` con estilos de menú de navegación *se ve* como un menú, pero un lector de pantalla no lo reconoce como tal — un `<nav>` sí. Esto también genera automáticamente un orden de tabulación (tab order) lógico para navegación por teclado.

Cuando el HTML semántico no alcanza (componentes interactivos custom: modales, tabs, comboboxes), se usan atributos **ARIA** (Accessible Rich Internet Applications) para comunicar rol, estado y relación entre elementos a las tecnologías asistivas — por ejemplo, `aria-label="Más información"` en un botón que solo muestra un ícono de "?".

Una nota importante desde la experiencia práctica: la primera regla de ARIA es *"no uses ARIA si el HTML nativo ya resuelve el problema"*. Un `<button>` nativo ya trae foco, rol y comportamiento de teclado correctos gratis; un `<div role="button">` obliga a reimplementar todo eso a mano, y es fácil dejar algo roto (por ejemplo, olvidar el manejo de la tecla Enter/Espacio). ARIA es para cuando construyes algo que el HTML no cubre, no un reemplazo por defecto.

## Diseño responsivo y accesibilidad

Los problemas de accesibilidad no son iguales en mobile y desktop. En mobile son comunes: objetivos de toque muy pequeños, gestos complejos, pinch-to-zoom deshabilitado, modales que bloquean el contenido, o funcionalidad que depende solo del touch sin alternativa de teclado (relevante si el usuario conecta un teclado físico o usa un lector de pantalla táctil como VoiceOver/TalkBack). En desktop, el punto crítico suele ser que componentes con hover (carruseles, menús desplegables) tengan también una vía de acceso por teclado.

Diseñar mobile-first ayuda aquí: si la versión más restringida (pantalla chica, un solo dedo, sin hover) funciona bien, escalar hacia desktop suele ser más simple que al revés.

## Cómo se verifica: auditorías y pruebas con usuarios reales

**Auditoría de accesibilidad:** evaluación (automática + manual) contra los 4 principios POUR, que detecta problemas técnicos como contraste insuficiente, `alt` faltante, o foco de teclado roto. Herramientas reales que hacen esto:

- **axe DevTools** (Deque) — extensión de navegador, también integrable como librería en tests automatizados.
- **Lighthouse** — integrado en Chrome DevTools, incluye una categoría de accesibilidad con puntaje.
- **WAVE** (WebAIM) — extensión de navegador con overlays visuales sobre los problemas encontrados.
- **eslint-plugin-jsx-a11y** — para proyectos React/Next.js, detecta problemas de accesibilidad directamente en el JSX antes de llegar a producción.

Estas herramientas detectan lo *técnicamente* incorrecto, pero no pueden decirte si la experiencia real es buena para alguien que de verdad usa un lector de pantalla todos los días — para eso hace falta **testing con usuarios reales con discapacidad**, reclutados con lenguaje respetuoso y sensibilidad (nunca asumir capacidades a partir del tipo de discapacidad). Ahí, las métricas cambian: un usuario de lector de pantalla puede tardar más en completar una tarea y aun así sentirse totalmente conforme con la experiencia — el tiempo de tarea deja de ser la métrica central; importan más la tasa de finalización, la frecuencia de errores y si el usuario logra completar la tarea sin ayuda externa.

## Por qué esto no es "opcional" en la práctica

Más allá del argumento ético, hay dos razones prácticas para no dejar accesibilidad para el final:

1. **Riesgo legal:** normativas como la ADA en Estados Unidos o la European Accessibility Act en la Unión Europea (aplicable desde junio de 2025) exigen accesibilidad digital en productos públicos y de e-commerce; varios países de Latinoamérica tienen marcos similares en desarrollo. Litigar por incumplimiento es cada vez más común.
2. **Costo de retrofit:** corregir accesibilidad al final de un proyecto (cambiar estructura HTML, rehacer componentes sin soporte de teclado) es mucho más caro que construirla desde el diseño y los componentes base del design system — la misma lógica de "corregir temprano es más barato" que ya vimos con deuda técnica en el [punto 2](../02-calidad-de-codigo-y-arquitectura/04-deuda-tecnica.md).

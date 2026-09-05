# Prompts Útiles: Fundamentos de Diseño UI/UX

Prompts base para pedir ayuda de forma estructurada al generar o revisar diseños — copiar, completar los corchetes, usar.

## Generar una propuesta de diseño

> Necesito diseñar [pantalla o flujo] para [contexto del producto: qué hace, quién lo usa]. Ayúdame a generar una propuesta de diseño aplicando estos fundamentos:
>
> - Jerarquía visual clara: qué debe verse primero, según el objetivo del usuario en esta pantalla.
> - Un esquema de color (marca + acento + semánticos) coherente, con contraste que cumpla WCAG (mínimo 4.5:1), sin depender solo del color para comunicar estados (error/éxito/advertencia).
> - Tipografía: máximo 2 familias, jerarquía con h1/h2/h3 sin saltos de nivel, tamaño mínimo 16px en cuerpo, texto que soporte zoom hasta 200% sin romper el layout.
> - Al menos 3 heurísticas de Nielsen aplicadas explícitamente — indica cuáles y cómo.
> - Un layout (grid, columna única, o split-screen) justificado según el tipo de contenido.
> - Accesibilidad (POUR): estructura pensada en HTML semántico (`header`, `nav`, `main`), toda interacción operable por teclado con foco visible, objetivos táctiles de al menos 44×44px, etiquetas de formulario visibles (nunca solo placeholder), y texto alternativo funcional para cualquier ícono o imagen con significado.
>
> Al final, revisa tu propia propuesta contra las 10 heurísticas de usabilidad y contra los 4 principios POUR de accesibilidad, y dime si hay alguna que se esté rompiendo.

## Criticar un diseño ya existente

> Tengo este diseño/mockup para [pantalla o flujo]: [descripción o captura]. Evalúalo aplicando:
>
> - Las 10 heurísticas de Nielsen — dime cuáles se cumplen y cuáles se rompen, con ejemplos concretos.
> - Los principios de Gestalt (proximidad, similitud, jerarquía) — ¿el agrupamiento visual comunica bien las relaciones entre elementos?
> - Contraste y accesibilidad — ¿el texto cumple el mínimo de 4.5:1? ¿hay algo que dependa solo del color para comunicarse? ¿la navegación funciona por teclado con foco visible? ¿la estructura usa HTML semántico y jerarquía de encabezados correcta?
> - Consistencia con un design system (si aplica) — ¿reusa componentes ya definidos, o improvisa estilos nuevos?
>
> Prioriza los 3 problemas más importantes, no una lista exhaustiva.

## Elegir una paleta de color con criterio

> Estoy eligiendo colores para [producto/marca], con este tono/personalidad: [ej: confiable y moderno / juguetón / corporativo]. Sugiéreme un esquema de color (monocromático, complementario, análogo o triádico) con un color de marca, uno de acento, y los colores semánticos (éxito/error/advertencia). Verifica que el contraste entre texto y fondo cumpla WCAG 4.5:1 en cada combinación.

## Revisar tipografía y jerarquía de texto

> Tengo este contenido de texto para una pantalla: [pegar contenido o describir las secciones]. Ayúdame a definir la jerarquía tipográfica (qué va en h1/h2/h3, qué tamaño y peso), verificando que el cuerpo de texto tenga al menos 16px y las líneas no superen los 50-75 caracteres.

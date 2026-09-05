# Design Systems

## Guía de estilo vs. Design System vs. Component Library vs. Pattern Library

Se suelen confundir estos 4 términos, pero cada uno cubre algo distinto:

- **Guía de estilo (style guide):** reglas generales de marca — colores, tipografías, tono de comunicación. Ejemplo: Material Design (Google), Human Interface Guidelines (Apple). Es conceptual, da las reglas, no los componentes ya construidos.
- **Component library:** repositorio de componentes reutilizables ya armados (botones, inputs, menús) que se copian directo a un nuevo diseño o código.
- **Pattern library:** repositorio de patrones — layouts que combinan varios componentes con una estructura de contenido específica (ej: un patrón de login = header + inputs + botón + mensajes de error).
- **Design System:** el conjunto completo — guía de estilo + component library + pattern library, documentado y mantenido por el equipo.

## Cómo se arma un Design System

1. **Inventario:** recopilar colores, tipografías, márgenes y componentes que ya existen.
2. **Acordar estándares:** decidir con el equipo cómo se van a usar de acá en adelante.
3. **Componentes reutilizables:** construir versiones definitivas de botones, formularios, menús, etc.

## Cómo se lleva a código: Design Tokens

Los valores del design system (colores, espaciados, tamaños de fuente) se representan como datos, típicamente JSON:

```json
{
  "color-primary": "#1e90ff",
  "spacing-small": "8px",
  "font-size-base": "16px"
}
```

Esto se traduce a variables de CSS/Sass, o al theme de un framework de UI (ej: styled-components, configuración de Tailwind), haciendo que todo el proyecto use exactamente los mismos valores en vez de que cada desarrollador tipee su propio hex de color.

## Ejemplos públicos reales

Google Material Design, Apple Human Interface Guidelines, Microsoft Fluid Design System, Atlassian Design System, Shopify Polaris — se pueden revisar como referencia real de cómo se documenta un design system completo.

## Por qué importa para un desarrollador

Un design system evita reinventar la rueda en cada pantalla nueva, reduce la "deuda visual" (inconsistencias que se acumulan con el tiempo), y facilita el onboarding de alguien nuevo al equipo — con un vistazo al design system, ya sabe qué componentes existen y cómo se usan, sin tener que preguntar.

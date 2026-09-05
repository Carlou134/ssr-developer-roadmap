# Interaction Design (IxD)

Diseñar cómo el usuario interactúa con un producto — es una subdisciplina de UX, enfocada específicamente en acciones como tipear, hacer scroll, deslizar, tocar o hacer hover.

## Affordances y Signifiers

Una **affordance** es lo que un objeto sugiere que se puede hacer con él, dadas sus propiedades. Una manija de puerta que se puede agarrar sugiere "tirar"; si en realidad hay que empujarla, la experiencia resulta confusa — el objeto está "mintiendo" sobre su affordance.

Un **signifier** es la señal perceptible de esa affordance — lo que comunica que una acción es posible. Un link celeste y subrayado es el signifier clásico de "esto se puede clickear".

## Las 5 dimensiones de Interaction Design

Definidas por Gillian Crampton Smith y Kevin Silver:

1. **Palabras:** etiquetas e instrucciones — claras y concisas.
2. **Representaciones visuales:** íconos, imágenes, tipografía — usando convenciones ya aceptadas (ej: el ícono "x" para cerrar).
3. **Objetos físicos y espacio:** el dispositivo (mouse vs. touch) y el contexto del usuario (ej: una app de GPS usada mientras se conduce).
4. **Tiempo:** animaciones, sonidos, y cuánto tiempo pasa el usuario usando el producto.
5. **Comportamiento:** cómo se combinan las 4 dimensiones anteriores para definir la interacción completa.

## Patrones de diseño comunes

Soluciones ya probadas para problemas recurrentes:

- **Indicadores de progreso:** barras de progreso (muestran %), steppers (muestran pasos completados/restantes), spinners (solo indican que algo pasa, sin más información — frustrantes en procesos largos).
- **Date picker:** ¿calendario, campo de texto, o híbrido? ¿fecha única o rango de fechas?
- **Resultados de búsqueda:** ¿lista, grilla de tarjetas, o tabla? ¿qué orden por defecto? ¿qué filtros disponibles?

## Dark Patterns

Término acuñado por **Harry Brignull** (2010) para patrones de diseño deliberadamente engañosos:

- **Sneak into basket:** agregar un producto extra al carrito sin que el usuario lo pida explícitamente.
- **Confirmshaming:** redactar la opción de "no" de forma que avergüence al usuario para que termine aceptando.
- **Misdirection:** dirigir la atención hacia una cosa para distraer de otra.

Generan ganancias de corto plazo, pero destruyen la confianza en el producto con el tiempo — casi siempre violan alguna de las 10 heurísticas de Nielsen (ver [02-heuristicas-de-usabilidad.md](./02-heuristicas-de-usabilidad.md)).

# Wireframes y Prototipos

## La diferencia

- **Wireframe:** el "esqueleto" del producto — estructura y funcionalidad, sin colores, imágenes ni estilos finales. Se hace al principio, antes de los mockups.
- **Prototipo:** una versión más detallada e interactiva. De baja fidelidad (wireframes con links clickeables para simular navegación) o de alta fidelidad (diseño visual completo, animaciones, comportamiento realista).

El flujo típico es **wireframe → prototipo → desarrollo**, iterando y validando con usuarios en cada paso, antes de escribir código real.

## Pasos para un buen wireframe

1. Definir el objetivo (¿para qué es esta pantalla?).
2. Bocetar (en papel, o con herramientas como Figma o Balsamiq).
3. Recién después, elegir tipografías, colores e íconos.
4. Validar poniéndose en el lugar de un usuario nuevo: ¿qué vería primero? ¿a dónde iría?

## Figma para desarrolladores ("Figma for Dev")

Permite ver y copiar CSS directo del diseño (colores, tamaños, bordes, fuentes), medir distancias sin preguntarle al diseñador, y exportar assets listos para usar. Conceptos que se traducen casi 1 a 1 a CSS:

- **Auto Layout** en Figma = Flexbox en CSS.
- **Frames** = contenedores de pantalla.
- El modelo de caja en Figma (padding, margin, border) funciona igual que el Box Model de CSS.

Esto acelera pasar de un diseño a código real, sin depender de que alguien pase specs a mano.

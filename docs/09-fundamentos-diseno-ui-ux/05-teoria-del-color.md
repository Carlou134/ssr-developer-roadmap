# Teoría del Color

## El círculo cromático

Herramienta para visualizar la relación entre colores, compuesta por tres grupos:

- **Primarios:** rojo, azul, amarillo — la base de todos los demás colores.
- **Secundarios:** verde, naranja, púrpura — se crean mezclando dos primarios.
- **Terciarios:** mezcla de un primario con un secundario adyacente (ej: rojo-naranja, azul-verde).

## HSL (el sistema que más le sirve a un diseñador)

Los códigos hexadecimales y RGB no se ajustan de forma intuitiva ("hacé este color un poco más brillante" no se traduce fácil a un cambio de hex). HSL es más semántico:

- **Hue (tono):** el color puro, expresado como un ángulo (0-360°) alrededor del círculo cromático.
- **Saturation (saturación):** intensidad del color — 100% es el color más vívido, 0% es escala de grises.
- **Lightness (luminosidad):** qué tan claro u oscuro es — 0% es negro, 100% es blanco.

Aplicar blanco a un color sube su luminosidad (crea un **tinte**); aplicar negro la baja (crea una **sombra**).

## Colores cálidos y fríos

- **Cálidos:** del rojo al amarillo, incluido el naranja — transmiten energía, calidez, a veces agresividad.
- **Fríos:** azul, púrpura, verde — transmiten calma.

## Esquemas de color

| Esquema | Cómo se arma | Efecto |
|---|---|---|
| **Monocromático** | Un solo color, variando tintes/sombras | Armonía, sensación organizada |
| **Complementario** | Dos colores opuestos en el círculo | Alto contraste, llamativo |
| **Análogo** | Tres o más colores adyacentes | Bajo contraste, relajante |
| **Triádico** | Tres colores equidistantes (forman un triángulo) | Equilibrio con variedad |
| **Tetrádico** | Cuatro colores equidistantes (forman cuadrado/rectángulo) | Máxima variedad, más difícil de balancear |

## Psicología del color

Cada color evoca asociaciones — algunas biológicas/evolutivas (el verde asociado a comida/agua/refugio, el rojo a peligro), otras culturales (el rojo en Occidente indica alerta/error; en China, riqueza — hasta la bolsa de valores se representa al revés: en EE.UU. baja = rojo, en China baja = verde). No hay una regla universal — depende del contexto cultural del usuario objetivo.

## Contraste y accesibilidad

- Colores opuestos en el círculo tienen más contraste; colores adyacentes, menos.
- Las Web Content Accessibility Guidelines (WCAG) definen un ratio de contraste mínimo de **4.5:1** entre texto y fondo — se puede verificar con la herramienta WebAIM Contrast Checker.
- Existen distintos tipos de daltonismo (rojo-verde, azul-amarillo, monocromático total). Por eso nunca conviene depender solo del color para comunicar algo (ej: un ✓ verde de éxito debería ir acompañado de un ícono o texto, no solo del color).

## Buenas prácticas

- Usar colores neón con moderación — cansan la vista.
- Evitar combinaciones de alta saturación que "vibran" (ej: rojo + verde saturados).
- Evitar bajo contraste (claro sobre claro, oscuro sobre oscuro) aunque técnicamente se pueda leer.

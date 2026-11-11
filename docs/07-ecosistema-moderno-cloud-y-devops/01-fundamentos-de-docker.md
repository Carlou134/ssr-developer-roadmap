# Fundamentos de Docker

## En una frase

Docker es una herramienta para empaquetar una aplicación con todas sus dependencias en una **imagen** y ejecutarla como un **contenedor**: un proceso aislado que **comparte el kernel del sistema anfitrión**, y por eso arranca en segundos y se comporta igual en cualquier máquina que tenga Docker.

-----

## Antes de empezar

Conviene que ya sepas:

* Usar una terminal: navegar carpetas, ejecutar comandos, leer la salida.
* Qué es un proceso del sistema operativo (un programa en ejecución con su propia memoria).
* Nociones de Linux: los contenedores de Docker son, en la práctica, procesos Linux.

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Imagen (*image*):** plantilla de solo lectura con el sistema de archivos y la configuración necesarios para ejecutar una aplicación.
* **Contenedor (*container*):** instancia en ejecución de una imagen. Es un proceso aislado.
* **Kernel:** el núcleo del sistema operativo. Gestiona CPU, memoria, disco y red para todos los procesos.
* **Docker daemon (`dockerd`):** servicio en segundo plano que crea y gestiona imágenes, contenedores, redes y volúmenes.
* **Docker CLI:** el comando `docker` que escribes en la terminal. Solo le envía órdenes al daemon.
* **Registry:** servidor donde se guardan y descargan imágenes. El público por defecto es Docker Hub.
* **Máquina virtual (*VM*):** computadora emulada con su propio sistema operativo completo, incluido su propio kernel.
* **Namespaces:** función del kernel de Linux que hace que un proceso vea solo "su" parte del sistema (sus procesos, su red, sus archivos).
* **cgroups (*control groups*):** función del kernel de Linux que limita cuánta CPU, memoria o disco puede usar un proceso.

-----

## El problema

Tienes una API en Node.js. En tu máquina funciona perfecto. Se la pasas a un compañero y le pasa esto:

```text
Tu máquina:            Node 20, Windows, variables en tu .env
Máquina del compañero: Node 16, macOS, sin .env
Servidor de prod:      Node 18, Ubuntu, otra versión de OpenSSL
```

```text
$ npm start
SyntaxError: Unexpected token '?'
```

El código es el mismo. Lo que cambia es **todo lo que lo rodea**: la versión del runtime, las librerías del sistema, las variables de entorno y el sistema operativo.

Las consecuencias:

* el clásico "en mi máquina sí funciona", y horas perdidas buscando qué es distinto;
* incorporar a alguien nuevo al equipo lleva días de "instala esto, ahora esto otro";
* el servidor de producción termina siendo una pieza única que nadie se anima a tocar.

-----

## Cómo funciona

### 1. Imagen y contenedor

La relación es la misma que entre una **clase** y un **objeto**:

| Concepto | Analogía en POO | Qué es |
| --- | --- | --- |
| Imagen | Clase | Plantilla inmutable: sistema de archivos + metadatos (qué comando ejecutar, qué puerto expone) |
| Contenedor | Objeto (instancia) | Un proceso vivo creado a partir de la imagen, con una capa propia donde puede escribir |

De una misma imagen puedes levantar diez contenedores, y cada uno tiene su propio estado.

```bash
docker run hello-world
```

Ese comando descarga la imagen `hello-world` (si no la tienes), crea un contenedor a partir de ella, lo ejecuta, imprime un mensaje y el contenedor termina.

> **Ojo:** un contenedor vive **mientras vive su proceso principal**. Cuando ese proceso termina, el contenedor se detiene. No es una "máquina" que queda encendida.

### 2. Contenedor vs. máquina virtual

Esta es la diferencia que más se pregunta y la que más se explica mal.

```text
        MÁQUINAS VIRTUALES                       CONTENEDORES

  ┌────────┐ ┌────────┐ ┌────────┐      ┌────────┐ ┌────────┐ ┌────────┐
  │ App A  │ │ App B  │ │ App C  │      │ App A  │ │ App B  │ │ App C  │
  │ Libs   │ │ Libs   │ │ Libs   │      │ Libs   │ │ Libs   │ │ Libs   │
  │ SO +   │ │ SO +   │ │ SO +   │      └────────┘ └────────┘ └────────┘
  │ kernel │ │ kernel │ │ kernel │      ┌──────────────────────────────┐
  └────────┘ └────────┘ └────────┘      │        Docker Engine         │
  ┌──────────────────────────────┐      ├──────────────────────────────┤
  │          Hipervisor          │      │  SO anfitrión (UN kernel)    │
  ├──────────────────────────────┤      ├──────────────────────────────┤
  │           Hardware           │      │           Hardware           │
  └──────────────────────────────┘      └──────────────────────────────┘
```

Una VM **virtualiza el hardware**: cada una arranca su propio sistema operativo completo. Un contenedor **virtualiza el sistema operativo**: es un proceso normal del anfitrión al que el kernel le "miente" con dos mecanismos:

* **Namespaces:** el proceso cree que es el único del sistema (ve el PID 1), que tiene su propia red y su propio sistema de archivos.
* **cgroups:** el kernel le pone un límite de CPU y memoria.

| | Máquina virtual | Contenedor |
| --- | --- | --- |
| Qué aísla | Hardware completo | Procesos, red y archivos |
| Kernel | Uno propio por VM | Compartido con el anfitrión |
| Tamaño típico | Gigabytes | Megabytes |
| Arranque | Minutos | Segundos o menos |
| Aislamiento de seguridad | Más fuerte | Más débil (comparten kernel) |

> **Ojo:** "comparte el kernel" es **la** idea de esta lección. De ahí sale que sea liviano y rápido, que sea menos aislado que una VM y el límite del punto 4.

Cuándo conviene cada una, cómo manejan los recursos, la seguridad y el estado, y por qué en producción se usan juntas: [Máquinas virtuales vs. contenedores](02-maquinas-virtuales-vs-contenedores.md).

### 3. Arquitectura: CLI, daemon y registry

Docker es cliente-servidor:

```text
  Tú                  Docker Engine (en tu máquina)            Internet
┌──────────┐  API   ┌──────────────────────────────┐  pull  ┌─────────────┐
│ docker   │ ─────► │ dockerd (daemon)             │ ◄───── │  Registry   │
│ (CLI)    │        │  ├─ imágenes                 │        │ (Docker Hub)│
└──────────┘        │  ├─ contenedores             │ ─────► │             │
                    │  ├─ redes                    │  push  └─────────────┘
                    │  └─ volúmenes                │
                    └──────────────────────────────┘
```

Cuando escribes `docker run nginx`:

1. La CLI le envía la orden al daemon por una API.
2. El daemon busca la imagen `nginx` en local. Si no está, la descarga del registry (*pull*).
3. El daemon crea el contenedor y arranca su proceso.

Si lo usas mal (el daemon no está corriendo, por ejemplo con Docker Desktop cerrado):

```text
$ docker ps
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

La CLI está instalada, pero no tiene a quién darle la orden.

### 4. "Funciona igual en todas partes"... con letra chica

Los apuntes de curso suelen decir "funciona igual sin importar el sistema operativo". Es **casi** cierto, y la parte que falta es importante:

* **Un contenedor Linux necesita un kernel Linux.** En Windows y macOS, Docker Desktop crea una VM Linux liviana (en Windows usa WSL2) y los contenedores corren **dentro** de esa VM. Funciona, pero no es magia: hay una VM escondida.
* **La arquitectura de CPU importa.** Una imagen compilada para `amd64` (Intel/AMD) no corre directamente en `arm64` (Mac con Apple Silicon, Raspberry Pi). Docker puede emularla, pero es más lento y te avisa:

```text
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
```

Y si no hay emulación disponible:

```text
exec /usr/local/bin/app: exec format error
```

Lo que Docker sí garantiza: **todo lo que está dentro de la imagen** (runtime, librerías, configuración) es idéntico en cualquier máquina. Lo que no garantiza: lo que está fuera (kernel y CPU).

### 5. Docker, Kubernetes y microservicios no son lo mismo

Se mezclan porque suelen aparecer juntos, pero están en niveles distintos:

| Término | Qué es | Nivel |
| --- | --- | --- |
| **Microservicios** | Un **estilo de arquitectura**: dividir la aplicación en servicios pequeños que se despliegan por separado | Diseño |
| **Docker** | Una **herramienta** para construir imágenes y ejecutar contenedores en **una** máquina | Empaquetado y ejecución |
| **Kubernetes** | Un **orquestador**: gestiona muchos contenedores en **muchas** máquinas (reparte la carga, reinicia los que fallan, escala) | Operación a escala |

Puedes tener cualquiera sin los otros:

* un **monolito** dentro de un contenedor Docker, sin microservicios ni Kubernetes (muy común y totalmente válido);
* microservicios desplegados sin contenedores (en VMs o en servicios *serverless*);
* Kubernetes ejecutando contenedores construidos con otra herramienta (Podman, Buildah).

> **Ojo:** Kubernetes ya no usa Docker para ejecutar contenedores. Usa runtimes como `containerd`, que es justamente el que Docker usa por dentro. Las imágenes que construyes con Docker siguen el estándar **OCI** y funcionan igual en Kubernetes.

-----

## Ejemplo completo

El mismo problema del principio: el compañero tiene Node 16 y tu código necesita Node 20. Con Docker, la versión de Node **viaja dentro de la imagen**.

En la terminal (no hace falta tener Node instalado):

```bash
# 1. Qué Node tiene tu máquina (si es que tiene)
node --version

# 2. Qué Node hay dentro de la imagen
docker run --rm node:20-alpine node --version

# 3. Prueba de que el contenedor comparte TU kernel
docker run --rm alpine uname -r
```

Resultado:

```text
v16.20.2
v20.18.0
5.15.167.4-microsoft-standard-WSL2
```

Qué observar:

* **Paso 2:** da la misma versión en tu máquina, en la del compañero y en el servidor, porque Node viene dentro de la imagen `node:20-alpine`. Esa es la solución a "en mi máquina funciona".
* **Paso 3:** `uname -r` muestra la versión del kernel. No es un kernel de Alpine: es el del anfitrión (en Windows, el kernel de la VM de WSL2). Es la prueba en vivo de que el contenedor **no** tiene un sistema operativo completo propio.
* `--rm` borra el contenedor cuando termina, para no acumular contenedores detenidos.
* Cada `docker run` crea un contenedor nuevo. Arranca en menos de un segundo porque no hay sistema operativo que iniciar.

-----

## Errores comunes

**1. Pensar que un contenedor es una VM liviana.**
Qué pasa: intentas "entrar al contenedor e instalar cosas" como si fuera un servidor, y al recrearlo pierdes todo.
Por qué: un contenedor es un proceso efímero creado desde una imagen inmutable. Lo que escribes dentro vive en una capa que se borra con el contenedor.
Arreglo: los cambios permanentes van en la imagen (con un `Dockerfile`) y los datos que deben sobrevivir van en volúmenes. Ambos temas vienen en las próximas lecciones.

**2. Confundir imagen con contenedor.**
Qué pasa: borras un contenedor y crees que perdiste la aplicación, o actualizas la imagen y el contenedor viejo sigue con la versión anterior.
Por qué: el contenedor se crea a partir de la imagen **en un momento dado**. No se actualiza solo.
Arreglo: para usar una imagen nueva, se crea un contenedor nuevo desde ella.

**3. Creer que "Docker = Kubernetes" o que "usar Docker = hacer microservicios".**
Qué pasa: se agrega complejidad de orquestación a un proyecto que es un solo servicio.
Por qué: se mezclan un estilo de arquitectura, una herramienta y un orquestador.
Arreglo: empieza con Docker (y después Docker Compose) en un solo servicio. Kubernetes se justifica cuando tienes muchos contenedores en muchas máquinas.

**4. Asumir que funciona igual en cualquier CPU.**
Qué pasa: `exec format error` en producción o en la Mac de un compañero.
Por qué: la imagen se construyó para otra arquitectura (`amd64` vs. `arm64`).
Arreglo: usar imágenes multi-arquitectura o indicar la plataforma al construir (`--platform`).

-----

## Evolución y versiones (opcional)

* **2008:** Linux incorpora cgroups y namespaces; aparece LXC, que ya permite contenedores.
* **2013:** sale Docker. No inventa los contenedores: los hace **fáciles** (imágenes portables, `Dockerfile`, registry).
* **2015:** se crea la OCI (*Open Container Initiative*) para estandarizar el formato de imágenes y los runtimes.
* **2020:** Kubernetes anuncia que deja de usar Docker como runtime (se retiró en 2022 con la versión 1.24) y pasa a usar `containerd`. Las imágenes de Docker siguen funcionando porque son OCI.

-----

## Cuándo sí y cuándo no

**Úsalo cuando:**

* quieres el mismo entorno en desarrollo, CI y producción;
* necesitas levantar dependencias locales (base de datos, Redis, colas) sin instalarlas en tu máquina;
* tienes que desplegar en la nube: casi todas las plataformas aceptan una imagen de contenedor.

**No lo uses cuando:**

* necesitas aislamiento fuerte de seguridad entre clientes que no confían entre sí: una VM aísla más porque no comparte el kernel;
* la aplicación depende de otro kernel (por ejemplo, software de escritorio de Windows sobre un anfitrión Linux);
* es un script personal que corre una sola vez en tu máquina: el costo de empaquetarlo no se recupera.

-----

## Resumen en 5 líneas

1. Una **imagen** es la plantilla inmutable; un **contenedor** es un proceso aislado creado a partir de ella.
2. Un contenedor **comparte el kernel** del anfitrión; una VM trae el suyo. Por eso el contenedor es más liviano y rápido, y también menos aislado.
3. El aislamiento lo ponen **namespaces** (qué ve el proceso) y **cgroups** (cuánto puede consumir).
4. Docker es cliente-servidor: la **CLI** le da órdenes al **daemon**, que descarga imágenes de un **registry**.
5. Docker empaqueta y ejecuta, Kubernetes orquesta a escala y los microservicios son un estilo de arquitectura: son tres cosas distintas.

-----

## Para profundizar (opcional)

<details>
<summary>¿Qué pasa exactamente cuando ejecutas <code>docker run</code>?</summary>

La CLI llama al daemon (`dockerd`). El daemon delega en `containerd`, que gestiona el ciclo de vida del contenedor, y este usa `runc`, el componente de bajo nivel que pide al kernel los namespaces y cgroups y arranca el proceso. Por eso Kubernetes pudo dejar de usar Docker: habla directamente con `containerd` y se salta la capa de `dockerd`.

```text
docker (CLI) → dockerd → containerd → runc → kernel (namespaces + cgroups)
```

</details>

<details>
<summary>¿Por qué una imagen pesa megabytes y no gigabytes?</summary>

Porque no trae kernel ni drivers: solo el sistema de archivos mínimo que necesita la aplicación. `alpine` pesa unos 8 MB. Además, las imágenes se arman en **capas** reutilizables: si diez imágenes usan la misma base, esa base se descarga y se guarda una sola vez. Las capas se ven en la lección de imágenes y `Dockerfile`.

</details>

-----

## En entrevista (opcional)

### Respuesta corta (junior)

Docker empaqueta una aplicación con sus dependencias en una imagen y la ejecuta como contenedor, que es un proceso aislado. A diferencia de una VM, el contenedor no trae su propio sistema operativo: comparte el kernel del anfitrión, por eso es liviano y arranca rápido.

### Respuesta ampliada (semi-senior)

El aislamiento lo da el kernel de Linux con namespaces y cgroups; Docker aporta el formato de imagen portable (OCI), el registry y las herramientas. Compartir el kernel trae ventajas (densidad, arranque rápido, entornos reproducibles) y límites: el aislamiento de seguridad es más débil que el de una VM, un contenedor Linux necesita un kernel Linux (en Windows y macOS hay una VM de por medio) y la imagen depende de la arquitectura de CPU. Docker resuelve el empaquetado y la ejecución en un host; la orquestación en varios hosts es tarea de Kubernetes u otro orquestador, y nada de esto obliga a usar microservicios.

### Preguntas frecuentes de seguimiento

**1. ¿Un contenedor es más seguro que una VM?**
No. Aísla menos porque todos los contenedores comparten el kernel: una vulnerabilidad del kernel puede afectar a todos. Cuando el aislamiento es crítico se usan VMs o tecnologías intermedias como gVisor o Firecracker.

**2. ¿Cómo corre Docker en Windows si los contenedores son Linux?**
Docker Desktop levanta una VM Linux liviana con WSL2 y los contenedores corren dentro de ella. También existen contenedores Windows nativos, pero son otra cosa y necesitan un anfitrión Windows.

**3. ¿Kubernetes necesita Docker?**
No. Usa runtimes compatibles con CRI, como `containerd` o CRI-O. Las imágenes construidas con Docker funcionan porque siguen el estándar OCI.

-----

## Práctica

**Ejercicio 1.** Ejecuta estos dos comandos y explica por qué la primera vez tarda y la segunda es casi instantánea:

```bash
docker run --rm alpine echo hola
docker run --rm alpine echo hola
```

<details>
<summary>Solución</summary>

La primera vez el daemon no encuentra la imagen `alpine` en local, así que la descarga del registry (verás `Unable to find image 'alpine:latest' locally` y luego `Pulling from library/alpine`). La segunda vez la imagen ya está en local y solo crea y arranca el contenedor, que no tiene sistema operativo que iniciar: es lanzar un proceso.

</details>

**Ejercicio 2.** ¿Qué tiene de malo esta afirmación, sacada de un README?

> "Usamos Docker, así que nuestra app es de microservicios y funciona idéntica en cualquier servidor, incluido el de ARM que compramos."

<details>
<summary>Solución</summary>

Tiene dos errores:

1. **Docker no implica microservicios.** Docker es una herramienta de empaquetado; los microservicios son un estilo de arquitectura. Un monolito en un contenedor sigue siendo un monolito.
2. **"Idéntica en cualquier servidor" ignora la arquitectura de CPU.** Si la imagen se construyó solo para `amd64`, en el servidor ARM dará `exec format error` o correrá emulada y lenta. Hay que publicar una imagen multi-arquitectura (`linux/amd64` y `linux/arm64`).

</details>

-----

## Siguiente lección

[Máquinas virtuales vs. contenedores](02-maquinas-virtuales-vs-contenedores.md)

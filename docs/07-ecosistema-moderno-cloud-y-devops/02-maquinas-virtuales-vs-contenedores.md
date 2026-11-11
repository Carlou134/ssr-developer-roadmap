# Máquinas virtuales vs. contenedores

## En una frase

Una máquina virtual **virtualiza el hardware** y un contenedor **virtualiza el sistema operativo**. Elegir entre una y otro es decidir entre **aislamiento fuerte** y **eficiencia**, y en la práctica casi siempre se usan **juntos**: los contenedores corren dentro de VMs.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es una imagen, un contenedor y por qué el contenedor comparte el kernel: [Fundamentos de Docker](01-fundamentos-de-docker.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Hipervisor (*hypervisor*):** software que crea y ejecuta máquinas virtuales y les reparte el hardware.
* **Sistema operativo invitado (*guest OS*):** el sistema operativo que corre dentro de una VM.
* **Snapshot:** copia del estado completo de una VM en un momento dado (disco, memoria, configuración), a la que se puede volver.
* **Sobreasignación (*overcommit*):** asignar a las VMs más recursos de los que el equipo tiene físicamente, confiando en que no todas los usen a la vez.
* **Escape de contenedor (*container escape*):** cuando un proceso sale del aislamiento del contenedor y obtiene acceso al anfitrión.
* **OOM (*Out Of Memory*):** cuando un proceso supera la memoria permitida y el kernel lo mata.

-----

## El problema

Un equipo toma dos decisiones el mismo día:

**Decisión A.** Para el entorno local de desarrollo (API, PostgreSQL y Redis) alguien propone "una VM por servicio, para que quede bien aislado". En una laptop de 16 GB:

```text
VM api       → 4 GB RAM reservados, 20 GB de disco, arranca en ~1 min
VM postgres  → 4 GB RAM reservados, 20 GB de disco, arranca en ~1 min
VM redis     → 4 GB RAM reservados, 20 GB de disco, arranca en ~1 min
              ─────────────────────
              12 GB de RAM ocupados antes de abrir el IDE
```

**Decisión B.** Para el laboratorio de ciberseguridad, donde van a ejecutar malware real para analizarlo, alguien propone "un contenedor, que es más liviano":

```bash
docker run --rm -it --privileged -v /:/host kali-lab
```

Las consecuencias:

* **A:** la laptop se arrastra, cada servicio carga un sistema operativo completo y nadie quiere levantar el entorno;
* **B:** con `--privileged` y el disco del anfitrión montado, el malware tiene acceso directo a la máquina real. El "aislamiento" es casi nulo.

Las dos decisiones están **al revés**: A pedía contenedores y B pedía una VM.

-----

## Cómo funciona

### 1. Hipervisores: lo que hay debajo de una VM

El hipervisor es la pieza que hace posible la VM. Hay dos tipos:

| | Tipo 1 (*bare metal*) | Tipo 2 (*hosted*) |
| --- | --- | --- |
| Dónde corre | Directamente sobre el hardware | Como un programa dentro de tu sistema operativo |
| Ejemplos | VMware ESXi, Hyper-V, KVM, Xen | VirtualBox, VMware Workstation, Parallels |
| Uso típico | Servidores y nubes (AWS, Azure, GCP) | Laptops, aprendizaje, pruebas |

Cada VM trae su propio sistema operativo invitado con su propio kernel. Por eso puedes correr Windows dentro de Linux, o al revés. Un contenedor **no puede** hacer eso: usa el kernel del anfitrión.

### 2. Recursos: reservar vs. consumir a demanda

Aquí hay un matiz que los apuntes de curso suelen simplificar.

**VM:** le asignas recursos al crearla (4 GB de RAM, 2 CPU, 40 GB de disco) y el sistema invitado los trata como su hardware. El sistema operativo invitado ocupa una parte solo para existir (kernel, servicios, caché). Los hipervisores pueden sobreasignar, pero el modelo mental es "esta porción del equipo es de la VM".

**Contenedor:** no reserva nada por defecto. Es un proceso más y consume lo que necesita, igual que cualquier programa.

```bash
docker run -d --name cache redis:7-alpine        # sin límite
docker run -d --name cache --memory 128m redis:7-alpine   # con límite (cgroups)
```

Si lo usas mal: un contenedor sin límite con una fuga de memoria puede consumir toda la RAM del anfitrión y afectar a todos los demás procesos. Con límite, el kernel mata solo a ese contenedor:

```text
$ docker ps -a
CONTAINER ID   IMAGE        STATUS                       NAMES
3f2a9c1b7e44   mi-api       Exited (137) 5 seconds ago   api

$ docker inspect api --format '{{.State.OOMKilled}}'
true
```

El código de salida `137` es `128 + 9`: el proceso recibió la señal `SIGKILL` (9) porque superó su límite de memoria.

> **Ojo:** "un contenedor consume menos" es cierto porque no carga un sistema operativo entero, **no** porque tenga un límite. Sin `--memory` ni `--cpus`, no tiene ningún límite.

### 3. Aislamiento y seguridad

Aquí está **el error más importante** del apunte de curso. Dice que los contenedores "no dependen del hardware ni del sistema operativo anfitrión". Es al revés:

* **Un contenedor depende DIRECTAMENTE del kernel del anfitrión.** Es un proceso del anfitrión. Si el kernel tiene una vulnerabilidad, todos los contenedores la comparten.
* **Una VM es la que se independiza más**: tiene su propio kernel, y para llegar al anfitrión un atacante tiene que romper el hipervisor, que tiene una superficie de ataque mucho más pequeña.

```text
Para escapar de...        Hay que romper...
────────────────────      ───────────────────────────────────────────
un contenedor             el aislamiento del kernel compartido
                          (namespaces, cgroups, capabilities)
una VM                    el hipervisor (mucho más chico y difícil)
```

Y hay configuraciones que **regalan** el escape, sin necesidad de ninguna vulnerabilidad:

```bash
# Monta el disco entero del anfitrión dentro del contenedor
docker run --rm -v /:/host alpine ls /host

# Desactiva casi todo el aislamiento
docker run --rm --privileged alpine

# Da control total del daemon de Docker (= root en el anfitrión)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock docker:cli docker ps
```

> **Ojo:** por defecto, el usuario `root` dentro del contenedor es el mismo `root` del anfitrión (con permisos recortados, pero el mismo usuario). Por eso las imágenes bien hechas ejecutan la app con un usuario sin privilegios. Lo verás en la lección de `Dockerfile`.

Conclusión: para ejecutar código **en el que no confías** (malware, código de clientes, laboratorios de ataque), la VM es la opción correcta. Lo que el apunte dice de las "simulaciones seguras" está bien. Lo que está mal es atribuirle a Docker un aislamiento que no tiene.

### 4. Estado: mascotas vs. ganado

Las dos tecnologías piensan el estado de forma opuesta:

| | VM | Contenedor |
| --- | --- | --- |
| Filosofía | **Mascota** (*pet*): se configura, se cuida, se respalda | **Ganado** (*cattle*): se reemplaza en lugar de repararse |
| Cómo se guarda | Snapshot del estado completo | La imagen (receta reproducible) + volúmenes (datos) |
| Cómo se recupera | Restauras el snapshot | Creas un contenedor nuevo desde la imagen |
| Qué se respalda | La VM entera | Solo los volúmenes con datos; la imagen se reconstruye desde el `Dockerfile` |

El apunte tiene razón en que una VM es fácil de respaldar y clonar. Pero en contenedores **no necesitas** respaldar el contenedor: si se rompe, lo borras y creas otro. Lo que sí respaldas son los datos (volúmenes).

### 5. No son rivales: se usan juntos

La pregunta de entrevista "¿VM o contenedor?" tiene trampa, porque en producción la respuesta casi siempre es **las dos**:

```text
┌──────────────────── Servidor físico ────────────────────┐
│  Hipervisor (tipo 1)                                    │
│  ┌──────── VM ─────────┐   ┌──────── VM ─────────┐      │
│  │ Linux + Docker      │   │ Linux + Docker      │      │
│  │ [api] [api] [web]   │   │ [api] [worker]      │      │
│  └─────────────────────┘   └─────────────────────┘      │
└─────────────────────────────────────────────────────────┘
```

* En la nube, cuando "despliegas contenedores", corren sobre VMs (los nodos de un clúster de Kubernetes son VMs).
* La VM aísla entre clientes o equipos; los contenedores empaquetan y densifican dentro de cada VM.
* Docker Desktop en tu Windows es exactamente eso: una VM Linux (WSL2) con contenedores dentro.
* Existen híbridos: **microVMs** como Firecracker (lo usan AWS Lambda y Fargate) arrancan en milisegundos como un contenedor, pero con el aislamiento de una VM.

-----

## Ejemplo completo

Las dos decisiones del problema, bien tomadas.

**Decisión A: entorno local con contenedores y límites.**

```bash
# -d: en segundo plano | -p 8080:80: puerto 8080 del anfitrión → 80 del contenedor
docker run -d --name db    --memory 512m -e POSTGRES_PASSWORD=dev postgres:16-alpine
docker run -d --name cache --memory 128m redis:7-alpine
docker run -d --name web   --memory 64m  -p 8080:80 nginx:alpine

docker stats --no-stream
```

Resultado (los números varían en cada máquina):

```text
CONTAINER ID   NAME    CPU %   MEM USAGE / LIMIT     MEM %
a1b2c3d4e5f6   db      0.02%   38.5MiB / 512MiB      7.52%
b2c3d4e5f6a7   cache   0.15%   4.1MiB / 128MiB       3.20%
c3d4e5f6a7b8   web     0.00%   3.2MiB / 64MiB        5.00%
```

Al terminar:

```bash
docker rm -f db cache web
```

**Decisión B: laboratorio de seguridad en una VM.** Una VM en VirtualBox o Hyper-V con:

* red en modo "solo anfitrión" o interna (sin acceso a tu red real);
* sin carpetas compartidas con tu sistema;
* un snapshot "limpio" tomado antes de cada análisis, para volver a él al terminar.

Qué observar:

* Los tres servicios juntos usan **unos 45 MB** de RAM, contra los 12 GB reservados de las tres VMs. Esa es la ventaja real del contenedor: no hay tres sistemas operativos invitados.
* Cada contenedor tiene un límite. Si `cache` tiene una fuga, el kernel lo mata a él solo y no tumba la laptop.
* En B el snapshot es la herramienta clave: es la "mascota" que quieres poder restaurar exactamente.

-----

## Errores comunes

**1. Creer que el contenedor es independiente del sistema operativo anfitrión.**
Qué pasa: se ejecuta código no confiable en contenedores pensando que está tan aislado como en una VM.
Por qué: el contenedor es un proceso del anfitrión y comparte su kernel.
Arreglo: código no confiable → VM o microVM. Contenedores → código propio o de confianza.

**2. Usar `--privileged` o montar `docker.sock` "porque si no, no funciona".**
Qué pasa: el contenedor puede controlar el anfitrión entero.
Por qué: esas opciones desactivan el aislamiento o entregan el control del daemon, que corre como root.
Arreglo: dar solo el permiso puntual que hace falta (`--cap-add`, `--device`) y nunca en código que no controlas.

**3. No poner límites de recursos.**
Qué pasa: un contenedor con una fuga se come la RAM y el anfitrión empieza a matar procesos al azar.
Por qué: por defecto un contenedor no tiene límites.
Arreglo: `--memory` y `--cpus` (o su equivalente en Compose y Kubernetes).

**4. Tratar un contenedor como una VM a la que se le sacan "snapshots".**
Qué pasa: alguien configura cosas a mano dentro del contenedor y lo guarda con `docker commit`. Nadie sabe cómo se armó esa imagen.
Por qué: se aplica la mentalidad de mascota a algo pensado como ganado.
Arreglo: todo cambio va al `Dockerfile`, para que la imagen sea reproducible.

**5. Creer en la "uniformidad absoluta".**
Qué pasa: dos desarrolladores usan `node:latest` y una semana después tienen versiones distintas.
Por qué: `latest` es una etiqueta que se mueve. La uniformidad depende de fijar versiones, del kernel y de la arquitectura de CPU (lección 1).
Arreglo: fijar versiones concretas (`node:20.18-alpine`) o incluso el *digest* de la imagen.

-----

## Evolución y versiones (opcional)

* **1999:** VMware Workstation populariza la virtualización en equipos x86.
* **2007:** KVM entra al kernel de Linux; es la base de buena parte de la nube actual.
* **2013:** Docker populariza los contenedores.
* **2017-2018:** aparecen Kata Containers y Firecracker: aislamiento de VM con la experiencia de un contenedor.

-----

## Cuándo sí y cuándo no

**Elige una VM cuando:**

* ejecutas código en el que no confías o necesitas aislamiento fuerte entre clientes;
* necesitas otro sistema operativo u otro kernel (Windows sobre Linux, un kernel personalizado);
* quieres experimentar con el sistema operativo en sí (instalarlo, romperlo, configurarlo);
* necesitas snapshots del estado completo para volver atrás.

**Elige contenedores cuando:**

* empaquetas y despliegas tus propias aplicaciones;
* quieres el mismo entorno en desarrollo, CI y producción;
* necesitas muchos servicios en una sola máquina o escalar rápido;
* quieres levantar y destruir entornos en segundos.

**Combínalos cuando** estés en producción o en la nube: VMs para aislar, contenedores dentro para empaquetar.

-----

## Resumen en 5 líneas

1. La VM virtualiza hardware y trae su propio kernel; el contenedor virtualiza el sistema operativo y comparte el kernel del anfitrión.
2. El contenedor **sí** depende del sistema operativo anfitrión. Por eso aísla menos que una VM.
3. La VM reserva recursos; el contenedor consume a demanda y **no tiene límites** si no se los pones.
4. La VM es una mascota (snapshots); el contenedor es ganado (imagen reproducible + volúmenes).
5. No compiten: en producción los contenedores corren dentro de VMs.

-----

## Para profundizar (opcional)

<details>
<summary>¿Se puede hacer que root en el contenedor no sea root en el anfitrión?</summary>

Sí, de dos formas. **User namespaces** (`userns-remap` en el daemon) mapean el root del contenedor a un usuario sin privilegios del anfitrión. **Docker rootless** ejecuta el daemon entero sin root. Podman funciona sin root por defecto. Las dos opciones reducen mucho el daño de un escape, aunque tienen limitaciones con algunos volúmenes y puertos bajos.

</details>

<details>
<summary>¿Qué es gVisor y en qué se diferencia de Firecracker?</summary>

Los dos buscan más aislamiento que un contenedor normal. **gVisor** (Google) pone un "kernel" escrito en Go entre el contenedor y el kernel real, que intercepta las llamadas al sistema. **Firecracker** (AWS) es una microVM real con su propio kernel mínimo, que arranca en unos 125 ms. gVisor es más liviano; Firecracker aísla más.

</details>

-----

## En entrevista (opcional)

### Respuesta corta (junior)

Una VM virtualiza el hardware y tiene su propio sistema operativo completo; un contenedor es un proceso aislado que comparte el kernel del anfitrión. El contenedor es más liviano y arranca más rápido; la VM aísla más.

### Respuesta ampliada (semi-senior)

La decisión es aislamiento contra eficiencia. Para código no confiable, multi-tenencia estricta u otro kernel, uso VMs o microVMs. Para empaquetar y desplegar mis aplicaciones, contenedores, siempre con límites de recursos, sin `--privileged` y con un usuario sin privilegios. En producción no es "uno u otro": los contenedores corren sobre VMs, y la VM pone la frontera de seguridad mientras el contenedor aporta reproducibilidad y densidad.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué un contenedor arranca en segundos y una VM en minutos?**
Porque la VM tiene que iniciar un sistema operativo completo (kernel, servicios). El contenedor solo lanza un proceso sobre un kernel que ya está corriendo.

**2. ¿Puedo correr un contenedor Windows en un anfitrión Linux?**
No directamente: un contenedor Windows necesita un kernel Windows. Para eso hace falta una VM Windows.

**3. Si los contenedores aíslan menos, ¿por qué la nube los usa para todo?**
Porque la frontera de seguridad entre clientes la pone la VM (o una microVM como Firecracker). Los contenedores de un mismo cliente comparten VM.

-----

## Práctica

**Ejercicio 1.** Para cada caso, elige VM, contenedor o ambos, y justifica:

1. Una plataforma donde los alumnos suben código para que se ejecute y se corrija automáticamente.
2. El entorno local de un proyecto con una API en .NET y una base SQL Server.
3. Probar cómo se instala y configura Ubuntu Server desde cero.

<details>
<summary>Solución</summary>

1. **VM o microVM** (o contenedores endurecidos con gVisor). Es código que no controlas: un contenedor normal compartiría kernel con el resto del sistema y con el código de otros alumnos.
2. **Contenedores.** Es tu propio código, necesitas el mismo entorno para todo el equipo y levantarlo rápido. Con límites de memoria, porque SQL Server consume mucha si no se lo frena.
3. **VM.** Lo que quieres aprender es el sistema operativo en sí. Un contenedor no instala un sistema operativo: usa el kernel del anfitrión y no tiene proceso de arranque.

</details>

**Ejercicio 2.** ¿Qué tiene de malo esto? Es el README de un laboratorio de pentesting:

```bash
# Ejecuta el entorno de ataque aislado
docker run -it --privileged --network host -v /:/host kali-lab
```

<details>
<summary>Solución</summary>

Del "aislado" queda muy poco:

* `--privileged` desactiva casi todas las protecciones del contenedor y da acceso a los dispositivos del anfitrión.
* `--network host` comparte la red del anfitrión: el contenedor ve y puede atacar tu red real.
* `-v /:/host` monta el disco entero del anfitrión con permisos de root.

Para un laboratorio de ataque lo correcto es una **VM** con red interna, sin carpetas compartidas y con un snapshot limpio. Si de verdad hace falta un contenedor, va dentro de esa VM y sin esas tres opciones.

</details>

-----

## Siguiente lección

Próximamente: imágenes, capas y `Dockerfile`.

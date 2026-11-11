# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**cgroups (*control groups*).** Función del kernel de Linux que limita cuánta CPU, memoria o disco puede usar un proceso.

**containerd.** Runtime de contenedores que gestiona su ciclo de vida. Docker lo usa por dentro y Kubernetes lo usa directamente.

**Contenedor (*container*).** Instancia en ejecución de una imagen. Es un proceso aislado que comparte el kernel del anfitrión.

**Docker CLI.** El comando `docker` que escribes en la terminal. Solo le envía órdenes al daemon.

**Docker daemon (`dockerd`).** Servicio en segundo plano que crea y gestiona imágenes, contenedores, redes y volúmenes.

**Docker Desktop.** Aplicación para Windows y macOS que instala Docker y una VM Linux liviana donde corren los contenedores.

**Docker Hub.** El registry público por defecto de Docker.

**Escape de contenedor (*container escape*).** Cuando un proceso sale del aislamiento del contenedor y obtiene acceso al anfitrión.

**Hipervisor (*hypervisor*).** Software que crea y ejecuta máquinas virtuales y les reparte el hardware. El tipo 1 corre sobre el hardware; el tipo 2, dentro de un sistema operativo.

**Imagen (*image*).** Plantilla de solo lectura con el sistema de archivos y la configuración necesarios para ejecutar una aplicación.

**Kernel.** El núcleo del sistema operativo. Gestiona CPU, memoria, disco y red para todos los procesos.

**Kubernetes.** Orquestador que gestiona muchos contenedores en muchas máquinas: reparte la carga, reinicia los que fallan y escala.

**Mascotas vs. ganado (*pets vs. cattle*).** Dos formas de tratar los servidores: las mascotas se cuidan y se reparan; el ganado se reemplaza. Las VMs suelen tratarse como mascotas; los contenedores, como ganado.

**Máquina virtual (*VM*).** Computadora emulada con su propio sistema operativo completo, incluido su propio kernel.

**Microservicios.** Estilo de arquitectura que divide una aplicación en servicios pequeños que se despliegan por separado.

**MicroVM.** Máquina virtual mínima que arranca en milisegundos, como Firecracker. Combina el aislamiento de una VM con la agilidad de un contenedor.

**Namespaces.** Función del kernel de Linux que hace que un proceso vea solo "su" parte del sistema: sus procesos, su red, sus archivos.

**OCI (*Open Container Initiative*).** Estándar abierto para el formato de imágenes y los runtimes de contenedores.

**OOM (*Out Of Memory*).** Cuando un proceso supera la memoria permitida y el kernel lo mata. En Docker se ve como código de salida `137`.

**Orquestador.** Sistema que gestiona contenedores en varias máquinas a la vez. Kubernetes es el más conocido.

**Pull.** Descargar una imagen desde un registry.

**Push.** Subir una imagen a un registry.

**Registry.** Servidor donde se guardan y descargan imágenes.

**Sistema operativo invitado (*guest OS*).** El sistema operativo que corre dentro de una VM.

**Snapshot.** Copia del estado completo de una VM (disco, memoria, configuración) a la que se puede volver.

**Sobreasignación (*overcommit*).** Asignar a las VMs más recursos de los que el equipo tiene físicamente, confiando en que no todas los usen a la vez.

**WSL2 (*Windows Subsystem for Linux 2*).** Función de Windows que ejecuta un kernel Linux real en una VM liviana. Docker Desktop la usa para correr contenedores Linux.

# Proyecto de Sistemas Operativos
## Proyecto 3: Tablas de páginas en acción (xv6)

### Integrantes

- Juan José Acevedo
- Miguel Marín
- Sebastian Henao

### Curso

Sistemas Operativos

### Semestre

2026-2

---

# Descripción

Extendimos el kernel de xv6-riscv (commit `06aad25`) con tres mecanismos que trabajan directamente sobre la tabla de páginas de cada proceso.

**Parte A.** Cada proceso tiene ahora una página en `USYSCALL` (la página justo debajo de `TRAPFRAME`) donde el kernel escribe su pid al crearlo. Está mapeada con `PTE_R | PTE_U`, así que el proceso la puede leer pero no escribir. `ugetpid()` lee el pid de ahí sin entrar al kernel. La página se libera cuando el proceso muere y el heap no puede crecer sobre ella.

**Parte B.** La syscall `vmprint()` imprime la tabla de páginas del proceso que la llama. Recorre los tres niveles de Sv39 de forma recursiva, igual que `freewalk()`, y muestra solo las PTEs válidas. Las páginas reservadas con `sbrklazy()` que el proceso no ha tocado no aparecen, porque todavía no tienen PTE.

**Parte C.** La syscall `pgaccess()` dice cuáles de un rango de páginas fueron leídas o escritas desde la última consulta. Para eso lee el bit `PTE_A`, que el hardware enciende solo en cada acceso, y lo limpia después de leerlo. Nunca asigna memoria: una página perezosa sin tocar se reporta como 0.

---

# Archivos modificados

**`kernel/memlayout.h`**
Se agregó `USYSCALL` como `TRAPFRAME - PGSIZE` y `struct usyscall { int pid; }`. La estructura va dentro de `#ifndef __ASSEMBLER__` porque `trampoline.S` también incluye este archivo y el ensamblador no entiende structs. También se actualizó el comentario del mapa de memoria del usuario.

**`kernel/proc.h`**
Nuevo campo `struct usyscall *usyscall` en `struct proc`. Guarda la dirección física de la página, que es distinta en cada proceso. Va al lado de `trapframe` porque las dos páginas siguen el mismo patrón.

**`kernel/proc.c`**
- `allocproc()`: reserva la página con `kalloc()`, la pone en cero completa (el proceso puede leer toda la página, no solo el pid) y escribe el pid.
- `proc_pagetable()`: mapea la página en `USYSCALL` con `PTE_R | PTE_U`. Si el mapeo falla, deshace el trampolín y el trapframe antes de liberar la tabla.
- `proc_freepagetable()`: desmapea `USYSCALL` sin liberar la memoria física.
- `freeproc()`: libera la página física.
- `growproc()`: el límite del heap pasa de `TRAPFRAME` a `USYSCALL`.

**`kernel/vm.c`**
Se agregaron `vmprintwalk()` (recursiva, `static`) y `vmprint()`. La dirección virtual de cada PTE se reconstruye con los índices del camino: `va | (i << PXSHIFT(nivel))`.

**`kernel/sysproc.c`**
- `sys_sbrk()`: el límite de la rama perezosa pasa de `TRAPFRAME` a `USYSCALL`.
- `sys_vmprint()`: llama a `vmprint()` con la tabla del proceso actual y devuelve 0.
- `sys_pgaccess()`: valida los argumentos, recorre las páginas con `walk(..., 0)`, arma la máscara, limpia `PTE_A` y copia el resultado con `copyout()`.

**`kernel/riscv.h`**
Se agregó `PTE_A` (bit 6).

**`kernel/syscall.h`, `kernel/syscall.c`**
Registro de `SYS_vmprint` (23) y `SYS_pgaccess` (24) en los números y en la tabla `syscalls[]`.

**`kernel/defs.h`**
Prototipo de `vmprint()`.

**`user/ulib.c`**
`ugetpid()`, que lee `((struct usyscall *)USYSCALL)->pid`. Incluye `kernel/memlayout.h` después de `kernel/riscv.h`, porque `memlayout.h` usa `MAXVA` y `PGSIZE`.

**`user/user.h`, `user/usys.pl`**
Declaraciones de `ugetpid()`, `vmprint()` y `pgaccess()` con las firmas que pide la guía, y los stubs de las dos syscalls nuevas. `ugetpid()` no está en `usys.pl` porque no es una syscall.

---

# Integración con xv6

1. Clonar xv6-riscv y hacer `git switch -c proyecto3 06aad25`.
2. Copiar los archivos entregados en sus rutas.
3. Agregar `$U/_pgtbltest` a UPROGS en el Makefile.
4. Compilar con `make qemu`.
5. Ejecutar `pgtbltest` y `usertests -q`.

---

# Respuestas a las preguntas

## 1. Interpretación de vmprint()

Esta es la tabla de un proceso recién creado, con un programa pequeño que llama a `vmprint()` al empezar:

```
page table 0x0000000087f22000
 ..0x0000000000000000: pte 0x0000000021fc7801 pa 0x0000000087f1e000
 .. ..0x0000000000000000: pte 0x0000000021fc7401 pa 0x0000000087f1d000
 .. .. ..0x0000000000000000: pte 0x0000000021fc7c5b pa 0x0000000087f1f000
 .. .. ..0x0000000000001000: pte 0x0000000021fc7017 pa 0x0000000087f1c000
 .. .. ..0x0000000000002000: pte 0x0000000021fc6c07 pa 0x0000000087f1b000
 .. .. ..0x0000000000003000: pte 0x0000000021fc68d7 pa 0x0000000087f1a000
 ..0x0000003fc0000000: pte 0x0000000021fc8401 pa 0x0000000087f21000
 .. ..0x0000003fffe00000: pte 0x0000000021fc8001 pa 0x0000000087f20000
 .. .. ..0x0000003fffffd000: pte 0x0000000021fd4813 pa 0x0000000087f52000
 .. .. ..0x0000003fffffe000: pte 0x0000000021fd00c7 pa 0x0000000087f40000
 .. .. ..0x0000003ffffff000: pte 0x000000002000184b pa 0x0000000080006000
```

Los permisos están en los 8 bits bajos de cada PTE: V (0), R (1), W (2), X (3), U (4), A (6) y D (7). Las entradas de nivel 1 y 2 terminan en `01`. Solo tienen V, y eso significa que apuntan a otra tabla y no a una página de datos.

| VA | Bits | Flags | Contenido |
|---|---|---|---|
| `0x0000` | `0x5b` | V R X U A | Texto. Se ejecuta y se lee, pero no se escribe, para que un error no pueda modificar el código. |
| `0x1000` | `0x17` | V R W U | Datos globales. Lectura y escritura, sin ejecución. |
| `0x2000` | `0x07` | V R W, sin U | Página de guarda, debajo de la pila. Ver pregunta 2. |
| `0x3000` | `0xd7` | V R W U A D | Pila de usuario. D está encendido porque ya se escribió en ella. |
| `0x3fffffd000` | `0x13` | V R U | `USYSCALL`. El usuario la lee, no la escribe. |
| `0x3fffffe000` | `0xc7` | V R W A D, sin U | Trapframe. Ahí `trampoline.S` guarda los registros del usuario, y solo el kernel puede tocarlo. |
| `0x3ffffff000` | `0x4b` | V R X A, sin U | Trampolín. Es código que corre en modo supervisor al entrar y salir del kernel, y está en la misma dirección virtual en la tabla del usuario y en la del kernel. |

El heap todavía no aparece porque este proceso no ha llamado a `sbrk()`. Cuando lo hace, aparece una página nueva en `0x4000` con `V R W U`. Si en cambio usa `sbrklazy()`, la página no aparece hasta que el proceso la toca por primera vez.

## 2. Página de guarda

La pila crece hacia direcciones bajas, y justo debajo está la página de guarda. Esa página tiene PTE válida pero sin `PTE_U`. El hardware no deja que el código en modo usuario acceda a una página sin `PTE_U`, aunque la PTE sea válida.

Si un programa desborda la pila, la siguiente escritura cae en la página de guarda. El procesador genera una falla de página, el kernel no la puede resolver y mata al proceso. Sin esa página, el desborde escribiría encima de los datos globales sin que nadie se diera cuenta, y el programa seguiría corriendo con memoria corrupta. Es mejor que el proceso muera de forma visible.

`usertests` lo comprueba en la prueba `stacktest`, donde aparece un `usertrap()` y la prueba sale OK.

## 3. Otras llamadas que se pueden acelerar

Sirve para llamadas que solo **leen** información que el kernel ya conoce y puede dejar escrita de antemano, sin efectos secundarios.

**Se podrían acelerar:**
- `getpid()`: ya está hecho. El pid no cambia en toda la vida del proceso, ni siquiera con `exec()`.
- `uptime()`: el kernel podría mantener `ticks` en una página de solo lectura compartida por todos los procesos y actualizarla en cada interrupción del reloj. Es la misma idea que el vDSO de Linux para `gettimeofday()`.
- El pid del padre, si se agregara una llamada para obtenerlo. Habría que actualizar la página cuando el proceso queda huérfano y lo adopta `init`.

**No se pueden acelerar:**
- Las que cambian estado: `fork()`, `exec()`, `exit()`, `kill()`, `sbrk()`, `open()`, `close()`, `write()`, `pipe()`. Necesitan que el kernel actúe, y leer memoria no ejecuta nada.
- Las que dependen de argumentos que el kernel tiene que validar o de datos que cambian con cada llamada, como `read()` o `fstat()`. No hay un valor fijo que se pueda dejar escrito antes.
- `wait()` y `pause()`, porque tienen que bloquear al proceso, y eso solo lo puede hacer el planificador.

## 4. Riesgo de USYSCALL con PTE_W

El proceso podría escribir cualquier valor en el campo `pid`, y desde ese momento `ugetpid()` devolvería un número falso.

El kernel no se ve afectado directamente, porque usa `p->pid` de `struct proc` y nunca vuelve a leer la página. El riesgo está en que la página deja de ser una fuente confiable. Un programa que use `ugetpid()` para decidir a quién mandar un `kill()`, o con qué identidad registrarse, podría equivocarse o ser engañado. Además, si en el futuro se agregara a la página información que el kernel sí lee, por ejemplo para acelerar otras llamadas, un proceso podría usarla para meterle datos falsos al kernel.

La regla general es que la información que el kernel publica para el usuario tiene que ser de solo lectura. Por eso el script de pruebas exige que la PTE de `USYSCALL` tenga exactamente V, R y U.

## 5. pgaccess() y páginas perezosas

`sbrklazy()` solo aumenta `p->sz` y no crea ninguna PTE. La página física y su PTE aparecen la primera vez que el proceso la toca: el hardware genera una falla, `vmfault()` reserva la página y la mapea. El bit `PTE_A` vive dentro de la PTE, así que si no existe PTE tampoco hay bit, y que nadie haya creado la página significa que nadie la usó. Reportar 0 es la respuesta correcta.

`pgaccess()` no debe asignarle memoria por tres razones:
- Rompería la asignación perezosa: el proceso pagaría memoria por páginas que tal vez nunca use, solo porque alguien preguntó por ellas.
- La respuesta sería engañosa. La página recién creada tendría `PTE_A` en 0, pero quedaría aparentando que es memoria en uso, y aparecería en `vmprint()`.
- Una consulta no debería cambiar lo que consulta.

Por eso usamos `walk()` con `alloc = 0`, que nunca crea tablas ni páginas.

---

# Resultados de las pruebas

Las dos suites se ejecutaron con `ejecutar_pruebas.py`. La salida completa está en `pruebas/`.

- **pgtbltest:** PASÓ, 11 de 11 pruebas. La revisión automática del formato de `vmprint()` dio OK.
- **usertests -q:** PASÓ (`ALL TESTS PASSED`), incluidas las pruebas de asignación perezosa y la verificación de páginas libres al final, así que no hay fugas de memoria.

Los mensajes `usertrap(): unexpected scause ...` que aparecen en las dos salidas son esperados. Vienen de pruebas que acceden a memoria prohibida a propósito para comprobar que el kernel mata al proceso. Por ejemplo, `usyscall_solo_lectura` intenta escribir en `0x3fffffd000`.

---

# Decisiones de diseño

**La página `USYSCALL` se libera en `freeproc()` y no en `proc_freepagetable()`.** `exec()` crea una tabla de páginas nueva y destruye la vieja con `proc_freepagetable()` mientras el proceso sigue vivo. Si esa función liberara la página física, la tabla nueva apuntaría a memoria ya liberada. Por eso `proc_freepagetable()` solo desmapea y `freeproc()` libera. Es el mismo patrón que xv6 ya usa con el trapframe. La prueba `ugetpid_exec` cubre este caso.

**El límite del heap se cambió en dos lugares.** El crecimiento normal pasa por `growproc()`, pero `sbrklazy()` no: tiene su propia comparación en `sys_sbrk()`. Si se cambiara solo uno de los dos, `sbrklazy()` podría subir `p->sz` hasta cubrir `USYSCALL`, y el kernel la trataría como parte del heap.

**`vmprint()` del kernel está separada de `sys_vmprint()`.** La función interna recibe cualquier tabla de páginas, y la syscall solo le pasa la del proceso actual. Así sirve para depurar otras tablas, por ejemplo en el Proyecto Final.

**`vmprint()` distingue hojas de tablas intermedias igual que `freewalk()`.** Si una PTE es válida y no tiene R, W ni X, apunta a otra tabla.

**`pgaccess()` usa `walk()` con `alloc = 0`.** Así no puede crear tablas ni páginas, ni siquiera por accidente.

**No fue necesario vaciar la TLB después de limpiar `PTE_A`.** Si la TLB conservara una traducción con el bit viejo, el siguiente acceso podría no volver a encenderlo en la tabla. Revisamos que `trampoline.S` ejecuta `sfence.vma` al volver a modo usuario, y eso vacía la TLB. La prueba que detecta accesos nuevos después de limpiar lo confirma.

**El bit `PTE_A` lo enciende el hardware.** En `start.c`, xv6 activa `MENVCFG_ADUE`, y con eso el procesador actualiza los bits A y D solo, sin generar una falla para que lo haga el sistema operativo. `pgaccess()` se limita a leerlo y limpiarlo.

---

# Limitaciones conocidas

- Si el puntero `mask` que recibe `pgaccess()` es inválido, la llamada devuelve -1, pero para ese momento los bits `PTE_A` del rango ya se limpiaron, así que esos accesos se pierden. Lo aceptamos porque un puntero inválido es un error del programa y no un caso normal.
- `pgaccess()` acepta como máximo 64 páginas por llamada, porque la máscara es de 64 bits. Lo exige el enunciado.
- La página compartida solo tiene el pid. Acelerar otras llamadas está fuera del alcance (ver pregunta 3).
- Al compilar aparecen dos advertencias del enlazador, `LOAD segment with RWX permissions`, en `kernel/kernel` y `user/_forktest`. No vienen de nuestro código: también salen al compilar xv6 sin modificar. El compilador no da ninguna advertencia, y se compila con `-Werror`.
- La prueba `heap_no_invade` tarda unos segundos. Al terminar el proceso, `uvmunmap()` recorre página por página todo el rango del heap perezoso, que llega casi hasta `USYSCALL`. Es el comportamiento original de xv6 y no lo modificamos.

---

# Declaración de uso de IA

Usamos Claude (Anthropic) como apoyo durante el proyecto. Lo usamos para:

- Entender el enunciado y los conceptos: Sv39, PTEs, el ciclo de vida de un proceso en xv6 y el bit `PTE_A`.
- Revisar el código fuente de xv6 en el commit `06aad25` y detectar casos delicados, como la liberación de la página en `exec()`, el doble límite del heap y la TLB.
- Proponer la implementación de cada parte, que nosotros integramos, compilamos, probamos en QEMU y revisamos paso a paso.
- Redactar este README a partir de nuestros resultados.

Los integrantes revisamos y entendemos todo el código entregado y asumimos plena responsabilidad académica sobre el contenido.

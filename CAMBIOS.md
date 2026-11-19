# Cambios — Jornadas del Sur (Android)

---

## 41 · Programa sin señal

### Problema

La versión original tenía dos defectos que impedían un funcionamiento offline correcto:

1. **La falla ocultaba la lista.** Cuando la descarga fallaba, el composable `aviso` reemplazaba la grilla completa: el usuario perdía de vista las charlas que ya tenía guardadas localmente, sus fichas de detalle y su agenda de favoritas.
2. **No se distinguía "nunca descargué" de "programa vacío".** Si la organización publicaba un programa vacío (v4, lista `[]`), la tabla `charlas` quedaba sin registros, igual que antes de la primera descarga. Al reabrir sin señal, la app no podía saber si debía mostrar "Sin charlas programadas" (edición válida) o "No hay copia guardada" (nunca se descargó).
3. **Solo se atrapaba `IOException`.** Cualquier otra excepción (error HTTP, respuesta malformada, error de parseo) no era manejada y propagaba un crash.

### Cambios realizados

#### Nueva tabla `edicion` (Room v2)

Se agregó la entidad `Edicion` con una sola fila (PK fija `id = 0`) y un campo `momento` (timestamp). Cada vez que `aplicarInstantanea` termina con éxito, se ejecuta `registrarEdicion`, insertando o reemplazando el registro. La consulta `hayEdicion()` devuelve un `Flow<Boolean>` que emite `true` si existe al menos un registro en la tabla.

Esto permite tres estados distinguibles:

| `hayEdicion` | `fallo` | Significado | UI |
|---|---|---|---|
| `false` | `true` | Nunca hubo descarga exitosa y la actual falló | Mensaje de error + botón Reintentar |
| `false` | `false` | Nunca hubo descarga y está en curso o no se inició | Mensaje "Todavía no se descargó" (+ Descargar si no está cargando) |
| `true` | `true` | Hay datos guardados pero la descarga actual falló | Lista/ficha/favoritas visibles + aviso "Sin conexión: se están usando los datos guardados" + Reintentar |
| `true` | `false` | Hay datos guardados y la descarga fue exitosa (o no se intentó) | Si la lista está vacía → "Sin charlas programadas"; si tiene charlas → grilla normal |

#### Lógica de UI (Catalogo.kt y Portada.kt)

En `Catalogo.kt`, la decisión se estructura así:

- Si `hayEdicion == null` o `fichas == null` → todavía se está leyendo la base local, no se muestra nada (evita parpadeos).
- Si `hayEdicion == false` → nunca hubo descarga exitosa:
  - Si `fallo` es `true` → texto informativo + botón `Reintentar`.
  - Si no está cargando → texto "Todavía no se descargó" + botón `Descargar`.
- Si `hayEdicion == true` → hay datos guardados:
  - Si `fallo` es `true` → botón `Reintentar` (la lista sigue visible debajo).
  - Si `lista.isEmpty()` → `"Sin charlas programadas"` (caso v4).
  - Si hay charlas → se filtran por día/tema y se muestran en la grilla.

En `Portada.kt`, el aviso `"Sin conexión: se están usando los datos guardados."` se muestra **por encima** del contenido (lista, ficha o Mi agenda), sin ocultarlo. Esto permite que el usuario siga navegando con los datos locales.

#### Manejo amplio de excepciones (ProgramaModel.kt)

El bloque `try/catch` ahora atrapa cualquier `Exception` **excepto** `CancellationException` (que se relanza para respetar la cancelación estructurada de coroutines). Esto cubre errores de red (`IOException`), errores HTTP, errores de parseo JSON (`JsonSyntaxException`) y cualquier otro fallo inesperado. Si la descarga falla, la base de datos no se toca: `sincronizar()` primero obtiene las fichas remotas y solo después llama a `aplicarInstantanea`.

#### Migración 1→2 (`MIGRACION_1_2`)

Para no perder los datos de los usuarios que actualizan desde la versión 1 de la base:

1. Se crea la tabla `edicion` con `CREATE TABLE IF NOT EXISTS`.
2. Se ejecuta un `INSERT OR IGNORE INTO edicion` condicionado a que ya existan charlas: si el usuario tenía charlas descargadas, eso se reconoce como una edición válida. Así no se le muestra erróneamente el mensaje de "nunca se descargó".

---

## 42 · Agenda personal

### Problema

La versión original usaba `@Insert(onConflict = REPLACE)` para guardar cada descarga. Esto causaba dos defectos:

1. **Se borraban todas las marcas de favoritos.** Las charlas del servidor venían con `favorita = false`, y el `REPLACE` sobreescribía la fila completa incluyendo la columna `favorita`. Cada sincronización desmarcaba todo.
2. **Nunca se eliminaban charlas retiradas.** Si la organización sacaba la charla 11 del programa en v3, esta seguía en la base local porque solo se insertaban/reemplazaban, nunca se borraban. La 11 aparecía en el catálogo y en Mi agenda indefinidamente.

### Cambios realizados

#### Nuevo método `aplicarInstantanea` (CarteleraDao)

Se reemplazó la inserción simple por una transacción atómica (`@Transaction`) de cuatro pasos:

**Paso 1 — Retirar charlas ausentes:**
```
DELETE FROM charlas WHERE id NOT IN (:vigentes)
```
Se borran las charlas cuyos IDs ya no vienen en la instantánea del servidor. Esto elimina la charla de la tabla `charlas` completa, incluyendo su marca. Consecuencia: si una charla se retira y más adelante vuelve a aparecer, se insertará como nueva con `favorita = false` (regresa sin marcar).

Caso especial: si la instantánea viene vacía (`fichas.isEmpty()`), se ejecuta `retirarTodas()` en lugar de `retirarAusentes` (que recibiría una lista vacía y el `NOT IN` no funcionaría correctamente).

**Paso 2 — Insertar charlas nuevas:**
```kotlin
@Insert(onConflict = OnConflictStrategy.IGNORE)
```
Se insertan las charlas que no existían previamente, con `favorita = false` (el default en `comoCharlaNueva()`). Las charlas que ya existían se **ignoran** (no se tocan) → su marca queda intacta.

**Paso 3 — Actualizar contenido publicado:**
```kotlin
@Update(entity = Charla::class)
abstract suspend fun actualizarContenido(fichas: List<ContenidoCharla>)
```
Se usa una **actualización parcial** de Room: la clase `ContenidoCharla` tiene todos los campos de `Charla` **excepto** `favorita`. Room genera un `UPDATE` que solo modifica las columnas presentes en `ContenidoCharla` (título, tema, día, hora, sala, ponente, resumen), dejando `favorita` con el valor que tiene en la base **en ese momento**.

Esto logra dos cosas:
- Si la charla 11 cambia de título y sala en v2, se actualiza el contenido pero la marca del usuario se conserva.
- Si el usuario cambia una marca **mientras la petición está en curso** (durante los 1.4s o 8s del `Thread.sleep` del interceptor), esa marca ya está escrita en la base cuando llega la transacción, y `actualizarContenido` no la toca.

**Paso 4 — Registrar edición:**
Se inserta/reemplaza un registro en la tabla `edicion` para marcar que hubo al menos una descarga exitosa (usado por el pedido 41).

#### Escenarios verificables

| Escenario | Comportamiento |
|---|---|
| v1, marcar charla 11 → v2 + Actualizar | La 11 conserva su marca. Muestra «Compose en producción avanzada» y «Sala Puerto» (datos actualizados de v2). |
| v2 → v3 + Actualizar | La 11 desaparece del catálogo y de Mi agenda (se borró con `retirarAusentes`). La 90 aparece sin marcar. Las marcas de las demás charlas vigentes se conservan. |
| v3 → v1 + Actualizar | La 11 vuelve a aparecer pero **sin marcar** (fue borrada en v3 e insertada como nueva en v1 con `favorita = false`). |
| Marcar charla durante petición en curso | La marca se escribe en la base inmediatamente. Cuando la respuesta llega, `actualizarContenido` no modifica `favorita` → la marca se respeta. |

---

## 43 · Volver al punto de lectura

### Problema

Los filtros no sobrevivían a los cambios de configuración ni a la muerte del proceso:

1. **El tema estaba en un `remember` del Catálogo.** Como `remember` vive en la composición, al abrir una ficha de detalle el composable del catálogo salía de la composición y el tema se perdía. Al rotar, también se perdía porque `remember` no sobrevive a recreation de la Activity.
2. **El día vivía en el ViewModel.** Sobrevivía a rotación (el ViewModel sobrevive a configuration changes) pero no a la muerte del proceso por el sistema (cuando Android mata la app en segundo plano para liberar memoria).

### Cambios realizados

#### Filtros en `SavedStateHandle` (ProgramaModel.kt)

Se movieron `dia` y `tema` al `SavedStateHandle` del ViewModel, usando `getStateFlow`:

```kotlin
val dia: StateFlow<Int> = guardado.getStateFlow(CLAVE_DIA, 0)
val tema: StateFlow<String> = guardado.getStateFlow(CLAVE_TEMA, "Todos")
fun elegirDia(valor: Int) { guardado[CLAVE_DIA] = valor }
fun elegirTema(valor: String) { guardado[CLAVE_TEMA] = valor }
```

`SavedStateHandle` persiste los valores en el `Bundle` del sistema, que sobrevive a:
- **Rotación** (configuration change): el ViewModel se reutiliza y el `SavedStateHandle` mantiene sus valores.
- **Muerte del proceso**: Android guarda el `Bundle` antes de matar la app. Al restaurar la tarea desde Recientes, el `SavedStateHandle` se reconstruye con los valores guardados.

Los valores por defecto (`0` para "Ambos días", `"Todos"` para tema) se usan cuando no hay estado previo guardado, es decir, en una **tarea nueva**.

#### Ficha abierta en `rememberSaveable` (Portada.kt)

La variable `detalle` (ID de la charla abierta) ya usaba `rememberSaveable`, que persiste el valor en el `Bundle` del sistema. Esto permite restaurar la ficha abierta tras la muerte del proceso.

#### Escenarios verificables

| Escenario | Comportamiento |
|---|---|
| Día 2 + Datos → abrir «Cachés para viajeros» → Volver → Rotar | Quedan los 2 resultados de Datos en Día 2. Los filtros sobreviven gracias al `SavedStateHandle`. |
| Misma selección + ficha abierta → Inicio → `adb shell am kill ar.jornadas` → Recientes | Vuelve la ficha de «Cachés para viajeros» (`detalle` restaurado por `rememberSaveable`). Al pulsar Volver, aparecen los filtros Día 2 + Datos con los 2 resultados (`dia` y `tema` restaurados por `SavedStateHandle`). |
| `adb shell pm clear ar.jornadas` → arranque nuevo | No hay estado guardado → `dia = 0` ("Ambos días"), `tema = "Todos"`. Arranca sin filtros. |

---

## 44 · Control de descargas

### Problema

La versión original disparaba descargas excesivas e incontroladas:

1. **Recarga en `onStart`.** Cada vez que la Activity volvía al primer plano (rotar, volver del fondo, volver de otra app) se lanzaba una descarga.
2. **`LaunchedEffect` en el Catálogo.** Cada vez que el composable del catálogo entraba en la composición (volver de una ficha, cambiar de pestaña) se lanzaba otra descarga.
3. **Dos toques = dos peticiones.** No había protección contra toques rápidos en el botón de actualización.
4. **`Servicios` se creaba por ViewModel.** Si había varias instancias de `ProgramaModel` (posible en ciertos escenarios), se creaban múltiples instancias de la base de datos Room y del contador de peticiones.

### Cambios realizados

#### `Servicios` como singleton por proceso (Servicios.kt)

`Servicios` ahora usa un patrón singleton con double-checked locking:

```kotlin
companion object {
    @Volatile private var instancia: Servicios? = null
    fun de(context: Context): Servicios =
        instancia ?: synchronized(this) {
            instancia ?: Servicios(context.applicationContext).also { instancia = it }
        }
}
```

Esto garantiza una sola instancia de Room, del cliente HTTP, del `ServicioLocal` (y su contador) y de la flag `cargaAutomaticaHecha` durante toda la vida del proceso. Al morir el proceso, el singleton se destruye y todo se reinicia.

#### Una carga automática por sesión (ProgramaModel.kt)

La flag `cargaAutomaticaHecha` vive en el singleton `Servicios`:

```kotlin
@Volatile var cargaAutomaticaHecha = false
```

`cargaAutomatica()` la revisa y la pone en `true` antes de llamar a `recargar()`:

```kotlin
fun cargaAutomatica() {
    if (servicios.cargaAutomaticaHecha) return
    servicios.cargaAutomaticaHecha = true
    recargar()
}
```

`MainActivity.onCreate` la invoca después de aplicar la configuración de la mesa:

```kotlin
model.cargaAutomatica()
```

Como `onCreate` se llama en cada rotación, pero la flag ya está en `true` tras la primera llamada, las rotaciones no disparan nuevas descargas. La flag se resetea al morir el proceso, permitiendo una nueva carga automática en la siguiente sesión.

#### Eliminación de descargas redundantes (MainActivity.kt, Catalogo.kt)

- Se **eliminó** la recarga en `onStart` de `MainActivity`. El comentario en la línea 22 lo documenta.
- Se **eliminó** el `LaunchedEffect` de recarga en `Catalogo.kt`. El comentario en la línea 18 lo documenta.

Esto asegura que rotar, entrar a una ficha, cambiar de pestaña o volver del fondo **no suma ninguna petición** al contador.

#### Protección contra toques duplicados (ProgramaModel.kt)

```kotlin
private var descarga: Job? = null

fun recargar() {
    if (descarga?.isActive == true) return
    descarga = viewModelScope.launch { ... }
}
```

Si ya hay un `Job` activo (petición en curso), `recargar()` retorna inmediatamente. Dos toques rápidos en "Actualizar" solo cuentan como un intento. Al terminar el Job (éxito o falla), `descarga.isActive` pasa a `false` y se puede volver a pedir.

El botón `Actualizar` también se deshabilita visualmente con `enabled = !cargando` para dar feedback al usuario, pero la protección real está en `recargar()`.

#### Filtros editables durante la espera (Catalogo.kt)

Los `FilterChip` de día y tema no tienen ninguna condición `enabled` ligada a `cargando`. El usuario puede cambiar filtros libremente mientras espera la respuesta del servidor. El comentario en la línea 26 lo explicita.

#### Escenarios verificables

| Escenario | Peticiones esperadas |
|---|---|
| Abrir la app (proceso nuevo) | 1 (carga automática) |
| Rotar 3 veces | 0 (flag ya activada + sin `onStart`) |
| Abrir ficha y volver | 0 (sin `LaunchedEffect`) |
| Cambiar a Mi agenda y volver a Programa | 0 (sin `LaunchedEffect`) |
| Tocar Actualizar 1 vez | 1 |
| Tocar Actualizar 2 veces rápido durante petición | 1 (el segundo toque se ignora) |
| Esperar a que termine (éxito o falla) + tocar Actualizar | 1 |

---

## Verificación manual sugerida

1. **v1**, marcar la 11 → **v2** + Actualizar: la 11 sigue marcada con «Compose en producción avanzada» / «Sala Puerto».
2. Marcar otra charla con la **demora larga** activa y la petición en curso → al llegar la respuesta la marca se mantiene.
3. **v3** + Actualizar: desaparece la 11 (catálogo y Mi agenda), aparece la 90, el resto conserva marcas. Volver a **v1**: la 11 vuelve sin marcar.
4. **v4** + Actualizar: «Sin charlas programadas». Apagar el servicio → `adb shell am kill ar.jornadas` → reabrir: sigue «Sin charlas programadas» con aviso de datos guardados.
5. `pm clear` + arranque con `servicio_disponible false`: mensaje + Reintentar.
6. Día 2 + Datos → abrir «Cachés para viajeros» → Volver → rotar: 2 resultados. Repetir con la ficha abierta + Inicio + `am kill` + Recientes: vuelve la ficha y, al volver, ambos filtros.
7. Contador de la Mesa: rotar, abrir fichas y cambiar de pestaña no suma; dos toques rápidos en Actualizar suman 1.

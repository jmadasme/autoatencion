# Bitácora del proyecto

**Naturaleza de este documento:** no se conservan actas de las decisiones originales. Esta bitácora se **reconstruye a partir del código, la documentación existente y los `VERSION.md`** de cada importación. Distingue entre hechos verificables (lo que el código hace) y decisiones inferidas (lo que el diseño sugiere). No inventa fechas ni autores.

**Alcance:** las tres generaciones —Autosale, turnero y notificador— y las decisiones transversales del repositorio de portafolio.

---

## 1. Línea de tiempo reconstruida

| Etapa | Entregable | Contexto | Tecnología | Resultado |
|---|---|---|---|---|
| 1 | Autosale | Tótems en faenas mineras y casinos, Chile | C# WinForms, .NET 4.5.2, SQL Server, serial | Sistema de venta con hardware; código parcialmente opaco |
| 2 | turnero | Oficina de atención con fila y múltiples pantallas | Python, Flask, SQLite, SSE, QR | Sistema ejecutable y documentado |
| 3 | notificador | Aviso posterior de un carrito | Python stdlib, SQLite, SMTP | Utilidad ejecutable con envío SMTP incompleto |
| 4 | autoatencion | Portafolio | Markdown, SVG | Este repositorio, que reúne y documenta las tres |

La secuencia no implica que una versión haya reemplazado a la otra en el mismo producto. Son proyectos independientes que abordan facetas distintas de la autoatención y se agrupan para mostrar evolución técnica.

---

## 2. Decisiones y su justificación

### 2.1 Autosale

#### D-01 · Windows Forms para el tótem

**Decisión:** aplicación de escritorio WinForms sobre .NET Framework 4.5.2.
**Contexto:** hardware serie/COM, impresora Epson con control COM, sistema operativo Windows en el tótem.
**Justificación:** era la vía directa para integrar periféricos y desplegar una UI táctil. No había requisito de acceso remoto.
**Costo:** instalación por equipo, dependencia de Windows, dificultad de actualización.
**Alternativa no adoptada:** una aplicación web con un agente local. En la época, la integración COM y la impresión fiscal habrían requerido un puente nativo igualmente.

#### D-02 · SQL Server local + sincronización

**Decisión:** base local SQL Server Express y copia periódica a una base central.
**Contexto:** red intermitente en faenas; la venta no puede depender de la conectividad.
**Justificación:** permite operar offline y sincronizar después. `SqlBulkCopy` es eficiente para copiar tablas completas.
**Costo:** dos bases, resolución de conflictos, `TRUNCATE` + copia completa, dependencia de Windows/SQL Server.
**Riesgo:** la sincronización por tablas completas puede ser pesada y propensa a duplicados si falla a mitad de camino. El código usa `KeepIdentity` y acciones `TruncateDestino`/`TruncaOrigen`, pero no se observa una transacción distribuida.

#### D-03 · Quartz.NET para tareas programadas

**Decisión:** usar Quartz.NET en lugar de un servicio Windows propio o el Programador de tareas.
**Justificación:** cron configurable desde la base, jobs con nombre, reintentos y logs.
**Costo:** dependencia adicional y configuración en cada tótem.
**Observación:** el cron se lee de la base, lo que permite ajustar la frecuencia sin recompilar; es una buena decisión para flotas.

#### D-04 · Protocolo serial Transbank implementado a mano

**Decisión:** clase `TransBank` con tramas STX/ETX, separador `|`, checksum XOR y parseo por offsets.
**Contexto:** integración con pin pad físico.
**Justificación:** el proveedor entrega el protocolo; no siempre hay un SDK .NET cómodo.
**Costo:** fragilidad ante cambios de firmware, offsets mágicos, difícil de testear sin hardware.
**Lección:** aislar el protocolo en una clase es correcto; lo que falta es una capa de pruebas con tramas grabadas y un parser por nombres de campo, no por posiciones.

#### D-05 · Duplicación de `TransBank`

**Hecho:** hay una clase `TransBank` en `librerias.cs` y otra casi idéntica en `Biblioteca/Class1.cs`.
**Causa probable:** evolución por copia entre proyectos o necesidad de compilar la biblioteca por separado.
**Consecuencia:** dos fuentes de verdad, riesgo de divergencia.
**Acción recomendada:** extraer a un proyecto compartido y eliminar la copia.

#### D-06 · Credenciales SMTP embebidas

**Hecho:** `funciones.enviarCorreo` contiene credenciales legacy en el código.
**Causa probable:** practica de la época, antes de los gestores de secretos.
**Riesgo:** exposición y rotación difícil.
**Acción recomendada:** rotar las credenciales, moverlas a variables de entorno o a un almacén de secretos, y eliminar el valor del historial si el repo se publica.

#### D-07 · Certificado temporal de firma en el repositorio

**Hecho:** `AutoSell_TemporaryKey.pfx` está presente.
**Riesgo:** no es una clave de producción, pero su presencia confunde y puede llevar a firmar releases con un certificado temporal.
**Acción recomendada:** excluirlo y documentar el proceso de firma.

#### D-08 · Archivos de formulario en formato opaco

**Hecho:** `Principal.cs` (246 KB), `Principal.Designer.cs` (96 KB), `IngresoTransbank.cs` (51 KB), `mensajesPantalla.*` y `Test_Int_POS.cs` no son texto C# legible; `file` los reporta como datos/OpenPGP.
**Causa probable:** guardado binario, ofuscación o corrupción durante el traslado.
**Consecuencia:** el proyecto no compila tal cual; la UI no se puede mantener desde este repositorio.
**Decisión:** conservarlos como evidencia histórica, **no** intentar descifrarlos ni publicar su contenido, y documentar la limitación.
**Lección:** conservar el proyecto original completo y verificar la integridad al archivar. Un repositorio de portafolio debe declarar lo que no es reproducible.

### 2.2 turnero

#### D-09 · Flask en lugar de un framework más pesado

**Decisión:** Flask con blueprints, SQLAlchemy y SQLite.
**Contexto:** oficina pequeña, pocas rutas, despliegue simple.
**Justificación:** mínima ceremonia, fácil de leer, un solo proceso. Django habría aportado admin y auth, pero también más configuración y una base más pesada.
**Costo:** auth y CSRF se implementan a mano; no hay admin.
**Lección:** para un equipo pequeño, Flask es adecuado; para requisitos de auditoría y administración, conviene Django o un conjunto de extensiones.

#### D-10 · SQLite como base principal

**Decisión:** SQLite embebida, sin servidor.
**Justificación:** instalación cero, backup por archivo, suficiente para una oficina.
**Costo:** escrituras serializadas, sin concurrencia real, sin roles de base.
**Límite:** si hay varias sucursales o más de ~10 operadores, migrar a PostgreSQL. El código usa SQLAlchemy, lo que reduce el costo de migración.

#### D-11 · SSE en lugar de WebSocket

**Decisión:** Server-Sent Events para el display.
**Justificación:** el flujo es unidireccional; SSE reconecta solo, funciona sobre HTTP y no requiere un servidor de sockets.
**Costo:** bus en memoria, un solo worker, sin acknowledgment.
**Alternativa descartada:** WebSocket. Se habría justificado si el kiosco o el ejecutivo necesitaran comunicación bidireccional de baja latencia. No es el caso.
**Lección:** para notificaciones unidireccionales, SSE es la opción más simple; para escalar, el bus debe ser externo.

#### D-12 · Token QR separado del código

**Decisión:** `qr_token = uuid4().hex`, distinto de `codigo`.
**Justificación:** el código es visible en la pantalla y en el ticket; el token es una capacidad que autoriza iniciar la atención. Separarlos evita que conocer el código baste para manipular el turno.
**Costo:** el token debe imprimirse y escanearse; si se pierde, no hay recuperación desde la UI.
**Lección:** modelar capacidades, no identificadores, para acciones sensibles.

#### D-13 · Validación de RUT en cliente y servidor

**Decisión:** duplicar la validación módulo 11.
**Justificación:** el cliente da feedback inmediato; el servidor no confía en el cliente.
**Costo:** dos implementaciones que pueden divergir.
**Lección:** compartir la lógica si se puede; si no, cubrir ambos lados con los mismos casos de prueba.

#### D-14 · Seed automático al primer arranque

**Decisión:** `db.create_all()` y `run_seed()` si no hay ejecutivos.
**Justificación:** permite probar el flujo sin pasos manuales.
**Riesgo:** credenciales demo conocidas y datos de ejemplo en producción.
**Acción recomendada:** separar el seed del arranque, exigir una variable `SEED_DEMO=1` o un comando explícito.

#### D-15 · CSRF artesanal

**Decisión:** exigir JSON, `X-Requested-With` y origen/referer del mismo host.
**Justificación:** no agregar dependencias y bloquear formularios cross-site simples.
**Límite:** no es un token CSRF; un mismo-origen comprometido o un cliente que replique encabezados puede pasar.
**Acción recomendada:** Flask-WTF o token por sesión.

#### D-16 · Contraseñas con bcrypt

**Decisión:** hash con bcrypt y salt por contraseña.
**Justificación:** estándar probado, resistente a fuerza bruta con costo configurable.
**Observación:** el costo por defecto de `gensalt()` es adecuado para 2026; revisar si el hardware es muy limitado.

#### D-17 · Sin migraciones

**Decisión:** `db.create_all()` en lugar de Alembic.
**Justificación:** simplicidad inicial.
**Riesgo:** cualquier cambio de esquema requiere intervención manual y puede perder datos.
**Acción recomendada:** introducir Alembic antes de la primera actualización en producción.

#### D-18 · Dependencias de frontend por CDN

**Decisión:** Bootstrap, qrcodejs y html5-qrcode por CDN.
**Justificación:** menos archivos y actualizaciones automáticas.
**Riesgo:** sin internet no hay estilos ni QR; además, un CDN comprometido afecta la UI.
**Acción recomendada:** para despliegue real, vendorizar los archivos y fijar versiones con integridad (SRI).

#### D-19 · Código correlativo por conteo

**Decisión:** `prefijo + (count + 1)`.
**Justificación:** simple y legible.
**Riesgo:** condición de carrera bajo solicitudes simultáneas; códigos duplicados si dos transacciones cuentan a la vez.
**Acción recomendada:** secuencia por tipo o índice único con reintento.

### 2.3 notificador

#### D-20 · Biblioteca estándar, sin dependencias

**Decisión:** usar solo `sqlite3`, `smtplib`, `email`, `logging`, `argparse`.
**Justificación:** despliegue trivial, sin venv obligatorio, compatible con servidores antiguos.
**Costo:** más código manual y menos abstracciones.
**Lección:** para una utilidad acotada, la stdlib es una ventaja; para crecer, conviene un cliente SMTP con reintentos.

#### D-21 · Leer el carrito por convención de nombre

**Decisión:** descubrir tablas `carrito_AAAAMMDD_HHMMSS` y ordenar por timestamp.
**Justificación:** evita una tabla de metadatos y funciona con el esquema existente.
**Riesgo:** si el productor cambia el formato, el selector falla o elige mal. Los nombres inválidos se ordenan al final, pero no se rechazan.
**Acción recomendada:** validar el patrón, ignorar nombres inválidos y registrar una advertencia.

#### D-22 · Dry-run

**Decisión:** `--dry-run` loguea el correo y retorna éxito.
**Justificación:** permite probar sin credenciales SMTP y sin enviar correos reales.
**Lección:** toda utilidad que envía algo debería tener un modo de simulación.

#### D-23 · SMTP sin TLS ni login

**Hecho:** el código abre `SMTP(host, 587)` y envía sin `starttls()` ni `login()`.
**Causa probable:** placeholder para un relay local o para completar después.
**Riesgo:** la mayoría de los proveedores rechazan el envío; sin TLS, las credenciales (si se agregaran) viajarían en claro.
**Acción recomendada:** completar TLS, autenticación, reintentos y manejo de errores.

#### D-24 · Ruta por defecto a otro proyecto

**Hecho:** `DB_PATH_DEFECTO` apunta a una base de un proyecto de kiosco, no al turnero.
**Causa probable:** el notificador nació en otro contexto y se importó tal cual.
**Consecuencia:** no es una integración lista con turnero.
**Decisión:** documentar la limitación y no fingir una integración inexistente.
**Lección:** al agrupar proyectos, explicitar las dependencias reales y no asumir que comparten esquema.

---

## 3. Problemas encontrados y cómo se resolvieron

### P-01 · El proyecto Autosale no compila

**Síntoma:** los archivos de formulario no son C# de texto; el diseñador y el compilador no pueden procesarlos.
**Diagnóstico:** `file` los reporta como datos/OpenPGP; los tamaños son anómalos (246 KB para un `.cs`).
**Solución:** conservarlos como evidencia, documentar la limitación y centrar el valor del repositorio en el código legible (`librerias.cs`, `Class1.cs`, formularios pequeños).
**Estado:** limitación aceptada; no se intentó descifrar ni reconstruir.

### P-02 · Credenciales legacy en Autosale

**Síntoma:** `librerias.cs` contiene usuario/clave SMTP y el repositorio incluye un `.pfx`.
**Diagnóstico:** código histórico sin gestión de secretos.
**Solución:** no reproducir los valores en la documentación; advertir la rotación; recomendar externalización.
**Estado:** riesgo documentado. Si el repositorio se publica, revisar el historial y rotar credenciales.

### P-03 · SSE se congela detrás de un proxy

**Síntoma posible:** el display no recibe eventos si hay un proxy con buffering.
**Diagnóstico:** Nginx acumula la respuesta `text/event-stream`.
**Solución:** `proxy_buffering off`, `X-Accel-Buffering: no` y timeout largo.
**Estado:** documentado en `developer.md` y en la instalación.

### P-04 · Varios workers no comparten eventos

**Síntoma:** con Gunicorn `-w 4`, algunos displays no ven los llamados.
**Diagnóstico:** `_event_listeners` es una lista en memoria de cada proceso.
**Solución:** usar un solo worker o un bus externo (Redis/PostgreSQL).
**Estado:** limitación documentada; el despliegue recomendado usa `-w 1`.

### P-05 · La base se bloquea con escrituras concurrentes

**Síntoma:** `database is locked` bajo carga.
**Diagnóstico:** SQLite serializa escrituras; una transacción larga bloquea a las demás.
**Solución:** transacciones cortas, WAL, un solo proceso de escritura; para más carga, PostgreSQL.
**Estado:** documentado como límite.

### P-06 · Códigos de turno duplicados

**Síntoma:** dos turnos con el mismo código bajo solicitudes simultáneas.
**Diagnóstico:** `COUNT + 1` no es atómico.
**Solución:** índice único y reintento, o secuencia por tipo. En la versión actual, el riesgo es bajo para una oficina pequeña.
**Estado:** deuda técnica identificada.

### P-07 · El notificador no encuentra la base

**Síntoma:** `No such file or directory`.
**Diagnóstico:** la ruta por defecto es relativa a otro proyecto; el `--db-path` se resuelve respecto al script, no al cwd.
**Solución:** usar una ruta absoluta o `--db-path` correcto; documentar la resolución.
**Estado:** resuelto por configuración.

### P-08 · El notificador elige una tabla inválida

**Síntoma:** se envía un carrito con nombre fuera de formato.
**Diagnóstico:** `seleccionar_ultimo_carrito` asigna clave `""` a nombres inválidos y, si no hay válidos, devuelve el primero.
**Solución:** filtrar por el patrón antes de ordenar y devolver `None` si no hay válidos.
**Estado:** mejora pendiente.

### P-09 · El correo no se envía

**Síntoma:** `SMTPAuthenticationError` o conexión rechazada.
**Diagnóstico:** falta `starttls()` y `login()`; el puerto 587 exige TLS en la mayoría de los proveedores.
**Solución:** completar el flujo SMTP y externalizar credenciales.
**Estado:** limitación conocida y documentada.

### P-10 · Logs duplicados al importar

**Síntoma:** cada mensaje aparece dos veces.
**Diagnóstico:** `configurar_logging()` agrega handlers en cada llamada.
**Solución:** la función retorna si ya hay handlers (implementado).
**Estado:** resuelto.

### P-11 · Dependencia de CDN en una red aislada

**Síntoma:** el kiosco se ve sin estilos y el QR no se genera.
**Diagnóstico:** Bootstrap/qrcodejs/html5-qrcode se cargan desde internet.
**Solución:** descargar los archivos y servirlos desde `static/`, o permitir el acceso a los CDN.
**Estado:** documentado.

### P-12 · El display muestra datos personales

**Riesgo:** usar `to_dict()` en un endpoint público expone RUT y `qr_token`.
**Diagnóstico:** el modelo tiene dos serializadores; el público debe ser `to_display_dict()`.
**Solución:** usar siempre `to_display_dict()` en display y SSE; revisar cualquier endpoint nuevo.
**Estado:** implementado; se recuerda en la guía.

---

## 4. Enfoques descartados y por qué

| Enfoque | Por qué se descartó / no se adoptó |
|---|---|
| WebSocket para el display | El flujo es unidireccional; SSE es más simple y reconecta solo |
| JWT para la sesión del ejecutivo | No hay consumidores externos; una cookie firmada alcanza y evita gestión de tokens |
| PostgreSQL desde el inicio | Instalación y operación más pesadas para una oficina sin DBA |
| Django | Más estructura y admin, pero también más configuración y dependencias |
| Microservicios | El volumen no los justifica; añaden red, despliegue y observabilidad |
| Guardar el estado del turno en el cliente | La fuente de verdad debe ser el servidor; el cliente solo refleja |
| Usar el código del turno como token QR | El código es público; el token debe ser una capacidad separada |
| Enviar el correo desde el frontend | Expondría credenciales SMTP y acoplaría la UI al correo |
| Leer el turno con un ORM en Autosale | El código histórico usa ADO.NET/SQL directo; migrar habría sido otro proyecto |
| Reconstruir los formularios opacos | Sin el proyecto original, es adivinar; se prefirió documentar la limitación |
| Publicar los archivos opacos como “código” | No son texto C#; induciría a error y podría exponer datos |
| Unificar las tres versiones en un solo ejecutable | Son contextos distintos; forzar la unificación habría ocultado la evolución |

---

## 5. Lecciones aprendidas

### Sobre arquitectura

1. **La unidad de despliegue define la arquitectura.** Un tótem por equipo y un servidor web central tienen costos distintos; no se puede copiar una solución de un contexto a otro.
2. **El estado debe tener una sola fuente de verdad.** El servidor decide; el cliente refleja. El display nunca debe inferir transiciones.
3. **Los identificadores no son capacidades.** El código del turno se muestra; el token QR autoriza. Separarlos reduce el riesgo.
4. **Los eventos en memoria no escalan horizontalmente.** SSE es simple con un proceso; con varios, hace falta un bus.
5. **Las bases embebidas son una decisión de producto, no solo técnica.** SQLite es correcta si el despliegue es pequeño y el backup simple; deja de serlo con concurrencia o sucursales.

### Sobre hardware

6. **Aislar el protocolo serial paga.** Una clase dedicada permite probar con tramas grabadas y cambiar el transporte sin tocar la UI.
7. **El parseo por offsets es frágil.** Un parser por nombres de campo y validación de longitud es más mantenible.
8. **La ausencia de red es un requisito, no una excepción.** Autosale lo entendió con base local y sincronización; turnero asume red local, lo cual es aceptable en su contexto.

### Sobre seguridad

9. **Los secretos no van en el código.** Las credenciales SMTP legacy son un recordatorio costoso.
10. **Las credenciales demo deben separarse del arranque.** Un seed automático en producción es un riesgo.
11. **CSRF no es un header mágico.** Se necesita un token ligado a la sesión; lo demás es defensa en profundidad.
12. **El display es público por diseño.** No incluir RUT, tokens ni datos personales en sus respuestas.

### Sobre documentación y portafolio

13. **Documentar lo que no se puede ejecutar también es valioso.** Autosale aporta valor como análisis de protocolo y arquitectura, aunque no compile.
14. **Declarar las limitaciones genera confianza.** Un README que dice “esto no está listo para producción” es más útil que uno que lo oculta.
15. **La evolución se explica con decisiones, no con fechas.** Sin actas, la bitácora debe distinguir hechos de inferencias.
16. **Los diagramas SVG son mantenibles y versionables.** Mejor que capturas de pantalla para un repositorio.

### Sobre operación

17. **Un solo worker es una decisión consciente.** Si se usa SSE en memoria, escalar workers rompe el tiempo real.
18. **El proxy importa.** `proxy_buffering off` es la diferencia entre un display vivo y uno congelado.
19. **El backup de SQLite se hace con `.backup`, no copiando el archivo abierto.** Evita copias inconsistentes.
20. **Las tareas programadas necesitan logs y códigos de salida.** El notificador lo hace bien; el cron debe monitorearlos.

---

## 6. Deuda técnica priorizada

| Prioridad | Elemento | Impacto | Esfuerzo |
|---|---|---|---|
| Alta | Rotar credenciales legacy de Autosale y excluir el `.pfx` | Seguridad | Bajo |
| Alta | Reemplazar CSRF artesanal por token de sesión | Seguridad | Medio |
| Alta | Completar TLS/login del notificador | Funcional | Bajo |
| Alta | Quitar seed automático o condicionarlo | Seguridad/operación | Bajo |
| Media | Migraciones con Alembic | Mantenimiento | Medio |
| Media | Secuencia atómica de códigos | Integridad | Medio |
| Media | Vendorizar assets del frontend | Disponibilidad | Bajo |
| Media | Tests de servicios y API | Calidad | Medio |
| Baja | Bus de eventos para escalar SSE | Escalabilidad | Alto |
| Baja | Informes históricos y exportación | Producto | Medio |
| Baja | Reemplazar `Query.get()` por `db.session.get()` | Modernización | Bajo |
| Baja | Adaptador de notificaciones para turnos | Integración | Medio |

---

## 7. Criterios para futuras iteraciones

Si el proyecto se retoma, el orden sugerido es:

1. **Sanear Autosale** (secretos, certificado, archivos opacos) antes de cualquier publicación.
2. **Endurecer turnero** (CSRF, seed, migraciones, rate limiting) antes de exponerlo.
3. **Completar notificador** (TLS, auth, reintentos) antes de usarlo con datos reales.
4. **Agregar pruebas** a los flujos críticos: solicitud, llamado, inicio, cierre, login.
5. **Documentar el despliegue real** con proxy, TLS, respaldo y monitoreo.
6. **Recién después** evaluar escalabilidad (PostgreSQL, bus de eventos, múltiples sucursales).

---

## 8. Registro de verificación documental

| Verificación | Resultado |
|---|---|
| README raíz con evolución y mapa de versiones | Sí |
| README con instalación y comandos de venv | Sí |
| README con pinout/conexiones | Sí (hardware de Autosale y despliegue) |
| README con ejemplos de uso | Sí |
| README con calibración | Sí (hardware y umbrales) |
| README con advertencias | Sí |
| `docs/datasheet.md` | Sí |
| `docs/developer.md` | Sí |
| `docs/REFERENCES.md` | Sí |
| `docs/BITACORA.md` | Sí |
| SVGs en `docs/assets/` | Sí |
| `LICENSE` MIT | Sí |
| Sin credenciales ni bases publicadas | Sí (se citan solo nombres de variables y usuarios demo) |
| Sin `docs/index.html` | Sí |
| Código y `.git` intactos | Sí |

---

*Bitácora reconstruida para el portafolio. Las decisiones inferidas se basan en el código y la documentación disponibles; no reemplazan un registro histórico de reuniones o commits.*

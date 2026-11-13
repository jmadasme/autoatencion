# Guía de desarrollador · Sistema de autoatención

**Audiencia:** personas que mantienen, extienden o estudian el código de `versions/`.
**Alcance:** arquitectura, módulos, modelo de datos, API, convenciones, pruebas y guías de extensión.
**Licencia:** MIT (ver `LICENSE`).

---

## 1. Mapa del repositorio

```text
autoatencion/
├── README.md                  ← visión de conjunto y operación
├── LICENSE                    ← MIT, Copyright (c) 2026 jmadasme
├── docs/
│   ├── datasheet.md           ← documentación de usuario
│   ├── developer.md           ← este documento
│   ├── REFERENCES.md          ← enlaces oficiales
│   ├── BITACORA.md            ← decisiones y lecciones
│   └── assets/                ← diagramas SVG
└── versions/
    ├── Autosale/              ← primera generación (C# WinForms)
    │   ├── AutoSell.sln
    │   ├── AutoSell/
    │   ├── Biblioteca/
    │   └── VERSION.md
    ├── turnero/               ← segunda generación (Flask)
    │   ├── app.py
    │   ├── config.py
    │   ├── models.py
    │   ├── seed.py
    │   ├── requirements.txt
    │   ├── routes/
    │   ├── services/
    │   ├── templates/
    │   ├── static/
    │   └── docs/
    └── notificador/           ← tercera generación (stdlib)
        ├── notificador.py
        ├── requirements.txt
        └── docs/
```

Cada versión es un proyecto independiente. No hay imports cruzados ni un paquete Python común. La documentación raíz describe el conjunto; la documentación dentro de cada versión describe su implementación original.

---

## 2. Filosofía de la evolución

La solución atravesó tres etapas con restricciones distintas:

1. **Autosale**: hardware propietario, Windows, SQL Server, operación crítica. Se priorizó que funcionara con periféricos y que tolerara redes intermitentes. El costo fue el acoplamiento y la dificultad de mantener.
2. **turnero**: oficina pequeña, múltiples pantallas, sin administrador de base de datos. Se priorizó la simplicidad de despliegue y la coherencia en tiempo real. El costo fue depender de un solo proceso y de una base embebida.
3. **notificador**: automatización acotada, sin UI. Se priorizó la ausencia de dependencias y la observabilidad. El costo fue un envío SMTP incompleto y un acoplamiento por convención de nombres.

La lección transversal: **la unidad de despliegue manda**. Un binario por tótem exige otra disciplina que un servicio web; un script de cron exige otra que una app con usuarios.

---

## 3. turnero · arquitectura interna

### 3.1 Application factory

`app.py` define `create_app()`:

```python
def create_app() -> Flask:
    app = Flask(__name__)
    app.config.from_object(Config)
    db.init_app(app)
    app.register_blueprint(display_bp)
    app.register_blueprint(kiosco_bp)
    app.register_blueprint(executive_bp)
    app.register_blueprint(api_bp)

    @app.errorhandler(500)
    def internal_error(error):
        if request.path.startswith("/api/"):
            return {"error": "Error interno del servidor"}, 500
        return render_template("base.html", error="Error interno"), 500

    with app.app_context():
        _initialize_database()
    return app
```

`_initialize_database()` ejecuta `db.create_all()` y, si `Ejecutivo.query.first()` es `None`, llama a `seed.run_seed()`. Es una inicialización idempotente y suficiente para desarrollo; en producción conviene migrar a Alembic y separar el seed.

`main()` lee `PORT` (default 5000), imprime la URL y ejecuta `app.run(host="0.0.0.0", port=port, debug=True)`. **No usar `debug=True` en producción.**

### 3.2 Configuración

`config.py`:

```python
class Config:
    SECRET_KEY = os.environ.get("SECRET_KEY", "turnero-secret-key-dev-2025")
    SQLALCHEMY_DATABASE_URI = os.environ.get("DATABASE_URL", "sqlite:///turnero.db")
    SQLALCHEMY_TRACK_MODIFICATIONS = False
```

Si `SECRET_KEY` no está definida, se emite un `RuntimeWarning`. La URI relativa de SQLite se resuelve dentro de `instance/` por Flask-SQLAlchemy.

### 3.3 Blueprints

| Archivo | Prefijo | Rutas |
|---|---|---|
| `routes/display.py` | ninguno | `GET /` |
| `routes/kiosco.py` | ninguno | `GET /kiosco` |
| `routes/executive.py` | ninguno | `GET /login`, `GET /ejecutivo` |
| `routes/api.py` | ninguno | `/api/...` |

Las rutas de página solo renderizan templates. Toda la lógica de mutación está en la API y en los servicios.

### 3.4 Modelo de datos

```python
class Modulo(db.Model):
    __tablename__ = "modulo"
    id, nombre, activo
    ejecutivos = relationship("Ejecutivo", backref="modulo")
    turnos = relationship("Turno", backref="modulo", foreign_keys="Turno.modulo_id")

class Ejecutivo(db.Model):
    __tablename__ = "ejecutivo"
    id, username (unique), password_hash, nombre, modulo_id (FK), activo
    turnos = relationship("Turno", backref="ejecutivo", foreign_keys="Turno.ejecutivo_id")

class TipoTramite(db.Model):
    __tablename__ = "tipo_tramite"
    id, codigo (unique, 1 char), nombre, prefijo (1 char)
    turnos = relationship("Turno", backref="tipo_tramite")

class Turno(db.Model):
    __tablename__ = "turno"
    id, codigo (unique), tipo_tramite_id (FK), rut_cliente, estado (index)
    modulo_id (FK nullable), ejecutivo_id (FK nullable)
    fecha_creacion, fecha_llamado, fecha_inicio_atencion, fecha_completado
    tipo_atencion, comentarios, qr_token (unique)
```

Notas:

- `fecha_creacion` usa `datetime.now(timezone.utc)`; SQLite guarda el valor sin zona, pero la intención es UTC.
- `qr_token` es `uuid4().hex` (32 caracteres), no el código visible.
- `to_dict()` incluye `qr_token`; `to_display_dict()` lo elimina. **Nunca** usar `to_dict()` en endpoints públicos.
- `Ejecutivo.to_dict()` incluye `modulo_nombre`, pero no el hash.

### 3.5 Servicios de turnos

#### `validar_rut(rut: str) -> bool`

1. Elimina puntos, guiones y espacios.
2. Exige al menos dos caracteres y cuerpo numérico.
3. Recorre el cuerpo de derecha a izquierda con multiplicadores 2..7.
4. Calcula `11 - (suma % 11)`; `11 → "0"`, `10 → "K"`.
5. Compara con el dígito ingresado en mayúscula.

No valida longitud mínima/máxima del cuerpo; un RUT de un dígito puede pasar si el DV coincide. Si se necesita una validación más estricta, agregar un rango de longitud.

#### `_generar_codigo_turno(tipo_tramite_id: int) -> str`

Busca el `TipoTramite`, cuenta los turnos históricos de ese tipo y devuelve `prefijo + (count + 1)`. Ejemplo: si hay 42 turnos B, el siguiente es `B43`.

**Condición de carrera:** dos solicitudes simultáneas pueden contar el mismo valor. Para una oficina pequeña es tolerable; para concurrencia alta, usar una secuencia por tipo o un índice único con reintento.

#### `solicitar_turno(tipo_tramite_id, rut_cliente) -> Turno`

Valida el RUT, genera código y token, crea el turno en `esperando`, lo persiste y lo devuelve. Lanza `ValueError("RUT inválido")` o `ValueError("Tipo de tramite no encontrado")`.

#### `llamar_siguiente(ejecutivo_id) -> Turno | None`

1. Busca el ejecutivo; si no existe, `ValueError`.
2. Verifica que no tenga un turno en `llamado` o `en_atencion`; si lo tiene, `ValueError`.
3. Toma el `esperando` más antiguo por `fecha_creacion`.
4. Asigna `modulo_id`, `ejecutivo_id`, `fecha_llamado` y estado `llamado`.
5. Devuelve el turno o `None` si no hay.

No hay bloqueo pesimista. En SQLite, las escrituras se serializan; en PostgreSQL habría que considerar `SELECT ... FOR UPDATE SKIP LOCKED` para varios operadores.

#### `iniciar_atencion(qr_token, ejecutivo_id) -> Turno | None`

Busca por token. Si no existe, `None`. Si el estado no es `llamado`, `ValueError`. Si el ejecutivo no coincide, `ValueError`. Si todo está bien, pasa a `en_atencion` y registra la fecha.

#### `finalizar_atencion(ejecutivo_id, tipo_atencion=None, comentarios=None) -> tuple[Turno, Turno | None]`

Busca el turno `en_atencion` del ejecutivo; si no hay, `ValueError`. Lo marca `completado`, guarda clasificación, comentarios y fecha, y llama a `llamar_siguiente`. Devuelve el turno cerrado y el siguiente (o `None`).

#### Consultas

- `get_cola()`: recorre todos los `TipoTramite` y devuelve, por tipo, `{tipo_tramite_id, tipo_tramite, codigo, total, turnos[]}` con `to_display_dict()`.
- `get_estado_display()`: por cada módulo activo, busca el primer ejecutivo activo y el turno actual (`llamado` o `en_atencion`) más reciente por `fecha_llamado`.
- `get_ejecutivo_estado(ejecutivo_id)`: devuelve `{ejecutivo, turno_activo}`; el turno activo se expone con `to_dict()` (incluye token) porque el endpoint requiere sesión.

### 3.6 Autenticación y sesión

`auth_service.py`:

```python
def hash_password(password: str) -> str:
    return bcrypt.hashpw(password.encode(), bcrypt.gensalt()).decode()

def check_password(password: str, password_hash: str) -> bool:
    return bcrypt.checkpw(password.encode(), password_hash.encode())

def login_ejecutivo(username: str, password: str) -> Ejecutivo | None:
    ejecutivo = Ejecutivo.query.filter_by(username=username, activo=True).first()
    if ejecutivo is None or not check_password(password, ejecutivo.password_hash):
        return None
    session["ejecutivo_id"] = ejecutivo.id
    session["ejecutivo_nombre"] = ejecutivo.nombre
    session["modulo_id"] = ejecutivo.modulo_id
    return ejecutivo
```

La sesión es una cookie firmada por Flask. `login_required` redirige a `executive.login`; `api_login_required` devuelve `{"error": "No autenticado"}, 401`.

#### Protección CSRF

`csrf_protected` exige:

1. `Content-Type` con `application/json`.
2. `X-Requested-With: XMLHttpRequest`.
3. Si hay `Origin`, debe empezar por `request.host_url`; si hay `Referer`, ídem.

Fortalezas: bloquea formularios HTML cross-site y peticiones simples. Debilidades: no usa token por sesión, y un atacante con capacidad de ejecutar JavaScript en el mismo origen ya estaría dentro. Para producción, migrar a Flask-WTF o a un token sincronizado.

### 3.7 API

Base: `http://<host>:5000`. Todas las respuestas son JSON salvo SSE.

#### Autenticación

`POST /api/auth/login`

```http
Content-Type: application/json
X-Requested-With: XMLHttpRequest

{"username": "ejecutivo1", "password": "<demo>"}
```

```json
200 OK
{
  "mensaje": "Login exitoso",
  "ejecutivo": {
    "id": 1,
    "username": "ejecutivo1",
    "nombre": "Ejecutivo Demo",
    "modulo_id": 1,
    "modulo_nombre": "Modulo 1",
    "activo": true
  }
}
```

Errores: `400` datos faltantes, `401` credenciales inválidas, `400` CSRF.

`POST /api/auth/logout` → `200 {"mensaje": "Sesión cerrada"}`.

#### Consultas públicas

`GET /api/tramites` → `{"tramites": [...]}`. Devuelve todos los tipos, sin filtro de activo (el modelo no tiene ese campo).

`GET /api/modulos` → `{"modulos": [...]}` solo activos.

`GET /api/turnos/display` → `{"modulos": [{"modulo": {...}, "ejecutivo": {...}|null, "turno_actual": {...}|null}]}`.

`GET /api/turnos/eventos` → `text/event-stream`. Cada evento:

```text
data: {"type": "turno_llamado", "data": {"codigo": "A1", "modulo_id": 1, "modulo_nombre": "Modulo 1", "ejecutivo": "Ejecutivo Demo", "turno": {...}}}

```

Heartbeat cada 30 s: `: heartbeat\n\n`. Sin autenticación.

#### Kiosco

`POST /api/turnos/solicitar`

```json
{"tipo_tramite_id": 2, "rut": "12345678-5"}
```

```json
201 Created
{
  "mensaje": "Turno creado exitosamente",
  "turno": {
    "id": 10,
    "codigo": "B1",
    "tipo_tramite": "Consultas Generales",
    "rut_cliente": "12345678-5",
    "estado": "esperando",
    "qr_token": "...",
    ...
  }
}
```

Errores: `400` JSON faltante, RUT inválido, tipo inexistente.

#### Ejecutivo (requieren sesión)

`GET /api/ejecutivo/siguiente`

- `200 {"mensaje": "Turno llamado exitosamente", "turno": {...}}`
- `200 {"mensaje": "No hay turnos en espera", "turno": null}`
- `400` si ya hay un turno activo.

`POST /api/ejecutivo/iniciar-atencion`

```json
{"qr_token": "..."}
```

- `200 {"mensaje": "Atención iniciada", "turno": {...}}`
- `400` estado incorrecto o ejecutivo ajeno.
- `404` token inexistente.

`POST /api/ejecutivo/finalizar-atencion`

```json
{"tipo_atencion": "Resolución", "comentarios": "Sin observaciones"}
```

- `200 {"mensaje": "Atención finalizada", "turno_completado": {...}, "turno_siguiente": {...}|null}`
- `400` si no hay turno en atención.

`GET /api/ejecutivo/cola` → `{"cola": [...]}`.

`GET /api/ejecutivo/estado` → `{"ejecutivo": {...}, "turno_activo": {...}|null}`.

### 3.8 SSE en detalle

```python
_event_listeners: list[queue.Queue] = []
_listeners_lock = threading.Lock()

def broadcast_event(event_type: str, data: dict) -> None:
    payload = json.dumps({"type": event_type, "data": data})
    dead = []
    with _listeners_lock:
        for q in _event_listeners:
            try:
                q.put_nowait(payload)
            except queue.Full:
                dead.append(q)
        for q in dead:
            _event_listeners.remove(q)
```

- Cada cliente tiene una cola de 100 eventos.
- Si se llena, se descarta el listener (el navegador reconecta).
- El generador consulta con timeout de 5 s para poder emitir heartbeats.
- `stream_with_context` mantiene el contexto de Flask.
- `X-Accel-Buffering: no` evita buffering en Nginx.

**Limitación:** el bus vive en memoria del proceso. Con Gunicorn y varios workers, cada worker tiene su lista y los eventos no se comparten. Opciones: un solo worker, o Redis/PostgreSQL como backend de eventos.

### 3.9 Frontend

Los templates extienden `base.html` y cargan Bootstrap 5.3.3 por CDN. `styles.css` define la identidad visual (azul petróleo, tarjetas, tipografía).

- `display.js`: `fetch('/api/turnos/display')` al cargar; `new EventSource('/api/turnos/eventos')`; en `onerror`, cierra y reintenta a los 3 s; renderiza tarjetas por módulo.
- `kiosco.js`: estado de tres pasos; `formatearRut` y `validarRut` en cliente; `fetch` con `Content-Type: application/json` y `X-Requested-With`; QR con `new QRCode(...)`; fallback de trámites.
- `executive.js`: polling de cola (15 s) y estado (10 s); escaneo con `Html5Qrcode`; cronómetro; acciones de llamar/iniciar/finalizar; logout.

### 3.10 Seed

`seed.py` crea:

- Tipos: `A` Créditos, `B` Consultas Generales, `C` Retiro de Documentos.
- Módulos: `Modulo 1`, `Modulo 2`, `Modulo 3`.
- Tres ejecutivos demo, uno por módulo, con contraseñas de desarrollo.

`run_seed()` no hace nada si ya existe algún ejecutivo. Para reiniciar, borrar `instance/turnero.db` y volver a arrancar. **Nunca** usar estas credenciales en producción.

### 3.11 Cómo extender turnero

#### Agregar un tipo de trámite

1. Insertar en `tipo_tramite` (`codigo`, `nombre`, `prefijo`) o editar `seed.py`.
2. Si el código es nuevo, usar una letra única.
3. El kiosco lo mostrará automáticamente vía `/api/tramites`.
4. Verificar que el prefijo no colisione con códigos existentes.

#### Agregar un módulo

1. Crear un `Modulo` activo.
2. Asignarle al menos un ejecutivo activo.
3. El display lo mostrará; el ejecutivo verá su módulo en la sesión.

#### Agregar un ejecutivo

1. Crear el `Ejecutivo` con `modulo_id` válido.
2. Generar el hash con `hash_password("...")`.
3. Marcar `activo=True`.
4. Verificar el login.

#### Agregar un campo al turno

1. Agregar la columna en `models.py`.
2. Si la base ya existe, migrar (Alembic o `ALTER TABLE` manual).
3. Actualizar `to_dict()` y, si corresponde, `to_display_dict()`.
4. Ajustar la API y la UI.
5. Agregar pruebas.

#### Cambiar la política de asignación

`llamar_siguiente` es el punto único. Para prioridades, agregar un campo `prioridad` y ordenar por `(prioridad DESC, fecha_creacion ASC)`. Para módulos especializados, filtrar por `tipo_tramite` compatible.

#### Agregar autenticación al display

Crear un blueprint con login o un token de acceso, y proteger `/` y `/api/turnos/display`. Considerar que el display suele ser un navegador en modo kiosco: una cookie de larga duración o un token en la URL puede ser más práctico que un login interactivo.

### 3.12 Pruebas sugeridas

No hay suite incluida. Un punto de partida con `pytest`:

```python
import pytest
from app import create_app
from models import db

@pytest.fixture
def client():
    app = create_app()
    app.config.update(TESTING=True, SQLALCHEMY_DATABASE_URI="sqlite:///:memory:")
    with app.app_context():
        db.create_all()
        yield app.test_client()
        db.drop_all()

def test_tramites(client):
    r = client.get("/api/tramites")
    assert r.status_code == 200
    assert "tramites" in r.get_json()
```

Casos mínimos:

- RUT válido, inválido, con K, con puntos.
- Solicitud de turno y código correlativo.
- FIFO en `llamar_siguiente`.
- Doble llamado rechazado.
- Inicio con token ajeno rechazado.
- Cierre y encadenamiento.
- Login correcto/incorrecto.
- CSRF sin encabezados.
- SSE: recibir un evento tras una acción.

---

## 4. notificador · arquitectura interna

### 4.1 Estructura

Un solo archivo, `notificador.py` (368 líneas), organizado en:

1. Constantes y imports.
2. `configurar_logging()`.
3. Funciones de base: `obtener_tablas_carrito`, `seleccionar_ultimo_carrito`, `leer_carrito`.
4. Funciones de correo: `formatear_email`, `enviar_email`.
5. CLI: `parse_args`, `main`.
6. Entrada: `sys.exit(main())`.

### 4.2 Funciones

```python
def configurar_logging() -> None
```
Configura root logger en DEBUG; consola INFO, archivo DEBUG. Si ya hay handlers, no hace nada (evita duplicados al importar).

```python
def obtener_tablas_carrito(con: sqlite3.Connection) -> list[str]
```
`SELECT name FROM sqlite_master WHERE type='table' AND name LIKE 'carrito_%'`.

```python
def seleccionar_ultimo_carrito(tablas: list[str]) -> str | None
```
Extrae `YYYYMMDD_HHMMSS` con `^carrito_(\d{8}_\d{6})$` y ordena descendente. Devuelve `None` si la lista está vacía. Los nombres inválidos reciben clave `""` y quedan al final; si **todos** son inválidos, devuelve el primero de la lista ordenada, no `None`. Es un caso a endurecer.

```python
def leer_carrito(con, nombre_tabla) -> tuple[list[dict[str, object]], float]
```
Lee `producto, cantidad, precio_unitario, subtotal`. Si `subtotal` es `NULL` o 0, calcula `cantidad * precio_unitario`; si alguno es `NULL`, omite la fila con warning. Redondea subtotal y total a 2 decimales.

**Inyección SQL:** el nombre de tabla se interpola con corchetes (`FROM [{nombre_tabla}]`). El nombre proviene de `sqlite_master`, no de entrada directa, pero un nombre malicioso podría escapar corchetes. Para endurecer, validar contra el patrón antes de interpolar o usar una lista blanca.

```python
def formatear_email(nombre_tabla, items, total, destino=DESTINO_DEFECTO) -> EmailMessage
```
Construye un `EmailMessage` de texto plano. Remitente fijo `notificador@kiosco.local`. Asunto `Notificación de carrito - {tabla}`. Cuerpo con fecha de generación, tabla de 60 columnas y TOTAL.

```python
def enviar_email(msg: EmailMessage, dry_run: bool) -> bool
```
- `dry_run=True`: loguea destinatario, asunto y cuerpo; retorna `True`.
- Real: `smtplib.SMTP(host, 587, timeout=30)`, `send_message`, sin `starttls` ni `login`.
- Captura `SMTPException` y `OSError`; retorna `False`.

```python
def parse_args(argv: list[str] | None = None) -> argparse.Namespace
```
Opciones `--dry-run`, `--db-path`, `--destino`.

```python
def main(argv: list[str] | None = None) -> int
```
Valida la base, conecta, elige carrito, lee, formatea, envía. Retorna 0 o 1. Cierra la conexión en `finally`.

### 4.3 Flujo de datos

```text
--db-path → Path.resolve() relativo al script
          → sqlite3.connect()
          → obtener_tablas_carrito()
          → seleccionar_ultimo_carrito()
          → leer_carrito()
          → formatear_email()
          → enviar_email(dry_run)
          → logging + exit code
```

### 4.4 Cómo extender notificador

#### Completar TLS y autenticación

```python
import os
import smtplib

with smtplib.SMTP(SMTP_HOST, SMTP_PORT, timeout=30) as server:
    server.starttls()
    server.login(os.environ["SMTP_USER"], os.environ["SMTP_PASS"])
    server.send_message(msg)
```

Agregar validación de certificado (por defecto `starttls()` ya la hace), manejo de `SMTPAuthenticationError` y reintentos con backoff.

#### Varios destinatarios

Aceptar `--destino` repetible (`action="append"`) y unir con comas en `msg["To"]`. Validar con `email.utils.parseaddr`.

#### Otra fuente de datos

Sustituir `leer_carrito` por un adaptador que lea la tabla `turno` del turnero y devuelva la misma estructura `(items, total)`. El resto del pipeline no cambia.

#### Adjuntos

`EmailMessage.add_attachment(...)` con `maintype`, `subtype` y `filename`. Cuidar el tamaño y no adjuntar datos personales sin consentimiento.

---

## 5. Autosale · mapa de código

### 5.1 Proyectos y archivos

| Archivo | Rol | Estado |
|---|---|---|
| `Program.cs` | Punto de entrada | Legible |
| `Autenticacion.cs` | Login local | Legible |
| `IngresoTarjeta.cs` | RFID `COM7` | Legible |
| `Confirmacion.cs` | Confirmación y moneda | Legible |
| `TransBankGestion.cs` | Comandos manuales | Legible |
| `librerias.cs` | Infraestructura | Legible, 1118 líneas |
| `Class1.cs` | Sync + copia de TransBank | Legible |
| `Principal.cs` | Formulario principal | Opaco (246 KB) |
| `Principal.Designer.cs` | Diseño | Opaco (96 KB) |
| `IngresoTransbank.cs` | Cobro | Opaco (51 KB) |
| `mensajesPantalla.*` | Mensajes | Opaco |
| `Test_Int_POS.cs` | Prueba | Opaco/OpenPGP (62 KB) |

### 5.2 Clases principales

```csharp
namespace Herramientas
{
    public class funciones { /* conexión, parámetros, correo, conectividad */ }
    public class SincronizaVentas { /* copia de tablas */ }
    public class Jobcalendario { /* Quartz */ }
    public class JobDeSincronizacion { /* productos, clientes, auditoría */ }
    public class JobDeCierreZ { /* cierre */ }
    public class ResgistroError { /* log */ }
    public class TransBank : IDisposable { /* protocolo serial */ }
    public class MensajesTbk : EventArgs { /* mensaje */ }
}
```

### 5.3 Protocolo serial

| Elemento | Valor |
|---|---|
| STX | `0x02` |
| ETX | `0x03` |
| Separador | `|` (`0x7C`) |
| ACK | `0x06` |
| Checksum | XOR de todos los caracteres, incluido ETX |
| Fin de trama | ETX |

Formato lógico: `STX + comando + "|" + campos + "|" + checksum + ETX`. La respuesta se parsea por offsets fijos, no por un parser genérico. Cualquier cambio de firmware del POS puede romper el parseo; por eso el código histórico debe tratarse como referencia, no como especificación vigente.

### 5.4 Ventas y sincronización

- `0200` envía `200|precio|timestamp|1|0`.
- La respuesta incluye boleta, autorización, monto, tarjeta, operación y tipo (`DB`/`CR`).
- `Insertatrx` guarda la transacción con 12 parámetros.
- `SincronizaTabla` usa `DELETE`, `SqlBulkCopy` y `KeepIdentity`; las acciones son `TruncateDestino`/`TruncaOrigen`.
- `JobDeSincronizacion` copia productos y clientes remotos → local y auditoría local → remota.
- `JobDeCierreZ` ejecuta el cierre según el cron de la base.

### 5.5 Cómo estudiar Autosale sin compilarlo

1. Leer `librerias.cs` para el protocolo y la sincronización.
2. Leer `IngresoTarjeta.cs` para RFID.
3. Leer `Confirmacion.cs` para el formato de moneda.
4. Leer `TransBankGestion.cs` para el mapa de comandos.
5. Usar `docs/datasheet.md` para los códigos de resultado.
6. **No** intentar descifrar los archivos opacos: no son texto y no se necesita para entender la arquitectura.

### 5.6 Deuda técnica y riesgos

- Duplicación de `TransBank`.
- Credenciales SMTP en claro (rotar).
- Certificado temporal de firma (no usar).
- Acoplamiento UI-negocio.
- Sin pruebas.
- Dependencia de hardware y COM.
- Formularios no mantenibles sin el proyecto original.

---

## 6. Convenciones y estilo

### Python

- PEP 8, nombres en `snake_case`, clases en `PascalCase`.
- Funciones de servicio sin dependencia de `request` cuando sea posible.
- Errores de negocio con `ValueError`; la capa de rutas los traduce a HTTP 400.
- Fechas en UTC con `datetime.now(timezone.utc)`.
- No exponer `password_hash` ni `qr_token` en respuestas públicas.

### JavaScript

- Funciones cortas, `const`/`let`, `fetch` con `async/await`.
- Encabezados JSON y `X-Requested-With` en mutaciones.
- Reintento de SSE con backoff fijo de 3 s.
- No concatenar HTML sin escapar datos del servidor.

### SQL

- Nombres de tabla en singular (`turno`, `modulo`).
- Claves foráneas explícitas.
- Índice en `turno.estado` para las consultas de cola.
- Evitar `SELECT *` en código nuevo.

### Documentación

- Enlaces relativos dentro del repo.
- Ejemplos ejecutables, sin credenciales reales.
- Advertencias de seguridad visibles.
- Diagramas en SVG, no en capturas.

---

## 7. Despliegue y operación

### 7.1 turnero con Gunicorn

```bash
pip install gunicorn
gunicorn -w 1 -b 0.0.0.0:5000 "app:create_app()"
```

**Un worker** si se usa SSE en memoria. Con más workers, los eventos no se comparten. Para escalar:

- Un bus de eventos (Redis Pub/Sub) y un endpoint SSE que se suscriba.
- O separar el display en polling cada 2-3 s (menos elegante, más simple).

### 7.2 Proxy inverso

```nginx
location / {
    proxy_pass http://127.0.0.1:5000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_buffering off;          # imprescindible para SSE
    proxy_read_timeout 3600s;
}
```

`proxy_buffering off` evita que Nginx acumule los eventos. Sin esto, el display parece congelado.

### 7.3 systemd

```ini
[Unit]
Description=Turnero
After=network.target

[Service]
WorkingDirectory=/opt/turnero
Environment=SECRET_KEY=<clave>
ExecStart=/opt/turnero/venv/bin/gunicorn -w 1 -b 127.0.0.1:5000 "app:create_app()"
Restart=always
User=turnero

[Install]
WantedBy=multi-user.target
```

No commitear la unidad con la clave. Usar `EnvironmentFile` con permisos 600.

### 7.4 Respaldos

```bash
sqlite3 instance/turnero.db ".backup '/backup/turnero-$(date +%F).db'"
```

El comando `.backup` es seguro con la base abierta.

---

## 8. Seguridad

| Riesgo | Dónde | Mitigación |
|---|---|---|
| Clave de sesión predecible | `config.py` | Definir `SECRET_KEY` robusta |
| Credenciales demo | `seed.py` | Rotar/eliminar en producción |
| CSRF artesanal | `auth_service.py` | Token por sesión / Flask-WTF |
| XSS | Templates/JS | Escapar datos; evitar `innerHTML` con entrada |
| Fuga de RUT | API/base | Minimizar, restringir acceso, no loguear |
| Fuga de QR token | API pública | Usar `to_display_dict()` |
| Fuerza bruta de login | `/api/auth/login` | Rate limiting, bloqueo, MFA |
| SSE sin auth | `/api/turnos/eventos` | No incluir datos personales |
| SMTP sin TLS | `notificador.py` | `starttls` + login + validación |
| Credenciales legacy | `librerias.cs` | Rotar, externalizar, no publicar |
| Certificado temporal | `AutoSell_TemporaryKey.pfx` | No usar para firmar releases |

Checklist mínimo antes de exponer:

- [ ] `SECRET_KEY` única y fuera del repo.
- [ ] Usuarios demo deshabilitados.
- [ ] HTTPS en el proxy.
- [ ] Base y backups fuera del webroot.
- [ ] Logs sin RUT ni tokens.
- [ ] Rate limiting en login y solicitud de turnos.
- [ ] Revisión de dependencias (`pip audit`).
- [ ] SMTP con TLS si se usa notificador.

---

## 9. Rendimiento y escalabilidad

- **SQLite**: lecturas concurrentes aceptables; escrituras serializadas. Para >10 operadores activos o varias sucursales, PostgreSQL.
- **SSE**: una conexión por display. Con 10 displays, 10 colas; es trivial. El costo crece con el número de eventos y clientes.
- **Índices**: `turno.estado` acelera la cola. Agregar `(estado, fecha_creacion)` para oficinas grandes.
- **Código correlativo**: el `COUNT` recorre los turnos del tipo. Con millones de filas, crear una tabla de secuencias.
- **notificador**: lee una tabla completa; si el carrito tiene miles de filas, considerar paginación o un límite.
- **Autosale**: la sincronización por `SqlBulkCopy` es eficiente, pero `TRUNCATE` + copia completa puede ser pesado; evaluar copia incremental.

---

## 10. Preguntas de diseño frecuentes

**¿Por qué no hay una API común entre las versiones?**
Porque son proyectos de épocas y contextos distintos, agrupados con fines de portafolio. Forzar una API común habría requerido reescribir Autosale y añadir complejidad artificial.

**¿Por qué el QR no contiene el código?**
Para separar identificación visible de autorización. El código se puede mostrar en público; el token es una capacidad que solo debe estar en el ticket.

**¿Por qué no se usa JWT?**
Porque el cliente es un navegador con sesión y no hay consumidores externos. Una cookie firmada es suficiente y más simple. Si se abriera la API a terceros, JWT o claves de API serían razonables.

**¿Por qué SQLite y no PostgreSQL?**
Despliegue de una oficina, sin DBA. SQLite elimina instalación y backup complejo. El límite se documenta y se acepta.

**¿Por qué no se usa WebSocket?**
El flujo es unidireccional. SSE reconecta solo, funciona sobre HTTP/1.1 y es más fácil de operar detrás de proxies.

**¿Por qué el notificador no usa Flask?**
Porque no necesita HTTP ni UI. Un script con argparse es más fácil de programar, probar y ejecutar en cron.

---

## 11. Referencias internas

- `README.md`: visión, evolución, instalación y operación.
- `docs/datasheet.md`: especificaciones y solución de problemas.
- `docs/BITACORA.md`: decisiones, problemas y lecciones.
- `docs/REFERENCES.md`: enlaces oficiales.
- `versions/turnero/docs/developer.md`: documentación original extendida del turnero.
- `versions/notificador/docs/developer.md`: documentación original del notificador.
- `versions/*/VERSION.md`: alcance y sanitización de cada importación.

---

*Fin de la guía de desarrollador.*

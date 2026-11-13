# Datasheet · Sistema de autoatención

**Proyecto:** `autoatencion` — archivo de tres generaciones de software para autoservicio.
**Autor:** jmadasme.
**Licencia:** MIT (ver `LICENSE`).
**Versión documentada:** contenido de `versions/Autosale`, `versions/turnero` y `versions/notificador` tal como se conserva en este repositorio.

---

## 1. Resumen ejecutivo

Este repositorio reúne tres implementaciones que cubren momentos distintos de un mismo problema: atender a una persona sin que un operador haga todo el trabajo. La primera generación, **Autosale**, es un tótem de venta Windows con cobro Transbank, identificación RFID, impresión Epson y sincronización con SQL Server. La segunda, **turnero**, es una aplicación web Flask con tres interfaces —kiosco, ejecutivo y display— que gestiona una fila de atención con turnos, QR y eventos en tiempo real. La tercera, **notificador**, es una utilidad de línea de comandos que lee un carrito desde SQLite y lo envía por correo.

No es un producto único ni un instalador. Es un **conjunto documental y ejecutable por partes**: `turnero` y `notificador` pueden correr en un equipo con Python; `Autosale` requiere Windows, Visual Studio, SQL Server y hardware, y su código está parcialmente preservado en un formato que impide una compilación directa.

| Característica | Autosale | turnero | notificador |
|---|---|---|---|
| Año/generación | Primera | Segunda | Tercera |
| Plataforma | Windows · .NET Framework 4.5.2 | Linux/macOS/Windows · Python 3.10+ | Linux/macOS/Windows · Python 3.10+ |
| Interfaz | WinForms | Web responsive | CLI + correo |
| Base de datos | SQL Server | SQLite | SQLite (solo lectura) |
| Tiempo real | No | SSE | No |
| Autenticación | Login local + RFID | Sesión + bcrypt | No |
| Hardware | Pin pad, RFID, Epson, Cashdro | Cámara (opcional) | Ninguno |
| Estado | Histórico, no compilable tal cual | Ejecutable | Ejecutable con SMTP pendiente |

---

## 2. Especificaciones técnicas comparadas

### 2.1 Requisitos de sistema

| Recurso | Autosale | turnero | notificador |
|---|---|---|---|
| SO | Windows 7/8/10 | Linux, macOS o Windows | Linux, macOS o Windows |
| Runtime | .NET Framework 4.5.2 | Python 3.10 o superior | Python 3.10 o superior |
| Gestor de paquetes | NuGet + referencia COM | pip | pip (sin dependencias) |
| Base de datos | SQL Server Express o superior | SQLite 3 (incluida en Python) | SQLite 3 |
| Navegador | No aplica | Chrome, Edge, Firefox o Safari actuales | No aplica |
| Red | Local y remota | HTTP/SSE en red local | SMTP saliente |
| Almacenamiento | Medio (binarios + base) | Bajo (decenas de MB) | Muy bajo |
| Usuario | Administrador para instalar | Usuario sin privilegios | Usuario sin privilegios |

### 2.2 Puertos y protocolos

| Componente | Puerto / protocolo | Dirección | Notas |
|---|---|---|---|
| turnero HTTP | 5000/TCP | Entrada | Configurable con `PORT` |
| turnero SSE | 5000/TCP | Entrada | Mismo puerto, `text/event-stream` |
| notificador SMTP | 587/TCP | Salida | `smtp.gmail.com` por defecto |
| Autosale pin pad | COM6, 115200 8N1 | Serie | Configurable en la clase |
| Autosale RFID | COM7, 9600 8N1 | Serie | Configurable en el formulario |
| Autosale SQL Server | 1433/TCP típico | Salida | Instancia local/remota |
| Cashdro | 443/TCP HTTPS | Salida | `https://{IP}/Cashdro3WS` |

### 2.3 Volumen y retención

- `turnero` guarda un registro por turno, sin purga automática. En una oficina de 200 turnos diarios, la base crece unos pocos MB al año.
- `notificador` no escribe en la base; solo lee la tabla más reciente.
- `Autosale` conserva transacciones, clientes, productos y auditoría; el crecimiento depende del volumen del tótem.

---

## 3. Autosale · datasheet de usuario

### 3.1 Qué hace

Autosale es una aplicación de escritorio para un tótem de venta. El cliente se identifica con una tarjeta RFID de convenio, el sistema consulta su saldo, muestra el producto y el precio, confirma la operación y cobra con un pin pad Transbank. Al finalizar, imprime el comprobante y registra la transacción. Un proceso programado sincroniza productos, clientes y ventas con un servidor central.

### 3.2 Flujo de uso

1. **Autenticación**: el operador o el sistema inicia la aplicación.
2. **Identificación**: el cliente acerca su tarjeta al lector RFID (`COM7`).
3. **Consulta**: la aplicación obtiene el código de cliente y consulta saldo y productos.
4. **Confirmación**: se muestran producto, cliente, saldo actual, precio y saldo resultante.
5. **Cobro**: se envía la venta al pin pad (`COM6`) y se espera la autorización.
6. **Comprobante**: se imprime el voucher en la Epson.
7. **Registro**: se inserta la transacción con `Insertatrx`.
8. **Sincronización**: Quartz ejecuta la copia a la base central según el cron configurado.

### 3.3 Mandos y comandos del POS

La pantalla `TransBankGestion` permite enviar comandos manuales de soporte:

| Comando | Nombre | Uso |
|---|---|---|
| `0070` | Inicialización | Prepara el pin pad |
| `0080` | Respuesta de inicialización | Consulta el resultado |
| `0100` | Polling | Verifica estado |
| `0200` | Venta | Cobra el monto |
| `0250` | Última venta | Recupera la última operación |
| `0410` | PAN encriptado | Flujo de tokenización |
| `0500` | Cierre | Cierra el lote |
| `0800` | Carga de llaves | Actualiza llaves |
| `1200` | Anulación | Anula una venta |

### 3.4 Códigos de resultado del pin pad

| Código | Significado | Acción sugerida |
|---|---|---|
| `02` | Autorizador no responde | Reintentar; verificar red |
| `03` | Conexión fallida | Revisar enlace del POS |
| `04` | Transacción ya anulada | No repetir la anulación |
| `05` | No existe transacción | Verificar número de boleta |
| `06` | Tarjeta no soportada | Usar otra tarjeta |
| `07` | Operación cancelada | Informar al cliente |
| `08` | No se puede anular débito | Seguir procedimiento del adquirente |
| `09` | Error de lectura | Limpiar/reintentar la tarjeta |
| `10` | Monto bajo el mínimo | Ajustar monto |
| `11` | No existe venta | Verificar referencia |
| `12` | Operación no soportada | Usar otro medio |
| `13` | Debe realizar cierre | Ejecutar `0500` |
| `14` | Error de encriptado PAN | Reintentar o llamar a soporte |
| `15` | Error de débito | Reintentar |
| `80` | Confirme el monto | Mostrar monto al cliente |
| `81` | Ingrese clave | Solicitar PIN |
| `82` | Enviando | Esperar |
| `90` | Inicialización correcta | Continuar |
| `91` | Inicialización fallida | Revisar conexión |
| `92` | Lector no conectado | Verificar cable/puerto |

### 3.5 Configuración de hardware

| Elemento | Valor por defecto | Dónde se ajusta |
|---|---|---|
| Pin pad | `COM6`, 115200 8N1 | `TransBank.PuertoCOM` |
| RFID | `COM7`, 9600 8N1 | `IngresoTarjeta.PuertoCOM` |
| Impresora | COM Epson | Referencia COM del proyecto |
| Lector de barras | Puerto configurable | `funciones.PortLectorBarras` |
| Cashdro | IP/usuario/clave/timeout | Tabla de parámetros |
| SQL Server | `.\SQLEXPRESS`, `DB` | `funciones.cnLocal` |

### 3.6 Limitaciones conocidas

- Los formularios `Principal.cs`, `Principal.Designer.cs`, `IngresoTransbank.cs`, `mensajesPantalla.*` y `Test_Int_POS.cs` están en un formato opaco/binario. No son editables como C# y el proyecto **no compila tal cual**.
- La UI y la lógica están acopladas; no hay pruebas automatizadas.
- Hay credenciales SMTP legacy embebidas y un certificado temporal de firma. No deben reutilizarse.
- Depende de hardware y de licencias de terceros.
- No hay una versión web ni una API HTTP.

---

## 4. turnero · datasheet de usuario

### 4.1 Qué hace

Gestiona una fila de atención con tres vistas:

- **Display** (`/`): pantalla pública con el turno actual de cada módulo.
- **Kiosco** (`/kiosco`): solicitud de turno con RUT y QR.
- **Ejecutivo** (`/ejecutivo`): login, llamado, inicio y cierre de atención.

### 4.2 Instalación

Requisitos: Python 3.10+, pip y un navegador moderno.

```bash
cd versions/turnero

python3 -m venv venv
source venv/bin/activate        # Linux/Mac
venv\Scripts\activate           # Windows

pip install -r requirements.txt
```

Dependencias declaradas en `versions/turnero/requirements.txt`:

| Paquete | Versión | Función |
|---|---|---|
| Flask | 3.1.* | Servidor y rutas |
| Flask-SQLAlchemy | 3.1.* | ORM y SQLite |
| bcrypt | 4.* | Hash de contraseñas |

### 4.3 Arranque y URLs

```bash
python app.py
```

| URL | Vista | Acceso |
|---|---|---|
| `http://localhost:5000/` | Display | Público |
| `http://localhost:5000/kiosco` | Kiosco | Público |
| `http://localhost:5000/login` | Login | Público |
| `http://localhost:5000/ejecutivo` | Panel | Requiere sesión |

La base se crea en `instance/turnero.db`. Si no hay ejecutivos, el arranque ejecuta el seed con tres tipos de trámite, tres módulos y tres usuarios demo. Las credenciales demo están en `seed.py`; **no usarlas en producción**.

### 4.4 Operación por pantalla

#### Display

- Consulta inicial a `GET /api/turnos/display`.
- Se suscribe a `GET /api/turnos/eventos` (SSE).
- Muestra un panel por módulo activo con el ejecutivo y el turno actual.
- Estados visibles: "Llamando", "En atención", "Sin turno".
- Overlay "Sin turnos en espera" cuando no hay actividad.
- Si se corta la conexión, reintenta a los 3 segundos.

#### Kiosco

- Paso 1: selección de tipo de trámite.
- Paso 2: RUT con formateo automático y validación de dígito verificador.
- Paso 3: ticket con código, trámite y QR.
- El QR contiene `qr_token`, no el código visible.
- Si la API de trámites falla, usa una lista de respaldo.

#### Ejecutivo

- Login con usuario y contraseña.
- Vista "sin turno": botón para llamar al siguiente.
- Vista "llamado": datos del turno y acción para iniciar.
- Vista "en atención": cronómetro, cierre con tipo y comentarios.
- Escaneo de QR con la cámara trasera.
- Cola refrescada cada 15 s; estado cada 10 s.
- Logout explícito.

### 4.5 Campos del turno

| Campo | Tipo | Visible en display | Descripción |
|---|---|---|---|
| `codigo` | Texto | Sí | Correlativo por tipo |
| `tipo_tramite` | Referencia | Sí | Trámite solicitado |
| `rut_cliente` | Texto | No | Dato personal |
| `estado` | Enum | Sí | esperando/llamado/en_atencion/completado |
| `modulo` | Referencia | Sí | Módulo asignado |
| `ejecutivo` | Referencia | Sí | Operador asignado |
| `fecha_creacion` | Fecha UTC | No | Marca de solicitud |
| `fecha_llamado` | Fecha UTC | No | Marca de llamado |
| `fecha_inicio_atencion` | Fecha UTC | No | Inicio de atención |
| `fecha_completado` | Fecha UTC | No | Cierre |
| `tipo_atencion` | Texto | No | Clasificación del cierre |
| `comentarios` | Texto | No | Observaciones |
| `qr_token` | Texto | No | Capacidad para iniciar atención |

### 4.6 Mensajes y errores frecuentes

| Situación | Respuesta | Causa probable |
|---|---|---|
| RUT inválido | `400` con mensaje | Dígito verificador incorrecto |
| Trámite inexistente | `400` | ID inválido |
| Credenciales incorrectas | `401` | Usuario/clave erróneos o inactivo |
| Sin sesión | `401` JSON o redirect | Sesión expirada |
| CSRF rechazado | `403` | Falta encabezado o origen |
| Ya tiene turno activo | `400` | Llamó dos veces sin cerrar |
| QR no corresponde | `400` | Token inválido o turno de otro ejecutivo |
| Sin turnos | `200` con `null` | Cola vacía |
| Error interno | `500` | Revisar log del servidor |

### 4.7 Límites

- Una sola instancia/worker para SSE en memoria.
- Sin prioridades, sucursales ni reprogramación.
- Sin impresión térmica ni audio.
- Sin informes históricos.
- Sin autenticación en display ni en kiosco.
- Sin purga automática de datos.

---

## 5. notificador · datasheet de usuario

### 5.1 Qué hace

Lee la tabla `carrito_AAAAMMDD_HHMMSS` más reciente de una base SQLite, arma un correo de texto plano con el detalle y lo envía por SMTP. No modifica la base.

### 5.2 Instalación

```bash
cd versions/notificador

python3 -m venv venv
source venv/bin/activate        # Linux/Mac
venv\Scripts\activate           # Windows

pip install -r requirements.txt
```

`requirements.txt` no declara dependencias externas; solo comentarios que indican Python 3.10+. El entorno virtual se recomienda igual para aislar futuras dependencias.

### 5.3 Uso

```bash
python notificador.py --help
python notificador.py --dry-run
python notificador.py --db-path /ruta/kiosco.db
python notificador.py --db-path ./kiosco.db --destino admin@ejemplo.cl
```

| Opción | Efecto |
|---|---|
| `--dry-run` | No envía; registra el correo en el log |
| `--db-path RUTA` | Base SQLite; relativa al directorio del script |
| `--destino CORREO` | Destinatario; si se omite, usa el valor por defecto |
| `--help` | Ayuda |

### 5.4 Formato del correo

```text
De: notificador@kiosco.local
Asunto: Notificación de carrito - carrito_20260115_143000

Producto                Cantidad   Precio unit.   Subtotal
----------------------------------------------------------
Café                        2          1.500       3.000
Sandwich                    1          3.200       3.200
----------------------------------------------------------
TOTAL                                              6.200
```

- Texto plano, sin HTML ni adjuntos.
- Subtotal calculado si falta o es cero.
- Filas con cantidad o precio nulo se omiten con warning.
- Total redondeado a dos decimales.

### 5.5 Logs y salida

- `notificador.log` en el directorio de ejecución.
- Consola en INFO; archivo en DEBUG.
- Código 0: éxito (incluye dry-run).
- Código 1: base inexistente, error de lectura o fallo de envío.

### 5.6 Limitaciones

- El envío real no usa TLS ni autenticación; es un esqueleto.
- No hay reintentos ni cola.
- Un solo destinatario por ejecución.
- No lee el esquema del turnero; espera tablas `carrito_*`.
- No valida direcciones de correo.

---

## 6. Dependencias

### 6.1 Dependencias de turnero

Archivo: `versions/turnero/requirements.txt`.

| Librería | Versión declarada | Rol |
|---|---|---|
| Flask | 3.1.* | Framework web |
| Flask-SQLAlchemy | 3.1.* | Integración ORM |
| bcrypt | 4.* | Hash de contraseñas |

Instalación dentro del entorno virtual del proyecto:

```bash
cd versions/turnero
python3 -m venv venv
source venv/bin/activate        # Linux/Mac
venv\Scripts\activate           # Windows
pip install -r requirements.txt
```

### 6.2 Dependencias de notificador

Archivo: `versions/notificador/requirements.txt`. No declara paquetes externos; el script usa solo la biblioteca estándar (`sqlite3`, `smtplib`, `email`, `logging`, `argparse`, `re`, `pathlib`). Aun así, el flujo recomendado es:

```bash
cd versions/notificador
python3 -m venv venv
source venv/bin/activate        # Linux/Mac
venv\Scripts\activate           # Windows
pip install -r requirements.txt
```

### 6.3 Dependencias de Autosale

Se gestionan con NuGet y referencias COM, no con `requirements.txt`:

| Dependencia | Versión | Tipo |
|---|---|---|
| Common.Logging | 3.0.0 | NuGet |
| Common.Logging.Core | 3.0.0 | NuGet |
| Newtonsoft.Json | 10.0.1 | NuGet |
| Quartz | 2.3.3 | NuGet |
| EpsonFPHostControlX | 1.8 | COM |
| .NET Framework | 4.5.2 | Runtime |

### 6.4 Dependencias de frontend (turnero)

Se cargan por CDN y no se instalan con pip:

| Recurso | Versión | Uso |
|---|---|---|
| Bootstrap | 5.3.3 | Estilos y layout |
| qrcodejs | 1.0.0 | Generación de QR |
| html5-qrcode | 2.3.8 | Lectura de QR con cámara |

Para un despliegue sin internet, descargar estos archivos y servirlos localmente.

---

## 7. Guía de solución de problemas

### 7.1 turnero no arranca

| Síntoma | Verificación | Solución |
|---|---|---|
| `ModuleNotFoundError` | `pip list` dentro del venv | Activar el venv e instalar `requirements.txt` |
| `Address already in use` | `ss -ltnp \| grep 5000` | Cambiar `PORT` o detener el proceso |
| Base bloqueada | Permisos y procesos abiertos | Cerrar instancias y revisar permisos |
| Página sin estilos | Consola del navegador | Verificar acceso a CDN o servir localmente |
| QR no se genera | Consola del navegador | Verificar carga de qrcodejs |
| Cámara no abre | Permisos del navegador | Usar HTTPS o `localhost`; permitir cámara |
| SSE no actualiza | Pestaña Red | Verificar proxy sin buffering; `X-Accel-Buffering: no` |
| Login siempre falla | Seed y hash | Verificar usuario activo y contraseña demo |
| `SECRET_KEY` warning | Variables de entorno | Definir una clave robusta |

### 7.2 notificador falla

| Síntoma | Causa | Solución |
|---|---|---|
| `No such file or directory` | Ruta de base incorrecta | Usar `--db-path` absoluto |
| "No se encontraron tablas" | Base sin tablas `carrito_*` | Verificar la base y la convención |
| "No hay tablas de carrito válidas" | Nombres fuera de formato | Revisar el proceso que crea tablas |
| Correo no enviado | SMTP sin TLS/login | Completar configuración o usar relay local |
| Timeout | Red/firewall | Verificar salida al puerto 587 |
| Log vacío | Directorio sin permiso | Ejecutar en un directorio escribible |

### 7.3 Autosale no compila

| Síntoma | Causa | Solución |
|---|---|---|
| Errores de sintaxis en formularios | Archivos opacos | No es un proyecto compilable tal cual |
| Falta referencia COM | Epson no registrada | Instalar el driver/control o quitar la referencia |
| No conecta a SQL Server | Instancia/cadena | Ajustar `cnLocal` y verificar SQLEXPRESS |
| Pin pad no responde | Puerto/velocidad | Verificar `COM6`, 115200 8N1 y cable |
| RFID no lee | Puerto/velocidad | Verificar `COM7`, 9600 8N1 |
| No imprime | Driver/COM | Verificar la impresora y la referencia Epson |

### 7.4 Diagnóstico rápido de turnero

```bash
# ¿Está escuchando?
curl -i http://localhost:5000/api/tramites

# ¿El display responde?
curl -i http://localhost:5000/api/turnos/display

# ¿El SSE emite heartbeat? (cortar con Ctrl+C)
curl -N http://localhost:5000/api/turnos/eventos
```

---

## 8. Mantenimiento

### 8.1 Respaldo

- **turnero**: copiar `instance/turnero.db` con el servicio detenido o usando la API de backup de SQLite.
- **notificador**: no requiere respaldo propio; la base es de otro sistema.
- **Autosale**: respaldar SQL Server y la configuración de hardware.

### 8.2 Actualización

1. Detener el servicio.
2. Respaldar la base.
3. Actualizar el código.
4. Instalar dependencias (`pip install -r requirements.txt`).
5. Arrancar y verificar el flujo completo.
6. Revisar logs.

### 8.3 Rotación de datos

No hay purga automática. Para turnero, archivar turnos `completado` con más de N días en una tabla histórica o exportarlos antes de borrarlos. Para notificador, rotar `notificador.log`.

---

## 9. Advertencias de seguridad

- Cambiar `SECRET_KEY` y las credenciales demo antes de exponer turnero.
- No publicar `instance/turnero.db` ni backups: contienen RUT y datos de atención.
- No reutilizar las credenciales SMTP legacy de Autosale ni el certificado `AutoSell_TemporaryKey.pfx`.
- El notificador no usa TLS; no enviar información sensible hasta completarlo.
- El display y el kiosco son públicos por diseño; no agregar datos personales a sus respuestas.
- Aplicar HTTPS mediante proxy inverso si el sistema sale de la red local.
- Revisar las licencias de Transbank, Epson, Cashdro y demás componentes de terceros.

---

## 10. Glosario

| Término | Definición |
|---|---|
| Autoatención | Sistema que permite al usuario completar un trámite o compra sin operador |
| Tótem | Equipo físico con pantalla y periféricos para autoservicio |
| Pin pad | Terminal de pago que procesa la tarjeta |
| POS | Point of Sale; en este contexto, el terminal de pago |
| RFID | Identificación por radiofrecuencia usada para leer la tarjeta de convenio |
| SSE | Server-Sent Events; canal unidireccional servidor → navegador |
| Kiosco | Interfaz de solicitud de turno |
| Display | Pantalla pública de llamados |
| Seed | Datos iniciales de demostración |
| QR token | Valor aleatorio impreso en el QR que autoriza iniciar la atención |

---

*Documento de usuario del repositorio. Para detalles de implementación, ver `developer.md`. Para enlaces oficiales, ver `REFERENCES.md`.*

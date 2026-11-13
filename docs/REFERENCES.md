# Referencias

**Proyecto:** autoatención · tres generaciones.
**Uso:** enlaces de consulta para mantenimiento, extensión y auditoría.
**Nota:** las URLs pueden cambiar con el tiempo. Se indica la documentación oficial siempre que existe; para temas históricos (protocolo serial Transbank), se incluyen referencias generales y se advierte que la especificación vigente debe obtenerse del adquirente.

---

## 1. Documentación interna

| Documento | Ruta | Contenido |
|---|---|---|
| README raíz | `README.md` | Evolución, mapa de versiones, arquitectura, instalación, operación, límites y licencia |
| Datasheet | `docs/datasheet.md` | Especificaciones, operación por pantalla, campos, códigos de error, dependencias y troubleshooting |
| Guía de desarrollador | `docs/developer.md` | Arquitectura interna, API, modelo de datos, extensiones, pruebas y seguridad |
| Bitácora | `docs/BITACORA.md` | Decisiones, problemas, enfoques descartados y lecciones aprendidas |
| Diagramas | `docs/assets/` | SVG de evolución, arquitecturas y flujo de estados |
| Autosale · alcance | `versions/Autosale/VERSION.md` | Origen, sanitización y limitaciones de la primera versión |
| turnero · README | `versions/turnero/README.md` | Documentación original del turnero |
| turnero · datasheet | `versions/turnero/docs/datasheet.md` | Especificaciones originales del turnero |
| turnero · developer | `versions/turnero/docs/developer.md` | Guía original extendida del turnero |
| notificador · README | `versions/notificador/README.md` | Documentación original del notificador |
| notificador · datasheet | `versions/notificador/docs/datasheet.md` | Especificaciones originales del notificador |
| notificador · developer | `versions/notificador/docs/developer.md` | Guía original del notificador |

---

## 2. turnero · stack web

### Flask

- Documentación oficial: <https://flask.palletsprojects.com/>
- Application factory: <https://flask.palletsprojects.com/en/stable/patterns/appfactories/>
- Blueprints: <https://flask.palletsprojects.com/en/stable/blueprints/>
- Sesiones: <https://flask.palletsprojects.com/en/stable/quickstart/#sessions>
- Manejo de errores: <https://flask.palletsprojects.com/en/stable/errorhandling/>
- Despliegue: <https://flask.palletsprojects.com/en/stable/deploying/>
- Streaming y SSE: <https://flask.palletsprojects.com/en/stable/patterns/streaming/>

### Flask-SQLAlchemy y SQLAlchemy

- Flask-SQLAlchemy: <https://flask-sqlalchemy.palletsprojects.com/>
- SQLAlchemy 2.0: <https://docs.sqlalchemy.org/en/20/>
- Tipos de columnas: <https://docs.sqlalchemy.org/en/20/core/type_basics.html>
- Relaciones: <https://docs.sqlalchemy.org/en/20/orm/basic_relationships.html>
- Migraciones con Alembic: <https://alembic.sqlalchemy.org/>

### SQLite

- Documentación: <https://sqlite.org/docs.html>
- Backup en línea: <https://sqlite.org/backup.html>
- WAL: <https://sqlite.org/wal.html>
- Limitaciones de concurrencia: <https://sqlite.org/lockingv3.html>

### bcrypt

- Paquete en PyPI: <https://pypi.org/project/bcrypt/>
- Repositorio: <https://github.com/pyca/bcrypt>

### SSE y HTTP

- MDN · Server-Sent Events: <https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events>
- MDN · EventSource: <https://developer.mozilla.org/en-US/docs/Web/API/EventSource>
- HTML Living Standard · server-sent events: <https://html.spec.whatwg.org/multipage/server-sent-events.html>
- MDN · Fetch API: <https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API>
- MDN · CORS: <https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS>

### Frontend

- Bootstrap 5.3: <https://getbootstrap.com/docs/5.3/>
- qrcodejs: <https://github.com/davidshimjs/qrcodejs>
- html5-qrcode: <https://github.com/mebjas/html5-qrcode>
- MDN · Cámara y `getUserMedia`: <https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia>

### Seguridad web

- OWASP · CSRF: <https://owasp.org/www-community/attacks/csrf>
- OWASP · XSS: <https://owasp.org/www-community/attacks/xss/>
- OWASP · Authentication Cheat Sheet: <https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html>
- OWASP · Session Management: <https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html>
- Flask-WTF (CSRF): <https://flask-wtf.readthedocs.io/en/stable/form.html#csrf-protection>

### Servidores y operación

- Gunicorn: <https://docs.gunicorn.org/en/stable/>
- Nginx · proxy_pass: <https://nginx.org/en/docs/http/ngx_http_proxy_module.html>
- Nginx · buffering: <https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffering>
- systemd · timers: <https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html>
- systemd · service: <https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html>
- cron · manual: <https://man7.org/linux/man-pages/man5/crontab.5.html>

---

## 3. notificador · Python estándar

- `sqlite3`: <https://docs.python.org/3/library/sqlite3.html>
- `smtplib`: <https://docs.python.org/3/library/smtplib.html>
- `email`: <https://docs.python.org/3/library/email.html>
- `email.message.EmailMessage`: <https://docs.python.org/3/library/email.message.html>
- `logging`: <https://docs.python.org/3/library/logging.html>
- `argparse`: <https://docs.python.org/3/library/argparse.html>
- `pathlib`: <https://docs.python.org/3/library/pathlib.html>
- `re`: <https://docs.python.org/3/library/re.html>
- `venv`: <https://docs.python.org/3/library/venv.html>
- Tutorial de SQLite en Python: <https://docs.python.org/3/library/sqlite3.html#sqlite3-tutorial>

### SMTP y correo

- Gmail · configuración SMTP: <https://support.google.com/mail/answer/7126229>
- Gmail · contraseñas de aplicación: <https://support.google.com/accounts/answer/185833>
- Python · ejemplo de TLS: <https://docs.python.org/3/library/smtplib.html#smtplib.SMTP.starttls>
- RFC 5321 (SMTP): <https://www.rfc-editor.org/rfc/rfc5321>
- RFC 5322 (formato de mensajes): <https://www.rfc-editor.org/rfc/rfc5322>

---

## 4. Autosale · .NET y escritorio

### Plataforma

- .NET Framework 4.5.2: <https://learn.microsoft.com/en-us/dotnet/framework/whats-new/>
- Guía de migración a .NET moderno: <https://learn.microsoft.com/en-us/dotnet/core/porting/>
- Windows Forms: <https://learn.microsoft.com/en-us/dotnet/desktop/winforms/>
- SerialPort: <https://learn.microsoft.com/en-us/dotnet/api/system.io.ports.serialport>
- SqlBulkCopy: <https://learn.microsoft.com/en-us/dotnet/api/system.data.sqlclient.sqlbulkcopy>
- SQL Server Express: <https://learn.microsoft.com/en-us/sql/sql-server/>

### Dependencias

- Quartz.NET: <https://www.quartz-scheduler.net/>
- Quartz.NET · tutorial: <https://www.quartz-scheduler.net/documentation/quartz-3.x/tutorial/index.html>
- Newtonsoft.Json: <https://www.newtonsoft.com/json>
- Common.Logging: <https://github.com/net-commons/common-logging>
- NuGet: <https://learn.microsoft.com/en-us/nuget/>

### Hardware y pagos

- Transbank · desarrolladores: <https://www.transbankdevelopers.cl/>
- Transbank · documentación POS: <https://www.transbankdevelopers.cl/documentacion/pos>
- Epson · soporte: <https://support.epson.net/>
- Epson · controladores y manuales: <https://epson.com/Support/sl/s251>
- Cashdro: <https://www.cashdro.com/>
- ISO/IEC 7816 (tarjetas con chip): <https://www.iso.org/standard/70484.html>
- ISO/IEC 14443 (RFID/NFC): <https://www.iso.org/standard/73541.html>

> **Advertencia:** el protocolo serial de Transbank cambia según el modelo de pin pad y el contrato. El código de Autosale es histórico y no reemplaza la documentación vigente del adquirente. No usar en producción sin certificación.

---

## 5. Diseño, accesibilidad y documentación

- W3C · WCAG 2.1 Quick Reference: <https://www.w3.org/WAI/WCAG21/quickref/>
- W3C · ARIA: <https://www.w3.org/WAI/ARIA/>
- MDN · accesibilidad: <https://developer.mozilla.org/en-US/docs/Web/Accessibility>
- Material Design · color: <https://m3.material.io/styles/color/overview>
- Coolors (paletas): <https://coolors.co/>
- W3C · SVG: <https://www.w3.org/TR/SVG2/>
- MDN · SVG: <https://developer.mozilla.org/en-US/docs/Web/SVG>

### Paleta del proyecto

| Color | Hex | Uso |
|---|---|---|
| Azul noche | `#06141b` | Fondo principal |
| Verde azulado | `#2A9D8F` | Acento, estados activos |
| Azul medio | `#457B9D` | Secundario, enlaces |
| Azul grisáceo | `#5B7C99` | Bordes, textos secundarios |

La paleta busca un tono sobrio, sin verdes neón, adecuado para paneles de operación y documentación técnica.

---

## 6. Licencia y propiedad intelectual

- MIT License: <https://opensource.org/licenses/MIT>
- SPDX · MIT: <https://spdx.org/licenses/MIT.html>
- Guía de licencias de software: <https://choosealicense.com/>
- Transbank, Cashdro, Epson, Microsoft, Flask y demás marcas pertenecen a sus titulares. Este repositorio no está afiliado a ellas.

---

## 7. Cómo citar este repositorio

```text
jmadasme. (2026). Autoatención: tres generaciones de un sistema de atención
y venta [Repositorio de código]. Licencia MIT.
```

Para las versiones individuales, citar la carpeta y su `VERSION.md`:

```text
jmadasme. Autosale (primera generación) [Código fuente]. Importado en autoatencion/versions/Autosale.
jmadasme. turnero (segunda generación) [Código fuente]. Importado en autoatencion/versions/turnero.
jmadasme. notificador (tercera generación) [Código fuente]. Importado en autoatencion/versions/notificador.
```

---

## 8. Referencias históricas y contexto

- Historia de las interfaces gráficas en Windows: <https://learn.microsoft.com/en-us/windows/apps/design/>
- Evolución de .NET: <https://learn.microsoft.com/en-us/dotnet/core/whats-new/>
- Comparativa SQLite/PostgreSQL: <https://www.sqlite.org/whentouse.html>
- Server-Sent Events vs WebSockets: <https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events>
- Patrón application factory: <https://flask.palletsprojects.com/en/stable/patterns/appfactories/>
- Patrón repositorio (relacionado): <https://martinfowler.com/eaaCatalog/repository.html>

---

*Si un enlace deja de funcionar, buscar el título del recurso en el sitio oficial correspondiente. Las referencias se verificaron al momento de generar esta documentación; los sitios de terceros pueden reorganizar sus rutas.*

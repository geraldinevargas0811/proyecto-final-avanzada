# Proyecto: Transaction Engine — Resumen rápido

Propósito
- Servicio backend que recibe transacciones (HTTP y TCP), las valida, procesa por tipo y las persiste en BD.
- Arquitectura simple, extensible y preparada para hilos y sockets.

Estructura (paquetes top-level en `src/`)
- model — POJOs / DTOs (Transaction, PaymentTransaction, RefundTransaction).
- processor — Procesadores por tipo y `ProcessorFactory`.
- validator — Validaciones (ej. `BasicTransactionValidator`).
- repository — Persistencia (interface `TransactionRepository`, `JdbcTransactionRepository`).
- service — Orquestador `TransactionService` y excepciones.
- web — Servlets / Listener (Jakarta): `TransactionServlet`, `AppStartupListener`.
- concurrent — Gestión de hilos: `ThreadPoolManager`, `TransactionWorker`, `WorkQueue`.
- network — Sockets: `SocketServer`, `SocketClientHandler`, `SocketProtocol`.
- util — Utilidades (IdGenerator, JsonUtils).

Puntos clave de funcionamiento
- Flujo HTTP: cliente → POST /transactions → `TransactionServlet` → `TransactionService` → `Processor` → `TransactionRepository` (save).
- Flujo TCP: cliente → TCP puerto 9000 → `SocketServer` → `SocketClientHandler` → (pool) → `TransactionService`.
- `AppStartupListener` inicializa: processors, repository (lee `db.properties`), `TransactionService`, `ThreadPoolManager` y `SocketServer`. Guarda `TransactionService` en `ServletContext` con clave `txService`.

Configuración base de datos
- Archivo plantilla: `web/WEB-INF/db.properties` o `WEB-INF/classes/db.properties`.
  - Propiedades: `jdbc.url`, `jdbc.user`, `jdbc.password`, `jdbc.driver`.
- Script para crear tabla: `sql/schema.sql`.
- Driver JDBC: añadir JAR en `web/WEB-INF/lib/` (mysql/postgres).

Buenas prácticas y extensibilidad
- Para añadir un nuevo tipo: crear `TransactionProcessor` para el tipo y registrarlo (idealmente en `AppStartupListener` vía `ProcessorFactory.register(...)`). No cambiar `TransactionService`.
- Concurrencia: cada worker abre su propia `Connection` — no compartir `Connection` entre hilos.
- Producción: usar DataSource/JNDI o un pool (HikariCP) en lugar de `DriverManager`.

Cómo probar rápido
- HTTP (form-data):
  curl -X POST -F "type=PAYMENT" -F "amount=10.00" -F "payload=test" http://localhost:8080/<app-context>/transactions
- TCP (manual):
  nc localhost 9000
  enviar línea JSON: {"type":"PAYMENT","amount":12.5,"payload":"texto"}

Notas técnicas
- `ProcessorFactory` usa `ConcurrentHashMap` (thread-safe).
- `db.properties` puede leerse desde `/WEB-INF/db.properties` por `AppStartupListener` o desde classpath.

Dónde mirar primero (archivos importantes)
- `web/AppStartupListener.java` — inicialización general.
- `service/TransactionService.java` — orquestación.
- `repository/JdbcTransactionRepository.java` — acceso a BD.
- `network/SocketServer.java` — servidor TCP.
- `sql/schema.sql` — DDL.
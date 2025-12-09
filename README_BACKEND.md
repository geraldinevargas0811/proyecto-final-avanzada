# Resumen breve del backend (qué se hizo y responsabilidades)

Este documento explica de forma concisa qué hace el backend y qué responsabilidades tienen los paquetes y las clases más importantes. Úsalo para que el equipo (front / DB) entienda cómo integrarse.

Estructura de paquetes (top-level)
- model
- processor
- validator
- repository
- service
- web
- util
- concurrent
- network

Descripción por paquete y clases clave

1. model/
- Transaction (interface): contrato mínimo de una transacción (id, type, amount, payload).
- PaymentTransaction, RefundTransaction: implementaciones concretas de Transaction.

2. processor/
- TransactionProcessor (interface): define process(Transaction) y supportedType().
- PaymentProcessor, RefundProcessor: lógica por tipo de transacción.
- ProcessorFactory: registro/registro thread-safe (ConcurrentHashMap). Para añadir nuevos tipos no se modifica la lógica existente (Open/Closed).

3. validator/
- TransactionValidator (interface)
- BasicTransactionValidator: validaciones básicas (id no nulo, amount > 0, tipo presente).

4. repository/
- TransactionRepository (interface): abstracción para persistencia.
- JdbcTransactionRepository: implementación JDBC que lee configuración desde properties y guarda transacciones en la tabla `transactions`.
- JdbcTransactionRepositoryJndi (opcional): alternativa que obtiene DataSource vía JNDI.

Archivos relacionados con BD:
- sql/schema.sql: script DDL para crear base de datos y tabla `transactions`.
- db.properties (ubicación esperada: web/WEB-INF/db.properties o WEB-INF/classes/db.properties): contiene jdbc.url, jdbc.user, jdbc.password, jdbc.driver.
- driver JDBC: colocar el JAR en web/WEB-INF/lib/.

5. service/
- TransactionService: orquesta el flujo: validar → obtener TransactionProcessor → procesar → persistir. Depende de abstracciones (validator, repository).
- Excepciones: ValidationException, ProcessingException, RepositoryException.

6. web/
- AppStartupListener (ServletContextListener, Jakarta): inicializa la aplicación al desplegar:
  - registra processors en ProcessorFactory,
  - construye JdbcTransactionRepository (lee db.properties),
  - crea TransactionService,
  - inicializa ThreadPoolManager y SocketServer,
  - guarda objetos en ServletContext (clave `txService`).
- TransactionServlet (Jakarta servlet): endpoint HTTP POST `/transactions` que recibe parámetros (type, amount, payload), crea la Transaction correspondiente y llama a TransactionService.handle(tx).

7. util/
- IdGenerator: genera IDs (UUID).
- JsonUtils: parsea JSON a objetos Transaction usando Gson (usado por socket handler).

8. concurrent/
- ThreadPoolManager: encapsula ExecutorService para ejecutar worknners de forma segura.
- TransactionWorker: Runnable que procesa transacciones (consume de una cola opcional).
- WorkQueue: wrapper para BlockingQueue (si se usa cola para desacoplar recepción y procesamiento).

9. network/
- SocketServer: servidor TCP que acepta conexiones y delega cada cliente a SocketClientHandler (usa pool o hilo por conexión). Puerto por defecto 9000.
- SocketClientHandler: lee líneas (cada línea JSON representa una transaction), parsea con JsonUtils y delega el procesamiento (síncrono o vía pool).
- SocketProtocol: mensajes mínimos de respuesta (OK / ERROR).

Comportamiento y puntos importantes
- Flujo principal HTTP: cliente → TransactionServlet → TransactionService → Processor → Repository (persistencia).
- Flujo por socket: cliente TCP (línea JSON) → SocketServer → SocketClientHandler → (pool) → TransactionService → Repository.
- Concurrencia: cada worker abre su propia Connection; JdbcTransactionRepository no comparte Connection entre hilos. ProcessorFactory usa ConcurrentHashMap para ser thread-safe.
- Configuración DB: colocar db.properties en WEB-INF (o WEB-INF/classes) y el driver en WEB-INF/lib. AppStartupListener intenta leer /WEB-INF/db.properties y luego classpath si no lo encuentra.
- Extensibilidad (SOLID):
  - Añadir un nuevo tipo de transacción: crear clase en model (si hace falta), crear TransactionProcessor, registrar en ProcessorFactory (idealmente desde AppStartupListener). No cambiar TransactionService.
  - Liskov: cualquier TransactionProcessor cumple el mismo contrato process(...) y puede sustituir a otro.
  - Dependency Inversion: TransactionService depende de interfaces (validator, repository).

Qué deben editar/añadir otros equipos
- Front: consumir POST /transactions. Formularios y AJAX deben enviar `type`, `amount`, `payload`. (No tocar lógica del service/repository).
- DB: ejecutar `sql/schema.sql`, editar `web/WEB-INF/db.properties` con credenciales reales y añadir el JAR del driver en `web/WEB-INF/lib/`. Pueden optar por JNDI: si se usa, avisar para que actualicemos AppStartupListener o usemos JdbcTransactionRepositoryJndi.

Pruebas rápidas
- Probar HTTP (desde máquina de desarrollo):
  curl -X POST -F "type=PAYMENT" -F "amount=10.00" -F "payload=test" http://localhost:8080/<app>/transactions
- Probar socket (manual):
  nc localhost 9000
  enviar una línea JSON: {"type":"PAYMENT","amount":12.5,"payload":"test"}

Logs y errores
- Revisar logs al iniciar AppStartupListener (inicialización de repo/pool/socket).
- Errores comunes: db.properties no encontrado, driver JDBC no en classpath, puerto 9000 ocupado.

Contacto / notas finales
- El backend está preparado para que el equipo Front use el servlet y para que el equipo DB configure la conexión. Para cambios en contratos (JSON vs form-data, endpoints adicionales como GET /transactions) coordinar con quien haga backend logic (se puede añadir endpoints sin romper lo ya existente).

```
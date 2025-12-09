# Assignment — Tareas claras y puntuales

A continuación las tareas para cada responsable. Mantener esto breve y accionable.

A) Front-end (JSP / páginas web)
Objetivo: Construir la interfaz que use el endpoint existente.

Archivos a crear/editar
- `web/transactions.jsp` — formulario de envío.
- `web/WEB-INF/jsp/result.jsp` — vista resultado (opcional).
- Recursos estáticos: `web/assets/css/`, `web/assets/js/`.

Qué implementar (pasos)
1. Crear formulario que haga POST a `/transactions` con campos:
   - type (PAYMENT | REFUND)
   - amount
   - payload
2. Validación cliente: amount > 0, type no vacío.
3. Manejar respuestas HTTP:
   - 200 → mostrar éxito (mensaje con id si lo hay).
   - 400/500 → mostrar error descriptivo.
4. Probar con curl y navegador:
   - curl -X POST -F "type=PAYMENT" -F "amount=10.00" -F "payload=test" http://localhost:8080/<app>/transactions

Entregables
- `web/transactions.jsp`, assets mínimos y README corto de cómo probar.

B) Base de datos (DB)
Objetivo: Dejar la BD y configuración listos para que la app persista transacciones.

Archivos a crear/editar
- Ejecutar `sql/schema.sql` en el servidor de BD.
- Editar `web/WEB-INF/db.properties` con credenciales reales.
- Añadir driver JDBC en `web/WEB-INF/lib/` (ej. `mysql-connector-java.jar`).

Qué implementar (pasos)
1. Ejecutar SQL:
   - MySQL: `mysql -u root -p < sql/schema.sql`
   - PostgreSQL: `psql -U postgres -f sql/schema.sql`
2. Crear usuario y permisos (ejemplo MySQL):
   ```
   CREATE USER 'tx_user'@'localhost' IDENTIFIED BY 'tx_password';
   GRANT INSERT, SELECT ON transactions_db.* TO 'tx_user'@'localhost';
   FLUSH PRIVILEGES;
   ```
3. Editar `web/WEB-INF/db.properties`:
   - jdbc.url=
   - jdbc.user=
   - jdbc.password=
   - jdbc.driver=
4. Copiar el JAR del driver a `web/WEB-INF/lib/`.
5. Verificar conexión (probar `DbTest` o enviar POST al servlet).

Entregables
- BD creada con tabla `transactions` y usuario con permisos.
- `web/WEB-INF/db.properties` rellenado (no subir credenciales a VCS).
- Informar si desean usar JNDI/DataSource (entregaremos ajuste en `AppStartupListener`).

Checklist común antes de entregar
- [ ] POST /transactions funciona y crea fila en `transactions`.
- [ ] Driver JDBC en `WEB-INF/lib`.
- [ ] `db.properties` configurado y accesible por la app.
- [ ] Front probó envíos básicos (curl o formulario).

Comunicación y cambios
- Cualquier cambio en contrato (ej. enviar JSON o añadir GET /transactions) debe coordinarse antes de implementar.
- Si DB eligió JNDI, avisar para ajustar inicialización (usaremos `JdbcTransactionRepositoryJndi`).

Cualquier cosita me avisan, tratare de estar lo mas pendiente posible. jajaj
De todos modos esto es una guia, tambien pueden hacerlo como les parezca conveniente.
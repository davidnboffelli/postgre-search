## Levantar una Imagen de Postgres con Docker Compose

Para realizar las pruebas de todo lo investigado, levanté una imagen de PostgreSQL con el siguiente `docker-compose.yaml`:

```yaml
version: "1"

services:
  postgres:
    image: postgres
    ports:
      - '8081:80'
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=1234
      - POSTGRES_USER=david
volumes:
  pgdata:
```

---

## Creación, Modificación y Eliminación de Bases de Datos y Roles de Usuario

Hay dos formas de crear bases de datos y roles.

### Primera Forma: Desde Dentro de la Base de Datos

#### Bases de Datos

Ejemplos de creación, edición y eliminación de bases de datos:

```sql
CREATE DATABASE nuevadb; -- Crea una base de datos con el nombre "nuevadb"
CREATE DATABASE nuevadbtemp WITH TEMPLATE template1; -- Crea una base de datos con el nombre nuevadbtemp, utilizando de base el template "template1". Un template es una plantilla que tiene una estructura de base de datos definida
CREATE DATABASE nuevadb WITH OWNER usuario; -- Crea una base de datos con el nombre nuevadb y asigna al usuario "usuario" como dueño de la misma
CREATE DATABASE nuevadblimit WITH CONNECTION_LIMIT 10; -- Crea una base de datos de nombre "nuevadblimit" y le pone un límite de 10 conexiones al pull de conexiones
ALTER DATABASE nuevadblimit RENAME TO ndbl; -- Edita la base de datos "nuevadblimit" y le cambia el nombre a "ndbl"
ALTER DATABASE ndbl WITH CONNECTION_LIMIT 15; -- Edita la base de datos "ndbl" y le cambia el límite de conexiones del pull de conexiones a 15
DROP DATABASE ndbl; -- Elimina la base de datos "ndbl". Solo puede hacerlo el dueño y no estando conectado a la misma
```

#### Roles/Usuarios

Ejemplos de creación, edición y eliminación de roles/usuarios:

```sql
CREATE USER otrouser WITH PASSWORD '1234'; -- Crea un usuario de nombre "otrouser" y le asigna un password
GRANT ALL PRIVILEGES ON DATABASE nuevadb TO otrouser; -- Al usuario "otrouser" le asigna todos los privilegios y permisos sobre la base de datos "nuevadb"
ALTER ROLE otrouser WITH PASSWORD '22'; -- Edita el usuario "otrouser" para asignarle un password
REVOKE ALL PRIVILEGES ON DATABASE nuevadb FROM otrouser; -- Revoca todos los privilegios y permisos del usuario "otrouser" sobre la base de datos "nuevadb"
DROP ROLE otrouser; -- Elimina al usuario "otrouser"

CREATE USER readaccess WITH PASSWORD 'jw8s0F4'; -- Crea el usuario "readaccess" con un password
GRANT CONNECT ON DATABASE library TO readaccess; -- Le da permiso para loguearse en la base de datos "library"
GRANT pg_read_all_data TO readaccess; -- Le asigna un rol predefinido que le da permisos de lectura

CREATE USER writeUser WITH PASSWORD 'jw8s0F4'; -- Crea el usuario "writeUser" con un password
GRANT CONNECT ON DATABASE library TO writeUser; -- Le da permiso para loguearse a la base de datos "library"
GRANT pg_write_all_data TO writeUser; -- Le asigna permisos de escritura al usuario "writeUser", vinculándolo a un rol preexistente
```

### Segunda Forma: A Través de Funciones

Ejemplos de creación, edición y eliminación de bases de datos:

```sh
createdb -U david tablatest # Crea la base de datos "tablatest" y asigna al usuario "david" como dueño
createdb -U david -T template1 tablatest2 # Crea la base de datos "tablatest2" con el template "template1" y asigna al usuario "david" como dueño
createdb -U david -c 10 tablatest3 # Crea la base de datos "tablatest3", con un límite de conexiones de 10 y asigna al usuario "david" como dueño
dropdb -U david tablatest # Elimina la base de datos "tablatest"
```

Ejemplos de creación, edición y eliminación de roles/usuarios:

```sh
createuser -U david --interactive todopostgre # Crea el usuario "todopostgre" en modo interactivo. Es decir, que a través de prompts consulta sobre la asignación de permisos.
dropuser -U david todopostgre; # Elimina al usuario "todopostgre"
```

Desde dentro de la base de datos:

```sh
\l # Para listar las bases de datos
\du # Para listar los roles
```

Referencia: [todopostgresql.com](https://www.todopostgresql.com/como-crear-base-de-datos-en-postgresql/)

---

## Copias de Seguridad y Restauraciones con `pg_dump` y `pg_restore`

### Crear una Copia de Seguridad

```sh
pg_dump -U david -d david -F t # Para crear un backup de la base de datos de nombre "david" en formato tar (comprimido)
```

### Verificar una Copia de Seguridad

```sh
pg_restore -l tmp/backup.tar # Para verificar el backup ubicado en "tmp/backup.tar"
```

---

## Optimización de Consultas y Uso de `EXPLAIN`

### Forma 1: Índices. 

En caso de que una tabla se consulte de forma recurrente y específicamente ciertos campos de la misma.

```sql
CREATE INDEX t_sales_date_idx ON t_sales (date); -- Crea el índice "t_sales_date_idx" sobre la tabla "t_sales" y el campo "date"
```

### Forma 2: Comando `EXPLAIN`. 

Devuelve el plan de ejecución de la consulta sobre la que se aplica, indicando los costos y el recorrido que hace para obtener el resultado

```sql
EXPLAIN SELECT * FROM t_sales s INNER JOIN t_sales_details sd ON s.number = sd.number WHERE s.number = '000000000001';
```

### Forma 3: Uso Correcto del Operador `LIKE`
Si se desea buscar los clientes con el apellido "Boffelli".
Evitar:

```sql
SELECT * FROM o_customers WHERE name LIKE 'Boffelli';
```

Usar:

```sql
SELECT * FROM o_customers WHERE name LIKE '%Boffelli';
```

### Forma 4: Uso de Límites.

En casos donde la consulta devuelve muchos registros, muchas veces no es necesario obtener TODOS para resolver lo requerido en la consulta por lo que se recomienda asignar límites a través de filtros en un WHERE o del uso del comando LIMIT

```sql
SELECT * FROM t_sales ORDER BY date DESC LIMIT 100; -- Trae las últimas 100 facturas realizadas
```

### Forma 5: Uso del Tipo Apropiado de Datos

Uso del tipo apropiado de tipo de datos, por ejemplo un smallint en caso de cargar una edad ya que ocupa mucho menos que un int tradicional.

### Forma 6: Evitar el Uso Excesivo de Subqueries

Evitar el uso excesivo de subqueries. Muchas veces una subquery se puede resolver con un join, los cuales usualmente son más óptimos.

Evitar:

```sql
SELECT * FROM t_sales WHERE customer = (SELECT id FROM o_customers WHERE name = 'Marco Ruben');
```

Usar:

```sql
SELECT * FROM t_sales s INNER JOIN o_customers c ON s.customer = c.id WHERE c.name = 'Marco Ruben';
```

### Forma 7: Uso de Prepared Statements

A través del uso de prepared statements. En casos donde la misma consulta se use repetidamente en el tiempo, esta se puede guardar a modo de plantilla y solo pasarle los parámetros que se desee cambiar

Evitar:

```sql
SELECT * FROM t_sales s INNER JOIN t_sales_details sd ON s.number = sd.number WHERE s.number = '000000000001';
SELECT * FROM t_sales s INNER JOIN t_sales_details sd ON s.number = sd.number WHERE s.number = '000000000002';
SELECT * FROM t_sales s INNER JOIN t_sales_details sd ON s.number = sd.number WHERE s.number = '000000000003';
```

Usar:

```sql
PREPARE get_document_data(number) AS SELECT * FROM t_sales s INNER JOIN t_sales_details sd ON s.number = sd.number WHERE s.number = $1;
EXECUTE get_document_data(000000000001);
EXECUTE get_document_data(000000000002);
EXECUTE get_document_data(000000000003);
```

### Forma 8: Uso de Pools de Conexiones

Gestionando pulls de conexiones se evita la sobrecarga generada por conectarse y desconectarse repetidamente a la base de datos. En lugar de esto, permite el reuso de conexiones

### Forma 9: Uso de `ANALYZE` y `VACUUM`

#### `ANALYZE`

Postgresql guarda estadísticas sobre queries realizadas e índices creados en la tabla pg_statistic. Si esta información no está actualizada al día, puede generar ineficientes planes de ejecución de queries. El comando ANALIZE permite actualizar esta información.

```sql
ANALYZE t_sales; -- Actualiza las estadísticas de consultas e índices realizados sobre la tabla "t_sales"
```

#### `VACUUM`

Cuando insertas, actualizas o eliminas registros en una tabla, postgresql no libera inmediatamente el espacio en disco. Lo marca como "reusable" o "espacio muerto" y estos "fragmentos" de espacio se pueden ir acumulando en el tiempo. VACUUM es una forma de "desfragmentar" el espacio de una tabla.

```sql
VACUUM t_sales; -- "Desfragmenta" el espacio muerto en la tabla "t_sales"
```

Referencia: [NodeTeam](https://nodeteam.medium.com/how-to-optimize-postgresql-queries-226e6ff15f72)

---

## Configuración de Herramientas de Monitoreo como `pg_stat_statements`

Para configurar `pg_stat_statements` hay que seguir varios pasos:

1. Instalar el paquete `postgresql-contrib`:

   ```sh
   sudo apt install postgresql-contrib
   ```

2. Editar el archivo ubicado en `/var/lib/pgsql/data/postgresql.conf`. Particularmente, el campo `shared_preload_libraries` hay que descomentarlo y agregarle el valor `pg_stat_statements`:

   ```conf
   shared_preload_libraries = 'pgaudit,pg_stat_statements'
   ```

3. Reiniciar PostgreSQL. En mi caso, reinicié el contenedor de Docker.

4. Conectarse a la base de datos y crear la extensión en la misma:

   ```sql
   CREATE EXTENSION pg_stat_statements;
   ```

### Verificar su Funcionamiento

Ejecutar la consulta:

```sql
SELECT * FROM pg_stat_statements;
```



### Ejemplo de Uso

```sql
SELECT substring(query, 1, 50) AS short_query,
       round(total_exec_time::numeric, 2) AS total_exec_time,
       calls,
       round(mean_exec_time::numeric, 2) AS mean,
       round((100 * total_exec_time /
       sum(total_exec_time::numeric) OVER ())::numeric, 2) AS percentage_cpu
FROM pg_stat_statements
ORDER BY total_exec_time DESC;
```

Devuelve las consultas que se ejecutan en la base de datos, ordenadas por tiempo de ejecución en forma descendente. Nos ayuda a identificar tiempos de procesamiento excesivos.

Referencia: [A2 Systems](https://a2systems.co/blog/blog-2/usando-pg-stat-statements-167)

---

## Transacciones en PostgreSQL y Manejo de Bloqueos (Deadlocks)

### Transacciones

Las transacciones permiten ejecutar un bloque de código SQL en su totalidad y aplicar los cambios correspondientes en la base de datos una vez finalizadas con éxito. Comienzan con `BEGIN` y finalizan con `COMMIT` o `ROLLBACK`.

Ejemplo de transacción:

```sql
BEGIN;
CREATE TABLE t_sales (
    number VARCHAR(12) PRIMARY KEY,
    external_number VARCHAR(12),
    class CHAR(1),
    date TIMESTAMP,
    total_price FLOAT,
    tax FLOAT
);

INSERT INTO t_sales (number, external_number, class, date, total_price, tax) VALUES
('000000000001', 'EXT000001', 'A', '2024-07-16 10:00:00', 100.50, 10.05),
('000000000002', 'EXT000002', 'B', '2024-07-16 11:00:00', 200.75, 20.07),
('000000000003', 'EXT000003', 'A', '2024-07-16 12:00:00', 150.20, 15.02),
('000000000004', 'EXT000004', 'C', '2024-07-16 13:00:00', 300.90, 30.09),
('000000000005', 'EXT000005', 'B', '2024-07-16 14:00:00', 250.60, 25.06);

SELECT * FROM t_sales;

COMMIT;
```

### Niveles de Aislamiento

1. **Read Committed**: Nivel por defecto. Permite a otras transacciones leer información modificada por la presente transacción incluso si todavía estos cambios no fueron commiteados.
2. **Repeatable Read**: No permite a otras transacciones leer los cambios no commiteados.
3. **Serializable**: Nivel más restrictivo. No permite a otras queries editar ni eliminar sobre esas tablas. Tampoco permite a otras queries ver los cambios realizados por la transacción serializable hasta que esta los comitee.

Referencias:
- [Medium - Transactions in PostgreSQL](https://medium.com/yavar/transactions-in-postgresql-a90b09faa80c)
- [Medium - SQL Transactions in PostgreSQL](https://medium.com/@anton.martyniuk/getting-started-with-sql-transactions-in-postgresql-ca6285175d62)
- [PostgreSQL - Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [YouTube - Transactions in PostgreSQL](https://www.youtube.com/watch?v=G8wDjV0N9tk)
- [The Nile Blog](https://www.thenile.dev/blog/transaction-isolation-postgres)

### Deadlocks

Un "deadlock" es un estado que ocurre cuando una transacción espera a que otra transacción libere un recurso, pero a su vez la segunda transacción se encuentra esperando a que la primera libere otro recurso. Ambas se encuentran esperando recursos bloqueados por la otra, por lo que se produce un bloqueo indefinido.

Referencias:
- [PostgreSQL - Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [LinkedIn - Resolving Deadlocks](https://www.linkedin.com/pulse/deadlock-resolving-deadlocks-skip-locked-postgresql-sarvaha-systems)
- [Cybertec - Understanding Deadlocks](https://www.cybertec-postgresql.com/en/postgresql-understanding-deadlocks/)

---

## Configuración de Autenticación y Autorización en PostgreSQL

Para gestionar la autenticación en PostgreSQL, se debe editar el archivo ubicado en `/var/lib/postgresql/data/pg_hba.conf`. Tiene un formato similar al siguiente:

```conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD

local   all             all                                   trust
```

- **TYPE**: Refiere a la forma en la que se accede a la base de datos. Ej.: local, ssh, host, etc.
- **DATABASE** y **USER**: Sobre cuál base de datos y cuál usuario se va a aplicar esta regla.
- **ADDRESS**: La dirección de la máquina del usuario que va a entrar a la base de datos.
- **METHOD**: El método de autenticación que se le exige a ese usuario. Ej.: `trust`, `reject`, `password`.

Referencia: [Medium - PostgreSQL Authentication](https://medium.com/@s.k.thakur.contact/postgresql-authorization-and-authentication-5f1ddb2c4a59)
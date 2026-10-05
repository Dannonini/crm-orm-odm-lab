# CRM ORM/ODM Lab

API REST de un CRM básico que combina un ORM (Sequelize + PostgreSQL) y un ODM (Mongoose + MongoDB).

## Stack

- Node.js 22, Express 5, CommonJS
- Sequelize + PostgreSQL 16 (`User`, `Company`, `Contact`)
- Mongoose + MongoDB 7 (`Activity`)
- Jest + Supertest
- GitHub Codespaces, Dev Containers, Docker Compose
- Supervisor (`npm run dev`)

## Arquitectura

```text
GitHub Codespace
│
├── app       Node.js 22  ──┬── Sequelize ──> postgres (PostgreSQL)
│                           └── Mongoose  ──> mongo    (MongoDB)
├── postgres
└── mongo
```

La aplicación se conecta por nombre de servicio (`postgres`, `mongo`). Las credenciales de desarrollo llegan como variables de entorno definidas en `.devcontainer/docker-compose.yml` (ver `.env.example`).

## Iniciar el Codespace

1. En GitHub: **Code → Codespaces → Create codespace on main**.
2. Espera a que se levanten los tres servicios (`app`, `postgres`, `mongo`). `postCreateCommand` ejecuta `npm install`.

## Instalar dependencias

```bash
npm install
```

## Seed y reset

```bash
npm run seed    # inserta datos deterministas (3 users, 4 companies, 8 contacts, 10 activities)
npm run reset   # elimina y recrea tablas/base de datos y vuelve a sembrar
```

## Iniciar la API

```bash
npm start       # node ./bin/www
npm run dev     # supervisor ./bin/www
```

Servidor en el puerto `3000` (variable `PORT`).

## Pruebas

```bash
npm test
```

Cada suite restablece PostgreSQL y MongoDB antes de ejecutarse y cierra las conexiones al terminar.

## Endpoints

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/health` | Health check |
| GET | `/users` | Listar usuarios |
| GET | `/users/:id` | Obtener usuario |
| POST | `/users` | Crear usuario |
| PUT | `/users/:id` | Actualizar usuario |
| DELETE | `/users/:id` | Eliminar usuario |
| GET | `/companies` | Listar compañías (`?industry=`) |
| GET | `/companies/:id` | Obtener compañía |
| POST | `/companies` | Crear compañía |
| PUT | `/companies/:id` | Actualizar compañía |
| DELETE | `/companies/:id` | Eliminar compañía |
| GET | `/contacts` | Listar contactos |
| GET | `/contacts/:id` | Obtener contacto |
| POST | `/contacts` | Crear contacto |
| PUT | `/contacts/:id` | Actualizar contacto |
| DELETE | `/contacts/:id` | Eliminar contacto |
| GET | `/activities` | Listar actividades (`?type=`) |
| GET | `/activities/:id` | Obtener actividad |
| POST | `/activities` | Crear actividad |
| PUT | `/activities/:id` | Actualizar actividad |
| DELETE | `/activities/:id` | Eliminar actividad |

Los errores se devuelven como JSON: `{ "error": "Contact not found" }`.

## Respuestas

**1. Dos motores.**
Activity es buena para una base documental porque su campo `metadata` cambia según el tipo y no todas las actividades tienen los mismos datos. Company y Contact son buenas para una base relacional porque están ligadas entre sí, o sea cada contacto pertenece a una compañía, y eso se controla con una llave foránea.

**2. ORM vs ODM.**
Un ORM conecta objetos de JavaScript con tablas de una base de datos relacional, y un ODM los conecta con documentos de una base documental. En este proyecto el ORM es Sequelize y el ODM es Mongoose. Una diferencia importante es que el ORM trabaja con tablas de estructura fija y el ODM con documentos que pueden ser más flexibles.

**3. Configuración por variables de entorno.**
Las credenciales se definen en `.devcontainer/docker-compose.yml` y llegan a la app como variables de entorno. Escribirlas en los archivos js es mala práctica porque se subirían a GitHub y cualquiera podría verlas, además de que habría que cambiar el código para modificarlas. La app usa los hosts `postgres` (DB_HOST) y `mongo` (MONGODB_URI) y no localhost, porque cada base de datos corre en su propio contenedor y se conectan por el nombre del servicio.

**4. Asociaciones.**
En `models/sequelize/index.js`, Company y Contact tienen una relación de uno a muchos: `Company.hasMany(Contact)`. La llave foránea es `companyId` y vive en la tabla de Contact. El alias `as: 'contacts'` es el nombre con el que se piden los contactos de una compañía y es la propiedad que aparece en la respuesta.

**5. Eager loading.**
Con dos consultas se hace un viaje a la base de datos para la compañía y otro para sus contactos. Con `include` lo traen juntos en una sola consulta. Es preferible `include` porque hace menos viajes a la base y el código queda más corto.

**6. Instancia vs consulta.**
En `update` de `controllers/contacts.js` primero se busca el contacto con `findByPk`, lo que permite responder 404 si no existe y regresar el contacto ya actualizado. `Model.update({...}, { where })` es una sola consulta y es más directa, pero no devuelve el registro actualizado, solo información de cuántas filas se modificaron.

**7. Esquema flexible.**
En `models/mongoose/activity.js`, `metadata` es de tipo `mongoose.Schema.Types.Mixed`, que acepta cualquier objeto. Por eso puede guardar `duration` y `result` para un CALL, `subject` y `opened` para un EMAIL, o `location` y `attendees` para un MEETING. La desventaja es que Mongoose no valida los campos ni sus tipos, así que se pueden guardar datos mal escritos o inconsistentes.

**8. Sin ref.**
`ref` y `populate` solo funcionan entre modelos de Mongoose dentro de MongoDB, y User y Contact están en PostgreSQL, por eso `contactId` y `userId` son simples números (`Number`). La consecuencia es que no hay integridad entre las dos bases: si se elimina un User en PostgreSQL, sus actividades siguen en MongoDB con un `userId` que ya no existe.

**9. Documento actualizado.**
Antes de mi corrección, `findByIdAndUpdate` en `update` de `controllers/activities.js` regresaba el documento como estaba antes del cambio, porque ese es su comportamiento por defecto. Le agregué la opción `new: true` para que regrese el documento ya actualizado y `runValidators: true` para que valide los datos con el esquema.

**10. Pruebas de comportamiento.**
Probar el comportamiento permite que cualquier solución correcta pase, sin importar si se usó `findAll`, `find` u otro método. Además, si cambio la implementación más adelante, las pruebas siguen sirviendo porque solo revisan lo que responde la API.

**11. Repetibilidad.**
En `tests/setup.js`, antes de cada suite (`beforeAll`) se conecta a PostgreSQL y a MongoDB y se ejecuta `reset()`, que restablece los datos del seed. Después (`afterAll`) se cierran las dos conexiones. Es necesario para que cada suite empiece siempre con los mismos datos; si no, lo que una prueba crea o modifica afectaría a las siguientes y `npm test` daría resultados distintos.

**12. Tu experiencia.**
El reto más difícil fue el 05 porque no sabía cómo traer los contactos junto con la compañía. Revisé el index.js, vi que el alias era contacts y lo usé en el include. Un mensaje de Jest que me ayudó fue el del reto 01, donde decía que esperaba 8 contactos y recibía 0, y eso me mostró que getAll devolvía un arreglo vacío.

## Evidencia
<img width="758" height="362" alt="imagen" src="https://github.com/user-attachments/assets/7d0029e3-7e96-4e06-9636-c803562459cc" />


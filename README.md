# Catálogo de Cursos · Next.js + Firebase

Aplicación web de catálogo e inscripción a cursos, **desplegada en producción** con un pipeline de CI/CD y
endurecida en seguridad de punta a punta: headers, autorización en el backend y reglas de base de datos con tests.

**Demo:** https://practica-ci-cd.web.app

Trabajo práctico de Programación Avanzada 2026 · Licenciatura en Sistemas de Información (FCyT, UADER)
**Autora:** Melina Mas Duarte

**Stack:** Next.js (export estático) · TypeScript · Firebase Hosting · Firebase Authentication ·
Firebase Data Connect (PostgreSQL / Cloud SQL) · Cloud Firestore · GitHub Actions

---

## Qué demuestra

| Área | Qué hice |
|---|---|
| **CI/CD** | Cada Pull Request se despliega en una URL de preview temporal; cada merge a `main` despliega a producción. Typecheck en el pipeline, `concurrency` para no pisar deploys y trazabilidad de rama y commit en el footer. |
| **Headers de seguridad** | Calificación de [securityheaders.com](https://securityheaders.com) de **D a A+**. CSP sin `'unsafe-inline'` (hashes `sha256` calculados en el pipeline), HSTS, `X-Frame-Options`, `Permissions-Policy`. Un *gate* del pipeline falla el PR si falta algún header. |
| **Caché** | HTML con `no-cache` (los deploys y rollbacks se ven al instante) y assets con hash con un año `immutable`. |
| **Autenticación y autorización** | Login con email y contraseña. En PostgreSQL, cada operación declara quién puede ejecutarla; el alumno siempre sale del token del usuario y nunca de una variable, lo que evita accesos a datos ajenos (IDOR). |
| **Modelo relacional** | Relación muchos-a-muchos `Alumno` ↔ `Curso` con clave primaria compuesta (no hay inscripciones duplicadas) e inscripción idempotente. |
| **Auditoría de seguridad** | Script `npm run atacar` que se comporta como un atacante con la configuración pública de la app y verifica que cada operación esté protegida. |
| **Firestore en tiempo real** | Reseñas de cursos que se actualizan en vivo entre navegadores, con resumen (promedio y cantidad) por agregación en el servidor. |
| **Reglas con tests** | Reglas de Firestore con lista blanca de campos, tipos y rangos, y **8 tests automatizados** contra el emulador. |

## Arquitectura

```
 Navegador (Next.js estático, servido por Firebase Hosting con CSP y caché)
   │
   ├── Firebase Auth ───────────────▶ token con el uid y el email del usuario
   │
   ├── Data Connect (conector "cursos")
   │     solo se envía el NOMBRE de la operación + variables, nunca SQL
   │     @auth decide quién ejecuta qué
   │        ▼
   │     PostgreSQL (Cloud SQL): Curso · Alumno · Matricula
   │
   └── Cloud Firestore (reseñas en tiempo real)
         firestore.rules valida dueño, campos, tipos y rangos
```

## Funcionalidades

- **Catálogo** de cursos leído desde PostgreSQL, con filtro.
- **Registro e inicio de sesión** con email y contraseña.
- **Inscribirme / darme de baja** de un curso; **Mis inscripciones** muestra solo las del usuario logueado.
- **Reseñas** con puntaje de 1 a 5 que se ven en tiempo real.

## Seguridad: decisiones clave

- **Mínimo privilegio en el backend.** La operación heredada `ListAlumnos` devolvía los emails de todos los
  alumnos a cualquiera sin login: la pasé a `NO_ACCESS`, porque ningún cliente la necesita.
- **Sin IDOR.** `Inscribirme`, `MisInscripciones` y `DarmeDeBaja` toman el alumno de `auth.uid`; aunque alguien
  conozca el id de otro usuario, no puede leer ni modificar sus datos.
- **El servidor decide los datos sensibles.** La fecha de inscripción y la de cada reseña las completa el servidor;
  el cliente no puede falsificarlas.
- **`connect-src` exacto.** La CSP permite solo los tres orígenes de Google que usa la app, en lugar de
  `*.googleapis.com`, para no dejar salidas a un script inyectado.

El detalle de cada decisión, con las mediciones antes y después, está en [`REFLEXION.md`](REFLEXION.md).

## Cómo ejecutarlo en local

Requisitos: Node.js 24 y un proyecto de Firebase (plan Spark).

```bash
npm install
npm run emulators     # terminal 1 · Auth, Data Connect y Firestore locales
npm run seed          # terminal 2 · datos iniciales
npm run dev           # app en http://localhost:3000
```

| Comando | Para qué |
|---|---|
| `npm run atacar` | Auditoría de seguridad de las operaciones contra los emuladores |
| `npm run test:reglas` | Tests de las reglas de Firestore |
| `npm run typecheck` | Chequeo de tipos de TypeScript |
| `node scripts/check-headers.mjs <URL>` | Verifica headers de seguridad y caché de una URL |

## Estructura

```
app/                  páginas (catálogo, mis inscripciones) y componentes
lib/                  acceso a Firebase, Data Connect y Firestore
dataconnect/          esquema PostgreSQL y operaciones con @auth
firestore.rules       reglas de seguridad de Firestore
tests/                tests de las reglas
scripts/              auditoría, verificación de headers, hashes de la CSP, seed
.github/workflows/    pipelines de preview y producción
docs/                 enunciados y guía original de la práctica
```

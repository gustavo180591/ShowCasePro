# Plan de Implementación — ShowCasePro
Catálogo online + Mini CRM + Dashboard básico (SvelteKit 2, Svelte 5, TypeScript, Prisma, PostgreSQL, Docker, Tailwind)

> Objetivo: dejar el proyecto corriendo en minutos y con un camino claro hasta producción. Sin humo.

---

## 0) Prerrequisitos
- Node.js 20+
- Docker y Docker Compose (opcional pero recomendado)
- Cuenta en: Vercel / Fly.io / Railway (elegí una)
- PostgreSQL local o gestionada
- Git + GitHub/GitLab/Bitbucket

---

## 1) Clonar y preparar el entorno
**Paso a paso**
1. Clonar el repo.
2. `cp .env.example .env` y completar valores: `DATABASE_URL`, `ADMIN_PASSWORD`, `AUTH_SECRET`, `VITE_WHATSAPP_PHONE`, `VITE_INSTAGRAM_URL`, `SMTP_*` si usarás email.
3. Instalar dependencias: `npm i`
4. Generar cliente Prisma: `npx prisma generate`
5. Crear/migrar DB: `npx prisma migrate dev`
6. (Opcional) Cargar datos demo: `npm run db:seed`

**Con Docker**
```bash
cp .env.example .env
docker compose up --build
```
Esto levanta **PostgreSQL** + **app** y aplica migraciones al iniciar.

---

## 2) Estructura clave
- `prisma/schema.prisma`: modelos (User, Customer, Tag, CustomerTag, Contact, Interaction, Product, Category)
- `src/lib/db`: cliente Prisma (con singleton)
- `src/routes`: páginas públicas (landing, catálogo, producto, contacto) y privadas (`/admin`, dashboard)
- `src/lib/validators`: Zod schemas (contacto, cliente, interacción)
- `src/lib/security`: rate-limit y sanitización básica
- `static/`: assets (OG image, favicon)
- `tests/`: Vitest (unidad) + Playwright (E2E)
- `docker-compose.yml`, `Dockerfile`, `.env.example`, `README.md`

---

## 3) Puesta en marcha local
- **Sin Docker**:
  ```bash
  npm run dev
  # App en http://localhost:5173
  ```
- **Con Docker**:
  ```bash
  docker compose up --build
  # App en http://localhost:5173
  ```

> Tip: Si cambia el esquema, corré `npx prisma migrate dev` y/o `npx prisma generate`.

---

## 4) Catálogo Online (Front público)
**Rutas y funcionalidades**
- `/` — Landing con propuesta de valor + CTA al catálogo.
- `/products` — Listado con búsqueda por texto y filtro por categoría/tags.
- `/products/[slug]` — Detalle con imágenes, precio, descripción y **CTA “Consultar por WhatsApp”** (deep link con mensaje prellenado) + enlace a Instagram.
- `/contact` — Formulario con validación cliente/servidor y estados (loading/éxito/error). Guarda en DB (`Contact`). Email opcional a admin (Nodemailer + SMTP).

**Checklist de calidad**
- [ ] SEO básico (title/description/OG)
- [ ] Accesibilidad (labels, aria, focus states)
- [ ] Performance (imágenes optimizadas, Lighthouse > 90)
- [ ] Sanitización de inputs

---

## 5) Mini CRM (panel `/admin`)
**Acceso**
- Login con contraseña (`ADMIN_PASSWORD`) y cookie firmada (`AUTH_SECRET`).

**Módulos**
- **Clientes** (`/admin/customers`)
  - Crear, ver, editar, eliminar.
  - Tags múltiples (N:N) por cliente.
  - Filtros por tag/canal y orden por fecha.
  - Ver contactos asociados (leads web).
- **Detalle de Cliente** (`/admin/customers/[id]`)
  - Asignar/crear tags al vuelo.
  - Registrar **interacciones** (nota + canal + fecha).
- **Exportaciones**
  - `/admin/export/customers.csv`
  - `/admin/export/contacts.csv`

**Checklist**
- [ ] Validación Zod en todas las actions
- [ ] Manejo de errores y toasts/estados en UI
- [ ] Rate-limit en acciones sensibles si se expone públicamente

---

## 6) Dashboard básico (`/dashboard`)
**KPIs**
- Nuevos clientes por período (últimos 30 días)
- Total de contactos
- Conversión estimada (% clientes / contactos)
**Gráficas**
- Contactos por canal (barras)
- Top tags (top 5)

> Mantenerlo liviano: SVG o librería mínima; nada pesado.

---

## 7) Seguridad y buenas prácticas
- TypeScript en todo (endpoints, actions, utils).
- Zod para validar *antes* de tocar DB o enviar email.
- Rate-limit por IP en formularios públicos (contacto).
- Sanitización de cadenas (evitar `<script>` y HTML no deseado).
- Cookies `httpOnly`, `secure`, `sameSite=lax`.
- **CSRF**: si `/admin` es público, agregar token anti-CSRF.
- **Headers**: `Content-Security-Policy`, `X-Frame-Options`, `Referrer-Policy` vía hooks.
- Logs de errores server-side (sin datos sensibles).

---

## 8) Tests
**Unit (Vitest)**
- Utils (`currency`, helpers)
- Validadores Zod (contact/customer/interaction)

**E2E (Playwright)**
- Flujo: login admin → alta de cliente → filtrar por tag/canal → export CSV
- Descarga y verificación de CSV
- Smoke público: visitar `/products`, buscar, entrar a detalle, enviar contacto (mock de SMTP)

Comandos:
```bash
npm run test       # unit
npm run e2e        # e2e
```

---

## 9) Observabilidad y salud
- Endpoint `/health` devuelve `{ ok: true, ts }` (para checks de plataforma).
- Logs de errores y métricas simples (requests por minuto, 4xx/5xx).

---

## 10) Deploy (elige tu aventura)
### A) Vercel
1. Crear proyecto y conectar repo.
2. Variables de entorno en Vercel (copiar `.env` sin secretos innecesarios).
3. Base de datos gestionada (Neon/Planetscale PG/ Railway PG).
4. `npx prisma migrate deploy` como paso de build o *post-deploy*.
5. Configurar `OUTPUT` SSR (SvelteKit adaptador node/edge según preferencia).

### B) Fly.io
1. `fly launch` (app + Postgres gestionado o externo).
2. Setear `DATABASE_URL` y secretos (`ADMIN_PASSWORD`, `AUTH_SECRET`, `SMTP_*`).
3. `fly deploy` y `npx prisma migrate deploy` (release command).
4. Health checks → `/health`.

### C) Railway
1. Crear servicio Web + Postgres.
2. Definir variables de entorno.
3. Añadir *Deploy command* que ejecute migraciones en cada release.
4. Probar `/health` y flujo de contacto.

> En todos los casos: **no** comitear `.env`. Usar store de secretos de la plataforma.

---

## 11) Operación diaria
- Revisar **Dashboard** a primera hora.
- Exportar CSV de **clientes** y **contactos** semanalmente (backup/analítica).
- Registrar interacciones importantes (llamadas, WhatsApp, IG).
- Mantener tags limpias y significativas (menos es más).

---

## 12) Roadmap sugerido (rápidas victorias)
- Importación CSV de clientes y tags.
- Webhooks (enviar nuevas consultas a Google Sheets/Discord/Slack).
- Variantes/atributos en productos (tamaños/colores) y múltiples listas de precios.
- Multiusuario con roles (ej.: admin, vendedor/a) con Lucia/Auth.js.
- Notas con recordatorios (follow-ups).

---

## 13) Checklist de salida a producción
- [ ] `ADMIN_PASSWORD` fuerte y único
- [ ] `AUTH_SECRET` rotado y guardado
- [ ] `DATABASE_URL` gestionada y con backups
- [ ] `SMTP_*` probados (si se usa email)
- [ ] `VITE_*` (WhatsApp/Instagram) configurados
- [ ] HTTPS activo + dominio propio
- [ ] `prisma migrate deploy` ejecutado
- [ ] `/health` OK en la plataforma
- [ ] Pruebas E2E verdes tras el deploy
- [ ] Política de privacidad y términos (mínimos) publicados

---

## 14) Troubleshooting exprés
- **No conecta a DB**: revisar `DATABASE_URL`, abrir puerto/SSL, `prisma migrate deploy`.
- **Form contacto falla**: chequear rate-limit (429), logs de server, validación Zod.
- **Email no sale**: probar credenciales SMTP, puerto, bloqueo del proveedor.
- **/admin redirige a login**: validar cookie, `AUTH_SECRET`, reloj del server.
- **Imágenes no cargan**: rutas relativas en `static/`, caché del navegador.
- **Lighthouse bajo**: comprimir imágenes, lazy-load, revisar métricas de red.

---

## 15) Mantenimiento
- Actualizaciones semanales de dependencias (menores/patch).
- Auditoría de dependencias trimestral (deprecations/vulnerabilidades).
- Backups automáticos de DB y verificación de restore (ensayo mensual).
- Revisión trimestral de KPI y limpieza de tags/duplicados.

---

**Listo.** Tu ShowCasePro queda con GPS: de local a prod, sin vueltas. ¡A producir ventas! 🚀

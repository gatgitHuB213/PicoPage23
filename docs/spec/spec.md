# Especificación del proyecto --- P.I.C.O Social Media (red social privada del grupo)

**Versión:** 0.3.1\
**Fecha:** octubre de 2026\
**Estado:** Especificación para desarrollo asistido por agentes (Claude Code)\
**Objetivo inmediato:** primera versión desplegada y probada para el
**31 de octubre de 2026** (fiesta de disfraces de Halloween).

------------------------------------------------------------------------

## 0. Cambios respecto a v0.2 (leer primero)

Esta versión resuelve las contradicciones de v0.2 y fija las decisiones
tomadas por el administrador. **Si algo de otro documento contradice
esta sección, manda esta sección.**

1.  **Los integrantes son solo lectores en la v1.** Pueden iniciar
    sesión, ver el calendario, ver los eventos y ver las bios
    privadas. No crean, editan ni activan nada.
2.  **El administrador publica todo.** Eventos, perfiles, textos e
    imágenes los gestiona únicamente el administrador.
3.  **No hay panel de administración visual en la v1.** El
    administrador opera mediante scripts CLI y endpoints protegidos.
4.  **No hay RSVP, highlights, publicaciones, comentarios ni
    reacciones en la v1.** Quedan documentados como evolución futura
    (sección 12.2).
5.  **No hay cambio de contraseña para usuarios.** Existe una función
    interna `changePass` (script CLI) que usa solo el administrador.
6.  **Un solo origen.** Un reverse proxy interno sirve el frontend y
    enruta `/api` hacia NestJS. Cloudflare Tunnel apunta únicamente al
    proxy.
7.  **El despliegue se hace temprano** (esqueleto funcionando en el
    dominio desde los primeros días), no al final.
8.  **La landing no debe indexarse** (`noindex` + `robots.txt`).
9.  **Prioridad de la v1: la parte visual** (landing e invitación),
    sobre una base segura y simple.
10. **Nombre del proyecto: P.I.C.O Social Media.** Los diseños de Figma
    (mocks) van en `docs/design/` junto con una nota de design tokens
    (paleta, tipografías, radios) y los assets exportados.
11. **Páginas de estado** definidas en la sección 6.1 (secciones no
    disponibles, login requerido, 403, 404).

------------------------------------------------------------------------

## 1. Resumen del proyecto

Aplicación web para un grupo de amigos que se conocen desde la
educación media. Combina:

1.  Una **landing pública**, visual y cinematográfica, con fotografías
    del grupo desde la infancia hasta la actualidad.
2.  Un **área privada** con inicio de sesión, solo lectura para los
    integrantes.
3.  **Biografías** por integrante, con texto, imágenes y bromas
    internas.
4.  Un **calendario de eventos** privado.
5.  **Páginas privadas de evento**, comenzando por la invitación de
    Halloween, escrita en código por el administrador.
6.  Una base técnica que permita añadir, en versiones futuras,
    highlights públicos, publicaciones, RSVP, comentarios y subida de
    archivos sin reescribir la arquitectura.

La aplicación debe seguir siendo útil después de Halloween; la
invitación es el primer evento, no el propósito final.

## 2. Objetivos y alcance

### 2.1 MVP (v1): obligatorio para el 31 de octubre

-   Landing pública responsive con animaciones ligadas al scroll.
-   Login, sesiones persistentes y logout.
-   Cuentas creadas por el administrador mediante script.
-   Perfiles/bios de integrantes (lectura para integrantes
    autenticados).
-   Calendario visible solo para usuarios autenticados.
-   Eventos creados y gestionados solo por el administrador, con
    estados de publicación.
-   Página de invitación de Halloween, escrita en código.
-   Todas las páginas y datos de eventos privados, con autorización
    verificada en el backend.
-   Despliegue con Docker Compose + reverse proxy + Cloudflare Tunnel.
-   Controles de seguridad básicos, `noindex` y copia de seguridad
    probada.

### 2.2 Fuera del alcance de la v1

-   Registro público de usuarios.
-   Cualquier función de escritura para integrantes (crear eventos,
    editar perfil, activar/desactivar eventos, RSVP).
-   Cambio de contraseña por parte del usuario y recuperación por
    correo (SMTP).
-   Highlights, publicaciones, comentarios, reacciones.
-   Subida de archivos por integrantes y videos.
-   Panel de administración visual y editor de eventos.
-   Paletas configurables de animaciones y colores.
-   Chat en tiempo real, app móvil nativa, Kubernetes, microservicios.

Estas funciones deben poder añadirse sin reescribir la arquitectura
central.

## 3. Stack técnico

### Frontend

-   React + TypeScript + Vite
-   React Router
-   Motion para transiciones y animaciones ligadas al scroll
-   CSS Modules u otra solución equivalente; decidir al iniciar

### Backend

-   Node.js + NestJS + TypeScript
-   API REST
-   Validación explícita de DTOs
-   OpenAPI/Swagger solo en desarrollo; deshabilitado en producción

### Persistencia

-   PostgreSQL + Prisma ORM
-   Migraciones versionadas
-   Restricciones e índices en la base de datos

### Infraestructura

-   Docker Compose
-   Reverse proxy interno (Caddy o nginx; decidir al implementar, Caddy
    suele ser más simple)
-   Cloudflare Tunnel (ya existe uno operativo para otro proyecto; se
    añade un hostname/ruta para este)
-   Dominio propio
-   PostgreSQL en red Docker interna, sin puertos publicados
-   Volúmenes persistentes para la base de datos
-   Imágenes base **Debian slim** (`node:*-slim`), no Alpine, para
    evitar problemas con dependencias nativas (Argon2, Prisma)
-   Versiones de Node y dependencias fijadas (LTS vigente)

No usar Kubernetes ni microservicios.

## 4. Arquitectura lógica

``` text
Navegador
   |
   | HTTPS
   v
Cloudflare (DNS / Tunnel)
   |
   v
cloudflared (ya existente)
   |
   v
Reverse proxy interno  <-- único punto de entrada (un solo origen)
   |
   +--> /        Frontend React (build estático)
   |
   +--> /api/*   NestJS (red Docker interna)
                    |
                    +--> Autenticación y sesiones
                    +--> Usuarios y perfiles
                    +--> Calendario y eventos
                    |
                    v
               PostgreSQL (red Docker interna, sin exposición)
```

Servicios de Compose previstos: `proxy` (sirve el build del frontend y
enruta `/api`), `api` y `db`.

Reglas:

-   El puerto del proxy no se expone a Internet. Si `cloudflared` corre
    en el mismo host, enlazar el puerto a `127.0.0.1`. Si corre en otra
    VM, restringir el puerto por firewall a esa IP.
-   PostgreSQL y la API no publican puertos al host.
-   El frontend nunca se conecta a PostgreSQL.
-   Con un solo origen no se necesita CORS. Si algún día se separan
    subdominios, definir orígenes explícitos y nunca usar `*` con
    credenciales.

### Organización recomendada del repositorio

``` text
grupo-social/
├── apps/
│   ├── web/                 # React + TypeScript + Vite
│   └── api/                 # NestJS + Prisma
├── docs/
│   ├── spec.md              # esta especificación
│   ├── architecture.md
│   ├── data-model.md
│   ├── security.md
│   └── api.md
├── infra/
│   ├── docker-compose.yml
│   ├── proxy/               # Caddyfile o nginx.conf
│   └── backup/              # scripts de backup/restauración
├── CLAUDE.md                # instrucciones cortas para el agente
├── .env.example
├── .gitignore
└── README.md
```

`CLAUDE.md` debe ser breve: apuntar a `docs/spec.md`, listar comandos
de desarrollo/prueba y repetir las reglas innegociables (sección 14).

## 5. Roles y autorización

### Visitante no autenticado

Puede: ver la landing pública y acceder a la pantalla de login.

No puede: consultar el calendario, la lista de eventos, páginas de
evento, bios privadas ni ningún dato privado por llamadas directas a la
API.

### Integrante autenticado (`MEMBER`), solo lectura

Puede: ver el calendario, abrir los eventos publicados, ver las bios
del grupo y cerrar sesión.

No puede: crear, editar, activar/desactivar o eliminar eventos; editar
perfiles; administrar cuentas; cambiar contraseñas; concederse el rol
de administrador.

### Administrador (`ADMIN`)

Puede: crear, deshabilitar y administrar cuentas (por CLI); gestionar
perfiles y bios; crear, editar, publicar, desactivar y eliminar
eventos; operar la configuración del sistema.

**Regla esencial:** ocultar botones en React no es autorización.
NestJS valida identidad y rol en cada endpoint protegido. Como en la
v1 los integrantes no tienen funciones de escritura, los endpoints de
escritura de contenido son exclusivos de `ADMIN`.

### Herramientas administrativas (CLI)

Scripts ejecutados dentro del contenedor `api`, nunca expuestos por
HTTP:

-   `user:create`: crea una cuenta (rol y username).
-   `user:change-password` (`changePass`): cambia la contraseña de un
    usuario y **revoca todas sus sesiones activas**.
-   `user:disable` / `user:enable`: deshabilita o habilita una cuenta
    (al deshabilitar se revocan sus sesiones).

Requisitos de los scripts: leer contraseñas por prompt interactivo o
stdin, **nunca como argumento de línea de comandos** (queda en el
historial); no imprimir contraseñas ni hashes en logs; hash con
Argon2id.

## 6. Landing pública y presentación visual

Esta es la prioridad visual de la v1.

### Dirección artística

-   Estética negra, cinematográfica y editorial.
-   Tipografía expresiva y contraste alto.
-   Fotografías de la infancia y de la actualidad.
-   Transiciones de opacidad, escala y posición ligadas al scroll.
-   Presentación de los integrantes.
-   Responsive y accesible.
-   Respetar `prefers-reduced-motion`.

### Estructura orientativa

1.  Hero con título y presentación del grupo.
2.  Secuencia de fotografías de la infancia.
3.  Transición a fotografías actuales.
4.  Presentación de los integrantes.
5.  Acceso al login.
6.  Pie de página.

### Contenido público

-   El contenido público de la landing (fotos y textos) es **estático y
    versionado en el repositorio**, elegido explícitamente por el
    administrador. Todos los integrantes ya dieron su conformidad para
    el uso de las fotografías.
-   La landing no consulta endpoints públicos de datos de usuarios en la
    v1.
-   No mostrar contenido privado automáticamente.

### No indexación

-   Cabecera `X-Robots-Tag: noindex, nofollow, noarchive` en **todas**
    las respuestas, configurada en el reverse proxy.
-   `<meta name="robots" content="noindex, nofollow">` en el HTML.
-   `robots.txt` con `Disallow: /`.
-   Sin sitemap.
-   Nota: la cabecera es el mecanismo principal. `robots.txt` es una
    señal adicional; los rastreadores que respetan `Disallow` no leen
    el `meta`, por eso se mantienen los tres.
-   Metadatos Open Graph genéricos (título e imagen neutros) para la
    vista previa al compartir el enlace por WhatsApp, **sin datos
    privados de eventos**.

### 6.1 Páginas de estado y secciones no disponibles

Todas comparten un mismo componente y estilo (mock "Todavía no está
lista") con un botón para volver al inicio.

-   **Sección no implementada en la v1 (Highlights):** ruta del
    frontend con la página "todavía no está lista". Es una página
    normal (HTTP 200), no un error. Alternativa: quitar el enlace del
    menú.
-   **Visitante sin sesión en una ruta privada** (Eventos, perfiles,
    evento): redirigir al login con un mensaje breve y volver a la ruta
    original tras autenticarse. El destino de retorno debe ser siempre
    una ruta interna (evitar redirecciones abiertas).
-   **API:** responde con los códigos HTTP reales. `401` sin sesión;
    `403` para un integrante autenticado sin permiso (por ejemplo,
    endpoints de admin); `404` para rutas inexistentes. El frontend
    solo presenta esos casos.
-   **Recursos privados que el usuario no puede ver** (evento
    `DRAFT` o `INACTIVE`, o inexistente): la API responde `404`, no
    `403`, para no revelar que existen.

## 7. Perfiles y biografías

Cada integrante tiene una página de perfil (solo lectura) con, como
mínimo: nombre visible, nombre de usuario, biografía o texto
personalizado e imagen o avatar.

-   Las bios completas son **privadas** (requieren sesión) y se sirven
    desde la API.
-   El administrador escribe y mantiene las bios (script, seed o
    endpoint de admin). Los integrantes no editan su perfil en la v1.
-   **Toda imagen que no sea pública se sirve mediante un endpoint
    autenticado.** Nada privado puede quedar en la carpeta `public/`
    del frontend ni en el build estático. Los avatares ya usados en la
    landing pública pueden vivir como assets estáticos.
-   Subida libre de archivos: fuera de la v1.

Ejemplos de tono (ilustrativos): "Gat. Yo soy el admin, te puedo
banear 😎" / "Santi, te voy a raguear cualquier cosa 😡".

## 8. Calendario y eventos

### Capacidades de la v1

-   El integrante autenticado ve el calendario y abre los eventos
    publicados.
-   El administrador crea, edita, publica, desactiva y elimina eventos.
-   Los integrantes no tienen ninguna acción de escritura sobre
    eventos.

### Estados del evento

-   `DRAFT`: borrador; solo lo ve el administrador.
-   `PUBLISHED`: visible para integrantes autenticados.
-   `INACTIVE`: oculto del calendario activo; conserva sus datos.

`cancelled` queda fuera de la v1 (se puede añadir como estado después).

Desactivar no es eliminar. Eliminar es una acción distinta, solo de
administrador, y se prefiere borrado lógico. Puede aplazarse la
eliminación permanente.

### Privacidad de los eventos

Todos los eventos son privados:

-   Requieren sesión válida; el backend verifica el acceso en cada
    solicitud.
-   Esconder el enlace no basta; una URL conocida o adivinada no debe
    exponer contenido.
-   Sin datos privados de eventos en páginas públicas, metadatos HTML
    ni endpoints públicos.

### Página de evento de Halloween

Experiencia personalizada escrita en código por el administrador:
título y descripción, fecha/hora/ubicación, fotografías, elementos
gráficos y animaciones, información logística y enlace de retorno al
calendario.

Se asocia a un evento de la base de datos mediante `templateKey` (por
ejemplo `halloween-2026`).

**Regla de privacidad del código:** un componente React viaja en el
bundle de JavaScript, que es público. Por lo tanto:

-   Los **datos sensibles** (fecha exacta, hora, ubicación, logística)
    se entregan desde la API solo a usuarios autenticados.
-   El componente contiene únicamente lo visual (animaciones,
    maquetación, imágenes ya públicas o servidas con autenticación).
-   Si se quiere ocultar también lo visual, cargar el componente como
    chunk separado servido solo con sesión válida. Opcional.

## 9. Modelo de datos (conceptual; concretar en Prisma)

### `User`

`id`, `username` (único), `displayName`, `passwordHash` (Argon2id),
`role` (`ADMIN` | `MEMBER`), `isActive`, `createdAt`, `updatedAt`.

Sin contraseñas en texto plano. Sin endpoint público de registro.

### `Session`

`id` (aleatorio e impredecible; guardar su hash, no el valor crudo),
`userId`, `createdAt`, `expiresAt`, `lastSeenAt`.

Debe permitir revocación al cerrar sesión, deshabilitar una cuenta o
ejecutar `changePass`.

### `Profile`

`userId` (único), `bio`, `avatarKey` (referencia controlada).

### `Event`

`id`, `creatorId`, `title`, `description`, `startsAt`, `endsAt`
(opcional), `location` (opcional), `status` (`DRAFT` | `PUBLISHED` |
`INACTIVE`), `templateKey` (opcional), `createdAt`, `updatedAt`,
`publishedAt` (opcional).

### Preparado para el futuro (sin migraciones en la v1)

Se documentan para no cerrar puertas, pero **no se crean tablas ni
endpoints en la v1**: `EventAttendee` (RSVP), `Highlight`, `Post`,
`PostMedia`, `CalendarSettings`.

No guardar binarios grandes en PostgreSQL; las tablas futuras guardarán
referencias al almacenamiento.

Aplicar claves foráneas, índices y restricciones únicas. Definir
deliberadamente las reglas de borrado para no eliminar contenido
relacionado por accidente.

## 10. Autenticación y seguridad

### Estrategia

Sesiones de servidor mediante cookies seguras. No usar JWT en el MVP.

### Requisitos

-   Cookies `HttpOnly`, `Secure` en producción y `SameSite=Lax` (o
    `Strict` si no rompe el flujo de login).
-   Expiración y revocación de sesiones; regenerar el identificador al
    autenticarse.
-   Política propuesta: sesión de 30 días con renovación móvil, para
    que los integrantes no tengan que reingresar desde el celular
    (confirmar en la sección 15).
-   Logout que invalida la sesión en el servidor.
-   Argon2id para contraseñas.
-   Rate limiting en login, con la IP real del cliente correctamente
    resuelta detrás de Cloudflare y el proxy (configurar proxy
    confiable y revisar la cabecera usada; no confiar en cabeceras
    arbitrarias del cliente).
-   Mensajes de error que no revelen si una cuenta existe.
-   Validación de DTOs en NestJS.
-   **CSRF:** con un solo origen y `SameSite`, verificar `Origin` /
    Fetch Metadata en métodos no seguros (POST/PUT/PATCH/DELETE). Como
    los integrantes no tienen mutaciones salvo login/logout, el riesgo
    es bajo; si se añaden mutaciones para integrantes, incorporar token
    CSRF.
-   Control de acceso por rol en cada endpoint (guards de NestJS).
-   Secretos solo en variables de entorno; nunca versionar `.env`.
-   Dependencias actualizadas y logs sin contraseñas, cookies ni
    tokens.
-   Deshabilitar cuentas sin borrar sus datos.
-   Swagger deshabilitado o restringido a admin en producción.

Consultar las guías vigentes de OWASP (autenticación, sesiones, control
de acceso, validación) antes de cerrar la implementación.

## 11. Cloudflare Tunnel y despliegue

-   Reutilizar el `cloudflared` existente añadiendo un hostname para
    este proyecto que apunte al reverse proxy. No mezclar rutas con el
    otro proyecto.
-   Publicar únicamente los hostnames necesarios.
-   No exponer PostgreSQL ni puertos administrativos de Docker o del
    host.
-   Persistir PostgreSQL con volúmenes.
-   Configuración y secretos fuera de las imágenes y del repositorio.
-   Health checks y política de reinicio (`restart: unless-stopped`).
-   Probar el acceso desde una red externa (datos móviles), no solo
    desde la LAN.
-   Probar el reinicio de contenedores y de la VM, y que la
    aplicación vuelva sola.
-   **Backup:** `pg_dump` periódico guardado **fuera de la VM** (otro
    disco o host), con restauración ensayada antes del lanzamiento.
    Además, snapshot de la VM antes del despliegue final.
-   Plan B de la invitación documentado (ver sección 15).

## 12. Plan de implementación

### 12.1 Fases

**Fase 1 --- Cimientos y despliegue esqueleto (primeros días)**

-   Repositorio, estructura, `CLAUDE.md` y `docs/spec.md`.
-   Docker Compose con `proxy`, `api` y `db`; API de salud conectada a
    PostgreSQL; frontend "hola mundo".
-   Hostname del Tunnel apuntando al proxy; verificación desde datos
    móviles.
-   Cabeceras `noindex` y `robots.txt` desde el inicio.
-   Documentar arquitectura y variables de entorno.

**Fase 2 --- Identidad y permisos**

-   Migraciones de `User`, `Session`, `Profile`, `Event`.
-   Scripts CLI: `user:create`, `user:change-password`, `user:disable`.
-   Login/logout con cookies seguras; guards por rol.
-   Pruebas de acceso directo a la API (sin sesión, `MEMBER`, `ADMIN`).
-   Cuenta administradora creada con el procedimiento seguro.

**Fase 3 --- Contenido y API de lectura**

-   Endpoint privado de perfiles/bios.
-   Endpoints de eventos: lectura para integrantes; escritura solo
    admin; estados y desactivación.
-   Carga inicial de integrantes, bios y el evento de Halloween.

**Fase 4 --- Visual**

-   Landing cinematográfica con animaciones de scroll.
-   Pantalla de login, páginas de perfiles y calendario.
-   Página de invitación de Halloween.
-   Esta fase puede avanzar en paralelo con la Fase 3 una vez acordados
    los contratos de API.

**Fase 5 --- Cierre y pruebas**

-   Pruebas de integración y seguridad: rutas privadas, URLs directas
    de eventos, datos públicos vs privados, cookies, persistencia tras
    reinicio.
-   Prueba desde móvil y escritorio, y desde red externa.
-   Backup y ensayo de restauración.
-   **Congelamiento de funcionalidades unos 2-3 días antes del 31 de
    octubre.** Lo restante es corrección de errores y pulido visual.

Priorizar entregas pequeñas, ejecutables y probables antes de iniciar
la siguiente fase.

### 12.2 Después del MVP (evolución prevista)

-   **Highlights:** apartado visible para todo el público. En el futuro
    los integrantes autenticados podrán proponer o autorizar que una
    publicación o un corto de un juego aparezca allí. El flujo de
    aprobación se define entonces (por ejemplo, revisión del
    administrador). En la v1 no existe.
-   RSVP/confirmación de asistencia (primera candidata tras la v1).
-   Creación de eventos y funciones de escritura para integrantes.
-   Cambio de contraseña por el usuario y recuperación por correo.
-   Publicaciones, comentarios y reacciones.
-   Almacenamiento de archivos y videos, con límites definidos.
-   Panel de administración y constructor visual de eventos.
-   Migración a VPS si la disponibilidad lo justifica.
-   Servicios adicionales en el dominio con hostnames y controles
    independientes.

## 13. Criterios de aceptación del MVP

### Landing

-   [ ] Carga sin autenticación.
-   [ ] Se adapta a móvil y escritorio.
-   [ ] Las animaciones no bloquean la navegación y respetan
    `prefers-reduced-motion`.
-   [ ] Solo muestra contenido elegido como público.
-   [ ] Todas las respuestas incluyen `X-Robots-Tag: noindex`; existen
    `meta robots` y `robots.txt`.

### Autenticación

-   [ ] Las cuentas las crea el administrador por script.
-   [ ] Login correcto con credenciales válidas; las incorrectas no
    crean sesión.
-   [ ] La sesión persiste según la expiración configurada.
-   [ ] El logout invalida la sesión en el servidor.
-   [ ] `changePass` cambia la contraseña y revoca las sesiones del
    usuario.
-   [ ] Una cuenta deshabilitada no puede iniciar sesión y pierde sus
    sesiones.
-   [ ] El rate limiting de login funciona con la IP real tras el
    proxy.
-   [ ] Las rutas privadas no se pueden consultar sin sesión.

### Perfiles

-   [ ] Cada integrante tiene un perfil identificable.
-   [ ] Los integrantes autenticados ven las bios privadas.
-   [ ] Un visitante no accede a datos privados mediante la API.
-   [ ] Ninguna imagen privada es accesible sin sesión.

### Calendario y eventos

-   [ ] Un integrante autenticado ve el calendario y abre eventos
    publicados.
-   [ ] Un `MEMBER` recibe 403 en cualquier endpoint de escritura de
    eventos.
-   [ ] Solo el `ADMIN` crea, edita, publica, desactiva y elimina
    eventos.
-   [ ] La página de Halloween se abre desde el calendario.
-   [ ] La URL directa de un evento no evita la autenticación y los
    datos sensibles no están en el bundle público.
-   [ ] Los eventos `INACTIVE` y `DRAFT` no aparecen para integrantes.
-   [ ] Los datos persisten tras reiniciar contenedores y la VM.

### Despliegue

-   [ ] La aplicación funciona a través del dominio, desde red
    externa, en móvil y escritorio.
-   [ ] Existe un único origen; no hay errores de CORS ni de cookies.
-   [ ] PostgreSQL y la API no son accesibles desde Internet ni desde
    la LAN por puertos publicados.
-   [ ] Los secretos no están en Git.
-   [ ] Backup y restauración ensayados, con copia fuera de la VM.

## 14. Instrucciones para agentes de código

Los agentes deben:

1.  Leer este documento, `CLAUDE.md` y `README.md` antes de modificar
    el proyecto.
2.  Respetar el stack acordado salvo razón técnica concreta, explicada.
3.  Trabajar en tareas pequeñas y verificables.
4.  No inventar credenciales, secretos, datos personales ni servicios
    externos.
5.  No eliminar ni sobrescribir datos persistentes para resolver un
    problema de desarrollo.
6.  Añadir migraciones versionadas al modificar el esquema.
7.  Añadir pruebas de autenticación, autorización y operaciones
    sensibles.
8.  No implementar un frontend que finja que la API ya es segura.
9.  No implementar funciones fuera del alcance de la v1 (sección 2.2)
    aunque parezcan fáciles.
10. No considerar terminada una tarea sin ejecutar las pruebas
    pertinentes y documentar las limitaciones.
11. Mantener actualizada la documentación cuando cambie una decisión.
12. No poner contenido privado en `public/` ni en el bundle.

Antes de añadir una biblioteca importante, revisar si el stack actual
ya resuelve la necesidad. Sin microservicios, Kubernetes ni
complejidad innecesaria.

## 15. Decisiones pendientes y supuestos por confirmar

Decididas:

-   Un solo origen con reverse proxy.
-   Integrantes solo lectura en la v1; admin publica todo.
-   Sin highlights, RSVP, publicaciones ni cambio de contraseña de
    usuario en la v1; `changePass` interno.
-   Fotografías aprobadas por todo el grupo; `noindex` obligatorio.
-   Deploy temprano usando el Tunnel ya existente.

Pendientes:

-   Estilo gráfico definitivo (en curso en Figma) y dominio/subdominio
    exacto. El nombre ya está decidido: P.I.C.O Social Media.
-   Política de sesión (propuesta: 30 días con renovación móvil).
-   Plan B de la invitación si el homelab no está disponible el día del
    evento (por ejemplo, una versión estática alternativa en otro
    hosting o un comunicado por el chat del grupo).
-   Cómo se entregan las credenciales iniciales a cada integrante.
-   Caddy o nginx como reverse proxy.
-   Si se desea ocultar también lo visual de la invitación (chunk
    servido solo con sesión) o basta con proteger los datos sensibles.

**Principio de desarrollo:** priorizar una versión segura, visualmente
sólida y desplegable antes del 31 de octubre. La arquitectura facilita
la evolución, pero las funciones futuras no deben convertirse en
obstáculos para el lanzamiento.

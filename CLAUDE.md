# P.I.C.O Social Media

## Contexto
Red social privada de un grupo de amigos + invitación de Halloween
(31 de octubre de 2026). Este repo es la parte web (apps/web). La API,
el proxy y el despliegue viven en la VM y los trabaja otro Claude Code.
Lee docs/spec/spec.md (v0.3.1) antes de modificar nada. Si algo choca
con este archivo, manda el spec.

## Alcance de la v1
- Los integrantes son SOLO LECTURA: login, calendario, eventos, bios
  y logout.
- Fuera de la v1: highlights, RSVP, posts, comentarios, cambio de
  contraseña, panel de admin, subida de archivos. No implementarlos
  aunque parezcan fáciles.
- Prioridad: lo visual (landing e invitación de Halloween) sobre una
  base segura.

## Reparto
- Windows (este equipo): apps/web (landing, páginas de estado, panel),
  con el MCP de Figma.
- VM: compose, proxy, API, auth y despliegue.
- Sincronización: git push / git pull.

## Reglas
- docs/reference/ es SOLO consulta (frontend del proyecto G5 de la U).
  Está en .gitignore: nunca lo commitees ni lo copies tal cual;
  reutiliza ideas y estructura.
- Diseño: docs/design/ (PNG de Figma y tokens.md). Los tokens de
  P.I.C.O mandan sobre los colores de la referencia.
- Stack de apps/web: React + TypeScript + Vite + React Router +
  Tailwind + Motion.
- Autenticación: sesión de servidor con cookie HttpOnly. El frontend
  NO guarda tokens ni datos personales en localStorage/sessionStorage.
  Usar fetch con credentials: 'include' y GET /api/auth/me para saber
  quién es el usuario.
- API: mismo origen, ruta relativa /api. En desarrollo, proxy de Vite
  hacia la VM (destino en .env.local, no versionado). Nada hardcodeado.
- El contrato de la API vive en docs/api-contract.md. Si cambia,
  avisar a la VM.
- Privacidad: nada privado en public/ ni en el bundle. Los datos
  sensibles de eventos (fecha, hora, lugar, logística) vienen siempre
  de la API. Mientras no haya API, datos falsos solo en src/mocks.
- Landing pública: contenido estático elegido a mano. Con noindex.
- Páginas de estado (login requerido, 403, 404, "no está lista"):
  spec sección 6.1.
- Accesibilidad: respetar prefers-reduced-motion, texto negro sobre
  amarillo, fuentes autohospedadas, imágenes en WebP optimizado.
- No modificar docs/spec/ sin avisar.
- Git: nunca commitear .env, docs/reference/ ni imágenes originales
  pesadas.
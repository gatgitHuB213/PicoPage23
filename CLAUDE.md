# P.I.C.O

## Contexto
Proyecto personal. Este repo es la parte web (apps/web). El backend, la API, el proxy y el despliegue viven en la VM y los trabaja otro Claude Code.

## Reparto
- Windows (este equipo): apps/web (landing, páginas de estado, panel), con el MCP de Figma.
- VM: compose, proxy, API, auth y despliegue.
- Sincronización: git push / git pull.

## Reglas
- docs/reference/ es SOLO consulta (frontend del proyecto G5 de la U). Está en .gitignore: nunca lo commitees ni lo copies tal cual; reutiliza ideas y estructura.
- Diseño: docs/design/ (PNG de Figma y tokens.md). Spec: docs/spec/ (v0.3.1).
- Stack de apps/web: React + TypeScript + Vite + Tailwind (igual que la referencia).
- Convención de sesión en localStorage: auth_token, auth_role, auth_rut, auth_user_id, auth_email. Si cambia, avisar a la VM.
- La URL de la API va en una variable de entorno (VITE_API_URL), nunca hardcodeada.

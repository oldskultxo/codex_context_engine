# Global AGENTS Instructions (Projects)

Aplica a cualquier repositorio bajo `/Users/santisantamaria/Documents/projects`.

## External Memory First

- Usa `/Users/santisantamaria/Documents/projects/codex_context` como primera capa de memoria.
- Antes de análisis profundo o cambios grandes:
  1. Consulta `ctx <topic-or-symptom>` para recuperar notas relevantes.
  2. Consulta `ctx --prefs` para cargar preferencias de trabajo del usuario.
- Si memoria y código/runtime discrepan, prioriza código/runtime y actualiza `codex_context`.

## User Preferences (Default Behavior)

- Toma `user_preferences.json` como defaults operativos entre sesiones.
- Precedencia:
  1. Instrucción explícita del usuario en este turno.
  2. Preferencias persistidas en `user_preferences.json`.
  3. Comportamiento por defecto del asistente.
- Si detectas una preferencia repetida (ejemplo: sin feedback intermedio, siempre tests/build/docs), persístela con:
  - `prefer <dot.key> <value> --source explicit_user|inferred --confidence low|medium|high --reason "..."`

## Session Workflow

- Inicio de tarea:
  - `ctx <tema>` y `ctx --prefs`.
- Cierre de tarea no trivial:
  - actualiza nota existente o crea nota nueva en `codex_context`;
  - registra cambios estructurales relevantes en `change_journal.md`.

## Commands

- Lookup: `ctx <keyword>`
- Open best note: `ctx --open <keyword>`
- JSON output: `ctx --json <keyword>`
- Preferences:
  - `ctx --prefs`
  - `prefer --show [key]`
  - `prefer <dot.key> <value> ...`

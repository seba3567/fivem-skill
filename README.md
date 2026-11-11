# FiveM Skills

Colección definitiva y optimizada de habilidades de agente de IA para el desarrollo de servidores y recursos de **FiveM (Cfx.re / GTA V)**.

Diseñado para agentes de programación (Claude Code, Gemini CLI, Antigravity, Cursor, Codex, OpenCode) que interactúan con código Lua 5.4, TypeScript, NUI y frameworks FiveM.

---

## 🚀 Contenido y Habilidades

### 1. `fivem-development` (Núcleo Principal)
El skill más exhaustivo y actualizado para el desarrollo en FiveM:
- **7,379 Nativas Oficiales Sincronizadas**: Compiladas en vivo desde las bases de datos oficiales de `citizenfx/fivem` (`static.cfx.re` y `runtime.fivem.net`) divididas en 47 namespaces markdown.
- **Referencias de Juego Oficiales**: Extraídas de `citizenfx/fivem-docs` (Controls IDs, Blips, Hashes de Peds, Hashes de Armas, Game Events, Zones, Markers).
- **Soporte para ox_inventory Personalizado**: Documentación completa del inventario personalizado de ChilencyRP (sistema físico de llaves de vehículos en guantera, hornos/fundición térmicos, telemetría balística, compatibilidad con `qb-inventory` y `origen_inventory`).
- **Arquitectura Flexible (*Zero-Hard-Dependency*)**: Patrón puente y adaptadores para crear recursos que no requieran obligatoriamente `ox_lib` ni `ox_inventory`, con fallbacks limpios a QBCore, ESX o Standalone nativo.
- **Frameworks Soportados**: Qbox, ESX Legacy, QBCore y Standalone.
- **Seguridad y Rendimiento**: Reglas estrictas de validación de servidor (`source`), mitigación de exploits, OneSync infinity y optimización de loops.

### 2. Habilidades FiveM Especializadas (Overextended Stack)
- **`deslop`**: Limpieza de código generado por IA eliminando redundancias sin alterar validaciones críticas de servidor.
- **`lua-best-practices`**: Buenas prácticas oficiales de CfxLua 5.4, imports modulares (`require`), convars y arquitectura de scripts.
- **`resource-best-practices`**: Manejo de entidades OneSync, ciclo de vida de entidades en servidor, state bags y control de red.
- **`typescript-best-practices`**: Estándares para NUI y recursos construidos en TypeScript/JavaScript.
- **`tdd`**: Desarrollo guiado por pruebas para recursos complejos.
- **`setup-ox`**: Configuración asistida del entorno de servidor y selección de modelos.

---

## 📦 Instalación

### En Gemini / Antigravity
Copia o enlaza las carpetas a tu directorio de configuración de skills:
```bash
git clone git@github.com:seba3567/fivem-skill.git
```
Y añade `skills/fivem/fivem-development` a `~/.gemini/config/skills/`.

### En Claude Code / Codex
```bash
claude plugin marketplace add seba3567/fivem-skill
```

---

## 🛠️ Herramientas CLI Incluidas (Python 3.8+)

En `skills/fivem/fivem-development/scripts/`:
- **Búsqueda instantánea de nativas**:
  ```bash
  python scripts/natives.py search <NombreDeNativa>
  ```
- **Detalle de nativa**:
  ```bash
  python scripts/natives.py show <NombreExacto>
  ```
- **Actualizar catálogo desde CitizenFX**:
  ```bash
  python scripts/build_natives_catalog.py --refresh
  ```
- **Auditoría de seguridad y backdoors en recursos**:
  ```bash
  python scripts/audit.py /ruta/a/recurso
  ```

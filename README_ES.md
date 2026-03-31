# Informes de Investigación sobre Claude Code

[![Version](https://img.shields.io/badge/Claude_Code-v2.1.88-blueviolet?style=for-the-badge)](https://www.npmjs.com/package/@anthropic-ai/claude-code)
[![Reports](https://img.shields.io/badge/Informes-12%2B-red?style=for-the-badge)](docs/)

Colección de informes de investigación independientes sobre **Claude Code v2.1.88**, centrados en arquitectura, bucle de agente, modelo de permisos, prompt de sistema, integración MCP y gestión de contexto.

Este repositorio se mantiene deliberadamente como **documentación solamente**. Contiene análisis y comentario técnico, no una distribución ejecutable de código fuente ni un build de CLI soportado.

**Idioma**: [English](README.md) | [中文](README_CN.md) | [日本語](README_JA.md) | [한국어](README_KO.md) | **Español**

---

## Propósito del Repositorio

Claude Code es uno de los agentes de programación con IA más sofisticados en producción. Estos informes buscan hacer más estudiables sus mecanismos internos.

- **Desarrolladores de agentes** pueden estudiar patrones de orquestación de herramientas, gestión de contexto y recuperación
- **Investigadores de seguridad** pueden observar telemetría, mecanismos de control remoto y límites de permisos
- **Investigadores de IA** pueden analizar ensamblaje de prompts, enrutamiento de modelos y diseño del bucle de agente
- **Ingenieros** pueden usar estos informes como referencia para sistemas de agentes basados en CLI

Estos informes se basan en análisis técnico de un **paquete npm distribuido públicamente** y se presentan como investigación y comentario.

---

## Informes de Investigación

### Análisis de Arquitectura Central

| # | Informe | Qué Aprenderás | Enlace |
|---|---------|----------------|--------|
| 06 | **Inmersión en el Bucle de Agente** | Análisis de `query.ts` — flujo de mensajes, ejecución de herramientas en streaming, auto-compactación y recuperación | [Leer →](docs/en/06-agent-loop-deep-dive.md) |
| 07 | **Arquitectura del Sistema de Herramientas** | Cómo se registran, validan y ejecutan en paralelo más de 40 herramientas | [Leer →](docs/en/07-tool-system-architecture.md) |
| 08 | **Modelo de Permisos y Seguridad** | Listas blancas/negras, auto-aprobación, modo YOLO e integración con sandbox | [Leer →](docs/en/08-permission-security-model.md) |
| 09 | **Ingeniería de Prompts de Sistema** | Cómo se ensamblan 15,000+ tokens desde más de 20 partes | [Leer →](docs/en/09-system-prompt-engineering.md) |
| 10 | **Integración MCP y Sistema de Plugins** | Implementación del cliente Model Context Protocol | [Leer →](docs/en/10-mcp-integration.md) |
| 11 | **Gestión de Ventana de Contexto** | Auto-compactación, compresión de conversación y conteo de tokens | [Leer →](docs/en/11-context-window-management.md) |
| 12 | **Gestión de Estado y Persistencia** | Estado de sesión, historial y sistema de memoria | [Leer →](docs/en/12-state-management.md) |

### Informes de Descubrimiento e Investigación

| # | Informe | Qué Aprenderás | Enlace |
|---|---------|----------------|--------|
| 01 | **Telemetría y Recolección de Datos** | Pipeline dual de análisis y huella del entorno | [Leer →](docs/en/01-telemetry-and-privacy.md) |
| 02 | **Funciones Ocultas y Nombres en Código** | Codenames, feature flags y diferencias entre builds internos y externos | [Leer →](docs/en/02-hidden-features-and-codenames.md) |
| 03 | **Modo Encubierto** | Cómo se ocultan señales de autoría por IA en repositorios públicos | [Leer →](docs/en/03-undercover-mode.md) |
| 04 | **Control Remoto e Interruptores de Emergencia** | Configuración del lado servidor, flags y controles de emergencia | [Leer →](docs/en/04-remote-control-and-killswitches.md) |
| 05 | **Hoja de Ruta Futura** | KAIROS, Numbat, pistas sobre modelos futuros y herramientas no publicadas | [Leer →](docs/en/05-future-roadmap.md) |

---

## Resumen del Código Analizado

Estas cifras se refieren a la **instantánea de código de Claude Code analizada**, no al contenido de este repositorio documental.

| Métrica | Valor |
|---------|-------|
| Archivos TypeScript | **1,884** |
| Líneas totales de código | **512,664** |
| Archivo más grande | `query.ts` — **785KB** |
| Herramientas integradas | **40+** |
| Comandos slash | **80+** |
| Dependencias npm | **192 paquetes** |
| Módulos Feature-Gated | **108** |
| Modelo de runtime | Paquete construido con Bun, orientado a Node.js |

---

## Vista General de la Arquitectura

```
                          ┌─────────────────┐
                          │   User Input     │
                          │ (CLI / SDK / IDE)│
                          └────────┬────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │        Entry Layer           │
                    │  cli.tsx → main.tsx → REPL   │
                    │              └→ QueryEngine   │
                    └──────────────┬──────────────┘
                                   │
              ┌────────────────────▼────────────────────┐
              │           Query Engine Core              │
              │   System Prompt Assembly / Agent Loop    │
              └────────────────────┬────────────────────┘
                                   │
         ┌─────────────────────────▼─────────────────────────┐
         │              Tool Layer (40+ tools)               │
         └─────────────────────────────────────────────────────┘
```

---

## Estructura de Módulos Observada

La estructura de código analizada en los informes se puede resumir así:

```
src/
├── main.tsx
├── QueryEngine.ts
├── query.ts
├── Tool.ts
├── Task.ts
├── tools.ts
├── commands.ts
├── context.ts
├── cost-tracker.ts
├── setup.ts
├── bridge/
├── cli/
├── commands/
├── components/
├── entrypoints/
├── hooks/
├── services/
├── state/
├── tasks/
├── tools/
├── types/
├── utils/
└── vendor/
```

Este repositorio en sí permanece como archivo documental:

```
docs/
├── en/
└── zh/

README.md
README_CN.md
README_JA.md
README_KO.md
README_ES.md
QUICKSTART.md
```

---

## Uso

- Leer los informes dentro de `docs/`
- Compartir enlaces a informes individuales
- Tratar este repositorio como un archivo documental, no como distribución de software

Consulta [QUICKSTART.md](QUICKSTART.md) para una guía mínima de lectura.

---

## Nota Legal

Este proyecto constituye **investigación, comentario y análisis educativo** sobre un paquete de software distribuido públicamente (el paquete npm `@anthropic-ai/claude-code`).

Los informes en `docs/` son análisis y comentarios originales del mantenedor. Si tienes alguna inquietud de derechos, abre un issue para revisión.

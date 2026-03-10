name: choreo-maestro
version: 1.0.0
description: Orquestra flujos de trabajo automatizados de múltiples pasos para las operaciones de bots de OpenClaw
author: OpenClaw Team
tags: [workflow, orchestration, coordination, process, automation]
type: orchestration
dependencies:
  - jq >= 1.6
  - curl >= 7.68.0
  - redis-cli (opcional, para persistencia de estado)
  - systemd (para integración de servicios)
enabled: true
```

# Skill Choreo Maestro

## Propósito

Choreo Maestro orquesta secuencias de automatización complejas de múltiples pasos en el ecosistema distribuido de bots de OpenClaw. Coordina comportamientos de bots, gestiona transiciones de estado, maneja recuperación de errores y garantiza la ejecución atómica de operaciones dependientes.

Casos de uso reales:
- Coordinar rotación de farming de 5 bots con tiempos de inicio escalonados y distribución de loot
- Ejecutar secuencias de jefes de múltiples fases con asignación de roles (tanque, sanador, DPS)
- Gestionar ventanas de mantenimiento de servidor: apagado graceful de bots, actualizaciones de configuración, orquestación de reinicio
- Manejar cadenas de misiones dinámicas que requieren transferencias de objetos bot-a-bot en puntos de control específicos
- Ejecutar automatización de eventos estacionales con desencadenantes temporizados y ramificación condicional

## Alcance

Comandos disponibles:
- `choreo:create-workflow` - Definir nueva secuencia orquestada
- `choreo:register-bot` - Añadir bot al grupo de coordinación
- `choreo:assign-role` - Mapear bots a roles de flujo de trabajo
- `choreo:start-sequence` - Ejecutar flujo de trabajo orquestado
- `choreo:pause-sequence` - Detener todos los bots de forma graceful
- `choreo:resume-sequence` - Continuar desde estado pausado
- `choreo:abort-sequence` - Parada de emergencia con limpieza
- `choreo:check-status` - Monitorear estado del flujo de trabajo
- `choreo:validate-dependencies` - Verificación pre-vuelo
- `choreo:rollback` - Revertir a estado estable anterior

## Proceso de Trabajo

1. **Definición de Flujo de Trabajo**
   ```
   choreo:create-workflow --name "farm-rotation-5" \
     --steps 12 \
     --duration 45m \
     --concurrency 3 \
     --failure-policy "retry:3,backoff:exponential"
   ```

2. **Registro de Bots**
   ```
   choreo:register-bot --id "bot-01" \
     --endpoint "ws://192.168.1.101:8080" \
     --capabilities "melee,ranged,tank" \
     --max-load 0.8
   ```

3. **Asignación de Roles**
   ```
   choreo:assign-role --workflow "farm-rotation-5" \
     --bot "bot-01" \
     --role "primary-dps" \
     --position "northeast-corner" \
     --timing-offset 0s
   ```

4. **Validación de Dependencias**
   ```
   choreo:validate-dependencies --workflow "farm-rotation-5" \
     --check network-latency \
     --check resource-availability \
     --check auth-tokens
   ```

5. **Inicio de Secuencia**
   ```
   choreo:start-sequence --workflow "farm-rotation-5" \
     --start-time "now" \
     --notify-on-completion \
     --log-level detailed
   ```

6. **Monitoreo en Tiempo de Ejecución**
   ```
   choreo:check-status --workflow "farm-rotation-5" \
     --format json \
     --watch
   ```

7. **Pausa/Reanudación Controlada**
   ```
   choreo:pause-sequence --workflow "farm-rotation-5" \
     --grace-period 30s \
     --save-state
   
   choreo:resume-sequence --workflow "farm-rotation-5" \
     --from-checkpoint "last-saved"
   ```

## Reglas de Oro

1. **Validar siempre las dependencias** antes de iniciar cualquier secuencia > 5 minutos de duración
2. **Nunca asignar posiciones superpuestas** a bots sin reglas explícitas de evitación de colisiones
3. **Establecer max-load por bot** - no exceder 85% CPU o 90% de umbral de memoria
4. **Implementar checkpointing** cada 10 minutos para flujos de trabajo de larga duración
5. **Usar backoff exponencial** para reintentos (mín: 2s, máx: 30s, factor: 2)
6. **Mantener logs de auditoría** - todos los cambios de estado deben ser journaled a `/var/log/choreo/`
7. **Probar rutas de rollback** mensualmente con entorno de staging
8. **Nunca hardcodear credenciales** - usar integración de vault: `choreo:get-secret <key>`
9. **Limpieza de recursos obligatoria** - incluir siempre flag `--cleanup` o paso de limpieza explícito
10. **Degradación graceful** - los flujos de trabajo deben continuar con capacidad reducida si los bots fallan

## Ejemplos

### Ejemplo 1: Rotación de Farming
```
Input:
choreo:create-workflow --name "gw2-farm-route" --steps 8 --duration 2h --concurrency 4
choreo:register-bot --id "anna" --endpoint "ws://10.0.0.1:8080" --capabilities "guardian"
choreo:assign-role --workflow "gw2-farm-route" --bot "anna" --role "tank" --path "/routes/farm01.txt"
choreo:start-sequence --workflow "gw2-farm-route" --start-time "2026-03-10T14:00:00Z"

Output:
✓ Workflow created (ID: wf_abc123)
✓ Bot registered (ID: bot_anna)
✓ Role assigned: anna → tank on route /routes/farm01.txt
✓ Dependencies validated (network OK, sufficient resources, tokens valid)
✓ Sequence scheduled for 2026-03-10T14:00:00Z
[14:00:00] ▶ Starting workflow gw2-farm-route
[14:00:02] ✓ Anna initialized, loading route file
[14:00:05] ✓ Route loaded (48 nodes, 12 farming spots)
[14:00:10] ▶ Bot "anna" executing step 1/8
```

### Ejemplo 2: Coordinación de Jefe
```
Input:
choreo:create-workflow --name "xd-boss-ragnos" --steps 20 --concurrency 5 --failure-policy "abort"
choreo:register-bot --id "tank01" --capabilities "tank,cc-immunity" --max-load 0.75
choreo:register-bot --id "heal01" --capabilities "heal,cleanse" --max-load 0.65
choreo:assign-role --workflow "xd-boss-ragnos" --bot "tank01" --role "main-tank" --phase 1,2,3
choreo:assign-role --workflow "xd-boss-ragnos" --bot "heal01" --role "main-heal" --phase 1,2,3
choreo:validate-dependencies --workflow "xd-boss-ragnos"
choreo:start-sequence --workflow "xd-boss-ragnos" --start-time "now+30s" --notify-discord "# raid-alerts"

Output:
✓ Workflow created (ID: wf_def456)
✓ Bots registered: tank01, heal01
✓ Role assignment complete (2 bots, 3 phases)
✓ Validation passed: all bots online, latency <50ms, healing capacity 12k/s
✓ Sequence starting in 30s
[14:00:30] ▶ Engaging boss: Ragnos the Fractured
[14:00:32] ✓ Tank01 positioning at (123,456)
[14:00:33] ✓ Heal01 monitoring tank
[14:01:15] ▶ Phase 2 transition
[14:01:17] ✓ Positioning adjusted, adds spawning
[14:02:45] ✓ Boss defeated, loot distribution triggered
```

### Ejemplo 3: Recuperación de Emergencia
```
Input:
choreo:pause-sequence --workflow "gw2-farm-route" --grace-period 15s --save-state
choreo:check-status --workflow "gw2-farm-route" --format json

Output:
[14:30:00] ⏸ Pausing workflow gw2-farm-route (grace: 15s)
[14:30:01] ✓ Anna completing current node, saving position
[14:30:03] ⏸ State saved at node #7, progress 63%
[14:30:15] ✓ All bots paused, network connections closed

Status:
{
  "workflow": "gw2-farm-route",
  "state": "PAUSED",
  "checkpoint": {
    "node": 7,
    "position": [1240.5, 890.3, 12.1],
    "inventory": [...],
    "timestamp": "2026-03-10T14:30:03Z"
  },
  "bots": {
    "anna": "paused",
    "bot_02": "paused"
  }
}
```

## Requisitos

Sistema:
- Linux kernel 5.4+
- Python 3.9+ (para motor de orquestación)
- Redis 6.2+ (persistencia de estado opcional)
- 100MB espacio libre en disco para logs
- Red: <100ms latencia a todos los endpoints de bot

Variables de entorno:
- `CHOREO_VAULT_ADDR` - Endpoint HashiCorp Vault (requerido para secrets)
- `CHOREO_LOG_LEVEL` - debug/info/warn/error (por defecto: info)
- `CHOREO_STATE_DIR` - directorio de checkpoint (por defecto: /var/lib/choreo)
- `CHOREO_MAX_RETRIES` - límite global de reintentos (por defecto: 3)

Estructura de archivos:
```
/var/lib/choreo/
├── workflows/          # Definiciones de flujo de trabajo
├── checkpoints/        # Estados guardados automáticamente
├── bots/               # Registro de bots
└── journal.log         # Trail de auditoría
```

## Pasos de Verificación

Después de cualquier cambio de orquestación:

1. Verificar sintaxis de flujo de trabajo:
   ```
   choreo:validate-dependencies --workflow <nombre> --strict
   ```

2. Simulación dry-run (sin afectar bots):
   ```
   choreo:start-sequence --workflow <nombre> --dry-run --verbose
   ```

3. Verificar asignación de recursos:
   ```
   choreo:check-status --workflow <nombre> --format json | jq '.resource_usage'
   ```

4. Verificar persistencia de estado:
   ```
   ls -l /var/lib/choreo/checkpoints/<workflow-id>.json
   ```

5. Probar ruta de rollback:
   ```
   choreo:rollback --workflow <nombre> --to-checkpoint latest --dry-run
   ```

## Solución de Problemas

**Problema**: "concurrency limit exceeded"
- **Solución**: Aumentar `--concurrency` o reducir asignaciones de bots por flujo de trabajo
- **Verificar**: `choreo:check-status --all | grep concurrency`

**Problema**: "bot endpoint unreachable"
- **Solución**: Verificar que WebSocket del bot está corriendo: `curl -I ws://<ip>:8080`
- **Verificar**: Firewall de red, estado del servicio bot

**Problema**: "checkpoint save failed"
- **Solución**: Asegurar que `/var/lib/choreo/checkpoints/` es escribible, disco no lleno
- **Verificar**: `df -h /var/lib/choreo`, `ls -ld /var/lib/choreo/checkpoints`

**Problema**: "state deserialization error on resume"
- **Solución**: Usar `choreo:rollback --to-checkpoint <timestamp>` para revertir a estado conocido-bueno
- **Verificar**: Validar JSON de checkpoint: `jq . /var/lib/choreo/checkpoints/<archivo>`

**Problema**: "dependency validation timeout"
- **Solución**: Aumentar timeout: `export CHOREO_VALIDATE_TIMEOUT=30s`
- **Verificar**: Latencia de respuesta del bot: `choreo:ping-bot --id <bot-id>`

**Problema**: "role assignment conflict"
- **Solución**: Asegurar pares únicos de `(position + phase)` por flujo de trabajo
- **Verificar**: `choreo:list-assignments --workflow <nombre> --format table`

## Comandos de Rollback

Rollback inmediato al checkpoint anterior:
```
choreo:rollback --workflow "gw2-farm-route" --to-checkpoint latest
```

Rollback a timestamp específico:
```
choreo:rollback --workflow "gw2-farm-route" --to-checkpoint "2026-03-10T14:30:03Z"
```

Revertir flujo de trabajo completo con limpieza:
```
choreo:rollback --workflow "gw2-farm-route" \
  --to-checkpoint initial \
  --cleanup-paused-bots \
  --notify-ops
```

Abort de emergencia + rollback (usar con cautela):
```
choreo:abort-sequence --workflow "gw2-farm-route" \
  --force \
  --rollback-to "stable-state" \
  --disconnect-bots
```

Rollback parcial (solo bots fallidos):
```
choreo:rollback --workflow "gw2-farm-route" \
  --bots "bot_03,bot_07" \
  --to-checkpoint "last-known-good"
```

Verificar éxito de rollback:
```
choreo:check-status --workflow "gw2-farm-route" | grep "state"
# Debería mostrar: ROLLED_BACK or STABLE
```

Audit trail de rollback:
```
grep "ROLLBACK" /var/log/choreo/journal.log | tail -10
```

Nota: Todos los rollbacks preservan entradas de auditoría y no pueden ser deshechos sin intervención manual. Siempre validar salud de bots después del rollback: `choreo:ping-all --workflow <nombre>`.
```
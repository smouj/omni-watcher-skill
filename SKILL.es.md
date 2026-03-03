name: omni-watcher
description: monitor de archivos con IA para mejora de escritura en tiempo real y generación de contenido
version: 2.1.0
author: OpenClaw Team
tags:
  - writing
  - ai
  - automation
  - real-time
  - markdown
  - content
dependencies:
  - node >=18.0.0
  - openai-api
  - chokidar
  - yaml
  - diff
  - ts-node
  - typescript
platforms:
  - linux
  - macos
  - windows
entrypoint: bin/omni-watcher
```

# Omni Watcher

Monitor de archivos en tiempo real impulsado por IA que supervisa archivos de escritura (Markdown, texto sin formato, JSON, YAML) y proporciona mejoras automáticas, finalizaciones y aplicación de estilo mientras escribes.

## Propósito

**Casos de uso reales:**

1. **Asistente de escritura de documentación** - Observar el directorio `/docs/` y mejorar automáticamente archivos Markdown para mayor claridad, gramática y precisión técnica mientras se preservan bloques de código y frontmatter.

2. **Flujo de trabajo de blog** - Monitorear un directorio `content/posts/`. Cuando se guarda un archivo `.md`, analizar la estructura de oraciones, sugerir mejores transiciones y asegurar títulos amigables para SEO.

3. **Guardián de documentación de API** - Observar archivos YAML OpenAPI/Swagger para garantizar terminología consistente, detectar descripciones faltantes y marcar parámetros no documentados automáticamente.

4. **Aplicación de estándares de escritura en equipo** - Ejecutar en CI o modo desarrollo para asegurar que toda la documentación de los miembros del equipo siga guías de estilo establecidas (AP Style, Google Developer Documentation Style, etc.).

5. **Monitor de calidad de traducción** - Observar archivos Markdown traducidos para detectar frases incómodas, mantener consistencia terminológica entre versiones lingüísticas y marcar posibles traducciones erróneas.

## Alcance

**Tipos de archivo admitidos:**
- `.md`, `.markdown` (Markdown con sabor GitHub)
- `.txt` (texto sin formato)
- `.yaml`, `.yml` (preservación de frontmatter YAML)
- `.json` (JSON con comentarios)
- `.mdx` (MDX con JSX)

**Comandos activos:**

```bash
# Iniciar observación con mejoras IA por defecto
omni-watcher --watch ./docs

# Observar con modelo IA específico y prompts personalizados
omni-watcher --watch ./content --model gpt-4 --prompt "Haz el tono entusiasta, añade emojis"

# Observar y aplicar correcciones automáticamente (modo interactivo requiere --confirm)
omni-watcher --watch ./src/docs --auto-fix --backup-dir ./backups

# Modo diferencial: mostrar cambios sin aplicar
omni-watcher --watch ./notes --preview-only --diff

# Análisis de archivo único (sin observación)
omni-watcher --analyze README.md --output improved.md

# Configurar vía YAML
omni-watcher --config .omni-watcher.yml

# Observar con patrones de ignorar personalizados
omni-watcher --watch . --ignore "**/drafts/**" --ignore "**/*.template.md"

# Modo integración: salida a stdout para CI
omni-watcher --ci ./docs --format json --fail-on-quality-score-below 80

# Dry-run con logging detallado
omni-watcher --watch ./docs --dry-run --log-level debug

# Observación multi-directorio con reglas diferentes por directorio
omni-watcher --watch ./api-docs --rules api-rules.yml --watch ./tutorials --rules tutorial-rules.yml
```

## Proceso de Trabajo

**Ejecución paso a paso:**

1. **Fase de inicialización**
   ```bash
   # Cargar config desde fuentes prioritarias:
   # 1. archivo --config si se especifica
   # 2. .omni-watcher.yml en directorio actual
   # 3. .omni-watcher.yml en directorio home
   # 4. Configuración por defecto
   ```

2. **Descubrimiento de archivos**
   - Escanear recursivamente directorios observados
   - Filtrar por extensión (lista blanca: md, markdown, txt, yaml, yml, json, mdx)
   - Aplicar patrones glob `--ignore`
   - Construir estado inicial de archivos con hashes SHA256

3. **Activación del observador**
   - Usar `chokidar` con `awaitWriteFinish: { stabilityThreshold: 500, pollInterval: 100 }`
   - Debounce de guardados rápidos consecutivos (por defecto: 1000ms)
   - Detectar adiciones, modificaciones y eliminaciones de archivos

4. **Detección de cambios**
   ```typescript
   if (newHash !== oldHash) {
     categorizeChange(file);
     categorizeChange(file);
   }
   ```

5. **Pipeline de procesamiento IA**
   ```
   Archivo de entrada → Extraer contenido (preservar frontmatter/bloques de código) → Aplicar reglas personalizadas →
   Dividir si >4000 tokens → Enviar a OpenAI API con prompt de sistema → Recibir sugerencias →
   Parsear diff → Aplicar cambios (interactivo o automático) → Generar backup si auto-fix →
   Escribir archivo mejorado → Registrar resumen → Disparar hooks si están configurados
   ```

6. **Puntuación de calidad**
   - Calcular puntuación de legibilidad (Flesch-Kincaid)
   - Revisar errores gramaticales (vía LanguageTool API o OpenAI)
   - Verificar cumplimiento de estilo (reglas regex personalizadas)
   - Devolver código de salida 0-100 para modo CI

## Reglas de Oro

**Restricciones específicas:**

1. **Preservación de frontmatter**: Nunca modificar frontmatter YAML entre delimitadores `---` en archivos Markdown. Tratarlo como inmutable a menos que el flag `--update-frontmatter` esté explícitamente activado.

2. **Integridad de bloques de código**: Todos los bloques de código con ```language deben permanecer sin cambios. Las sugerencias de IA solo pueden apuntar a texto fuera de bloques de código. Usar regex `^```[\s\S]*?^```$` con flag multilínea para extraer y omitir.

3. **Requisito de backup**: Cuando `--auto-fix` está activado, crear backups con marca de tiempo en `--backup-dir` antes de aplicar cambios. Formato de nombre de backup: `original-YYYYMMDD-HHMMSS.md`. Retener últimos 10 backups por archivo, eliminar más antiguos.

4. **Aplicación de límite de tokens**: Si el contenido del archivo excede 4000 tokens, dividir en secciones lógicas (por encabezados o párrafos) y procesar independientemente, luego combinar resultados. Nunca truncar contenido; siempre procesar secciones completas.

5. **Limitación de velocidad**: Implementar retroceso exponencial para fallos de API de OpenAI. Reintentos máximos: 3. En límite de velocidad (429), esperar 2^n * 1000ms donde n es el conteo de reintentos. Registrar advertencias en reintentos.

6. **Sin red**: Cuando el modo `--offline` está activo, omitir procesamiento IA y solo validar contra reglas locales. Salir con código 2 para indicar procesamiento parcial.

7. **Cumplimiento CI**: En modo `--ci`, la salida debe ser JSON legible por máquina:
   ```json
   {
     "file": "docs/README.md",
     "status": "passed|failed|error",
     "quality_score": 85,
     "issues": ["line 12: passive voice", "line 45: jargon detected"],
     "changes_applied": 3
   }
   ```

8. **Seguridad de hilos**: Los procesos del observador procesan archivos secuencialmente, no en paralelo, para prevenir condiciones de carrera cuando múltiples archivos se guardan simultáneamente. Límite de tamaño de cola: 10 archivos.

## Ejemplos

**Ejemplo 1: Mejora de documentación con guía de estilo**

```bash
# Comando
omni-watcher --watch ./docs --model gpt-4 --prompt-file .styleguide.txt --auto-fix --backup-dir ./backups

# Contenido de .styleguide.txt:
# Usar voz activa
# Longitud máxima de oración: 25 palabras
# Evitar voz pasiva
# Usar coma de Oxford
# Términos técnicos: Primera aparición debe estar capitalizados (API, REST, JSON)

# Entrada: docs/getting-started.md antes
## Introducción
The configuration file is created by the installer. Settings can be modified by the user.

# Después de auto-fix:
## Introducción
The installer creates the configuration file. You can modify settings.

# Salida de log:
[INFO] Archivo cambiado: docs/getting-started.md
[INFO] Procesando con GPT-4...
[INFO] Aplicadas 3 mejoras (voz pasiva, coma de Oxford)
[INFO] Backup creado: backups/getting-started-20240315-143022.md
[INFO] Puntuación de calidad: 92/100
```

**Ejemplo 2: Modo CI con calidad de puerta**

```bash
# .github/workflows/docs.yml
- name: Verificar calidad de documentación
  run: |
    omni-watcher --ci ./docs --format json --fail-on-quality-score-below 85 > quality-report.json
    cat quality-report.json
    
# Salida en fallo:
{
  "file": "docs/api.md",
  "status": "failed",
  "quality_score": 78,
  "issues": [
    "line 23: Missing description for 'userId' parameter",
    "line 45: Inconsistent capitalization (api vs API)",
    "line 89: Sentence exceeds 30 words"
  ],
  "changes_applied": 0
}
# Código de salida 1 (CI falla)
```

**Ejemplo 3: Multi-directorio con reglas diferentes**

```bash
# Comando:
omni-watcher \
  --watch ./api-reference --rules api-rules.yml \
  --watch ./tutorials --rules tutorial-rules.yml \
  --diff --log-level info

# api-rules.yml:
rules:
  - type: terminology
    enforce: ["REST API", "HTTP", "endpoint"]
    forbid: ["rest api", "Rest API", "http"]
  - type: docstring
    require: ["description", "parameters", "returns"]

# Cuando ./api-reference/users.md cambia:
[INFO] Aplicando api-rules.yml a api-reference/users.md
[DIFF] Línea 14: 'rest api' → 'REST API'
[DIFF] Línea 22: Falta descripción de parámetro para 'role'
[PROMPT] Añadir descripciones de parámetros para: role, createdAt
```

**Ejemplo 4: Análisis de archivo único para calidad de traducción**

```bash
omni-watcher --analyze ./translations/es/marketing.md --target-language en --check-consistency --output consistency-report.txt

# Salida:
Verificación de consistencia de traducciones para es/marketing.md
✓ Consistencia de términos: "carrito" siempre mapeado a "cart"
⚠ Potencial discordancia: "pago" traducido como "payment" (esperado: "checkout")
✗ Traducción faltante: 3 cadenas encontradas en versión en no presentes en es
Recomendación: Revisar secciones 2.1 y 4.3 para completitud.
```

## Comandos de Rollback

**Procedimientos específicos de rollback:**

```bash
# 1. Revertir archivo único a backup anterior (si se usó --backup-dir)
cp ./backups/getting-started-20240315-143022.md ./docs/getting-started.md

# 2. Restaurar todos los archivos desde lote de backup (usar con precaución)
omni-watcher --restore ./backups/batch-20240315-143022 --target ./docs

# 3. Rollback basado en git (recomendado para archivos bajo git)
git checkout HEAD -- docs/changed-file.md
# o para múltiples archivos:
git restore --source=HEAD ./docs/

# 4. Detener observador gracefulmente (SIGTERM) o forzadamente (SIGKILL)
# Graceful: envía señal de completado, termina archivo actual, sale
pkill -TERM -f "omni-watcher"
# Forzar:
pkill -KILL -f "omni-watcher"

# 5. Rollback de último lote de auto-fixes (si se usa ledger de omni-watcher)
omni-watcher --undo --since "2024-03-15T14:30:00Z"

# 6. Deshabilitar modo auto-fix mientras se preserva observador
# Editar config o reiniciar sin flag --auto-fix:
omni-watcher --watch ./docs  # (sin --auto-fix)

# 7. Limpiar estado del observador (nuevos hashes de archivos)
rm -f .omni-watcher-state.json

# 8. Revertir configuración a por defecto
mv .omni-watcher.yml .omni-watcher.yml.backup
omni-watcher --watch ./docs  # usa valores por defecto

# Verificación de seguridad antes de rollback:
# 1. Listar backups disponibles
ls -lht ./backups/
# 2. Verificar integridad de backup
head -20 ./backups/file-20240315-143022.md
# 3. Confirmar que archivo no está modificado desde backup
diff ./docs/file.md ./backups/file-20240315-143022.md

# Parada de emergencia (matar todos los procesos relacionados):
pkill -f "omni-watcher" && echo "Observador detenido"
```

**Verificación después de rollback:**
```bash
# Verificar que el observador se ha detenido
ps aux | grep omni-watcher

# Verificar archivo restaurado
cat ./docs/restored-file.md | head -5

# Reiniciar observador en modo preview para asegurar sin cambios pendientes
omni-watcher --watch ./docs --preview-only --log-level info
```
```
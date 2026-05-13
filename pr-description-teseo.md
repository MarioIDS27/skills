# PR Description TESEO

Genera descripciones de PR en formato TESEO leyendo los cambios reales del branch actual.

## Proceso

1. **Recopilar cambios reales de la PR:** Si hay numero de PR, usar SIEMPRE `gh pr diff {numero}` y `gh pr diff {numero} --name-only` para obtener los cambios exactos. Tambien `gh pr view {numero} --json commits --jq '.commits[].messageHeadline'` para los commits. **NUNCA usar `git diff {base}...HEAD`** ya que en ramas con muchos merges incluye cambios de otras PRs que no corresponden.
2. **Sin numero de PR:** Si no hay PR aun, usar `git diff {base}...HEAD -- "$(pwd)"` filtrado al directorio actual, y verificar que los archivos correspondan solo a los cambios del branch.
3. **Identificar rama y objetivo:** Extraer nombre de rama con `git branch --show-current`. Derivar el objetivo del PR a partir de los commits y los cambios de codigo.
4. **Analizar cambios tecnicos:** Para cada archivo modificado, resumir que cambio y por que es relevante. Agrupar por componente/modulo.
5. **Detectar clientes afectados:** Revisar si hay logica condicional por `codigoCustomer` en los cambios. Si no hay, marcar "Todos".
6. **Generar descripcion** usando el template exacto de abajo.
7. **Aplicar a la PR automaticamente:** Si el usuario proporciona un numero de PR, ejecutar `gh pr edit {numero} --body "{descripcion}"` para actualizar el body de la PR directamente en GitHub. Si no hay numero de PR, mostrar la descripcion al usuario para que la copie.

## Template

```markdown
# TESEO
Hola equipo!
 
Este PR trae mejoras visuales y de funcionalidad. Aqui tienes el detalle:
 
## Cambios Introducidos
- **Rama:** `{nombre-rama}`
- **Objetivo:** {objetivo-claro-del-PR}
 
### Implementacion Tecnica
**Componentes/Archivos modificados:**
{lista-de-archivos-modificados-con-guion}

**Cambios relevantes:**
{lista-de-cambios-con-descripcion-tecnica-detallada}

### Consideraciones Adicionales
{consideraciones-o-NA}
 
## Evidencia Visual
**Capturas:**
- Mejoras implementadas

{usuario-agrega-capturas-manualmente}
 
## Impacto en Clientes
**Clientes afectados:**
{lista-de-clientes-con-checkmark}

**Requiere comunicacion?** {Si/No}
 
## Checklist
- [x] Se valido y probo correcto funcionamiento.
 
Listo para revision!
```

## Reglas

- Escribir siempre en espanol.
- La seccion de evidencia visual queda como placeholder - el usuario la completa.
- Derivar el objetivo del PR del contenido real de los cambios, no inventar.
- En "Cambios relevantes", ser tecnico y especifico: que se agrego/modifico y por que.
- Si hay logica por customer (codigoCustomer), listar los clientes especificos. Si no, usar "Todos".
- No incluir archivos de configuracion triviales (package-lock.json, .vscode) a menos que sean relevantes.
- Cuando el usuario da un numero de PR, aplicar automaticamente con `gh pr edit`.

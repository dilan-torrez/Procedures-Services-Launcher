# Tarjeta de Referencia Rápida

## Sistema de Importación CSV/Excel

---

### Para Agregar Nueva Importación (10 minutos)

```bash
# 1. Crear Entity + DTO en el microservicio destino
# 2. Registrar handler NATS (copy-paste)
# 3. Crear configuración:
curl -X POST http://localhost:3000/api/import/configs \
  -H "Content-Type: application/json" \
  -d '{
    "name": "mi_importacion",
    "microservice": "collections",
    "schema": "collections",
    "table": "mi_tabla",
    "skipRows": 2,
    "columnMappings": [
      { "columnIndex": 1, "fieldName": "campo1" },
      { "columnIndex": 2, "fieldName": "campo2" }
    ]
  }'

# 4. Importar:
curl -X POST http://localhost:3000/api/import/mi_importacion \
  -F "file=@datos.csv"
```

---

### Formatos de Fecha Soportados

| Formato | Ejemplo | Estado |
|---------|---------|--------|
| Serial Excel | `46097` | ✅ |
| ISO 8601 | `2024-01-15T10:30:00Z` | ✅ |
| YYYY-MM-DD | `2024-01-15` | ✅ |
| DD/MM/YYYY | `15/01/2024` | ✅ |
| DD-MM-YYYY | `15-01-2024` | ✅ |
| MM/DD/YYYY | `01/15/2024` | ✅ |

---

### Endpoints Principales

```bash
# CRUD Configs
POST   /api/import/configs          # Crear/actualizar
GET    /api/import/configs          # Listar
GET    /api/import/configs/:name    # Ver una
PUT    /api/import/configs/:name    # Editar
DELETE /api/import/configs/:name    # Eliminar

# Importar
POST   /api/import/:name            # Subir archivo

# Historial
GET    /api/import/history          # Ver historial
GET    /api/import/records/:id      # Ver registro
```

---

### Arquitectura en 1 Línea

```
HTTP → Gateway → FTP(backup) → NATS → Global → Collections → PostgreSQL
```

---

### Características Clave

- ✅ Streaming O(1) memoria
- ✅ Deduplicación SHA256
- ✅ Lock secuencial por config
- ✅ Fechas multi-formato
- ✅ IDs reales de PostgreSQL
- ✅ Respaldos FTP automáticos
- ✅ Rollback automático en errores

---

### Documentación

- `PRESENTACION-PROPUESTA.md` - Presentación completa
- `RESUMEN-EJECUTIVO.md` - Resumen ejecutivo
- `MANUAL-TECNICO.md` - Documentación técnica
- `AGENTS.md` - Guía de implementación
- `MANUAL-IMPORTACION.md` - Para usuarios finales

---

### ¿Necesitas Ayuda?

Contactar al equipo de desarrollo o revisar la documentación en el repositorio.

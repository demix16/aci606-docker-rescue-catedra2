# Bitácora de reparaciones — ACI606

## Aclaración del historial Git

El commit b7a0e51 fue registrado accidentalmente
con el mensaje:

fix(R-04): corregir autenticacion PostgreSQL

Sin embargo, el cambio corresponde a la incidencia R-05,
relacionada con la corrección del mapeo de puertos de Nginx.

Modificación realizada:
- Archivo: compose.yaml
- Configuración anterior: 8080:8080
- Configuración corregida: 8080:80

Verificación:
- El frontend responde HTTP 200 OK.
- Nginx recibe conexiones correctamente en el puerto 80.

Se conserva el commit original para mantener la
trazabilidad del historial de reparaciones.
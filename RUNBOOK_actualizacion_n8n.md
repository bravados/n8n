# Cómo actualizar n8n sin dejar a tus clientes inoperativos

Tu instancia está **fijada en una versión concreta** (el tag `n8nio/n8n:2.25.7`
del docker-compose). Eso significa que **nada se actualiza solo**: un reinicio,
un `docker compose up` o un reboot del servidor **mantienen tu versión actual**.
Puedes quedarte en una versión que funciona durante meses sin tocar nada.

Solo actualizas cuando TÚ decides, y siguiendo este procedimiento.

---

## Regla de oro

**Nunca actualices directamente en producción.** Siempre: backup → staging →
prueba → producción. Y nunca un viernes ni en hora punta de tus clientes.

---

## Paso 1 — Lee qué cambia antes de saltar

1. Mira tu versión actual (el tag en `docker-compose.yml`).
2. Revisa el changelog oficial entre tu versión y la nueva:
   https://docs.n8n.io/release-notes/
3. Busca específicamente la sección **"Breaking changes"**.
4. **No saltes muchas versiones mayores de golpe.** Sube por incrementos
   (p. ej. de 2.25 a 2.26, no de 2.25 a 2.40 directamente).

Si el changelog no menciona breaking changes que afecten a tus nodos
(Webhook, Redis, Postgres, AI Agent, HTTP Request, Google Calendar), el
riesgo es bajo. Aun así, prueba en staging.

---

## Paso 2 — Backup ANTES de tocar nada

Backup de la base de datos de n8n (contiene workflows + credenciales):

```bash
docker compose exec -T postgres-n8n \
  pg_dump -U n8n -d n8n > backup_n8n_$(date +%Y%m%d_%H%M).sql
```

Backup del volumen de configuración de n8n (incluye la encryption key):

```bash
docker run --rm \
  -v n8n_data:/data \
  -v "$(pwd)":/backup \
  alpine tar czf /backup/n8n_data_$(date +%Y%m%d_%H%M).tar.gz -C /data .
```

Guarda ambos archivos fuera del servidor (otro disco, S3, etc.).

---

## Paso 3 — Prueba en staging

1. Copia el stack a otra carpeta (p. ej. `/opt/n8n-staging`) con otros
   puertos y otra red, o levántalo en otra máquina.
2. Restaura el dump de producción en el Postgres de staging:

   ```bash
   cat backup_n8n_AAAAMMDD_HHMM.sql | \
     docker compose exec -T postgres-n8n psql -U n8n -d n8n
   ```

3. En el `docker-compose.yml` de staging, cambia el tag a la versión nueva:

   ```yaml
   image: n8nio/n8n:NUEVA_VERSION
   ```

4. Levanta staging y **prueba de verdad**:
   - Abre cada workflow, comprueba que no hay nodos en rojo / deprecados.
   - Dispara manualmente un mensaje de prueba de WhatsApp de punta a punta.
   - Verifica que la memoria (Postgres) y el buffer (Redis) funcionan.

Si algo falla en staging, NO actualices producción. Resuélvelo primero.

---

## Paso 4 — Actualiza producción

Solo si staging pasó todas las pruebas:

1. Edita `docker-compose.yml` de producción y cambia el tag:

   ```yaml
   image: n8nio/n8n:NUEVA_VERSION
   ```

2. Aplica:

   ```bash
   docker compose pull n8n
   docker compose up -d n8n
   ```

   (Solo recrea el contenedor de n8n; Postgres y Redis siguen intactos.)

3. Comprueba logs y que arranca bien:

   ```bash
   docker compose logs -f n8n
   ```

4. Haz una prueba real de un mensaje de WhatsApp en producción.

---

## Paso 5 — Si algo sale mal: ROLLBACK

El rollback es rápido porque la versión vive en un tag:

1. Vuelve a poner el tag anterior en `docker-compose.yml`:

   ```yaml
   image: n8nio/n8n:2.25.7   # tu versión anterior
   ```

2. Recrea el contenedor:

   ```bash
   docker compose up -d n8n
   ```

3. Si la versión nueva había migrado el esquema de la BD y el rollback no
   arranca, restaura el dump del Paso 2:

   ```bash
   # primero recrea limpia la BD si es necesario, luego:
   cat backup_n8n_AAAAMMDD_HHMM.sql | \
     docker compose exec -T postgres-n8n psql -U n8n -d n8n
   ```

Por esto el backup del Paso 2 es innegociable: es tu botón de "deshacer".

---

## Resumen mental

- Versión fijada = nada cambia sin tu permiso.
- Backup siempre antes.
- Staging siempre antes de producción.
- Rollback = cambiar el tag + reiniciar (+ restaurar dump si hubo migración).
- Incrementos pequeños, nunca un viernes.

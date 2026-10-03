# ADR-008 — Entorno de Desarrollo Basado 100% en Docker

**Estado:** Aceptada · **Fecha:** 2026-10-03  
**Contexto del Reto:** Prueba Técnica SDD · Ficha ADSO 3413974

## Contexto

El proyecto requiere ser evaluado bajo condiciones estrictas a las 3:00 p. m.[cite: 2] Para evitar el clásico problema de "en mi máquina sí funciona" causado por diferencias en versiones de PHP, Composer, Node o MySQL en los equipos de los desarrolladores, se necesita un entorno unificado y aislado. 

Además, las reglas de la prueba técnica prohíben la instalación de herramientas en la máquina host.

## Decisión

1. **Cero Instalaciones Locales:** Queda estrictamente prohibido instalar PHP, Composer, Node.js o MySQL en la máquina anfitriona (Windows/Mac/Linux).
2. **Contenedores Efímeros para Comandos:** Toda ejecución de comandos (`composer install`, `php artisan`, `npm run build`) se realizará mediante contenedores efímeros montando el volumen local (ej. `docker run --rm composer`).
3. **Infraestructura Base:** El repositorio `test-simple-stock-flow-infra` será la única fuente de verdad para el despliegue local mediante `docker-compose.yml`, orquestando los servicios de `api` (PHP 8.2+) y `db` (MySQL 8.4).
4. **Volúmenes Nombrados:** Se utilizarán volúmenes persistentes para la base de datos (`db_data`) y los archivos subidos (`media_data`).

## Consecuencias

- **Positivas:**
  - Garantiza que el código correrá exactamente igual en la máquina del evaluador.
  - Mantiene el sistema operativo host completamente limpio de dependencias de desarrollo.
  - Estandariza la versión de PHP y extensiones (como `pdo_mysql` y `bcmath`) para todo el equipo.
- **Negativas asumidas:**
  - Requiere comandos más largos y verbosos en la terminal.
  - La instalación inicial de dependencias (Composer) puede ser más lenta debido a la sincronización de archivos entre el contenedor y el sistema host en Windows (I/O bridge).
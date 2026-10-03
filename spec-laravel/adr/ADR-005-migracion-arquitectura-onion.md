# ADR-005 — Adopción de Arquitectura Onion en Laravel y Ubicación de Puertos en Application

**Estado:** Aceptada · **Fecha:** 2026-10-03  
**Contexto del Reto:** Prueba Técnica SDD · Ficha ADSO 3413974

## Contexto

La especificación base del proyecto describe una arquitectura hexagonal en Python y .NET. Por directriz de la prueba, la implementación se realiza en **PHP con Laravel** bajo **Arquitectura Onion (4 anillos + punto de ensamblaje Bootstrap)**.

Existía debate técnico sobre la ubicación de las interfaces de persistencia y puertos de salida:
1. La Onion clásica de Palermo sitúa los contratos de repositorio en el anillo interior (`Domain`).
2. Nuestra Constitución técnica establece en el **Artículo II** que *«el núcleo declara sus puertos en `ports/outbound/`»* y en el **Artículo IV (DIP)** que *«los módulos de `ports/outbound/` viven en el paquete de aplicación»*.

## Decisión

Adoptar una Arquitectura Onion de 4 anillos concéntricos con las siguientes reglas innegociables:

```text
        ┌──────────────────────────────────────┐
        │  Bootstrap                           │  ← app/Bootstrap/
        │  PortBindingsServiceProvider.php     │    Enlaza interfaces con implementaciones
        └───────────────┬──────────────────────┘    (NO es un anillo)
                        │
     ┌──────────────────┴──────────────────┐
     ▼                                     ▼
┌─────────────────┐            ┌──────────────────────┐
│ Presentation    │            │ Infrastructure       │
│ Anillo 4        │            │ Anillo 3             │
│ controllers     │            │ Eloquent · mappers   │
│ requests        │            │ repos · JWT · media  │
│ resources       │            │ migraciones · config │
│ ═══ PROHIBIDO ══│            └──────────┬───────────┘
│ importar Infra  │                       │
└────────┬────────┘                       │
         │        imports hacia adentro   │
         ▼                                ▼
       ┌───────────────────────────────────────┐
       │ Application   Anillo 2                │
       │ casos de uso · 5 puertos inbound      │
       │ 10 puertos outbound · DTOs            │
       └──────────────────┬────────────────────┘
                          │ solo importa Domain
                          ▼
       ┌───────────────────────────────────────┐
       │ Domain   Anillo 1                     │
       │ entidades · value objects             │
       │ Cero dependencias externas o de BD    │
       └───────────────────────────────────────┘
Ubicación de Puertos: Los 5 puertos entrantes (Inbound) y los 10 puertos salientes (Outbound) se ubican estrictamente en app/Application/Ports/. Domain no conoce repositorios ni puertos de infraestructura.

Domain puro: Prohibido importar cualquier clase de Illuminate o paquete externo (salvo Brick\Math\BigDecimal en Money).

Presentation no conoce a Infrastructure: Los controladores únicamente inyectan casos de uso (Application\Ports\Inbound\*). Jamás inyectan repositorios directamente.

Punto de ensamblaje único: app/Bootstrap/PortBindingsServiceProvider.php es el único archivo donde se asocian las interfaces a las implementaciones concretas.

Mappers obligatorios: Eloquent se aísla en Infrastructure/Persistence/Model/. Los mappers convierten entre modelos de Eloquent y entidades de dominio puras, absorbiendo campos de control como version (ADR-002) y deleted_at (ADR-003) sin ensuciar la entidad.

Consecuencias
Positivas:

Cumplimiento estricto y comprobable de los Artículos I, II, III y IV de constitution.md.

La capa Application puede probarse al 100% con dobles en memoria sin necesidad de arrancar el framework Laravel ni la base de datos.

Prevención de fugas de Eloquent hacia la lógica de negocio.

Negativas asumidas:

Requiere escribir mappers explícitos entre Eloquent y el Dominio.

Exige verificar con herramientas estáticas (Deptrac / reflexión) que Presentation no importe Infrastructure.
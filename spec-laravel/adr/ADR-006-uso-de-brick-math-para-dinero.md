# ADR-006 — Uso de Brick\Math para el Manejo de Dinero

**Estado:** Aceptada · **Fecha:** 2026-10-03  
**Contexto del Reto:** Prueba Técnica SDD · Ficha ADSO 3413974

## Contexto

El contrato de la API y el modelo de datos exigen manejar importes monetarios (precios, totales) con precisión exacta de dos decimales[cite: 2]. Por defecto, PHP maneja los números decimales utilizando el tipo `float`, el cual está basado en el estándar IEEE 754.

El uso de `float` en operaciones financieras introduce problemas conocidos de precisión debido a cómo se representan los números binarios en memoria. Por ejemplo, la operación `0.1 + 0.2` en punto flotante no resulta exactamente en `0.3`, lo que puede causar errores acumulativos en sumas, descuentos o cálculos de inventario.

El proyecto requiere un redondeo determinista (`HALF_UP`) y cálculos matemáticos sin pérdida de precisión, independiente de la arquitectura del procesador.

## Decisión

1. **Prohibición del tipo `float`:** Queda estrictamente prohibido utilizar primitivas `float` nativas de PHP en cualquier capa de la aplicación (Domain, Application, Infrastructure o Presentation) para representar o calcular dinero[cite: 2].
2. **Adopción de `Brick\Math\BigDecimal`:** Se instalará el paquete `brick/math` mediante Composer (`composer require brick/math`) para manejar todas las operaciones de moneda.
3. **Encapsulamiento en el Dominio:** La librería se utilizará exclusivamente dentro de un Value Object puro llamado `Money` en la capa de `Domain` (`app/Domain/ValueObject/Money.php`). 
4. **Redondeo Innegociable:** Todas las operaciones aritméticas que resulten en decimales deberán aplicar redondeo utilizando la estrategia `RoundingMode::HALF_UP` a una escala de 2.

## Consecuencias

- **Positivas:**
  - Precisión absoluta en los cálculos financieros (ventas, sumatorias, descuentos).
  - Cumplimiento de las reglas de negocio estrictas descritas en la especificación.
  - El encapsulamiento en un Value Object `Money` evita que la dependencia `brick/math` se esparza por los casos de uso o controladores.
- **Negativas asumidas:**
  - Requiere instalar una dependencia externa en la capa de Dominio (única excepción permitida por el ADR-005).
  - Mayor verbosidad en el código al realizar operaciones matemáticas (`$price1->plus($price2)` en lugar de `$price1 + $price2`).
  - Ligero impacto en el rendimiento en comparación con las operaciones de hardware nativas, lo cual es despreciable para la escala de este proyecto.
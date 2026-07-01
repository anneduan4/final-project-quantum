# Proyecto final SS26 — Matching 4×4 → QUBO → QAOA local

Qubit.mx · QMexico Summer School 2026

## 1. Resumen

Este proyecto formula como matching bipartito 4×4 la especialización económica de cuatro
entidades federativas mexicanas frente a los cuatro grandes grupos de actividad económica
que reporta oficialmente el INEGI. El problema se traduce a QUBO, se resuelve de forma
exacta (fuerza bruta + permutaciones) y se resuelve también con **QAOA local** simulado
vectorialmente. La instancia molecular incluida en el notebook base fue reemplazada por
completo por este dataset real.

## 2. Fuente del dataset

- **Programa:** Producto Interno Bruto por Entidad Federativa (PIBE), año base 2018.
- **Institución:** INEGI (Instituto Nacional de Estadística y Geografía), Sistema de
  Cuentas Nacionales de México (SCNM).
- **Cifras usadas:** PIBE 2024, cifras preliminares, boletines individuales por entidad
  publicados el 5 de diciembre de 2025.
- **Licencia / condiciones de uso:** información estadística oficial de acceso público,
  bien público conforme a la Ley del Sistema Nacional de Información Estadística y
  Geográfica.
- **URLs consultadas (30 de junio de 2026):**
  - Boletín nacional: `https://www.inegi.org.mx/contenidos/saladeprensa/boletines/2025/pibent/PIBE2024_CP.pdf`
  - Ciudad de México: `https://www.inegi.org.mx/contenidos/saladeprensa/boletines/2025/pibent/PIBE2024_12_CP_CdMx.pdf`
  - Estado de México: `https://www.inegi.org.mx/contenidos/saladeprensa/boletines/2025/pibent/PIBE2024_12_RR_EdMx.pdf`
  - Nuevo León: `https://www.inegi.org.mx/contenidos/saladeprensa/boletines/2025/pibent/PIBE2024_12_CP_NL.pdf`
  - Jalisco: `https://www.inegi.org.mx/contenidos/saladeprensa/boletines/2025/pibent/PIBE2024_12_CP_Jal.pdf`
  - Serie completa: `https://www.inegi.org.mx/programas/pibent/2018/`
- **Fecha de consulta:** 30 de junio de 2026.

## 3. Por qué es un matching bipartito honesto

| Criterio | Respuesta |
|---|---|
| Dos lados identificables | $A$ = 4 entidades federativas; $B$ = 4 grandes grupos de actividad económica |
| Tamaño reducible a 4×4 | Se seleccionaron las 4 entidades con mayor participación en el PIB nacional 2024 (CDMX, Edomex, NL, Jalisco); los 4 grupos son exactamente los que INEGI reporta para **todas** las entidades (primarias, secundarias, terciarias, ISPN), así que no hay elección arbitraria de columnas |
| Score justificable | $S_{ij}$ es el porcentaje oficial que el grupo de actividad $j$ representa dentro del PIB de la entidad $i$ (dato publicado directamente, sin fórmulas propias) |
| Decisión binaria | $x_{ij}=1$ significa que, en un ejercicio **ilustrativo** de diversificación de cartera regional, el grupo de actividad $j$ se asigna a la entidad $i$ como su eje estructural distintivo |
| Restricciones | Cada entidad recibe un único grupo asignado; cada grupo se asigna a una única entidad (uno-a-uno, como en el modelo del notebook base) |
| Fuente legítima | INEGI/SCNM, con URL, institución y fecha de consulta documentadas arriba |
| Riesgo ético controlado | Datos macroeconómicos agregados y públicos; no hay personas identificables ni decisiones de alto impacto real |

## 4. Selección de las 4 entidades y los 4 grupos (regla reproducible)

- **Entidades ($A$):** las 4 con mayor participación en el PIB nacional 2024 según el
  boletín nacional del PIBE (CDMX 15.0 %, Estado de México 9.1 %, Nuevo León 8.1 %,
  Jalisco 7.5 %).
- **Grupos de actividad ($B$):** los 4 grandes grupos en los que INEGI divide el PIB de
  **cualquier** entidad federativa: actividades primarias, secundarias, terciarias e
  Impuestos y Subsidios a los Productos, Netos (ISPN). Al ser exactamente 4 y estar
  definidos igual para las 32 entidades, no hubo necesidad de recortar columnas de una
  tabla más grande: el propio programa estadístico ya entrega una matriz 4 (o más) × 4.

## 5. Matriz de score $S$ (participación % del grupo $j$ en el PIB de la entidad $i$, 2024)

| Entidad | Primarias | Secundarias | Terciarias | ISPN |
|---|---|---|---|---|
| CDMX | 0.0 | 9.4 | 83.5 | 7.0 |
| Estado de México | 1.4 | 26.7 | 65.3 | 6.6 |
| Nuevo León | 0.5 | 40.2 | 52.0 | 7.2 |
| Jalisco | 7.6 | 27.1 | 58.2 | 7.2 |

Cada fila suma aproximadamente 100 % (diferencias menores por redondeo del propio INEGI).
El archivo `data/dataset_real_4x4.csv` contiene estos mismos 16 pares `(a_id, b_id, score)`
más nombres legibles, en el formato que el notebook carga automáticamente.

## 6. Formulación QUBO

Variables binarias $x_{ij}\in\{0,1\}$, $i\in\{$CDMX, EDOMEX, NL, JAL$\}$,
$j\in\{$PRIM, SEC, TER, ISPN$\}$:

$$
\max_x \sum_{i}\sum_{j} S_{ij}x_{ij}
\quad\text{s.a.}\quad
\sum_j x_{ij}=1\ \forall i,\qquad \sum_i x_{ij}=1\ \forall j.
$$

Convertido a minimización QUBO con penalizaciones cuadráticas (igual que en el notebook
base):

$$
E(x)= -\sum_{i,j} S_{ij}x_{ij}
+\lambda_A\sum_i\Big(\sum_j x_{ij}-1\Big)^2
+\lambda_B\sum_j\Big(\sum_i x_{ij}-1\Big)^2.
$$

Las penalizaciones se calculan automáticamente en el notebook como
$\lambda = \lceil 4\max_{ij}|S_{ij}|+1\rceil$, suficientemente grandes para que el mínimo
del QUBO coincida con la asignación factible de mayor score.

## 7. Resultados (ejecución local, semilla 2026)

- **Óptimo clásico exacto** (fuerza bruta sobre las $2^{16}$ configuraciones y verificado
  por permutaciones): **CDMX → Terciarias, Nuevo León → Secundarias, Jalisco →
  Primarias, Estado de México → ISPN**, con score total **137.9** (energía QUBO
  −137.9). La asignación es factible (cumple las 8 restricciones de una-vez-cada-uno).
- **QAOA local ($p=1$, 2000 shots simulados):** la mejor muestra observada coincide
  exactamente con el óptimo clásico (score 137.9, factible). La probabilidad ideal de
  muestrear una solución factible fue de ≈2.1 % y la de muestrear directamente el óptimo
  clásico fue de ≈0.09 %, típico de una capa QAOA poco profunda ($p=1$) sobre un espacio
  de 16 qubits con mixer estándar (no preserva restricciones).
- **Hardware real:** no se ejecutó (sección avanzada opcional, desactivada).

## 8. Interpretación mínima (puntos 1–10 pedidos por el notebook)

1. **Mejor asignación encontrada:** CDMX→Terciarias, Nuevo León→Secundarias,
   Jalisco→Primarias, Estado de México→ISPN.
2. **Score en el dominio:** 137.9 puntos porcentuales acumulados (suma de las 4
   participaciones asignadas).
3. **¿Cumple restricciones?** Sí: cada entidad recibe un único grupo y cada grupo se usa
   una sola vez.
4. **¿QAOA local observó el óptimo clásico?** Sí, en la mejor muestra de 2000 shots.
5. **Frecuencia de soluciones factibles:** ideal ≈2.1 % de la distribución QAOA (el
   mixer no preserva restricciones, así que la mayoría de estados muestreados no son
   permutaciones válidas).
6. **Limitaciones del modelo 4×4:** solo captura 4 de las 32 entidades y los 4 grupos más
   agregados de actividad económica; ignora dinámica temporal, informalidad y
   heterogeneidad dentro de cada grupo (p. ej. "servicios" incluye desde comercio hasta
   servicios financieros).
7. **Si el dataset creciera:** con más entidades o con los 20 sectores SCIAN finos, el
   espacio de búsqueda crecería como $n!$ para el clásico exacto y $2^{n^2}$ qubits para
   el QUBO, obligando a usar QAOA con más capas ($p>1$), mixers que preserven
   restricciones, o descomposición del problema.
8. **Riesgos éticos y mitigación:** se usaron exclusivamente datos macroeconómicos
   agregados y públicos, sin información de personas ni empresas identificables. El
   resultado se presenta como un ejercicio ilustrativo de optimización, **no como una
   recomendación real de política de fomento económico regional**.
9. **Hardware real:** no aplica, no se ejecutó en esta entrega.
10. **Reparación clásica:** no aplica, no se usó postprocesamiento híbrido en esta
    entrega.

## 9. Advertencia ética y de alcance

Este ejercicio usa datos oficiales de PIB estatal únicamente para construir una instancia
de matching pequeña y pedagógica. **La salida de QAOA no debe interpretarse como una
recomendación real de política pública, inversión o asignación de recursos entre
entidades federativas.** Cualquier decisión de política económica regional real requiere
análisis mucho más completo que el que permite un modelo 4×4.

## 10. Contenido del repositorio

```text
.
├── data/
│   └── dataset.csv     # a_id, a_nombre, b_id, b_nombre, score (INEGI PIBE 2024)
├── proyecto_final_ss26.ipynb    # notebook con A_df/B_df/S reales + QUBO + QAOA local
└── README.md
```

El notebook detecta automáticamente `data/dataset_real_4x4.csv` y también construye
`A_df`, `B_df` y `S` de forma explícita en las celdas de la sección "Dataset real: PIB
estatal por grandes actividades económicas", con la fuente puntual de cada fila citada en
comentarios de código.

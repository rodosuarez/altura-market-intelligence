# ALTURA MARKET INTELLIGENCE: plan de desarrollo

> Aplicación web de Altura Marketplace que analiza exportaciones de Helium 10 (Xray y Cerebro) y genera dos informes PDF: uno **comercial** y otro **técnico**.
> Proyecto **totalmente independiente de ALTURA CONTROL**: repositorio, dependencias, configuración, marca y despliegue propios, sin código ni datos compartidos.

---

## 1. Estado actual del repositorio

- Repositorio `rodosuarez/altura-market-intelligence` **vacío**: no hay commits, ficheros ni ramas remotas.
- Se trabaja en la rama `claude/dazzling-dirac-1fat6g`.
- Entorno disponible: Node 22 y npm 10.
- No hay ninguna referencia a ALTURA CONTROL, así que se parte de cero.

## 2. Principios del proyecto

| Principio | Cómo se aplica |
|---|---|
| Sin APIs de pago | Todo el análisis y la redacción de conclusiones se basan en reglas y plantillas propias, sin IA externa. |
| Sin base de datos | Los datos solo viven en memoria durante la sesión. Opcionalmente, `localStorage` para recordar preferencias (por ejemplo, el nombre del analista). |
| Sin integraciones externas | Los ficheros **no salen del navegador**: el parseo, el análisis y los PDF se generan en el cliente. Esto aporta privacidad y coste de servidor cero. |
| Sencillez | Un único flujo: subir → revisar → descargar. |
| Independencia | Un `package.json` propio, sin paquetes compartidos ni variables de entorno o autenticación comunes con ALTURA CONTROL. |

## 3. Stack técnico (todo gratuito y de código abierto)

| Necesidad | Librería | Motivo |
|---|---|---|
| Framework | **Next.js (App Router) + TypeScript estricto** | Requisito. Se puede desplegar como sitio estático (`output: 'export'`). |
| Estilos y UI | **Tailwind CSS** + componentes propios (opcional: shadcn/ui, MIT) | Aspecto profesional y consistente. |
| CSV | **PapaParse** (MIT) | Robusto: detecta el delimitador (`,` o `;`), la codificación y las comillas. |
| Excel (.xlsx/.xls) | **SheetJS CE** (Apache 2.0), instalado desde su CDN oficial | La versión de npm está desactualizada. Alternativa: `read-excel-file`. |
| Validación | **Zod** | Tipado y validación de filas. |
| Gráficos en pantalla | **Recharts** (MIT) | Para la vista previa interactiva. |
| PDF | **@react-pdf/renderer** (MIT) | Maquetación con componentes React, en el navegador y con tipografías embebidas. |
| Gráficos en PDF | Componentes SVG propios (`<Svg>` de react-pdf) | Gráficos vectoriales nítidos, sin conversión a imagen. |
| Tests | **Vitest** + ficheros de ejemplo | Para el motor de análisis y el parseo. |
| Calidad | ESLint + Prettier | — |

## 4. Arquitectura

```
src/
  app/                    # Rutas Next.js: inicio y flujo de análisis
  components/             # UI: subida de ficheros, KPIs, tablas, gráficos
  lib/
    ingest/               # Lectura CSV/XLSX, detección Xray/Cerebro, mapeo de columnas
    normalize/            # Números europeos/US, monedas, "N/A", fechas
    analysis/             # Motor de métricas (funciones puras y testeables)
    scoring/              # Índices de oportunidad y de barreras de entrada
    insights/             # Reglas → textos de conclusiones en español
  pdf/
    shared/               # Tema, tipografías, cabecera/pie, gráficos SVG
    commercial/           # Informe comercial
    technical/            # Informe técnico
  types/                  # Modelos de dominio (Product, Keyword, MarketAnalysis…)
tests/fixtures/           # Exportaciones reales anonimizadas de Helium 10
```

Flujo de datos: `Fichero → parse → detectar tipo → mapear columnas → normalizar → validar → MarketDataset → análisis → MarketAnalysis → (UI | PDF comercial | PDF técnico)`.
Ambos PDF consumen el **mismo objeto `MarketAnalysis`**, de modo que las cifras son coherentes entre los dos informes y la pantalla.

### Columnas esperadas (se confirmarán con exportaciones reales)

- **Xray**: Product Details/Title, ASIN, Brand, Price, ASIN Sales, Parent Level Sales, ASIN Revenue, Parent Level Revenue, BSR, Category, Seller Country, Active Sellers, Fees, Ratings, Review Count, Review Velocity, Images, Fulfillment (FBA/FBM/AMZ), Size Tier, Creation Date/Age, Variation Count, Sponsored.
- **Cerebro**: Keyword Phrase, Cerebro IQ Score, Search Volume, Search Volume Trend, Keyword Sales, Suggested PPC Bid, Sponsored ASINs, Competing Products, CPR, Title Density, Organic/Sponsored Rank, Competitor Rank (avg), Ranking Competitors, Word Count.

Se usará un **diccionario de alias** (inglés y español, variantes por versión de Helium 10 y por moneda, como "Price €" o "Price $"). Si falta una columna, la métrica correspondiente se omite y se avisa al usuario, sin romper el informe.

## 5. Métricas y lógica de análisis

**Mercado (Xray)**
- Facturación y unidades mensuales estimadas del nicho: total, media y mediana por producto.
- **Top 3 y Top 10**: cuota de facturación, ventas, precio medio, reseñas y antigüedad.
- Concentración: cuota por marca, índice HHI, número de marcas relevantes y presencia de Amazon como vendedor.
- Precios: rango, media, mediana, distribución por tramos y relación precio–ventas.
- Reseñas y valoraciones: distribución, mediana y "reseñas necesarias para entrar en el Top 10".
- Logística y origen: % FBA/FBM/AMZ y origen de los vendedores (por ejemplo, % China).
- Dinamismo: productos de menos de 12 meses con ventas relevantes (indica si es posible entrar).
- Eficiencia: facturación por reseña y estimación de márgenes con las comisiones (si vienen en el fichero).

**Demanda (Cerebro)**
- Volumen de búsqueda total y por palabra clave, tendencia y palabras principales frente a la cola larga.
- Competencia por palabra clave: productos competidores, densidad de título, ASINs patrocinados y puja PPC sugerida.
- **Oportunidades**: alto volumen con baja densidad de título o pocos competidores.
- Agrupación simple por raíces o n‑gramas (por ejemplo, "botella acero", "botella niños").
- CPR (unidades necesarias para posicionar) como estimación del coste de lanzamiento.

**Índices (0–100, con pesos transparentes y documentados)**
- **Índice de oportunidad**: demanda, crecimiento, concentración inversa, barrera de reseñas, coste PPC y precio.
- **Índice de barreras de entrada**: reseñas del Top 10, concentración de marca, presencia de Amazon, CPR y puja PPC.
- Semáforo y veredicto: *Recomendable / Con cautela / No recomendable*.

**Conclusiones automáticas**: un motor de reglas genera párrafos en español a partir de los umbrales (por ejemplo, "El 68 % de la facturación se concentra en 3 marcas…"). Habrá dos registros de lenguaje: **sencillo** para el informe comercial y **técnico** para el interno.

## 6. Contenido de los informes

**Informe comercial (para empresas, de 6 a 10 páginas)**
1. Portada (nicho, marketplace, fecha, empresa cliente, marca Altura).
2. Resumen ejecutivo en una página: veredicto, 4–6 KPIs clave y un gráfico.
3. Tamaño del mercado y facturación estimada.
4. Demanda: qué busca el cliente y las palabras clave principales (gráfico).
5. Competidores principales (Top 10 resumido y gráfico de cuotas).
6. Precios (gráfico de tramos).
7. Oportunidades detectadas.
8. Conclusiones y próximos pasos con Altura Marketplace (llamada a la acción).
9. Nota metodológica breve (las cifras son estimaciones de Helium 10).

**Informe técnico (para Altura, de 15 a 25 páginas)**
1. Portada y ficha del análisis: ficheros, fechas, filas válidas y descartadas, avisos.
2. Resumen de índices (oportunidad y barreras) con su desglose.
3. Ventas y facturación: totales, distribución y curva de Pareto.
4. Top 3 en detalle y Top 10 en tabla completa.
5. Marcas: cuotas, HHI y presencia de Amazon.
6. Precios: estadística, tramos, precio–ventas y precio recomendado de entrada.
7. Reseñas, valoraciones, antigüedad y nuevos entrantes.
8. Logística, origen de vendedores y comisiones.
9. Palabras clave: tabla de las 30–50 principales, clusters, oportunidades, PPC y CPR.
10. Competencia y barreras de entrada.
11. Estrategia recomendada: precio objetivo, reseñas objetivo, palabras clave para el listing y el PPC, presupuesto estimado de lanzamiento y riesgos.
12. Anexo: listado completo de productos y glosario.

## 7. Experiencia de usuario

1. **Inicio**: explicación breve y una zona de arrastrar y soltar (admite uno o varios ficheros Xray y/o Cerebro).
2. **Revisión**: tipo detectado, filas leídas, columnas reconocidas, avisos y vista previa de la tabla. Opción de excluir productos atípicos.
3. **Datos del informe**: nombre del nicho o producto, marketplace y moneda, empresa cliente, analista y logo opcional.
4. **Panel**: KPIs, gráficos y veredicto en pantalla.
5. **Descarga**: botones "Informe comercial (PDF)" e "Informe técnico (PDF)".

Diseño corporativo sobrio, responsive y accesible, con mensajes de error claros en español.

## 8. Fases de desarrollo

| Fase | Objetivo | Entregables | Criterio de aceptación |
|---|---|---|---|
| **0. Base del proyecto** | Esqueleto profesional | Next.js + TS estricto, Tailwind, ESLint/Prettier, Vitest, estructura de carpetas, tema de marca, README | `npm run build`, `lint` y `test` pasan; la página de inicio se ve con la marca |
| **1. Ingesta de datos** | Leer cualquier exportación de Xray o Cerebro | Subida de ficheros, parseo CSV/XLSX, detección automática del tipo, alias de columnas, normalización numérica, validación con Zod, avisos | Las exportaciones reales de ejemplo se leen sin errores; tests con fixtures |
| **2. Motor de análisis** | Calcular todas las métricas | Modelos de dominio, métricas de mercado y palabras clave, Top 3/10, marcas, precios, índices, motor de conclusiones | Tests unitarios con resultados verificados a mano sobre las fixtures |
| **3. Interfaz y panel** | Flujo completo en pantalla | Pantallas de revisión, datos del informe y panel con Recharts | Un usuario no técnico completa el flujo sin ayuda |
| **4. Informe comercial PDF** | Primer entregable para clientes | Plantilla react-pdf, gráficos SVG, tipografías y textos sencillos | PDF legible, con la marca y cifras idénticas a las del panel |
| **5. Informe técnico PDF** | Informe interno exhaustivo | Todas las secciones del punto 6, tablas paginadas y anexos | Cubre todas las secciones y funciona con conjuntos de datos grandes |
| **6. Pulido y despliegue** | Calidad de producción | Web Worker para ficheros grandes, estados de carga y error, accesibilidad, pruebas de extremo a extremo con datos reales, despliegue (Vercel gratuito o hosting estático), guía de uso | Funciona con ficheros reales de varios nichos; documentación completa |

Cada fase se entregará en commits independientes y se revisará antes de pasar a la siguiente.

## 9. Riesgos y mitigación

- **Formato de Helium 10 cambiante o en otro idioma** → diccionario de alias, detección tolerante y avisos claros.
- **Formatos numéricos europeos y monedas** → normalizador dedicado y probado.
- **Ficheros grandes** → parseo en un Web Worker.
- **Las cifras de Helium 10 son estimaciones** → nota metodológica en ambos informes.
- **Peso y velocidad de generación del PDF** → `@react-pdf/renderer` cargado bajo demanda (import dinámico).

## 10. Información necesaria antes de empezar

1. **Exportaciones reales de ejemplo** de Xray y Cerebro (idealmente, una de cada en CSV y en XLSX, y si es posible en inglés y en español).
2. **Marketplace principal** (¿Amazon.es en EUR?) y si se necesitan otros.
3. **Identidad visual**: logo, colores corporativos y tipografía de Altura Marketplace.
4. **Cierre del informe comercial**: datos de contacto o la llamada a la acción que deben aparecer.
5. **Plataforma de despliegue** preferida (recomendada: Vercel, plan gratuito).

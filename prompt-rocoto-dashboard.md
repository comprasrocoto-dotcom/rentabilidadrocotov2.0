# Prompt: Construir/ajustar el dashboard "Rocoto" (mismas reglas que Casa de Nadie y Arrebatao)

Voy a construir el dashboard de rentabilidad para la marca **"Rocoto"**, con la misma arquitectura que ya usamos en Casa de Nadie y Arrebatao. El Excel/Google Sheet tiene estas hojas: `VENTAS`, `COMPRAS`, `TRASLADOS`, `FORMATO INVENTARIOS`, `RENTABILIDAD`. **No existe una hoja "MERCANCIA VENDIDA" precalculada** — la Mercancía Vendida se debe calcular en vivo (JS), igual que hicimos en Arrebatao.

## 0. Lección aprendida — matching de columnas a prueba de tildes y renombres

En Arrebatao perdimos varias horas porque el código buscaba nombres de columna exactos ("Almacen" sin tilde) y el Excel real tenía "Almacén" con tilde — `"almacén".includes("almacen")` da `false` en JS porque la tilde es un carácter distinto, no una variante de mayúscula/minúscula. Además, la hoja de inventarios cambió de nombre de columna varias veces ("Centro de Costos" → "CC", "Costo Ajuste" → "Total") sin avisar.

Por eso, para Rocoto:
- Usa siempre **prefijos cortos sin tildes** en las búsquedas de columna (ej. `["Almac"]` en vez de `["Almacen"]`, `["Artic"]`... en realidad usa `["Art"]` que ya es seguro), para que sobreviva a variaciones de tilde/mayúsculas.
- Para columnas que podrían renombrarse, busca **varios alias a la vez** (ej. `["Centro","CC"]` para Centro de Costos; `["Costo Ajuste","Total"]` para el valor de inventario).
- No hagas que el parseo dependa de una columna que podría desaparecer (ej. no uses `r.estado` como filtro obligatorio para aceptar una fila — úsalo solo si existe, pero no descartes filas por su ausencia).

## 1. Estructura de columnas actual de Rocoto (verificada en el Excel)

- **VENTAS**: `Establecimiento`, `Tarifa Venta`, `Centro de costos`, `Tipo Documento`, `Fecha Doc`, `Mes`, `Familia`, `Artículo`, `Uds.V`, `Base`, `Impuestos`, `Neto`.
- **COMPRAS**: `Fecha`, `Mes`, `Almacén`, `Proveedor`, `Familia`, `CC`, `Artículo`, `Uds.C`, `Total Neto`.
- **TRASLADOS** (nueva, no existía en Arrebatao): `Fecha Doc`, `Mes`, `Almacén`, `Centro de Costos`, `Familia`, `Artículo`, `Tipo Documento`, `Motivo Traspaso`, `Tipo Movimiento Stock`, `Variación Stock`, `Coste Unitario`, `Coste Variación`.
- **FORMATO INVENTARIOS**: `FECHA`, `Mes`, `Almacen`, `FAMILIA`, `Centro de Costos`, `Artículo`, `Coste Línea`, `Stock a Fecha`, `Variación Stock`, `Stock Inventario`, `Costo Ajuste`, `Ajuste`, `Estado`.

## 2. Reglas de negocio (idénticas a Arrebatao)

1. **Ventas Excluidas**: excluir de Ventas Totales/Base Neta solo las filas con Centro de Costos = **"Otros"** (y su variante "Otro"), sumándolas aparte en una tarjeta "Ventas Excluidas". Los descuentos (Comercial, Socios, Empleados, Cortesía, Cumpleaños, Invitación) **NO se excluyen** — ya vienen con Base negativa en el Excel, así que sumarlos tal cual ya resta correctamente. **OJO**: no filtres filas por `Base>0` o `Neto>0` — eso fue el bug que nos costó una sesión entera en Arrebatao, porque borra los descuentos (que son negativos) antes de que lleguen al cálculo. Usa solo `Base !== 0` para descartar basura.

2. **⚠️ Calidad de datos a revisar antes de programar la exclusión**: la columna "Centro de costos" de VENTAS en Rocoto tiene basura mezclada — valores como `"Factura venta electrónica"`, `"Abono factura venta electrónica"`, `"Factura venta contingencia"` que parecen ser en realidad valores de "Tipo Documento" que se filtraron por error a la columna de Centro de Costos. Antes de asumir que todo lo que no sea "Bar"/"Cocina"/"Otros" es un descuento válido, audita cuántas filas y qué valor de Base tienen esas categorías raras — si son pocas filas y bajo valor, probablemente son ruido de captura y no afectan el total; si son muchas, avísame antes de decidir cómo tratarlas.

3. **Compras**: excluir solo Centro de Costos = **"Otros"/"Otro"** del cálculo de Compras para Mercancía Vendida. Todo lo demás (Material de Aseo, Material de Empaque, Utensilios, Menaje, Gastos Administrativos, Publicidad, etc. — sin importar la variación de mayúsculas/espacios) suma normal.

4. **Inventario**: para el cálculo de Mercancía Vendida, solo cuentan Centro de Costos = **"Bar"** y **"Cocina"** (comparación exacta, ya que en esta hoja esos dos valores sí vienen limpios). Cualquier otro (Material de Aseo, Material de Empaque) se excluye de ese cálculo específico.

5. **Traslados (NUEVO — Rocoto sí tiene datos reales)**: en Arrebatao "Traslados" siempre daba $0/N/A porque no había datos. Acá sí hay una hoja TRASLADOS con movimientos entre sedes. Incorpórala a la fórmula de Mercancía Vendida:
   - Usa la columna **"Coste Variación"** (o Variación Stock × Coste Unitario si esa columna no cuadra) como el valor del traslado.
   - Determina con la columna "Tipo Movimiento Stock" si es una entrada o salida de esa sede/CC (revisa los valores únicos de esa columna para saber cómo distinguirlos).
   - Fórmula ajustada: **Mercancía Vendida = Inv. Inicial (heredado) + Compras (excluye Otros) + Traslados Entrada − Traslados Salida − Inv. Final**.
   - Muéstrame qué valores únicos tiene "Tipo Movimiento Stock" antes de asumir el signo, para no adivinar.

6. **Inventario heredado**: el Inventario Final de un mes (Bar/Cocina, por sede) se usa como Inventario Inicial del mes siguiente, igual que en Arrebatao y Casa de Nadie (recorrer meses en orden cronológico real, no alfabético).

7. **Valor de inventario**: usar la columna **"Costo Ajuste"** (no existe columna "Total" en esta hoja todavía, pero deja la búsqueda con ambos alias por si cambia como pasó en Arrebatao).

## 3. Al terminar

Muéstrame en qué funciones quedó cada punto, y especialmente:
- El desglose de filas/valores de las categorías raras de VENTAS (punto 2), antes de decidir si se excluyen o no.
- Los valores únicos de "Tipo Movimiento Stock" en TRASLADOS (punto 5), antes de fijar el signo en la fórmula.

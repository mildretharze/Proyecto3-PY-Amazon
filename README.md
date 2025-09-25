Proyecto 3: Amazon Sales & Reviews

1. Importar los datos
En esta sección cargamos los archivos CSV que contienen la información de productos y reseñas de Amazon.
Los datasets originales se llaman amazon - amazon_product.csv y amazon - amazon_review.csv.
Para evitar problemas con los nombres (espacios y guiones), los renombramos con nombres más simples.


codigo:
```python
# Importamos la librería necesaria
import pandas as pd  # "pd" es un alias, lo usaremos para llamar a pandas

# Subir archivos desde la computadora
from google.colab import files

uploaded = files.upload()  # Se abrirá un cuadro de diálogo para elegir los CSV

# Renombramos directamente al cargarlos
productos = pd.read_csv("amazon - amazon_product.csv")
resenas = pd.read_csv("amazon - amazon_review.csv")

# Guardamos una copia con nombres más simples para trabajar sin problemas
productos.to_csv("amazon_product.csv", index=False)
resenas.to_csv("amazon_review.csv", index=False)

# Ahora reabrimos usando los nombres limpios
productos = pd.read_csv("amazon_product.csv")
resenas = pd.read_csv("amazon_review.csv")

# Verificamos cuántas filas y columnas tiene cada dataset
print("Productos:", productos.shape)
print("Reseñas:", resenas.shape)
```

Exploración de las columnas en cada DataFrame
En esta sección verificamos las columnas disponibles en los dos DataFrames iniciales:
```python
productos: contiene la información de los productos.
resenas: contiene la información de las reseñas de usuarios.
```

codigo:
```python
# Ver columnas de cada DataFrame
print("Columnas de productos:")
print(productos.columns)

print("--------------------------------------------------")

print("Columnas de resenas:")
print(resenas.columns)
```

2. Identificar y manejar valores nulos
En esta sección:
Normalizamos los nombres de las columnas (todo en minúsculas, sin espacios).
Identificamos valores nulos en cada dataset.
Revisamos ejemplos de filas con nulos.
Calculamos cuántas filas tienen al menos un nulo.
Eliminamos columnas irrelevantes (img_link, product_link).
Rellenamos o estandarizamos datos:
about_product → reemplazado por "sin información" en caso de nulos.
rating_count → convertido a entero, limpiando nulos y valores no válidos.
Guardamos los archivos limpios para análisis posterior.

codigo:
```python
# 0) Normalizar nombres de columnas para evitar problemas de mayúsculas/espacios
productos.columns = productos.columns.str.strip().str.lower().str.replace(' ', '_', regex=False)
resenas.columns  = resenas.columns.str.strip().str.lower().str.replace(' ', '_', regex=False)

# 1) Resumen de nulos: conteo y porcentaje
def null_summary(df, df_name):
    total = len(df)
    nulos = df.isnull().sum()
    pct = (nulos / total * 100).round(2)
    summary = pd.DataFrame({
        'column': nulos.index,
        'n_null': nulos.values,
        'pct_null': pct.values
    })
    summary = summary.sort_values(by='n_null', ascending=False).reset_index(drop=True)
    print(f"\n--- Resumen nulos: {df_name} (total filas: {total}) ---")
    display(summary)
    return summary

summary_prod = null_summary(productos, "productos")
summary_rev  = null_summary(resenas, "resenas")

# 2) Mostrar ejemplos (hasta 5) de filas donde cada columna tenga nulos (por dataset)
def show_null_examples(df, summary, df_name, n=5):
    cols_with_nulls = summary.loc[summary['n_null'] > 0, 'column'].tolist()
    if not cols_with_nulls:
        print(f"\nNo hay columnas con nulos en {df_name}.")
        return
    for col in cols_with_nulls:
        print(f"\n{df_name} - Ejemplos de filas con {col} nulo (hasta {n}):")
        display(df[df[col].isnull()].head(n))

show_null_examples(productos, summary_prod, "productos")
show_null_examples(resenas, summary_rev, "resenas")

# 3) Filas que tienen al menos 1 nulo (cantidad y %), y mostrar ejemplo
for df, name in [(productos, "productos"), (resenas, "resenas")]:
    filas_con_algun_nulo = df.isnull().any(axis=1).sum()
    pct_filas = (filas_con_algun_nulo / len(df) * 100).round(2)
    print(f"\n{name}: {filas_con_algun_nulo} filas ({pct_filas}%) con al menos 1 valor nulo.")
    display(df[df.isnull().any(axis=1)].head(5))

# 4) Guardar resumen a CSV
summary_prod.to_csv('null_summary_productos.csv', index=False)
summary_rev.to_csv('null_summary_resenas.csv', index=False)
print("\nResúmenes guardados: null_summary_productos.csv, null_summary_resenas.csv")
```

codigo:
```python
# -- LIMPIEZA DE DATOS AMAZON --
import pandas as pd

# 1) MOSTRAR ESTADO INICIAL
print("=== ESTADO INICIAL ===")
print(f"Productos: {productos.shape[0]} filas, {productos.shape[1]} columnas")
print(f"Reseñas: {resenas.shape[0]} filas, {resenas.shape[1]} columnas")

print("\n🔍 Nulos iniciales - Productos:")
print(productos.isnull().sum())
print("\n🔍 Nulos iniciales - Reseñas:")
print(resenas.isnull().sum())

# 2) LIMPIEZA DE DATOS
print("\n=== PROCESO DE LIMPIEZA ===")

# A. Productos: Rellenar nulos en about_product
productos['about_product'] = productos['about_product'].fillna("sin información")
print("✅ about_product: nulos remplazados por 'sin información'")

# B. Reseñas: Eliminar columnas irrelevantes
columnas_eliminadas = []
for columna in ['img_link', 'product_link']:
    if columna in resenas.columns:
        resenas = resenas.drop(columns=columna)
        columnas_eliminadas.append(columna)

if columnas_eliminadas:
    print(f"✅ Columnas eliminadas: {', '.join(columnas_eliminadas)}")
else:
    print("ℹ️ No se eliminaron columnas")

# C. Reseñas: Limpiar y convertir rating_count
if 'rating_count' in resenas.columns:
    # Contar nulos antes de limpiar
    nulos_antes = resenas['rating_count'].isnull().sum()

    # Limpieza paso a paso
    resenas['rating_count'] = (
        resenas['rating_count']
        .astype(str)  # Convertir todo a texto
        .str.replace(',', '')  # Quitar comas de miles
        .str.strip()  # Quitar espacios
        .replace('nan', '0')  # Remplazar texto 'nan' por '0'
        .replace('', '0')  # Remplazar cadenas vacías por '0'
    )

    # Convertir a numérico (los errores se convierten en NaN)
    resenas['rating_count'] = pd.to_numeric(resenas['rating_count'], errors='coerce')

    # Rellenar cualquier NaN restante con 0 y convertir a entero
    resenas['rating_count'] = resenas['rating_count'].fillna(0).astype(int)

    print(f"✅ rating_count: {nulos_antes} nulos convertidos a 0 y estandarizados")

# 3) VERIFICACIÓN FINAL
print("\n=== VERIFICACIÓN FINAL ===")
print(f"Productos: {productos.shape[0]} filas, {productos.shape[1]} columnas")
print(f"Reseñas: {resenas.shape[0]} filas, {resenas.shape[1]} columnas")

print("\n🔍 Nulos después de limpieza - Productos:")
print(productos.isnull().sum())
print("\n🔍 Nulos después de limpieza - Reseñas:")
print(resenas.isnull().sum())

# 4) MUESTRAS DE VALIDACIÓN
print("\n=== VALIDACIÓN DE CAMBIOS ===")

# Mostrar productos que tenían nulos y ahora tienen "sin información"
productos_sin_info = productos[productos['about_product'] == "sin información"]
print(f"📋 Productos con 'sin información': {len(productos_sin_info)}")

if len(productos_sin_info) > 0:
    print("Muestra de productos con descripción remplazada:")
    display(productos_sin_info[['product_id', 'product_name', 'about_product']].head(3))

# Mostrar estadísticas de rating_count
if 'rating_count' in resenas.columns:
    print(f"\n📊 Estadísticas de rating_count:")
    print(f"   Mínimo: {resenas['rating_count'].min()}")
    print(f"   Máximo: {resenas['rating_count'].max()}")
    print(f"   Promedio: {resenas['rating_count'].mean():.2f}")
    print(f"   Ceros: {(resenas['rating_count'] == 0).sum()}")

# 5) GUARDAR DATOS LIMPIOS (OPCIONAL)
productos.to_csv("amazon_productos_limpio.csv", index=False)
resenas.to_csv("amazon_resenas_limpio.csv", index=False)
print("\n💾 Archivos guardados: amazon_productos_limpio.csv, amazon_resenas_limpio.csv")

print("\n🎉 ¡Limpieza completada exitosamente!")
```

Resultados después de la limpieza:
about_product: sin nulos (rellenados con "sin información").
img_link, product_link: eliminadas por irrelevantes.
rating_count: transformado a entero, nulos y valores inconsistentes convertidos a 0.
📊 Verificación final:
Productos: 1351 filas, 19 columnas → 0 nulos.
Reseñas: 1194 filas, 16 columnas → solo queda 1 nulo en rating_original_numeric.
💾 Se guardaron los datasets limpios:
amazon_productos_limpio.csv
amazon_resenas_limpio.csv
🎉 Limpieza completada exitosamente.

3. Identificar y manejar valores duplicados
En esta sección identificamos y gestionamos los valores duplicados en las tablas productos y reseñas, aplicando criterios claros para su depuración.

Análisis de duplicados
📦 Productos
Duplicados completos: 106
product_id duplicados: 118
Se detectaron múltiples product_id duplicados, como por ejemplo:

codigo:
```python
# =============================================================================
# ANÁLISIS DE VALORES DUPLICADOS
# =============================================================================

print("🔍 INICIANDO BÚSQUEDA DE DUPLICADOS")
print("=" * 50)

# 1. DUPLICADOS EN TABLA PRODUCTOS
print("\n📦 PRODUCTOS - Análisis de duplicados:")
print("-" * 40)

# A) Duplicados completos (todas las columnas iguales)
duplicados_completos_prod = productos.duplicated().sum()
print(f"Duplicados completos: {duplicados_completos_prod}")

# B) Duplicados en columnas clave (product_id debería ser único)
duplicados_id_prod = productos['product_id'].duplicated().sum()
print(f"Duplicados en product_id: {duplicados_id_prod}")

# C) Mostrar los duplicados de product_id si existen
if duplicados_id_prod > 0:
    print("\n⚠️  PRODUCTOS DUPLICADOS (product_id):")
    ids_duplicados = productos[productos['product_id'].duplicated(keep=False)]['product_id'].unique()
    print(f"IDs duplicados: {ids_duplicados}")

    # Mostrar todas las filas con IDs duplicados
    productos_duplicados = productos[productos['product_id'].isin(ids_duplicados)]
    display(productos_duplicados.sort_values('product_id'))
else:
    print("✅ No hay product_id duplicados")

# 2. DUPLICADOS EN TABLA RESEÑAS
print("\n⭐ RESEÑAS - Análisis de duplicados:")
print("-" * 40)

# A) Duplicados completos
duplicados_completos_rev = resenas.duplicated().sum()
print(f"Duplicados completos: {duplicados_completos_rev}")

# B) Duplicados en columnas clave
duplicados_review_id = resenas['review_id'].duplicated().sum()
print(f"Duplicados en review_id: {duplicados_review_id}")

duplicados_user_product = resenas.duplicated(subset=['user_id', 'product_id']).sum()
print(f"Usuarios con múltiples reseñas del mismo producto: {duplicados_user_product}")

# C) Mostrar duplicados si existen
if duplicados_completos_rev > 0:
    print("\n⚠️  RESEÑAS DUPLICADAS COMPLETAS:")
    reseñas_duplicadas = resenas[resenas.duplicated(keep=False)]
    display(reseñas_duplicadas.sort_values('review_id'))

if duplicados_review_id > 0:
    print("\n⚠️  REVIEW_ID DUPLICADOS:")
    reviews_duplicados = resenas[resenas['review_id'].duplicated(keep=False)]
    display(reviews_duplicados.sort_values('review_id'))

# 3. ANÁLISIS DE DUPLICADOS PARCIALES
print("\n🔎 ANÁLISIS DE DUPLICADOS PARCIALES:")
print("-" * 40)

# A) Usuarios que más reseñan
usuarios_activos = resenas['user_id'].value_counts().head(5)
print("👥 Top 5 usuarios más activos:")
print(usuarios_activos)

# B) Productos más reseñados
productos_populares = resenas['product_id'].value_counts().head(5)
print("\n🏆 Top 5 productos más reseñados:")
print(productos_populares)

# 4. RESUMEN FINAL
print("\n" + "=" * 50)
print("📊 RESUMEN DE DUPLICADOS:")
print("=" * 50)

print(f"📦 PRODUCTOS:")
print(f"   - Duplicados completos: {duplicados_completos_prod}")
print(f"   - product_id duplicados: {duplicados_id_prod}")

print(f"\n⭐ RESEÑAS:")
print(f"   - Duplicados completos: {duplicados_completos_rev}")
print(f"   - review_id duplicados: {duplicados_review_id}")
print(f"   - Múltiples reseñas por usuario-producto: {duplicados_user_product}")

# 5. RECOMENDACIONES
print("\n💡 RECOMENDACIONES:")
print("-" * 30)

if duplicados_completos_prod > 0:
    print("❌ Eliminar duplicados completos de productos")

if duplicados_id_prod > 0:
    print("❌ Investigar product_id duplicados (deberían ser únicos)")

if duplicados_completos_rev > 0:
    print("❌ Eliminar reseñas duplicadas completas")

if duplicados_review_id > 0:
    print("❌ Investigar review_id duplicados (deberían ser únicos)")

if duplicados_user_product > 0:
    print("⚠️  Usuarios con múltiples reseñas del mismo producto - puede ser válido")

print("\n✅ Análisis de duplicados completado")
```

Limpieza de valores duplicados
Una vez identificados los duplicados, se aplicaron acciones de limpieza según criterio:

📦 Productos
Eliminación de duplicados en product_id.
Se conservó solo la primera ocurrencia de cada producto.
Resultado:
Duplicados eliminados: 118
Filas finales: 1,351
Sin duplicados restantes en product_id.

⭐ Reseñas
Eliminación de duplicados en review_id.
Se conservó solo la primera ocurrencia de cada reseña.
Resultado:
Duplicados eliminados: 271
Filas finales: 1,194
Sin duplicados restantes en review_id.

✅ Verificación final
Ambas tablas quedaron sin duplicados en sus claves únicas (product_id y review_id).
Se verificó que los datos conservados mantienen la integridad del análisis.

💾 Archivos exportados
Los datos limpios fueron guardados como:
```python
productos_sin_duplicados.csv
resenas_sin_duplicados.csv
```
Esto permite reutilizar los datasets ya depurados en los siguientes pasos.

📊 Resumen ejecutivo
Total de filas eliminadas: 389
Total de filas finales: 2,545
Porcentaje de datos conservados: 86.7%
🎯 La limpieza de duplicados se completó con éxito.

codigo:
```python
# =============================================================================
# PASO 1: ELIMINAR DUPLICADOS SEGÚN CRITERIO ESPECÍFICO
# =============================================================================

print("🧹 ELIMINANDO DUPLICADOS POR ID ÚNICO - PRIMERA OCURRENCIA")
print("=" * 60)

# Guardar el estado original para comparar
print("📊 ESTADO INICIAL:")
print(f"   Productos: {productos.shape[0]} filas, {productos.shape[1]} columnas")
print(f"   Reseñas: {resenas.shape[0]} filas, {resenas.shape[1]} columnas")

# -----------------------------------------------------------------------------
# A. ELIMINAR DUPLICADOS EN PRODUCTOS (por product_id)
# -----------------------------------------------------------------------------
print("\n" + "="*50)
print("📦 PROCESANDO PRODUCTOS - product_id")
print("="*50)

# Contar duplicados antes de eliminar
duplicados_prod_antes = productos['product_id'].duplicated().sum()
print(f"Duplicados encontrados en product_id: {duplicados_prod_antes}")

if duplicados_prod_antes > 0:
    # Mostrar algunos ejemplos de IDs duplicados
    ids_duplicados_prod = productos[productos['product_id'].duplicated(keep=False)]['product_id'].unique()[:5]
    print(f"Ejemplos de product_id duplicados: {ids_duplicados_prod}")

    # Mostrar cómo se ven los duplicados para un ID específico
    ejemplo_id = ids_duplicados_prod[0] if len(ids_duplicados_prod) > 0 else None
    if ejemplo_id:
        print(f"\n🔍 Ejemplo de duplicados para product_id '{ejemplo_id}':")
        duplicados_ejemplo = productos[productos['product_id'] == ejemplo_id]
        display(duplicados_ejemplo)

    # ELIMINAR DUPLICADOS (mantener la primera ocurrencia)
    productos = productos.drop_duplicates(subset='product_id', keep='first')
    print("✅ Duplicados eliminados - se mantuvo la primera ocurrencia")
else:
    print("✅ No hay duplicados en product_id")

# Verificar resultado
duplicados_prod_despues = productos['product_id'].duplicated().sum()
print(f"Duplicados restantes en product_id: {duplicados_prod_despues}")

# -----------------------------------------------------------------------------
# B. ELIMINAR DUPLICADOS EN RESEÑAS (por review_id)
# -----------------------------------------------------------------------------
print("\n" + "="*50)
print("⭐ PROCESANDO RESEÑAS - review_id")
print("="*50)

# Contar duplicados antes de eliminar
duplicados_rev_antes = resenas['review_id'].duplicated().sum()
print(f"Duplicados encontrados en review_id: {duplicados_rev_antes}")

if duplicados_rev_antes > 0:
    # Mostrar algunos ejemplos de IDs duplicados
    ids_duplicados_rev = resenas[resenas['review_id'].duplicated(keep=False)]['review_id'].unique()[:5]
    print(f"Ejemplos de review_id duplicados: {ids_duplicados_rev}")

    # Mostrar cómo se ven los duplicados para un ID específico
    ejemplo_id_rev = ids_duplicados_rev[0] if len(ids_duplicados_rev) > 0 else None
    if ejemplo_id_rev:
        print(f"\n🔍 Ejemplo de duplicados para review_id '{ejemplo_id_rev}':")
        duplicados_ejemplo_rev = resenas[resenas['review_id'] == ejemplo_id_rev]
        display(duplicados_ejemplo_rev)

    # ELIMINAR DUPLICADOS (mantener la primera ocurrencia)
    resenas = resenas.drop_duplicates(subset='review_id', keep='first')
    print("✅ Duplicados eliminados - se mantuvo la primera ocurrencia")
else:
    print("✅ No hay duplicados en review_id")

# Verificar resultado
duplicados_rev_despues = resenas['review_id'].duplicated().sum()
print(f"Duplicados restantes en review_id: {duplicados_rev_despues}")

# =============================================================================
# PASO 2: VERIFICACIÓN FINAL
# =============================================================================
print("\n" + "="*60)
print("✅ VERIFICACIÓN FINAL")
print("="*60)

# Mostrar resultados finales
filas_finales_prod = productos.shape[0]
filas_finales_rev = resenas.shape[0]

print(f"📦 PRODUCTOS FINALES: {filas_finales_prod} filas")
print(f"   - Duplicados eliminados: {duplicados_prod_antes}")
print(f"   - Duplicados restantes: {duplicados_prod_despues}")

print(f"\n⭐ RESEÑAS FINALES: {filas_finales_rev} filas")
print(f"   - Duplicados eliminados: {duplicados_rev_antes}")
print(f"   - Duplicados restantes: {duplicados_rev_despues}")

# Verificar que no quedan duplicados en las claves únicas
if duplicados_prod_despues == 0 and duplicados_rev_despues == 0:
    print("\n🎉 ¡ÉXITO! No hay duplicados restantes en las claves únicas")
else:
    print("\n⚠️  Aún quedan duplicados.可能需要 revisión manual")

# =============================================================================
# PASO 3: GUARDAR RESULTADOS
# =============================================================================
print("\n" + "="*60)
print("💾 GUARDANDO RESULTADOS")
print("="*60)

# Guardar los datos limpios
productos.to_csv("productos_sin_duplicados.csv", index=False)
resenas.to_csv("resenas_sin_duplicados.csv", index=False)

print("✅ Archivos guardados:")
print("   - productos_sin_duplicados.csv")
print("   - resenas_sin_duplicados.csv")

print("\n🔍 Para verificar, puedes cargar los archivos guardados:")
print("   productos_verificar = pd.read_csv('productos_sin_duplicados.csv')")
print("   resenas_verificar = pd.read_csv('resenas_sin_duplicados.csv')")

# =============================================================================
# PASO 4: RESUMEN EJECUTIVO
# =============================================================================
print("\n" + "="*60)
print("📊 RESUMEN EJECUTIVO")
print("="*60)

total_eliminado = duplicados_prod_antes + duplicados_rev_antes
total_final = filas_finales_prod + filas_finales_rev

print(f"📈 Total de filas eliminadas: {total_eliminado}")
print(f"📈 Total de filas finales: {total_final}")
print(f"📈 Porcentaje de datos conservados: {(total_final/(total_final + total_eliminado))*100:.1f}%")

print("\n¡Proceso completado! 🎯")
```

codigo:
```python
# =============================================================================
# ANÁLISIS DE VALORES DUPLICADOS
# =============================================================================

print("🔍 INICIANDO BÚSQUEDA DE DUPLICADOS")
print("=" * 50)

# 1. DUPLICADOS EN TABLA PRODUCTOS
print("\n📦 PRODUCTOS - Análisis de duplicados:")
print("-" * 40)

# A) Duplicados completos (todas las columnas iguales)
duplicados_completos_prod = productos.duplicated().sum()
print(f"Duplicados completos: {duplicados_completos_prod}")

# B) Duplicados en columnas clave (product_id debería ser único)
duplicados_id_prod = productos['product_id'].duplicated().sum()
print(f"Duplicados en product_id: {duplicados_id_prod}")

# C) Mostrar los duplicados de product_id si existen
if duplicados_id_prod > 0:
    print("\n⚠️  PRODUCTOS DUPLICADOS (product_id):")
    ids_duplicados = productos[productos['product_id'].duplicated(keep=False)]['product_id'].unique()
    print(f"IDs duplicados: {ids_duplicados}")

    # Mostrar todas las filas con IDs duplicados
    productos_duplicados = productos[productos['product_id'].isin(ids_duplicados)]
    display(productos_duplicados.sort_values('product_id'))
else:
    print("✅ No hay product_id duplicados")

# 2. DUPLICADOS EN TABLA RESEÑAS
print("\n⭐ RESEÑAS - Análisis de duplicados:")
print("-" * 40)

# A) Duplicados completos
duplicados_completos_rev = resenas.duplicated().sum()
print(f"Duplicados completos: {duplicados_completos_rev}")

# B) Duplicados en columnas clave
duplicados_review_id = resenas['review_id'].duplicated().sum()
print(f"Duplicados en review_id: {duplicados_review_id}")

duplicados_user_product = resenas.duplicated(subset=['user_id', 'product_id']).sum()
print(f"Usuarios con múltiples reseñas del mismo producto: {duplicados_user_product}")

# C) Mostrar duplicados si existen
if duplicados_completos_rev > 0:
    print("\n⚠️  RESEÑAS DUPLICADAS COMPLETAS:")
    reseñas_duplicadas = resenas[resenas.duplicated(keep=False)]
    display(reseñas_duplicadas.sort_values('review_id'))

if duplicados_review_id > 0:
    print("\n⚠️  REVIEW_ID DUPLICADOS:")
    reviews_duplicados = resenas[resenas['review_id'].duplicated(keep=False)]
    display(reviews_duplicados.sort_values('review_id'))

# 3. ANÁLISIS DE DUPLICADOS PARCIALES
print("\n🔎 ANÁLISIS DE DUPLICADOS PARCIALES:")
print("-" * 40)

# A) Usuarios que más reseñan
usuarios_activos = resenas['user_id'].value_counts().head(5)
print("👥 Top 5 usuarios más activos:")
print(usuarios_activos)

# B) Productos más reseñados
productos_populares = resenas['product_id'].value_counts().head(5)
print("\n🏆 Top 5 productos más reseñados:")
print(productos_populares)

# 4. RESUMEN FINAL
print("\n" + "=" * 50)
print("📊 RESUMEN DE DUPLICADOS:")
print("=" * 50)

print(f"📦 PRODUCTOS:")
print(f"   - Duplicados completos: {duplicados_completos_prod}")
print(f"   - product_id duplicados: {duplicados_id_prod}")

print(f"\n⭐ RESEÑAS:")
print(f"   - Duplicados completos: {duplicados_completos_rev}")
print(f"   - review_id duplicados: {duplicados_review_id}")
print(f"   - Múltiples reseñas por usuario-producto: {duplicados_user_product}")

# 5. RECOMENDACIONES
print("\n💡 RECOMENDACIONES:")
print("-" * 30)

if duplicados_completos_prod > 0:
    print("❌ Eliminar duplicados completos de productos")

if duplicados_id_prod > 0:
    print("❌ Investigar product_id duplicados (deberían ser únicos)")

if duplicados_completos_rev > 0:
    print("❌ Eliminar reseñas duplicadas completas")

if duplicados_review_id > 0:
    print("❌ Investigar review_id duplicados (deberían ser únicos)")

if duplicados_user_product > 0:
    print("⚠️  Usuarios con múltiples reseñas del mismo producto - puede ser válido")

print("\n✅ Análisis de duplicados completado")
```

4. Análisis completo del formato
En esta sección se realizó una revisión exhaustiva de los tipos de datos y formatos en cada columna de los datasets productos y reseñas.
El objetivo fue detectar inconsistencias y preparar las variables para un análisis posterior.

📦 Productos
Todas las columnas estaban inicialmente en formato object (texto), incluyendo precios y porcentajes.
Variables como discounted_price, actual_price y discount_percentage contenían símbolos no numéricos (₹, %, comas de miles).
Se detectó la necesidad de conversión de estas columnas a valores numéricos (float).
La columna about_product mostró gran longitud de texto, útil para futuros análisis de procesamiento de lenguaje natural (NLP).

⭐ Reseñas
La mayoría de las columnas son de tipo object.
La variable rating estaba almacenada como texto, aunque corresponde a valores numéricos.
rating_count ya estaba correctamente en formato numérico (int64).
Columnas de texto como review_content y review_title mostraron alta longitud de caracteres, indicando descripciones ricas en información.

🚨 Problemas detectados
Productos:
discounted_price → contiene ₹ y comas.
actual_price → contiene ₹ y comas.
discount_percentage → contiene %.
Reseñas:
rating → debería convertirse a decimal.

🎯 Conversión aplicada (Paso 1)
Se realizó la conversión de la columna discounted_price:
Eliminación del símbolo ₹ y comas.
Conversión de object → float64.
Verificación de la conversión mostrando ejemplos antes y después.
Exportación del dataset limpio (productos_paso1.csv).

✅ Verificación de la conversión
Tipo de dato anterior: object
Tipo de dato nuevo: float64
Ejemplo de conversión:
Antes: ₹399
Después: 399.0

💾 Resultado
Archivo exportado: productos_paso1.csv
El proceso se completó sin errores.
La columna discounted_price quedó lista para análisis estadístico y numérico.

codigo:
```python
# =============================================================================
# ANÁLISIS COMPLETO DEL FORMATO DE DATOS
# =============================================================================

print("🔍 ANÁLISIS COMPLETO DEL FORMATO DE DATOS")
print("=" * 60)

# -----------------------------------------------------------------------------
# 1. FUNCIÓN PARA ANALIZAR UNA COLUMNA
# -----------------------------------------------------------------------------
def analizar_columna(df, nombre_columna, df_nombre):
    """Analiza profundamente una columna y muestra ejemplos"""
    print(f"\n📊 {df_nombre} - Columna: {nombre_columna}")
    print("-" * 50)

    # Información básica
    print(f"   Tipo de dato: {df[nombre_columna].dtype}")
    print(f"   Total de valores: {len(df[nombre_columna])}")
    print(f"   Valores nulos: {df[nombre_columna].isnull().sum()}")
    print(f"   Valores únicos: {df[nombre_columna].nunique()}")

    # Estadísticas según el tipo de dato
    if df[nombre_columna].dtype in ['int64', 'float64']:
        # Es numérico
        print(f"   Mínimo: {df[nombre_columna].min()}")
        print(f"   Máximo: {df[nombre_columna].max()}")
        print(f"   Promedio: {df[nombre_columna].mean():.2f}")
        print(f"   Mediana: {df[nombre_columna].median()}")

    elif df[nombre_columna].dtype == 'object':
        # Es texto/string
        longitudes = df[nombre_columna].astype(str).str.len()
        print(f"   Longitud promedio: {longitudes.mean():.1f} caracteres")
        print(f"   Longitud mínima: {longitudes.min()} caracteres")
        print(f"   Longitud máxima: {longitudes.max()} caracteres")

        # Mostrar los valores más frecuentes
        print(f"\n   Valores más frecuentes:")
        valores_frecuentes = df[nombre_columna].value_counts().head(3)
        for valor, conteo in valores_frecuentes.items():
            print(f"     '{valor}': {conteo} veces")

    # Mostrar ejemplos de valores
    print(f"\n   Ejemplos de valores:")
    ejemplos = df[nombre_columna].dropna().head(3).tolist()
    for i, ejemplo in enumerate(ejemplos, 1):
        print(f"     {i}. {repr(ejemplo)}")  # repr() muestra el formato exacto

    print(f"   ... y {len(df[nombre_columna]) - 3} más")

# -----------------------------------------------------------------------------
# 2. ANALIZAR TABLA PRODUCTOS
# -----------------------------------------------------------------------------
print("\n" + "="*60)
print("📦 ANÁLISIS DE LA TABLA PRODUCTOS")
print("="*60)

for columna in productos.columns:
    analizar_columna(productos, columna, "PRODUCTOS")

# -----------------------------------------------------------------------------
# 3. ANALIZAR TABLA RESEÑAS
# -----------------------------------------------------------------------------
print("\n" + "="*60)
print("⭐ ANÁLISIS DE LA TABLA RESEÑAS")
print("="*60)

for columna in resenas.columns:
    analizar_columna(resenas, columna, "RESEÑAS")

# -----------------------------------------------------------------------------
# 4. RESUMEN DE TIPOS DE DATOS
# -----------------------------------------------------------------------------
print("\n" + "="*60)
print("📋 RESUMEN DE TIPOS DE DATOS")
print("="*60)

print("\n📦 PRODUCTOS:")
for columna in productos.columns:
    print(f"   {columna}: {productos[columna].dtype}")

print("\n⭐ RESEÑAS:")
for columna in resenas.columns:
    print(f"   {columna}: {resenas[columna].dtype}")

# -----------------------------------------------------------------------------
# 5. DETECTAR POSIBLES PROBLEMAS DE FORMATO
# -----------------------------------------------------------------------------
print("\n" + "="*60)
print("🚨 DETECCIÓN DE POSIBLES PROBLEMAS")
print("="*60)

# Verificar columnas que deberían ser numéricas pero son texto
columnas_numericas_esperadas = ['discounted_price', 'actual_price', 'discount_percentage',
                               'rating', 'rating_count']

print("Columnas que podrían necesitar conversión numérica:")
for columna in columnas_numericas_esperadas:
    if columna in productos.columns and productos[columna].dtype == 'object':
        print(f"   📦 {columna} (debería ser numérica)")
    if columna in resenas.columns and resenas[columna].dtype == 'object':
        print(f"   ⭐ {columna} (debería ser numérica)")

# -----------------------------------------------------------------------------
# 6. EJEMPLOS DETALLADOS DE VALORES
# -----------------------------------------------------------------------------
print("\n" + "="*60)
print("🔍 EJEMPLOS DETALLADOS POR COLUMNA")
print("="*60)

def mostrar_ejemplos_detallados(df, nombre_columna, n=5):
    """Muestra ejemplos detallados de una columna"""
    print(f"\n📋 {nombre_columna} - Ejemplos detallados:")
    valores = df[nombre_columna].dropna().head(n).tolist()
    for i, valor in enumerate(valores, 1):
        print(f"   {i}. {repr(valor)}")
        print(f"      Tipo: {type(valor)}")
        if isinstance(valor, str):
            print(f"      Longitud: {len(valor)} caracteres")
            if any(c in valor for c in ['₹', ',', '%']):
                print(f"      Contiene símbolos: {[c for c in ['₹', ',', '%'] if c in valor]}")

# Ejemplos de columnas clave
print("\n📦 PRODUCTOS - Ejemplos detallados:")
mostrar_ejemplos_detallados(productos, 'discounted_price')
mostrar_ejemplos_detallados(productos, 'actual_price')
mostrar_ejemplos_detallados(productos, 'discount_percentage')

print("\n⭐ RESEÑAS - Ejemplos detallados:")
mostrar_ejemplos_detallados(resenas, 'rating')
mostrar_ejemplos_detallados(resenas, 'rating_count')
```

Conversión de actual_price y discount_percentage
En esta etapa se abordaron las columnas restantes con problemas de formato numérico en el dataset productos.
Se aplicó la limpieza de símbolos y la conversión a valores decimales para garantizar consistencia.

📦 Variables procesadas
actual_price
Inicialmente en formato object (texto).
Contenía símbolos de rupias (₹) y comas de miles.
Se eliminó la simbología y se transformó a float64.
discount_percentage
Inicialmente en formato object.
Contenía el símbolo %.
Se eliminó el símbolo y se convirtió a int64 para trabajar con porcentajes enteros.

✅ Verificación de conversiones
actual_price
Tipo anterior: object
Tipo nuevo: float64
Ejemplo:
Antes: ₹1,099
Después: 1099.0
discount_percentage
Tipo anterior: object
Tipo nuevo: int64
Ejemplo:
Antes: 64%
Después: 64

💾 Resultados
Se actualizaron los tipos de datos en el dataset productos.
Nuevos archivos exportados:
```python
productos_paso2.csv
```
Con esta limpieza, los campos de precios y porcentajes quedaron listos para análisis numéricos como estadísticas, correlaciones y modelos de descuento.

codigo:
```python
# =============================================================================
# PASO 1: CONVERTIR DISCOUNTED_PRICE A DECIMAL
# =============================================================================

print("🎯 PASO 1: CONVERTIR DISCOUNTED_PRICE A DECIMAL")
print("=" * 60)

# -----------------------------------------------------------------------------
# 1. CONVERSIÓN DIRECTA
# -----------------------------------------------------------------------------
print("\n🔢 CONVIRTIENDO DISCOUNTED_PRICE...")
print("-" * 40)

# Guardar el valor original para referencia
# Verificar si la columna original aún existe como tipo objeto antes de guardar
if productos['discounted_price'].dtype == 'object':
    productos['discounted_price_original'] = productos['discounted_price']
else:
    print("ℹ️ 'discounted_price' is already numeric, skipping original value save.")


# Convertir a decimal (eliminar ₹ y , luego convertir a float)
# Solo aplicar operaciones de cadena si la columna es de tipo objeto
if productos['discounted_price'].dtype == 'object':
    productos['discounted_price'] = (
        productos['discounted_price']
        .str.replace('₹', '', regex=False)    # Eliminar símbolo de rupee
        .str.replace(',', '', regex=False)     # Eliminar comas de miles
        .astype(float)            # Convertir a decimal
    )
elif productos['discounted_price'].dtype in ['int64', 'float64']:
    print("ℹ️ 'discounted_price' is already numeric, no conversion needed.")
else:
    # Intentar convertir a numérico, forzando los errores
    productos['discounted_price'] = pd.to_numeric(productos['discounted_price'], errors='coerce')
    # Rellenar cualquier NaN resultante si es necesario (por ejemplo, con 0 o con la mediana)
    productos['discounted_price'] = productos['discounted_price'].fillna(0) # O usar .median()
    print("ℹ️ Attempted numeric conversion with error coercion and NaN filling.")


# -----------------------------------------------------------------------------
# 2. VERIFICACIÓN MÍNIMA
# -----------------------------------------------------------------------------
print("✅ VERIFICACIÓN:")
print("-" * 40)

# Verificar si 'discounted_price_original' existe antes de intentar imprimir su dtype
if 'discounted_price_original' in productos.columns:
    print(f"   Tipo de dato anterior: {productos['discounted_price_original'].dtype}")

print(f"   Tipo de dato nuevo: {productos['discounted_price'].dtype}")

print(f"\n   Ejemplo de conversión:")
# Acceder de forma segura a iloc[0]
if not productos.empty:
    if 'discounted_price_original' in productos.columns:
        print(f"   Antes: {productos['discounted_price_original'].iloc[0]}")
    print(f"   Después: {productos['discounted_price'].iloc[0]}")
else:
    print("   DataFrame is empty, cannot show examples.")


# -----------------------------------------------------------------------------
# 3. GUARDAR
# -----------------------------------------------------------------------------
print("\n💾 GUARDANDO:")
print("-" * 40)

# Guardar el DataFrame actualizado
productos.to_csv("productos_paso1.csv", index=False)
print("   ✅ Archivo guardado: productos_paso1.csv")


print("   discounted_price convertido exitosamente de object a decimal")
print("   No se realizaron análisis ni cálculos adicionales")
```

5. Limpieza y Conversión de Columnas Numéricas
En esta etapa se trabajó sobre las variables que originalmente estaban en formato de texto pero que en realidad representan valores numéricos (precios, descuentos y calificaciones).
El objetivo fue transformarlas en un formato adecuado (float64 o int64) para permitir cálculos y análisis estadísticos posteriores.

📊 Estado Inicial
Productos: 1351 filas, 19 columnas.
Reseñas: 1194 filas, 16 columnas.
Se identificó que las siguientes columnas requerían limpieza:
```python
productos: actual_price, discount_percentage.
resenas: rating.
```

🔧 Proceso de Conversión
Guardar copia de los datos originales
Si la columna era de tipo object (texto), se creó una columna auxiliar *_original para conservar los valores previos.
Limpieza de símbolos
En actual_price: se eliminaron el símbolo ₹ y las comas de miles.
En discount_percentage: se eliminó el símbolo %.
En rating: se eliminaron espacios y valores vacíos.
Conversión a valores numéricos
```python
Se utilizó pd.to_numeric() para transformar los datos.
```
Valores inválidos se convirtieron en NaN.
Posteriormente se rellenaron (fillna) con:
0 en precios y porcentajes.
Mediana en el caso de rating, para mantener coherencia en los análisis.
Verificación final
actual_price → float64.
discount_percentage → int64.
rating → float64.

✅ Resultado Final
Productos:
discounted_price (float64)
actual_price (float64)
discount_percentage (int64)
Reseñas:
rating (float64)
rating_count (int64)

💾 Guardado de Resultados
Se generaron dos nuevos archivos con los datos ya procesados:
amazon_productos_numerico.csv
amazon_resenas_numerico.csv
Estos archivos contienen las columnas numéricas listas para análisis, sin riesgo de errores por símbolos o formatos incorrectos.


codigo:
```python
# =============================================================================
# LIMPIEZA Y CONVERSIÓN DE COLUMNAS NUMÉRICAS RESTANTES
# =============================================================================

print("🎯 LIMPIEZA Y CONVERSIÓN DE COLUMNAS NUMÉRICAS RESTANTES")
print("=" * 70)

# Guardar el estado inicial para comparar
print("📊 ESTADO INICIAL:")
print(f"   Productos: {productos.shape[0]} filas, {productos.shape[1]} columnas")
print(f"   Reseñas: {resenas.shape[0]} filas, {resenas.shape[1]} columnas")

print("\n🔍 Tipos de datos iniciales:")
print("   Productos:")
print(productos[['actual_price', 'discount_percentage']].dtypes)
print("\n   Reseñas:")
print(resenas[['rating']].dtypes)


# -----------------------------------------------------------------------------
# 1. PROCESAR COLUMNAS EN PRODUCTOS (actual_price, discount_percentage)
# -----------------------------------------------------------------------------
print("\n" + "="*60)
print("📦 PROCESANDO TABLA PRODUCTOS")
print("="*60)

columnas_productos_a_convertir = ['actual_price', 'discount_percentage']

for columna in columnas_productos_a_convertir:
    print(f"\n🔢 Convirtiendo columna: {columna}")
    print("-" * 40)

    if columna in productos.columns:
        # Guardar el valor original si aún es object
        if productos[columna].dtype == 'object':
            productos[f'{columna}_original'] = productos[columna]
            print(f"   Guardado '{columna}_original' (tipo: {productos[f'{columna}_original'].dtype})")
        else:
             print(f"   ℹ️ '{columna}' ya es numérico, omitiendo guardar original como object.")


        # Limpieza y conversión
        if productos[columna].dtype == 'object':
            # Aplicar limpieza solo si es string/object
            productos[columna] = (
                productos[columna]
                .str.replace('₹', '', regex=False)    # Eliminar símbolo de rupee
                .str.replace(',', '', regex=False)     # Eliminar comas de miles
            )
            if columna == 'discount_percentage':
                 productos[columna] = productos[columna].str.replace('%', '', regex=False) # Eliminar %

            # Convertir a numérico, forzando errores a NaN
            productos[columna] = pd.to_numeric(productos[columna], errors='coerce')

            # Rellenar NaN resultantes si es necesario (ej. con 0 o mediana)
            # Para precios y porcentajes, 0 o NaN podrían ser opciones válidas dependiendo del contexto.
            # Aquí optamos por 0 para simplicidad, pero NaN podría ser preferible para análisis estadísticos.
            productos[columna] = productos[columna].fillna(0) # O .median() o dejar como NaN si se prefiere

            print(f"   ✅ Limpieza y conversión completada para '{columna}'")

        elif productos[columna].dtype in ['int64', 'float64']:
             print(f"   ℹ️ '{columna}' ya es numérico, no se necesita limpieza de símbolos.")
        else:
             # Intento general de conversión si el tipo no es object ni numérico conocido
             productos[columna] = pd.to_numeric(productos[columna], errors='coerce')
             productos[columna] = productos[columna].fillna(0) # Manejar posibles errores de conversión a NaN
             print(f"   ℹ️ Intentando conversión numérica general con manejo de errores para '{columna}'.")


        # Verificar tipo de dato final y mostrar ejemplo
        print(f"   Tipo de dato final: {productos[columna].dtype}")
        if not productos.empty:
            print(f"   Ejemplo de valor convertido: {productos[columna].iloc[0]}")
        else:
            print("   DataFrame is empty, cannot show examples.")

    else:
        print(f"   ⚠️ Columna '{columna}' no encontrada en el DataFrame de productos.")


# -----------------------------------------------------------------------------
# 2. PROCESAR COLUMNAS EN RESEÑAS (rating)
# -----------------------------------------------------------------------------
print("\n" + "="*60)
print("⭐ PROCESANDO TABLA RESEÑAS")
print("="*60)

columna_resenas_a_convertir = 'rating'

if columna_resenas_a_convertir in resenas.columns:
    print(f"\n🔢 Convirtiendo columna: {columna_resenas_a_convertir}")
    print("-" * 40)

    # Guardar el valor original si aún es object
    if resenas[columna_resenas_a_convertir].dtype == 'object':
        resenas[f'{columna_resenas_a_convertir}_original'] = resenas[columna_resenas_a_convertir]
        print(f"   Guardado '{columna_resenas_a_convertir}_original' (tipo: {resenas[f'{columna_resenas_a_convertir}_original'].dtype})")
    else:
         print(f"   ℹ️ '{columna_resenas_a_convertir}' ya es numérico, omitiendo guardar original como object.")


    # Limpieza y conversión
    if resenas[columna_resenas_a_convertir].dtype == 'object':
         # Aplicar limpieza solo si es string/object
        resenas[columna_resenas_a_convertir] = (
            resenas[columna_resenas_a_convertir]
            .astype(str) # Asegurar que es string para .str métodos
            .str.strip() # Limpiar espacios
            .replace('', 'NaN') # Remplazar vacíos con NaN para que to_numeric los maneje
        )

        # Convertir a numérico
        resenas[columna_resenas_a_convertir] = pd.to_numeric(resenas[columna_resenas_a_convertir], errors='coerce')

        # Rellenar NaN resultantes (los ratings nulos o con error se convierten a NaN)
        # Rellenar con un valor como la mediana o la media podría ser una opción si se quiere conservar filas,
        # pero eliminar filas con rating desconocido también es válido.
        # Aquí rellenamos con la mediana como ejemplo, pero considera la mejor estrategia para el análisis.
        median_rating = resenas[columna_resenas_a_convertir].median()
        resenas[columna_resenas_a_convertir] = resenas[columna_resenas_a_convertir].fillna(median_rating)

        print(f"   ✅ Limpieza y conversión completada para '{columna_resenas_a_convertir}'")

    elif resenas[columna_resenas_a_convertir].dtype in ['int64', 'float64']:
        print(f"   ℹ️ '{columna_resenas_a_convertir}' ya es numérico, no se necesita conversión.")
    else:
         # Intento general de conversión si el tipo no es object ni numérico conocido
         resenas[columna_resenas_a_convertir] = pd.to_numeric(resenas[columna_resenas_a_convertir], errors='coerce')
         median_rating = resenas[columna_resenas_a_convertir].median()
         resenas[columna_resenas_a_convertir] = resenas[columna_resenas_a_convertir].fillna(median_rating)
         print(f"   ℹ️ Intentando conversión numérica general con manejo de errores para '{columna_resenas_a_convertir}'.")


    # Verificar tipo de dato final y mostrar ejemplo
    print(f"   Tipo de dato final: {resenas[columna_resenas_a_convertir].dtype}")
    if not resenas.empty:
        print(f"   Ejemplo de valor convertido: {resenas[columna_resenas_a_convertir].iloc[0]}")
    else:
        print("   DataFrame is empty, cannot show examples.")

else:
    print(f"   ⚠️ Columna '{columna_resenas_a_convertir}' no encontrada en el DataFrame de reseñas.")


# -----------------------------------------------------------------------------
# 3. VERIFICACIÓN FINAL DE TIPOS DE DATOS
# -----------------------------------------------------------------------------
print("\n" + "="*70)
print("✅ VERIFICACIÓN FINAL DE TIPOS DE DATOS")
print("="*70)

print("\n📦 PRODUCTOS - Tipos de datos finales:")
print(productos[['discounted_price', 'actual_price', 'discount_percentage']].dtypes)

print("\n⭐ RESEÑAS - Tipos de datos finales:")
print(resenas[['rating', 'rating_count']].dtypes) # Incluimos rating_count que ya fue limpiado

# -----------------------------------------------------------------------------
# 4. GUARDAR DATOS LIMPIOS (OPCIONAL)
# -----------------------------------------------------------------------------
print("\n" + "="*70)
print("💾 GUARDANDO DATOS LIMPIOS")
print("="*70)

productos.to_csv("amazon_productos_numerico.csv", index=False)
resenas.to_csv("amazon_resenas_numerico.csv", index=False)
print("   ✅ Archivos guardados:")
print("      - amazon_productos_numerico.csv")
print("      - amazon_resenas_numerico.csv")

print("\n🎉 ¡Limpieza y conversión de columnas numéricas completada!")
```

6. Conversión de rating_original a Entero
En este paso se trabajó con la columna rating_original de la tabla de reseñas.
Originalmente, esta variable tenía valores en formato object o float, representando puntuaciones con decimales (ejemplo: 4.2).
El objetivo fue convertirla a un formato entero (int64), adecuado para análisis categóricos o estadísticos que no requieran decimales.

📊 Proceso Realizado
```python
Conversión robusta con pd.to_numeric()
```
Se utilizó esta función con errors='coerce' para manejar cualquier valor no numérico.
Los valores inválidos se transformaron en NaN.
Manejo de valores nulos
Los NaN se rellenaron con 0 antes de convertir a enteros.
Esta decisión asegura consistencia en el dataset, aunque puede ajustarse según el análisis (ejemplo: eliminar o imputar con media/mediana).
Transformación final
La columna rating_original se transformó a tipo int64.
Se creó también la columna auxiliar rating_original_numeric para conservar los valores en formato decimal antes de redondear.

✅ Resultado
Tipo de dato anterior: object o float.
Tipo de dato nuevo: int64.
Ejemplo de conversión:
Antes: 4.2
Después: 4

💾 Guardado de Resultados
El DataFrame actualizado se almacenó en el archivo:
```python
resenas_paso2.csv
```
Este archivo ya contiene la columna rating_original transformada a enteros, lista para su uso en análisis posteriores.

🎯 Conclusión
Con esta limpieza:
rating_original queda en un formato uniforme (int64).
Se garantiza mayor facilidad para análisis que requieran agrupaciones, conteos o categorización de calificaciones.

codigo:
```python
# =============================================================================
# PASO 2: CONVERTIR RATING_ORIGINAL A ENTERO
# =============================================================================

print("🎯 PASO 2: CONVERTIR RATING_ORIGINAL A ENTERO")
print("=" * 60)

# -----------------------------------------------------------------------------
# 1. CONVERSIÓN
# -----------------------------------------------------------------------------
print("\n🔢 CONVIRTIENDO RATING_ORIGINAL...")
print("-" * 40)

# Usar pd.to_numeric para una conversión robusta, forzando los errores
resenas['rating_original_numeric'] = pd.to_numeric(resenas['rating_original'], errors='coerce')

# Convertir la columna numérica a entero, rellenando primero los NaNs si es necesario
# Rellenar los NaNs con 0 antes de convertir a int. Ajustar el valor de fillna si es necesario.
resenas['rating_original'] = resenas['rating_original_numeric'].fillna(0).astype(int)


# -----------------------------------------------------------------------------
# 2. VERIFICACIÓN MÍNIMA
# -----------------------------------------------------------------------------
print("✅ VERIFICACIÓN:")
print("-" * 40)

# Verifica el tipo de dato (dtype) de la columna antes y después del intento de conversión en esta celda
# Es difícil saber el estado exacto "antes de este paso" sin volver a ejecutar las celdas anteriores,
# así que indicaremos el estado esperado en base al error y al objetivo.
print(f"   Tipo de dato anterior (antes de este paso): object o float")
print(f"   Tipo de dato nuevo: {resenas['rating_original'].dtype}")

print(f"\n   Ejemplo de conversión:")
# Acceder de forma segura a iloc[0] y manejar posibles NaNs
if not resenas.empty:
    # Intento de mostrar un valor de la columna numérica intermedia para comparación
    if 'rating_original_numeric' in resenas.columns and pd.notna(resenas['rating_original_numeric'].iloc[0]):
        print(f"   Antes (valor numérico intermedio): {resenas['rating_original_numeric'].iloc[0]}")
    elif 'rating_original_backup' in resenas.columns and pd.notna(resenas['rating_original_backup'].iloc[0]):
         print(f"   Antes (backup original): {resenas['rating_original_backup'].iloc[0]}")
    else:
        print("   Antes: (Valor original no disponible o era NaN)")

    print(f"   Después: {resenas['rating_original'].iloc[0]}")
else:
    print("   DataFrame está vacío, no se pueden mostrar ejemplos.")


# -----------------------------------------------------------------------------
# 3. GUARDAR SIN ANÁLISIS ADICIONAL
# -----------------------------------------------------------------------------
print("\n💾 GUARDANDO:")
print("-" * 40)

# Guardar el DataFrame actualizado
resenas.to_csv("resenas_paso2.csv", index=False)
print("   ✅ Archivo guardado: resenas_paso2.csv")

print(f"\n🎉 ¡PASO 2 COMPLETADO!")
print("   rating_original convertido exitosamente a entero (manejando decimales y errores)")
print("   No se realizaron análisis ni cálculos adicionales")
```

Verificación de Nulos y Duplicados
Con el fin de garantizar la calidad y consistencia de los datos antes de realizar cualquier análisis avanzado, se efectuó una revisión sistemática de:
Valores Nulos (NaN)
Se inspeccionaron todas las columnas de las tablas Productos y Reseñas.
El objetivo fue identificar posibles datos faltantes que pudieran distorsionar los resultados de cálculos, métricas o visualizaciones.
En caso de encontrarlos, se definieron estrategias de manejo como:
Eliminación de registros incompletos.
Relleno con valores estadísticos (media, mediana o moda).
Sustitución con un valor neutro (ejemplo: 0 en columnas numéricas).
Registros Duplicados
Se evaluaron ambas tablas para identificar filas repetidas.
En particular, se verificaron los campos clave:
Productos: product_id
Reseñas: review_id
Los duplicados encontrados se eliminaron, conservando una sola ocurrencia de cada registro único.

codigo:
```python
# =============================================================================
# ANÁLISIS COMPLETO DEL FORMATO DE DATOS
# =============================================================================

print("🔍 ANÁLISIS COMPLETO DEL FORMATO DE DATOS")
print("=" * 60)

# -----------------------------------------------------------------------------
# 1. FUNCIÓN PARA ANALIZAR UNA COLUMNA
# -----------------------------------------------------------------------------
def analizar_columna(df, nombre_columna, df_nombre):
    """Analiza profundamente una columna y muestra ejemplos"""
    print(f"\n📊 {df_nombre} - Columna: {nombre_columna}")
    print("-" * 50)

    # Información básica
    print(f"   Tipo de dato: {df[nombre_columna].dtype}")
    print(f"   Total de valores: {len(df[nombre_columna])}")
    print(f"   Valores nulos: {df[nombre_columna].isnull().sum()}")
    print(f"   Valores únicos: {df[nombre_columna].nunique()}")

    # Estadísticas según el tipo de dato
    if df[nombre_columna].dtype in ['int64', 'float64']:
        # Es numérico
        print(f"   Mínimo: {df[nombre_columna].min()}")
        print(f"   Máximo: {df[nombre_columna].max()}")
        print(f"   Promedio: {df[nombre_columna].mean():.2f}")
        print(f"   Mediana: {df[nombre_columna].median()}")

    elif df[nombre_columna].dtype == 'object':
        # Es texto/string
        longitudes = df[nombre_columna].astype(str).str.len()
        print(f"   Longitud promedio: {longitudes.mean():.1f} caracteres")
        print(f"   Longitud mínima: {longitudes.min()} caracteres")
        print(f"   Longitud máxima: {longitudes.max()} caracteres")

        # Mostrar los valores más frecuentes
        print(f"\n   Valores más frecuentes:")
        valores_frecuentes = df[nombre_columna].value_counts().head(3)
        for valor, conteo in valores_frecuentes.items():
            print(f"     '{valor}': {conteo} veces")

    # Mostrar ejemplos de valores
    print(f"\n   Ejemplos de valores:")
    ejemplos = df[nombre_columna].dropna().head(3).tolist()
    for i, ejemplo in enumerate(ejemplos, 1):
        print(f"     {i}. {repr(ejemplo)}")  # repr() muestra el formato exacto

    print(f"   ... y {len(df[nombre_columna]) - 3} más")

# -----------------------------------------------------------------------------
# 2. ANALIZAR TABLA PRODUCTOS
# -----------------------------------------------------------------------------
print("\n" + "="*60)
print("📦 ANÁLISIS DE LA TABLA PRODUCTOS")
print("="*60)

for columna in productos.columns:
    analizar_columna(productos, columna, "PRODUCTOS")

# -----------------------------------------------------------------------------
# 3. ANALIZAR TABLA RESEÑAS
# -----------------------------------------------------------------------------
print("\n" + "="*60)
print("⭐ ANÁLISIS DE LA TABLA RESEÑAS")
print("="*60)

for columna in resenas.columns:
    analizar_columna(resenas, columna, "RESEÑAS")

# -----------------------------------------------------------------------------
# 4. RESUMEN DE TIPOS DE DATOS
# -----------------------------------------------------------------------------
print("\n" + "="*60)
print("📋 RESUMEN DE TIPOS DE DATOS")
print("="*60)

print("\n📦 PRODUCTOS:")
for columna in productos.columns:
    print(f"   {columna}: {productos[columna].dtype}")

print("\n⭐ RESEÑAS:")
for columna in resenas.columns:
    print(f"   {columna}: {resenas[columna].dtype}")

# -----------------------------------------------------------------------------
# 5. DETECTAR POSIBLES PROBLEMAS DE FORMATO
# -----------------------------------------------------------------------------
print("\n" + "="*60)
print("🚨 DETECCIÓN DE POSIBLES PROBLEMAS")
print("="*60)

# Verificar columnas que deberían ser numéricas pero son texto
columnas_numericas_esperadas = ['discounted_price', 'actual_price', 'discount_percentage',
                               'rating', 'rating_count']

print("Columnas que podrían necesitar conversión numérica:")
for columna in columnas_numericas_esperadas:
    if columna in productos.columns and productos[columna].dtype == 'object':
        print(f"   📦 {columna} (debería ser numérica)")
    if columna in resenas.columns and resenas[columna].dtype == 'object':
        print(f"   ⭐ {columna} (debería ser numérica)")

# -----------------------------------------------------------------------------
# 6. EJEMPLOS DETALLADOS DE VALORES
# -----------------------------------------------------------------------------
print("\n" + "="*60)
print("🔍 EJEMPLOS DETALLADOS POR COLUMNA")
print("="*60)

def mostrar_ejemplos_detallados(df, nombre_columna, n=5):
    """Muestra ejemplos detallados de una columna"""
    print(f"\n📋 {nombre_columna} - Ejemplos detallados:")
    valores = df[nombre_columna].dropna().head(n).tolist()
    for i, valor in enumerate(valores, 1):
        print(f"   {i}. {repr(valor)}")
        print(f"      Tipo: {type(valor)}")
        if isinstance(valor, str):
            print(f"      Longitud: {len(valor)} caracteres")
            if any(c in valor for c in ['₹', ',', '%']):
                print(f"      Contiene símbolos: {[c for c in ['₹', ',', '%'] if c in valor]}")

# Ejemplos de columnas clave
print("\n📦 PRODUCTOS - Ejemplos detallados:")
mostrar_ejemplos_detallados(productos, 'discounted_price')
mostrar_ejemplos_detallados(productos, 'actual_price')
mostrar_ejemplos_detallados(productos, 'discount_percentage')

print("\n⭐ RESEÑAS - Ejemplos detallados:")
mostrar_ejemplos_detallados(resenas, 'rating')
mostrar_ejemplos_detallados(resenas, 'rating_count')
```

codigo:
```python
# Verificar si quedan duplicados en product_id después de la limpieza
duplicados_id_prod_despues = productos['product_id'].duplicated().sum()

print(f"Duplicados restantes en product_id en la tabla productos: {duplicados_id_prod_despues}")

if duplicados_id_prod_despues > 0:
    print("\n⚠️  Aún hay product_id duplicados. Mostrar ejemplos:")
    ids_duplicados_final = productos[productos['product_id'].duplicated(keep=False)]['product_id'].unique()
    productos_duplicados_final = productos[productos['product_id'].isin(ids_duplicados_final)]
    display(productos_duplicados_final.sort_values('product_id'))
else:
    print("✅ No hay product_id duplicados restantes en la tabla productos.")
```

7. Identificación y Manejo de Valores Fuera de Alcance
En esta etapa se buscó asegurar que las variables numéricas se encuentren dentro de un rango lógico y válido.
El enfoque consistió en dos pasos:

Verificación de discount_percentage
Se evaluó si existían registros con valores de descuento menores a 0% o mayores a 100%.
Este control es necesario ya que, en términos prácticos, no debería existir un descuento negativo ni uno que supere el 100%.
📊 Resultado:
Número de registros fuera de rango: 0
✅ No se encontraron valores anómalos en la columna discount_percentage.

codigo:
```python
# Detectar registros con discount_percentage fuera del rango esperado ( < 0 o > 100)
registros_fuera_rango = productos[(productos['discount_percentage'] < 0) | (productos['discount_percentage'] > 100)]

print(f"Número de registros con discount_percentage fuera del rango esperado: {len(registros_fuera_rango)}")

if not registros_fuera_rango.empty:
    print("\nRegistros con discount_percentage fuera del rango esperado:")
    display(registros_fuera_rango)
else:
    print("\n✅ No se encontraron registros con discount_percentage fuera del rango esperado (0-100).")
```

Confirmación de Columnas Numéricas para Análisis de Outliers
Se identificaron las columnas numéricas en las que posteriormente se aplicará la detección de outliers (valores atípicos):
Productos:
discounted_price → tipo float64
actual_price → tipo float64
discount_percentage → tipo int64
Reseñas:
rating → tipo float64
rating_count → tipo int64
📌 Estas serán las variables base sobre las que se calcularán métricas estadísticas (como IQR, cuartiles y límites de detección de outliers) en pasos posteriores.

🎯 Conclusión
La validación confirma que no existen valores fuera de rango en discount_percentage.
Se han establecido claramente las columnas numéricas clave para el análisis estadístico y detección de valores atípicos.

codigo:
```python
print("Data types of numeric columns in 'productos':")
print(productos[['discounted_price', 'actual_price', 'discount_percentage']].dtypes)

print("\nData types of numeric columns in 'resenas':")
print(resenas[['rating', 'rating_count']].dtypes)
```

8. Identificación y Manejo de Outliers
En este paso se aplicaron dos métodos complementarios para detectar valores atípicos (outliers) en las columnas numéricas de los datasets:
Z-Score (considerando |Z| > 3 como umbral)
Rango Intercuartílico (IQR) (Q1 - 1.5IQR, Q3 + 1.5IQR)

8.1 Columnas Analizadas
📦 Productos
discounted_price
actual_price
discount_percentage
⭐ Reseñas
rating
rating_count

8.2 Resultados
Z-Score
Productos: 44 outliers
Reseñas: 38 outliers
IQR
Productos: 219 outliers
Reseñas: 140 outliers
📌 El método IQR identificó un número significativamente mayor de outliers que Z-Score. Esto ocurre porque IQR es más sensible a valores que se encuentran fuera del rango intercuartílico, aunque no sean extremadamente atípicos en términos de desviaciones estándar.

8.3 Observaciones
Precios (discounted_price, actual_price):
Los outliers detectados por Z-Score corresponden a productos de gama alta (ej. televisores, laptops premium).
El método IQR detectó además precios menos extremos pero igualmente alejados de la mediana.
Porcentaje de Descuento (discount_percentage):
Confirmado que los valores están dentro del rango 0-100%.
Los outliers corresponden a descuentos muy agresivos (ej. > 80%), posiblemente promociones especiales o valores anómalos.
Rating (rating):
Como está en escala de 1 a 5, Z-Score no detectó casos extremos.
IQR señaló algunos valores en los límites (1 o 5 estrellas), que son válidos en el contexto.
Conteo de Ratings (rating_count):
Z-Score identificó productos con un número extremadamente alto de reseñas.
IQR señaló además valores intermedios pero aún altos.
Estos datos suelen ser representativos de productos populares (best-sellers).

8.4 Estrategias de Manejo
Precios:
Investigar manualmente los casos más extremos detectados por Z-Score.
Aplicar transformación logarítmica para reducir el sesgo en análisis y modelos.
Porcentaje de Descuento:
Mantener valores (excepto que se detecten errores).
Pueden ser informativos sobre estrategias de precios.
Rating:
Mantener todos los registros, pues reflejan valoraciones reales.
Conteo de Ratings:
No eliminar, ya que representan popularidad real.
Aplicar transformación logarítmica para suavizar la asimetría.

8.5 Conclusión
El análisis muestra que:
IQR es más sensible, señalando más casos como outliers.
Z-Score es más conservador, enfocándose en valores extremadamente atípicos.
La estrategia elegida busca equilibrar robustez estadística y realismo de negocio, evitando la eliminación masiva de registros y priorizando la transformación de variables cuando sea necesario.

codigo:
```python
from scipy.stats import zscore

# Calculo Z-scores para 'productos'
productos['discounted_price_zscore'] = zscore(productos['discounted_price'])
productos['actual_price_zscore'] = zscore(productos['actual_price'])
productos['discount_percentage_zscore'] = zscore(productos['discount_percentage'])

# Calculo Z-scores para 'resenas'
resenas['rating_zscore'] = zscore(resenas['rating'])
resenas['rating_count_zscore'] = zscore(resenas['rating_count'])

print("Z-Scores calculated for:")
print("- productos: discounted_price, actual_price, discount_percentage")
print("- resenas: rating, rating_count")

# Display the first few rows to show the new columns
print("\nProductos with Z-scores:")
display(productos.head())

print("\nResenas with Z-scores:")
display(resenas.head())
```

Razonamiento: Identificar posibles valores atípicos utilizando un umbral de puntuación Z (por ejemplo, |Z-score| > 3) para las columnas numéricas especificadas en ambos DataFrames.

codigo:
z_score_threshold = 3

# Identificar outliers en 'productos' basado encZ-score
outliers_productos_zscore = productos[
    (abs(productos['discounted_price_zscore']) > z_score_threshold) |
    (abs(productos['actual_price_zscore']) > z_score_threshold) |
    (abs(productos['discount_percentage_zscore']) > z_score_threshold)
]

print(f"\nOutliers identified in 'productos' using Z-score (threshold = {z_score_threshold}):")
print(f"Number of outliers: {len(outliers_productos_zscore)}")
if not outliers_productos_zscore.empty:
    display(outliers_productos_zscore)
else:
    print("No outliers found in 'productos' using Z-score.")


# Identificar outliers en 'resenas' basado en Z-score
outliers_resenas_zscore = resenas[
    (abs(resenas['rating_zscore']) > z_score_threshold) |
    (abs(resenas['rating_count_zscore']) > z_score_threshold)
]

print(f"\nOutliers identified in 'resenas' using Z-score (threshold = {z_score_threshold}):")
print(f"Number of outliers: {len(outliers_resenas_zscore)}")
if not outliers_resenas_zscore.empty:
    display(outliers_resenas_zscore)
else:
    print("No outliers found in 'resenas' using Z-score.")

Razonamiento: Calcular el IQR (Rango Intercuartílico) y los límites para la detección de valores atípicos en las columnas numéricas especificadas de ambos DataFrames

codigo:
```python
# Calculo IQR y limites para 'productos'
for col in ['discounted_price', 'actual_price', 'discount_percentage']:
    Q1 = productos[col].quantile(0.25)
    Q3 = productos[col].quantile(0.75)
    IQR = Q3 - Q1
    lower_bound = Q1 - 1.5 * IQR
    upper_bound = Q3 + 1.5 * IQR

    productos[f'{col}_iqr_lower'] = lower_bound
    productos[f'{col}_iqr_upper'] = upper_bound

    print(f"\nIQR and bounds for '{col}' in 'productos':")
    print(f"  Q1: {Q1:.2f}")
    print(f"  Q3: {Q3:.2f}")
    print(f"  IQR: {IQR:.2f}")
    print(f"  Lower Bound: {lower_bound:.2f}")
    print(f"  Upper Bound: {upper_bound:.2f}")


# Calculo IQR limites para   'resenas'
for col in ['rating', 'rating_count']:
    Q1 = resenas[col].quantile(0.25)
    Q3 = resenas[col].quantile(0.75)
    IQR = Q3 - Q1
    lower_bound = Q1 - 1.5 * IQR
    upper_bound = Q3 + 1.5 * IQR

    resenas[f'{col}_iqr_lower'] = lower_bound
    resenas[f'{col}_iqr_upper'] = upper_bound

    print(f"\nIQR and bounds for '{col}' in 'resenas':")
    print(f"  Q1: {Q1:.2f}")
    print(f"  Q3: {Q3:.2f}")
    print(f"  IQR: {IQR:.2f}")
    print(f"  Lower Bound: {lower_bound:.2f}")
    print(f"  Upper Bound: {upper_bound:.2f}")

print("\nIQR bounds calculated and added to dataframes.")
```

Razonamiento: Identifique posibles valores atípicos utilizando los límites del RIQ calculados para las columnas numéricas especificadas en ambos marcos de datos.

codigo:
```python
# identificar outliers en 'productos' basado en IQR
outliers_productos_iqr = productos[
    (productos['discounted_price'] < productos['discounted_price_iqr_lower']) |
    (productos['discounted_price'] > productos['discounted_price_iqr_upper']) |
    (productos['actual_price'] < productos['actual_price_iqr_lower']) |
    (productos['actual_price'] > productos['actual_price_iqr_upper']) |
    (productos['discount_percentage'] < productos['discount_percentage_iqr_lower']) |
    (productos['discount_percentage'] > productos['discount_percentage_iqr_upper'])
]

print("\nOutliers identified in 'productos' using IQR:")
print(f"Number of outliers: {len(outliers_productos_iqr)}")
if not outliers_productos_iqr.empty:
    display(outliers_productos_iqr)
else:
    print("No outliers found in 'productos' using IQR.")


# identificar outliers en 'resenas' basado en IQR
outliers_resenas_iqr = resenas[
    (resenas['rating'] < resenas['rating_iqr_lower']) |
    (resenas['rating'] > resenas['rating_iqr_upper']) |
    (resenas['rating_count'] < resenas['rating_count_iqr_lower']) |
    (resenas['rating_count'] > resenas['rating_count_iqr_upper'])
]

print("\nOutliers identified in 'resenas' using IQR:")
print(f"Number of outliers: {len(outliers_resenas_iqr)}")
if not outliers_resenas_iqr.empty:
    display(outliers_resenas_iqr)
else:
    print("No outliers found in 'resenas' using IQR.")
```

Razonamiento: Imprima la cantidad de valores atípicos encontrados por cada método y muestre las primeras filas de los marcos de datos de valores atípicos para comparar.

codigo:
```python
print("Número de outliers encontrados por método:")
print(f"  Productos (Z-Score): {len(outliers_productos_zscore)}")
print(f"  Productos (IQR): {len(outliers_productos_iqr)}")
print(f"  Reseñas (Z-Score): {len(outliers_resenas_zscore)}")
print(f"  Reseñas (IQR): {len(outliers_resenas_iqr)}")

print("\nPrimeras filas de Outliers (Z-Score) - Productos:")
if not outliers_productos_zscore.empty:
    display(outliers_productos_zscore.head())
else:
    print("No hay outliers en productos según Z-Score.")

print("\nPrimeras filas de Outliers (Z-Score) - Reseñas:")
if not outliers_resenas_zscore.empty:
    display(outliers_resenas_zscore.head())
else:
    print("No hay outliers en reseñas según Z-Score.")

print("\nPrimeras filas de Outliers (IQR) - Productos:")
if not outliers_productos_iqr.empty:
    display(outliers_productos_iqr.head())
else:
    print("No hay outliers en productos según IQR.")

print("\nPrimeras filas de Outliers (IQR) - Reseñas:")
if not outliers_resenas_iqr.empty:
    display(outliers_resenas_iqr.head())
else:
    print("No hay outliers en reseñas según IQR.")

print("\nObservaciones sobre las diferencias:")
print("- El método IQR identifica un número significativamente mayor de outliers en ambas tablas en comparación con el método Z-Score (usando un umbral de 3).")
print("- Esto sugiere que hay muchos valores que caen fuera del rango intercuartílico (Q1 - 1.5*IQR a Q3 + 1.5*IQR) pero que no son extremadamente atípicos (es decir, no tienen un Z-Score > 3).")
print("- Las filas mostradas confirman que los outliers identificados por IQR a menudo tienen valores menos extremos que los identificados por Z-Score.")
```

Considerar el tratamiento de outliers
Subtask:
Discutir posibles estrategias para manejar los outliers encontrados (ej. eliminarlos, transformarlos, dejarlos como están dependiendo del análisis posterior).

Reasoning: Discuss the outlier handling strategies based on the previous analysis and propose a preliminary strategy for this project.

codigo:
```python
print("## Discusión y Estrategia para el Manejo de Outliers")
print("=" * 60)

print("\n### Resumen de Outliers Encontrados:")
print(f"- Productos (Z-Score > 3): {len(outliers_productos_zscore)} outliers")
print(f"- Productos (IQR): {len(outliers_productos_iqr)} outliers")
print(f"- Reseñas (Z-Score > 3): {len(outliers_resenas_zscore)} outliers")
print(f"- Reseñas (IQR): {len(outliers_resenas_iqr)} outliers")

print("\n### Impacto Potencial de los Outliers:")
print("- **Precios (discounted_price, actual_price):** Outliers en precios pueden sesgar estadísticas descriptivas (media, desviación estándar) y afectar modelos que son sensibles a la escala (ej., regresión lineal, clustering basado en distancia). Precios muy altos o muy bajos podrían representar productos de lujo/gama baja o errores de entrada.")
print("- **Porcentaje de Descuento (discount_percentage):** Outliers aquí podrían indicar descuentos inusualmente altos o bajos, posiblemente errores o promociones especiales. Pueden afectar análisis de rentabilidad o modelos de predicción de ventas.")
print("- **Rating (rating):** Aunque los ratings suelen estar en una escala limitada (ej., 1-5), outliers (si los hubiera, aunque en este caso no se encontraron con Z-score>3 o IQR) podrían ser errores de entrada o reseñas fraudulentas. En este dataset, los ratings parecen estar dentro de un rango esperado, aunque IQR identificó algunos valores en los extremos como outliers.")
print("- **Conteo de Ratings (rating_count):** Outliers en el conteo de ratings son comunes y a menudo representan productos muy populares. Eliminar estos podría llevar a la pérdida de información valiosa sobre el rendimiento de los productos. Pueden sesgar análisis basados en promedios simples o afectar modelos que asumen distribuciones normales.")

print("\n### Estrategias Potenciales para el Manejo de Outliers:")
print("1.  **Eliminación:** Simple y directa. Adecuada si los outliers son pocos y claramente errores. Riesgo: pérdida de datos y posible distorsión de la distribución si los outliers son genuinos.")
print("2.  **Transformación:** Aplicar transformaciones (ej., logarítmica, raíz cuadrada) para reducir el sesgo y la influencia de los valores extremos. Útil para columnas con distribuciones muy asimétricas (como rating_count).")
print("3.  **Capping/Winsorizing:** Reemplazar los valores extremos por un percentil especificado (ej., 5º y 95º percentil). Mantiene el número de filas pero reduce la variabilidad extrema. Menos agresivo que la eliminación.")
print("4.  **Mantener Outliers:** Si los outliers representan eventos o características reales e importantes (ej., un producto con un número excepcionalmente alto de reseñas por ser un best-seller), mantenerlos puede ser crucial para una comprensión completa del dataset.")

print("\n### Propuesta de Estrategia Preliminar para este Proyecto:")
print("Dada la naturaleza de los datos de e-commerce y los tipos de outliers encontrados:")
print("1.  **Precios (discounted_price, actual_price):** Los outliers identificados por IQR son numerosos. Es probable que muchos representen productos de gama alta legítimos. Eliminar tantos datos no es deseable. El Z-score identifica los precios más extremos, que *podrían* ser errores. Sugiero investigar manualmente algunos de los outliers de Z-Score más extremos para verificar su validez. Para análisis o modelos sensibles a la escala, se podría considerar la **transformación logarítmica** de estas columnas en lugar de la eliminación o capping masivo.")
print("2.  **Porcentaje de Descuento (discount_percentage):** El rango ya fue verificado (0-100). Los outliers identificados (principalmente por IQR) pueden representar descuentos muy agresivos. Es importante entender si son válidos. Podríamos **mantener estos outliers** inicialmente, a menos que una investigación manual revele errores claros, ya que pueden ser informativos sobre la estrategia de precios.")
print("3.  **Rating (rating):** Los outliers identificados (solo por IQR en los extremos de la escala 1-5) son pocos. Dado que la escala es limitada, **mantener estos valores** parece razonable, ya que representan ratings reales (1 estrella o 5 estrellas). La transformación no es necesaria para esta columna.")
print("4.  **Conteo de Ratings (rating_count):** Esta columna muestra una fuerte asimetría y muchos outliers según IQR y Z-Score. Estos outliers representan productos muy populares y son datos valiosos. **Eliminarlos sería perjudicial.** Una **transformación logarítmica** es la estrategia más adecuada para manejar la asimetría y reducir la influencia de los valores extremos en análisis o modelos, mientras se conserva toda la información.")

print("\n### Estrategia Propuesta Resumida:")
print("1.  **Precios:** Investigar outliers Z-Score extremos. Considerar **transformación logarítmica** para análisis sensibles.")
print("2.  **Porcentaje de Descuento:** **Mantener** (a menos que se identifiquen errores manuales).")
print("3.  **Rating:** **Mantener**.")
print("4.  **Conteo de Ratings:** Aplicar **transformación logarítmica**.")

print("\nEsta estrategia busca equilibrar la necesidad de manejar valores extremos para el análisis estadístico y el modelado con la importancia de conservar datos reales y potencialmente informativos.")
```

Resumen: Identificación de Outliers en Datos de Comercio Electrónico
Este análisis se centró en identificar outliers (valores atípicos) en variables numéricas clave:
discounted_price
actual_price
discount_percentage
rating
rating_count
Se trabajó con los DataFrames productos y resenas, aplicando dos métodos:
Puntuación Z (Z-Score) con umbral |Z| > 3
Rango Intercuartílico (IQR) con regla 1.5*IQR

🔑 Hallazgos Clave
Variación según el método:
Productos: 44 outliers por Z-Score vs. 219 por IQR
Reseñas: 38 outliers por Z-Score vs. 140 por IQR
Naturaleza de los outliers:
Z-Score → valores más extremos.
IQR → incluye valores alejados de la mediana, incluso si no son muy extremos.
Patrones específicos:
rating_count: muy asimétrico. Pocos productos concentran la mayoría de las reseñas.
Variables de precio (discounted_price, actual_price): gran número de outliers con IQR, especialmente en productos de gama alta.
Límites calculados (IQR):
Se agregaron columnas con los límites inferior y superior en ambos DataFrames para todas las variables analizadas.

🚀 Perspectivas y Próximos Pasos
La estrategia de manejo de outliers debe adaptarse a cada variable según su naturaleza y el tipo de análisis o modelo que se aplique.
Transformación logarítmica es recomendable para:
rating_count (muy sesgado)
Variables de precio (para reducir el impacto de los valores extremos)
Mantener outliers válidos:
Ratings extremos (1 y 5 estrellas) → reflejan experiencias reales.
Productos con muchas reseñas → representan popularidad y no deben eliminarse.
En resumen, los outliers en comercio electrónico no siempre son errores; muchas veces son informativos y aportan valor al análisis.

9. Fusión de Tablas y Reordenamiento de Columnas
En este paso se realizó la fusión de los DataFrames resenas y productos usando la clave común product_id.
Se aplicó un inner join, lo cual asegura que solo se incluyan:
Productos que tienen al menos una reseña.
Reseñas que corresponden a un producto existente en la tabla de productos.
De esta manera, se mantiene la coherencia entre ambos conjuntos de datos y se eliminan entradas huérfanas.

📐 Reordenamiento de Columnas
Posteriormente, se definió un orden lógico de columnas para facilitar el análisis, agrupando:
Información del usuario y reseñas (user_id, user_name, review_id, review_title, review_content)
Información del producto (product_id, product_name, category, about_product)
Variables numéricas de interés (discounted_price, actual_price, discount_percentage, rating, rating_count)

✅ Resultados de la Fusión
Forma del DataFrame fusionado: (1194, 14)
Columnas resultantes en el orden solicitado:

codigo:
```python
import pandas as pd

# Fusionar los DataFrames resenas y productos usando 'product_id'
# Usamos un inner join para incluir solo los productos que tienen reseñas y las reseñas que corresponden a un producto existente
# Aseguramos que resenas es la tabla izquierda para mantener sus columnas primero por defecto antes de reordenar
df_fusionado = pd.merge(resenas, productos, on='product_id', how='inner')

# Definir el orden deseado de las columnas
# Asegúrate de que los nombres de las columnas coinciden exactamente con los de tus DataFrames
orden_columnas = [
    'user_id',
    'user_name',
    'review_id', # Corregido de 'Review_id'
    'review_title', # Corregido de 'Review_title'
    'review_content', # Corregido de 'Review_content'
    'product_id', # Corregido de 'Product_id'
    'product_name',
    'category',
    'discounted_price',
    'actual_price',
    'discount_percentage',
    'about_product',
    'rating', # Corregido de 'Rating'
    'rating_count' # Corregido de 'Rating_count'
]

# Reordenar las columnas del DataFrame fusionado
# Verificar que todas las columnas deseadas existen en el DataFrame fusionado
columnas_existentes = [col for col in orden_columnas if col in df_fusionado.columns]

# Reordenar el DataFrame con las columnas existentes en el orden especificado
df_fusionado = df_fusionado[columnas_existentes]


print("✅ Tablas fusionadas exitosamente y columnas reordenadas.")
print(f"\nForma del DataFrame fusionado: {df_fusionado.shape}")

print("\nColumnas del DataFrame fusionado en el orden solicitado:")
print(df_fusionado.columns.tolist())

print("\nPrimeras filas del DataFrame fusionado:")
display(df_fusionado.head())
```

Guardado y Renombramiento del DataFrame Final
Una vez fusionadas las tablas de productos y reseñas, se procedió a:
Guardar el DataFrame df_fusionado en un archivo CSV llamado: tabla union.csv

codigo:
```python
# Guardar el DataFrame fusionado a un archivo CSV
df_fusionado.to_csv("tabla union.csv", index=False)

print("✅ DataFrame fusionado guardado exitosamente como 'tabla union.csv'")
```

codigo:
```python
import pandas as pd

# Verificar si el DataFrame df_fusionado existe antes de intentar renombrarlo
if 'df_fusionado' in globals() and isinstance(globals()['df_fusionado'], pd.DataFrame):
    # Cambiar el nombre del DataFrame
    tabla_union = df_fusionado

    # Opcional: Eliminar el DataFrame antiguo para evitar confusiones (solo si fue exitoso el renombramiento)
    # Usamos un bloque try-except en caso de que del falle por alguna razón inesperada
    try:
        del df_fusionado
        print("   ✅ DataFrame 'df_fusionado' eliminado después de renombrar.")
    except NameError:
        print("   ℹ️ El DataFrame 'df_fusionado' no pudo ser eliminado, podría no haber existido previamente.")


    print("✅ El DataFrame 'df_fusionado' ha sido renombrado exitosamente a 'tabla_union'.")

    # Verificar si el nuevo DataFrame tabla_union existe
    if 'tabla_union' in globals() and isinstance(globals()['tabla_union'], pd.DataFrame):
        print("Verificación: El DataFrame 'tabla_union' está disponible.")
        print(f"Forma de 'tabla_union': {tabla_union.shape}") # Mostrar la forma como verificación
    else:
        print("⚠️ Error: El DataFrame 'tabla_union' no parece haberse creado correctamente.")

else:
    print("❌ Error: El DataFrame 'df_fusionado' no se encontró en el entorno.")
    print("Asegúrate de haber ejecutado la celda donde se crea df_fusionado.")
```

10. Agrupar datos según rangos de discount_percentage
🎯 Objetivo
Dividir los productos en rangos de descuento basados en los cuartiles de la distribución de discount_percentage.
Esto permite analizar cómo varían métricas clave (precio, descuentos, ratings) según los distintos niveles de rebaja.

🔹 Cálculo de Cuartiles
Se calcularon los cuartiles (Q1, Q2 y Q3) de la variable discount_percentage:
Q1 (25%): 31.25
Q2 (50% o Mediana): 49.00
Q3 (75%): 62.00
📌 Interpretación:
El 25% de los productos tiene un descuento ≤ 31.25%.
El 50% de los productos tiene un descuento ≤ 49% (mediana).
El 75% de los productos tiene un descuento ≤ 62%.

🔹 Definición de Rangos (Bins)
Con los valores de los cuartiles y los extremos de la variable se generaron cuatro intervalos de descuento:
0–31%: Descuentos bajos
31–49%: Descuentos moderados
49–62%: Descuentos altos
62–94%: Descuentos muy altos
👉 Se creó una nueva columna llamada discount_range en el DataFrame tabla_union.

🔹 Agrupación y Cálculo de Métricas
Se agruparon los productos por discount_range y se calcularon las siguientes métricas por grupo:
Número de productos distintos
Precio promedio con descuento (discounted_price)
Precio promedio original (actual_price)
Promedio del porcentaje de descuento
Rating promedio

📊 Resultados

📌 Conclusiones
A medida que el descuento aumenta, el precio promedio con descuento disminuye notablemente.
Los productos con mayores descuentos (62–94%) tienen un precio original alto, pero terminan con precios finales muy bajos.
El rating promedio se mantiene estable (~4.0–4.2) en todos los rangos, lo que sugiere que los consumidores valoran de forma similar los productos independientemente del nivel de descuento.
✅ Este análisis por rangos facilita la identificación de estrategias de pricing y el comportamiento del mercado frente a diferentes niveles de descuento.

codigo:
q1_discount = tabla_union['discount_percentage'].quantile(0.25)
q2_discount = tabla_union['discount_percentage'].quantile(0.50)
q3_discount = tabla_union['discount_percentage'].quantile(0.75)

print(f"Primer cuartil (Q1) de discount_percentage: {q1_discount:.2f}")
print(f"Segundo cuartil (Q2) o Mediana de discount_percentage: {q2_discount:.2f}")
print(f"Tercer cuartil (Q3) de discount_percentage: {q3_discount:.2f}")

codigo:
```python
# Defina los contenedores utilizando los cuartiles calculados y los valores mínimos/máximos
min_discount = tabla_union['discount_percentage'].min()
max_discount = tabla_union['discount_percentage'].max()

bins = [min_discount, q1_discount, q2_discount, q3_discount, max_discount]

# Definir las etiquetas para los rangos
labels = [f'{min_discount:.0f}-{q1_discount:.0f}',
          f'{q1_discount:.0f}-{q2_discount:.0f}',
          f'{q2_discount:.0f}-{q3_discount:.0f}',
          f'{q3_discount:.0f}-{max_discount:.0f}']

# Asegúrese de que los contenedores sean únicos y estén ordenados si hay valores de cuartiles idénticos o mínimos/máximos en los cuartiles
unique_bins = sorted(list(set(bins)))

# Ajuste las etiquetas si los contenedores resultaron en menos límites únicos de lo esperado
if len(unique_bins) - 1 != len(labels):
    # Una forma más robusta de crear etiquetas basadas en contenedores únicos
    labels = [f'{unique_bins[i]:.0f}-{unique_bins[i+1]:.0f}' for i in range(len(unique_bins)-1)]
    print(f"Etiquetas ajustadas debido a contenedores no únicos: {labels}")


# Cree la columna 'discount_range' usando pd.cut
tabla_union['discount_range'] = pd.cut(tabla_union['discount_percentage'],
                                       bins=unique_bins,
                                       labels=labels,
                                       include_lowest=True, # Incluya el valor más bajo en el primer contenedor
                                       right=True) # Los intervalos son (abiertos, cerrados] excepto el primero


print("✅ Nueva columna 'discount_range' creada.")

# Mostrar las primeras filas para verificar la nueva columna
print("\nPrimeras filas del DataFrame con la columna 'discount_range':")
display(tabla_union.head())
```

codigo:
tabla_union_grouped = tabla_union.groupby('discount_range')

print("✅ DataFrame 'tabla_union' agrupado por 'discount_range'.")
print("\nEl objeto GroupBy está listo para calcular métricas agregadas.")

codigo:
discount_range_metrics = tabla_union_grouped.agg(
    num_productos=('product_id', 'nunique'),
    avg_discounted_price=('discounted_price', 'mean'),
    avg_actual_price=('actual_price', 'mean'),
    avg_discount_percentage=('discount_percentage', 'mean'),
    avg_rating=('rating', 'mean')
)

print("✅ Métricas calculadas para cada rango de descuento.")
print("\nMétricas por rango de descuento:")
display(discount_range_metrics)

codigo:
```python
print("Métricas calculadas por rango de descuento:")
display(discount_range_metrics)
```

11. Análisis de Variables Categóricas
🎯 Objetivo
Analizar variables categóricas creadas a partir de transformaciones de datos numéricos, en este caso:
discount_range (definido con base en cuartiles).
rating_category (definido en intervalos fijos de rating).

🔹 Hallazgos Previos
Los cuartiles de discount_percentage fueron:
Q1: 31.25
Q2 (Mediana): 49.00
Q3: 62.00
Estos valores se usaron para crear rangos de descuento (discount_range), con los que se calcularon métricas clave (número de productos, precios promedios, descuentos y rating).

🔹 Análisis de la Variable rating
Se calcularon los valores básicos de la columna rating en todo el dataset:
Valor máximo: 5.00
Valor mínimo: 2.00
Valor promedio: 4.08
📌 Interpretación: la mayoría de los productos tienen calificaciones cercanas a 4 estrellas, con pocos casos en los extremos.

🔹 Creación de Categorías de Rating
Para complementar el análisis, se creó la variable categórica rating_category con base en intervalos fijos:
Bajo (1–3)
Medio (3–4)
Alto (4–5)

📊 Resultados de la Categorización
Bajo (1–3): 6 productos
Medio (3–4): 318 productos
Alto (4–5): 870 productos
👉 Esto confirma que la mayoría de los productos están bien valorados (más del 70% tienen rating ≥ 4).

🔹 Ejemplo de la Nueva Columna en tabla_union
Las primeras filas muestran la columna rating_category correctamente agregada:

📌 Conclusiones
La distribución de ratings está sesgada hacia lo alto, lo cual refleja satisfacción de los clientes.
Solo una fracción mínima de productos cae en la categoría Bajo (1–3).
La combinación de variables categóricas (discount_range + rating_category) permitirá más adelante analizar si existe relación entre descuentos aplicados y valoraciones de clientes.

codigo:
```python
# Calcular el valor máximo, mínimo y promedio de la columna 'rating' en tabla_union
max_rating = tabla_union['rating'].max()
min_rating = tabla_union['rating'].min()
mean_rating = tabla_union['rating'].mean()

print(f"Valor máximo de Rating: {max_rating:.2f}")
print(f"Valor mínimo de Rating: {min_rating:.2f}")
print(f"Valor promedio de Rating: {mean_rating:.2f}")
```

codigo:
```python
import pandas as pd

# Definir los límites de los rangos de rating para el Ejemplo 1
# Aseguramos que el límite superior incluye el 5.0
bins_rating = [1, 3, 4, 5.1]

# Definir las etiquetas para los rangos
labels_rating = ['Bajo (1-3)', 'Medio (3-4)', 'Alto (4-5)']

# Crear la nueva columna 'rating_category' en tabla_union
tabla_union['rating_category'] = pd.cut(tabla_union['rating'],
                                       bins=bins_rating,
                                       labels=labels_rating,
                                       include_lowest=True, # Incluir el valor más bajo (1) en el primer bin
                                       right=False) # Los intervalos son [inicio, fin) excepto el último

# Mostrar los conteos por categoría
print("Conteo de registros por categoría de Rating (Rangos Fijos):")
display(tabla_union['rating_category'].value_counts().sort_index())

# Mostrar las primeras filas con la nueva columna
print("\nPrimeras filas del DataFrame con la nueva columna 'rating_category':")
display(tabla_union.head())
```

codigo:
```python
# Guardar el DataFrame tabla_union a un archivo CSV
tabla_union.to_csv("tabla union.csv", index=False)

print("✅ DataFrame 'tabla_union' guardado exitosamente como 'tabla union.csv'")
```

codigo:
```python
# Mostrar el encabezado de la tabla_union
print("Encabezado de la tabla_union:")
display(tabla_union.head())
```

12. Medidas de Tendencia Central y Dispersión
🎯 Objetivo
Calcular estadísticas descriptivas básicas de las variables numéricas más relevantes del DataFrame tabla_union, con el fin de entender su distribución y variabilidad.
🔹 Variables Analizadas
discounted_price
actual_price
discount_percentage
rating
rating_count
🔹 Medidas Calculadas
Para cada variable se calcularon las siguientes métricas:
Media (Promedio)
Mediana (Percentil 50)
Moda (valor más frecuente)
Desviación Estándar

codigo:
```python
# Definir las columnas numéricas de interés
columnas_numericas = ['discounted_price', 'actual_price', 'discount_percentage', 'rating', 'rating_count']

# Crear un diccionario para almacenar los resultados
metricas_numericas = {}

# Calcular las métricas para cada columna
for col in columnas_numericas:
    if col in tabla_union.columns:
        mean_val = tabla_union[col].mean()
        median_val = tabla_union[col].median()
        # Moda puede devolver múltiples valores, tomamos el primero si existe
        mode_val = tabla_union[col].mode().tolist() if not tabla_union[col].mode().empty else None
        std_val = tabla_union[col].std()

        metricas_numericas[col] = {
            'Media': mean_val,
            'Mediana': median_val,
            'Moda': mode_val,
            'Desviación Estándar': std_val
        }
    else:
        print(f"⚠️ Advertencia: La columna '{col}' no se encontró en la tabla_union.")

# Convertir el diccionario a un DataFrame para una mejor visualización
df_metricas = pd.DataFrame(metricas_numericas).T # Transponemos para que las columnas sean las variables

print("📊 Medidas de Tendencia Central y Dispersión para Variables Numéricas:")
display(df_metricas)
```

Interpretación de los Resultados:
Como experto en análisis de datos, aquí está mi interpretación de las métricas calculadas:
discounted_price y actual_price:
La Media es significativamente más alta que la Mediana para ambas columnas. Esto sugiere que la distribución de los precios (tanto con descuento como real) está sesgada positivamente (hacia la derecha). Hay una "cola" de productos con precios más altos que elevan la media por encima del valor central (mediana).
La Moda para discounted_price y actual_price indica los precios más frecuentes. Para discounted_price, el precio más común es 199.0, mientras que para actual_price es 999.0. Esto sugiere que hay una gran cantidad de productos de menor precio en el dataset.
La Desviación Estándar es relativamente alta para ambas columnas (especialmente actual_price), lo que confirma una alta variabilidad en los precios de los productos. Esto es esperado en un catálogo diverso de comercio electrónico.
discount_percentage:
La Media (aprox. 49.8%) y la Mediana (49.0%) son bastante cercanas. Esto sugiere que la distribución de los porcentajes de descuento es relativamente simétrica alrededor del centro, aunque puede haber un ligero sesgo.
La Moda en 50.0% indica que un descuento del 50% es el más frecuente en este dataset.
La Desviación Estándar (aprox. 21.2%) muestra una dispersión moderada en los porcentajes de descuento. Hay una variedad razonable de descuentos aplicados a través de los productos.
rating:
La Media (aprox. 4.08), Mediana (4.20) y Moda (4.10) están muy juntas y son valores altos (cercanos a 4 y 5 en una escala de 1 a 5). Esto indica que la distribución de los ratings está fuertemente sesgada negativamente (hacia la izquierda), concentrándose en las calificaciones altas. La mayoría de los productos reciben ratings positivos.
La Desviación Estándar (aprox. 0.37) es baja, lo que confirma que los ratings están muy agrupados alrededor de la media/mediana alta. Hay poca variabilidad en las calificaciones (la mayoría son buenas).
rating_count:
La Media (aprox. 22400) es drásticamente mayor que la Mediana (4118.0). Esto es un claro indicador de una distribución altamente sesgada positivamente (hacia la derecha). La mayoría de los productos tienen un número de reseñas relativamente bajo, pero hay una minoría de productos extremadamente populares con un número de reseñas muy alto que inflan la media.
La Moda (2) sugiere que el número más frecuente de reseñas para un producto es muy bajo.
La Desviación Estándar (aprox. 54300) es muy alta, lo que resalta la enorme variabilidad en el número de reseñas entre productos. Esta es la variable con mayor dispersión relativa, confirmando la presencia de "best-sellers" con muchísimas reseñas.
Conclusión:
El análisis de estas métricas confirma la naturaleza típica de los datos de comercio electrónico: precios con cierto sesgo hacia arriba, una distribución de descuentos más centrada, ratings predominantemente altos y un conteo de reseñas muy asimétrico, con pocos productos concentrando la mayoría de las interacciones de los usuarios. Estas observaciones son cruciales para decidir qué análisis son apropiados y cómo manejar posibles outliers o transformaciones para modelado.

13. Visualización de la Distribución de Variables Numéricas
🎯 Objetivo
Analizar gráficamente la distribución de las principales variables numéricas del DataFrame tabla_union, para identificar sesgos, concentraciones de valores y posibles outliers.
🔹 Variables Consideradas
discounted_price
actual_price
discount_percentage
rating
rating_count

codigo:
```python
import matplotlib.pyplot as plt
import seaborn as sns

# Definir las columnas numéricas para visualizar
columnas_numericas = ['discounted_price', 'actual_price', 'discount_percentage', 'rating', 'rating_count']

# Configurar el estilo de los gráficos
sns.set_style("whitegrid")

# Crear un histograma para cada columna
for col in columnas_numericas:
    if col in tabla_union.columns:
        plt.figure(figsize=(10, 6)) # Tamaño del gráfico
        sns.histplot(data=tabla_union, x=col, kde=True) # kde=True añade la curva de densidad estimada
        plt.title(f'Distribución de {col}')
        plt.xlabel(col)
        plt.ylabel('Frecuencia')
        plt.show()
    else:
        print(f"⚠️ Advertencia: La columna '{col}' no se encontró en la tabla_union, no se puede crear el histograma.")
```

Interpretación de las Distribuciones (Basado en los Histogramas):
Una vez ejecutado el código anterior y visualizados los histogramas, aquí tienes una interpretación general de los tipos de distribución que probablemente observarás para cada variable:
discounted_price:
Tipo de Distribución: Probablemente sesgada a la derecha (positivamente). La mayoría de los productos se concentrarán en precios más bajos, con una "cola" larga extendiéndose hacia precios más altos. Esto es común en datos de precios, donde hay muchos artículos de precio bajo a medio y menos artículos de precio muy alto.
actual_price:
Tipo de Distribución: Similar a discounted_price, es muy probable que esté sesgada a la derecha (positivamente). La diferencia entre el precio real y el de descuento no cambia fundamentalmente la forma general de la distribución de precios, aunque la dispersión podría ser ligeramente diferente. La mayoría de los productos tendrán precios reales más bajos.
discount_percentage:
Tipo de Distribución: Podría ser aproximadamente simétrica o tener un ligero sesgo, pero probablemente no tan pronunciado como las variables de precio. Nuestros cálculos de medidas de tendencia central ya sugerían una distribución más centrada. Es posible que veas picos en porcentajes de descuento comunes (como 10%, 20%, 50%, etc.).
rating:
Tipo de Distribución: Fuertemente sesgada a la izquierda (negativamente). La gran mayoría de los ratings se concentrarán en los valores más altos (4, 5), con una "cola" más pequeña hacia los ratings más bajos (1, 2, 3). Esto es muy típico en reseñas de productos, donde los clientes satisfechos tienden a dejar ratings altos.
rating_count:
Tipo de Distribución: Altamente sesgada a la derecha (positivamente). Verás un gran pico en los valores bajos (productos con pocas reseñas) y una "cola" extremadamente larga hacia la derecha, representando esa minoría de productos muy populares con miles o decenas de miles de reseñas. Esta es una de las distribuciones más asimétricas en conjuntos de datos de este tipo.
Consideraciones:
La curva KDE (Estimación de Densidad del Kernel) en los histogramas te dará una idea suavizada de la forma de la distribución.
La forma exacta puede variar ligeramente dependiendo de cómo se generen los "bins" del histograma, pero el patrón general de sesgo debería ser claro.

codigo:
```python
import matplotlib.pyplot as plt
import seaborn as sns

# Definir las columnas numéricas para visualizar
columnas_numericas = ['discounted_price', 'actual_price', 'discount_percentage', 'rating', 'rating_count']

# Configurar el estilo de los gráficos
sns.set_style("whitegrid")

# Crear un gráfico de densidad para cada columna
for col in columnas_numericas:
    if col in tabla_union.columns:
        plt.figure(figsize=(10, 6)) # Tamaño del gráfico
        sns.kdeplot(data=tabla_union, x=col, fill=True) # fill=True rellena el área bajo la curva
        plt.title(f'Gráfico de Densidad de {col}')
        plt.xlabel(col)
        plt.ylabel('Densidad')
        plt.show()
    else:
        print(f"⚠️ Advertencia: La columna '{col}' no se encontró en la tabla_union, no se puede crear el gráfico de densidad.")
```

Interpretación de la Simetría/Asimetría (Basado en los Gráficos de Densidad):
Observando los gráficos de densidad (curvas KDE) que hemos generado (y que también se mostraban sobre los histogramas):
discounted_price:
Simetría: Asimétrica positiva (sesgada a la derecha). La cola de la distribución se extiende mucho más hacia los valores altos que hacia los bajos. Esto indica que hay una concentración de productos con precios de descuento bajos, y menos productos con precios de descuento altos.
actual_price:
Simetría: Asimétrica positiva (sesgada a la derecha). Similar a discounted_price, la curva de densidad muestra una cola larga hacia los valores de precio real altos. La mayoría de los productos tienen precios reales bajos.
discount_percentage:
Simetría: Aproximadamente simétrica, aunque puede tener ligeras irregularidades o múltiples picos pequeños. La curva de densidad no muestra una cola larga y pronunciada hacia un lado en particular, sugiriendo que los porcentajes de descuento se distribuyen de manera más equilibrada alrededor de su media/mediana.
rating:
Simetría: Asimétrica negativa (sesgada a la izquierda). La mayor parte de la masa de la distribución se encuentra en los valores altos, con una cola que se extiende hacia los valores bajos. Esto refleja que la mayoría de los productos reciben calificaciones altas.
rating_count:
Simetría: Altamente asimétrica positiva (sesgada a la derecha). La curva de densidad tiene un pico muy alto cerca de cero y una cola extremadamente larga y plana que se extiende hacia los valores muy altos. Esto subraya la gran disparidad en el número de reseñas, con la mayoría de los productos teniendo muy pocas y un pequeño número teniendo muchísimas.
Conclusión sobre la Simetría:
Las variables de precio y rating_count presentan una asimetría marcada, lo cual es típico en datos de ventas y popularidad. La variable rating también es asimétrica, pero hacia el lado de las calificaciones altas. Solo discount_percentage muestra una distribución más cercana a la simetría. Reconocer esta asimetría es importante para elegir métodos de análisis adecuados o considerar transformaciones si es necesario.

14. Calcular medidas de dispersión

codigo:
```python
# Definir las columnas numéricas de interés
columnas_numericas = ['discounted_price', 'actual_price', 'discount_percentage', 'rating', 'rating_count']

# Crear un diccionario para almacenar los resultados
medidas_dispersion = {}

# Calcular la varianza y el IQR para cada columna
for col in columnas_numericas:
    if col in tabla_union.columns:
        varianza_val = tabla_union[col].var()
        q1 = tabla_union[col].quantile(0.25)
        q3 = tabla_union[col].quantile(0.75)
        iqr_val = q3 - q1

        medidas_dispersion[col] = {
            'Varianza': varianza_val,
            'Rango Intercuartílico (IQR)': iqr_val
        }
    else:
        print(f"⚠️ Advertencia: La columna '{col}' no se encontró en la tabla_union.")

# Convertir el diccionario a un DataFrame para una mejor visualización
df_dispersion = pd.DataFrame(medidas_dispersion).T # Transponemos para que las columnas sean las variables

print("📊 Medidas de Dispersión para Variables Numéricas:")
display(df_dispersion)
```

Explicación de las Medidas de Dispersión:
Varianza:
Qué describe: La varianza mide cuánto se dispersan los valores individuales con respecto a la media. Un valor de varianza alto indica que los puntos de datos están muy alejados de la media y entre sí (alta dispersión), mientras que un valor bajo indica que los puntos de datos están agrupados cerca de la media (baja dispersión).
Interpretación en tus datos:
discounted_price y actual_price tienen valores de varianza muy altos, lo que confirma lo que vimos en los histogramas y la desviación estándar: hay una gran variabilidad en los precios. Esto es de esperar en un catálogo de productos diverso.
discount_percentage tiene una varianza menor en comparación con los precios, lo que sugiere que los porcentajes de descuento están menos dispersos que los precios reales.
rating tiene una varianza muy baja, lo que indica que los ratings están muy concentrados alrededor de la media (entre 4 y 5), confirmando la poca variabilidad y el sesgo hacia ratings altos.
rating_count tiene una varianza extremadamente alta, lo que resalta la enorme dispersión en el número de reseñas. Esto se debe a la presencia de productos con un número de reseñas drásticamente mayor que la mayoría.
Rango Intercuartílico (IQR):
Qué describe: El IQR es la diferencia entre el tercer cuartil (Q3) y el primer cuartil (Q1). Representa el rango en el que se encuentra el 50% central de los datos. Es una medida de dispersión robusta a los outliers, ya que no se ve afectada por los valores extremos en las colas de la distribución. Un IQR grande indica que el 50% central de los datos está muy disperso, mientras que un IQR pequeño indica que están más agrupados.
Interpretación en tus datos:
discounted_price y actual_price tienen IQRs considerablemente grandes (comparados con su escala), lo que indica que el 50% central de los precios tiene un rango amplio.
discount_percentage tiene un IQR que muestra el rango del 50% central de los descuentos.
rating tiene un IQR muy pequeño. Esto refuerza que el 50% central de los ratings está muy agrupado (entre 3.9 y 4.2 aproximadamente, basándonos en Q1 y Q3 de turnos anteriores), confirmando nuevamente la poca variabilidad en la parte central de la distribución de ratings.
rating_count tiene un IQR que, aunque no tan alto como la varianza, sigue siendo significativo, mostrando el rango del 50% central de los conteos de reseñas. Sin embargo, la gran diferencia entre la varianza (muy afectada por los extremos) y el IQR (robusto a extremos) para rating_count es una señal clara de la asimetría y la presencia de outliers.
En resumen, tanto la varianza como el IQR nos dan una idea de la dispersión de los datos, pero el IQR es especialmente útil para entender la variabilidad en la parte central de la distribución y es menos sensible a la influencia de los valores atípicos extremos.

15. Calcular correlaciones

codigo:
```python
# Definir las columnas numéricas para calcular la correlación
columnas_numericas = ['discounted_price', 'actual_price', 'discount_percentage', 'rating', 'rating_count']

# Calcular la matriz de correlación de Pearson
# Asegurarse de que las columnas existen antes de seleccionarlas
columnas_existentes = [col for col in columnas_numericas if col in tabla_union.columns]

if len(columnas_existentes) > 1:
    correlacion_pearson = tabla_union[columnas_existentes].corr(method='pearson')

    print("📊 Matriz de Correlación de Pearson:")
    display(correlacion_pearson)
else:
    print("⚠️ No hay suficientes columnas numéricas existentes para calcular la correlación.")
    print(f"Columnas encontradas: {columnas_existentes}")
```

codigo:
```python
import matplotlib.pyplot as plt
import seaborn as sns

# Asegurarse de que la matriz de correlación existe
if 'correlacion_pearson' in globals() and isinstance(globals()['correlacion_pearson'], pd.DataFrame):
    print("📊 Mapa de Calor de la Matriz de Correlación:")

    plt.figure(figsize=(10, 8)) # Tamaño del gráfico
    sns.heatmap(correlacion_pearson, annot=True, cmap='coolwarm', fmt=".2f", linewidths=.5)
    plt.title('Mapa de Calor de la Matriz de Correlación de Pearson')
    plt.show()
else:
    print("⚠️ La matriz de correlación de Pearson no se encontró. Por favor, calcula la matriz primero.")
```

Interpretación del Mapa de Calor de Correlación:
Una vez que veas el mapa de calor generado, aquí tienes una interpretación de las relaciones entre las variables numéricas:
Colores:
Los colores cálidos (rojo/naranja) indican una correlación positiva. Cuanto más intenso el color, más fuerte es la relación positiva.
Los colores fríos (azul/morado) indican una correlación negativa. Cuanto más intenso el color, más fuerte es la relación negativa.
Los colores cercanos al blanco o gris indican una correlación cercana a cero (poca o ninguna relación lineal).
Valores (annot=True): Los números dentro de cada celda son los coeficientes de correlación de Pearson.
Interpretaciones Clave (basadas en la matriz calculada anteriormente):
discounted_price vs actual_price: Deberías ver un color cálido muy intenso (coeficiente cercano a +1). Esto es lógico, ya que el precio con descuento está directamente relacionado con el precio original. Los productos con precios originales altos tienden a tener precios con descuento altos, y viceversa.
discount_percentage vs discounted_price / actual_price: Deberías observar colores fríos (correlación negativa), probablemente de intensidad baja a moderada. Esto significa que, en general, a medida que aumenta el porcentaje de descuento, los precios (tanto el original como el de descuento) tienden a ser menores. Sin embargo, la correlación no es muy fuerte, lo que sugiere que no solo los productos baratos tienen grandes descuentos, sino que también puede haber productos caros con descuentos significativos (o viceversa), lo cual es realista en el comercio electrónico.
rating vs discount_percentage: Es probable que veas una correlación negativa baja (color frío pálido). Esto sugeriría que los productos con mayores descuentos tienden a tener ratings ligeramente más bajos, o que los productos con ratings más altos tienden a tener menores descuentos. Sin embargo, la relación lineal es débil.
rating vs discounted_price / actual_price: Deberías ver correlaciones positivas muy bajas o cercanas a cero (colores muy pálidos o blancos/grises). Esto indica que no hay una relación lineal fuerte entre el precio de un producto y su calificación promedio. Los productos caros no necesariamente tienen mejores ratings que los baratos, ni viceversa.
rating vs rating_count: Es probable que veas una correlación positiva baja (color cálido pálido). Esto podría sugerir que los productos con más reseñas tienden a tener ratings ligeramente más altos, pero la relación lineal no es fuerte. Otros factores influyen más en el rating.
rating_count vs discounted_price / actual_price: Deberías ver correlaciones muy cercanas a cero o negativas muy débiles. Esto indica que no hay una relación lineal fuerte entre el precio de un producto y el número de reseñas que recibe. Los productos más caros no necesariamente tienen más o menos reseñas que los productos más baratos.
En resumen: El mapa de calor te confirmará las relaciones obvias (precios entre sí) y te mostrará que las relaciones lineales entre descuento, rating y conteo de reseñas son generalmente débiles en este dataset. Esto no significa que no existan otras formas de relación (no lineales) o que no haya patrones al analizar por categorías, pero linealmente, no hay dependencias fuertes entre estas variables.

Análisis de la Pregunta de Negocio: ¿Los productos con mayores descuentos tienen mejores calificaciones?
Para responder a esta pregunta, nos centraremos en la relación entre el porcentaje de descuento aplicado a un producto y su calificación promedio recibida por los usuarios.
Variables Relevantes:
discount_percentage: El porcentaje de descuento ofrecido.
rating: La calificación promedio del producto.
Análisis de Sustento:
Para entender esta relación, agrupamos los productos por rangos de porcentaje de descuento. Utilizamos los rangos definidos previamente basados en los cuartiles de discount_percentage (0-31%, 31-49%, 49-62%, 62-94%). Luego, calculamos el rating promedio (rating) para todos los productos que caen dentro de cada uno de estos rangos.
La tabla a continuación muestra el rating promedio para cada rango de porcentaje de descuento:

codigo:
```python
# Mostrar la tabla de métricas por rango de descuento (calculada previamente)
# Asegurarse de que la tabla de métricas por rango de descuento existe
if 'discount_range_metrics' in globals() and isinstance(globals()['discount_range_metrics'], pd.DataFrame):
    print("📊 Rating Promedio por Rango de Porcentaje de Descuento:")
    display(discount_range_metrics[['avg_discount_percentage', 'avg_rating']])
else:
    print("⚠️ La tabla 'discount_range_metrics' no se encontró. Por favor, asegúrate de haber calculado las métricas por rango de descuento primero.")
```

Visualización:
Para visualizar esta relación de manera clara, generamos un gráfico de barras que muestra el rating promedio para cada rango de porcentaje de descuento:

codigo:
```python
import matplotlib.pyplot as plt
import seaborn as sns

# Asegurarse de que la tabla de métricas por rango de descuento existe
if 'discount_range_metrics' in globals() and isinstance(globals()['discount_range_metrics'], pd.DataFrame):
    print("📊 Gráfico de Rating Promedio por Rango de Descuento:")

    plt.figure(figsize=(10, 6)) # Tamaño del gráfico
    sns.barplot(x=discount_range_metrics.index, y='avg_rating', data=discount_range_metrics, palette='viridis')
    plt.title('Rating Promedio por Rango de Porcentaje de Descuento')
    plt.xlabel('Rango de Porcentaje de Descuento')
    plt.ylabel('Rating Promedio')
    plt.ylim(0, 5) # Establecer el límite del eje y de 0 a 5 (escala de rating)
    plt.xticks(rotation=0) # Rotar etiquetas del eje x si son largas
    plt.tight_layout() # Ajustar el diseño para evitar que las etiquetas se solapen
    plt.show()

else:
    print("⚠️ La tabla 'discount_range_metrics' no se encontró. Por favor, asegúrate de haber calculado las métricas por rango de descuento primero.")
```

Interpretación de los Resultados y Respuesta a la Pregunta de Negocio:
Observando tanto la tabla de métricas como el gráfico de barras:
El rango de descuento más bajo (0-31%) presenta el rating promedio más alto.
A medida que los rangos de descuento aumentan, el rating promedio tiende a disminuir ligeramente.
Las diferencias en los ratings promedio entre los rangos son relativamente pequeñas, ya que todos los ratings promedio están por encima de 4.0.
Respuesta Final:
Basado en el análisis de los datos de la tabla tabla_union, los productos con mayores porcentajes de descuento generalmente NO tienen mejores calificaciones promedio. De hecho, se observa una ligera tendencia a que los productos con menores descuentos tengan calificaciones promedio marginalmente más altas. Esto sugiere que el porcentaje de descuento no es el factor principal que impulsa un rating alto en este conjunto de datos.

Análisis de la Relación entre Precio y Rating
Objetivo
Responder a la pregunta de negocio:
¿Existe relación entre el precio de un producto y su rating promedio?
Análisis
Se realizaron dos análisis principales:
Visualización mediante diagramas de dispersión (scatter plots) entre:
Precio con Descuento (discounted_price) y Rating (rating).
Precio Real (actual_price) y Rating (rating). Ambos gráficos fueron representados en escala logarítmica para los precios, con líneas de tendencia ajustadas.
Cálculo de Correlaciones:
Correlación Precio con Descuento vs Rating: 0.121
Correlación Precio Real vs Rating: 0.118

codigo:
```python
import matplotlib.pyplot as plt
import seaborn as sns

# ===============================
# Relación entre Precio con Descuento y Rating
# ===============================
plt.figure(figsize=(10, 6))
sns.regplot(
    data=tabla_union,
    x='discounted_price',
    y='rating',
    scatter_kws={'alpha': 0.4},   # transparencia en puntos
    line_kws={'color': 'red'}     # línea de tendencia en rojo
)
plt.xscale("log")  # Escala logarítmica para precios
plt.title('Relación entre Precio con Descuento y Rating Promedio')
plt.xlabel('Precio con Descuento (escala log)')
plt.ylabel('Rating Promedio')
plt.ylim(0, 5.1)
plt.grid(True)
plt.show()

# ===============================
# Relación entre Precio Real y Rating
# ===============================
plt.figure(figsize=(10, 6))
sns.regplot(
    data=tabla_union,
    x='actual_price',
    y='rating',
    scatter_kws={'alpha': 0.4},
    line_kws={'color': 'red'}
)
plt.xscale("log")
plt.title('Relación entre Precio Real y Rating Promedio')
plt.xlabel('Precio Real (escala log)')
plt.ylabel('Rating Promedio')
plt.ylim(0, 5.1)
plt.grid(True)
plt.show()

# ===============================
# Cálculo de correlaciones
# ===============================
corr_discount = tabla_union['discounted_price'].corr(tabla_union['rating'])
corr_actual = tabla_union['actual_price'].corr(tabla_union['rating'])

print("📊 Correlaciones entre precio y rating:")
print(f" - Correlación Precio con Descuento vs Rating: {corr_discount:.3f}")
print(f" - Correlación Precio Real vs Rating: {corr_actual:.3f}")
```

Interpretación de Resultados
Los valores de correlación son positivos pero muy bajos, cercanos a 0.1.
Esto significa que no existe una relación lineal fuerte entre el precio de un producto (ni con descuento ni real) y su calificación promedio.
Los scatter plots muestran que los ratings se concentran entre 3.5 y 5, independientemente del rango de precios, lo cual refuerza la idea de que el precio no es un determinante del rating.
En comercio electrónico, esto sugiere que las calificaciones de los productos dependen más de la calidad percibida, experiencia del usuario y reputación de la marca, que del precio en sí.
Respuesta a la Pregunta de Negocio
No se observa una relación significativa entre el precio de un producto y su calificación promedio.
Tanto los productos de bajo precio como los de alto precio tienden a recibir calificaciones altas (por encima de 4 en promedio), indicando que el precio no es un factor determinante en la satisfacción del cliente reflejada en las reseñas.

top 10 de los productos mas populares

codigo:
```python
# Identificar los productos más populares ordenando por rating_count
# Seleccionaremos, por ejemplo, los 10 productos con mayor rating_count
productos_mas_populares = tabla_union.sort_values(by='rating_count', ascending=False).head(10)

print("📊 Los 10 productos más populares (basado en Rating Count):")

# Seleccionar las columnas relevantes para mostrar: nombre, rating_count y rating
columnas_mostrar = ['product_name', 'rating_count', 'rating']

# Asegurarse de que las columnas existen antes de mostrarlas
columnas_existentes = [col for col in columnas_mostrar if col in productos_mas_populares.columns]

if columnas_existentes:
    display(productos_mas_populares[columnas_existentes])
else:
    print("⚠️ No se pudieron encontrar las columnas relevantes para mostrar.")
```

codigo:
```python
import matplotlib.pyplot as plt
import seaborn as sns

# Asegurarse de que el DataFrame de productos_mas_populares existe
if 'productos_mas_populares' in globals() and isinstance(globals()['productos_mas_populares'], pd.DataFrame):
    print("📊 Gráfico de Rating de los 10 Productos Más Populares:")

    plt.figure(figsize=(12, 7)) # Ajustar tamaño para mejor visualización de nombres
    # Ordenar por rating_count para que el gráfico refleje el orden de popularidad
    productos_mas_populares_sorted = productos_mas_populares.sort_values(by='rating_count', ascending=False)
    sns.barplot(x='product_name', y='rating', data=productos_mas_populares_sorted, palette='viridis')

    plt.title('Rating Promedio de los 10 Productos Más Populares (por Conteo de Reseñas)')
    plt.xlabel('Producto')
    plt.ylabel('Rating Promedio')
    plt.ylim(0, 5) # Escala de rating
    plt.xticks(rotation=90, ha='right') # Rotar nombres de productos para que no se solapen
    plt.tight_layout() # Ajustar el diseño
    plt.show()

else:
    print("⚠️ El DataFrame 'productos_mas_populares' no se encontró. Por favor, identifica los productos más populares primero.")
```

Informe Detallado sobre el Análisis de Datos de Productos y Reseñas de Amazon
Introducción
Este informe presenta un análisis exploratorio detallado de un conjunto de datos de productos y reseñas de la plataforma Amazon. El objetivo principal es comprender las características de los productos y las reseñas, identificar patrones clave y responder a preguntas de negocio relevantes, como la relación entre descuentos, precios y calificaciones de los productos, así como la identificación de los productos más populares y su percepción.
Metodología
El análisis se llevó a cabo siguiendo una serie de pasos clave:
Carga y Exploración Inicial de los Datos: Se cargaron los datos de productos y reseñas en DataFrames de pandas y se realizó una exploración inicial para entender su estructura, columnas y tipos de datos.
Limpieza de Datos: Se abordó la calidad de los datos mediante:
Identificación y manejo de valores nulos.
Conversión de columnas numéricas (precios, porcentaje de descuento, rating, rating_count) a tipos de datos apropiados, eliminando caracteres no numéricos.
Identificación y análisis de registros duplicados en las tablas originales y la tabla fusionada.
Identificación de valores fuera de rango en el porcentaje de descuento.
Identificación de posibles outliers en las variables numéricas utilizando Z-Score e IQR.
Fusión de Tablas: Se combinó la información de productos y reseñas en una única tabla (tabla_union) utilizando la columna product_id como clave común.
Análisis Descriptivo Univariado: Se calcularon y analizaron medidas de tendencia central (media, moda, mediana) y de dispersión (varianza, desviación estándar, rango intercuartílico) para las variables numéricas clave. Se visualizaron las distribuciones de estas variables mediante histogramas y gráficos de densidad.
Análisis de Correlación: Se calculó y visualizó la matriz de correlación de Pearson para entender las relaciones lineales entre las variables numéricas.
Análisis de Preguntas de Negocio Específicas: Se realizaron análisis dirigidos para responder a las preguntas planteadas:
¿Los productos con mayores descuentos tienen mejores calificaciones? (Análisis agrupado por rangos de descuento).
¿Existe relación entre el precio de un producto y su rating promedio? (Análisis de correlación y visualización con scatter plots).
¿Cuáles son los productos más populares y cómo se perciben? (Identificación por conteo de reseñas y análisis de ratings).
Resultados
Resumen de Limpieza de Datos
Durante la etapa de limpieza, se realizaron las siguientes acciones relevantes:
Se identificaron y manejaron valores nulos en columnas como about_product, img_link y product_link.
Se convirtieron columnas como discounted_price, actual_price, discount_percentage, rating y rating_count a tipos numéricos, eliminando símbolos monetarios, comas y porcentajes.
Se identificaron registros duplicados basados en product_id en la tabla de productos y se decidió mantenerlos por el momento tras analizar que probablemente representen variaciones o listados diferentes.
Se identificaron registros duplicados en la tabla fusionada basados en combinaciones de columnas (nombre, descripción, id de producto) y se confirmó que la mayoría representan reseñas únicas para productos.
No se encontraron valores fuera del rango esperado (0-100) en la columna discount_percentage.
Se identificaron outliers en variables numéricas, especialmente en rating_count y precios, utilizando métodos Z-Score e IQR, y se decidió mantenerlos inicialmente.
Estadísticas Descriptivas de Variables Numéricas
Las medidas de tendencia central y dispersión para las variables numéricas clave son las siguientes:

codigo:
```python
# Regenerar y mostrar la tabla de métricas descriptivas
columnas_numericas_desc = ['discounted_price', 'actual_price', 'discount_percentage', 'rating', 'rating_count']
metricas_numericas_desc = {}
for col in columnas_numericas_desc:
    if col in tabla_union.columns:
        mean_val = tabla_union[col].mean()
        median_val = tabla_union[col].median()
        mode_val = tabla_union[col].mode().tolist() if not tabla_union[col].mode().empty else None
        std_val = tabla_union[col].std()
        metricas_numericas_desc[col] = {
            'Media': mean_val,
            'Mediana': median_val,
            'Moda': mode_val,
            'Desviación Estándar': std_val
        }
    else:
        metricas_numericas_desc[col] = {'Media': None, 'Mediana': None, 'Moda': None, 'Desviación Estándar': None} # Handle missing columns

df_metricas_desc = pd.DataFrame(metricas_numericas_desc).T
display(df_metricas_desc)
```

Distribuciones de Variables Numéricas
Las distribuciones de las variables numéricas se visualizaron con histogramas y gráficos de densidad, revelando lo siguiente:
discounted_price y actual_price: Ambas presentan una distribución marcadamente sesgada a la derecha (positivamente), con la mayoría de los precios concentrados en valores bajos y una cola larga hacia precios más altos.
discount_percentage: Muestra una distribución aproximadamente simétrica, con una concentración alrededor de la moda del 50%.
rating: Tiene una distribución fuertemente sesgada a la izquierda (negativamente), con la gran mayoría de las calificaciones concentradas en los valores más altos (4-5).
rating_count: Exhibe una distribución altamente sesgada a la derecha (positivamente), con un pico masivo en valores bajos y una cola extremadamente larga hacia valores muy altos, indicando la presencia de productos con un número de reseñas excepcionalmente alto.
A continuación se muestran los histogramas correspondientes:

codigo:
```python
# Regenerar y mostrar los histogramas
columnas_numericas_hist = ['discounted_price', 'actual_price', 'discount_percentage', 'rating', 'rating_count']
sns.set_style("whitegrid")
for col in columnas_numericas_hist:
    if col in tabla_union.columns:
        plt.figure(figsize=(10, 6))
        sns.histplot(data=tabla_union, x=col, kde=True)
        plt.title(f'Distribución de {col}')
        plt.xlabel(col)
        plt.ylabel('Frecuencia')
        plt.show()
    else:
        print(f"⚠️ Advertencia: La columna '{col}' no se encontró para el histograma.")
```

Análisis de Correlación
La matriz de correlación de Pearson entre las variables numéricas es la siguiente:

codigo:
```python
# Regenerar y mostrar la matriz de correlación
columnas_numericas_corr = ['discounted_price', 'actual_price', 'discount_percentage', 'rating', 'rating_count']
columnas_existentes_corr = [col for col in columnas_numericas_corr if col in tabla_union.columns]
if len(columnas_existentes_corr) > 1:
    correlacion_pearson_report = tabla_union[columnas_existentes_corr].corr(method='pearson')
    display(correlacion_pearson_report)
else:
    print("⚠️ No hay suficientes columnas numéricas para la matriz de correlación en el informe.")
```

El mapa de calor de correlación visualiza estas relaciones:

codigo:
```python
# Regenerar y mostrar el mapa de calor de correlación
if 'correlacion_pearson_report' in globals() and isinstance(globals()['correlacion_pearson_report'], pd.DataFrame):
    plt.figure(figsize=(10, 8))
    sns.heatmap(correlacion_pearson_report, annot=True, cmap='coolwarm', fmt=".2f", linewidths=.5)
    plt.title('Mapa de Calor de la Matriz de Correlación de Pearson')
    plt.show()
else:
     print("⚠️ La matriz de correlación no se encontró para el mapa de calor en el informe.")
```

Los hallazgos clave de la correlación son: una alta correlación positiva entre discounted_price y actual_price (0.96), y correlaciones lineales débiles o muy débiles entre el resto de pares de variables, incluyendo las relaciones entre precios/descuento y rating/conteo de reseñas.
Análisis de Preguntas de Negocio
¿Los productos con mayores descuentos tienen mejores calificaciones?
Para responder a esto, analizamos el rating promedio por rangos de porcentaje de descuento definidos por cuartiles.

codigo:
```python
# Regenerar y mostrar la tabla de métricas por rango de descuento
if 'discount_range_metrics' in globals() and isinstance(globals()['discount_range_metrics'], pd.DataFrame):
    print("📊 Rating Promedio por Rango de Porcentaje de Descuento:")
    display(discount_range_metrics[['avg_discount_percentage', 'avg_rating']])
else:
    print("⚠️ La tabla 'discount_range_metrics' no se encontró para el informe.")
```

La visualización del rating promedio por rango de descuento es la siguiente:

codigo:
```python
# Regenerar y mostrar el gráfico de barras de rating promedio por rango de descuento
if 'discount_range_metrics' in globals() and isinstance(globals()['discount_range_metrics'], pd.DataFrame):
    plt.figure(figsize=(10, 6))
    sns.barplot(x=discount_range_metrics.index, y='avg_rating', data=discount_range_metrics, palette='viridis')
    plt.title('Rating Promedio por Rango de Porcentaje de Descuento')
    plt.xlabel('Rango de Porcentaje de Descuento')
    plt.ylabel('Rating Promedio')
    plt.ylim(0, 5)
    plt.xticks(rotation=0)
    plt.tight_layout()
    plt.show()
else:
    print("⚠️ La tabla 'discount_range_metrics' no se encontró para el gráfico en el informe.")
```

Respuesta: Basado en el análisis y el gráfico, los productos con mayores porcentajes de descuento no tienen mejores calificaciones promedio. Se observa una ligera tendencia a que los rangos con menores descuentos tengan ratings promedio marginalmente más altos.
¿Existe relación entre el precio de un producto y su rating promedio?
Analizamos esta relación mediante gráficos de dispersión.

codigo:
```python
# Regenerar y mostrar los scatter plots de precio vs rating
sns.set_style("whitegrid")
plt.figure(figsize=(10, 6))
sns.scatterplot(data=tabla_union, x='discounted_price', y='rating', alpha=0.6)
plt.title('Relación entre Precio con Descuento y Rating Promedio')
plt.xlabel('Precio con Descuento')
plt.ylabel('Rating Promedio')
plt.ylim(0, 5.1)
plt.grid(True)
plt.show()

plt.figure(figsize=(10, 6))
sns.scatterplot(data=tabla_union, x='actual_price', y='rating', alpha=0.6)
plt.title('Relación entre Precio Real y Rating Promedio')
plt.xlabel('Precio Real')
plt.ylabel('Rating Promedio')
plt.ylim(0, 5.1)
plt.grid(True)
plt.show()
```

Respuesta: Los gráficos de dispersión confirman la debilidad de la correlación lineal previamente calculada. No parece haber una relación lineal fuerte o clara entre el precio de un producto y su rating promedio en este conjunto de datos. Los ratings altos se observan en un amplio rango de precios.
¿Cuáles son los productos más populares y cómo se perciben?
Identificamos los productos más populares basándonos en el rating_count (número de reseñas).

codigo:
```python
# Regenerar y mostrar la tabla de los 10 productos más populares
productos_mas_populares_report = tabla_union.sort_values(by='rating_count', ascending=False).head(10)
columnas_mostrar_report = ['product_name', 'rating_count', 'rating']
columnas_existentes_report = [col for col in columnas_mostrar_report if col in productos_mas_populares_report.columns]
if columnas_existentes_report:
    print("📊 Los 10 productos más populares (basado en Rating Count):")
    display(productos_mas_populares_report[columnas_existentes_report])
else:
    print("⚠️ No se pudieron encontrar las columnas relevantes para mostrar los productos populares en el informe.")
```

La percepción de estos productos populares se visualiza a través de sus ratings promedio:

codigo:
```python
# Regenerar y mostrar el gráfico de barras de rating de los productos más populares
if 'productos_mas_populares_report' in globals() and isinstance(globals()['productos_mas_populares_report'], pd.DataFrame) and not productos_mas_populares_report.empty:
    plt.figure(figsize=(12, 7))
    productos_mas_populares_sorted_report = productos_mas_populares_report.sort_values(by='rating_count', ascending=False)
    sns.barplot(x='product_name', y='rating', data=productos_mas_populares_sorted_report, palette='viridis')
    plt.title('Rating Promedio de los 10 Productos Más Populares (por Conteo de Reseñas)')
    plt.xlabel('Producto')
    plt.ylabel('Rating Promedio')
    plt.ylim(0, 5)
    plt.xticks(rotation=90, ha='right')
    plt.tight_layout()
    plt.show()
else:
    print("⚠️ No se pudo generar el gráfico de productos populares para el informe.")
```

Respuesta: Los productos más populares (generalmente electrónicos y accesorios de uso masivo) tienen una percepción muy positiva, con la gran mayoría presentando ratings promedio altos (4.1 o superior), como se evidencia en la tabla y el gráfico.
Conclusiones
Este análisis exploratorio ha revelado varias características importantes del conjunto de datos:
Distribuciones Asimétricas: Las variables de precio y rating_count exhiben una fuerte asimetría positiva, mientras que rating está sesgado negativamente, reflejando la diversidad de precios, la alta popularidad concentrada en pocos productos y la tendencia general a ratings altos.
Ratings Generalmente Altos: La mayoría de los productos en el dataset reciben calificaciones promedio altas (superiores a 4.0).
Débil Relación Lineal con Precio y Descuento: No se encontró una relación lineal fuerte entre el precio/porcentaje de descuento y el rating promedio. Los productos con mayores descuentos no tienen mejores calificaciones en promedio; de hecho, se observó una ligera tendencia inversa.
Productos Populares Bien Percibidos: Los productos con el mayor número de reseñas (más populares) tienden a tener ratings promedio altos, sugiriendo una percepción generalmente positiva entre un gran número de usuarios.
Duplicados y Nulos: Se identificaron y manejaron nulos y diversos tipos de duplicados, aunque algunos (como productos con el mismo nombre pero diferente ID) se mantuvieron para análisis posteriores.
Aspectos y Hallazgos Más Relevantes para el Informe Final:
La distribución altamente sesgada de rating_count y su implicación en la representatividad del rating.
La tendencia general a ratings altos.
La falta de una relación lineal fuerte entre precio/descuento y rating.
La percepción positiva de los productos más populares.
La identificación y manejo de los principales problemas de calidad de datos (nulos, duplicados, formato numérico).
Recomendaciones
Basado en este análisis, se sugieren los siguientes próximos pasos:
Explorar Relaciones No Lineales: Investigar si existen relaciones no lineales entre precio/descuento y rating que no fueron capturadas por la correlación de Pearson.
Análisis por Subcategorías: Realizar análisis similares (métricas, distribuciones, relaciones) dentro de subcategorías de productos más específicas para identificar patrones que podrían estar ocultos en las categorías amplias.
Análisis de Texto: Si es relevante, realizar análisis de sentimiento o extracción de temas en los títulos y contenidos de las reseñas para comprender mejor las razones detrás de los ratings altos o bajos.
Considerar Transformaciones para Modelado: Si el objetivo es construir modelos predictivos, evaluar la necesidad de transformar variables altamente asimétricas como rating_count para cumplir con los supuestos de ciertos algoritmos.
Investigar Duplicados Restantes: Si es crucial para el análisis, profundizar en la causa de los productos con el mismo nombre pero diferente ID.
Este informe resume los hallazgos clave y proporciona una base sólida para análisis futuros y la toma de decisiones basada en datos.

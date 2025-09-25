# Proyecto3-PY-Amazon
"Análisis de datos de productos y reseñas de Amazon. Incluye limpieza, exploración, manejo de nulos y duplicados, análisis de outliers. Investiga la relación entre descuentos, precios y ratings. Identifica productos populares. Uso de Pandas, Matplotlib, Seaborn para insights clave en e-commerce."
# 1. Importar los datos
En esta sección cargamos los archivos CSV que contienen la información de productos y reseñas de Amazon.
Los datasets originales se llaman amazon - amazon_product.csv y amazon - amazon_review.csv.
Para evitar problemas con los nombres (espacios y guiones), los renombramos con nombres más simples.
# cargar_datos.py
# -------------------------------
# Script para cargar y preparar datasets de Amazon
# -------------------------------

# Importamos la librería necesaria
import pandas as pd  # "pd" es un alias, lo usaremos para llamar a pandas

# Subir archivos desde la computadora (solo en Google Colab)
from google.colab import files

uploaded = files.upload()  # Se abrirá un cuadro de diálogo para elegir los CSV

# Renombramos directamente al cargarlos
productos = pd.read_csv("amazon - amazon_product.csv")
resenas = pd.read_csv("amazon - amazon_review.csv")

# Guardamos una copia con nombres más simples para trabajar sin problemas
productos.to_csv("amazon_product.csv", index=False)
resenas.to_csv("amazon_review.csv", index=False)

# Ahora reabrimos usando los nombres limpios
productos = pd.read_csv("amazon_product.csv")
resenas = pd.read_csv("amazon_review.csv")

# Verificamos cuántas filas y columnas tiene cada dataset
print("Productos:", productos.shape)
print("Reseñas:", resenas.shape)

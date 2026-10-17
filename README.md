# Rasterizar.py import matplotlib.pyplot as plt
import numpy as np
import geopandas as gpd
import rasterio
from rasterio import features
from rasterio.plot import show
from rasterio.enums import MergeAlg

# --- PARTE 1: EJEMPLO MATPLOTLIB (RASTERIZACIÓN GRÁFICA) ---
d = np.arange(100).reshape(10, 10)  
x, y = np.meshgrid(np.arange(11), np.arange(11))

theta = 0.25 * np.pi
xx = x * np.cos(theta) - y * np.sin(theta)  
yy = x * np.sin(theta) + y * np.cos(theta)  

fig, ((ax1, ax2), (ax3, ax4)) = plt.subplots(2, 2, layout="constrained")

ax1.set_aspect(1)
ax1.pcolormesh(xx, yy, d)
ax1.set_title("No Rasterization")

ax2.set_aspect(1)
ax2.set_title("Rasterization")
ax2.pcolormesh(xx, yy, d, rasterized=True)

ax3.set_aspect(1)
ax3.pcolormesh(xx, yy, d)
ax3.text(0.5, 0.5, "Text", alpha=0.2, va="center", ha="center", size=50, transform=ax3.transAxes)
ax3.set_title("No Rasterization")

ax4.set_aspect(1)
m = ax4.pcolormesh(xx, yy, d, zorder=-10)
ax4.text(0.5, 0.5, "Text", alpha=0.2, va="center", ha="center", size=50, transform=ax4.transAxes)
ax4.set_rasterization_zorder(0)
ax4.set_title("Rasterization z$<-10$")

plt.savefig("test_rasterization.pdf", dpi=150)
plt.savefig("test_rasterization.eps", dpi=150)

# --- PARTE 2: PROCESAMIENTO SIG (RASTERIZACIÓN DE SHAPEFILE) ---
if not plt.rcParams["text.usetex"]:
    plt.savefig("test_rasterization.svg", dpi=150)
    
    try:
        # 1. Cargar archivos espaciales
        vector = gpd.read_file(r"../_static/e_vector_shapefiles/sf_bay_counties/sf_bay_counties.shp")
        raster = rasterio.open(r"../_static/e_raster/bay-area-wells_kde_sklearn.tif")
        
        # 2. Crear valores únicos para quemar en el raster (ID por fila)
        vector['id'] = range(0, len(vector))
        
        # Generar tuplas (geometría, valor_id)
        geom_value = ((geom, val) for geom, val in zip(vector.geometry, vector['id']))

        # 3. Rasterizar usando los metadatos de la imagen base
        rasterized = features.rasterize(
            geom_value,
            out_shape=raster.shape,
            transform=raster.transform,
            all_touched=True,          # Activa píxeles que intersecten parcialmente
            fill=-5,                   # Valor para el fondo (NoData)
            merge_alg=MergeAlg.replace,
            dtype='int16'              # String estándar aceptado por numpy/rasterio
        )

        # 4. Visualizar el resultado geoespacial
        fig, ax = plt.subplots(1, figsize=(10, 10))
        show(rasterized, ax=ax)
        plt.gca().invert_yaxis()
        
        # 5. Guardar el nuevo GeoTIFF en el disco
        with rasterio.open(
                "../temp/rasterized_vector.tif", "w",
                driver="GTiff",
                crs=raster.crs,
                transform=raster.transform,
                dtype=rasterio.int16,   # Consistente con el dtype anterior
                count=1,
                width=raster.width,
                height=raster.height) as dst:
            dst.write(rasterized, indexes=1)
            
        print("¡Rasterización espacial completada con éxito!")
        
    except FileNotFoundError:
        print("Nota: No se ejecutó la sección GIS porque las rutas a los archivos .shp o .tif no existen.")

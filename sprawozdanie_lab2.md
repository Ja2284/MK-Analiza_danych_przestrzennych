# LAB2 – Sprawozdanie z analizy danych przestrzennych: GeoPandas i Rasterio

**Zakres sprawozdania:** część GeoPandas (A1–A9) oraz część Rasterio (kroki 1–9).

## Cel i zakres ćwiczenia

Celem ćwiczenia było poznanie podstaw pracy z danymi przestrzennymi w Pythonie przy użyciu bibliotek **Shapely**, **GeoPandas**, **Pandas** i **Rasterio**. Część pierwsza obejmowała analizę danych wektorowych i pomiarów PM2.5, a część druga – przetwarzanie rastrów Sentinel-2, obliczenie wskaźnika NDVI, klasyfikację pokrycia oraz statystyki strefowe. W ramach zadania utworzono geometrie punktowe i poligonowe, wczytano oraz oczyszczono dane pomiarowe PM2.5, przekształcono tabelę do postaci GeoDataFrame, wykonano operacje na układach współrzędnych, przeanalizowano dzielnice Opola, wykonano złączenia przestrzenne i bufory wokół szkół, a na końcu zapisano wyniki do plików przestrzennych.

## 1. Przygotowanie środowiska

### 1.1. Struktura katalogów

W katalogu roboczym `lab_geo` przygotowano następującą strukturę:

```text
lab_geo/
├── dane/
│   ├── pm25_sensors.csv
│   ├── opole_districts.geojson
│   ├── opole_schools.geojson
│   ├── S2_L2A_2026-10-06_B04.tiff
│   └── S2_L2A_2026-10-06_B08.tiff
├── wyniki/
├── lab_geopandas.ipynb
└── lab_rasterio.ipynb
```

W katalogu `dane/` umieszczono pliki wejściowe udostępnione do ćwiczenia. Katalog `wyniki/` służy do zapisywania rezultatów analizy.

### 1.2. Utworzenie środowiska i instalacja bibliotek

W wierszu polecenia, w katalogu `lab_geo`, utworzono i aktywowano środowisko wirtualne:

```cmd
python -m venv .venv
.venv\Scripts\activate.bat
```

Następnie zainstalowano biblioteki wykorzystywane w ćwiczeniu:

```cmd
python -m pip install --upgrade pip
pip install geopandas pandas matplotlib mapclassify folium jupyter
```

W ćwiczeniu wykorzystano Python 3.13.16. Kod wykonywano w notatniku Jupyter `lab_geopandas.ipynb`.

---

## A1. Geometria i GeoDataFrame od zera

Na początku utworzono punkt reprezentujący Rynek w Opolu oraz przykładowy poligon. Następnie sprawdzono powierzchnię poligonu i to, czy punkt znajduje się w jego wnętrzu.

```python
# A1
from shapely.geometry import Point, Polygon

rynek = Point(17.9226, 50.6681)

square = Polygon([
    (17.92, 50.66),
    (17.93, 50.66),
    (17.93, 50.67),
    (17.92, 50.67)
])

print(rynek)
print(square.area)
print(square.contains(rynek))
```

Następnie utworzono GeoDataFrame zawierający obie geometrie:

```python
import geopandas as gpd

places = gpd.GeoDataFrame(
    {"name": ["Rynek", "Square"]},
    geometry=[rynek, square],
    crs="EPSG:4326"
)

places

print(type(places))
print(places.crs)
print(places.geom_type)
```

Sprawdzono typ obiektu, układ współrzędnych oraz typy geometrii. Utworzony GeoDataFrame zawierał geometrię `Point` i `Polygon`, a jego układem współrzędnych był `EPSG:4326`.

## A2. Wczytanie surowych danych

Wczytano plik CSV z pomiarami czujników PM2.5. Kolumnę `measured_at` odczytano jako datę i czas.

```python
# A2
import pandas as pd

raw = pd.read_csv("dane/pm25_sensors.csv", parse_dates=["measured_at"])

print(raw.shape)
raw.head(10)
```

Następnie sprawdzono podstawowe statystyki, braki danych oraz duplikaty:

```python
raw[["lat", "lon", "pm25"]].describe()
```

```python
raw.isna().sum()
```

```python
raw.duplicated().sum()
```

Dane wejściowe zawierały **145 wierszy i 5 kolumn**. Wykryto braki w kolumnach współrzędnych (`lat` – 3 braki, `lon` – 4 braki), 5 zduplikowanych rekordów oraz nieprawidłową wartość pomiaru PM2.5 równą `-999`. Zauważono także rekordy z zamienionymi miejscami wartościami szerokości i długości geograficznej.

## A3. Czyszczenie danych

Przed dalszą analizą utworzono kopię danych, usunięto duplikaty i rekordy bez współrzędnych, naprawiono zamienione współrzędne oraz odrzucono nieprawidłowe pomiary PM2.5.

```python
# A3

df = raw.copy()
report = {"input rows": len(df)}

df = df.drop_duplicates()
report["after drop_duplicates"] = len(df)

df = df.dropna(subset=["lat", "lon"])
report["after dropna"] = len(df)

swapped = df["lat"].between(14, 24) & df["lon"].between(49, 55)
df.loc[swapped, ["lat", "lon"]] = df.loc[swapped, ["lon", "lat"]].to_numpy()
report["swapped rows fixed"] = int(swapped.sum())

df = df[df["pm25"] > -999].reset_index(drop=True)
report["clean rows"] = len(df)

report
```

Kontrola kolejnych etapów czyszczenia wykazała:

| Etap | Liczba rekordów |
|---|---:|
| Dane początkowe | 145 |
| Po usunięciu duplikatów | 140 |
| Po usunięciu brakujących współrzędnych | 134 |
| Naprawione zamienione współrzędne | 3 |
| Po odrzuceniu nieprawidłowych pomiarów | 130 |

Do dalszej analizy wykorzystano 130 oczyszczonych rekordów.

## A4. Z tabeli do GeoDataFrame

Z oczyszczonej tabeli utworzono GeoDataFrame. Punkty wygenerowano z kolumn `lon` (X) i `lat` (Y), przyjmując układ współrzędnych WGS84.

```python
# A4

sensors = gpd.GeoDataFrame(
    df,
    geometry=gpd.points_from_xy(df["lon"], df["lat"]),
    crs="EPSG:4326"
)

sensors.head()
```

Sprawdzono typ geometrii, układ współrzędnych i zasięg danych:

```python
print(sensors.geom_type.unique())
print(sensors.crs)
print(sensors.total_bounds)
```

Dane przedstawiono również na interaktywnej mapie, kolorując punkty według wartości PM2.5:

```python
sensors.explore(
    column="pm25",
    cmap="Oranges",
    marker_kwds={"radius": 6}
)
```

Wynikiem był GeoDataFrame zawierający 130 geometrii typu `Point` w układzie `EPSG:4326`.

## A5. Układy współrzędnych

Najpierw obliczono odległość pomiędzy dwoma pierwszymi punktami w układzie geograficznym. Wynik wyrażony w stopniach nie jest odległością w metrach.

```python
# A5

a, b = sensors.geometry.iloc[0], sensors.geometry.iloc[1]
a.distance(b)
```

Następnie przekształcono punkty do układu metrycznego `EPSG:2180` i ponownie obliczono odległość:

```python
sensors_m = sensors.to_crs("EPSG:2180")

sensors_m.geometry.iloc[0].distance(sensors_m.geometry.iloc[1])

sensors_m.head()[["sensor_id", "geometry"]]
```

Porównano także odległość geodezyjną pomiędzy Gdańskiem i Zakopanem z odległościami obliczonymi w dwóch układach odwzorowanych:

```python
from pyproj import Geod

gdansk, zakopane = (18.6466, 54.3520), (19.9466, 49.2992)

pts = gpd.GeoSeries.from_xy(
    [gdansk[0], zakopane[0]],
    [gdansk[1], zakopane[1]],
    crs="EPSG:4326"
)

_, _, true_m = Geod(ellps="WGS84").inv(*gdansk, *zakopane)

print(f"geodesic: {true_m / 1000:.1f} km")

for crs in ["EPSG:3857", "EPSG:2180"]:
    p = pts.to_crs(crs)
    d = p.iloc[0].distance(p.iloc[1])
    print(f"{crs}: {d / 1000:.1f} km | error ({100 * (d / true_m - 1):+.2f}%)")
```

Wniosek: odległości i powierzchnie należy obliczać w odpowiednim układzie metrycznym, a nie bezpośrednio w geograficznym `EPSG:4326`. Do dalszych analiz lokalnych wykorzystano `EPSG:2180`.

## A6. Poligony: dzielnice Opola

Wczytano granice dzielnic Opola z pliku GeoJSON, a następnie przekształcono je do układu `EPSG:2180`, aby obliczyć powierzchnie w metrach kwadratowych i kilometrach kwadratowych.

```python
# A6

districts = gpd.read_file("dane/opole_districts.geojson")

print(len(districts), districts.crs)
districts.head()
```

```python
districts = districts.to_crs("EPSG:2180")

districts["area_km2"] = districts.area / 1e6

print(f"City area: {districts['area_km2'].sum():.1f} km²")

districts.sort_values("area_km2", ascending=False).head()
```

Wykonano mapę tematyczną powierzchni dzielnic:

```python
import matplotlib.pyplot as plt

ax = districts.plot(
    column="area_km2",
    cmap="Blues",
    legend=True,
    edgecolor="white",
    figsize=(7, 7)
)

ax.set_title("Opole districts by area (km²)")
ax.set_axis_off()
plt.show()
```

W zbiorze znajdowało się **29 dzielnic**. Łączna powierzchnia wynosiła około **148,8 km²**. W kontroli wyników największą dzielnicą były Czarnowąsy (około 14,1 km²), a najmniejszą Borki (około 0,66 km²).

## A7. Złączenie przestrzenne

Wykonano złączenie przestrzenne punktów czujników z poligonami dzielnic. Dla każdego punktu przypisano nazwę dzielnicy, w której się znajduje.

```python
# A7

joined = gpd.sjoin(
    sensors_m,
    districts[["name", "geometry"]],
    how="left",
    predicate="within",
)

print(len(joined), joined["name"].isna().sum())
joined[["sensor_id", "pm25", "name"]].head()
```

Obliczono średnią wartość PM2.5 i liczbę czujników w każdej dzielnicy:

```python
stats = (
    joined.groupby("name")
    .agg(
        pm25_mean=("pm25", "mean"),
        sensor_count=("pm25", "size")
    )
    .round(1)
)

summary = districts.merge(stats, on="name", how="left")
summary.sort_values("pm25_mean", ascending=False).head()
```

Liczbę czujników uzupełniono zerami dla dzielnic bez przypisanych rekordów, a następnie wybrano dzielnice z co najmniej trzema czujnikami:

```python
summary["sensor_count"] = summary["sensor_count"].fillna(0).astype(int)

reliable = summary[summary["sensor_count"] >= 3]

reliable.nlargest(
    3, "pm25_mean"
)[["name", "pm25_mean", "sensor_count"]]
```

Wykonano mapę choropleth średniego PM2.5:

```python
summary.explore(
    column="pm25_mean",
    cmap="Oranges",
    scheme="quantiles",
    k=5,
    legend_kwds={"caption": "mean PM2.5 (µg/m³)"},
    missing_kwds={"color": "lightgray", "label": "no sensors"}
)
```

Po złączeniu uzyskano 130 rekordów i nie stwierdzono punktów bez przypisanej dzielnicy. Trzy dzielnice miały mniej niż trzy czujniki; nie oznacza to automatycznie, że nie miały żadnego czujnika. Wśród najwyższych średnich PM2.5 wskazano Stare Miasto (29,6), Śródmieście (26,8) i Zaodrze (22,5).

## A8. Bufory wokół szkół

Wczytano lokalizacje szkół, przekształcono je do układu metrycznego `EPSG:2180` i utworzono bufory o promieniu 500 metrów.

```python
# A8

schools = gpd.read_file("dane/opole_schools.geojson").to_crs("EPSG:2180")

buffers = schools.copy()
buffers["geometry"] = schools.buffer(500)

buffers.head(3)
```

Wykonano złączenie przestrzenne, aby wskazać czujniki znajdujące się wewnątrz buforów szkół:

```python
near_schools = gpd.sjoin(
    sensors_m,
    buffers[["name", "geometry"]],
    predicate="within"
)

print("schools:", len(schools))
print("schools with a sensor within 500 m:", near_schools["name"].nunique())
print(f"mean PM2.5 near schools: {near_schools['pm25'].mean():.1f} µg/m³")
```

Obliczono również, jaki procent powierzchni miasta pokrywają połączone bufory:

```python
city = districts.union_all()

coverage = buffers.union_all().intersection(city).area / city.area

print(f"coverage: {coverage:.1%} of the city")
```

Przygotowano mapę dzielnic, buforów i punktów pomiarowych:

```python
m = districts.explore(
    color="gray",
    style_kwds={"fillOpacity": 0.05},
    name="districts"
)

buffers.explore(
    m=m,
    color="orange",
    name="500 m buffers"
)

sensors_m.explore(
    m=m,
    color="black",
    marker_kwds={"radius": 3},
    name="sensors"
)

m
```

Dodatkowo sprawdzono większy promień bufora – 1500 metrów:

```python
buffers_1500 = schools.copy()
buffers_1500["geometry"] = schools.buffer(1500)

near_schools_1500 = gpd.sjoin(
    sensors_m,
    buffers_1500[["name", "geometry"]],
    predicate="within"
)

print(near_schools_1500["name"].nunique())
```

W kontroli wyników dla buforów 500 m oczekiwano 19 szkół, 11 szkół z co najmniej jednym czujnikiem w promieniu 500 m, średniej PM2.5 w pobliżu szkół około 22,3 µg/m³ oraz pokrycia około 9,5% powierzchni miasta.

## A9. Zapis wyników

Na końcu utworzono katalog `wyniki/` i zapisano wyniki do plików GeoPackage oraz GeoJSON.

```python
# A9

from pathlib import Path

out = Path("wyniki")
out.mkdir(exist_ok=True)

summary.to_file(out / "pm25.gpkg", layer="districts", driver="GPKG")
sensors_m.to_file(out / "pm25.gpkg", layer="sensors", driver="GPKG")

summary.to_crs("EPSG:4326").to_file(
    out / "districts.geojson",
    driver="GeoJSON"
)

gpd.list_layers(out / "pm25.gpkg")
```

Sprawdzono poprawność zapisu warstwy dzielnic:

```python
check = gpd.read_file(out / "pm25.gpkg", layer="districts")

print(len(check), check.crs)
```

Wynikiem kontroli było **29 obiektów** w układzie **EPSG:2180**. Plik `pm25.gpkg` zawiera dwie warstwy: `districts` oraz `sensors`. Dodatkowo zapisano `districts.geojson` w układzie `EPSG:4326`.

## Wyniki końcowe i pliki

W wyniku ćwiczenia przygotowano:

- `wyniki/pm25.gpkg` – GeoPackage z warstwami `districts` i `sensors`;
- `wyniki/districts.geojson` – granice dzielnic z wyliczonymi atrybutami, zapisane w `EPSG:4326`;
- `lab_geopandas.ipynb` – notatnik Jupyter zawierający kod i wyniki kolejnych etapów.

Pliki wynikowe części GeoPandas to `wyniki/pm25.gpkg` (warstwy `districts` i `sensors`) oraz `wyniki/districts.geojson`.

## Podsumowanie części GeoPandas

W części GeoPandas przeprowadzono proces analizy danych przestrzennych: utworzono geometrie Shapely, oczyszczono dane pomiarowe PM2.5, przekształcono je do GeoDataFrame, wykonano operacje na układach współrzędnych, przeanalizowano dzielnice Opola, przeprowadzono złączenie przestrzenne i utworzono bufory wokół szkół. Zastosowano `EPSG:4326` do danych geograficznych oraz `EPSG:2180` do obliczeń odległości i powierzchni. Wyniki zapisano do GeoPackage i GeoJSON.

---

## Część II – Rasterio: dane rastrowe i NDVI

### Cel części Rasterio

Celem ćwiczenia było odczytanie dwóch pasm satelitarnych Sentinel-2 L2A dla Opola i okolic, sprawdzenie ich metadanych, przeliczenie współrzędnych pikseli, sprowadzenie pasm do wspólnej siatki w układzie metrycznym, obliczenie wskaźnika NDVI, sklasyfikowanie pikseli, wykonanie statystyk strefowych dla dzielnic Opola oraz zapis wyniku do pliku GeoTIFF.

Wykorzystano dwa pliki pobrane z Copernicus Browser:

- `S2_L2A_2026-10-06_B04.tiff` – pasmo Red (B04),
- `S2_L2A_2026-10-06_B08.tiff` – pasmo NIR (B08).

Analizowana scena pochodzi z 6 października 2026 r. i obejmuje Opole oraz okolice. Oba rastry są jednopasmowe i zawierają wartości typu `float32`.

### Środowisko i dane wejściowe

Ćwiczenie wykonano w JupyterLab, w istniejącym środowisku wirtualnym Pythona. Notatnik zapisano jako `lab_rasterio.ipynb` w katalogu projektu `lab_geo`.

Wymagane biblioteki:

```bash
pip install rasterio numpy pandas matplotlib geopandas shapely
```

Pliki wejściowe umieszczono w katalogu `dane/`, a wynikowy raster zapisano w katalogu `wyniki/`.

Przykładowa struktura projektu:

```text
lab_geo/
├── dane/
│   ├── S2_L2A_2026-10-06_B04.tiff
│   ├── S2_L2A_2026-10-06_B08.tiff
│   └── opole_districts.geojson
├── wyniki/
│   └── ndvi.tif
├── lab_geopandas.ipynb
└── lab_rasterio.ipynb
```

## R1. Raster i GeoTIFF – otwarcie plików i metadane

Do odczytu metadanych wykorzystano `rasterio.open()`. Pliki otwierano wewnątrz bloku `with`, dzięki czemu po zakończeniu odczytu zostały prawidłowo zamknięte.

```python
# R1
import rasterio

files = {
    "B04": "dane/S2_L2A_2026-10-06_B04.tiff",
    "B08": "dane/S2_L2A_2026-10-06_B08.tiff"
}

for band, path in files.items():
    print(f"\n--- {band} ---")
    with rasterio.open(path) as src:
        print(src.profile)
        print(src.crs)
        print(src.bounds)
        print(src.res)
        print(src.nodata)
```

### Uzyskane metadane

| Parametr | B04 | B08 |
|---|---:|---:|
| Format | GTiff | GTiff |
| Szerokość | 1037 px | 1037 px |
| Wysokość | 697 px | 697 px |
| Liczba pasm | 1 | 1 |
| Typ danych | float32 | float32 |
| CRS | EPSG:4326 | EPSG:4326 |
| NoData | None | None |

Oba rastry mają ten sam zasięg przestrzenny. W przybliżeniu granice sceny wynoszą: długość geograficzna 17,87647–18,06290° E oraz szerokość geograficzna 50,62464–50,70407° N. Rozdzielczość źródłowa wynosi około 0,00017977° w kierunku X i 0,00011396° w kierunku Y.

**Wniosek:** oba pliki są jednopasmowymi rastrami GeoTIFF w układzie WGS 84 (EPSG:4326), mają jednakowe wymiary i zasięg. W metadanych nie określono wartości NoData (`None`).

![Wynik odczytu metadanych obu pasm](images/r1_metadane.png)

## R2. Piksel – współrzędne

Transformacja afiniczna rastra pozwala przeliczać indeksy piksela (wiersz, kolumna) na współrzędne geograficzne jego środka. Wykorzystano `src.xy()`. Następnie sprawdzono przeliczenie odwrotne za pomocą `src.index()` oraz odczytano wartość pojedynczego piksela przez `Window`.

```python
# R2
import rasterio
from rasterio.windows import Window

path = "dane/S2_L2A_2026-10-06_B04.tiff"

with rasterio.open(path) as src:
    row = src.height // 2
    col = src.width // 2

    x, y = src.xy(row, col)
    print("row, col:", row, col)
    print("x, y:", x, y)

    row_back, col_back = src.index(x, y)
    print("Odtworzone row, col:", row_back, col_back)

    pixel = src.read(1, window=Window(col, row, 1, 1))
    print("Wartość piksela B04:", pixel[0, 0])
```

### Wynik

| Parametr | Wartość |
|---|---:|
| Wiersz (row) | 348 |
| Kolumna (col) | 518 |
| X (długość geograficzna) | 17,9696846° |
| Y (szerokość geograficzna) | 50,6643566° |
| Wartość piksela B04 | 0,0403 |
| Indeksy po przeliczeniu odwrotnym | row 348, col 518 |

Przeliczenie zwrotne zwróciło te same indeksy piksela. Potwierdza to poprawność użytej transformacji i odczytu pojedynczego piksela.

![Wynik przeliczenia indeksu piksela na współrzędne](images/r2_pixel_wspolrzedne.png)

## R4. Odczyt pasm i statystyki

Odczytano pierwsze pasmo z każdego pliku przy użyciu `masked=True`. Dla obu pasm obliczono minimum, maksimum, średnią, medianę, odchylenie standardowe oraz liczbę pikseli zamaskowanych jako NoData. Przygotowano również podgląd obu pasm w skali szarości i wspólny histogram wartości odbicia.

```python
# R4
import rasterio
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

files = {
    "Red (B04)": "dane/S2_L2A_2026-10-06_B04.tiff",
    "NIR (B08)": "dane/S2_L2A_2026-10-06_B08.tiff"
}

bands = {}
stats = {}

for name, path in files.items():
    with rasterio.open(path) as src:
        band = src.read(1, masked=True).astype("float32")
        bands[name] = band
        values = band.compressed()
        values = values[np.isfinite(values)]
        stats[name] = {
            "min": values.min(),
            "max": values.max(),
            "średnia": values.mean(),
            "mediana": np.median(values),
            "odch. std.": values.std(),
            "piksele nodata": np.ma.getmaskarray(band).sum()
        }

display(pd.DataFrame(stats))

fig, axes = plt.subplots(1, 2, figsize=(14, 6))
for ax, (name, path) in zip(axes, files.items()):
    with rasterio.open(path) as src:
        image = ax.imshow(bands[name], cmap="gray")
        ax.set_title(name)
        fig.colorbar(image, ax=ax, shrink=0.85)

fig.tight_layout()
plt.show()

fig, ax = plt.subplots(figsize=(12, 4))
for name, band in bands.items():
    values = band.compressed()
    values = values[np.isfinite(values)]
    ax.hist(values, bins=120, alpha=0.6, label=name)
ax.set_title("Histogram odbicia – B04 i B08")
ax.set_xlabel("odbicie")
ax.set_ylabel("liczba pikseli")
ax.legend()
plt.show()
```

### Uzyskane statystyki

| Statystyka | Red (B04) | NIR (B08) |
|---|---:|---:|
| Minimum | 0,000000 | 0,002900 |
| Maksimum | 1,622100 | 1,560500 |
| Średnia | 0,060958 | 0,214585 |
| Mediana | 0,050300 | 0,209400 |
| Odchylenie standardowe | 0,045236 | 0,081175 |
| Piksele NoData | 0 | 0 |

Wartości B08 są przeciętnie wyższe niż wartości B04. Histogram pokazuje odmienny rozkład odbicia w paśmie czerwonym i bliskiej podczerwieni. W danych nie wykryto pikseli zamaskowanych jako NoData. Ponieważ źródłowe pliki mają `nodata=None`, wartości zerowe nie były automatycznie traktowane jako NoData.

![Histogram odbicia pasm B04 i B08](images/r4_histogram.png)

## R5. Wspólna siatka w układzie metrycznym

Obliczenie NDVI wymaga, aby oba rastry miały zgodne wymiary, transformację i układ współrzędnych. Sprawdzono zgodność rastrów źródłowych, a następnie oba pasma przekształcono do układu metrycznego EPSG:2180 z rozdzielczością 10 m. Zastosowano interpolację biliniową (`Resampling.bilinear`).

```python
# R5
import numpy as np
import rasterio
from rasterio.warp import calculate_default_transform, reproject, Resampling

with rasterio.open("dane/S2_L2A_2026-10-06_B04.tiff") as red_src, \
     rasterio.open("dane/S2_L2A_2026-10-06_B08.tiff") as nir_src:

    same_grid = (
        red_src.crs == nir_src.crs
        and red_src.transform == nir_src.transform
        and red_src.shape == nir_src.shape
    )
    print("Czy rastry mają tę samą siatkę:", same_grid)

    dst_crs = "EPSG:2180"
    dst_transform, width, height = calculate_default_transform(
        red_src.crs, dst_crs,
        red_src.width, red_src.height,
        *red_src.bounds,
        resolution=10
    )

    def warp(src):
        out = np.full((height, width), np.nan, dtype="float32")
        reproject(
            source=rasterio.band(src, 1),
            destination=out,
            src_transform=src.transform,
            src_crs=src.crs,
            dst_transform=dst_transform,
            dst_crs=dst_crs,
            src_nodata=src.nodata,
            dst_nodata=np.nan,
            resampling=Resampling.bilinear
        )
        return np.ma.masked_invalid(out)

    red, nir = warp(red_src), warp(nir_src)

print("CRS wynikowy:", dst_crs)
print("Wymiary:", width, "x", height)
print("Rozdzielczość:", dst_transform.a, abs(dst_transform.e))
print("Kształt B04:", red.shape)
print("Kształt B08:", nir.shape)
print("Czy pasma mają identyczną siatkę:", red.shape == nir.shape)
```

### Wynik reprojekcji

- Rastry źródłowe miały tę samą siatkę: `True`.
- Docelowy CRS: `EPSG:2180`.
- Rozdzielczość: 10 × 10 m.
- Wymiary: 1329 × 902 px.
- Kształt tablic obu pasm: `(902, 1329)`.
- Oba pasma po reprojekcji mają identyczny kształt i siatkę.

## R6. Obliczenie NDVI

Wskaźnik NDVI obliczono ze wzoru:

\[
NDVI = \frac{NIR-Red}{NIR+Red} = \frac{B08-B04}{B08+B04}
\]

W obliczeniach wykorzystano pasma `nir` i `red` po reprojekcji. Niepoprawne wyniki, np. powstałe przez dzielenie przez zero, zamaskowano.

```python
# R6
import numpy as np
import matplotlib.pyplot as plt
from rasterio.transform import array_bounds

with np.errstate(divide="ignore", invalid="ignore"):
    ndvi = (nir - red) / (nir + red)

ndvi = np.ma.masked_invalid(ndvi)
valid = ndvi.compressed()
valid = valid[np.isfinite(valid)]

print("Średnie NDVI:", round(float(valid.mean()), 3))
print("Mediana NDVI:", round(float(np.median(valid)), 3))
print("Minimalne NDVI:", round(float(valid.min()), 3))
print("Maksymalne NDVI:", round(float(valid.max()), 3))
print("Liczba poprawnych pikseli:", len(valid))
```

### Wyniki NDVI

| Parametr | Wynik |
|---|---:|
| Średnie NDVI | 0,545 |
| Mediana NDVI | 0,616 |
| Minimum | -0,599 |
| Maksimum | 0,931 |
| Liczba poprawnych pikseli | 1 163 275 |

Najwyższa wartość NDVI wystąpiła w pikselu `(124, 884)`, mediana była najbliższa pikselowi `(425, 344)`, a najniższa wartość wystąpiła w pikselu `(882, 866)`.

Mapę NDVI i histogram przygotowano następująco:

```python
# R6 – mapa NDVI i histogram
from rasterio.transform import array_bounds

west, south, east, north = array_bounds(
    ndvi.shape[0], ndvi.shape[1], dst_transform
)

fig, axes = plt.subplots(
    1, 2, figsize=(15, 7), gridspec_kw={"width_ratios": [1.4, 1]}
)

im = axes[0].imshow(
    ndvi, cmap="RdYlGn", vmin=-1, vmax=1,
    extent=[west, east, south, north]
)
axes[0].set_title("NDVI")
axes[0].set_xlabel("X [m]")
axes[0].set_ylabel("Y [m]")
fig.colorbar(im, ax=axes[0], label="NDVI")

counts, bins, patches = axes[1].hist(valid, bins=100, range=(-1, 1))
for patch, left, right in zip(patches, bins[:-1], bins[1:]):
    patch.set_facecolor(plt.cm.RdYlGn((left + right) / 4 + 0.5))

axes[1].axvline(0, color="black", linestyle="--", label="NDVI = 0")
axes[1].set_title("Rozkład wartości NDVI")
axes[1].set_xlabel("NDVI")
axes[1].set_ylabel("Liczba pikseli")
axes[1].legend()

plt.tight_layout()
plt.show()
```

Na mapie wyższe wartości NDVI przedstawiono odcieniami zieleni, a niższe odcieniami żółci, pomarańczu i czerwieni. Histogram pokazuje rozkład wartości wskaźnika w całej scenie.

![Mapa NDVI i histogram – wynik R6](images/r6_ndvi_mapa_histogram.png)

## R7. Klasyfikacja i powierzchnie

Piksele podzielono na cztery klasy według progów NDVI:

| Klasa | Zakres NDVI |
|---|---|
| Woda | poniżej 0 |
| Zabudowa / goła gleba | 0–0,2 |
| Rzadka roślinność | 0,2–0,5 |
| Gęsta roślinność | powyżej 0,5 |

Do klasyfikacji użyto progów `0.0`, `0.2` i `0.5`. Ponieważ raster wynikowy jest w EPSG:2180 i ma rozdzielczość 10 m, powierzchnia jednego piksela wynosi 100 m². Powierzchnię w hektarach obliczono przez pomnożenie liczby pikseli przez 100 m² i podzielenie przez 10 000.

```python
# R7
classes = np.digitize(
    ndvi.filled(np.nan),
    bins=[0.0, 0.2, 0.5]
)
classes = np.ma.masked_where(
    np.ma.getmaskarray(ndvi), classes
)

pixel_area_m2 = abs(dst_transform.a * dst_transform.e)
pixel_counts = np.bincount(
    classes.compressed().astype(int),
    minlength=4
)
area_ha = pixel_counts * pixel_area_m2 / 10_000
shares = pixel_counts / pixel_counts.sum() * 100

class_names = [
    "woda",
    "zabudowa / goła gleba",
    "rzadka roślinność",
    "gęsta roślinność"
]

results = pd.DataFrame({
    "klasa": class_names,
    "piksele": pixel_counts,
    "powierzchnia [ha]": area_ha.round(1),
    "udział [%]": shares.round(1)
})

display(results)
print(f"Powierzchnia piksela: {pixel_area_m2:.1f} m²")
print(f"Łączna powierzchnia: {area_ha.sum():.2f} ha")
```

### Wyniki klasyfikacji

| Klasa | Liczba pikseli | Powierzchnia | Udział |
|---|---:|---:|---:|
| Woda | 12 961 | 129,6 ha | 1,1% |
| Zabudowa / goła gleba | 142 533 | 1 425,3 ha | 12,3% |
| Rzadka roślinność | 303 169 | 3 031,7 ha | 26,1% |
| Gęsta roślinność | 704 612 | 7 046,1 ha | 60,6% |
| **Łącznie** | **1 163 275** | **11 632,7 ha** | **100,0%** |

Największy udział wśród sklasyfikowanych pikseli ma klasa gęstej roślinności – 60,6%. Wynik dotyczy wyłącznie obszaru sceny satelitarnej, a nie całej powierzchni administracyjnej Opola.

Mapę NDVI wyświetlono z legendą i w układzie współrzędnych metrycznych:

```python
# R7 – mapa klasyfikacji i powierzchnie
fig, ax = plt.subplots(figsize=(10, 8))

west, south, east, north = array_bounds(
    ndvi.shape[0], ndvi.shape[1], dst_transform
)

img = ax.imshow(
    ndvi, cmap="RdYlGn", vmin=-1, vmax=1,
    extent=[west, east, south, north]
)
ax.set_title("NDVI – klasyfikacja roślinności")
ax.set_xlabel("X [m]")
ax.set_ylabel("Y [m]")
fig.colorbar(img, ax=ax, label="NDVI")
plt.tight_layout()
plt.show()

display(results)
```


![Tabela klasyfikacji i powierzchni – R7](images/r7_tabela_klasyfikacji.png)

![Mapa klasyfikacji NDVI – R7](images/r7_mapa_klasyfikacji.png)

## R8. Statystyki strefowe z GeoPandas

W tym kroku połączono raster NDVI z poligonami dzielnic Opola z pliku `opole_districts.geojson`. Warstwę wektorową przekształcono do EPSG:2180. Dla każdej dzielnicy wycięto część rastra leżącą wewnątrz jej geometrii i obliczono pokrycie sceną oraz statystyki NDVI.

Zgodnie z założeniem ćwiczenia średnią NDVI interpretowano tylko dla dzielnic, których pokrycie sceną wynosi co najmniej 50%. Dzielnice z pokryciem poniżej 50% oznaczono kreskowaniem na mapie i pozostawiono bez obliczonej średniej.

Poniżej przedstawiono kompletną komórkę używaną do obliczeń, przygotowania tabeli i wykonania mapy. Pokrycie jest szacowane na podstawie liczby poprawnych pikseli przypisanych do dzielnicy i powierzchni piksela.

```python
# R8 – Statystyki strefowe z GeoPandas

import geopandas as gpd
import rasterio
import rasterio.mask
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

from rasterio.io import MemoryFile
from rasterio.transform import array_bounds
from shapely.geometry import box, mapping
from matplotlib.patches import Patch

raster_crs = "EPSG:2180"
zones = gpd.read_file("dane/opole_districts.geojson").to_crs(raster_crs)

pixel_area = abs(
    dst_transform.a * dst_transform.e
    - dst_transform.b * dst_transform.d
)

west, south, east, north = array_bounds(
    ndvi.shape[0], ndvi.shape[1], dst_transform
)
scene = box(west, south, east, north)

zones["coverage_pct"] = 0.0
zones["ndvi_mean"] = np.nan
zones["ndvi_min"] = np.nan
zones["ndvi_max"] = np.nan

profile = {
    "driver": "GTiff",
    "height": ndvi.shape[0],
    "width": ndvi.shape[1],
    "count": 1,
    "dtype": "float32",
    "crs": raster_crs,
    "transform": dst_transform,
    "nodata": np.nan
}

with MemoryFile() as memfile:
    with memfile.open(**profile) as dst:
        dst.write(np.ma.filled(ndvi, np.nan).astype("float32"), 1)

    with memfile.open() as src:
        for i, zone in zones.iterrows():
            geom = zone.geometry
            if geom is None or geom.is_empty or geom.area == 0:
                continue

            overlap = geom.intersection(scene)
            if overlap.is_empty or overlap.area == 0:
                continue

            clipped, _ = rasterio.mask.mask(
                src, [mapping(overlap)], crop=True, filled=False
            )
            values = clipped[0].compressed()
            values = values[np.isfinite(values)]

            if values.size == 0:
                continue

            coverage = values.size * pixel_area / geom.area * 100
            zones.loc[i, "coverage_pct"] = min(coverage, 100.0)

            if coverage >= 50:
                zones.loc[i, "ndvi_mean"] = values.mean()
                zones.loc[i, "ndvi_min"] = values.min()
                zones.loc[i, "ndvi_max"] = values.max()

low_coverage = zones[zones["coverage_pct"] < 50]
valid_zones = zones[
    (zones["coverage_pct"] >= 50) & zones["ndvi_mean"].notna()
]

display(
    zones[["name", "coverage_pct", "ndvi_mean", "ndvi_min", "ndvi_max"]]
    .sort_values("ndvi_mean", ascending=False, na_position="last")
    .round(3)
)

print("Liczba dzielnic:", len(zones))
print("Dzielnice z pokryciem poniżej 50%:", len(low_coverage))
print("Dzielnice z obliczonym NDVI:", len(valid_zones))

display(
    low_coverage[["name", "coverage_pct"]]
    .sort_values("coverage_pct").round(3)
)

fig, ax = plt.subplots(figsize=(13, 10))
im = ax.imshow(
    ndvi, cmap="RdYlGn", vmin=-1, vmax=1,
    extent=[west, east, south, north]
)
zones.boundary.plot(ax=ax, color="black", linewidth=0.7)

if len(low_coverage) > 0:
    low_coverage.plot(
        ax=ax, facecolor="none", edgecolor="black",
        hatch="///", linewidth=0.7
    )

for _, zone in valid_zones.iterrows():
    point = zone.geometry.representative_point()
    ax.annotate(
        f"{zone['name']}\n{zone['ndvi_mean']:.2f}",
        (point.x, point.y), fontsize=6, ha="center"
    )

fig.colorbar(im, ax=ax, label="NDVI")
ax.legend(handles=[Patch(
    facecolor="white", edgecolor="black", hatch="///",
    label="Pokrycie sceną < 50%"
)], loc="upper right")
ax.set_title("NDVI + średnie w poligonach")
ax.set_xlabel("X [m]")
ax.set_ylabel("Y [m]")
plt.tight_layout()
plt.show()
```

### Wynik kontroli

- Liczba analizowanych dzielnic: 29.
- Dzielnice z pokryciem poniżej 50%: 13.
- Dzielnice z obliczoną średnią NDVI: 16.

W wykonaniu uzyskano **13 dzielnic z pokryciem poniżej 50%**, podczas gdy w podglądzie instrukcji podano 12. Wyniku nie zmieniano ręcznie w celu dopasowania go do przykładu; zachowano rezultat otrzymany z własnych danych i zastosowanego sposobu obliczania pokrycia.

Dzielnice z pokryciem poniżej 50% w uzyskanym zestawieniu: Bierkowice, Borki, Brzezie, Chmielowice, Czarnowąsy, Grotowice, Krzanowice, Malina, Sławice, Wrzoski, Wróblin, Świerkle i Żerkowice. Dla tych 13 dzielnic średnia NDVI pozostała nieobliczona (`NaN`).

Najwyższe średnie NDVI wśród dzielnic z pokryciem co najmniej 50%:

| Dzielnica | Pokrycie sceną | Średnie NDVI |
|---|---:|---:|
| Grudzice | 97,013% | 0,714 |
| Nadodrze | 99,988% | 0,660 |
| Zaodrze | 86,020% | 0,526 |
| Półwieś | 70,471% | 0,498 |
| Kolonia Gosławicka | ok. 100% | 0,491 |

Wyników dla dzielnic o pokryciu poniżej 50% nie należy traktować jako reprezentatywnych dla całej dzielnicy. Przykładowo, niewielki fragment poligonu może mieć wysokie NDVI, mimo że pozostała część dzielnicy nie znajduje się w zasięgu sceny.

![Tabela statystyk dzielnic – R8](images/r8_tabela_statystyki.png)

![Mapa NDVI z granicami dzielnic – R8](images/r8_mapa_dzielnic.png)

![Kontrola liczby dzielnic i progów pokrycia – R8](images/r8_kontrola_pokrycia.png)

## R9. Zapis wyniku do GeoTIFF

Wynik NDVI zapisano do pliku `wyniki/ndvi.tif`. Profil źródłowego rastra B04 wykorzystano jako punkt wyjścia, a następnie zaktualizowano CRS, transformację, wymiary, typ danych, liczbę pasm, wartość NoData i kompresję. Do pliku zapisano jedną warstwę NDVI typu `float32`.

```python
# R9
from pathlib import Path
import rasterio
import numpy as np

out_dir = Path("wyniki")
out_dir.mkdir(exist_ok=True)
output_path = out_dir / "ndvi.tif"

with rasterio.open(
    "dane/S2_L2A_2026-10-06_B04.tiff"
) as src:
    profile = src.profile.copy()

profile.update(
    crs=dst_crs,
    transform=dst_transform,
    width=ndvi.shape[1],
    height=ndvi.shape[0],
    dtype="float32",
    count=1,
    nodata=np.nan,
    compress="deflate"
)

with rasterio.open(output_path, "w", **profile) as dst:
    dst.write(ndvi.filled(np.nan).astype("float32"), 1)
    dst.set_band_description(1, "NDVI")

print("Zapisano plik:", output_path)

with rasterio.open(output_path) as src:
    print(src.profile)
```

### Parametry zapisanego pliku

| Parametr | Wynik |
|---|---|
| Nazwa | `ndvi.tif` |
| Format | GTiff |
| Typ danych | float32 |
| NoData | NaN |
| Szerokość | 1329 px |
| Wysokość | 902 px |
| Liczba pasm | 1 |
| CRS | EPSG:2180 |
| Rozdzielczość | 10 × 10 m |
| Kompresja | DEFLATE |

Na końcu plik `wyniki/ndvi.tif` otwarto w QGIS. Raster został poprawnie wczytany i wyświetlony. Pozostawiono domyślną prezentację w skali szarości.

![Kontrola metadanych zapisanego GeoTIFF – R9](images/r9_metadane_geotiff.png)

![Plik NDVI otwarty w QGIS](images/r9_qgis.png)

## 10. Podsumowanie

W ćwiczeniu wykorzystano Rasterio do odczytu rastrów GeoTIFF, sprawdzenia metadanych, przeliczania współrzędnych pikseli oraz reprojekcji dwóch pasm Sentinel-2 do wspólnej siatki w EPSG:2180. Na podstawie pasm B04 (Red) i B08 (NIR) obliczono NDVI, przygotowano mapę i histogram, sklasyfikowano piksele według progów NDVI oraz policzono powierzchnie klas.

Wykonano również statystyki strefowe dla 29 dzielnic Opola. Średnie NDVI obliczono dla 16 dzielnic, których pokrycie sceną wyniosło co najmniej 50%. Dla 13 dzielnic pokrycie było mniejsze niż 50%, dlatego nie wyznaczano dla nich reprezentatywnej średniej.

Końcowy raster NDVI zapisano jako `wyniki/ndvi.tif` w układzie EPSG:2180, z rozdzielczością 10 m i typem danych `float32`. Poprawność zapisu potwierdzono przez ponowne odczytanie metadanych oraz otwarcie pliku w QGIS.

---

# Podsumowanie całego LAB2

W ramach LAB2 wykonano dwie części ćwiczenia:

1. **GeoPandas (A1–A9):** przygotowano i oczyszczono pomiary PM2.5, utworzono geometrie punktowe, przeliczono układy współrzędnych, wykonano analizy z użyciem dzielnic i szkół Opola oraz zapisano dane wektorowe.
2. **Rasterio (kroki 1–9):** odczytano pasma Sentinel-2 B04 i B08, sprawdzono metadane i współrzędne pikseli, wykonano statystyki pasm, reprojekcję do EPSG:2180, obliczono NDVI, sklasyfikowano piksele, obliczono statystyki strefowe i zapisano wynik do GeoTIFF.

## Pliki wynikowe

- `wyniki/pm25.gpkg` – GeoPackage z warstwami dzielnic i czujników.
- `wyniki/districts.geojson` – warstwa dzielnic z atrybutami analitycznymi.
- `wyniki/ndvi.tif` – wynikowy raster NDVI w EPSG:2180.
- `lab_geopandas.ipynb` – notatnik części GeoPandas.
- `lab_rasterio.ipynb` – notatnik części Rasterio.

## Końcowa struktura katalogu projektu

```text
lab_geo/
├── dane/
│   ├── pm25_sensors.csv
│   ├── opole_districts.geojson
│   ├── opole_schools.geojson
│   ├── S2_L2A_2026-10-06_B04.tiff
│   └── S2_L2A_2026-10-06_B08.tiff
├── wyniki/
│   ├── pm25.gpkg
│   ├── districts.geojson
│   └── ndvi.tif
├── lab_geopandas.ipynb
└── lab_rasterio.ipynb
```

**Uwaga:** w części Rasterio rzeczywisty wynik kontroli R8 to 13 dzielnic z pokryciem poniżej 50% (16 dzielnic z obliczoną średnią NDVI). W instrukcji pokazano 12. Rozbieżność została jawnie odnotowana, bez sztucznego dostosowywania wyniku.

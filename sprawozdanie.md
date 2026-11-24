# LAB2 – Analiza danych przestrzennych w Pythonie z GeoPandas

## Cel ćwiczenia

Celem ćwiczenia było poznanie podstaw pracy z danymi przestrzennymi w Pythonie przy użyciu bibliotek **Shapely**, **GeoPandas** i **Pandas**. W ramach zadania utworzono geometrie punktowe i poligonowe, wczytano oraz oczyszczono dane pomiarowe PM2.5, przekształcono tabelę do postaci GeoDataFrame, wykonano operacje na układach współrzędnych, przeanalizowano dzielnice Opola, wykonano złączenia przestrzenne i bufory wokół szkół, a na końcu zapisano wyniki do plików przestrzennych.

## 1. Przygotowanie środowiska

### 1.1. Struktura katalogów

W katalogu roboczym `lab_geo` przygotowano następującą strukturę:

```text
lab_geo/
├── dane/
│   ├── pm25_sensors.csv
│   ├── opole_districts.geojson
│   └── opole_schools.geojson
├── wyniki/
└── lab_geopandas.ipynb
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

Końcowa struktura katalogu roboczego:

```text
lab_geo/
├── dane/
│   ├── pm25_sensors.csv
│   ├── opole_districts.geojson
│   └── opole_schools.geojson
├── wyniki/
│   ├── pm25.gpkg
│   └── districts.geojson
└── lab_geopandas.ipynb
```

## Podsumowanie

W ćwiczeniu przeprowadzono pełny proces analizy danych przestrzennych od geometrii tworzonych w Shapely, przez czyszczenie danych pomiarowych i przekształcenie ich do GeoDataFrame, po analizy przestrzenne z użyciem dzielnic Opola oraz lokalizacji szkół. Zastosowano dwa układy współrzędnych: `EPSG:4326` do danych geograficznych oraz `EPSG:2180` do obliczeń odległości i powierzchni. Wyniki zapisano w formatach GeoPackage i GeoJSON, aby można było je ponownie wykorzystać, np. w QGIS.

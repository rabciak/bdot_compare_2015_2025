Analiza zmian urbanistycznych w Lublinie (2015 - 2025)

Projekt to interaktywna mapa porównawcza dla Lublina, która pokazuje jak zmieniło się miasto na przestrzeni ostatniej dekady. Skupiłem się na dwóch aspektach: ekspansji budynków oraz zmianach w zalesieniu terenu. Całość opiera się na danych z BDOT10k i rozwiązaniach open source.

Jak to działa?
Dane siedzą w bazie PostgreSQL z rozszerzeniem PostGIS. GeoServer pobiera je stamtąd i serwuje jako warstwy WMS na frontend zbudowany w Leaflet.js. Wszystko jest skonteneryzowane (Docker) i wystawione na świat przez Traefika z certyfikatami SSL.

Co można zobaczyć na mapie?
- Budynki z 2025 i 2015 roku: Porównanie starej i nowej zabudowy.
- Lasy: Porównanie zmian w danych z 2015 oraz 2025.
- Analiza zagęszczenia: Siatka hexagonów kolorowana wg. liczby budynków.
- Nowe inwestycje: Warstwa pokazująca tylko te obiekty, któych nie było w 2015 a istnieją w 2025.

Logika analizy w PostGIS:

Poniżej znajdują się kluczowe zapytania SQL, które wykorzystałem w SQL Views GeoServera do generowania analiz przestrzennych w locie.

1. Generowanie siatki hexagonów z zagęszczeniem budynków:
To zapytanie tworzy geometryczną siatkę i zlicza budynki w każdym oczku, co pozwala stworzyć heatmapę bez statycznych plików.
~~~~sql
WITH bounds AS (
    -- Tutaj definiujemy zasięg danych
    SELECT ST_SetSRID(ST_MakeEnvelope(22.4, 51.1, 22.7, 51.3), 4326) as envelope
),
grid AS (
    -- Generujemy siatkę na podstawie stałego zasięgu
    SELECT geom 
    FROM ST_HexagonGrid(0.002, (SELECT envelope FROM bounds))
)
SELECT 
    row_number() OVER() as id, -- Unikalny identyfikator dla GeoServera
    grid.geom, 
    count(b.id) as ile_budynkow
FROM grid
LEFT JOIN budynki_2025 b 
    ON ST_Intersects(grid.geom, b.geom)
GROUP BY grid.geom
HAVING count(b.id) > 0
~~~~

2. Wykrywanie nowych budynków (2025 vs 2015):
Zapytanie filtruje budynki z najnowszej bazy, które nie mają części wspólnej z obrysami z roku 2015.
~~~~sql
SELECT b25.id, b25.geom
FROM budynki_2025 b25
LEFT JOIN budynki_2015 b15 
  ON ST_Intersects(b25.geom, b15.geom)
WHERE b15.id IS NULL
~~~~
Najważniejsze funkcje frontendu:
Dodałem suwak przezroczystości, który działa jak kurtyna – przesuwasz go i widzisz jak miasto "puchnie" od nowych budynków. Warstwy są ułożone w panele (Leaflet Panes), żeby lasy nigdy nie przykrywały budynków, a hexagony były czytelne jako tło analityczne.

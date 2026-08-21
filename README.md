# Analiza danych w Google BigQuery

Korzystając z dostępu do zbioru danych w Google Bigquery dotyczących podróży taksówkami w Chicago dokonałem analizy:
- sprawdziłem co zawiera zbiór danych
- oceniłem i zweryfikowałem jego jakość
- wyciągnałem wnioski oraz rekomendacje dla biznesu
- używałem zapytań  SQL
  
Link do zbioru danych:
- https://console.cloud.google.com/marketplace/product/city-of-chicago-public-data/chicago-taxi-trips

---
## Przed Analizą
### Sprawdzam wielkość pliku oraz ilość wierszy
![1.png](assets/1.png)
```sql
SELECT COUNT(*) AS total_rows
FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`;
```
![2.png](assets/2.png)

### Sprawdzam, czy dane posiadają odpowiednie wartości
```sql
SELECT COUNT(*) AS missing_fare
FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
WHERE fare IS NULL OR fare <= 0;
```
![3.png](assets/3.png)

Wartość fare oznacza opłatę należną za przejazd taksówką. W naszym przykładzie tylko poniżej 1% wszystkich wartości danej kolumny jest nieprawidłowa. 

### Sprawdzam więcej kolumn jednym zapytaniem
```sql
SELECT
COUNT(*) AS total_rows,
COUNT(trip_start_timestamp) AS non_null_start_timestamp,
COUNTIF(trip_seconds > 0) AS valid_trip_seconds,
COUNTIF(trip_miles > 0) AS valid_trip_miles,
COUNTIF(fare > 0) AS valid_fare
FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`;
```
![4.png](assets/4.png)

W niektórych przypadkach warto również sprawdzić pojawianie się duplikatów.

---

## Analiza

### Najpopularniejsze obszary początkowe, rok 2022
```sql
SELECT pickup_community_area AS area, COUNT(*) AS trips
FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
WHERE trip_start_timestamp BETWEEN '2022-01-01' AND '2022-12-31'
GROUP BY area
ORDER BY trips DESC
LIMIT 5;
```
![5.png](assets/5.png)

### Godziny szczytu, rok 2022
Należy uwazać na strefę czasową, domyślnie czas jest pokazywany w UTC.
```sql
SELECT
EXTRACT(HOUR FROM trip_start_timestamp AT TIME ZONE 'America/Chicago') AS hour,
COUNT(*) AS trips
FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
WHERE EXTRACT(YEAR FROM trip_start_timestamp AT TIME ZONE 'America/Chicago') = 2022
GROUP BY hour
ORDER BY trips DESC;
```
![9.png](assets/9.png)

### Średnia, przybliżona mediana, odchylenie standardowe dla poszczególnych lat
```sql
SELECT
EXTRACT(YEAR FROM trip_start_timestamp) AS year,
COUNT(*) AS trip_count,

-- Duration
AVG(trip_seconds) AS avg_duration,
APPROX_QUANTILES(trip_seconds, 100)[OFFSET(50)] AS median_duration,
STDDEV(trip_seconds) AS stddev_duration,
MIN(trip_seconds) AS min_duration,
MAX(trip_seconds) AS max_duration,

-- Distance
AVG(trip_miles) AS avg_distance,
APPROX_QUANTILES(trip_miles, 100)[OFFSET(50)] AS median_distance,
STDDEV(trip_miles) AS stddev_distance,
MIN(trip_miles) AS min_distance,
MAX(trip_miles) AS max_distance,

-- Fare
AVG(fare) AS avg_fare,
APPROX_QUANTILES(fare, 100)[OFFSET(50)] AS median_fare,
STDDEV(fare) AS stddev_fare,
MIN(fare) AS min_fare,
MAX(fare) AS max_fare

FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`

WHERE trip_start_timestamp IS NOT NULL
AND trip_seconds > 0
AND trip_miles > 0
AND fare > 0

GROUP BY year
ORDER BY year;
```
![8.png](assets/8.png)

![7.png](assets/7.png)

---

## Google PowerQuery → Power BI (DirectQuery)
W tym konkretnym przypadku do analizy danych w PowerBI najlepiej użyć DirectQuery zamiast klasycznego importa danych. Dane nie znajdują się bezpośrednio lokalnie na komputerze, a Power BI jest bezpiecznie połączony i korzysta z konkretnych danych tylko jeśli potrzeba. Niektóre kolumny w tej bazie danych zawierają w większości puste wartości. Dla lepszego performance należy takie kolumny pominąć tworząc odpowiednie zapytanie podczas nawiązywania połączenia DirectQuery z bazą danych.

Jeśli interesuje nas głównie jakiś okres czasu to (próbka, np. ostatnie 2-3 lata) to można również użyć odpowiedniego zapytania SQL, aby wygenerować odpowiednie dane oraz zaimportować je do bezpośrednio do PowerBI, do głębokiej analizy.

Plik .pdf dotyczący wizualizacji wszystkich danych w PowerBI jest dostępny wyżej do pobrania.

---

## Wnioski i rekomendacje
Analiza cenowa: Utrzymać ceny konkurencyjne latem, a zimą rozważyć promocje (np. rabaty na nocne kursy), żeby przyciągnąć klientów mimo słabszego popytu.

Optymalizacja tras i harmonogramów: Skoncentrować taksówki w najbardziej ruchliwych obszarach (Area 8 to Near North Side) szczególnie w godzinach szczytu (03:00 - 13:00). Zapewnienie gotowości w dzielnicach generujących najwięcej postojów zwiększy liczbę zleceń.
git
Segmentacja klientów: Wykorzystać fakt, że głównie centra biznesowe i turystyczne generują kursy – można kierować tam reklamy internetowe lub zniżki lojalnościowe dla podróżujących biznesmenów i turystów.

Kontrola kosztów: Obserwować firmy liderów (Taxi Affiliation Services itp.) – jeśli oferują dodatkowe usługi (np. lepsze warunki płatności kartą czy promocje).

---

## Co jeśli tych danych będzie 10 razy więcej?

### Skalowanie do większych zbiorów
Jeśli dane byłyby **10 razy większe**, nadal da się je analizować BigQuery, ale trzeba zastosować dodatkowe optymalizacje. Przede wszystkim warto **partycjonować tabelę**, np. po dacie kursu, co pozwoli czytać tylko pasujące partycje zamiast całej tabeli. Można też **klastrować** tabelę po kluczowych kolumnach (np. po obszarze community), co przyspieszy filtrowanie wielowymiarowe. Pozwala to znacząco ograniczyć ilość danych przetwarzanych w zapytaniu. Dodatkowe kroki to korzystanie z **próbkowania** danych i optymalizacja samego zapytania. Jeśli BigQuery jest niewystarczające, można sięgnąć również po inne narzędzia.
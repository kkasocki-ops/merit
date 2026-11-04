# CLAUDE.md — praca inżynierska: mini-EUPHEMIA w PyPSA

## Jak ze mną pracować (NAJWAŻNIEJSZE)
- **Domyślnie NIE piszesz kodu w plikach.** Odpowiadasz w czacie: tłumaczysz i pokazujesz przykład kodu, a ja przepisuję go i uruchamiam sam. Nie twórz i nie edytuj notatników ani plików `.py`, nie uruchamiaj kodu za mnie.
  - Wyjątek: gdy wprost poproszę („napisz to za mnie”, „wstaw do notatnika”), piszesz kod w pliku, ale tylko to, o co prosiłem.
  - Przykład w czacie ma być krótki (jeden krok, jedna komórka), żebym mógł go przepisać i zrozumieć, a nie gotowe rozwiązanie całego zadania naraz.
- Jestem **początkujący w programowaniu**. Kod ma być **prosty, nie optymalny**. Lepiej dłużej i czytelnie niż krótko i sprytnie.
- **Wzór stylu:** zajrzyj raz do moich notatników w `/Users/kacper/_taski/2_porownanie_modeli_load/comparison/` (spoza tego projektu, nic stamtąd nie kopiuj), żeby zobaczyć, jak mniej więcej piszę. W skrócie:
  - importy zawsze takie same: `import pandas as pd`, `import numpy as np`, `import matplotlib.pyplot as plt`,
  - notatnik podzielony nagłówkami markdown na części (np. `# Wczytanie danych`, `# Funkcje`, `# Porównanie`, `# Wykresy`),
  - każdy plik wczytany osobną linijką `pd.read_csv("data/plik.csv")` do zmiennej o opisowej nazwie (`ceny_pl`, `load_de`),
  - podgląd danych w osobnych komórkach: samo `df`, `df.info()`, `df.describe()`, filtr `df[df["kolumna"] > x]`,
  - proste funkcje `def`, które liczą coś jednego i zwracają `pd.DataFrame` (jak moja `comparison(name, df)` z MAE i RMSE),
  - słownik `{nazwa: df}` i zwykła pętla `for nazwa, df in slownik.items():`, gdy to samo robię dla wielu krajów lub plików,
  - łączenie tabel przez `pd.merge(..., on="kolumna_czasu")`, zmiana nazw przez `.rename(columns={...})`,
  - czas: `pd.to_datetime(df["kolumna"], utc=True).dt.tz_localize(None)`, czyli UTC bez strefy w obiekcie,
  - wykresy: `plt.figure(figsize=(18, 6))`, `plt.plot(...)`, `plt.title(...)`, `plt.ylabel(...)`, `plt.legend()`, `plt.show()`,
  - tabele wyników przez `pivot`, heatmapy przez `seaborn.heatmap`.
- Kod (w przykładach i gdy piszesz za mnie) w stylu notatnika (`.ipynb`): małe komórki, jedna komórka = jeden krok.
- Kluczy API i haseł nie wpisuj w kod. Trzymam je w pliku `.env` w głównym folderze projektu (jest w `.gitignore`, nie commitujemy go). Wzór bez klucza: `.env.example`.
  - Klucz ENTSO-E: zmienna `ENTSOE_API_KEY`. W notatniku wczytuję go przez `load_dotenv("../.env")` i `os.getenv("ENTSOE_API_KEY")` (biblioteka `python-dotenv`).
- Przed kodem: 1–2 zdania, co robimy i po co.
- Po kodzie: wytłumacz każdą nową rzecz, czyli co robi funkcja, co znaczą argumenty i co wychodzi. Bez zakładania, że coś „wiem”.
- Komentarze w kodzie po polsku.
- Po każdym kroku dodaj sprawdzenie wyniku (`.head()`, `.shape`, `print`, prosty wykres), żebym widział, co się dzieje.
- Opisowe nazwy zmiennych (`ceny_pl`, `zapotrzebowanie`), a nie `df2`, `x`, `tmp`.
- Proste konstrukcje: `pd.read_csv`, filtrowanie `df[df["zone"] == "PL"]`, `groupby`, `pivot`, zwykła pętla `for`, `if`, proste funkcje `def`.
- **Unikaj:** klas, dekoratorów, zagnieżdżonych list comprehension, `lambda`, długich łańcuchów metod (`.a().b().c().d()`), type hints, `argparse`, `logging`, wielowątkowości i „sprytnych” jednolinijkowców.
- Jeśli coś jest wolne (np. duże CSV), powiedz o tym i zaproponuj prosty sposób (np. `usecols=`, wczytanie tylko potrzebnych stref).
- Przy błędzie: wyjaśnij, co znaczy komunikat, skąd się wziął i jak go naprawić.
- Zmieniaj tylko to, o co proszę. Nie przepisuj całego notatnika bez pytania.
- Nowe biblioteki dodawaj dopiero po zapytaniu, przez `uv add <pakiet>`.
- Komunikacja po polsku, zwięźle.

## Temat i cel
- **Temat:** wyznaczenie cen energii elektrycznej na rynku dnia następnego (day-ahead) dla PL, DE, CZ, SK, LT i SE uproszczonym modelem market couplingu („mini-EUPHEMIA”) w PyPSA (Python).
- **Cel:** odtworzyć historyczne ceny DA w strefach (backcast / hindcast) i porównać je z rzeczywistymi cenami.
- **Kontekst:** praca inżynierska, kierunek energetyka.
- **Oryginalność:** nie znaleziono pracy z dokładnie tym zestawem krajów jako „mini-EUPHEMIA” w PyPSA, ale sama metodologia jest ugruntowana i można ją cytować (patrz „Literatura”).

## Ustalenia i założenia

### Metoda
- Strefa cenowa = jeden węzeł (`Bus`). Połączenie transgraniczne = `Link` z limitem NTC.
- Park wytwórczy zamrożony na stanie historycznym. Optymalizujemy tylko dispatch (merit order, koszt krańcowy).
- Cena modelu = `n.buses_t.marginal_price`.
- Walidacja = **backcast** (nie „backtest”): model dostaje rzeczywiste dane wejściowe danego roku, a wynik porównujemy z rzeczywistymi cenami DA.
- Miary błędu: MAE [EUR/MWh] i SMAPE [%]. Najlepiej mieć rok główny i rok kontrolny jako ochronę przed overfittingiem (jak w pypsa-eur-backcast).
- Ograniczenia modelu kosztowego, które trzeba opisać w pracy od razu:
  - zaniżone szczyty cenowe, bo model nie ma narzutu ofertowego,
  - prawie brak cen ≤ 0,
  - NTC zamiast flow-based (granice DE–PL–CZ–SK w regionie CORE działają w trybie flow-based od 06.2022). Argument za uproszczeniem: na granicy DE–PL modele NTC i flow-based dają zgodne wyniki (gskurokawa).

### Dane: baza bloków (stan na X 2026)
- **Źródła:** powerplantmatching (PyPSA), OPSD (rozbicie na bloki), weryfikacja w GEM Wiki, na liście BNetzA (11.03.2026) i u Ignitis.
  - Bez bloków wyłączonych do X 2026 i bez bloków w budowie (Grudziądz CC1, Ostrołęka C, Kruonis 5).
  - Bez jednostek < 1 MW, poza rekordami zbiorczymi DE: biomasa 6,7 GW, baterie 18,5 GW.
  - Bloki DE w rezerwie sieciowej (28 rekordów) mają w nazwie **„[rezerwa sieciowa]”**. W modelu DA trzeba je **odfiltrować**.
  - Biogaz DE ≥ 1 MW: 784 instalacje, etykieta „biomasa – biogaz”.
  - Sprawność jest albo zgłoszona (OPSD, Ignitis), albo szacowana (regresja względem roku uruchomienia z bloków DE albo PyPSA technology-data). Który przypadek, mówi kolumna `źródło`.
- **NTC (plik `ntc_granice_PL_DE_CZ_SK_LT_SE.csv`, 18 kierunków):**
  - łącza HVDC / NTC: SE4–LT 700, SE4–DE 615, SE4–PL 600, PL–LT 500 MW (Nord Pool / moc znamionowa); PL–LT po synchronizacji realnie ok. 150 MW,
  - granice flow-based: historyczne maksima NTC 2015–2016 (JRC Dispa-SET); DE–PL (2231 / 230 MW) oznaczone jako wątpliwe,
  - lepszą alternatywą są pliki maxBex w `data/` (patrz niżej).

### Kategorie paliw (kolumna `rodzaj paliwa`)
- węgiel kamienny, węgiel brunatny, olej, jądrowe (uran), biomasa stała, biomasa – biogaz
- gaz: CCGT, OCGT, blok parowy, silnik tłokowy
- woda – przepływowa / zbiornikowa / typ nieokreślony (razem „woda – bez ESP”)
- magazyn – elektrownia szczytowo-pompowa (ESP), magazyn – bateria
- Uwagi: CZ Vřesová spala gaz z węgla (ENTSO-E: „Fossil coal-derived gas”). Kategoria „Others” w ENTSO-E dla SE = biomasa + olej + gaz.

### Oprogramowanie
- Python ≥ 3.14, środowisko **uv** (`pyproject.toml`, `uv.lock`, `.venv/`).
- Zainstalowane: pandas, matplotlib, entsoe-py, python-dotenv. Do dodania: `pypsa` i solver (np. `highspy`, darmowy, domyślny w PyPSA).
- Ciężkie obliczenia można puszczać na domowym PC przez zdalny pulpit.

### Jednostki i konwencje
- Moc: MW (zestawienia zbiorcze: GW). Energia: MWh. Ceny: EUR/MWh. Sprawność: ułamek 0–1.
- Strefy jak w danych: `PL`, `DE-LU`, `CZ`, `SK`, `LT`, `SE1`–`SE4`.
- Czas: trzymamy w UTC, a na lokalny zamieniamy tylko na wykresach. Przy każdym pliku sprawdź, w jakiej strefie czasowej jest kolumna z datą.
- CSV: UTF-8, separator przecinek. W moich plikach nazwy kolumn są po polsku, z jednostką w nawiasie (np. `moc zainstalowana [MW]`).
- Każdy rekord danych ma `źródło`. Wartości szacowane albo niezweryfikowane oznaczamy wprost.
- Luki i rozbieżności w danych zgłaszaj od razu, z wyjaśnieniem.
- Terminologia: backcast (nie backtest), strefa cenowa, NTC, flow-based, ESP.

## Zrobione
- Przegląd podobnych projektów i literatury (patrz niżej).
- Baza bloków `data/bloki_PL_DE_CZ_SK_LT_SE.csv`: 3907 rekordów.
- Pliki NTC (`ntc_granice_PL_DE_CZ_SK_LT_SE.csv`) i porównanie mocy z ENTSO-E (`porownanie_z_ENTSO-E.csv`). **Nie ma ich jeszcze w `data/`.**
- Projekt uv, repozytorium git (bez commitów), dane z 2025–2026 w `data/` (opis poniżej).

## Pliki w `data/`
| Plik | Co zawiera | Zakres | Uwagi |
|---|---|---|---|
| `bloki_PL_DE_CZ_SK_LT_SE.csv` | baza bloków | stan X 2026 | opis wyżej |
| `price_actual.csv` | rzeczywiste ceny DA (`zone`, `delivery_start`, `delivery_end`, `price`, `currency`) | 02.01.2025–10.2026, wszystkie strefy | cały plik w krokach 15 min, choć do 30.09.2025 rynek był godzinowy (cztery kwadranse w godzinie powinny mieć tę samą cenę, sprawdzić); czas prawdopodobnie UTC (pierwszy rekord 2025-01-01 23:00 = północ czasu polskiego) |
| `prognoza_load.csv` | prognozy zapotrzebowania (Kpler, model EC_OP), krok 15 min | 01.2025–11.2026 | **brak DE-LU**; LT, SK i SE dopiero od 31.01.2025; kilka przebiegów prognozy na jedną godzinę (kolumny `run`, `rn`) – wybrać jeden |
| `prognoza_gen.csv` | prognozy generacji wiatr i PV (`fuel_type`: `wind`, `solar`), Kpler | 01.2025–11.2026 | brak wiatru SK; PV SE od 07.03.2025; jak wyżej z przebiegami |
| `links/sce.csv` | planowana wymiana handlowa DA między strefami (`zone_from`, `zone_to`, `value` [MW]), ENTSO-E | 01.2025–10.2026 | do porównania przepływów z modelu |
| `links/nordic_max_bex.csv` | maxBex flow-based dla regionu nordyckiego, granice w kolumnie `extra_json` (`border_SE4_PL` itp.) | 01.2025–10.2026 | dane JSON w kolumnie trzeba rozpakować |
| `323d9994-…-bd83ea9c7b00.csv` | prawdopodobnie maxBex flow-based dla regionu CORE (`border_AT_CZ`, `border_DE_PL` …) w `extra_json` | 01.2025–10.2026 | warto zmienić nazwę na np. `core_max_bex.csv` |

- Dla cen DA prognozy zapotrzebowania i OZE są uzasadnionym wejściem, bo rynek DA rozlicza się na prognozach z dnia poprzedniego. Trzeba to opisać w pracy.
- Dane obejmują 2025 r., więc to naturalny kandydat na rok główny backcastu.

## Do zrobienia

### Decyzje otwarte
- Rok główny i kontrolny analizy.
- SE jako SE1–SE4 (dane już są w tym podziale; LT i PL łączą się z SE4).
- Krok czasowy: 1 h czy 15 min (od 10.2025 rynek DA działa w kroku 15 min).
- Limity przesyłu: maxBex z `data/` czy NTC na 2035 r. z ERAA 2025.

### Dane
- Ceny paliw i CO₂, dynamiczne (dzienne), nie stałe.
- Zapotrzebowanie DE-LU (brakuje go w `prognoza_load.csv`).
- Produkcja wodna (szczególnie SE) i dostępność atomu.
- `p_min_pu` dla węgla w PL (wymuszona praca JWCD; wzór: PyPSA-PL-mini).

### Poprawki bazy bloków
- Pątnów 2 B9 (474 MW): prawdopodobnie nie pracuje, do usunięcia po potwierdzeniu.
- DE olej: odfiltrować agregaty awaryjne z MaStR (6,25 vs 3,54 GW).
- Możliwe duplikaty w ppm (np. trzy rekordy dla Łodzi).
- Brakująca biomasa: CZ −0,38, SK −0,21, LT −0,11 GW.
- Brakujące EC: PL (Orlen Płock, ok. 0,4 GW gaz + 0,41 GW olej), LT (Wilno, Kowno, ok. 0,44 GW).
- Niezweryfikowane bloki DE 2020+ (np. Reuter West, GKH).

### Model i praca
- Kolejność: wczytanie danych → sieć PyPSA → optymalizacja → ceny → walidacja (MAE, SMAPE) → wykresy.
- Porównanie MAE z pypsa-eur-backcast.
- Rozdziały: wstęp, rynek DA i EUPHEMIA, metodyka, dane, wyniki, walidacja, dyskusja ograniczeń, wnioski.

### Scenariusze (gdy model będzie działał)
Każdy scenariusz liczymy jako różnicę względem roku bazowego: zmieniamy jedną rzecz, resztę zostawiamy.
1. **Z market couplingiem i bez niego:** NTC = 0 wobec rynków połączonych (zbieżność cen, welfare, renta z ograniczeń przesyłowych). Wariant pośredni: NTC ×0,5 / ×1,5.
2. **Szok cen paliw i CO₂:** gaz ok. 130 vs 30–40 EUR/MWh; CO₂ 50 / 100 / 150 EUR/t.
3. **Polska 2035:** dane z **ERAA 2025** (rok docelowy 2035: moce wg strefy, godzinowe zapotrzebowanie, profile PECD, NTC). Pliki: `ERAA_2025_Dashboard_RawData.zip`, `ERAA_2025_DemandData.zip`, `ERAA_2025_PECDRES.zip`, `ERAA_2025_NTCs.zip` → `data/eraa2025/`.
   - ACER uważa, że ERAA 2025 zawyża wzrost zapotrzebowania, więc wariant wrażliwości z niższym zapotrzebowaniem (KPEiK: ok. 200 TWh w 2030, ok. 270 TWh w 2040).
   - Wariant bazowy **bez atomu w PL** (KPEiK: pierwsze bloki w 2. połowie lat 30.), atom jako osobny wariant.
   - Ten sam rok klimatyczny dla zapotrzebowania, wiatru, PV i wody.
4. **Rok suchy i mokry w SE:** produkcja wodna ±20%.
5. **Kontrfakty:** DE z atomem (ok. 4 GW wyłączone w 04.2023); LT przed i po synchronizacji (PL–LT 500 vs ok. 150 MW).
- Mierzymy: średnią cenę w strefie, % godzin z równymi cenami, przepływy, saldo eksportu, koszt systemu / welfare, emisje CO₂.

## Struktura folderów
Stan obecny:
```
untitled folder/
  CLAUDE.md          # tylko odsyłacz do docs/CLAUDE.md
  docs/CLAUDE.md     # ten plik
  pyproject.toml     # środowisko uv
  uv.lock
  .venv/
  main.py            # szablon uv, nieużywany
  data/              # dane wejściowe (opis wyżej)
  notebooks/
    data.ipynb       # start: na razie same importy
```
Docelowo (propozycja, prosta):
```
data/                # surowe dane, nie edytować ręcznie
  eraa2025/          # dane do scenariusza 2035
  paliwa/            # ceny gazu, węgla, CO₂
notebooks/           # numerowane, wykonywane po kolei
  01_dane.ipynb              # wczytanie i czyszczenie danych
  02_siec_pypsa.ipynb        # budowa sieci
  03_backcast_walidacja.ipynb  # porównanie z cenami rzeczywistymi
  04_scenariusze.ipynb
results/
  wykresy/
  ceny/
docs/                # CLAUDE.md, notatki, tekst pracy
```
- Plik `.py` z funkcjami (np. `notebooks/funkcje.py`) dopiero wtedy, gdy ten sam kod powtarza się w kilku notatnikach.
- Dużych CSV z `data/` nie commitujemy do gita (część ma ponad 100 MB, a to limit GitHuba dla jednego pliku).

## Literatura i projekty referencyjne
- pypsa-eur-backcast (Open Energy Transition): najbliższy odpowiednik; MAE 2023/2019, lista znanych problemów (LT za drogie, DE za duży eksport).
- PyPsa-European-power-spreads (gskurokawa): NTC vs flow-based (maxNetPos, CNEC z JAO).
- Karkossa, Saretta, Victoria (DTU, 2026), arXiv 2606.16486: hindcast 2020–2024, SMAPE, dynamiczne ceny paliw.
- PyPSA-Eur: walidacja (`validate_elec_networks`); przykład PyPSA „scigrid-redispatch”.
- PyPSA-PL-mini (Instrat): kalibracja PL.
- OpenDAM: otwarta implementacja market couplingu (punkt odniesienia dla EUPHEMII). ASAM: model agentowy MESA + PyPSA.
- ERAA 2025 (ENTSO-E) i uwagi ACER do danych wejściowych; KPEiK (Ministerstwo Energii).

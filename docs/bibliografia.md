# Bibliografia

Wszystkie źródła danych, wartości i faktów użytych w pracy i w notatnikach.
Przy każdej pozycji: co z niej wzięto i gdzie jest użyta. Pozycje oznaczone **[do uzupełnienia]** wymagają jeszcze dokładnego źródła (link, data dostępu).

## Dane wejściowe

1. **ENTSO-E Transparency Platform – planowana wymiana handlowa DA (Scheduled Commercial Exchanges)**, pobrane przez API Kpler (`/power/interconnections/commercial-schedules`, `provider = entsoe`).
   https://transparency.entsoe.eu
   Użyte: `data/links/sce.csv` → statystyki wymiany, przepustowości granic CORE w `data/links/ntc_2025.csv` (99. percentyl 2025).
2. **Kpler – prognozy zapotrzebowania oraz generacji wiatrowej i PV** (model EC_OP, przebieg 00z, krok 15 min, prognozy D-1).
   Użyte: `data/prognoza_load.csv`, `data/prognoza_gen.csv`. **[do uzupełnienia: opis usługi / dokumentacja API]**
3. **Ceny DA (rzeczywiste)**, tabela `obs_price_day_ahead`. Użyte: `data/price_actual.csv`.
   **[do uzupełnienia: pierwotne źródło cen – giełda / ENTSO-E / dostawca danych]**
4. **maxBex CORE (flow-based)** – plik `data/323d9994-dc99-4910-9b23-bd83ea9c7b00.csv`. Publikuje JAO.
   https://www.jao.eu
   Użyte: wykresy maxBex dla granic PL, DE, CZ, SK; granica PL–LT jako wirtualny hub `PL_LT_BigHub` (dane od 10.06.2026).
5. **maxBex Nordic (flow-based)** – plik `data/links/nordic_max_bex.csv`. **[do uzupełnienia: wydawca – Nordic RCC / JAO Nordic]**
   Użyte: łącza HVDC SE4–PL (600 MW), SE4–LT (700 MW), SE4–DE (615 / 600 MW) w `data/links/ntc_2025.csv` (mediana 2025).

## Ceny paliw i CO₂ (planowane, dzienne, 2025)

- **EUA (ETS)** – Investing.com, kontrakty futures na uprawnienia do emisji CO₂ (EUR/t). **[do uzupełnienia: dokładna nazwa serii, data pobrania]**
- **Gaz** – Investing.com, „Dutch TTF Natural Gas Futures” (EUR/MWh). **[do uzupełnienia]**
- **Węgiel kamienny** – Investing.com, „Rotterdam Coal Futures” (API2 CIF ARA, USD/t); dla PL alternatywnie indeks PSCMI 1 (ARP, miesięczny, PLN/GJ). **[do uzupełnienia]**
- **Olej** – Investing.com, „ICE Crude Oil (IPCI)” – dane historyczne (USD/bbl) jako przybliżenie ceny oleju opałowego.
  https://www.investing.com/indices/ice-crude-oil-historical-data (dostęp: 04.11.2026)
  Użyte: cena oleju w koszcie krańcowym bloków olejowych.
- **Przeliczenie USD/bbl → USD/GJ** – 1 BOE (baryłka ekwiwalentu ropy) = 6,12 GJ, czyli cena [USD/GJ] = cena [USD/bbl] / 6,12.
  TraditionalOven.com, „Convert barrel of oil equivalent (BOE) to giga joule (GJ)”.
  https://www.traditionaloven.com/tutorials/energy/convert-barrel-oil-equivalent-boe-to-giga-joule-gj.html (dostęp: 04.11.2026)
  Użyte: przeliczenie ceny oleju na USD/GJ.
- **Przeliczenie alternatywne (niewykorzystane)** – 1 bbl = 158,987 l; gęstość Brent ok. 0,835 kg/l; wartość opałowa ropy naftowej 42,3 GJ/t (IPCC 2006 Guidelines, Vol. 2, Ch. 1, Tab. 1.2) → ok. 5,6 GJ/bbl (1,56 MWh_th/bbl). **[do uzupełnienia: źródło gęstości Brent]**
- **Kursy walut** – EUR/USD (Investing.com) i EUR/PLN (NBP, tabela A). **[do uzupełnienia]**
- **Wskaźniki emisji CO₂ paliw** – „Wskaźniki emisji KOBiZE 2025 – kompletny przewodnik po aktualnych wartościach i zmianach w raportowaniu”, RECO.
  https://reco.com.pl/wskazniki-emisji-kobize-2025-kompletny-przewodnik-po-aktualnych-wartosciach-i-zmianach-w-raportowaniu/ (dostęp: 04.11.2026)
  Źródło pierwotne: KOBiZE, „Wartości opałowe (WO) i wskaźniki emisji CO₂ (WE)” do raportowania w ramach EU ETS. **[do uzupełnienia: dokument KOBiZE, rok]**
  Użyte: współczynnik emisji [t CO₂/MWh_th] w koszcie krańcowym bloków.
- **Koszty stałe paliw** (węgiel brunatny, uran, biomasa) – PyPSA technology-data. https://github.com/PyPSA/technology-data

## Wartości przyjęte ręcznie

6. **LitPol Link** – Wikipedia (hasło „LitPol Link”). https://en.wikipedia.org/wiki/LitPol_Link (dostęp: 04.11.2026).
   Użyte: przepustowość PL–LT = 500 MW w obu kierunkach w `data/links/ntc_2025.csv` (brak danych o tej granicy za 2025 w maxBex i w `sce.csv`).

## Fakty o rynku użyte w opisach

7. Flow-based w regionie CORE od 06.2022. **[do uzupełnienia: komunikat JAO / CORE TSOs]**
8. Flow-based w regionie nordyckim od 10.2024. **[do uzupełnienia: komunikat Nordic RCC / Nord Pool]**
9. Rynek DA w kroku 15 min od 10.2025 (SDAC 15-min MTU). **[do uzupełnienia: komunikat SDAC / ENTSO-E]**
10. Synchronizacja państw bałtyckich z siecią kontynentalną w 02.2025. **[do uzupełnienia: ENTSO-E / Litgrid]**
11. Zamknięcie aukcji DA o 12:00 CET w dniu D-1 (użyte do sprawdzenia, czy prognozy są D-1). **[do uzupełnienia: SDAC / EUPHEMIA]**

## Baza bloków (`data/bloki_PL_DE_CZ_SK_LT_SE.csv`)

12. powerplantmatching v0.9.1 (PyPSA). https://github.com/PyPSA/powerplantmatching
13. Open Power System Data (OPSD) – rozbicie na bloki. https://open-power-system-data.org
14. PyPSA technology-data, `costs_2025.csv` (sprawności typowe dla technologii). https://github.com/PyPSA/technology-data
15. Global Energy Monitor – GEM Wiki (weryfikacja bloków).
16. Bundesnetzagentur – lista elektrowni (stan 11.03.2026).
17. Ignitis (dane bloków LT).

## Literatura i projekty referencyjne

18. pypsa-eur-backcast (Open Energy Transition).
19. PyPsa-European-power-spreads (gskurokawa) – porównanie NTC i flow-based.
20. Karkossa, Saretta, Victoria (DTU, 2026), arXiv:2606.16486 – hindcast 2020–2024, SMAPE, dynamiczne ceny paliw.
21. PyPSA-Eur. https://github.com/PyPSA/pypsa-eur
22. PyPSA-PL-mini (Instrat) – kalibracja PL.
23. OpenDAM – otwarta implementacja market couplingu.
24. ASAM – model agentowy MESA + PyPSA.
25. ERAA 2025 (ENTSO-E) i uwagi ACER do danych wejściowych.
26. KPEiK (Ministerstwo Energii).
27. JRC Dispa-SET – historyczne maksima NTC 2015–2016.

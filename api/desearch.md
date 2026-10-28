# Desearch — ocena API dla Tropici3l

Stan na: 2026-10-28. Źródła: strona, cennik, dokumentacja i specyfikacja OpenAPI Desearch (`https://api.desearch.ai/openapi.json`, wersja 1.0.0), pobrane tego dnia. Nie testowano API: brak klucza, żadnych wywołań płatnych. Ceny Desearch są „stawkami referencyjnymi” z 2026-05-07 (People Search: 2026-10-19); aktualne stawki pokazuje Console.

## Czym jest

Desearch to API wyszukiwania dla agentów AI, jeden klucz do kilku źródeł: web, X, ekstrakcja stron, TikTok, Instagram, Facebook, wyszukiwanie osób. Indeks budowany jest przez Bittensor Subnet 22; Desearch udostępnia API na jego podstawie. Jest REST, SDK Python i JS oraz hostowany serwer MCP (`https://mcp.desearch.ai/mcp`).

## Najważniejsze ustalenie: Reddit nie jest obsługiwany jako osobne źródło

Narzędzia Reddit, Hacker News, YouTube, Wikipedia i ArXiv są w API **wycofane**. Pole `reddit_search` nigdy nie jest zwracane. Dokumentacja każe szukać Reddita przez Web Search z `site:reddit.com` albo `include_domains: ["reddit.com"]`.

Oznacza to: Desearch znajdzie **linki do wątków z Reddita z fragmentem tekstu**. Nie daje dedykowanego dostępu do wątków, komentarzy, głosów ani historii subreddita. Pełną treść wątku można próbować pobrać przez Extract API; czy działa dla Reddita, trzeba sprawdzić testem.

Dla porównania TGramAI już czyta Reddita: `forum_scraper.py` pobiera nowe posty ze wskazanego subreddita przez `old.reddit.com/r/<sub>/new.json` (commit `ea37d83`). Wymaga znanego subreddita i przy blokadzie Reddita zwraca błąd 403. Nie wyszukuje wzmianek o projekcie w całym Reddicie. Ta luka (wyszukiwanie wzmianek) jest tym, co Desearch lub Brave mogą uzupełnić.

## Co jest przydatne

| Endpoint | Co daje | Znaczenie dla Tropici3l |
|---|---|---|
| `GET /twitter` | wyszukiwanie postów X z operatorami: `from:`, `$cashtag`, `since:`/`until:`, `min_faves:`, `lang:`; do 100 postów na zapytanie; sortowanie Top lub Latest | wzmianki o projekcie, kto pisze, historia cashtagów autorów |
| `GET /twitter/user/posts` | posty użytkownika z paginacją (`cursor`) | historia autora w deep research |
| `GET /twitter/post/retweeters`, `/twitter/replies/post` | kto udostępnił, odpowiedzi | sekcja X: reposty i reakcje |
| `GET /twitter/urls`, `/twitter/post` | pobranie konkretnych postów | dowód do linku w raporcie |
| `GET /web` | wyniki wyszukiwarki (tytuł, fragment, link), paginacja do 100 | Reddit przez `site:reddit.com`, fora, artykuły |
| `POST /search` | wyszukiwanie web z filtrem dat publikacji | źródła z zadanego okna czasu |
| `GET /web/extract` | czysta treść strony z URL, opcja JS | treść dokumentacji, wątków, blogów |
| `POST /desearch/ai/search` | wyniki web i X z podsumowaniem AI | mało przydatne: potrzebujemy dowodów, nie cudzych podsumowań |

Przydatne do kontroli kosztów: każda udana odpowiedź ma nagłówek `X-Desearch-Cost-Usd` z kosztem zapytania.

## Cennik (pay as you go, USD)

| Produkt | Cena | Uwagi |
|---|---|---|
| X Search | 0,15 USD / 1000 postów | minimum 10 postów na zapytanie; płatność za faktycznie zwrócone |
| Web Search | 1,00 USD / 1000 wyszukiwań | |
| Extract | 0,50 USD / 1000 stron | |
| AI Search | 0,40 USD / 1000 elementów | |
| X Profile Search | 40 USD / 1000 profili | FAQ podaje 0,15 USD / 1000 użytkowników; strona tego nie wyjaśnia |
| YouTube, LinkedIn, TikTok — wyszukiwanie osób | 50 USD / 1000 | nie dotyczy Tropici3l |

- Na start 5 USD kredytu po podpięciu karty w Console. Strona wspomina też o „do 155 USD” za zadania, bez opisu warunków.
- Brak abonamentów; Enterprise na indywidualnych warunkach.
- Limity zapytań nie są publikowane: obowiązują okna minutowe, dzienne i miesięczne widoczne w Console.
- Nie pobiera opłat m.in. za błędy 503 na endpointach X i za puste zapytanie 422 na `/web`. Dla innych błędów dokumentacja tego nie mówi.

## Szacunek kosztu dla Tropici3l

Założenia własne, do sprawdzenia testem.

| Scenariusz | Zużycie | Koszt |
|---|---|---|
| Badanie podstawowe: X | 5 zapytań × 100 postów | ok. 0,08 USD |
| Badanie podstawowe: web i Reddit | 10–20 wyszukiwań | 0,01–0,02 USD |
| Badanie podstawowe: ekstrakcja | 20 stron | 0,01 USD |
| Deep research: historia autorów X od 2020 | 20 autorów × 3000 postów = 60 000 postów | ok. 9 USD |
| Deep research: aktywni autorzy | 20 autorów × 10 000 postów = 200 000 postów | ok. 30 USD |

Zwykły research kosztuje groszowe kwoty. Realny koszt pojawia się dopiero przy pełnej historii autorów; tam warto ustalić limit postów na autora.

## Porównanie z Brave Search API

Stan na 2026-10-28, źródło: `https://brave.com/search/api/`.

| | Desearch | Brave Search API |
|---|---|---|
| Wyszukiwanie web | 1 USD / 1000 | 5 USD / 1000 |
| Darmowo | 5 USD jednorazowo, po podpięciu karty | 5 USD kredytu co miesiąc, czyli ok. 1000 zapytań |
| Limit | niepublikowany | 50 zapytań/s w planie Search |
| X | tak, tanio | nie |
| Reddit | tylko `site:reddit.com` | tylko przez wyniki web; nie sprawdzono, czy plan zwraca osobną sekcję dyskusji z forów |
| Prawo do zapisu wyników | nie znaleziono zapisu | tylko w planach z prawami do przechowywania; strona nie podaje, w których |

Przy ręcznym researchu miesięczny darmowy kredyt Brave prawdopodobnie wystarczy na wyszukiwanie web i Reddit. Desearch wyróżnia się tanim dostępem do X.

## Ocena

**Przydatne: tak, głównie dla X.** Desearch może zastąpić drogie lub ograniczone pobieranie X, zwłaszcza w deep research (historia cashtagów autorów od 2020 r.), gdzie operatory `from:` i `since:`/`until:` pozwalają dzielić pobieranie na okresy, zgodnie z zasadami projektu.

**Dla Reddita: tylko częściowo.** Znajduje wątki, ale nie zastępuje czytania subredditów i komentarzy. Jeśli brakującym elementem ScraperX jest wyszukanie, gdzie mówi się o projekcie, Desearch lub Brave to załatwią. Jeśli brakuje treści wątków z komentarzami, trzeba osobnego rozwiązania: Extract (do testu), skrypt w stylu TGramAI albo oficjalne API Reddita.

**Ryzyka:**
- Nieznana głębokość i kompletność archiwum X; dokumentacja tego nie podaje. Nie wolno deklarować pełnej historii bez testu.
- Dane pochodzą z sieci Bittensor; dostępność bywa zmienna (kody 503 `upstream_*`).
- Niepublikowane limity i niespójna cena profili X.
- Warunki korzystania z danych X przez zewnętrznego dostawcę: nie sprawdzono regulaminu Desearch.
- Częściowe dublowanie z TGramAI (X przez xAI Grok, Reddit przez `.json`) i ScraperX.

## Proponowany test (za 5 USD kredytu startowego)

Klucz API trzymać wyłącznie w `.env` na Proxmox, nie w repozytorium.

1. `GET /twitter` dla cashtagu znanego projektu z ostatnich 30 dni. Porównać wynik ze ScraperX: liczba postów, duplikaty, brakujące wpisy.
2. `from:<autor> since:2020-01-01 until:2020-12-31` dla jednego aktywnego konta. Sprawdzić, jak daleko wstecz sięga archiwum.
3. `GET /web` z `site:reddit.com <projekt>`, potem `GET /web/extract` na jednym wątku. Sprawdzić, czy przychodzą komentarze.
4. To samo wyszukiwanie przez Brave na darmowym kredycie. Porównać trafność.
5. Zapisać koszty z nagłówka `X-Desearch-Cost-Usd`.

Po teście decyzja do `decyzje.md`: czy dodać Desearch jako źródło X i jak połączyć go z narzędziami (serwer MCP w sesji agenta na Proxmox albo wywołania REST).

## Źródła

- Strona: https://www.desearch.ai/
- Cennik: https://www.desearch.ai/pricing
- Dokumentacja: https://desearch.ai/docs/guide/introduction/desearch-ai
- Limity i błędy: https://desearch.ai/docs/guide/introduction/rate-limit
- Operatory X: https://desearch.ai/docs/guide/apis/x-queries
- OpenAPI: https://api.desearch.ai/openapi.json
- Bittensor SN22: https://learnbittensor.org/subnets/datura-ai/desearch
- Brave Search API: https://brave.com/search/api/
- TGramAI `forum_scraper.py`: https://github.com/popek1990/tgramai-2026/blob/ea37d83/forum_scraper.py#L204

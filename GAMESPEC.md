# Cel i ogólny opis gry

- Gra typu city-builder osadzona w klimacie rzymskiej osady.
- Gracz zarządza jedną wioską: rozwija budynki, gromadzi surowce, rekrutuje wojska.
- Rozgrywka asynchroniczna w czasie rzeczywistym (produkcja zasobów, budowy, szkolenie jednostek).
- Brak mapy świata, walk, handlu i badań w wersji MVP.

## Podstawowe założenia projektowe

- Użytkownik: brak logowania/hasła; identyfikacja przez nazwę/e-mail lub wklejenie tokenu (Bearer).
- Przy utworzeniu użytkownika automatycznie powstaje jedna wioska (relacja docelowo 1:N).
- Wszystkie zasoby i systemy są przypisane do wioski (nie do gracza).
- Maksymalny poziom każdego budynku: 20.
- Surowce: drewno, kamień, żelazo oraz żywność (jako osobny zasób).
- Brak mapy, ataków, murów, handlu, badań, stajni i warsztatów w MVP.
- Minimalizm interfejsu: jeden główny widok wioski z akcjami rozwoju i rekrutacji.

## Główne mechaniki

### Mechanika rdzeniowa (core loop)

- Odbierz produkcję surowców (rośnie w czasie rzeczywistym).
- Rozbuduj forum, aby skracać czasy budów.
- Rozbuduj budynki surowcowe, aby zwiększyć produkcję.
- Rozbuduj spichlerz, aby zwiększyć pojemność zasobów.
- Rozbuduj koszary, aby skracać czas szkolenia i odblokowywać jednostki.
- Rekrutuj jednostki w koszarach, zużywając surowce i żywność.

### Progresja (poziomy, punkty, zasoby)

- Budynki (1–20): poziom zwiększa produkcję/pojemność/bonusy zgodnie z typem.
- Produkcja surowców:
    - Poziom 1: 10 jednostek/h dla każdego surowca.
    - Każdy kolejny poziom mnoży produkcję przez 1.40 (produktywność jest mnożona kaskadowo).
    - Produkcja zatrzymuje się po osiągnięciu limitu spichlerza.
- Czas rozbudowy budynków:
    - Bazowy czas: t(level→level+1) = 30s + (level-1)*20s.
    - Modyfikator forum: każdy poziom forum skraca czasy budów o 2% (multiplikatywnie).
- Koszary i rekrutacja:
    - Czas szkolenia jednostek zależy od typu i skraca się o 5% za każdy poziom koszar (multiplikatywnie).
    - Koszty rekrutacji są stałe (nie maleją z poziomem).
    - Odblokowania:
        - Auxilia (lekka piechota): dostępna od koszar 1.
        - Legionista (ciężka piechota): od koszar 6.
        - Łucznik: od koszar 10.
- Żywność i ludność:
    - Każda rekrutacja zużywa „1 żywność” (koszt jednorazowy przy szkoleniu).
    - W tej wersji MVP nie prowadzimy osobnego licznika ludności – jej rola jest odwzorowana przez żywność (upraszcza zapis z notatek: „ludność ~ dostępność żywności”).
    - Budowy nie wymagają ludności (uprośnienie na MVP, by uniknąć wprowadzania nowego zasobu „ludność”).

### Systemy pomocnicze

- Kolejka budów: w wiosce może być aktywna tylko jedna budowa/rozbudowa jednocześnie. Próba uruchomienia kolejnej zwraca błąd.
- Kolejka rekrutacji: koszary posiadają kolejkę; można zlecić ilość (quantity), pozycje wykonują się sekwencyjnie.
- Limity zasobów: spichlerz zwiększa pojemność wszystkich zasobów; po osiągnięciu limitu produkcja przestaje rosnąć.
- Tryb developerski (opcjonalny przełącznik): możliwość skracania czasów w celu testów (nie wpływa na logikę gry).

## Logika gry krok po kroku

### Stany gry

- Idle: brak aktywnych budów/rekrutacji; zasoby rosną do limitu.
- Budowa w toku: jedna aktywna budowa/rozbudowa, licznik czasu do zakończenia.
- Rekrutacja w toku: kolejka w koszarach, licznik czasu dla pierwszej pozycji.
- Zasoby pełne: produkcja zatrzymana do czasu wydania surowców lub zwiększenia pojemności.

### Interakcje gracza

- Wejście do gry:
    - Wpisanie identyfikatora lub tokenu -> załadowanie wioski.
- Rozbudowa budynku:
    - Wybór budynku -> przycisk „Rozbuduj”.
    - Warunki: brak aktywnej budowy, wystarczające surowce, budynek < 20 poziom.
    - Efekt: zdjęcie surowców, start timera; po zakończeniu poziom+1.
- Rekrutacja jednostek:
    - Wybór typu jednostki w koszarach (jeśli odblokowana) -> podanie quantity -> „Rekrutuj”.
    - Warunki: wystarczające surowce i żywność, koszary dostępne; pozycje trafiają do kolejki.
- Odbiór informacji:
    - Podgląd statusów: czasy do zakończenia budowy/rekrutacji, aktualne poziomy, produkcje/h, limity, aktualne zasoby.
- Wylogowanie:
    - Usunięcie tokenu z sesji (front) lub unieważnienie po stronie serwera.

### Warunki zwycięstwa/przegranej/zakończenia

- Brak warunków zwycięstwa/przegranej w MVP. Gra ma charakter ciągły (sandbox).

### Struktura danych i obiektów (wysoki poziom)

- User
    - id, displayName, token (opcjonalnie), lista wiosek (w MVP jedna)
- Village
    - id, ownerId, name (w MVP brak edycji), resources, storageCap, buildings[], army, queues
    - resources: {wood, stone, iron, food, updatedAt}
    - storageCap: maksymalna pojemność dla każdego surowca
- Building
    - type (forum, farma, tartak, kamieniołom, hutaZelaza, spichlerz, koszary)
    - level (1–20)
    - status: idle | upgrading (z endTime)
- Army
    - units: {auxilia, legionary, archer} – liczności aktualne
- Queues
    - buildQueue: maks. 1 pozycja {buildingType, finishAt}
    - recruitQueue: lista pozycji {unitType, countRemaining, perUnitTime, finishCurrentAt}
- UnitType (definicje stałe)
    - id, name, baseTrainTime, cost (wood/stone/iron/food), unlockBarracksLevel, role (light/heavy/archer)
- Balance/Formula (stałe)
    - productionBase=10/h, productionMultiplier=1.40/poziom
    - buildTime(level)=30s+(level-1)*20s; forumTimeReduction=2%/poziom
    - trainTimeReduction=5%/poziom koszar
    - storageProgression: patrz Ekonomia gry

## Interfejs użytkownika

### Widoki

- Ekran wejściowy:
    - Pole identyfikatora użytkownika lub tokenu; przycisk „Wejdź”.
- Widok wioski (główny):
    - Panel zasobów (aktualne ilości/limity + przewidywane przyrosty).
    - Lista budynków z poziomami, przyciskami „Rozbuduj” i statusami.
    - Panel koszar: typy jednostek (odblokowane/zablokowane), pola quantity, przycisk „Rekrutuj”, widok kolejki.
    - Statusy aktywności (budowa/rekrutacja – timery).
    - Przycisk wylogowania.

### Elementy interaktywne

- Rozbuduj (dla każdego budynku).
- Rekrutuj (dla każdego typu jednostki – z polem ilości).
- Anuluj (opcjonalnie tylko w dev – poza MVP produkcyjnym).
- Wyloguj (czyści token).

### Flow ekranów

- Ekran wejściowy -> Widok wioski.
- W widoku wioski: cykliczne odświeżanie danych (lub obliczenia po stronie frontu stosując updatedAt + stawki produkcji).
- Brak dodatkowych ekranów w MVP.

## Ekonomia gry

### Surowce

- Drewno, Kamień, Żelazo, Żywność. Wszystkie per wioska.
- Produkcja:
    - Tartak/Kamieniołom/Huta Żelaza: 10/h na poziomie 1; każdy poziom x1.40.
    - Far­ma (żywność): analogicznie jak surowce – 10/h na poziomie 1; każdy poziom x1.40.
- Limity:
    - Spichlerz zwiększa wspólną pojemność wszystkich zasobów.
    - Propozycja pojemności:
        - Poziom 1: 1 000 z każdego zasobu.
        - Każdy poziom spichlerza: +50% pojemności względem poprzedniego (mnożnik 1.5).
- Budowle (lista i funkcje):
    - Forum: skraca czasy budów innych budynków i siebie o 2%/poziom.
    - Far­ma: produkuje żywność.
    - Tartak: produkuje drewno.
    - Kamieniołom: produkuje kamień.
    - Huta żelaza: produkuje żelazo.
    - Spichlerz: zwiększa limity zasobów.
    - Koszary: skracają czas szkolenia o 5%/poziom; odblokowują jednostki.
- Jednostki (koszty i czasy bazowe, koszty stałe):
    - Auxilia (lekka):
        - Koszt: 3 drewna, 1 żelaza, 1 żywności.
        - Czas: 10 s (na koszarach 1; redukcja 5%/poziom koszar).
    - Legionista (ciężka):
        - Koszt: 3 drewna, 20 żelaza, 1 żywności.
        - Czas: 60 s (redukcja 5%/poziom koszar).
    - Łucznik:
        - Koszt: 30 drewna, 5 żelaza, 1 żywności.
        - Czas: 120 s (redukcja 5%/poziom koszar).
- Kontry jednostek (papier-kamień-nożyce):
    - Lekka piechota > Łucznicy
    - Łucznicy > Ciężka piechota
    - Ciężka piechota > Lekka piechota
    - (System walk nie jest w MVP; układ zależności definiuje równowagę na przyszłość.)

## Integracje techniczne / podstawowe wymagania technologiczne

- Sesja: token Bearer; wylogowanie przez skasowanie tokenu (front) lub unieważnienie (back).
- Zasoby naliczane w czasie rzeczywistym:
    - Serwer trzyma updatedAt i stawki; przy każdym żądaniu wylicza aktualny stan (z uwzględnieniem limitów).
- Endpoints (kierunkowo, REST):
    - GET /village/{id}/buildings -> lista budynków, poziomy, statusy (upgrading, finishAt), produkcje/h, limity.
    - POST /village/{id}/buildings/{buildingType}/upgrade -> zlecenie rozbudowy; 409 gdy budowa trwa; 400 gdy brak zasobów; 422 gdy max poziom.
    - GET /village/{id}/army -> stan armii, kolejka rekrutacji, odblokowania jednostek.
    - POST /village/{id}/army/{unitType}/recruit {quantity} -> dodanie do kolejki; 400/422 na błędach (brak zasobów, nieodblokowane).
    - GET /village/{id}/resources -> aktualny stan zasobów i limity (opcjonalnie, jeśli nie zawarte w buildings).
- Błędy:
    - 400 – nieprawidłowe dane/warunki (np. brak zasobów).
    - 409 – konflikt (już trwa budowa).
    - 422 – nie spełniono warunków (np. wymagany poziom koszar).
- Kolejki:
    - Budowa: 1 aktywna pozycja; finishAt.
    - Rekrutacja: FIFO; każda pozycja ma countRemaining, perUnitTime i finishCurrentAt.

# Podsumowanie kluczowych zasad gry

- Jedna wioska na użytkownika w MVP, wszystkie zasoby i systemy per wioska.
- Siedem budynków: forum, farma (żywność), tartak, kamieniołom, huta żelaza, spichlerz, koszary; każdy do poziomu 20.
- Produkcja surowców od poziomu 1: 10/h, mnożnik 1.40 na poziom; zatrzymanie przy pełnym spichlerzu.
- Rozbudowa budynków: bazowo 30s+(poziom-1)*20s; forum skraca czasy o 2%/poziom; jedna budowa naraz.
- Koszary skracają czasy szkolenia o 5%/poziom i odblokowują jednostki (Auxilia 1+, Legionista 6+, Łucznik 10+).
- Jednostki i koszty: Auxilia (3D/1Ż/1F, 10s), Legionista (3D/20Ż/1F, 60s), Łucznik (30D/5Ż/1F, 120s); koszty stałe; F=żywność.
- Zależności bojowe na przyszłość: lekka > łucznik > ciężka > lekka.
- Brak mapy, walk, handlu i badań w MVP.
- Wejście bez hasła; identyfikacja użytkownika lub token; proste REST API; statusy z finishAt.
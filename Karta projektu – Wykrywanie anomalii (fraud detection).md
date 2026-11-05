# Karta projektu – Wykrywanie anomalii w transakcjach kartowych

Oct 12, 2026 · @Test

## 1. Problem biznesowy

Projekt realizuje archetyp #8 „Wykrywanie anomalii” w kontekście wykrywania oszustw w transakcjach kartowych w domenie \[DOMENA: bank / płatności online / e-commerce\]. Transakcje oszukańcze stanowią niewielki ułamek ruchu, ale każda niewykryta generuje bezpośrednią stratę finansową. Ręczna weryfikacja wszystkich transakcji jest niemożliwa, dlatego potrzebny jest mechanizm wskazujący najbardziej podejrzane przypadki. Celem jest system, który porządkuje transakcje według ryzyka oszustwa, aby ograniczony zespół analityków sprawdzał najpierw te najbardziej prawdopodobne.

## 2. Użytkownik końcowy

Użytkownikiem jest analityk antyfraudowy, który decyduje, czy transakcję zatwierdzić, zablokować, czy eskalować (np. kontakt z klientem). Obecnie analityk przegląda kolejkę alertów generowanych przez proste reguły (np. wysoka kwota), co daje wiele fałszywych alarmów. Po wdrożeniu system przypisuje każdej transakcji wynik ryzyka i sortuje kolejkę, tak aby analityk w ramach dziennego limitu k przypadków sprawdzał najpierw te najbardziej podejrzane. System wspiera osąd analityka, ale go nie zastępuje — ostateczna decyzja zawsze należy do człowieka.

## 3. Dane

| Zbiór | Źródło | Rozmiar | Udział oszustw | Rola |
| --- | --- | --- | --- | --- |
| IEEE-CIS Fraud Detection (Vesta Corporation) | Kaggle: <https://www.kaggle.com/competitions/ieee-fraud-detection/data> | \~590 540 transakcji | \~3,5% | główny |
| ULB Credit Card Fraud Detection | Kaggle / OpenML: <https://www.kaggle.com/datasets/mlg-ulb/creditcardfraud> | 284 807 transakcji, 492 oszustwa | 0,172% | sprawdzenie generalizacji |

**IEEE-CIS — struktura.** Dane składają się z dwóch tabel łączonych po kluczu `TransactionID`: `train_transaction` (394 kolumny, wszystkie \~590 540 transakcji) oraz `train_identity` (41 kolumn, ok. 144 tys. transakcji — informacje o urządzeniu i połączeniu są dostępne tylko dla części ruchu). Etykieta `isFraud` przyjmuje wartość 1 dla oszustwa i 0 dla transakcji legalnej. Dostępny zbiór testowy Kaggle nie ma etykiet, dlatego wszystkie oceny wykonujemy na danych treningowych.

| Grupa | Kolumny | Co zawierają |
| --- | --- | --- |
| Czas i kwota | TransactionDT, TransactionAmt | czas w sekundach od nieujawnionej daty odniesienia (ok. 6 miesięcy ruchu); kwota w USD |
| Produkt | ProductCD | kod kategorii produktu/usługi (5 wartości: W, C, R, H, S) |
| Karta | card1–card6 | zanonimizowane cechy karty, m.in. sieć (Visa, Mastercard…), typ (debetowa/kredytowa), kody wydawcy |
| Adres i odległość | addr1–addr2, dist1–dist2 | region i kraj rozliczeniowy; odległości między adresami |
| E-mail | P\_emaildomain, R\_emaildomain | domena e-mail płacącego i odbiorcy |
| Liczniki | C1–C14 | zliczenia powiązań, np. ile adresów lub urządzeń łączy się z kartą (znaczenie zamaskowane) |
| Odstępy czasowe | D1–D15 | liczba dni od wcześniejszych zdarzeń, np. poprzedniej transakcji kartą |
| Zgodność | M1–M9 | flagi T/F zgodności danych, np. imienia na karcie z adresem |
| Cechy Vesta | V1–V339 | cechy wyliczone przez Vesta (rankingi, zliczenia, relacje między encjami); znaczenie niejawne |
| Tożsamość (opcjonalnie) | DeviceType, DeviceInfo, id\_01–id\_38 | typ i model urządzenia, przeglądarka, system, sieć, wyniki weryfikacji tożsamości |

## 4. Technika SI

Porównamy podejście nienadzorowane (Isolation Forest, odległość od centroidu) z klasyfikatorem nadzorowanym (Random Forest / drzewo decyzyjne); w wariancie na najwyższą ocenę dodatkowo autoenkoder MLP. Metody nienadzorowane nie wymagają etykiet i mogą wychwycić nowe wzorce oszustw, ale nie wiedzą, które anomalie są faktycznie oszustwami. Klasyfikator nadzorowany wykorzystuje etykiety `isFraud`, lecz uczy się tylko znanych schematów. Porównanie pokaże, ile zyskujemy dzięki etykietom i czy wykrywanie anomalii jest realną alternatywą, gdy etykiety są niepełne lub opóźnione.

## 5. Metryka sukcesu

- **PR AUC** — główna metryka, odporna na silną nierównowagę klas.
- **Recall** — jaki odsetek oszustw zostaje wykryty.
- **Precision@k** — jaki odsetek z k transakcji, które analityk zdąży ręcznie sprawdzić, to faktyczne oszustwa.

Accuracy odrzucamy: przy \~3,5% oszustw (IEEE-CIS) model oznaczający wszystko jako legalne osiąga \~96,5% accuracy, nie wykrywając żadnego oszustwa. Sukcesem będzie wyraźna poprawa PR AUC i precision@k względem baseline; konkretny próg \[PRÓG\] ustalimy po zmierzeniu baseline.

## 6. Baseline

- **DummyClassifier (most\_frequent):** zawsze przewiduje klasę „legalna”, więc spodziewamy się wysokiego accuracy, ale recall = 0 i PR AUC na poziomie udziału oszustw (\~0,035).
- **Reguła progowa na TransactionAmt:** oznacza transakcje powyżej progu kwoty; spodziewamy się niewielkiego recall przy niskiej precyzji, co odzwierciedla obecne reguły.

Modele SI muszą wyraźnie pokonać oba punkty odniesienia.

## 7. Walidacja

1. Stratyfikowany podział train/test, zachowujący udział oszustw w obu częściach.
2. Stratyfikowana walidacja krzyżowa — raportujemy rozrzut wyników (średnia ± odchylenie), a nie pojedynczą liczbę.
3. Podział czasowy wg TransactionDT: trening na wcześniejszych transakcjach, test na późniejszych, aby sprawdzić dryf danych.
4. Analiza błędów: co najmniej 5 źle sklasyfikowanych przypadków, obejmujących zarówno przeoczone oszustwa, jak i fałszywe alarmy.
5. Sprawdzenie generalizacji na zbiorze ULB.

## 8. Ryzyka i klasyfikacja AI Act

Wstępna klasyfikacja: **ryzyko minimalne**. Systemy wykrywania oszustw finansowych są wprost wyłączone z kategorii wysokiego ryzyka w Załączniku III AI Act, a nasz system jedynie szereguje transakcje do ręcznej weryfikacji i nie podejmuje autonomicznych decyzji.

- **Możliwe szkody:** przeoczone oszustwo (strata finansowa) oraz fałszywy alarm (niesłuszne zablokowanie legalnego klienta).
- **Ograniczenia:** dryf danych (wzorce oszustw zmieniają się w czasie) oraz brak interpretowalności cech V1–V339 / V1–V28.
- **Nadzór człowieka:** każdą decyzję o blokadzie podejmuje analityk; model dostarcza wyłącznie wynik ryzyka i kolejność w kolejce.

## 9. Role w zespole

| Rola | Osoba | Odpowiedzialność |
| --- | --- | --- |
| Lider / Product Owner | Artem Kuchai | definicja problemu i użytkownika, karta projektu, harmonogram |
| Inżynier AI/ML | Diana Starchuk | implementacja i strojenie modeli (Isolation Forest, Random Forest, autoenkoder MLP) |
| Inżynier danych i walidacji | Artemii Zubriichuk | ładowanie i przygotowanie danych, baseline, schemat walidacji, analiza błędów |

W 3-osobowym zespole role się przenikają — każda osoba wspiera pozostałe obszary.

# MECHANIZM_PL — jak to działa i jak o tym mówić

> Materiał do nauki. Firma Nordvale Pharma, leki Velquanor, Orvatil i Brenquilo oraz wszystkie dane są zmyślone. To nie jest porada prawna ani medyczna.

## 1. O co chodzi w tym projekcie (w dwóch zdaniach)

Dział informacji medycznej firmy farmaceutycznej dostaje pytania od lekarzy, farmaceutów i pacjentów. Zbudowałem system, który te pytania sortuje, **od razu** wysyła do działu bezpieczeństwa leków wszystko, co może być działaniem niepożądanym, a dla zwykłych pytań przygotowuje szkic odpowiedzi **wyłącznie z zatwierdzonych tekstów** — i zawsze to człowiek decyduje i wysyła.

---

## 2. Jak wygląda wdrożenie — od analizy do przekazania

Wdrożenie to nie tylko „zbudować automatyzację”. To pięć kroków, a każdy kończy się dokumentem, który ktoś po stronie klienta czyta i podpisuje.

| Etap | Dokument | Po co jest | Kto go czyta u klienta |
|---|---|---|---|
| 1. Analiza (discovery) | `docs/01_discovery.md` | Opisuje, jak jest dziś, gdzie boli i **po czym poznamy sukces**. Bez tego nie da się potem udowodnić, że wdrożenie coś dało | Szef działu, właściciel procesu |
| 2. Projekt rozwiązania | `docs/02_solution_design.md` | Pokazuje, jak będzie: diagram, kto o czym decyduje, co robi AI, a czego **nie** robi. Tu zapadają decyzje projektowe | Szef działu, IT, bezpieczeństwo leków |
| 3. Budowa | `workflow/` | Sama automatyzacja w n8n | IT |
| 4. Test | `results/README.md` | Dowód liczbami, że działa — i uczciwy opis każdego błędu | Wszyscy przed decyzją „startujemy” |
| 5. Dokumenty wdrożeniowe | `docs/03`–`06` | Ryzyka, zgodność z prawem, instrukcja dla ludzi, szkolenie i przekazanie | Inspektor ochrony danych (DPO), prawnicy, użytkownicy, zespół wsparcia |

**Ważna myśl na rozmowę:** w branży regulowanej (farmacja) klient nie kupi „sprytnego bota”. Kupi rozwiązanie, które ma udokumentowane ryzyka, test, ślad audytowy i plan szkolenia. Dlatego połowa tego projektu to dokumenty.

---

## 3. Pojęcia, które musisz rozumieć

**Informacja medyczna (Medical Information, MI)** — dział firmy farmaceutycznej, który odpowiada na pytania o jej leki: jak przechowywać, co zawiera, jak przyjmować. Odpowiada na podstawie **zatwierdzonych standardowych odpowiedzi** (standard response documents), a nie „z głowy”.

**Działanie niepożądane (adverse event, AE)** — każde niechciane zdarzenie medyczne u osoby, która użyła leku: wysypka, ból głowy, wymioty, pobyt w szpitalu. Nie trzeba udowadniać, że to *przez* lek — wystarczy, że wystąpiło po nim. Często jest ukryte w innym pytaniu: „czy mama może do tego wziąć coś przeciwbólowego? Po tabletkach miała wysypkę, ale już przeszła”.

**Sytuacje szczególne (special situations)** — rzeczy, które dział bezpieczeństwa leków też musi zobaczyć, nawet bez objawów: ciąża lub karmienie piersią w trakcie leczenia, przedawkowanie, pomyłka (zły lek, zła dawka), dziecko, które przypadkiem połknęło lek, „lek przestał działać”.

**Bezpieczeństwo farmakoterapii (pharmacovigilance, PV, „drug safety”)** — system nadzoru nad bezpieczeństwem leków po wprowadzeniu na rynek. Firma musi zbierać zgłoszenia działań niepożądanych i przekazywać je do urzędów w określonym czasie: **poważne w 15 dni, pozostałe w 90 dni**. Zegar startuje w chwili, gdy **ktokolwiek w firmie** dostał informację — także dział informacji medycznej. Dlatego zgłoszenie, które leży w zwykłej kolejce, to realny problem.

**Pytanie poza wskazaniami (off-label)** — pytanie o użycie leku inaczej, niż mówi jego oficjalna ulotka i charakterystyka: u dzieci, gdy lek jest dla dorosłych; inna dawka; rozkruszanie tabletki; inna choroba. Na takie pytania może odpowiadać tylko wykwalifikowany specjalista, ostrożnie i bez promowania. AI nie robi tu szkicu nigdy.

**Reklamacja jakościowa (product complaint)** — wada produktu lub opakowania: puste miejsca w blistrze, pęknięta nakrętka, dziwny zapach. Trafia do działu jakości. Jeśli ktoś przy tym zachorował — najpierw bezpieczeństwo leków.

---

## 4. Workflow krok po kroku — każdy węzeł

### 4.1 PIC MI Triage (główny)

| # | Węzeł | Co robi, prostym językiem |
|---|---|---|
| 1 | **Enquiry form** | Formularz dla pytającego: kim jest, który lek, pytanie, imię, e-mail, telefon |
| 2 | **When called by test** | Drugie „wejście” — przez nie test wpuszcza 26 zapytań. Dzięki temu test sprawdza ten sam system, którego używają ludzie |
| 3 | **Normalize enquiry** | Ujednolica dane z obu wejść i nadaje numer sprawy (`MI-…` albo `TEST-…`) oraz czas wpływu — to jest „dzień zero” dla bezpieczeństwa leków |
| 4 | **Save intake (contact details)** | Zapisuje dane kontaktowe w **jednej** tabeli (`PIC Intake`). Nigdzie indziej ich nie ma |
| 5 | **Mask personal data** | Zamienia w treści pytania: e-maile → `[EMAIL]`, telefony → `[PHONE]`, numery pacjenta → `[ID_NUMBER]`, datę urodzenia → `[DATE_OF_BIRTH]`, imiona i nazwiska → `[NAME]` (np. po słowach „Dr”, „Patient”, „My name is”, „Thanks,”). Robi to zwykły kod, **przed** AI |
| 6 | **Check masking** | Drugi przegląd tekstu po maskowaniu. Jeśli coś wciąż wygląda na dane osobowe, AI **w ogóle nie dostaje** tekstu |
| 7 | **AE keyword rules** | Lista słów i zwrotów (wysypka, wymioty, „by mistake”, „pregnant”, „doesn't work”…) — wykrywa możliwe działania niepożądane **bez AI**. Działa nawet wtedy, gdy AI jest niedostępne |
| 8 | **Clean for AI?** | Rozwidlenie: tekst czysty → idzie do AI; nie → omija AI i trafia do człowieka |
| 9 | **Get response library** | Pobiera bibliotekę zatwierdzonych odpowiedzi (każda z numerem i wersją) |
| 10 | **Build AI request** | Składa wiadomość dla Claude'a: zamaskowany tekst + **lista numerów i tematów** zatwierdzonych odpowiedzi dla tego leku. Bez treści odpowiedzi, bez danych kontaktowych |
| 11 | **Classify with Claude** | Claude zwraca JSON: kategoria (informacja o produkcie / działanie niepożądane / reklamacja / poza wskazaniami / inne), czy podejrzewa działanie niepożądane, numer pasującej odpowiedzi albo `NONE`, krótkie uzasadnienie. Ponawia 3 razy przy błędzie; jeśli dalej błąd — nie zatrzymuje systemu, tylko oddaje sprawę człowiekowi |
| 12 | **Route enquiry** | Mózg systemu, w kodzie. Kolejność decyzji: (1) dane osobowe nie przeszły kontroli → człowiek; (2) reguły **albo** AI widzą działanie niepożądane → bezpieczeństwo leków, bez szkicu; (3) AI nie odpowiedziało poprawnie → człowiek; (4) reklamacja → jakość; (5) poza wskazaniami → starszy specjalista; (6) informacja o produkcie z pasującą odpowiedzią → **szkic złożony słowo w słowo z biblioteki** + `[Source: ID vX]`; (7) reszta → człowiek. Potem trzy kontrole: numer źródła istnieje i jest zatwierdzony, w tekście dla AI nie było danych osobowych, w szkicu nie ma zakazanych zwrotów. Jedna nie przejdzie → szkic wyrzucony, sprawa do człowieka |
| 13 | **Save to queue** | Zapis sprawy do kolejki (`PIC Queue`) — z kategorią, powodem, szkicem i źródłem |
| 14 | **Write audit log** | Wpis do dziennika: kto (system), kiedy, jaka decyzja, jakie źródło i wersja, wersje reguł i promptu |
| 15 | **Return result** | Oddaje wynik do testu. Nic nie wysyła na zewnątrz |

### 4.2 PIC Draft Review (przegląd przez człowieka)

| Węzeł | Co robi |
|---|---|
| **Open draft review** | Formularz wymagający **zalogowania do n8n** — dzięki temu wiemy, kto podjął decyzję |
| **Get drafts to review → Build review page** | Pobiera szkice czekające na przegląd i buduje dla każdego kartę: zamaskowane pytanie, źródło, wynik kontroli, szkic, wybór decyzji, pole do poprawki, notatka |
| **Review drafts** | Strona formularza, na której człowiek wybiera: zatwierdź / odrzuć / popraw / zostaw |
| **When called by test** | Wejście dla testu (te same decyzje, bez klikania) |
| **Collect decisions** | Zbiera decyzje z formularza albo z testu w jedną listę i zapisuje, kto decyduje |
| **Load current drafts → Check decisions** | Sprawdza każdą decyzję: odrzucenie wymaga powodu; poprawka musi zostawić linię `[Source: …]`, nie może mieć zakazanych zwrotów ani danych osobowych. Zła decyzja = zablokowana |
| **Decision valid? → Update queue row** | Tylko poprawne decyzje zmieniają status sprawy (`approved`, `rejected`) |
| **Write review to audit log** | **Każda** decyzja — także zablokowana — trafia do dziennika |
| **Summarize decisions → Decisions saved / Return to test** | Podsumowanie dla człowieka albo dla testu. „Zatwierdzone” znaczy „gotowe, żeby **człowiek** wysłał” |

### 4.3 PIC Eval (test)

Pobiera 26 zmyślonych zapytań → puszcza każde przez prawdziwy system → porównuje z oczekiwanym wynikiem → zapisuje wynik każdego przypadku (`PIC Test Results`) → wysyła 5 decyzji do formularza przeglądu (w tym 2, które muszą zostać zablokowane) → sprawdza dziennik audytowy → zapisuje jeden wiersz podsumowania (`PIC Test Runs`).

---

## 5. Co wyszło w teście i czego to uczy

- Były **trzy przebiegi testu**. W każdym **7 z 7** działań niepożądanych trafiło do bezpieczeństwa leków — i to **każdym z dwóch detektorów osobno**. Trzeci przebieg: 26 z 26 przypadków bez żadnego błędu.
- **Test 1:** przy reklamacji Claude odpowiedział dobrze, ale dwa razy w jednej wiadomości. Kod tego nie przyjął i oddał sprawę człowiekowi. Lekcja: **nawet poprawna odpowiedź AI może mieć zły format** — dlatego zawsze jest bezpieczna ścieżka awaryjna. Poprawiłem kod i powtórzyłem test.
- **Test 2:** pytanie „czy mogę wziąć podwójną dawkę?” AI uznało za możliwy problem bezpieczeństwa. Fałszywy alarm, ale w bezpieczną stronę. Lekcja: **w bezpieczeństwie lepiej fałszywy alarm niż przeoczenie** — i model nie odpowiada zawsze identycznie (w przebiegach 1 i 3 tego samego pytania nie oznaczył), dlatego testy trzeba powtarzać.
- **Przed testem 3** zmieniłem jedno zapytanie testowe, bo zawierało nazwę prawdziwej substancji leczniczej — a zasada projektu to zero prawdziwych leków. Dlatego powtórzyłem test na poprawionych danych, zamiast tylko podmienić tekst.
- Uczciwe zastrzeżenie: reguły i przypadki testowe pisała ta sama osoba, więc wynik reguł jest optymistyczny. W prawdziwym projekcie sprawdza się je na historycznych, zanonimizowanych zgłoszeniach.

---

## 6. RODO prostym językiem

- **Kto za co odpowiada:** firma (Nordvale) decyduje o danych — to **administrator**. Dostawca AI i n8n przetwarzają dane na jej polecenie — to **podmioty przetwarzające**; muszą mieć z firmą umowę powierzenia (art. 28).
- **Dane o zdrowiu są szczególnie chronione** (art. 9). Można je przetwarzać tylko z konkretnego powodu z listy — np. ze względu na interes publiczny w zakresie bezpieczeństwa leków (art. 9 ust. 2 lit. i).
- **Podstawa prawna:** przekazanie działania niepożądanego do bezpieczeństwa leków to **obowiązek prawny** firmy. Odpowiadanie na zwykłe pytania — do decyzji inspektora ochrony danych.
- **Minimalizacja danych** (art. 5 i 25): przetwarzaj tylko to, co potrzebne. U nas: AI nie widzi imion, telefonów ani e-maili; dane kontaktowe są w jednej tabeli.
- **Ocena skutków (DPIA, art. 35):** gdy przetwarzasz dużo danych o zdrowiu, trzeba wcześniej opisać ryzyka i zabezpieczenia. Przygotowałem zarys.
- **Retencja (art. 5):** dane trzyma się tylko tak długo, jak trzeba. Dokumentacja bezpieczeństwa leków ma własne, dłuższe zasady.
- **Transfer poza UE (art. 44–46):** jeśli dostawca AI przetwarza dane np. w USA, potrzebna jest podstawa — decyzja o odpowiednim poziomie ochrony albo standardowe klauzule umowne.
- **Decyzje automatyczne (art. 22):** człowiek ma prawo nie podlegać decyzji podjętej *wyłącznie* przez maszynę. U nas zawsze decyduje człowiek.

## 7. AI Act prostym językiem

- AI Act dzieli systemy AI według ryzyka: zakazane, **wysokiego ryzyka** (np. ocena kredytowa, rekrutacja, triaż pacjentów w nagłych przypadkach, wyroby medyczne), systemy z obowiązkiem przejrzystości (np. chatboty) i cała reszta.
- **Nasz system nie jest wysokiego ryzyka:** nie stawia diagnozy ani nie decyduje o leczeniu (więc nie jest wyrobem medycznym), nie jest triażem pacjentów w nagłych przypadkach, nie decyduje o dostępie do usług. Sortuje korespondencję i kopiuje zatwierdzone teksty.
- **Obowiązek przejrzystości** dotyczy systemów, które rozmawiają z ludźmi bezpośrednio. U nas pytający nie rozmawia z AI — dostaje odpowiedź od człowieka. Mimo to zalecam informację w polityce prywatności.
- **Co obowiązuje na pewno:** firma ma wspierać **kompetencje AI** pracowników (art. 4 — w lipcu 2026 r. przepis zmieniono, teraz chodzi o „wspieranie” wiedzy o AI). Stąd szkolenie w dokumencie 06.
- **Mimo że nie musimy**, stosujemy dobre praktyki z wymogów dla wysokiego ryzyka: nadzór człowieka, dziennik zdarzeń, testy przed każdą zmianą. Bo ryzyko biznesowe (przeoczone zgłoszenie) jest wysokie.
- Terminy: obowiązki dla systemów wysokiego ryzyka przesunięto nowelizacją z 2026 r. (na grudzień 2027 i sierpień 2028).

---

## 8. Słowniczek

| Polski | Angielski | Znaczenie |
|---|---|---|
| informacja medyczna | Medical Information (MI) | dział odpowiadający na pytania o leki |
| działanie niepożądane | adverse event (AE) | niechciane zdarzenie medyczne po leku |
| podejrzewane działanie niepożądane | suspected adverse reaction | AE, przy którym podejrzewa się związek z lekiem |
| sytuacje szczególne | special situations | ciąża, przedawkowanie, pomyłka, brak działania itp. |
| bezpieczeństwo farmakoterapii | pharmacovigilance (PV), drug safety | nadzór nad bezpieczeństwem leków |
| zgłoszenie spontaniczne | spontaneous report | zgłoszenie, o które nikt nie prosił — także przez dział informacji medycznej |
| dzień zero | day zero | chwila, gdy firma dostała minimum informacji — start zegara |
| pytanie poza wskazaniami | off-label question | użycie inne niż w zatwierdzonej charakterystyce leku |
| charakterystyka produktu leczniczego | Summary of Product Characteristics (SmPC) | oficjalny opis leku dla fachowców |
| reklamacja jakościowa | product quality complaint | wada produktu lub opakowania |
| standardowa odpowiedź | standard response | zatwierdzony tekst odpowiedzi z numerem i wersją |
| maskowanie danych | data masking | zamiana danych osobowych na znaczniki |
| minimalizacja danych | data minimisation | przetwarzanie tylko niezbędnych danych |
| administrator / podmiot przetwarzający | controller / processor | role z RODO |
| ocena skutków dla ochrony danych | DPIA | analiza ryzyk przed przetwarzaniem |
| człowiek w pętli | human in the loop | człowiek zatwierdza decyzje systemu |
| dziennik audytowy | audit log | zapis kto, co, kiedy, na jakiej podstawie |
| bezpieczna ścieżka awaryjna | fail-safe | przy błędzie sprawa idzie do człowieka |
| okres wzmożonego wsparcia | hypercare | pierwsze tygodnie po starcie z codzienną kontrolą |
| przekazanie | handover | oddanie systemu zespołowi wsparcia |
| rejestr ryzyk | risk register | lista ryzyk z oceną i zabezpieczeniami |

---

## 9. Jak to opowiedzieć w 60 sekund (po angielsku)

> "I built a full implementation case for a fictional pharma company's Medical Information team. The problem: enquiries from doctors and patients were triaged by reading, so possible adverse events — like 'my mother got a rash after the tablets' — could sit in the normal queue, while safety reporting deadlines were already running.
>
> I started with discovery: the current process, roles, baseline metrics and success criteria. Then I designed the target process and built it in n8n. Personal data is masked in code before anything reaches the AI. Two independent checks — keyword rules and Claude — look for adverse events, and either one sends the case straight to Drug Safety with no draft. For ordinary questions, Claude only picks the ID of an approved response; the draft is copied word for word from the library, checked in code, and a person approves it. Every decision goes into an audit log.
>
> I tested it on 26 fictional enquiries in three runs: all 7 adverse events were caught every time, personal data was masked 10 out of 10, and every draft cited a valid approved source. I documented the two errors I found and fixed one. I also wrote the risk register, GDPR and AI Act assessment, user guide and handover plan."

---

## 10. Pięć pytań rekrutera z odpowiedziami

**1. "Why didn't you let the AI just write the answers? It would be faster."**
> "In pharma, the risk isn't speed, it's saying something that isn't approved. If the model writes free text, you can never be sure it didn't invent a dose or a claim. So I limited the AI to choosing the ID of an approved response, and the code copies the approved text. The AI does the sorting, which is where the time goes; the content stays controlled. It also makes every reply traceable to a source and version."

**2. "How do you know you didn't miss any adverse events?"**
> "On the test set, none were missed: 7 out of 7 in all three runs, and each detector caught all 7 on its own. But I'm careful with that number — the rules and the test cases were written by the same person. Before going live I'd ask Drug Safety to validate the rules on de-identified historical enquiries, and during hypercare Drug Safety would review a sample of non-flagged enquiries every day to look for misses."

**3. "What happens when the AI fails or gives a strange answer?"**
> "It fails safe. That actually happened in my first test run: Claude classified a complaint correctly but returned two JSON blocks, so the code rejected the reply and the enquiry went to a human queue. Nothing was lost. I fixed the parser so it accepts repeated answers only if they agree, and any safety flag always wins, then re-ran the whole test."

**4. "How did you handle GDPR?"**
> "Data minimisation by design. Names, emails, phone numbers, IDs and dates of birth are masked in code before the AI step, and a second check blocks the AI call if anything is left. Contact details live in one table only. In the compliance document I prepared the questions for the DPO: legal basis, a DPIA outline, retention, the processor contract with the AI provider and international transfers. I'm not a lawyer, so I framed it as decisions for the client, with links to the official texts."

**5. "Is this system high-risk under the EU AI Act?"**
> "Based on my reading, no: it doesn't diagnose or decide treatment, so it isn't a medical device, and it isn't emergency patient triage or any other Annex III use. And requesters never talk to the AI directly. But I still applied the core high-risk practices voluntarily — human oversight, logging and testing before every change — because a missed adverse event is a serious business risk. The final classification is for the client's legal team to confirm."

> Przed rozmową przećwicz te odpowiedzi na głos i połącz je z własnymi przykładami z pracy.

---

## 11. Mini-test (10 pytań)

1. **Pacjentka pyta, czy może brać Orvatil dwa razy dziennie, i dodaje, że skóra ją piecze. Do której kolejki trafi i dlaczego?**
   → Do bezpieczeństwa leków (`drug_safety`), bez szkicu. Pieczenie skóry to możliwe działanie niepożądane; bezpieczeństwo ma pierwszeństwo przed pytaniem o dawkowanie.

2. **Co dokładnie dostaje Claude?**
   → Typ pytającego, nazwę leku, zamaskowany tekst pytania i listę numerów oraz tematów zatwierdzonych odpowiedzi. Nie dostaje danych kontaktowych ani treści odpowiedzi.

3. **Dlaczego są dwa detektory działań niepożądanych, a nie jeden?**
   → Bo przeoczenie jest najgorszym błędem. Reguły działają bez AI (także gdy AI padnie), AI łapie zdania, których reguły nie znają. Wystarczy jeden sygnał.

4. **Czym jest „dzień zero” i czemu jest ważny dla tego systemu?**
   → To chwila, gdy ktokolwiek w firmie dostał minimum informacji o działaniu niepożądanym. Od niej liczą się terminy zgłoszenia (15/90 dni). Dlatego zgłoszenie musi trafić do bezpieczeństwa leków od razu.

5. **Co się stanie, jeśli AI wskaże numer odpowiedzi, który nie istnieje albo jest wycofany?**
   → Kontrola cytowania w kodzie go odrzuci, szkic nie powstanie, sprawa trafi do człowieka z powodem `citation_check_failed`.

6. **Specjalista poprawia szkic i dopisuje „you should stop taking it”. Co się stanie?**
   → Poprawka zostanie zablokowana (zakazany zwrot), szkic zostaje w kolejce, a próba trafia do dziennika audytowego.

7. **Czym jest pytanie poza wskazaniami? Podaj przykład z testu.**
   → Pytanie o użycie niezgodne z zatwierdzoną charakterystyką leku, np. Velquanor u 6-letniego dziecka albo rozkruszanie tabletek. Zawsze do starszego specjalisty, bez szkicu.

8. **Dlaczego w teście 2 był jeden fałszywy alarm i czemu go nie „naprawiłeś”?**
   → AI uznało pytanie o podwójną dawkę za możliwy problem bezpieczeństwa. Zmniejszenie czułości mogłoby spowodować przeoczenia; fałszywy alarm kosztuje kilka minut pracy działu bezpieczeństwa.

9. **Jaka jest różnica między administratorem a podmiotem przetwarzającym w tym projekcie?**
   → Administrator to Nordvale (decyduje o celu i sposobie). Podmioty przetwarzające to dostawca AI i n8n (działają na polecenie firmy, na podstawie umowy).

10. **Po co jest okres wzmożonego wsparcia (hypercare) i co się w nim robi?**
    → To pierwsze tygodnie po starcie. Codziennie sprawdza się kolejki, każdy powód przekazania do człowieka i próbkę zgłoszeń niezaznaczonych jako działania niepożądane, żeby szybko wyłapać błędy, zanim system przejdzie do zwykłego wsparcia.

---
title: Jak jest zbudowany system plików FAT
---

# Jak jest zbudowany system plików FAT

W tym wpisie podstawowe zasady działania systemu plików z rodziny FAT. Jeśli jesteś ciekawy co się dzieje "pod maską", gdy dodajesz lub usuwasz plik na nośniku danych z systemem plików FAT, to zachęcam do lektury tego wpisu. Na przykładach i w praktyce pokażę jakie zmiany zachodzą na dysku podczas manipulacji plikami.

Z tego wpisu dowiesz się:

- Jakie informacje można znaleźć w *Boot Sector*
- Czym jest tablica alokacji (FAT)
- Jak reprezentowany jest plik na dysku z systemem FAT
- Jak przywrócić plik po usunięciu go z dysku z systemem FAT

## Czym jest system plików
**System plików** jest to metoda przechowywania i zarządzania danymi na nośniku pamięci. Najmniejszą porcję danych użytkownika w systemie plików najczęściej stanowi plik. Do opisania budowy FAT potrzebne nam będą bardziej szczegółowe pojęcia: *bajt*, *sektor* i *klaster*. Definicja ta nie jest szczególnie istotna i przydatna w życiu. Jeśli potrzebujesz to wiedzieć do kolokwium, to zajrzyj do [Wikipedii](https://pl.wikipedia.org/wiki/System_plik%C3%B3w).

Po przeczytaniu tego wpisu *wyczujesz* czym jest system plików i będziesz się czuł pewnie, aby opowiedzieć co taki system robi.

## Budowa FAT
Zacznijmy od pokazania jak taki system plików wygląda. Wszystkie czynności tutaj przedstawione wykonuję na komputerze z systemem Linux Mint.

### Utworzenie czystego systemu plików FAT
Będziemy potrzebowali urządzenia do testów. Takiego, na którym będziemy mogli wykonywać normalne operacje na plikach (utwórz, kopiuj, usuń). Niezbędna jest również możliwość niskopoziomowego podglądu jak dane są zapisane na nośniku (bajt po bajcie).

Na szczęście nie będziemy używać fizycznego urządzenia. Stworzymy plik, który następnie zamontujemy w systemie jako wirtualny "nośnik danych". Można sobie to zestawić z analogią do fizycznego pendrive'a. Plik to nasz pendrive, a montowanie pliku w systemie, to po prostu wpięcie pendriva do komputera (coś jak plik z formatem *\*.iso*).

Najpierw utwórzmy pusty plik o rozmiarze, powiedzmy 100kB. Rozmiar pliku w tym przypadku będzie oznaczał pojemność naszego wirtualnego urządzenia. 
**Uwaga** plikowi nie można tak po prostu nadać rozmiaru. Trzeba go czymś wypełnić. Tworzę więc plik o nazwie `{fat.img}` i wypełniam go bajtami o wartości zero (nie mylić z wypełnianiem pliku znakami cyfry '0')

```bash
$ dd if=/dev/zero of=fat.img bs=1024 count=100
```

Utworzyliśmy w ten sposób plik o rozmiarze 100kB. Dane tego pliku składają się z samych zer. Jest to plik binarny, więc wyświetlanie jego zawartości programami typu `less` czy `cat` nie jest dobrym pomysłem.

Zobaczmy jak wygląda
```bash
$ hexdump -C fat.img

00000000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00019000
```

Widać, że plik jest wypełniony zerami. Program hexdump inteligentnie pokazuje tylko pierwsze 16 bajtów, po których wstawia znak `*` oznaczający, że ta sekwencja się powtarza aż do offsetu `00019000`, czyli do samego końca pliku (19000<sub>16</sub> = 102400<sub>10</sub>)

Stwórzmy w końcu system plików w naszym pliku.

```bash
$ mkfs.msods fat.img

mkfs.fat 4.1 (2017-01-24)
```

Dlaczego używam tutaj prehistorycznego FAT12? Ponieważ jest najprostszy w budowie i pozwoli w prosty sposób pokazać omawiane zagadnienia. Późniejsza analiza, np. FAT32 będzie o wiele łatwiejsza znając już podstawy.

Zerknijmy teraz jak wygląda nasz plik w środku.

```bash
$ hexdump -C fat.img
00000000  eb 3c 90 6d 6b 66 73 2e  66 61 74 00 02 04 01 00  |.<.mkfs.fat.....|
00000010  02 00 02 c8 00 f8 01 00  20 00 40 00 00 00 00 00  |........ .@.....|
00000020  00 00 00 00 80 00 29 9c  38 71 69 4e 4f 20 4e 41  |......).8qiNO NA|
00000030  4d 45 20 20 20 20 46 41  54 31 32 20 20 20 0e 1f  |ME    FAT12   ..|
00000040  be 5b 7c ac 22 c0 74 0b  56 b4 0e bb 07 00 cd 10  |.[|.".t.V.......|
00000050  5e eb f0 32 e4 cd 16 cd  19 eb fe 54 68 69 73 20  |^..2.......This |
00000060  69 73 20 6e 6f 74 20 61  20 62 6f 6f 74 61 62 6c  |is not a bootabl|
00000070  65 20 64 69 73 6b 2e 20  20 50 6c 65 61 73 65 20  |e disk.  Please |
00000080  69 6e 73 65 72 74 20 61  20 62 6f 6f 74 61 62 6c  |insert a bootabl|
00000090  65 20 66 6c 6f 70 70 79  20 61 6e 64 0d 0a 70 72  |e floppy and..pr|
000000a0  65 73 73 20 61 6e 79 20  6b 65 79 20 74 6f 20 74  |ess any key to t|
000000b0  72 79 20 61 67 61 69 6e  20 2e 2e 2e 20 0d 0a 00  |ry again ... ...|
000000c0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
000001f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 55 aa  |..............U.|
00000200  f8 ff ff 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000210  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000400  f8 ff ff 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000410  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00019000
```

Otrzymaliśmy pusty system plików na naszym urządzeniu. Gdybyśmy przy tworzeniu systemu plików zamiast pliku `fat.img` podali ścieżkę do zamontowanego pendrive'a, to byśmy go w ten sposób sformatowali i ustawili na nim system plików FAT12.

### Budowa Boot Sectora w FAT12

Pierwsze 512 bajtów, a więc do offsetu `000001f0` zajmuje tzw. *Boot sector* naszego urządzenia. Znajdują się tam szczegółowe informacje o używanym systemie plików na urządzeniu. Boot sector w FAT zawsze kończy się sekwencją `55 aa`.

Po pełny opis co oznacza który bajt w tym sektorze możesz odwiedzić [tę stronę](https://www.win.tue.nl/~aeb/linux/fs/fat/fat-1.html#ss1.2)

Ja wypiszę kilka ciekawych i najważniejszych wartości.

| Nr bajtu | Opis |
|----------|------|
| 11-12 | Liczba **bajtów** na jeden **sektor**.<br>Dozwolone wartości: 512, 1024, 2048, 3096. |
|  13   | Liczba **sektorów** na jeden **klaster**.<br>Dozwolone wartości: 1, 2, 4, 8, 16, 32, 64, 128 |
|  16   | Liczba kopii **FAT (tablicy alokacji)** |
| 22-23 | Liczba sektorów na jedną tablicę alokacji |
|510-511| Sygnatura `55 aa` |

Zerknijmy jeszcze raz na *Boot Sector*, tym razem dla ułatwienia z offsetem dziesiętnym i spróbujmy odczytać z niego wartości posługując się tabelą powyżej.

```bash
$ od -Ad -tx1z fat.img
0000000 eb 3c 90 6d 6b 66 73 2e 66 61 74 00 02 04 01 00  >.<.mkfs.fat.....<
0000016 02 00 02 c8 00 f8 01 00 20 00 40 00 00 00 00 00  >........ .@.....<
0000032 00 00 00 00 80 00 29 9c 38 71 69 4e 4f 20 4e 41  >......).8qiNO NA<
0000048 4d 45 20 20 20 20 46 41 54 31 32 20 20 20 0e 1f  >ME    FAT12   ..<
0000064 be 5b 7c ac 22 c0 74 0b 56 b4 0e bb 07 00 cd 10  >.[|.".t.V.......<
0000080 5e eb f0 32 e4 cd 16 cd 19 eb fe 54 68 69 73 20  >^..2.......This <
0000096 69 73 20 6e 6f 74 20 61 20 62 6f 6f 74 61 62 6c  >is not a bootabl<
0000112 65 20 64 69 73 6b 2e 20 20 50 6c 65 61 73 65 20  >e disk.  Please <
0000128 69 6e 73 65 72 74 20 61 20 62 6f 6f 74 61 62 6c  >insert a bootabl<
0000144 65 20 66 6c 6f 70 70 79 20 61 6e 64 0d 0a 70 72  >e floppy and..pr<
0000160 65 73 73 20 61 6e 79 20 6b 65 79 20 74 6f 20 74  >ess any key to t<
0000176 72 79 20 61 67 61 69 6e 20 2e 2e 2e 20 0d 0a 00  >ry again ... ...<
0000192 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
0000496 00 00 00 00 00 00 00 00 00 00 00 00 00 00 55 aa  >..............U.<
0000512 f8 ff ff 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
0000528 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
0001024 f8 ff ff 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
0001040 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
0102400
```

Bajty liczymy od zera, więc bajt numer 1 ma wartość `3c`, bajt numer 16 `02` itd.

Spróbuj samodzielnie odczytać wartości z Boot Sectora. Odpowiedzi zamieszczam w tabeli poniżej.

| Pole | raw | hex | dec |
|------|-----|-----|-----|
|Liczba bajtów na sektor|`00 02`|`0200`|`512`|
|Liczba sektorów na klaster| `04` | `04` | `4` |
|Liczba kopii FAT| `02` | `02` | `2` |
|Liczba sektorów na FAT| `01 00` | `0001` | `1` |

Mimo tego, że Boot Sector mamy przedstawiony w postaci ciągu bajtów zapisanych szesnastkowo, muszę rozróżniać tutaj wartość odczytaną (raw) od wartości liczbowej, którą reprezentuje ten zapis (hex). Wartość uzyskamy czytając bajty w odwrotnej kolejności. Jeśli nie rozumiesz dlaczego, przeczytaj o formie zapisu [Little Endian](https://pl.wikipedia.org/wiki/Kolejno%C5%9B%C4%87_bajt%C3%B3w).

Z opisów z tabel już się pewnie domyślasz jaki stosunek do siebie mają *bajty*, *sektory* i *klastry*.

**Sektor** to zbiór **bajtów** o stałym rozmiarze.
**Klaster** to zbiór **sektorów** o stałym rozmiarze.

Mamy informację, że tablica alokacji FAT *(File Allocation Table)* składa się z jednego sektora, a więc z 512 bajtów. Wiemy również, że nasz system plików przechowuje 2 kopie tablicy alokacji (jest to pewna forma zabezpieczenia). Sprawdźmy, czy to co odczytaliśmy z *Boot Sectora* się zgadza.

Dokładnie od offsetu 512 zaczyna się pierwsza tablica alokacji FAT, która kończy się 512 bajtów dalej. Następnie zaczyna się identyczna kopia FAT. Są tam już wpisane jakieś wartości. Omówimy je za moment.

### Budowa tablicy alokacji FAT
Tablica alokacji jest taką mapą po pamięci. Jeśli mamy jakiś duży plik, który zajmuje kilka sektorów, tablica alokacji powie nam które kolejne sektory powinniśmy odczytywać, aby otrzymać cały plik.

W FAT12 jeden klaster jest kodowany na 12 bitach, a więc 3 bajtach. Powoduje to trochę komplikacji przy próbie ręcznego odczytywania wartości z postaci szesnastkowej, którą my mamy. Dla FAT16 i FAT32 jest to dużo prostsze.

Odczytajmy zawartość tablicy alokacji, bo widać, że już coś się tam w niej znajduje. Zaczynamy od offsetu 512. Algorytm jest następujący:
1. Wypisz bajty z tablicy alokacji. Zera z końca możemy pominąć.
```
f8 ff ff
```
2. Łączymy bajty w trójki od lewej do prawej (w naszym przypadku mamy już trójkę, więc nic nie robimy)
```
ff ff f8
```
3. Przepisujemy wszystkie **bajty** w trójkach, w odwrotnej kolejności
(Taki zapis: `ff ff 8f` **nie** jest poprawnym odwróceniem bajtów)
```
ff ff f8
```
4. Wybrane trójki dzielimy na dwie równe części
```
fff ff8
```
5. Każdą z par odczytujemy w odwróconej kolejności. Otrzymaliśmy tablicę alokacji w postaci szesnastkowej.
```
ff8 fff
[0] [1] - numery klastrów
```

Później będziemy odczytywać bardziej skomplikowane tablice alokacji, więc powrócimy do tego algorytmu.

Wiemy zatem, że klaster o numerze 0 jest zakodowany jako `ff8`, a klaster 1 jako `fff`. Wszystkie pozostałe klastry w urządzeniu są oznaczone jako `000`. Co te liczby oznaczają?

| Wartość | Opis |
|---------|------|
| `000` | Klaster wolny |
| `002-fef` | Klaster zajęty |
| `ff0-ff6` | Klaster zarezerwowany |
| `ff7` | Zły sektor |
| `ff8-fff` | Ostatni klaster |

Wiemy zatem, że klastry 0 i 1 są ostatnimi klastrami. Nasze urządzenie nie ma jeszcze żadnego pliku, więc klastry te nie reprezentują żadnego pliku. 

Co to znaczy, że klaster jest ostatni? Pokażemy to dokładnie później, przy omawianiu plików, które zajmują więcej niż jeden klaster.

### Root Directory Table
Dodajmy w końcu jakiś plik do naszego urządzenia i zobaczmy co się zmieni. Najpierw będziemy musieli zamontować nasze urządzenie `fat.img` w systemie.

```
$ mkdir fs
$ sudo mount -t msdos fat.img fs -o umask=000,loop
```

Po utworzeniu katalogu montujemy do niego nasze urządzenie. Dodatkowe opcje, których używamy rozwiązują problem uprawnień (umask) i pozwalają zamontować plik jako urządzenie [Loop Device](https://en.wikipedia.org/wiki/Loop_device).

Możemy teraz przejść do folderu `fs`. Będzie pusty. Utwórzmy tam plik.

```
$ cd fs
$ echo "Witaj" > hello
```

Powróćmy do naszego `fat.img` i sprawdźmy jego zawartość. Zanim to jednak zrobimy zapiszmy wszystkie oczekujące zmiany do pamięci trwałej poprzez wykonanie `sync`.

```
$ sync
$ od -Ax -tx1z fat.img
[...]
000200 f8 ff ff 00 f0 ff 00 00 00 00 00 00 00 00 00 00  >................<
000210 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
000400 f8 ff ff 00 f0 ff 00 00 00 00 00 00 00 00 00 00  >................<
000410 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
000600 48 45 4c 4c 4f 20 20 20 20 20 20 20 00 00 00 00  >HELLO       ....<
000610 00 00 00 00 00 00 fc 6d 07 4f 03 00 06 00 00 00  >.......m.O......<
000620 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
004e00 57 69 74 61 6a 0a 00 00 00 00 00 00 00 00 00 00  >Witaj...........<
004e10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
019000
```

Pomijam już wypisywanie Boot Sectora, bo nie ulega on zmianom. Widzimy nazwę naszego pliku oraz jego zawartość. Offset `600` (hex) jest to początek tzw. Root Directory Table. Jest to, jak sama nazwa wskazuje, nasz folder główny, które może przechowywać pliki i foldery. Ilość plików i folderów w nim jest ograniczona. Obecne systemy plików nie posiadają już takich ograniczeń.

Dane pliku "hello" są zapisane na 32 bajtach (2 wiersze). Każdy bajt koduje pewną informację o pliku.


| Nr bajtu | Opis |
|----------|------|
| 0-10 | Nazwa pliku (8), rozszerzenie (3) |
| 11 | Atrybuty pliku |
| 12-21 | *Reserved* |
| 22-23 | Czas |
| 24-25 | Data |
| 26-27 | Początkowy klaster (0 - pusty plik) |
| 28-31 | Rozmiar pliku w bajtach |

Po więcej informacji zajrzyj [tutaj](https://www.win.tue.nl/~aeb/linux/fs/fat/fat-1.html#ss1.4).

## Odczytywanie krótkiego pliku
Spróbujmy odczytać plik `hello` z naszego urządzenia w podobny sposób jak robi to komputer. 

W *Root Directory Table* po nazwie odnajdujemy nasz plik `hello`.

```
000600 48 45 4c 4c 4f 20 20 20 20 20 20 20 00 00 00 00  >HELLO       ....<
000610 00 00 00 00 00 00 fc 6d 07 4f 03 00 06 00 00 00  >.......m.O......<
```

Odnajduję rozmiar pliku na bajtach 28-31. Jest to więc liczba 4-bajtowa zapisana w [Little Endian](https://pl.wikipedia.org/wiki/Kolejno%C5%9B%C4%87_bajt%C3%B3w). **Rozmiar pliku to 6 bajtów**

|bajty| raw | hex | dec |
|-----|-----|-----|-----|
|28-31|`06 00 00 00` | `00000006` | `6` |

Odczytuję również na bajtach 26 i 27 początkowy klaster, gdzie znajduje się zawartość tego pliku.

|bajty| raw | hex | dec |
|-----|-----|-----|-----|
|26-27|`03 00` | `0003` | `3` |

Wiemy, zatem, że **trzeci sektor jest sektorem, w którym zaczyna się nasz plik**. Udajemy się do tablicy alokacji i patrzymy jak jest zakodowany trzeci sektor. 

Tablica alokacji wygląda tak:
```
000200 f8 ff ff 00 f0 ff 00 00 00 00 00 00 00 00 00 00  >................<
000210 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
```

Posłużymy się algorytmem opisanym wcześniej, aby odczytać kodowania dla poszczególnych klastrów. Jak nabierzesz wprawy, to będziesz to odczytywać zwykłym spojrzeniem na powyższy ciąg bajtów (choć nie wiem po co ktokolwiek miałby nabierać wprawy w ręcznym odczytywaniu tablicy alokacji).

```
f8 ff ff 00 f0 ff     - (1) wypisujemy tablicę (bez zer na końcu)
f8 ff ff   00 f0 ff   - (2) łączymy bajty w trójki
ff ff f8   ff f0 00   - (3) odwracamy kolejność
fff ff8    fff 000    - (4) sklejamy trójki
ff8 fff    000 fff    - (5) odwracamy

ff8 fff 000 fff - wynik końcowy
[0] [1] [2] [3] - numery klastrów

```

Nasz plik zaczyna się od klastra nr 3. Według tablicy alokacji, klaster [3] (`fff`) jest **klastrem ostatnim** (patrz [tabelka z oznaczeniami](#budowa-tablicy-alokacji-fat)).

Nasz plik zatem składa się tylko z klastra nr 3. Znamy długość każdego klastra oraz miejsce, gdzie zaczyna się klaster [0]. Przeskakujemy więc zadaną liczbę klastrów i zaczynamy czytać klaster [3] od bajtu 004e00<sub>16</sub>.

```
004e00 57 69 74 61 6a 0a 00 00 00 00 00 00 00 00 00 00  >Witaj...........<
004e10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
```

Plik w systemie FAT nie może zajmować zajmować na dysku mniej niż jeden klaster. Nawet jeśli zawiera kilka liter jak w naszym przypadku. Musimy zatem wiedzieć gdzie w klastrze kończą się dane pliku. Odczytaliśmy wcześniej rozmiar pliku, który wynosi **6 bajtów**.

6 pierwszych bajtów trzeciego klastra to: `Witaj\n`. Koniec.

## Odczytywanie długiego pliku
Przyjżyjmy się teraz co się stanie, gdy plik będzie dłuższy niż jeden klaster. Najpierw musimy taki plik utworzyć. Zróbmy to inteligentnie. Odczytaliśmy wcześniej, że klaster składa się z 4 sektorów. Jeden sektor ma 512 bajtów, zatem klaster pomieści 2048 bajtów. 

Plik, który dodamy będzie się składał z 2048 literek `a`, a na koniec dokleimy `bcdef`. Jeśli nasze obliczenia są poprawne, to literki `a` zostaną zapisane do jednego sektora, a dalsza część pliku `bcdef` do innego.

Przechodzimy do katalogu `fs` i generujemy plik.
```
$ python -c "print('a'*2048 + 'bcdef')" > duzy
$ sync
```

Zobaczmy teraz co nam powstało w pliku `fat.img`.

```
000200 f8 ff ff 00 f0 ff 05 f0 ff 00 00 00 00 00 00 00  >................<
000210 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
000400 f8 ff ff 00 f0 ff 05 f0 ff 00 00 00 00 00 00 00  >................<
000410 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
000600 48 45 4c 4c 4f 20 20 20 20 20 20 20 00 00 00 00  >HELLO       ....<
000610 00 00 00 00 00 00 fc 6d 07 4f 03 00 06 00 00 00  >.......m.O......<
000620 44 55 5a 59 20 20 20 20 20 20 20 20 00 00 00 00  >DUZY        ....<
000630 00 00 00 00 00 00 dc 84 07 4f 04 00 06 08 00 00  >.........O......<
000640 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
004e00 57 69 74 61 6a 0a 00 00 00 00 00 00 00 00 00 00  >Witaj...........<
004e10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
005600 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61  >aaaaaaaaaaaaaaaa<
*
005e00 62 63 64 65 66 0a 00 00 00 00 00 00 00 00 00 00  >bcdef...........<
005e10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
019000
```

Jak ostatnio obciąłem *Boot Sector*.

Tym razem z *Root Directory Table* odczytujemy, że **pierwszym sektorem jest sektor [4]**. Nie będę tu powtarzał procedury odczytywania tego. Postępowanie jest analogiczne co z krótkim plikiem.

Odczytajmy tablicę alokacji, która zdecydowanie uległa zmianie. Wygląda ona tak:

```
000200 f8 ff ff 00 f0 ff 05 f0 ff 00 00 00 00 00 00 00  >................<
000210 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
```

Zasady odczytu takie same jak ostatnio.

```
f8 ff ff 00 f0 ff 05 f0 ff       - (1) wypisujemy tablicę
f8 ff ff   00 f0 ff   05 f0 ff   - (2) łączymy w trójki
ff ff f8   ff f0 00   ff f0 05   - (3) odwracamy kolejność
fff ff8    fff 000    fff 005    - (4) sklejamy trójki
ff8 fff    000 fff    005 fff    - (5) odwracamy

ff8 fff 000 fff 005 fff- wynik końcowy
[0] [1] [2] [3] [4] [5]- numery klastrów
```

Odczytujemy
- Plik `duzy` zaczyna się od klastra [4].
- Patrzymy, a klaster o tym numerze jest kodowany jako `005`. 
- Oznacza to, że dalsza część pliku znajduje się w klastrze [5]. 
- Patrzymy do klastra [5] i odczytujemy `fff`. Zatem jest to **ostatni klaster**.

Teraz wystarczy przeczytać klastry w kolejności `[4][5]`. Klaster [4] zaczyna się od offsetu `005600` i rzeczywiście 2048 bajtów dalej, tam gdzie spodziewamy się klastra [5] odnajdujemy dalszą część pliku `bcdef`.

Myślę, że już *czujesz* jak ten system działa. Oczywiście są to tylko proste operacje. Zachęcam do samodzielnego eksperymentowania. Jak system plików będzie zaśmiecony, to zawsze możesz utworzyć nowy. Spróbuj np. dopisać coś do któregoś z plików i sprawdź co się stanie 
*(spoiler: stara wersja pliku sprzed zmiany zostanie zachowana w pamięci)*

## Usuwanie i odzyskiwanie pliku
Przechodzimy do jednej z najciekawszych rzeczy, czyli co się dzieje gdy usuwamy plik i czy można go odzyskać. Jeśli tak, to jak to zrobić. Tym się teraz zajmiemy.

Nadal będziemy pracować na pliku `fat.img`, który mamy z poprzednich paragrafów. Jeśli Twój obecny po zabawach uległ uszkodzeniu lub zaśmieceniu, to możesz go odtworzyć w ten sposób:

```
dd if=/dev/zero of=fat.img bs=1024 count=100
mkfs.msdos fat.img
mkdir fs
sudo mount -t msdos fat.img fs -o umask=000,loop
cd fs
echo "Witaj" > hello
python -c "print('a'*2048 + 'bcdef)" > duzy
sync
```

Plik z którym zaczynamy wygląda następująco.

```
$ od -Ax -tx1z fat.img
[...]
000200 f8 ff ff 00 f0 ff 05 f0 ff 00 00 00 00 00 00 00  >................<
000210 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
000400 f8 ff ff 00 f0 ff 05 f0 ff 00 00 00 00 00 00 00  >................<
000410 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
000600 48 45 4c 4c 4f 20 20 20 20 20 20 20 00 00 00 00  >HELLO       ....<
000610 00 00 00 00 00 00 fc 6d 07 4f 03 00 06 00 00 00  >.......m.O......<
000620 44 55 5a 59 20 20 20 20 20 20 20 20 00 00 00 00  >DUZY        ....<
000630 00 00 00 00 00 00 dc 84 07 4f 04 00 06 08 00 00  >.........O......<
000640 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
004e00 57 69 74 61 6a 0a 00 00 00 00 00 00 00 00 00 00  >Witaj...........<
004e10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
005600 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61  >aaaaaaaaaaaaaaaa<
*
005e00 62 63 64 65 66 0a 00 00 00 00 00 00 00 00 00 00  >bcdef...........<
005e10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
019000
```

Mamy zatem dwa pliki. Jeden mały i duży. Każdy z nich usuniemy i spróbujemy odzyskać.

### Usuwanie krótkiego pliku
Sama procedura jest dość prosta. Po prostu wchodzimy do kadalogu `fs` i usuwamy plik `hello`.

```
$ rm hello
$ sync
```

No i poszło. Pliku nie ma, ale pytanie co się w praktyce stało? Okazuje się, że w całym pliku `fat.img` zmieniło się tylko **kilka bajtów**. To wyjaśnia dlaczego usuwanie pliku jest dużo szybsze niż jego kopiowanie. Dane cały czas siedzą na dysku. Poniżej wycinek zrzutu pamięci po usunięciu pliku `hello`.

```
000400 f8 ff ff 00 00 00 05 f0 ff 00 00 00 00 00 00 00  >................<
000410 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
000600 e5 45 4c 4c 4f 20 20 20 20 20 20 20 00 00 00 00  >.ELLO       ....<
000610 00 00 00 00 00 00 fc 6d 07 4f 03 00 06 00 00 00  >.......m.O......<
000620 44 55 5a 59 20 20 20 20 20 20 20 20 00 00 00 00  >DUZY        ....<
000630 00 00 00 00 00 00 dc 84 07 4f 04 00 06 08 00 00  >.........O......<
000640 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
004e00 57 69 74 61 6a 0a 00 00 00 00 00 00 00 00 00 00  >Witaj...........<
004e10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
```

Co dokładnie zaszło to:
* W tablicy alokacji (offset `000400` i `000200`) zwolnił się klaster [3] zajmowany przez plik `hello`
* Bajt z pierwszą literą w nazwie pliku zmienił wartość na `e5`.

Plik i jego dane nadal znajdują się na dysku. Oczywiście może zostać w każdej chwili nadpisany jeśli zaczniemy tworzyć nowe pliki.

### Odzyskiwanie krótkiego pliku

Co należy zrobić, aby odzyskać ten plik? W przypadku pliku, który zajmuje tylko jeden klaster jest to proste. 
* Odnajdujemy plik w *Root Directory Table* (lub w folderze gdzie się znajdował)
* Odczytujemy klaster od którego się rozpoczyna ten plik.
* Jeśli klaster jest oznaczony jako zajęty, to już jest za późno. Dane zostały nadpisane przez inny plik.
* Jeśli klaster jest wolny `000`, to jest szansa, że dane z tego pliku nadal się w nim znajdują.
* Ręcznie oznaczamy klaster jako zajęty `fff`
* Zmieniamy bajt w nazwie pliku `e5` na inny znak, np. literę.

Aby to wykonać można posłużyć się jakimś Hexedytorem. Jest to oczywiste, gdy dane mamy zapisane w pliku.

Co jeśli chcemy to wykonać na urządzeniu typu pendrive? Jest to trochę trudniejsze, ale nadal wykonalne. Wystarczy zrzucić z pendrive'a pamięć do pliku, podobnie jak robiliśmy tworząc plik `fat.img` programem `dd`. Nie musimy nawet zrzucać całości, wystarczy fragment. Po dokonaniu zmian wgrywamy zmodyfikowaną pamięć do urządzenia (w to samo miejsce, z którego ją pobraliśmy).

Problemy się zaczynają, gdy chcemy odzyskać duży plik. Taki, który zajmował kilka klastrów.

### Usuwanie długiego pliku
Usuńmy `duzy` plik i popatrzmy co się stało.

```
$ rm duzy
$ sync
```

A oto efekt naszych zmian.

```
000200 f8 ff ff 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
000210 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
000400 f8 ff ff 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
000410 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
000600 e5 45 4c 4c 4f 20 20 20 20 20 20 20 00 00 00 00  >.ELLO       ....<
000610 00 00 00 00 00 00 fc 6d 07 4f 03 00 06 00 00 00  >.......m.O......<
000620 e5 55 5a 59 20 20 20 20 20 20 20 20 00 00 00 00  >.UZY        ....<
000630 00 00 00 00 00 00 dc 84 07 4f 04 00 06 08 00 00  >.........O......<
000640 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
004e00 57 69 74 61 6a 0a 00 00 00 00 00 00 00 00 00 00  >Witaj...........<
004e10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
005600 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61  >aaaaaaaaaaaaaaaa<
*
005e00 62 63 64 65 66 0a 00 00 00 00 00 00 00 00 00 00  >bcdef...........<
005e10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
*
019000
```

Tablica alokacji jest pusta. Wszystkie klastry zajmowane przez plik zostały zwolnione, a pierwszy bajt w nazwie pliku `duzy` ma wartość `e5`. Żadnych zaskoczeń.

### Odzyskiwanie długiego pliku
Teraz mamy problem i to poważny. Co prawda z *Root Directory Table* nadal możemy odczytać pierwszy sektor zajmowany przez plik, ale gdy udamy się do tablicy alokacji pod wskazany sektor, to okaże się, że jest on wolny `000`. To świetne wieści, ale pozostaje pytanie, **który klaster jest tym następny**?

Po rozmiarze pliku możemy obliczyć ile klastrów będzie on zajmować. Musimy je "tylko" odnaleźć i to jeszcze **w dobrej kolejności**.

Niestety nie ma tutaj błyskotliwego rozwiązania tego problemu. Zgadywanie zwykle nie wchodzi w grę. Jest jednak wiele metod, które pozwolą zwiększyć szansę na odzyskanie takiego pliku. Poniżej kilka z nich:

* Defragmentacja dysku często sprawia, że wszystkie klastry pliku znajdują się obok siebie jeden za drugim. Z drugiej strony wykonanie defragmentacji **po** usunięciu pliku może "zapchać dziury po usuniętych plikach", a więc nadpisać ich dane.
* Stosowanie różnego rodzaju heurystyk
* Rozpoznawanie formatu pliku po pierwszym klastrze. Wiemy wtedy jakiej następnej części się spodziewać, np. gdy odzyskujemy plik wideo, to czasami wraz z zakodowanym obrazem znajdują się timestamp'y określające moment w filmie.

Jest zapewne dużo więcej sposobów. Wiele z nich to tajemnice firm produkujących oprogramowanie do odzyskiwania danych z dysków. Jeśli próbowałeś kiedyś któregoś z tych "darmowych" narzędzi do odzyskiwania plików po ich usunięciu, to wiesz już teraz dlaczego one z taką łatwością wypisywały usunięte pliki z nazwami, a za ich odzyskanie trzeba było już zapłacić. Odnalezienie usuniętych plików jest po prostu banalnie proste. 

Mówię tutaj oczywiście o formacie FAT. Struktury plików zapisanych w NTFS czy ext są zupełnie inne i bardziej skomplikowane. Nawet w samym FAT32 jest już sporo zmian w stosunku do FAT12, który przedstawiałem.

## Podsumowanie

Jeśli nigdy wcześniej nie miałeś, czytelniku, okazji pobawić się z systemem plików, to mam nadzieję, że wykonane tutaj eksperymenty i zaprezentowane przykłady sporo nauczyły. To dopiero wierzchołek góry lodowej. O samym FAT12 możnaby zrobić jeszcze 10 podobnej długości artykułów. Zamiast to robić, lepszym pomysłem jest zapoznanie się z nowszymi systemami plików. Porównać omówiony tutaj FAT12 z używanym jeszcze powszechnie na pendrive'ach FAT32.

Do czego może się przydać taka wiedza? Mało kto ręcznie grzebie w bajtach, więc wiedza o działaniu systemu plików przyda się do napisania oprogramowania, który na takim systemie operuje. Poza tym daje ogólne **zrozumienie wielu procesów, które zachodzą w komputerze**. Mimo, że o tym nie wspomniałem, to po przeczytaniu tego artykułu powinieneś już rozumieć dlaczego w Windowsie każdy plik ma *rozmiar* i *rozmiar na dysku*. To tak jak nasz krótki plik, który miał rozmiar 6 bajtów, a na dysku zajmował jeden klaster, a więc 2 kB. To spore marnotrawstwo pamięci i współczesne systemy plików lepiej sobie z tym radzą.

***
*Dygresja*

W NTFS na Windowsie jest trochę inaczej. Zrób eksperyment. Utwórz w Windowsie pusty plik i sprawdź jego *rozmiar* i *rozmiar na dysku*. Oba będą wynosić 0 B. Dodaj literkę `a` do pliku, zapisz i sprawdź. Ile wynosi rozmiar? 1 B. Rozmiar na dysku nadal 0 B. NTFS działa trochę inaczej i jest w stanie dane krótkich plików zmieścić wraz z nazwą w jednym rekordzie bez zajmowania całego klastra. Po zwiększeniu pliku do 1 kB, rozmiar na dysku wyniesie 4 kB (w moim przypadku). Po zmniejszeniu pliku do 1 B, rozmiar na dysku pozostaje 4 kB. Widać, że ten system plików jest inteligentniejszy.
***

Za wszelkie przypadkowe błędy i skróty myślowe przepraszam. Nie traktujcie, proszę, tego wpisu jako specyfikacji systemu FAT ani nic takiego. Mam nadzieję, że ktoś się zaciekawił i wyniósł nową wiedzę z tego tekstu.
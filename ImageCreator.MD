# Generator obrazów dysku - pomysł na projekt

Celem projektu jest stworzenie programu-biblioteki, dzięki której użytkownik mógłby na podstawie pliku konfiguracyjnego (lub struktury danych w kodzie) wygenerować pełny obraz dysku jako plik binarny - gotowy do zapisu bezpośrednio na fizycznym nośniku pamięci.

## Zastosowania

W toku pracy nad jądrami systemu lub programami samodzielnymi zachodzi potrzeba wielokrotnego generowania obrazów dysku, które zawierają opracowywany program, a których używa się do testów z wykorzystaniem maszyn wirtualnych. Obecnie takie obrazy generuje się ręcznie z wykorzystaniem podstawowych narzędzi *binutils* oferowanych przez większość dystrybucji Linuxa.

Możliwość automatyzacji tworzenia obrazów dysku usprawniłaby tok pracy nad jądrami systemu oraz programami samodzielnymi.

## Opis szczegółowy

### Pojęcia

**Obraz dysku** - plik binarny, który, z dokładnością do jednego bajta, odzwierciedla informacje zapisane (lub mogące zostać zapisane) na fizycznym nośniku pamięci. Istnieje wiele formatów zapisu obrazu dysku. Podstawowym jest oczywiście zwykły zapis binarny. Do innych popularnych formatów zapisu zaliczyć można starszy .VHD lub nowszy .VMDK. Różnią się one od prostego zapisu binarnego tym, że zawierają dodatkowe informacje o dysku, na podstawie którego dany obraz został stworzony. 

**Partycja** - wydzielony obszar dysku, na którym istnieje system plików. W przeszłości dyski dzielono na partycje aby na pojedynczym dysku mogły współistnieć różne systemy plików. W epoce dysków wielkości szaf istniały rózne systemy plików zoptymalizowane pod różne rodzaje zadań, więc wykorzystanie systemu plików odpowiedniego do konkretnego zadania mogło znacznie usprawnić pracę programu. 

Obecnie wykorzystanie różnych systemów plików do różnych zadań jest dużo mniej popularne, chociaż zdarza się, np. podczas instalacji Linuxa można wydzielić osobną partycję na dane użytkowników. Głównym powodem, dla którego dysk dzielony jest na partycje jest kompatybilność - systemy operacyjne takie jak Windows, MacOS czy Linux[^1] nie pozwolą na transfer danych z/do nośnika danych, który nie będzie miał wydzielonej przynajmniej jednej partycji.

**Tablica partycji** - binarna struktura danych, która opisuje partycje, na które podzielony jest dysk. Istnieją dwa rodzaje tablic partycji - Master Boot Record (MBR) oraz GUID Partition Table (GPT).

**Master Boot Record (MBR)** - stary typ tablicy partycji. Zawsze zajmuje pierwsze 512 bajtów dysku. Zawiera maksymalnie tylko 4 partycje. Istnieją również ograniczenia co do wielkości samych partycji. Są to główne powody, dla których powstał nowy typ tablicy partycji, tzw. GPT. Oprócz samego opisu partycji, zawiera też inne dane, np. bootloader. Instnieje kilka wersji MBR, które, zdaje się, są wstecznie kompatybilne z wersją pierwotną. Szczegóły dotyczące każdej wersji, włącznie z dokładnymi opisami każdej wersji, można znaleźć [tu](https://en.wikipedia.org/wiki/Master_boot_record).

**GUID Partition Table (GPT)** - nowy typ tablicy partycji. Pozwala na wydzielenie dużo większej ilości, dużo większych partycji. Na dysku zapisany jest dwukrotnie: na początku, oraz na końcu dysku. Szczegółowy opis można znaleźć [tu](https://en.wikipedia.org/wiki/GUID_Partition_Table).

**System plików** - binarna struktura danych opisująca pliki i foldery zapisane na danej partycji. Istnieje wiele systemów plików, które różnią się stopniem złożoności, oferowanymi funkcjami i ograniczeniami. Najbardziej popularne systemy plików to rodzina FAT (File Allocation Table), w tym FAT12, FAT16, FAT32 oraz najnowszy exFAT; NTFS (domyślnie wykorzystywany przez system Windows), oraz rodzina EXT, w szczególności EXT4 (domyślnie wykorzystywany przez Linuxa).

**Bootowanie** - proces, którego celem jest uruchomienie systemu operacyjnego zapisanego na danym dysku. W trakcie uruchamiania komputera, BIOS odczytuje tablice partycji każdego podłączonego dysku i decycuje czy na danym dysku występuje system operacyjny, który można spróbować uruchomić. Ów proces decyzyjny nie jest skomplikowany i sprowadza się do odczytania kilku atrybutów partycji zawartych w tablicy partycji (MBR) oraz wczytaniu pliku bootloadera z partycji bootowalnej (GPT). Dysk nazywamy bootowalnym kiedy BIOS wykrywa go jako kandydata do bootowania na podstawie jego tablicy partycji. Bootowalne dyski można zobaczyć w trakcie włączania się komputera wciskając wielokrotnie klawisz na klawiaturze (który konkretnie - to zależy od producenta komputera, a konkretniej - płyty głownej, np. na płytach głównych ASUS jest to klawisz F8, a na komputerach Dell - F2 lub F12).

### Wizja projektu

Użytkownik korzysta z programu za pomocą wiersza poleceń, albo wykorzystuje funkcje programu zawierając część jego kodu w swojej aplikacji - importując bibliotekę kodu.

Aby wygenerować obraz dysku poprzez wiersz poleceń, użytkownik przygotowuje plik tekstowy w formacie YAML lub TOML, w którym szczegółowo opisuje żądany obraz. W opis żądanego obrazu wchodzi m.in.:
  - typ tablicy partycji
  - atrybuty poszczególnych partycji zależnie od wybranego rodzaju tablicy partycji (np. GUID partycji w tablicy GPT)
  - typ systemu plików na każdej partycji
  - pliki i foldery zawarte w każdym systemie plików - przez wskazanie ścieżki do istniejącego pliku, który będzie skopiowany do tworzonego obrazu dysku
  - inne atrybuty wspierane przez poszczególne systemy plików (np. właściciel pliku czy uprawienia do pliku dla właściciela/grupy/innych użytkowników)

Aby wygenerować obraz dysku w kodzie, należy udostępnić użytkownikowi metodę, która jako argument przyjmie strukturę danych opisującą żądany obraz dysku, oraz ścieżkę do pliku gdzie żądany obraz ma być zapisany.

### Poziomy zaawansowania

**Podstawowy** - użytkownik używa biblioteki do wygenerowania obrazu dysku o formacie partycji MBR w najbardziej podstawowym wariancie (opisanym na Wikipedii jako *classical generic MBR*), który dzieli na dowolną (ograniczoną specyfikacją tablicy partycji) liczbę partycji. Jest w stanie wskazać partycję, którą chce oznaczyć jako bootowalną. Jest w stanie określić zawartość obszaru *Bootstrap code area* przez wskazanie wybranego przez siebie pliku, który przechowuje żądaną zawartość. Stworzone partycje zawierają dowolny system plików z rodziny FAT, który zawiera pliki i foldery określone przez użytkownika.

Weryfikacja zdolności programu będzie dwuetapowa. Pierwszy etap będzie polegał na nagraniu wygenerowanego obrazu na nośnik pamięci i podłączeniu go do komputera z systemem Linux, a następnie na weryfikacji zawartości żądanych plików i folderów. Drugi etap sprawdzi czy nośnik pamięci jest bootowalny poprzez zapisanie odpowiedniej zawartości *Boostrap code area* i wykorzystanie maszyny wirtualnej QEMU.

**Zaawansowany** - Postawowy + co samo co w Podstawowym, ale z wykorzystaniem tablicy partycji w formacie GPT, z uwzględnieniem różnic w wymaganiach, które dyski/partycje muszą spełnić aby być bootowalne (np. partycja bootowalna w tablicy GPT musi mieć odpowiedni type GUID oraz system plików FAT32). 

Weryfikacja zdolności programu będzie przebiegać analogicznie do poziomu podstawowego, z uwzględnieniem różnic.

**Bardzo zaawansowany** - Zaawansowany + obsługa systemów plików EXT2/3/4 i NTFS.

Weryfikacja zdolności j.w.

Do projektu można podejść na dwa sposoby - można albo spróbować wygenerować wszystkie wymagane struktury danych od podstaw - zapoznając się z dokumentacją techniczną i implementując niezbędne jej fragmenty w kodzie, albo można wygenerować te struktury wywołując w kodzie odpowiednie narzędzia *binutils*. Sposób pierwszy jest w oczywisty sposób trudniejszy, ale też obiektywnie lepszy, gdyż pozwala na pracę programu w środowiskach gdzie narzędzia *binutils* nie są dostępne, np. na systemie Windows. Dodatkowym sposobem pośrednim byłoby wygenerowanie części struktur samodzielnie, a części z wykorzystaniem *binutils*. 

Jeśli program wykorzystuje narzędzia *binutils* do wygenerowania wszystkich rodzajów struktur (tj. tablicy partycji i systemów plików), jego poziom zaawansowania w stosunku do oferowanych funkcji obniża się o dwa stopnie. Jeśli natomiast przynajmniej jeden z rodzajów struktur danych (tj. tablica partycji lub systemy plików) jest generowany samodzielnie przez program, poziom zaawansowania w stosunku do oferowanych funkcji obniża się o jeden stopień.

### Uwagi dodatkowe

Obsługa systemów plików jest najtrudniejszym elementem tego projektu. O ile obsługi FAT12 czy FAT32 można nauczyć się z internetowych tutoriali i/lub czytając dokumentację techniczną, nie jestem pewien czy to samo można powiedzieć o EXT2/3/4 czy NTFS. Są to zdecydowanie bardziej skomplikowane systemy plików i raczej nie omawia się ich na studiach informatycznych (może poza MiT czy Harvardem), dlatego uważam, że poziom bardzo zaawansowany wykracza poza standardowe wymagania projektu na poziomie inż./mgr inż. informatyki (oczywiście jeśli zostanie zrealizowany samodzielnie przez program, nie przez wykorzystanie narzędzi *binutils*).

### Przykład - ręczne wygenerowanie niebootowalnego obrazu dysku

```bash
  dd if=/dev/zero of=obraz.bin bs=1M count=40 # Tworzy plik o rozmiarze 40MiB wypełniony zerami
  fdisk obraz.bin                             # Fdisk to interaktywny program
    > o [enter]                               # Tworzy nową tablicę partycji MBR
    > n [enter]                               # Dodaje nową partycję
    > p [enter]                               # Określa typ partycji (primary vs extended)
    > 1 [enter]                               # Określa numer partycji
    > 2048 [enter]                            # Określa początek partycji (numer pierwszego bloku)
    > 81919 [enter]                           # Określa koniec partycji (numer ostatniego bloku)
    > w [enter]                               # Zapisuje tablicę partycji i zamyka program
  sudo losetup -P /dev/loop0 obraz.bin        # "Oszukuje" system Linux, że obraz.bin to podłączony dysk
  ls /dev/loop0*                              # Zwraca /dev/loop0  /dev/loop0p1 <- stworzona partycja
  sudo mkfs.ext4 /dev/loop0p1                 # Tworzy system plików EXT4 na właśnie stworzonej partycji
  mkdir /tmp/PARTYCJA1                        # Tworzy folder, pod który "podpięta" będzie partycja
  sudo mount /dev/loop0p1 /tmp/PARTYCJA1      # Podpina partycję do wyznaczonego folderu
  sudo mkdir /tmp/PARTYCJA1/NowyFolder        # Tworzy folder na wygenerowanym systemie plikow
  ls /tmp/PARTYCJA1                           # Zwraca lost+found NowyFolder <- właśnie stworzony folder
  sudo umount /tmp/PARTYCJA1                  # Odpina partycję od folderu
  sudo losetup -d /dev/loop0                  # Przestaje "oszukiwać" system
```

Po wykonaniu ww. komend `obraz.bin` to pełnoprawny obraz dysku z tablicą partycji MBR, jedną partycją, systemem plików EXT4 oraz jednym folderem o nazwie `NowyFolder`.


[^1]: Linux umożliwia transfer danych do dysku niepodzielonego na partycje, ale trzeba taki dysk ręcznie skonfigurować przez "zamontowanie" całego urządzenia typu block jako partycji, np. `mount /dev/sda /tmp/PART1` zamiast klasycznego `mount /dev/sdaX /tmp/PART1`

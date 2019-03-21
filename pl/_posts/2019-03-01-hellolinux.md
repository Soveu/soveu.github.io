---
title: "Linux - jak zacząć?"
category: pl
tags: linux
---

Mam wrażenie, że dużo ludzi wyobraża sobie używanie Linuxa mniej więcej tak -
czarny ekran i dużo białego tekstu, którego nie do końca można zrozumieć.
Jednak ten stereotyp od dawna jest już obalony. Co prawda, dalej się można
spotkać z tym, ale jest to już albo kwestia własnej preferencji albo jest to
częściowo konieczne - przykładowo ciężko by było przesyłać na żywo obraz
pulpitu z serwera na drugim końcu kraju, tutaj dużo praktyczniejsze jest
używanie komunikacji tekstowej przy pomocy SSH. Dzisiaj systemy oparte na Linuxie
są dosyć podobne do Windowsa, dobrym przykładem mogłoby tutaj być środowisko
pulpitowe(?) (dosłowne tłumaczenie z angielskiego "desktop environment") KDE,
które można napotkać między innymi na takich dystrybucjach jak Kubuntu lub 
Manjaro KDE. Praktycznie, bardzo mało rzeczy musimy robić w terminalu, ponieważ
dużo jest graficznych programów, które są odpowiednikami tych tekstowych.

Ale warto pamiętać o tym, że system operacyjny to (tylko) narzędzie, czyli
jeżeli ktoś woli programować na Windowsie albo na Macu (chociaż MacOS jest
częściowo oparty na Unix'ie i też można bez problemu korzystać z terminala),
to niech programuje na Windowsie albo Macu! Stereotyp, że *"Każdy prawdziwy
programista potrafi korzystać z Linuxa"* jest również bzdurą - od osoby,
która miała okazję pracować dla takich korporacji jak Facebook czy Google, 
mogę powiedzieć, że nie jest to konieczność. Warto dopowiedzieć, że
[Gynvael Coldwind](https://gynvael.coldwind.pl/) najchętniej korzysta z Windowsa,
a [LiveOverflow](https://www.youtube.com/channel/UClcE-kVhqyiHCcjYwcpfj9w) używa
trzech systemów - Maca do pracy, Linuxa do wirtualnej maszyny (np. do pokazywania 
exploitów), a Windowsa do montażu wideo.

A teraz, przechodząc już do tematu

Potrzebne rzeczy:
* Obraz ISO [Kubuntu](https://kubuntu.org) lub innej dystrybucji Linuxa
* [Fedora Media Writer](https://getfedora.org/pl/workstation/download/)

Antywirus może was ostrzec o zagrożeniu, ale to jest w 100% bezpieczny program,
które mogą posłużyć do 'wypalenia' dystrybucji Linuxa na pendrivie.
Oczywiście można to też zrobić na dowolnej płycie DVD, ale to jest już powoli
przeżytkiem.

Obraz Kubuntu waży ok. 1,8GB, dlatego trzeba też zadbać o odpowiednio duży nośnik.
Fedora Media Writer zaproponuje zainstalowanie Fedory Workstation lub Server, można
zdecydować się na ten krok, jednak ja nie mam doświadczenia z tą dystrybucją i nie jestem
w stanie stwierdzić, czy nadaje się dla początkującego. Klikamy "Inny Obraz (Wybierz plik
z dysku)" i wybieramy odpowiedni obraz ISO.

## **UWAGA! Dane na nośniku zostaną nadpisane - pamiętaj o wykonaniu kopii zapasowej!**

Media Writer jest o tyle przyjemnym narzędziem, ponieważ pozwala na przywrócenie
poprzedniego układu partycji jednym kliknięciem.

Nie będę opisał instalacji - jest dosyć prosta, a partycjonowanie u każdego może i
wyglądać inaczej. Jedynie co polecam to stworzenie osobnej partycji dla systemu (/)
i osobnego dla użytkowników (/home). Pozwala to zaoszczędzić trochę czasu przy
reinstalacji systemu. Również, jeżeli to jest możliwe, zainstalować Kubuntu na
osobnym dysku - Windows potrafi nadpisać bootloader Linuxa przy aktualizacji.

Dobra, jest już dostęp do pulpitu - co dalej?
Przedtem prawdopodobnie korzystałeś z Codeblocks, jednak na Linuxie z mojego
doświadczenia nie jest to dobry wybór.

Może zacznijmy od podstaw - uruchomienia edytora tekstowego. W przypadku tej 
dystrybucji edytorem jest Kate (lub w przypadku czystego Ubuntu - Gedit) 
\- wchodzimy w menu i otwieramy. Używa się go podobnie 
jak notatnik na Windowsie. Stwórzmy plik o nazwie `hello.cpp`...

```c
#include<iostream>

int main() {
  std::cout << "Hello World!" << std::endl;
}
```

...i zapiszmy go.


Na Kubuntu emulatorem terminala jest Konsole. Żeby go uruchomić wystarczy wejść
do menu i powinien znajdować się na 'stronie głównej'.

Najczęściej używanym kompilatorem jest `g++` (do C++) i `gcc` (do C).
Nie jest on zawsze dołączony ze systemem, dlatego najpierw należy go zainstalować
przy pomocy `sudo apt install g++`.
Zostaniemy poproszeni o nasze hasło - nie zdziwcie się, jeśli nie zobaczycie gwiazdek/kropek
cenzurujących hasło jak w przeglądarce. Tutaj nic się nie wyświetla pomimo wpisywania hasła
(ze względów bezpieczeństwa).

Wracając do tematu kompilatora, używa się go w bardzo prosty sposób:

```
g++ hello.cpp -g
```

wtedy `hello.cpp` zostaje skompilowany do pliku `a.out`. Opcja `-g` przyda się
później, dodaje ona symbole do debugowania do pliku wykonywalnego. Jeżeli
chcemy, aby program skompilował się odrazu z konkretną nazwą

```
g++ hello.cpp -g -o hello
```

W ten sposób nastąpi kompilacja do pliku `hello`
Uruchomienie jest wygląda w następujący sposób - będąc w tym samym katalogu,
co plik, wpisujemy

```
./hello
```

Program powinien wypisać

```
Hello World!
```

## Inne narzędzia 

Oczywiście, nie jesteśmy zmuszeni do używania jednego edytora. Dzięki rozłożeniu
pracy na różne narzędzia, możemy bez problemu zmieniać dowolne z nich.
Aktualnie używamy `Kate` do pisania i terminala `Konsole` + `g++` do kompilacji,
jednak można np. zmienić edytor na Visual Studio Code (który jest dosyć dobrym
i popularnym edytorem), vim, nano, KDevelop, zmienić kompilator na clang++,
emulator terminala na gnome-terminal, termite. Wszystkie te programy można zainstalować
przy pomocy `sudo apt install <nazwa>`, z wyjątem VSCode - instaluje się go przez tzw. snapy.

```
sudo snap install vscode --classic
```

Przy czym `--classic` jest trochę mylącą nazwą - po prostu daje trochę większe
uprawnienia aplikacji. Polecam ten edytor, można go używać na dowolnej platformie,
posiada dosyć sporą liczbę wtyczek (przez co też niestety lubi być powolny).
Nawet posiada wbudowany terminal - nie trzeba się przełączać pomiędzy różnymi okienkami.

Pewnie zapytacie *Ale jak korzystać z terminala??*

Po prostu z niego korzystajcie :)
Dla ułatwienia napisałem krótki cheat-sheet:

* `cd katalog` - wchodzi do katalogu, bez podania go, wchodzi do głównego folderu użytkownika ($HOME lub ~)
* `cd ..` - przejdź do poprzedniego katalogu
* `mkdir katalog` - tworzy katalog
* `ls katalog` - wyświetla zawartość katalogu, bez podania katalogu wyświetla zawartość aktualnego
* `touch plik` - tworzy plik lub jeżeli istnieje, odświeża datę ostatniej modyfikacji
* `rm plik` - usuwa plik
* `rm -r katalog` - usuwa katalog
* `cat plik` - wypisz zawartość pliku jako dane wyjściowe
* `echo text` - wypisz text jako dane wyjściowe
* `sudo program` - wykonaj program jako root
* `grep wzór` - przefiltruj dane wejściowe używając wzoru
* `program1 | program2` - przekaż wyjście programu1 do wejścia programu2
* `program < plik` - użyj pliku jako dane wejściowe, podobny efekt do `cat plik | program`
* `program > plik` - zapisz dane wyjściowe programu do pliku
* `program >> plik` - dopisz dane wyjściowe programu do pliku

Przykłady do wypróbowania:

```
mkdir test
cd test
ls ~ | grep Do
sudo whoami
echo Przykładowy tekst > test.txt
echo Sample text >> test.txt
grep te < test.txt
rm test.txt
cd ..
rm -r test
```


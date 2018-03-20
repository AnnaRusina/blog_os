+++
title = "A minimal Multiboot Kernel"
order = 1
path = "multiboot-kernel/"
date  = 2015-08-18
+++

Эта статья объясняет как создать минимальное ядро операционной системы используя стандарт мультизагрузки. По факту, оно будет просто загружаться и печатать `OK` на экране. В последующих статьях мы расширием его используя язык программирования `Rust`.  

Я попытался объяснить всё в деталях и оставить код максимально простым насколько это возможно. Если у вас возникли вопросы, предложения или какие-либо проблемы пожалуйста оставьте комментарий или [создайте таску](https://github.com/phil-opp/blog_os/issues) на `GitHub`. Исходный код доступен в [репозитории](https://github.com/phil-opp/blog_os/tree/post_1/src/arch/x86_64).

<cut />

#Обзор

Когда вы включаете компьютер, он загружает [BIOS](https://en.wikipedia.org/wiki/BIOS) из специальной флэш памяти. `BIOS` запускает тесты самопроверки и инициализацию аппаратного обеспечения, затем он ищет загрузочные устройства. Если оно было найдено хотя бы одно он передаёт контроль загрузчику, который является небольшой частью запускаемого кода сохранённого в начале устройства хранения. Загрузчик определяет местоположение образа ядра находящегося на устройстве и загружает его в память. Ему так же необходимо переключить процессор в так называемый [защищённый режим](https://en.wikipedia.org/wiki/Protected_mode) потому что x86 процессоры по-умолчанию стартуют в очень ограниченном [режиме реального времени](https://wiki.osdev.org/Real_Mode) (чтобы быть совместимыми с программами из 1978).

Мы не будем писать загрузчик потому что это сам по себе сложный проект (если вы действительно хотите это сделать [почитайте об этом здесь](https://wiki.osdev.org/Rolling_Your_Own_Bootloader)). Вместо этого мы будем использовать один из [многих испытанных загрузчиков](https://en.wikipedia.org/wiki/Comparison_of_boot_loaders) для загрузки нашего ядра с CD-ROM. Но какой?

#Мультизагрузка

К счастью есть стандарт загрузчика: [спецификация мультизагрузки](https://en.wikipedia.org/wiki/Multiboot_Specification). Наше ядро должно лишь указать, что поддерживает спецификацию и любой совместимый загрузчик сможет загрузить его. Мы будем использовать спецификацию `Multiboot 2` ([PDF](http://nongnu.askapache.com/grub/phcoder/multiboot.pdf)) 
вместе с известным загрузчиком [GRUB 2](https://wiki.osdev.org/GRUB_2).

Чтобы сказать загрузчику о поддержке `Multiboot 2`, наше ядро должно начинаться  с `заголовка мультизагрузки`, который имеет следующий формат:

Field         | Type            | Value
------------- | --------------- | ----------------------------------------
магическое число  | u32             | `0xE85250D6`
арихтектура  | u32             | `0` для i386, `4` для MIPS
длина заголовка | u32             | общий размер заголовка включая тэги
контрольная сумма      | u32             | `-(магическое число + архитектура + длина заголовка)`
тэги          | variable        |
завершающий тэг       | (u16, u16, u32) | `(0, 0, 8)`

В переводе на x86 ассемблер это будет выглядеть так (`Intel` синтаксис):

```asm
section .multiboot_header
header_start:
    dd 0xe85250d6                ; магическое число (multiboot 2)
    dd 0                         ; архитектура 0 (защищённый режим i386)
    dd header_end - header_start ; длина заголовка
    ; контрольная сумма
    dd 0x100000000 - (0xe85250d6 + 0 + (header_end - header_start))

    ; вставьте опциональные `multiboot` тэги здесь

    ; требуюется завершающий тэг
    dw 0    ; тип
    dw 0    ; флаги
    dd 8    ; размер
header_end:
```

Если вы не знаете x86 ассемблер, то вот небольшая вводная:

- заголовок будет записан в секцию названную `.multiboot_header` (нам понадобится это позже)
- `header_start` и `header_end` это метки, которые указывают на месторасположение в памяти, мы используем их чтобы вычислить длину заголовка
- `dd` означает `define double` (32bit) и `dw` означает `define word` (16bit). Они просто выводят указанные 32bit/16bit константы.
- константа `0x100000000` в вычислении контрольной суммы это небольшой хак, чтобы избежать предупреждений компилятора

Мы уже можем собрать данный файл (который я назвал `multiboot_header.asm`) используя `nasm`.

<spoiler title="Установка nasm на `archlinux`">
```
[loomaclin@loomaclin ~]$ yaourt nasm
1 extra/nasm 2.13.02-1
    An 80x86 assembler designed for portability and modularity
2 extra/yasm 1.3.0-2
    A rewrite of NASM to allow for multiple syntax supported (NASM, TASM, GAS, etc.)
3 aur/intel2gas 1.3.3-7 (3) (0.20)
    Converts assembly language files between NASM and GNU assembler syntax
4 aur/nasm-git 20150726-1 (1) (0.00)
    80x86 assembler designed for portability and modularity
5 aur/sasm 3.9.0-1 (18) (0.61)
    Simple crossplatform IDE for NASM, MASM, GAS, FASM assembly languages
6 aur/yasm-git 1.3.0.r30.g6caf1518-1 (0) (0.00)
    A complete rewrite of the NASM assembler under the BSD License
==> Enter n° of packages to be installed (e.g., 1 2 3 or 1-3)
==> ---------------------------------------------------------
==> 1

[sudo] password for loomaclin: 
resolving dependencies...
looking for conflicting packages...

Packages (1) nasm-2.13.02-1

Total Download Size:   0.34 MiB
Total Installed Size:  2.65 MiB

:: Proceed with installation? [Y/n] 
:: Retrieving packages...
 nasm-2.13.02-1-x86_64                                                                                346.0 KiB  1123K/s 00:00 [#############################################################################] 100%
(1/1) checking keys in keyring                                                                                                 [#############################################################################] 100%
(1/1) checking package integrity                                                                                               [#############################################################################] 100%
(1/1) loading package files                                                                                                    [#############################################################################] 100%
(1/1) checking for file conflicts                                                                                              [#############################################################################] 100%
(1/1) checking available disk space                                                                                            [#############################################################################] 100%
:: Processing package changes...
(1/1) installing nasm                                                                                                          [#############################################################################] 100%
:: Running post-transaction hooks...
(1/1) Arming ConditionNeedsUpdate...
[loomaclin@loomaclin ~]$ nasm --version
NASM version 2.13.02 compiled on Dec 10 2017
[loomaclin@loomaclin ~]$ 
```
</spoiler>

Следующая команда произведёт плоский двоичный файл, результирующий файл будет содержать 24 байта (в `little endian`, если вы работаете на x86 машине):

```
[loomaclin@loomaclin ~]$ cd IdeaProjects/
[loomaclin@loomaclin IdeaProjects]$ mkdir a_minimal_multiboot_kernel
[loomaclin@loomaclin IdeaProjects]$ cd a_minimal_multiboot_kernel/
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ nano multiboot_header.asm
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ nasm multiboot_header.asm 
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ hexdump -x multiboot_header
0000000    50d6    e852    0000    0000    0018    0000    af12    17ad
0000010    0000    0000    0008    0000                                
0000018
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ 
```

#Загрузочный код

Чтобы загрузить наше ядро, мы должны добавить код, который сможет вызвать загрузчик. Давайте создадим файл `boot.asm`:

```
global start

section .text
bits 32
start:
    ; печатает `OK` на экране
    mov dword [0xb8000], 0x2f4b2f4f
    hlt
```

Здесь есть несколько новых комманд:

- `global` экспортирует метки (делает их публичными).  Метка `start` будет входной точкой в наше ядро, она должна быть публичной.
- `.text` секция это секция по-умолчанию для исполняемого кода
- `bits 32` говорит о том, что следующие строки это 32-битные инструкции. Это необходимо потому что процессор ещё находится в [защищённом режиме](https://en.wikipedia.org/wiki/Protected_mode) когда `GRUB` запускает наше ядро. Когда переключимся в [Long mode](https://en.wikipedia.org/wiki/Long_mode) в следующей статье сможем запускать `bits 64` (64-битные инструкции).
- `mov dword` инструкция помещает 32-битную константу `0x2f4b2f4f` в адрес памяти  `b8000` (это выводит `OK` на экран, объяснено будет в следующих статьях)
- `hlt` это инструкция остановки и говорит процессору остановить выполнение комманд

После сборки, просмотра и дизассемблирования мы можем увидеть [опкоды](https://en.wikipedia.org/wiki/Opcode) процессора в действии:

```
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ nano boot.asm
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ nasm boot.asm 
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ hexdump -x boot
0000000    05c7    8000    000b    2f4f    2f4b    00f4                
000000b
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ ndisasm -b 32 boot
00000000  C70500800B004F2F  mov dword [dword 0xb8000],0x2f4b2f4f
         -4B2F
0000000A  F4                hlt
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ 
```

#Создание исполняемого файла

Чтобы загрузить наш исполняемый файл позже через `GRUB,` он должен быть исполняемым `ELF` файлом. Поэтому необходимо, с помощью `nasm` создать `ELF` объектные файлы вместо простых бинарников. Для этого мы просто добавляем в аргументы `-f elf64`.

Для создания самого `ELF` исполняемого кода, мы должны связать объектные файлы. Будем использовать кастомный [скрипт для связывания](https://sourceware.org/binutils/docs/ld/Scripts.html) называемый `linker.ld`:

```
ENTRY(start)

SECTIONS {
    . = 1M;

    .boot :
    {
        /* в начале оставим заголовк мультизагрузки */
        *(.multiboot_header)
    }

    .text :
    {
        *(.text)
    }
}
```

Переведём что написано на человеческий язык:

- `start` это точка входа, загрузчик перейдёт к этой метке после загрузки ядра
- `. = 1M;` уставливает адрес загрузки первой секции с 1-го мегабайта, это стандарт расположения для загрузки ядра
- исполняемая часть имеет две секции: в начале `boot` и `.text` после
- конечная секция `.text` будет  содержать в себе все входящие секции `.text`
- секции именованные как `.multiboot_header` будут добавлены в первую выходную секцию (`.boot`), чтобы они распологались в начале исполняемого кода. Это необходимо потому что `GRUB` ожидает найти заголовок мультизагрузки в начале файла.

Давайте создадим `ELF` объектные файлы и слинкуем их используя вышеуказанный линкер скрипт:

```
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ nasm -f elf64 multiboot_header.asm
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ nasm -f elf64 boot.asm 
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ ld -n -o kernel.bin -T linker.ld multiboot_header.o boot.o
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ 
```

Очень важно передать `-n` (или `--nmagic`) флаг линкеру, который отключает автоматическое выравнивание секций в исполняемом файле. В противном случае линкер может выравнить страницу секции `.boot` в исполняемом файле. Если это произойдёт, `GRUB` не сможет найти заголовок мультизагрузки потому что он будет находится уже не в начале.

Воспользуемся командой `objdump` для того чтобы вывести секции сгенерированного исполняемого файла и проверить, что `.boot` секция имеет наименьшее смещение в файле:

```
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ objdump -h kernel.bin 

kernel.bin:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .boot         00000018  0000000000100000  0000000000100000  00000080  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  1 .text         0000000b  0000000000100020  0000000000100020  000000a0  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ 
``` 

> Примечание: комманды `ld` и `objdump` платформо-зависимы. Если вы работает не на x86_64 архитектуре, вы нуждаетесь в [кросс компиляции binutils](https://os.phil-opp.com/cross-compile-binutils/). После этого воспользуйтесь  `x86_64‑elf‑ld` и `x86_64‑elf‑objdump` соответственно, вместо `ld` и `objdump`.

#Создание ISO-образа

Все персональные компьютеры работающие на базе `BIOS` знают как загружаться с компакт-диска, так что нам необходимо создать загружаемый образ компакт-диска, содержащий наше ядро и файлы загрузчика `GRUB` в единственном файле называемом [ISO](https://en.wikipedia.org/wiki/ISO_image). Создайте следующую структуру директорий и скопируйте `kernel.bin` в директорию `boot`:

```
isofiles
└── boot
    ├── grub
    │   └── grub.cfg
    └── kernel.bin
```

`grub.cfg` указывает имя файла нашего ядра и совместимость с `multiboot 2`. Выглядит это так:

```
set timeout=0
set default=0

menuentry "my os" {
    multiboot2 /boot/kernel.bin
    boot
}
```

Исполняем комманды:
```
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ mkdir isofiles
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ mkdir isofiles/boot
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ mkdir isofiles/boot/grub
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ cp kernel.bin isofiles/boot/
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ nano grub.cfg
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ cp grub.cfg isofiles/boot/grub/
```

Теперь мы можем создать загружаемый образ используя следующую комманду:

```
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ grub-mkrescue -o os.iso isofiles
xorriso 1.4.8 : RockRidge filesystem manipulator, libburnia project.

Drive current: -outdev 'stdio:os.iso'
Media current: stdio file, overwriteable
Media status : is blank
Media summary: 0 sessions, 0 data blocks, 0 data, 7675m free
Added to ISO image: directory '/'='/tmp/grub.jN4u6m'
xorriso : UPDATE : 898 files added in 1 seconds
Added to ISO image: directory '/'='/home/loomaclin/IdeaProjects/a_minimal_multiboot_kernel/isofiles'
xorriso : UPDATE : 902 files added in 1 seconds
xorriso : NOTE : Copying to System Area: 512 bytes from file '/usr/lib/grub/i386-pc/boot_hybrid.img'
ISO image produced: 9920 sectors
Written to medium : 9920 sectors at LBA 0
Writing to 'stdio:os.iso' completed successfully.
```

> Примечание: вызов `grub-mkrescue` может вызвать проблемы на некоторых платформах. Если она у вас не сработала попробуйте следующие шаги:
- запустить команду с `--verbose`
- удостовертись, что библиотека `xorriso` установлена (`xorriso` или `libisoburn` пакет)


<spoiler title="На `Archlinux пришлось поставить `libisoburn`">
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ yaourt xorriso
1 extra/libisoburn 1.4.8-2
    frontend for libraries libburn and libisofs
==> Enter n° of packages to be installed (e.g., 1 2 3 or 1-3)
==> ---------------------------------------------------------
==> 1

[sudo] password for loomaclin: 
resolving dependencies...
looking for conflicting packages...

Packages (3) libburn-1.4.8-1  libisofs-1.4.8-1  libisoburn-1.4.8-2

Total Download Size:   1.15 MiB
Total Installed Size:  3.09 MiB

:: Proceed with installation? [Y/n] 
:: Retrieving packages...
 libburn-1.4.8-1-x86_64                                                                               259.7 KiB   911K/s 00:00 [#############################################################################] 100%
 libisofs-1.4.8-1-x86_64                                                                              237.8 KiB  2.04M/s 00:00 [#############################################################################] 100%
 libisoburn-1.4.8-2-x86_64                                                                            683.8 KiB  2.34M/s 00:00 [#############################################################################] 100%
(3/3) checking keys in keyring                                                                                                 [#############################################################################] 100%
(3/3) checking package integrity                                                                                               [#############################################################################] 100%
(3/3) loading package files                                                                                                    [#############################################################################] 100%
(3/3) checking for file conflicts                                                                                              [#############################################################################] 100%
(3/3) checking available disk space                                                                                            [#############################################################################] 100%
:: Processing package changes...
(1/3) installing libburn                                                                                                       [#############################################################################] 100%
(2/3) installing libisofs                                                                                                      [#############################################################################] 100%
(3/3) installing libisoburn      
</spoiler>
-  если вы использует EFI-систему, `grub-mkrescue` попробует создать `EFI` образ по-умолчанию. Вы можете задать аргумент `-d /usr/lib/grub/i386-pc` чтобы избавиться от этого поведения или установить пакет `mtools` и получить работающий `EFI` образ
- на некоторых системах комманда названа `grub2-mkrescue`

#Загрузка

Пришло время загрузить нашу ОС. Для этого используем [QEMU](https://en.wikipedia.org/wiki/QEMU):

```
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ qemu-system-x86_64 -cdrom os.iso 

(qemu-system-x86_64:10878): Gtk-WARNING **: Allocating size to GtkScrollbar 0x7f2337e5a280 without calling gtk_widget_get_preferred_width/height(). How does the code know the size to allocate?

(qemu-system-x86_64:10878): Gtk-WARNING **: Allocating size to GtkScrollbar 0x7f2337e5a480 without calling gtk_widget_get_preferred_width/height(). How does the code know the size to allocate?

(qemu-system-x86_64:10878): Gtk-WARNING **: Allocating size to GtkScrollbar 0x7f2337e5a680 without calling gtk_widget_get_preferred_width/height(). How does the code know the size to allocate?
```

Появится окно эмулятора:
![Окно эмулятора](https://habrastorage.org/webt/7g/fg/tn/7gfgtnzoyal0krrhb9z4xgyvgua.png)

Обратите внимание на зелёный текст `OK` в верхнем левом углу. Если у вас это не работает, посмотрите секцию комментариев.

Резюмируем, что произошло:

1. BIOS загрузил загрузчик (GRUB) из виртуального компакт-диска (ISO)
2. Загрузчик прочёл исполняемый код ядра и нашёл заголовок мультизагрузки
3. Скопировал секцию `.boot` и `.text` в память (по адресу `0x100000` и `0x100020`)
4. Переместился к точке входа (`0x100020`, это можно узнать вызвав `objdump -f`)
5. Ядро вывело на экран текст `OK` зелёным цветом и остановило процессор

Вы так же можете протестировать это на настоящем железе. Необходимо записать получившийся образ  на диск или USB накопитель и загрузиться с него.

#Автоматизация сборки

Сейчас необходимо вызвать 4 комманды в правильном порядке каждый раз, когда мы меняем файл. Это плохо. Давайте автоматизируем этот процесс используя [Makefile](http://mrbook.org/blog/tutorials/make/). Но для начала мы должны создать некоторую чистую структуру директорий для наших исходных файлов для разделения архитектуро-зависимых файлов:

```
…
├── Makefile
└── src
    └── arch
        └── x86_64
            ├── multiboot_header.asm
            ├── boot.asm
            ├── linker.ld
            └── grub.cfg
```
Создаём: 

```
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ mkdir -p src/arch/x86_64
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ cp multiboot_header.asm src/arch/x86_64/
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ cp boot.asm src/arch/x86_64/
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ cp linker.ld src/arch/x86_64/
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ cp grub.cfg src/arch/x86_64/
[loomaclin@loomaclin a_minimal_multiboot_kernel]$ nano Makefile
```
Makefile должен иметь следующий вид:

```makefile
arch ?= x86_64
kernel := build/kernel-$(arch).bin
iso := build/os-$(arch).iso

linker_script := src/arch/$(arch)/linker.ld
grub_cfg := src/arch/$(arch)/grub.cfg
assembly_source_files := $(wildcard src/arch/$(arch)/*.asm)
assembly_object_files := $(patsubst src/arch/$(arch)/%.asm, \
    build/arch/$(arch)/%.o, $(assembly_source_files))

.PHONY: all clean run iso

all: $(kernel)

clean:
    @rm -r build

run: $(iso)
    @qemu-system-x86_64 -cdrom $(iso)

iso: $(iso)

$(iso): $(kernel) $(grub_cfg)
    @mkdir -p build/isofiles/boot/grub
    @cp $(kernel) build/isofiles/boot/kernel.bin
    @cp $(grub_cfg) build/isofiles/boot/grub
    @grub-mkrescue -o $(iso) build/isofiles 2> /dev/null
    @rm -r build/isofiles

$(kernel): $(assembly_object_files) $(linker_script)
    @ld -n -T $(linker_script) -o $(kernel) $(assembly_object_files)

# compile assembly files
build/arch/$(arch)/%.o: src/arch/$(arch)/%.asm
    @mkdir -p $(shell dirname $@)
    @nasm -felf64 $< -o $@
```
Некоторые комментарии (если вы не работали до этого с `make` посмотрите [makefile туториал](http://mrbook.org/blog/tutorials/make/)):

- $(wildcard src/arch/$(arch)/*.asm) выбирает все файлы ассемблера в директории `src/arch/$(arch)`, так что вам не нужно обновлять Makefile при добавлении файлов
- операция `patsubst` для  `assembly_object_files` просто переводит  `src/arch/$(arch)/XYZ.asm` в `build/arch/$(arch)/XYZ.o`
- таргеты сборки`$<` и `$@` это [автоматически выводимые переменные](https://www.gnu.org/software/make/manual/html_node/Automatic-Variables.html)
- если вы используете [кросс-комплированные binutils](https://os.phil-opp.com/cross-compile-binutils/) просто замените `ld` на `x86_64-elf-ld`

Теперь мы можем вызвать `make` и все обновлённые файлы ассемблера будут скопилированы и скомпанованы. Команда `make iso` так же создаёт ISO образ, а `make run` в дополнение запускает QEMU.

#Что дальше?

В [следующей статье](https://os.phil-opp.com/entering-longmode/) мы создадим таблицу страниц и проведём некоторую конфигурацию процессора для переключения в 64-битный [long-mode](https://en.wikipedia.org/wiki/Long_mode).

#Примечания

1) Формула из таблицы `-(magic + architecture + header_length)` создаёт негативное значение которое не влезает в 32 бита. С помощью вычитания из `0x100000000` мы оставляем значение положительным без изменения вычтенного значения. В результате без дополнительного знакового бита результат вмещается в 32 бита и компилятор счастлив :)

2) Мы не хотим загружать ядро по офсету 0x0 так как много специфичной памяти может быть расположено до метки в 1 мегабайт (для примера так называемый VGA буфер по адресу `0xb8000`, который мы используем чтобы вывести `OK` на экран).
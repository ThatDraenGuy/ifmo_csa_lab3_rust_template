# Лабораторная работа №3 по "Архитектуре компьютера" - 🦀 Rust edition 🦀
**Хайкин Олег Игоревич, P33312**  
Вариант - asm | cisc | neum | hw | tick | struct | stream | port | pstr | prob2 | \[4\]char
## Разбор структуры репозитория
Детали о структуре репозитория можно прочитать [здесь](./template.md)

## Язык программирования
```
program ::= terms

terms ::= term
        | terms term

term ::= instruction
       | label_def
       | directive


instruction ::= math_op
              | branch_op
              | alter_op
              | io_op
              | control_op
              | stack_op

math_op ::= math_opcode math_args
math_opcode ::= 'mov'
              | 'add'
              | 'sub'
              | 'cmp'
              | 'shl'
              | 'shr'

math_args ::= reg_id ',' reg_id
            | mem_by_reg_id ',' reg_id
            | reg_id ',' mem_by_reg_id
            | reg_id ',' label_ref
            | label_ref ',' reg_id
            | reg_id ',' immed


branch_op ::= branch_opcode branch_args
branch_opcode ::= 'jmp'
                | 'jz'
                | 'jnz'
                | 'js'
                | 'jns'
branch_args ::= label_ref


alter_op ::= alter_opcode alter_args
alter_opcode ::= 'inc'
               | 'dec'
alter_args ::= reg_id


io_op ::= io_opcode io_args
io_opcode ::= 'in'
            | 'out'
io_args ::= port_id


control_op ::= 'exit'
             | 'div'


stack_op ::= 'ret'
           | 'call' label_ref
           | 'pop' reg_id
           | 'push' push_args
push_args ::= reg_id
            | immed


mem_by_reg_id ::= '[' reg_id ']'
reg_id ::= 'eax'
         | 'ebx'
         | 'ecx'
         | 'edx'
         | 'esp'
         | 'eip'

immed ::= u32
        | i32
        | symbol
port_id ::= u16

directive ::= "word" literal
literal ::= immed
          | string
string ::= \"(\\.|[^\"])*\"

label_ref ::= label_id
label_def ::= label_id ':'
label_id ::= <combination of any english lowercase letter, number or undeerscore>
```

Способы выделения памяти:
- Статически, через директиву word
- Локально на стеке (технически, это можно назвать выделением вручную через ручной вызов инструкций push/pop или смещение стэк-поинтера)

Переменные не существуют.

В качестве литералов (immed-значений) поддерживаются u32/i32 числа и одиночные символы (char). Для директивы word также поддерживаются строковые литералы.


## Организация памяти
Память процессора представлена массивом из машинных слов. Каждое машинное слово занимает 4 байта. Система рассчитана на 32-битные адреса, поддерживая до U32_MAX ячеек в памяти процессора.

Стоит отметить, что такая память достигала бы 16 гигабайт. В целях работы с симуляцией реальный размер памяти был уменьшен до 4096 ячеек. Адресация при этом не изменена и способна поддерживать "полную" память.

Отображение данных и инструкций в память примитивна - все данные и инструкции располагаются последовательно, начиная с начальной ячейки памяти, в порядке, в котором они были заданы с исходном asm-коде.

Программисту доступны:
- 5 регистров (Accumulator, Base, Count, Data, Stack Pointer и Instruction Pointer)
- Память процессора (путём работы с метками, реальными адресами или указателем на стек)

Статические данные располагаются в памяти процессора. Динамические данные располагабтся на стэке. Куча не поддерживается.


## Система команд
Машинное слово занимает 4 байта. Тип данных определяется только интерпретацией программиста - с точки зрения процессора всё является просто набором бит.

Регистры, как и машинные слова, рассчитаны на 4 байта (32 бита). 

Реализована только прямая адресация.

Устройства ввода-вывода подключены к портам процессора. Доступ к ним осуществуляется через id порта. Система поддерживает до 2**16 различных портов.

Прерывания отсутствуют. Поток управления управляется с помощью eip (регистра-указателя на следующую инструкцию).

Инструкции занимают от одного до двух машинных слов. Необходимость в использовании двух машинных слов для одной инструкции появляется из-за размера immed-значений и адресов (оба могут достигать 4-ёх байт, т.е. целого машинного слова, из-за чего места под код операции и тип её аргументов не остаётся)

Инструкции условно делятся на несколько категорий:

### MathOp
"Математические" операции. Эти инструкции принимают 2 аргумента - dest и src. Они выполняют некую операцию над 2-мя операндами (взятыми из dest и src соответственно) и помещают значение в dest. Операндами могут быть:
- пара регистров
- dest - регистр, src - значение в памяти по адресу из регистра
- dest - значение в памяти по адресу из регистра, src - регистр
- dest - регистр, src - значение в памяти по адресу-константе
- dest - значение в памяти по адресу-константе, src - регистр
- dest - регистр, src - immed-значение (литерал)

В этой категории реализованы следующие операции:
- MOV - записывает значение src в dest
- ADD - прибавляет значение src к dest
- SUB - вычитает значение src из dest
- CMP - вычитает значение src из dest, но не изменяет dest, а только выставляет флаги
- SHL - арифметический сдвиг src на dest бит влево
- SHR - арифметический сдвиг src на dest бит вправо

### BranchOp
Операции ветвления. Изменяют значение eip при выполнении какого-то условия. Их аргументом является адрес, куда совершается переход при выполнении условия (представляется меткой).

Условия представлены проверкой значения какого-то из флагов процессора. Реализованы переходы на основе следующих флагов:
- ZF (zero flag) - флаг, отображающий нулевое значение
- SF (sign flag) - флаг, отображающий "знак" значения (наличие единицы в его старшем бите)

В этой категории реализованы следующие операции:
- JMP - безусловный переход
- JZ - переход, если ZF=1
- JNZ - переход, если ZF=0
- JS - переход, если SF=1
- JNS - переход, если SF=0

### AlterOp
Операции альтерации. Изменяют значение регистра, переданного им как аргумент.

В этой категории реализованы следующие операции:
- INC - инкремент регистра
- DEC - декремент регистра

### IoOp
Операции ввода-вывода. Принимают в качестве аргумента id порта, к которму подключено устройство. Выполняют ввод/вывод между выбранным устройством и младшим байтом аккумулятора.

В этой категории реализованы следующие операции:
- IN - чтение байта из устройства в младший байт аккумулятора
- OUT - запись байта из младшего байта аккумулятора в устройство

### ControlOp
Операции управления. Не принимают аргументов и выполняют какую-то "особую" операцию.

В этой категории реализованы следующие операции:
- EXIT - выполняет завершение работы процессора
- DIV - выполняет деление значения в аккумуляторе на значение из base-регистра. Помещает частное в аккумулятор и остаток в дата-регистр.

### StackOp
Операции работы со стеком. Имеют разный набор аргументов и объединены в категорию только по смыслу.

Стек реализован на основе esp-регистра (указателя на вершину стека). При инициализации стек указывает на адрес последней ячейки памяти + 1. Стек "растёт" вниз, путём декремента esp-регистра. При помещении значения в стек esp-регистр декрементируется и значение кладётся в память по адресу из него. При удалении значения из стека, выполняется обращение к памяти по адресу из esp-регистра и его полседующий инкремент.

В этой категории реализованы следующие операции:
- PUSH - кладёт значение в стек, аргументом является регистр или immed-значение.
- POP - достаёт значения из стека и кладёт его в регистр, переданный как аргумент
- CALL - вызов процедуры(функции). Помещает в стек текущее значение eip-регистра и заменяет его адресом, переданным как аргумент, тем самым осуществляя переход в тело процедуры
- RET - выполняет возвращение из процедуры. Достаёт значение из стека и кладёт его в eip. Не принимает аргументов. Эта операция аналогична вызову `pop eip`

## Транслятор
**Входные данные**:
- Путь к файлу с исходным кодом
- Путь к файлу для размещения результата

**Выходные данные**:
- Транслированная программа, размещённая в указанном файле

Вызов транслятора осуществляется следующим образом:  
`./translator <source_file> <target_file>`

Транслятор работает в 2 прохода:

В первом проходе транслятор рассчитывает адреса всех указанных меток.

Во втором проходе транслятор оссуществляет реальный перевод из языка в машинные инструкции, подставляя вместо обращений к меткам их рассчитанные адреса.


## Модель процессора
**Входные данные**:
- Путь к файлу с программой
- Путь к файлу с данными, которые вводятся в процессор (через устройство в порте 0)

**Выходные данные**:
- Вывод данных из процессора (через устройство в порте 1)
- Данные логирования состояния регистров и флагов процессора

Вызов виртуальной мащшины осуществляется следующим образом:  
`./machine <code_file> <input_file>`

### Схема Datapath
```
                    to Decoder
                  ▲     ▲     ▲                       │from Decoder               │from Decoder    ▲to Decoder
                  │     │     │                       │                           │                │
                  │     │     │                       │                           │                │
                  │     │     │                       │                           │                │
                  │     │     │                       │                           │                │
                  │     │     │                       │                           │                │
                  │     │     │                       │ ┌────────────────┐        │                │
                  │     │     │          ┌────────────┼─┤     Memory     │    ┌───▼────┐           │
                  │     │     │          │            │ │                ◄────┤Mem Addr◄──┐        │
                  │     │     │   sel ┌──▼──┐         │ │                │    └────────┘  │        │
                  │     │     │   ────►DEMUX│         │ │                │                │        │
                  │     │     │       └┬─┬─┬┘         │ │                │RD SIG          │        │
                  │     │     │        │ │ │          │ │                ◄────            │        │
                  │     │     └────────┘ │ │          │ │                │                │        │
                  │     │                │ │          │ │                │                │        │
                  │     │     ┌──────────┘ │          │ │                │WR SIG          │        │
                  │     │     │            │          │ │                ◄───             │        │
                  │  ┌──┴─────▼────┐       │          │ │                │                │        │
to/from Ports     │  │  Registers  │       │          │ │                │   ┌──────────┐ │        │
◄─────────────────┼──►             │   ┌───▼───────┐  │ │                ◄───┤Mem In Buf│ │        │
                  │  │             │   │Mem Out Buf│  │ │                │   └─▲────────┘ │        │
                  │  └─▲─────┬─────┘   └────┬──────┘  │ └────────────────┘     │          │        │
                  │    │     │              │         │                        │          │        │
                  │    │     │ ┌────────────┤         │                        │          │        │
                  │    │     │ │            │         │                        │          │        │
                  │    │     ├─┼──────────┐ │ ┌───────┘                        │          │        │
                  │    │     │ │          │ │ │                                │          │        │
                  │    │ sel┌▼─▼┐        ┌▼─▼─▼┐sel                            │          │        │
                  │    │ ───►MUX│        │ MUX ◄───                            │          │        │
                  │    │    └─┬─┘        └─┬───┘                               │          │        │
                  │    │      │            │                                   │          │        │
                  │    │    ┌─▼────────────▼─┐                                 │          │        │
                  │    │    │      ALU       │                                 │          │        │
                  │    │    │                │ALU signals                      │          │        │
                  │    │    │                ◄───────                          │          │        │
                  │    │    │                │                                 │          │        │
                  │    │    │                │                                 │          │        │
                  │    │    └───────┬────────┘                                 │          │        │
                  │    │            │                        ┌─────┐           │          │        │
                  │    │            ├────────────────────────►flags├───────────┼──────────┼────────┘
                  │    │            │                        └─────┘           │          │
                  │    │       ┌────▼────┐sel                                  │          │
                  │    │       │  DEMUX  ◄───                                  │          │
                  │    │       └─┬─┬──┬─┬┘                                     │          │
                  │    │         │ │  │ │                                      │          │
                  │    └─────────┘ │  │ └──────────────────────────────────────┘          │
                  │                │  │                                                   │
                  └────────────────┘  └───────────────────────────────────────────────────┘
```

### Схема ControlUnit
```
                                                                         ┌───────────┐
                                                                         │ microcode │
                                                                         │ decoder   │
                                                                         └───┬──────┬┘
                                                                             │      │
                                                                             │      │
                                                                             │      │
                                                                             │      │
┌───────────────────────────────────────────────────────────────────┐        │      │
│                            Decoder                                │        │      │
│                                                                   │        │      │
│                                                                   │        │      │
│                                                                   │        │      │
│                                                                   │Signals │      │
│                        *чёрная магия*                             ◄────────┘      │
│                                                                   │               │
│                                                                   │               │
│                           operand pipe                            │               │
│                        ┌─────────────────┐                        │               │
│                        │                 │                        │               │
│        ┌───────────────┴──┐            ┌─▼────────────┐           │               │
│        │Instruction buffer│            │Operand buffer│           │               │
│        └─▲────────────────┘            └─▲──────────┬─┘           │               │
│          │                               │          │             │               │
│          │                               │          │             │               │
│          │                               │          │             ├───────────────┤
│          │                               │          │             │               │
│          │  ┌────────────────────────────┘          │             │               │
│          │  │                                       │             │               │
│         ┌┴──┴─┐                                     │             │               │
│         │DEMUX│                                     │             │ flags         │
│         └─▲───┘                                     │             ◄─────────┐     │
│           │                                         │             │         │     │
└───────────┼─────────────────────────────────────────┼─────────────┘         │     │
            │                                         │                       │     │
            │                                         │                       │     │
            │                                         │                       │     │
            │                                         │                       │     │
            │                                         │                       │     │
            │                                         │                       │     │
            │from ALU DEMUX                           ▼to right ALU MUX       │     │
┌───────────┴───────────────────────────────────────────────────────┐         │     │
│                              Datapath                             │         │     │
│                                                                   ├─────────┘     │
│                                                                   │               │
│                                                                   │               │
│                                                                   ◄───────────────┘
│                                                                   │ Signals
│                                                                   │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```



## Тестирование
### Интеграционное тестирование
Для интеграционного тестирования используется библиотека [insta](https://crates.io/crates/insta), предоставляющий возможность тестирования на основе снапшотов.  
Я считаю эту библиотеку наилучшим аналогом golden-тестов в рамках Rust'а. Технически, снапшоты хранят в себе только результаты, а не входные данные, но это исправляется просто ручной обработкой файлов входных данных, настроенной мною.

В директории [inputs](./tests/inputs/) размещаются входные данные для отдельных тестов. Каждый yml файл в этой директории содержит ассемблер-код для транслятора и входные данные для виртуальной машины. Интеграционный тест совершает цикл обработки отдельно для каждого из этих файлов.

Для каждого файла создаётся набор временных файлов для входных данных транслятора и машины и файла с машинным кодом. Выводы в stdout (вывод виртаульной машины) и stderr (данные логирования) также захватываются. По окончанию работы транслятора и машины (не важно, успешной или нет), создаётся специальный снапшот-файл.

В зависимости от настройки фреймоворка для тестирования, новые снапшот-файлы могут:
- мгновенно заменить старые
- отправиться на review diff'а программистом
- удалиться, если они совпадают со старыми (нет изменений)
- "провалить" тест, если существует разница со старыми снапшот-файлами
- и т.д. (лучше про это описывается в рамках документации самого фреймворка)

### CI
[Workflow](./.github/workflows/rust.yml) настроен для Github Actions, и производит следующиие действия:
- Проверка форматирования
- Проверка линтером
- Тестирование
- Сборка

CI настроен для работы при выполнении push и pull request на ветках master и workflow

### Тестовые ассемблер-программы
#### hello
[input-file](./tests/inputs/hello.yml)

[snapshot-file](./tests/snapshots/integration__test@hello.yml.snap)
```asm
hello: word "Hello"

start: 
    mov eax, hello
    call print_string
    exit

print_string:
    mov edx, eax
    mov ecx, [eax]
    jz print_string_end

print_string_main_loop:
    inc edx
    mov eax, [edx]
    mov ebx, 4

print_string_word_loop:
    out 1

    dec ecx
    jz print_string_end

    dec ebx
    jz print_string_main_loop

    shr eax, 8
    jmp print_string_word_loop

print_string_end:
    ret
```

#### cat
[input-file](./tests/inputs/cat.yml)

[snapshot-file](./tests/snapshots/integration__test@cat.yml.snap)
```asm
start: 
    in 0
    cmp eax, 0
    jz end
    out 1
    jmp start
end:
    exit
```

#### hello_user_name
[input-file](./tests/inputs/hello_user_name.yml)

[snapshot-file](./tests/snapshots/integration__test@hello_user_name.yml.snap)
```asm
  prompt: word "What_is_your_name?"
  hello_start: word "Hello,_"
  hello_end: word "!"
  username_buf: word 0 word 0 word 0 word 0

  start: 
    mov eax, prompt
    call print_string
    call print_newline

    mov eax, username_buf
    call read_string

    mov eax, hello_start
    call print_string

    mov eax, username_buf
    call print_string

    mov eax, hello_end
    call print_string

    exit


  read_string:
    push eax
    mov edx, eax
    mov ecx, 0

  read_string_main_loop:
    inc edx
    mov ebx, 0

  read_string_word_loop:
    mov eax, 0
    in 0
    cmp eax, 0
    jz read_string_end
    inc ecx
    push ebx
  read_string_shift_loop:
    cmp ebx, 0
    jz read_string_word_stuff
    shl eax, 8
    dec ebx
    jmp read_string_shift_loop
  read_string_word_stuff:
    add [edx], eax
    pop ebx
    inc ebx
    cmp ebx, 4
    jz read_string_main_loop
    jmp read_string_word_loop

  read_string_end:
    pop eax
    mov [eax], ecx
    ret


  print_newline:
    mov eax, 10
    out 1
    ret


  print_string:
    mov edx, eax
    mov ecx, [eax]
    jz print_string_end

  print_string_main_loop:
    inc edx
    mov eax, [edx]
    mov ebx, 4

  print_string_word_loop:
    out 1

    dec ecx
    jz print_string_end

    dec ebx
    jz print_string_main_loop

    shr eax, 8
    jmp print_string_word_loop

  print_string_end:
    ret
```

#### prob2
[input-file](./tests/inputs/prob2.yml)

[snapshot-file](./tests/snapshots/integration__test@prob2.yml.snap)
```asm
  num: word 1
  result: word 2

  start:
    mov ecx, 3
    mov eax, 2
  loop:
    mov edx, eax
    add eax, [num]
    mov [num], edx
    dec ecx
    jnz loop
    mov ecx, 3
    cmp eax, 4000000
    jns end
    add [result], eax
    jmp loop

  end:
    mov eax, [result]
    call print_uint
    exit


  print_uint:
    mov ebx, 10
    mov ecx, esp

  print_uint_loop:
    dec esp
    mov edx, 0
    div
    add edx, 48
    mov [esp], edx
    cmp eax, 0
    jnz print_uint_loop
  print_uint_print:
    mov eax, [esp]
    out 1
    inc esp
    cmp esp, ecx
    jnz print_uint_print
  print_uint_end:
    ret
```

| ФИО | <алг> | <LoC> | <code байт> | <code инстр.> | <инстр.> | <такт.> | <вариант> |
|-|-|-|-|-|-|-|-|

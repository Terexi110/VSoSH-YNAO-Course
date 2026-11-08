# Как снять 3 реальных скриншота Ghidra

В папке лежит учебный бинарь **`reverse_demo`** (исходник - `reverse_demo.c`). Это пример из
п. 8.5:
- пароль - `secret99`;
- при верном вводе печатается `flag{r3v3rs3_m3}`;
- каждый символ `^ 0x2A` сравнивается с массивом `target[8]`;
- собран с `-no-pie -fno-stack-protector`, **не stripped**: `main` и `check` видны.

## Подготовка (один раз)
1. Ghidra -> *File -> New Project* -> Non-Shared Project.
2. *File -> Import File* -> `reverse_demo` из этой папки -> OK.
3. Двойной клик по файлу -> **Analyze -> Yes** (15-30 сек).
4. Разверни CodeBrowser на весь экран.

## Три кадра (сохрани PNG в эту же папку под ТОЧНО такими именами - они уже прописаны в `ghidra.md`)

### 1. `ghidra-layout.png` - общий вид окна
- В **Symbol Tree -> Functions** видны `main` и `check`.
- По центру **Listing**, справа **Decompile** с `main` (`printf("Enter password: ")`,
  `check(...)`, ветки `Correct!` / `Wrong`).
- Сними всё окно целиком.

### 2. `ghidra-decompile.png` - разбор проверки
- Двойной клик по `check` -> справа её псевдокод.
- Виден `target` с байтами `0x59,0x4f,...`, цикл `^ 0x2A` и сравнение `!= target[i]`.
- Сними панель Decompile крупно.

### 3. `ghidra-strings-xref.png` - строки и переход в код
- **Window -> Defined Strings**.
- Найди `"Enter password: "` и `"Correct! flag{r3v3rs3_m3}"`.
- По желанию: правый клик -> *References -> Show References to...* (XREF в `main`).
- Сними окно Defined Strings вместе с References, если открыл.

## После
Три PNG в `08-reverse/img/` подхватятся текстом главы (`![...](img/ghidra-layout.png)` и т.д.
в `ghidra.md`). Схемы можно оставить как запасной вариант или удалить.

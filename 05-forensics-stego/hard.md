# Глава 5 (hard). Глубокая форензика и стего

Разберём битовые плоскости, чанки PNG, полиглоты, трафик, аудиоспектрограммы и образы дисков.

## 1. LSB по всем битовым плоскостям и каналам

Данные могут лежать не в младшем бите, а в **другой битовой плоскости** или **одном канале**.
Переберём все каналы и биты:

```python
from PIL import Image
import numpy as np, re

img = np.array(Image.open("img.png").convert("RGB"))
h, w, _ = img.shape
for ch in range(3):                         # каналы R,G,B
    for bit in range(8):                     # биты 0..7
        plane = (img[:, :, ch] >> bit) & 1   # вытащить нужный бит каждого пикселя
        data = np.packbits(plane).tobytes()  # мгновенно собрать биты в байты (вместо медленного join)
        m = re.search(rb'flag\{[^}]+\}', data)
        if m:
            print(f"канал={ch} бит={bit}:", m.group())
```

💡 Для PNG/BMP начинай с `zsteg -a`: он перебирает эти и другие варианты. Скрипт удобен
для нестандартного формата или своего порядка обхода.

## 2. Разбор структуры PNG по чанкам

PNG состоит из **чанков**: `IHDR` задаёт размеры, `IDAT` хранит сжатые пиксели, `IEND` завершает файл.
В служебных `tEXt` и `zTXt` бывает текст, а после `IEND` - дописанные данные.

```python
import struct
def png_chunks(data):
    assert data[:8] == b"\x89PNG\r\n\x1a\n", "не PNG"
    i = 8; out = []
    while i < len(data):
        ln  = struct.unpack(">I", data[i:i+4])[0]     # длина (big-endian)
        typ = data[i+4:i+8].decode("latin1")           # тип чанка
        out.append((typ, ln, i))
        i += 12 + ln                                    # 4 длина +4 тип +ln данные +4 CRC
        if typ == "IEND":
            if i < len(data):
                print("!!! данные после IEND:", data[i:i+32])   # спрятанный хвост
            break
    return out

print(png_chunks(open("img.png","rb").read()))
```

⚠️ При **заниженных размерах** часть картинки скрыта. Исправь первые 8 байт данных `IHDR`:
4 байта ширины и 4 высоты, big-endian. Проверяй результат через `pngcheck -v`.

После правки пересчитай CRC `IHDR`, иначе просмотрщик может отвергнуть файл:
```python
import zlib, struct
d = open("img.png", "rb").read()
w, h = 800, 600                                  # настоящие размеры
ihdr = struct.pack(">II", w, h) + d[24:29]       # width,height + остальные 5 байт IHDR
crc  = zlib.crc32(b"IHDR" + ihdr)                # CRC = от типа чанка + его данных
fixed = d[:16] + ihdr + struct.pack(">I", crc) + d[33:]
open("fixed.png", "wb").write(fixed)
```

## 3. Полиглоты и приклеенные файлы

Полиглот читается как два формата; другой приём - дописать данные после конца файла:

```bash
binwalk файл                                # обзор вложений
foremost -i файл -o out                      # вырезать по сигнатурам
# вручную: найти начало вложенного zip и вырезать
python3 -c "d=open('f','rb').read(); i=d.find(b'PK\x03\x04'); open('x.zip','wb').write(d[i:])"
```
Конец JPEG - `FF D9`, PNG - чанк `IEND` вместе с CRC, GIF - трейлер `3B` (часто после `00`).
Данные за ними стоит проверить.

## 4. Глубокий анализ трафика (tshark)

```bash
tshark -r d.pcap -Y 'http.request' -T fields -e http.host -e http.request.uri   # все URL
tshark -r d.pcap -Y 'dns' -T fields -e dns.qry.name                              # DNS-запросы (эксфильтрация в поддоменах!)
tshark -r d.pcap -Y 'ftp' -T fields -e ftp.request.command -e ftp.request.arg   # логины FTP
tshark -r d.pcap --export-objects http,out_dir                                   # вытащить переданные файлы
tshark -r d.pcap -qz follow,tcp,ascii,3                                          # тело TCP-потока №3
tshark -r usb.pcap -T fields -e usbhid.data -e usb.capdata                        # нажатия USB-клавиатуры/мыши (HID)
```

💡 Собери разбитые по пакетам данные через Follow Stream или `export-objects`, затем сними
Base64/hex по главе 4. Утечка может идти через **DNS-запросы** (`секрет.attacker.com`) или ICMP.

## 5. Аудио-стего: спектрограмма своими руками

Надпись в спектре `.wav` можно увидеть на спектрограмме. Пример для 16-битного PCM WAV:

```python
import numpy as np, wave, matplotlib
matplotlib.use("Agg"); import matplotlib.pyplot as plt
w = wave.open("a.wav"); fr = w.getframerate(); ch = w.getnchannels()
sig = np.frombuffer(w.readframes(w.getnframes()), dtype=np.int16)[::ch]  # [::ch] -> один канал (стерео в моно)
plt.specgram(sig, Fs=fr); plt.savefig("spec.png")       # открыть spec.png - флаг виден надписью
```
Либо в GUI: **Audacity** / **Sonic Visualiser** -> режим спектрограммы.

## 6. Образы дисков и файловых систем

В образе диска (`.img`/`.dd`/`.raw`) ищи файловую систему и удалённые файлы.
Работай с копией, монтируй только на чтение:

```bash
7z l disk.img                      # просто посмотреть содержимое (часто уже хватает)
mount -o loop,ro disk.img /mnt     # примонтировать только на чтение
# пакет sleuthkit - анализ ФС и удалённых файлов:
fls -r disk.img                    # список файлов, включая удалённые (помечены *)
icat disk.img <inode>              # вытащить файл по номеру inode
```
Для образа всего диска сначала найди разделы: `mmls disk.img`. В `fls` и `icat` укажи
`-o <начальный_сектор>` нужного раздела; напрямую монтируют образ файловой системы.

(Перебор пароля `steghide` через `stegseek` разобран в `medium.md`, п. 5.5.)

🎯 **Тренировка (hard).** (а) Зашей флаг в бит №3 зелёного канала и найди перебором каналов и битов.
(б) Разбери чанки PNG, допиши текст после `IEND` и проверь, что скрипт его находит.

---
*Уровень: **hard**. Рядом: [easy](easy.md) | [medium](medium.md) | [оглавление](../00-about.md)*

# Silent Breach — CyberDefenders Writeup

## Информация о лабе

- **Платформа:** CyberDefenders
- **Ссылка:** [CyberDefenders — Silent Breach](https://cyberdefenders.org/blueteam-ctf-challenges/silent-breach/)
- **Название:** Silent Breach
- **Категория:** Endpoint Forensics
- **Сложность:** Medium
- **Формат улик:** Образ диска Windows-машины (`.ad1`)

## Сценарий

Сотрудник IMF скачал файл с секретной информацией, но тот оказался зашифрован. В роли аналитика нужно разобраться, как malware это сделал, и восстановить данные.

## Инструменты

- `ad1-forensic` (Rust) / `pyad1` — чтение AD1-образа
- `sqlite3` — просмотр истории браузера
- `strings` — извлечение текста из бинарников
- CyberChef — декодирование Reverse + Base64
- Python + `pycryptodome` — расшифровка AES

## Ход решения

### 1. Подозрительный файл

Смонтировал образ `ethanPC.ad1`. В папке `Downloads` обнаружил файл с двойным расширением:

```
IMF-Info.pdf.exe
```

Он маскируется под PDF, но является исполняемым файлом. MD5-хеш этого файла — ответ на первый вопрос.

### 2. Источник заражения

Рядом с файлом (или внутри него) найден альтернативный поток данных NTFS — `Zone.Identifier`. Внутри него хранится URL, откуда файл был скачан:

```
http://192.168.16.128:8000/IMF-Info.pdf.exe
```

Это ответ на второй вопрос.

### 3. Браузер

В `AppData\Local` находятся папки браузеров. Внутри каждой — файл `History` (база SQLite). Открыл через `sqlite3`:

```bash
sqlite3 History "SELECT * FROM downloads;"
```

Запись о скачивании `IMF-Info.pdf.exe` найдена в истории **Microsoft Edge**. Это ответ на третий вопрос.

### 4. Письмо с IP-адресами

Файл `HxStore.hxd` (хранилище Windows Mail) лежит в:

```
AppData\Local\Packages\microsoft.windowscommunicationsapps_8wekyb3d8bbwe\LocalState\
```

Извлёк его и прогнал через `strings`:

```bash
strings HxStore.hxd | grep -E '([0-9]{1,3}\.){3}[0-9]{1,3}'
```

Нашёл три IP-адреса, упомянутых в одном письме. Это ответ на четвёртый вопрос.

### 5. Пароль для расшифровки

Извлёк `IMF-Info.pdf.exe` из образа. Прогнал через `strings`:

```bash
strings IMF-Info.pdf.exe > strings_output.txt
```

В выводе нашёл фрагмент PowerShell-кода, где строка сначала **переворачивается** (Reverse), а затем декодируется из **Base64**.

Скопировал закодированную строку и вставил в **CyberChef**. Применил операции:
1. **Reverse** (by character)
2. **From Base64**

В результате получил читаемый PowerShell-скрипт. Внутри нашёл переменную с паролем:

```
Imf!nfo#2025Sec$
```

Это ответ на пятый вопрос.

### 6. Расшифровка файлов

В том же PowerShell-скрипте найдены параметры шифрования:

- **Алгоритм:** AES
- **Режим:** CBC
- **Паддинг:** PKCS7
- **Соль:** `[0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08]`
- **Итерации:** `10000`
- **Размер ключа:** `32` байта
- **Размер IV:** `16` байт

Ключ и IV генерируются из пароля через `Rfc2898DeriveBytes` (PBKDF2 в .NET).

Написал Python-скрипт для расшифровки:

```python
from Crypto.Cipher import AES
from Crypto.Protocol.KDF import PBKDF2
from Crypto.Util.Padding import unpad
import os

password = b"Imf!nfo#2025Sec$"
salt = bytes([0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08])
iterations = 10000
key_size = 32
iv_size = 16

derived = PBKDF2(password, salt, key_size + iv_size, count=iterations)
key = derived[:key_size]
iv = derived[key_size:]

files_to_decrypt = ["IMF-Secret.enc", "IMF-Mission.enc"]

for file_name in files_to_decrypt:
    if not os.path.exists(file_name):
        print(f"Файл не найден: {file_name}")
        continue

    with open(file_name, "rb") as f:
        ciphertext = f.read()

    cipher = AES.new(key, AES.MODE_CBC, iv)
    try:
        plaintext = unpad(cipher.decrypt(ciphertext), AES.block_size)
    except ValueError as e:
        print(f"Ошибка расшифровки {file_name}: {e}")
        continue

    output_name = file_name.replace(".enc", ".pdf")
    with open(output_name, "wb") as f:
        f.write(plaintext)
    print(f"Расшифрован: {file_name} -> {output_name}")
```

После запуска скрипта получил `IMF-Mission.pdf`. Внутри — флаг:

```
CyberDefenders{N3v3r_eX3cuTe_F!l3$_dOwnL0ded_fr0m_M@lic10u5_$erV3r}
```

## Выводы

- Malware распространялся через файл с двойным расширением (`IMF-Info.pdf.exe`), скачанный с локального IP.
- Для сокрытия полезной нагрузки использовалась **Reverse + Base64** обфускация PowerShell-скрипта.
- Файлы шифровались алгоритмом **AES-CBC** с ключом, производным от пароля через PBKDF2.
- Пароль был жёстко зашит в скрипте и извлекался после декодирования.
- Расшифровка возможна при повторении логики генерации ключа и IV.

## Что потренировал

- Работа с AD1-образами (`ad1-forensic`, `pyad1`)
- Понимание альтернативных потоков данных NTFS (`Zone.Identifier`)
- Анализ SQLite-баз (история браузера)
- Извлечение строк из бинарников через `strings`
- Reverse + Base64 декодирование в CyberChef
- Повторение логики AES-шифрования на Python
- Работа с PBKDF2 для генерации ключа и IV

## Ссылки

- [CyberDefenders — Silent Breach](https://cyberdefenders.org/blueteam-ctf-challenges/silent-breach/)

---

*Writeup написан в образовательных целях. Все действия выполнялись в изолированной лабораторной среде.*

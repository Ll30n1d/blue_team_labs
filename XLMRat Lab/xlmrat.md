# XLMRat — Writeup

## Информация о лабе

- **Платформа:** CyberDefenders
- **Ссылка:** [CyberDefenders — XLMRat](https://cyberdefenders.org/blueteam-ctf-challenges/xlmrat/)
- **Название:** XLMRat
- **Категория:** Network Forensics / Malware Analysis
- **Сложность:** Easy
- **Формат улик:** PCAP-файл с сетевым трафиком

## Инструменты

- Wireshark — анализ PCAP
- CyberChef — декодирование hex-строк
- sha256sum (Linux) — вычисление хеша
- VirusTotal — проверка хеша и определение семейства malware

## Ход решения

### 1. Анализ трафика

Открыл PCAP в Wireshark. Отфильтровал HTTP-трафик (`http.request`). Обнаружил два скачанных файла:

- `doc-0321.doc` — приманка (doc-файл)
- `mdm.jpg` — файл, маскирующийся под картинку

Первый скачанный файл — `doc-0321.doc`. Его URL и есть ответ на первый вопрос.

### 2. Информация о хостинге

Взял IP-адрес из URL, откуда скачивался файл. Проверил через WHOIS. Провайдер — **Constant Hosting**.

### 3. Извлечение payload

В HTTP-потоке файла `mdm.jpg` обнаружил скрипт с двумя hex-переменными:

- `$hexString_pe` — загрузчик (loader)
- `$hexString_bbb` — основной исполняемый файл malware

**Важно:** для этого лаба нужен именно `$hexString_bbb`. Я сначала взял `$hexString_pe` и получил неверный хеш (`029c...`). После замены переменной на `$hexString_bbb` хеш совпал с ожидаемым.

Действия:

1. Скопировал значение `$hexString_bbb` из потока Wireshark.
2. В CyberChef убрал символы `_` (Find / Replace → `_` → пусто).
3. Применил операцию **From Hex**.
4. Скачал полученный бинарник.
5. Посчитал SHA256:

```bash
sha256sum payload.exe
```

Полученный хеш:
1eb7b02e18f67420f42b1d94e74f3b6289d92672a0fb1786c30c03d68e81d798


### 4. Определение семейства malware

Загрузил свой .exe файл SHA256 в VirusTotal. В списке детекций нашёл вендора **Alibaba**. Вердикт — **AsyncRat** (RAT для удалённого доступа).

### 5. Время создания файла

На странице VirusTotal во вкладке **Details** посмотрел поле **Creation Time**. Это время компиляции исполняемого файла.

### 6. LOLBin

Вернулся в Wireshark, в тот же HTTP-поток `mdm.jpg`. В скрипте нашёл путь к legitimate Windows-бинарнику, который malware использует для запуска. Это **`C:\Windows\Microsoft.NET\Framework\v4.0.30319\RegAsm.exe`**.

### 7. Сброшенные файлы

В том же скрипте нашёл имена файлов, которые создаются на диске. Три файла с одинаковым базовым именем, но разными расширениями (`.ps1`, `.bat`, `.vbs`).

## Выводы

- Malware доставлялся через doc-файл-приманку и замаскированный под картинку скрипт.
- Внутри `mdm.jpg` лежал hex-кодированный payload, который декодировался в исполняемый файл.
- Payload — **AsyncRat**, семейство RAT для удалённого доступа.
- Для запуска использовался LOLBin `RegAsm.exe` — legitimate-бинарник .NET Framework.
- На диск сбрасывались скрипты для персистентности и запуска.

## Что потренировал

- Фильтрация HTTP-трафика в Wireshark
- Работа с HTTP-потоками и извлечение данных из скриптов
- Декодирование hex через CyberChef
- Вычисление SHA256 и работа с VirusTotal
- Понимание разницы между загрузчиком и полезной нагрузкой
- Знакомство с LOLBin-техникой

## Ссылки

- [CyberDefenders — XLMRat](https://cyberdefenders.org/blueteam-ctf-challenges/xlmrat/)
- [VirusTotal](https://www.virustotal.com/)
- [CyberChef](https://gchq.github.io/CyberChef/)

---

*Writeup написан в образовательных целях. Все действия выполнялись в изолированной лабораторной среде.*

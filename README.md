Лабораторная работа: фаззинг-тестирование `jpegoptim` с использованием AFL++ и анализ покрытия

0. Цель работы

Цель работы — провести фаззинг-тестирование консольной утилиты `jpegoptim`, написанной на языке C, с использованием фреймворка AFL++, а также выполнить анализ покрытия кода при помощи инструментов `gcov` и `lcov`. В ходе выполнения работы требуется:

- собрать целевую программу с инструментированием AFL++;
- запустить фаззинг-тестирование и проанализировать статистику AFL++;
- собрать программу с флагами покрытия GCC, прогнать через неё набор входов из AFL++;
- получить и проанализировать отчёты покрытия (`lcov`, HTML-репорты);
- дополнительно получить карты покрытия путей при помощи `afl-showmap`.

---

1. Объект тестирования

В качестве объекта фаззинг-тестирования используется утилита `jpegoptim` — реальная open-source программа для оптимизации JPEG-файлов.

Основные характеристики:

- язык реализации: C;
- тип программы: консольная утилита;
- тип входных данных: JPEG-файлы;
- формат запуска:
  ```bash
  jpegoptim [опции] файл1.jpg файл2.jpg ...

2. Подготовка окружения
Работа выполнялась в среде Linux (семейство Debian/Ubuntu). Для сборки AFL++, jpegoptim и анализа покрытия были установлены следующие пакеты:
sudo apt update
sudo apt install -y \
    git build-essential autoconf automake libtool \
    libjpeg-dev lcov \
    llvm-18 llvm-18-dev clang-18 lld-18 \
    gcc g++

<img width="619" height="220" alt="Снимок экрана 2025-11-26 в 12 46 41" src="https://github.com/user-attachments/assets/4b96ef17-3129-4fed-a5f5-ae19a1bf4ca3" />


3. Сборка AFL++
Переход в каталог и сборка AFL++:
mkdir -p ~/ssdlfuzz
cd ~/ssdlfuzz

git clone https://github.com/AFLplusplus/AFLplusplus.git
cd AFLplusplus

make LLVM_CONFIG=llvm-config-18

4. Сборка jpegoptim с инструментированием AFL++
4.1. Клонирование и подготовка исходников
   cd ~/ssdlfuzz
git clone https://github.com/tjko/jpegoptim.git
cd jpegoptim
make clean || true

4.2. Конфигурация и сборка с использованием AFL-компилятора
Конфигурирование с указанием AFL-компиляторов:
CC=../AFLplusplus/afl-cc \
CXX=../AFLplusplus/afl-c++ \
./configure

Сборка:
make CC=../AFLplusplus/afl-cc \
     CXX=../AFLplusplus/afl-c++ \
     LD=../AFLplusplus/afl-cc

4.3. Подсчёт контрольной суммы бинарника
Для фиксации версии и воспроизводимости эксперимента рассчитывается SHA-256 хэш бинарного файла:
cd ~/ssdlfuzz/jpegoptim
sha256sum jpegoptim

<img width="633" height="89" alt="Снимок экрана 2025-11-26 в 12 35 15" src="https://github.com/user-attachments/assets/7c53d048-0491-475d-bd27-77286c61d655" />

4.4. Проверка наличия AFL-инструментации
Проверка наличия строк, связанных с AFL, в бинарнике:
strings jpegoptim | grep afl || true

5. Подготовка входного корпуса для фаззинга
Для начального корпуса фаззинга создаётся директория in и наполняется набором JPEG-файлов:
cd ~/ssdlfuzz
mkdir -p in

# Использование тестовых файлов из репозитория jpegoptim
cp jpegoptim/test/*.jpg in/ 2>/dev/null || true

6. Фаззинг-тестирование jpegoptim с помощью AFL++
6.1. Запуск AFL++
Фаззинг выполняется командой:
cd ~/ssdlfuzz/AFLplusplus

./afl-fuzz -i ../in -o ../out -- ../jpegoptim/jpegoptim @@

<img width="686" height="494" alt="Снимок экрана 2025-11-26 в 12 43 30" src="https://github.com/user-attachments/assets/e0ba1e30-11c9-4eec-89a6-f2e04aefcd9f" />

6.2. Анализ статистики fuzzer_stats
После завершения (или по истечении времени фаззинга) в каталоге out/default находится файл fuzzer_stats:
cd ~/ssdlfuzz/out/default
cat fuzzer_stats

<img width="671" height="835" alt="Снимок экрана 2025-11-26 в 12 36 48" src="https://github.com/user-attachments/assets/88ab2023-cc20-42f3-ad29-e26ce1dab413" />

6.3. Построение графиков при помощи afl-plot
Для наглядного представления динамики фаззинга запускается afl-plot:
cd ~/ssdlfuzz/AFLplusplus
./afl-plot ../out/default ../plots

<img width="1000" height="200" alt="low_freq" src="https://github.com/user-attachments/assets/068e3d60-df38-46bf-817e-d1805902df5e" />
<img width="1000" height="300" alt="high_freq" src="https://github.com/user-attachments/assets/381b5eda-7495-4a14-88ed-434de3fff3b4" />
<img width="1000" height="200" alt="exec_speed" src="https://github.com/user-attachments/assets/9575934f-0061-4d6a-956c-aa1655795494" />
<img width="1000" height="300" alt="edges" src="https://github.com/user-attachments/assets/8045b6c3-102c-4681-8af3-5ab8bc8b7954" />

7. Сбор покрытия кода с использованием GCC, gcov и lcov
На данном этапе jpegoptim пересобирается с флагами покрытия GCC, после чего через инструментированную версию прогоняется очередь входов, полученная AFL++.
7.1. Пересборка jpegoptim с флагами покрытия
   cd ~/ssdlfuzz/jpegoptim

make clean

./configure CC="gcc" CFLAGS="-fprofile-arcs -ftest-coverage -O0 -g" \
            LDFLAGS="--coverage"

make

7.2. Очистка старых файлов профилей
Перед запуском тестов удаляются старые файлы .gcda:
cd ~/ssdlfuzz/jpegoptim
find . -name "*.gcda" -delete

7.3. Прогон входов из очереди AFL++
Для получения фактического покрытия инструментированная версия jpegoptim запускается на всех тестах, накопленных AFL++ в очереди:
cd ~/ssdlfuzz/jpegoptim

for f in ../out/default/queue/id:*; do
    ./jpegoptim -d /tmp "$f" >/dev/null 2>&1
done

7.4. Сбор отчёта покрытия через lcov
Создание файла покрытия coverage.info:
cd ~/ssdlfuzz/jpegoptim

lcov --capture --directory . --output-file coverage.info

<img width="1656" height="468" alt="Снимок экрана 2025-11-26 в 12 39 01" src="https://github.com/user-attachments/assets/dee7405c-eae5-4ecd-b4f3-60d65efca72c" />

7.5. Генерация HTML-отчёта покрытия
Для удобного анализа покрытия в браузере формируется HTML-отчёт:
genhtml coverage.info --output-directory coverage_html

<img width="1588" height="296" alt="Снимок экрана 2025-11-26 в 12 40 56" src="https://github.com/user-attachments/assets/de5f26cc-7470-4bd6-bd9e-3eb4d6dcdeb4" />

8. Анализ карт покрытия путей при помощи afl-showmap
Для дополнительного анализа путей, достигаемых различными входами, используется afl-showmap.
8.1. Генерация .map-файлов
   cd ~/ssdlfuzz/AFLplusplus

mkdir -p ../maps

for f in ../out/default/queue/id_*; do
    base=$(basename "$f")
    ./afl-showmap -o "../maps/$base.map" -m none -- \
        ../jpegoptim/jpegoptim "$f" >/dev/null 2>&1
done




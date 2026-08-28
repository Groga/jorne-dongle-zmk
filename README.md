# Jorne + донгл на ZMK (Pro Micro nRF52840)

Версия: 1.0.0
Автор: Evgeniy Moskovkin
Дата: 2026-08-28

Схема: обе половинки Jorne (на чипах ProMicro-nRF52840, например nRFMicro,
nice!nano, SuperMini nRF52840) работают как BLE-периферия, а третье
отдельное устройство на таком же чипе nRF52840 — **донгл**, вставленный
в USB компьютера, — является BLE-центром и единственным, что видит хост.

## Почему так делают
- обе половинки клавиатуры не тратят батарею на удержание BLE-связи с ОС,
  им нужен только короткий линк с донглом → заряда хватает намного дольше;
- ноутбучный Bluetooth часто "тупит" при разрыве/восстановлении соединения
  сплит-клавиатуры — донгл с адаптером USB работает стабильнее;
- смена компьютера = просто переткнуть донгл, без переспаривания половинок.

## Что нужно
1. Три платы на nRF52840 в форм-факторе Pro Micro (2 в половинках Jorne +
   1 как донгл, воткнутая в любой корпус/переходник или просто голая в USB).
2. Официальный shield `jorne` уже входит в апстрим ZMK — отдельно тянуть
   его не нужно, `west.yml` в этом репозитории просто подключает сам ZMK.
3. Собственный shield `jorne_dongle` — лежит в
   `config/boards/shields/jorne_dongle/` этого репозитория.

## Структура репозитория
```
jorne-dongle-zmk/
├── build.yaml                         # матрица сборки GitHub Actions
└── config/
    ├── west.yml                       # манифест west (подключает ZMK)
    └── boards/shields/jorne_dongle/
        ├── Kconfig.shield             # регистрация shield'а
        ├── Kconfig.defconfig          # донгл = ZMK_SPLIT_ROLE_CENTRAL
        ├── jorne_dongle.overlay       # физраскладка без реальной матрицы
        ├── jorne_dongle.conf          # доп. настройки донгла
        └── jorne_dongle.keymap        # каркас — перенесите сюда layers
```

## Шаг 1 — доделать keymap донгла (обязательно!)
Логику слоёв и клавиш теперь обрабатывает **донгл**, а не левая половинка.
Откройте официальный `jorne.keymap`:
https://github.com/zmkfirmware/zmk/blob/main/app/boards/shields/jorne/jorne.keymap
(или свой изменённый вариант) и перенесите блоки `behaviors { }` и
`keymap { }` в `config/boards/shields/jorne_dongle/jorne_dongle.keymap`,
заменив заглушку `default_layer`.

Также сверьте `jorne_dongle.overlay` с актуальным `jorne.dtsi`
(https://github.com/zmkfirmware/zmk/tree/main/app/boards/shields/jorne) —
имя chosen-раскладки может отличаться между версиями ZMK, поправьте при
необходимости `zmk,physical-layout = &default_layout;`.

## Шаг 2 — указать реальную плату
В `build.yaml` замените `nice_nano_v2` на вашу плату:
| Чип/плата                     | Имя board в ZMK       |
|--------------------------------|------------------------|
| nice!nano v2                   | `nice_nano_v2`         |
| nRFMicro 1.3+                  | `nrfmicro_13`          |
| Seeed XIAO BLE (nRF52840)      | `seeeduino_xiao_ble`   |
| SuperMini nRF52840             | `nice_nano_v2` (pin-совместим) |

Половинки и донгл могут быть на разных платах — главное, чтобы у каждой
были реальные GPIO под то, что ей поручено (у половинок — матрица клавиш,
у донгла — почти ничего, только USB).

## Шаг 3 — собрать прошивки
### Вариант А — GitHub Actions (проще всего)
1. Создайте свой репозиторий на GitHub и залейте туда содержимое этой
   папки.
2. Включите Actions во вкладке Settings → Actions.
3. Запушьте любой коммит — сборка запустится автоматически по
   `build.yaml`, на выходе (Artifacts) вы получите 4 `.uf2`:
   - `jorne_dongle.uf2` — прошивка донгла;
   - `jorne_left_peripheral.uf2` — левая половинка;
   - `jorne_right_peripheral.uf2` — правая половинка;
   - `settings_reset.uf2` — сброс BLE-пар (см. Шаг 5).

### Вариант Б — локальная сборка (west)
```bash
# один раз: настройка окружения Zephyr/west
pip install west --break-system-packages
west init -l config
west update
west zephyr-export

# донгл (центр)
west build -p -b nice_nano_v2 -- -DSHIELD=jorne_dongle

# левая половинка (периферия!)
west build -p -b nice_nano_v2 -d build_left -- \
    -DSHIELD=jorne_left -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n

# правая половинка (периферия, как и в обычной схеме)
west build -p -b nice_nano_v2 -d build_right -- -DSHIELD=jorne_right
```
Файлы прошивок появятся в `build*/zephyr/zmk.uf2`.

## Шаг 4 — прошить
На каждой плате дважды быстро нажмите кнопку Reset (или замкните RST-GND
пинцетом дважды) — появится USB-накопитель (обычно `NICENANO`/`NRF52BOOT`).
Перетащите на него соответствующий `.uf2`-файл. Так по очереди — донгл,
левая, правая.

## Шаг 5 — сопряжение (первый запуск)
1. Прошейте `settings_reset.uf2` на всех трёх платах по очереди — это
   очистит старые BLE-связи (важно, если половинки раньше уже были
   спарены как обычная схема "левая = центр").
2. Прошейте боевые прошивки: донгл, левая, правая.
3. Воткните донгл в USB. Включите половинки — они должны автоматически
   найти и связаться с донглом (донгл — единственное BLE-устройство,
   которое их "ждёт").
4. Хост увидит только донгл как одну HID-клавиатуру — это нормально и
   есть цель всей схемы.

## Диагностика
- Половинка не коннектится к донглу → пересоберите её с
  `-DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n` (в build.yaml это уже задано,
  проверьте, что артефакт `jorne_left_peripheral` собрался с этим флагом).
- Донгл не виден компьютером как клавиатура → проверьте, что в
  `Kconfig.defconfig` донгла `ZMK_USB` включён (`default y`), а сам чип
  реально поддерживает нативный USB (все перечисленные платы — да).
- Раскладка "съехала" / клавиши не те → скорее всего в
  `jorne_dongle.overlay` физраскладка не совпадает с той, что использует
  `jorne_left`/`jorne_right` — сверьтесь заново с апстримным `jorne.dtsi`.

## Визуальное отображение раскладки

В репозитории уже есть `.github/workflows/keymap-drawer.yml` — он использует
инструмент [keymap-drawer](https://github.com/caksoylar/keymap-drawer) и при
каждом пуше, меняющем `jorne_dongle.keymap`, автоматически:
1. разбирает keymap на слои/behaviors/комбо;
2. рисует SVG-картинку итоговой раскладки;
3. коммитит её в папку `keymap-drawer/` вашего репозитория.

После первого пуска экшена вставьте картинку в README командой:
```markdown
![keymap](keymap-drawer/jorne_dongle.svg)
```

Если хотите **редактировать раскладку визуально мышкой** (а не руками в
devicetree-синтаксисе), используйте веб-редактор
https://nickcoutsos.github.io/keymap-editor — он умеет открывать ваш GitHub
репозиторий напрямую (авторизация через GitHub), показывает клавиатуру
графически, а изменения сохраняет обратно коммитом в `jorne_dongle.keymap`.
После такого коммита сборка (build.yaml) и отрисовка SVG (keymap-drawer)
запустятся автоматически.

## Полезные готовые примеры для сверки
- https://github.com/aroum/zmk-enki42-dongle — донгл-конфиг для
  Corne-подобных клавиатур (структура очень похожа на предложенную здесь).
- https://github.com/itsdmd/zmk-dongle — аналогичный универсальный пример.
- https://github.com/joric/jorne-zmk-config — официальный ZMK-конфиг для
  обычной (без донгла) схемы Jorne, откуда берётся эталонный keymap.

AlifeBalance: слой балансировки A-Life для STALKER Anomaly, автор Damian
Версия: 1.0.2 (xlibs 1.7.0)
GitHub: https://github.com/damiansirbu-stalker/AlifeBalance
Список изменений: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/changelog
Архитектура: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/architecture.md
English: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/readme.txt
Баги, предложения: https://github.com/damiansirbu-stalker/AlifeBalance/issues

! Сбросьте настройки MCM к значениям по умолчанию после обновления !

AlifeBalance — мод из нескольких систем, стремящийся к сбалансированному и плавному игровому процессу.

Smart Balance:
  Ванильные кулдауны респавна работают по фиксированному расписанию и не учитывают боевое давление. Регионы, в которых вы сражаетесь, пустеют, а тихие части карты остаются полными.
  Smart Balance считает смерти по региону и фракции, и как только их накапливается достаточно, ускоряет кулдаун респавна одной подходящей умной территории на этой карте, выбирая ту, что дальше всех от вас.
  Активный бой возвращает пополнения быстрее, на том уровне, где он случился, и за пределами экрана.

Loot Balance:
  Ванильный лут работает нормально в изоляции, но долгоживущие NPC продолжают подбирать снаряжение с трупов и не останавливаются, поэтому их инвентари разрастаются в хабары, влияющие на баланс, производительность и размер сохранения.
  Лёгкий сканер обходит онлайн-сталкеров по кадрам и обрабатывает каждого NPC не чаще одного раза за игровые сутки. Торговцы пропускаются, потому что их инвентарь и есть их товар.
  Трупы NPC по-прежнему оставляют пару предметов для подбора или продажи. 50-предметная пиньята исчезла.

MCM (General):
  Smart Balance: включение, продвижения до минимума, минимальные минуты.
  Loot Balance: включение, NPC за кадр, NPC за цикл, кулдаун повторного скана.
  Development: уровень логов, маркеры на карте, статус, сброс счётчиков.

Политика Loot Balance:
  Лимиты min/max по категориям лежат в configs/alifebalance/ab_inventory_policy.ltx. Всё, что выше max, выбрасывается. По умолчанию покрываются патроны в слотах оружия (в патронах), гранаты, расходники (medkit / bandage / antirad / stim / pill / food / drink) и снаряжение (weapons / outfits / helmets / artefacts / crafting / device). Поддерживает переопределение через DLTX для интеграторов модпаков.

Совместимость:
  Тестировалось с ванильным Anomaly 1.5.3, GAMMA, ZCP, Redone, Warfare, AlifeGuard, AlifePlus, Night Mutants, Nocturnal Mutants, GAMMA Dynamic Despawner, Guards Spawner.
  Loot Balance требует, чтобы ванильный лут с трупов NPC был включён. Отключите любой мод, блокирующий лутание (например "NPC Stop Looting Dead Bodies") для полного эффекта.

Моды-компаньоны:

AlifePlus (реактивный фреймворк A-Life) -- https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01 | https://www.nexusmods.com/stalkeranomaly/mods/105
AlifeGuard (очистка сущностей) -- https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001 | https://www.nexusmods.com/stalkeranomaly/mods/104

Требования:
Anomaly 1.5.3
xlibs 1.7.0+ (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
MCM

Установка (MO2):
1. Установите xlibs
2. Установите AlifeBalance
3. Порядок загрузки не имеет значения
4. Настройте через MCM

Удаление (MO2):
Отключите или удалите в MO2.

Архитектура и проверка:
Работает на xlibs поверх движка X-Ray, используя только рантайм-коллбэки и не трогая базовые скрипты и бинарь движка. Полное описание дизайна и многоступенчатый пайплайн проверки (luacheck, selene, AST-анализ, контрактные правила, интеграционные тесты) — в doc/architecture.md.

Авторы:
Altogolik — поддержка, идеи, исходные материалы.

Использование и лицензия:
  Модпаки: разрешены и приветствуются. Сохраняйте файлы readme и лицензии.
  Аддоны, патчи, интеграции: разрешены. Укажите "AlifeBalance by Damian Sirbu" заметно на странице вашего мода.
  Воспроизведение реализации в другом программном обеспечении: запрещено, даже с указанием авторства.
  Полная лицензия в файле LICENSE и на GitHub.

Сохраняйте Зону живой, оставляя ванильный A-Life ванильным.

# Лаба 3 — практика на Python

Это практическая часть [лабораторной работы 3](../laba3.md): требования к сдаче и вопросы к защите находятся в
основном тексте лабы, а здесь шаг за шагом разобрано эталонное решение.

Понадобится Python 3.10 или новее, сторонние библиотеки не нужны. Шаблон репозитория: [tech-prog-labs-2026-python](https://github.com/tidbid-kt/tech-prog-labs-2026-python). Код лабы пишите в пакете `labs/lab3`.

В этой лабе мы напишем небольшую текстовую игру.

Герой нашей игры разбивает лагерь на опушке мистического леса. Из лагеря он ходит в экспедиции и сражается с врагами. 
За победы получает опыт, золото и добычу, растет в уровнях. Цель нашего героя - победить древнего дракона, притаившегося в своем логове.

## Игральная кость (Одиночка / Singleton)

В нашей игре многое решает случай: разброс урона, добыча, зачарования. Паттерном *одиночка* реализуем генератор
случайности - игральную кость, через которую будут проходить вообще все броски в игре.

*Одиночка* гарантирует, что у класса есть только один экземпляр, и дает к нему глобальную точку доступа. Нам это и
нужно: источник случайности должен быть один на всю программу, тогда, задав ему сид (начальное значение генератора),
можно в точности воспроизвести все броски экспедиции. Если бы каждый класс создавал свой
генератор случайных чисел, наш сид бы слетал.

```python
from __future__ import annotations

import random
from typing import Sequence, TypeVar

T = TypeVar("T")


# Одиночка: общий источник случайности на всю игру
class Dice:

    # единственный экземпляр класса
    _instance: Dice | None = None

    _random: random.Random

    # __new__ вызывается при каждом Dice() еще до создания объекта:
    # если экземпляр уже есть, возвращаем его, а не создаем новый
    def __new__(cls) -> Dice:
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._random = random.Random()

        return cls._instance

    # после установки сида все последующие броски повторяются в точности
    def reseed(self, seed: int) -> None:
        self._random.seed(seed)

    # случайный сид
    def next_seed(self) -> int:
        return self._random.randrange(1_000_000)

    # случайное целое число от low до high включительно
    def roll(self, low: int, high: int) -> int:
        return self._random.randint(low, high)

    # возвращает True с вероятностью probability (от 0 до 1)
    def chance(self, probability: float) -> bool:
        return self._random.random() < probability

    # случайный элемент последовательности
    def pick(self, options: Sequence[T]) -> T:
        return self._random.choice(options)
```

В Python нет приватных конструкторов, поэтому одиночку обычно делают через метод `__new__`. Он вызывается при каждом
`Dice()` еще до создания объекта, и если экземпляр уже есть, возвращается он. Поэтому в коде мы просто пишем `Dice()`
и всегда получаем один и тот же объект.

Давайте разберем, что в этой реализации важно.

1. Создание объекта перехвачено в `__new__`. Сколько бы раз ни вызывали `Dice()`, объект создается только при первом
   вызове, а дальше возвращается он же.
2. Экземпляр хранится в самом классе и создается лениво, при первом обращении. Пока кость никому не нужна, ее нет.
3. Со звездочкой заметим, что наша реализация не потокобезопасна: если два потока одновременно впервые обратятся к кости, могут создаться два
   объекта. В игре поток один, поэтому нам это не страшно, но в многопоточной программе понадобилась бы синхронизация
   (или создание экземпляра сразу, а не лениво).
   Тему многопоточности мы здесь затрагивать не будем, но для самостоятельного изучения она очень интересна.

Преимущество одиночки очевидно - у всех составляющих игры (оружия, врагов, локаций) одна и та же кость, поэтому один сид задает все броски. 
А еще кость не нужно передавать через конструкторы каждого предмета и каждого врага - к ней можно
обратиться откуда угодно.

Но тут есть и обратная сторона: одиночка - это глобальное состояние. Лук, посох и враги зависят от игральной кости, однако
по их конструкторам этого не видно, а подменить кость на другую (например, в тестах) не получится. Поэтому применять
одиночку стоит только там, где объект действительно должен быть единственным.

Также давайте сразу заведем исключение, которое будем бросать, если по коду ничего не нашлось (неизвестная локация,
неизвестный предмет и т.д.), в будущем оно нам пригодится:

```python
class UnknownCodeError(LookupError):

    def __init__(self, what: str, code: str) -> None:
        super().__init__(f"{what} с кодом '{code}' не существует")
```

## Боевая механика

Давайте определим базовые абстракции нашей игры.

Интерфейс сущности, которая умеет атаковать и получать урон - так с точки зрения механики выглядят и герой, и враги:

```python
from __future__ import annotations

from abc import ABC, abstractmethod

from .hit import Hit


# всё, что может участвовать в бою: и герой, и враги
class Combatant(ABC):

    # код вида сущности: класс героя или тип врага
    @property
    @abstractmethod
    def code(self) -> str: ...

    @property
    @abstractmethod
    def name(self) -> str: ...

    @property
    @abstractmethod
    def health(self) -> int: ...

    @property
    @abstractmethod
    def max_health(self) -> int: ...

    @abstractmethod
    def is_alive(self) -> bool: ...

    # удар, который будет нанесен данной сущностью
    @abstractmethod
    def roll_damage(self) -> Hit: ...

    # получение удара, возвращает урон, который прошел на самом деле
    @abstractmethod
    def receive_hit(self, damage: int) -> int: ...
```

В Python нет интерфейсов, их роль играют абстрактные классы из модуля `abc`: методы с `@abstractmethod` обязаны
реализовать наследники, а создать объект самого абстрактного класса нельзя. Простые значения (код, имя, здоровье) мы
оформляем свойствами `@property`, чтобы читать их как поля, но не давать менять снаружи.

Удар представляет собой вот такую небольшую сущность:

```python
from dataclasses import dataclass


# удар: сколько урона и был ли он критическим
@dataclass(frozen=True)
class Hit:
    damage: int
    critical: bool
```

`@dataclass(frozen=True)` - это удобный способ работать с неизменяемыми объектами: Python сам создаст для такого класса конструктор, а также
методы `__eq__`, `__hash__` и `__repr__`, а поменять поля после создания объекта будет нельзя.

Давайте теперь опишем механику боя. Зафиксируем стороны, типы событий и исход:

```python
from enum import Enum


class Side(Enum):
    HERO = "hero"
    ENEMY = "enemy"
```

```python
from enum import Enum


class EventType(Enum):
    HIT = "hit"
    CRITICAL_HIT = "critical_hit"
    DEATH = "death"
```

```python
from enum import Enum


class Outcome(Enum):
    VICTORY = "victory"
    DEFEAT = "defeat"
```

Бой делится на раунды. Каждое событие раунда (удар, критический удар, гибель) запишем вот так - тут и пригодятся наши `enum`:

```python
from dataclasses import dataclass

from .event_type import EventType
from .side import Side


@dataclass(frozen=True)
class BattleEvent:
    round: int
    type: EventType
    source: Side
    target: Side
    amount: int
    target_health_after: int
```

Чтобы в отчете было видно, кто сражался, заведем класс участника боя: в нем все, что мы выводим о герое и его противнике:

```python
from __future__ import annotations

from dataclasses import dataclass

from .combatant import Combatant


# участник боя в том виде, в котором он попадает в отчет
@dataclass(frozen=True)
class Participant:
    code: str
    name: str
    max_health: int

    @staticmethod
    def of(combatant: Combatant) -> Participant:
        return Participant(combatant.code, combatant.name, combatant.max_health)
```

Супер! Теперь можем составить общий отчет о бое, в нем будут два участника (герой и его противник), исход боя, а также информация о событиях, 
получится что-то вот такое:

```python
from __future__ import annotations

from dataclasses import dataclass

from .battle_event import BattleEvent
from .outcome import Outcome
from .participant import Participant


# результат боя: кто сражался, чем все закончилось и что происходило по ходу дела
@dataclass(frozen=True)
class BattleReport:
    hero: Participant
    enemy: Participant
    outcome: Outcome
    events: tuple[BattleEvent, ...]
```

А теперь наконец-то определим механику самого боя! В целом уже ясно, что идти она будет по раундам, 
в рамках которых герой и враг будут обмениваться ударами, давайте так и реализуем:

```python
from __future__ import annotations

from .battle_event import BattleEvent
from .battle_report import BattleReport
from .combatant import Combatant
from .event_type import EventType
from .outcome import Outcome
from .participant import Participant
from .side import Side


# бой идет по раундам: сначала бьет герой, потом враг, пока кто-то не погибнет
class Battle:

    def __init__(self, hero: Combatant, enemy: Combatant) -> None:
        self._hero = hero
        self._enemy = enemy
        self._events: list[BattleEvent] = []
        self._round = 0

    def run(self) -> BattleReport:
        while self._hero.is_alive() and self._enemy.is_alive():
            self._round += 1
            self._strike(self._hero, Side.HERO, self._enemy, Side.ENEMY)
            if self._enemy.is_alive():
                self._strike(self._enemy, Side.ENEMY, self._hero, Side.HERO)

        outcome = Outcome.VICTORY if self._hero.is_alive() else Outcome.DEFEAT
        return BattleReport(
            Participant.of(self._hero), Participant.of(self._enemy), outcome, tuple(self._events)
        )

    def _strike(self, attacker: Combatant, source: Side, defender: Combatant, target: Side) -> None:
        hit = attacker.roll_damage()
        dealt = defender.receive_hit(hit.damage)
        event_type = EventType.CRITICAL_HIT if hit.critical else EventType.HIT
        self._emit(BattleEvent(self._round, event_type, source, target, dealt, defender.health))

        if not defender.is_alive():
            self._emit(BattleEvent(self._round, EventType.DEATH, source, target, 0, 0))

    def _emit(self, event: BattleEvent) -> None:
        self._events.append(event)
```

Обратите внимание: `Battle` работает только с интерфейсом `Combatant`, ничего не зная о конкретных героях и
врагах. Поэтому сюда можно подставить любую реализацию интерфейса, и переписывать этот класс не придется.

## Предметы (Прототип / Prototype)

Перейдем к предметам. В игре будет много разных предметов одного вида, например, мечей: тренировочный меч, железный меч, клинок рыцаря - все
это мечи, они бьют одинаково стабильно, отличаются только названием, ценой и уроном. Поэтому вид предмета (меч, лук,
посох, броня) мы сделаем классом, а конкретные предметы - объектами этих классов с разными свойствами.

Конкретные предметы будут храниться в каталоге в виде шаблонов. Когда герою нужен предмет (добыча, покупка, стартовое
снаряжение), каталог отдает копию шаблона. Копию можно менять (например, наложить на нее зачарование), шаблон при этом
останется прежним. Для этого применим паттерн *Прототип*: он позволяет создавать новые объекты копированием уже существующих, не завязываясь
на их конкретные классы. У нас каждый предмет сам умеет создавать свою копию.

Начнем с зачарований:

```python
from dataclasses import dataclass


# зачарование неизменяемо (frozen=True): его нельзя поменять после создания,
# поэтому одно и то же зачарование можно безопасно повесить на несколько предметов
@dataclass(frozen=True)
class Enchantment:
    code: str
    name: str
    bonus_damage: int
    bonus_defense: float
    bonus_price: int


SHARP = Enchantment("sharp", "Острое", 3, 0, 20)
FIERY = Enchantment("fiery", "Огненное", 6, 0, 50)
STURDY = Enchantment("sturdy", "Прочное", 0, 0.05, 20)
WARDING = Enchantment("warding", "Охранное", 0, 0.10, 50)

FOR_WEAPONS = (SHARP, FIERY)
FOR_ARMOR = (STURDY, WARDING)
```

В базовый класс предмета заложим и контракт `copy`:

```python
from __future__ import annotations

import copy
from abc import ABC, abstractmethod

from .enchantment import Enchantment


class Item(ABC):

    def __init__(self, code: str, name: str, base_price: int) -> None:
        # код шаблона, из которого получен предмет
        self._code = code
        self._name = name
        self._base_price = base_price
        self._enchantments: list[Enchantment] = []

    def copy(self) -> Item:
        # copy.copy создает объект того же класса, что и оригинал, с теми же значениями полей,
        # поэтому копировать умеет любой наследник, и переопределять этот метод им не нужно
        clone = copy.copy(self)
        # но копия поверхностная: поля копии ссылаются на те же объекты, что и у оригинала,
        # поэтому список зачарований заменяем на новый, иначе копия и оригинал делили бы его
        clone._enchantments = list(self._enchantments)
        return clone

    # какие зачарования подходят этому предмету
    @property
    @abstractmethod
    def possible_enchantments(self) -> tuple[Enchantment, ...]: ...

    # описание характеристик предмета, например "урон 12"
    @property
    @abstractmethod
    def description(self) -> str: ...

    def enchant(self, enchantment: Enchantment) -> None:
        if enchantment not in self.possible_enchantments:
            raise ValueError(f"Зачарование '{enchantment.name}' не подходит для предмета '{self._name}'")

        self._enchantments.append(enchantment)

    @property
    def code(self) -> str:
        return self._code

    @property
    def name(self) -> str:
        return self._name

    @property
    def price(self) -> int:
        return self._base_price + sum(enchantment.bonus_price for enchantment in self._enchantments)

    @property
    def enchantments(self) -> tuple[Enchantment, ...]:
        return tuple(self._enchantments)
```

Обратите внимание на метод `copy`. `copy.copy` создает поверхностную копию: новый объект того же класса, поля которого
ссылаются на те же объекты, что и у оригинала. Если бы мы на этом остановились, копия и шаблон делили бы один и тот же
список зачарований, и зачарование, наложенное на копию, появилось бы и на шаблоне в каталоге. Поэтому список мы
заменяем новым. А вот сами зачарования копировать не нужно - они неизменяемые, и копия может ссылаться на те же объекты.

Можно было бы сразу вызвать `copy.deepcopy`, который рекурсивно копирует все вложенные объекты, но тогда копировались
бы и неизменяемые зачарования, а это лишняя работа.

Супер, общий контракт предметов понятен, давайте создадим контракт для оружия:

```python
from __future__ import annotations

from abc import abstractmethod

from ..combat.hit import Hit
from .enchantment import FOR_WEAPONS, Enchantment
from .item import Item


class Weapon(Item):

    def __init__(self, code: str, name: str, base_price: int, damage: int) -> None:
        super().__init__(code, name, base_price)
        self._damage = damage

    # как именно оружие наносит урон, решают наследники
    @abstractmethod
    def roll_damage(self) -> Hit: ...

    # урон с учетом зачарований
    @property
    def damage(self) -> int:
        return self._damage + sum(enchantment.bonus_damage for enchantment in self.enchantments)

    @property
    def possible_enchantments(self) -> tuple[Enchantment, ...]:
        return FOR_WEAPONS
```

Отлично, теперь наше первое оружие, меч. Его урон всегда одинаковый:

```python
from ..combat.hit import Hit
from .weapon import Weapon


class Sword(Weapon):

    def roll_damage(self) -> Hit:
        return Hit(self.damage, False)

    @property
    def description(self) -> str:
        return f"урон {self.damage}"
```

Лук же с некоторым шансом наносит критический, двойной урон:

```python
from ..combat.hit import Hit
from ..core.dice import Dice
from .weapon import Weapon

CRITICAL_MULTIPLIER = 2


class Bow(Weapon):

    def __init__(self, code: str, name: str, base_price: int, damage: int, critical_chance: float) -> None:
        super().__init__(code, name, base_price, damage)
        self._critical_chance = critical_chance

    def roll_damage(self) -> Hit:
        if Dice().chance(self._critical_chance):
            return Hit(self.damage * CRITICAL_MULTIPLIER, True)

        return Hit(self.damage, False)

    @property
    def description(self) -> str:
        return f"урон {self.damage}, шанс крита {round(self._critical_chance * 100)}%"
```

Урон магического посоха каждый раз немного разный:

```python
from ..combat.hit import Hit
from ..core.dice import Dice
from .weapon import Weapon


# урон посоха непредсказуем: он может быть как больше, так и меньше базового
class Staff(Weapon):

    def __init__(self, code: str, name: str, base_price: int, damage: int, scatter: float) -> None:
        super().__init__(code, name, base_price, damage)
        self._scatter = scatter

    def roll_damage(self) -> Hit:
        return Hit(Dice().roll(self._min_damage, self._max_damage), False)

    @property
    def description(self) -> str:
        return f"урон {self._min_damage}-{self._max_damage}"

    @property
    def _min_damage(self) -> int:
        return round(self.damage * (1 - self._scatter))

    @property
    def _max_damage(self) -> int:
        return round(self.damage * (1 + self._scatter))
```

Дальше давайте создадим класс для брони, он у нас будет общий для всех ее видов:

```python
from __future__ import annotations

from .enchantment import FOR_ARMOR, Enchantment
from .item import Item

# больше этой доли урона никакая броня не поглощает
MAX_DEFENSE = 0.75


# броня поглощает долю входящего урона
class Armor(Item):

    def __init__(self, code: str, name: str, base_price: int, defense: float) -> None:
        super().__init__(code, name, base_price)
        self._defense = defense

    # доля поглощаемого урона с учетом зачарований
    @property
    def defense(self) -> float:
        total = self._defense + sum(enchantment.bonus_defense for enchantment in self.enchantments)
        return min(total, MAX_DEFENSE)

    @property
    def possible_enchantments(self) -> tuple[Enchantment, ...]:
        return FOR_ARMOR

    @property
    def description(self) -> str:
        return f"защита {round(self.defense * 100)}%"
```

Наследникам переопределять `copy` не нужно: `copy.copy` сам создает объект нужного класса. Переопределять его пришлось
бы, только если бы у наследника появились собственные изменяемые поля.

Теперь давайте напишем каталог: он хранит шаблоны всех предметов игры и выдает по ним конкретные экземпляры оружия и
брони:

```python
from __future__ import annotations

from ..core.unknown_code_error import UnknownCodeError
from .armor import Armor
from .item import Item
from .weapon import Weapon


# каталог хранит шаблоны предметов, а наружу отдает только их копии
class ItemCatalog:

    def __init__(self) -> None:
        self._prototypes: dict[str, Item] = {}

    def register(self, prototype: Item) -> None:
        self._prototypes[prototype.code] = prototype

    # каталог не знает, какой конкретно класс у шаблона - копию создает сам шаблон
    def create(self, code: str) -> Item:
        prototype = self._prototypes.get(code)
        if prototype is None:
            raise UnknownCodeError("Предмет", code)

        return prototype.copy()

    def create_weapon(self, code: str) -> Weapon:
        item = self.create(code)
        if isinstance(item, Weapon):
            return item

        raise ValueError(f"Предмет '{code}' не является оружием")

    def create_armor(self, code: str) -> Armor:
        item = self.create(code)
        if isinstance(item, Armor):
            return item

        raise ValueError(f"Предмет '{code}' не является броней")

    @property
    def codes(self) -> list[str]:
        return list(self._prototypes)
```

В методе `create` и работает прототип: каталог вызывает `copy()` у шаблона и получает копию, не зная, меч это,
лук или броня. Если в игре появится новый вид предметов, каталог менять не придется.

Копию при этом создает сам объект, а не внешний код. Только предмет знает свой конкретный класс и все свои поля,
включая закрытые, поэтому снаружи корректно скопировать его не получится. Важно и понимать, насколько глубоко
копировать: изменяемые вложенные объекты (список зачарований) копируются, иначе копия и оригинал будут влиять друг на
друга, а неизменяемые (сами зачарования) можно делить между копиями. Шаблоны каталог наружу не отдает, только их копии,
так что внешний код шаблон не изменит.

Без прототипа на каждый "Железный меч" и "Клинок рыцаря" пришлось бы либо заводить свой класс, либо хранить где-то все
параметры и собирать предмет заново. А так конкретный предмет - это настроенный объект: мы один раз задаем шаблон и
дальше просто копируем его - при выдаче добычи, при покупке у торговца и так далее.

Минусы у паттерна тоже есть. Копирование нужно поддерживать в актуальном состоянии: если у предмета появится новое
изменяемое поле, а скопировать его забудут, копия и оригинал снова начнут делить данные. А объекты со сложными связями,
особенно со ссылками друг на друга по кругу, копировать вообще непросто. Применять прототип стоит, когда объекты
отличаются в основном данными, а не поведением, и их удобно получать из заранее настроенных образцов - как наши предметы
из каталога.

А теперь давайте наполним наш каталог конкретными шаблонами, определив все предметы игры и их характеристики:

```python
from .armor import Armor
from .bow import Bow
from .item_catalog import ItemCatalog
from .staff import Staff
from .sword import Sword


# коды шаблонов: по ним предметы ищутся в каталоге
# стартовое снаряжение
TRAINING_SWORD = "training-sword"
HUNTING_BOW = "hunting-bow"
APPRENTICE_STAFF = "apprentice-staff"
CHAINMAIL = "chainmail"
LEATHER_ARMOR = "leather-armor"
LINEN_ROBE = "linen-robe"

# товары торговца
IRON_SWORD = "iron-sword"
ELVEN_BOW = "elven-bow"
EMBER_STAFF = "ember-staff"
PLATE_ARMOR = "plate-armor"

# добыча
GOBLIN_CLEAVER = "goblin-cleaver"
BONE_BOW = "bone-bow"
NECROMANCER_STAFF = "necromancer-staff"
KNIGHT_BLADE = "knight-blade"
CRYPT_MAIL = "crypt-mail"
DAWN_BLADE = "dawn-blade"
DRAGONSCALE_ARMOR = "dragonscale-armor"


# все предметы, которые есть в игре
def create_default_catalog() -> ItemCatalog:
    catalog = ItemCatalog()

    # стартовое снаряжение
    catalog.register(Sword(TRAINING_SWORD, "Тренировочный меч", 20, 12))
    catalog.register(Bow(HUNTING_BOW, "Охотничий лук", 20, 10, 0.3))
    catalog.register(Staff(APPRENTICE_STAFF, "Посох ученика", 20, 14, 0.3))
    catalog.register(Armor(CHAINMAIL, "Кольчуга", 40, 0.3))
    catalog.register(Armor(LEATHER_ARMOR, "Кожаная броня", 30, 0.2))
    catalog.register(Armor(LINEN_ROBE, "Льняная роба", 15, 0.1))

    # товары торговца
    catalog.register(Sword(IRON_SWORD, "Железный меч", 80, 17))
    catalog.register(Bow(ELVEN_BOW, "Эльфийский лук", 90, 14, 0.4))
    catalog.register(Staff(EMBER_STAFF, "Тлеющий посох", 90, 20, 0.3))
    catalog.register(Armor(PLATE_ARMOR, "Латы", 120, 0.4))

    # добыча
    catalog.register(Sword(GOBLIN_CLEAVER, "Гоблинский тесак", 40, 15))
    catalog.register(Bow(BONE_BOW, "Костяной лук", 140, 17, 0.4))
    catalog.register(Staff(NECROMANCER_STAFF, "Посох некроманта", 150, 25, 0.35))
    catalog.register(Sword(KNIGHT_BLADE, "Клинок рыцаря", 150, 23))
    catalog.register(Armor(CRYPT_MAIL, "Кольчуга стража склепа", 160, 0.45))
    catalog.register(Sword(DAWN_BLADE, "Клинок рассвета", 500, 35))
    catalog.register(Armor(DRAGONSCALE_ARMOR, "Драконья чешуя", 500, 0.55))

    return catalog
```

## Стартовое снаряжение (Абстрактная фабрика / Abstract Factory)

По логике нашей игры мы хотим выдавать герою стартовое снаряжение в зависимости от его класса.
В целом это можно сделать простыми if'ами в коде основной программы, однако мы знаем, что в дальнейшем в игру будут
добавляться новые классы, поэтому здесь уместно будет применить шаблон *Абстрактная фабрика*.
Она позволяет создавать семейства связанных между собой объектов, не привязываясь к их конкретным классам. Семейство у
нас - это стартовое снаряжение класса.

В роли базовой фабрики выступит стартовый набор снаряжения:

```python
from abc import ABC, abstractmethod

from ..item.armor import Armor
from ..item.weapon import Weapon


# Абстрактная фабрика: стартовый набор снаряжения
class StarterKitFactory(ABC):

    # возвращает некое оружие
    @abstractmethod
    def create_weapon(self) -> Weapon: ...

    # возвращает некую броню
    @abstractmethod
    def create_armor(self) -> Armor: ...
```

И для каждого из классов героя мы создадим свою реализацию этого набора: снаряжение каждого класса - это и есть
семейство объектов.

В наборе для воина будут лежать меч и кольчуга:

```python
from ..item.armor import Armor
from ..item.default_catalog import CHAINMAIL, TRAINING_SWORD
from ..item.item_catalog import ItemCatalog
from ..item.weapon import Weapon
from .starter_kit_factory import StarterKitFactory


class WarriorKitFactory(StarterKitFactory):

    def __init__(self, catalog: ItemCatalog) -> None:
        self._catalog = catalog

    def create_weapon(self) -> Weapon:
        return self._catalog.create_weapon(TRAINING_SWORD)

    def create_armor(self) -> Armor:
        return self._catalog.create_armor(CHAINMAIL)
```

В наборе для следопыта - лук и кожаная броня:

```python
from ..item.armor import Armor
from ..item.default_catalog import HUNTING_BOW, LEATHER_ARMOR
from ..item.item_catalog import ItemCatalog
from ..item.weapon import Weapon
from .starter_kit_factory import StarterKitFactory


class RangerKitFactory(StarterKitFactory):

    def __init__(self, catalog: ItemCatalog) -> None:
        self._catalog = catalog

    def create_weapon(self) -> Weapon:
        return self._catalog.create_weapon(HUNTING_BOW)

    def create_armor(self) -> Armor:
        return self._catalog.create_armor(LEATHER_ARMOR)
```

Ну а для мага - посох и роба:

```python
from ..item.armor import Armor
from ..item.default_catalog import APPRENTICE_STAFF, LINEN_ROBE
from ..item.item_catalog import ItemCatalog
from ..item.weapon import Weapon
from .starter_kit_factory import StarterKitFactory


class MageKitFactory(StarterKitFactory):

    def __init__(self, catalog: ItemCatalog) -> None:
        self._catalog = catalog

    def create_weapon(self) -> Weapon:
        return self._catalog.create_weapon(APPRENTICE_STAFF)

    def create_armor(self) -> Armor:
        return self._catalog.create_armor(LINEN_ROBE)
```

Классы героя опишем через `Enum`, и каждый класс будет сам знать свою фабрику стартового снаряжения:

```python
from __future__ import annotations

from enum import Enum

from ..core.unknown_code_error import UnknownCodeError
from ..item.item_catalog import ItemCatalog
from ..kit.mage_kit_factory import MageKitFactory
from ..kit.ranger_kit_factory import RangerKitFactory
from ..kit.starter_kit_factory import StarterKitFactory
from ..kit.warrior_kit_factory import WarriorKitFactory


class HeroClass(Enum):

    # значение каждого класса: код, название, стартовое здоровье и класс фабрики стартового набора
    WARRIOR = ("warrior", "Воин", 120, WarriorKitFactory)
    RANGER = ("ranger", "Следопыт", 100, RangerKitFactory)
    MAGE = ("mage", "Маг", 85, MageKitFactory)

    def __init__(self, code: str, display_name: str, base_health: int, kit_factory: type[StarterKitFactory]) -> None:
        self.code = code
        self.display_name = display_name
        self.base_health = base_health
        self._kit_factory = kit_factory

    # каждый класс сам знает свою фабрику стартового снаряжения
    def starter_kit(self, catalog: ItemCatalog) -> StarterKitFactory:
        return self._kit_factory(catalog)

    @classmethod
    def by_code(cls, code: str) -> HeroClass:
        for hero_class in cls:
            if hero_class.code == code:
                return hero_class

        raise UnknownCodeError("Класс героя", code)
```

Значение каждого элемента `Enum` здесь - кортеж, который Python передает в `__init__`. Фабрику мы храним как класс
(`WarriorKitFactory`) и вызываем его с каталогом, чтобы получить объект фабрики.

Супер, теперь если у нас добавится новый класс - мы сможем просто написать еще одну фабрику стартового снаряжения и добавить
значение в `HeroClass`, в остальном наш код никак не поменяется (потому что он работает с абстракциями).

Обратите внимание, что фабрика объявляет сразу несколько порождающих методов - по одному на каждый вид продукта в
семействе (у нас это оружие и броня). Клиенты фабрики работают только с абстрактными оружием и броней, а то, что под
ними скрыты согласованные между собой реализации, гарантирует фабрика: весь набор приходит из одной фабрики, поэтому
воину не достанется роба, а магу - кольчуга.

При этом у паттерна есть и слабое место: если в набор понадобится добавить новый вид продукта (например, зелье), придется
менять интерфейс фабрики и все ее реализации. Поэтому абстрактная фабрика хорошо подходит, когда состав семейства
устоялся, а сами семейства добавляются часто - как у нас классы героя.

## Герой (Строитель / Builder)

Теперь наконец опишем самого героя. У него будет довольно много характеристик, часть из которых обязательна (имя, класс,
снаряжение), а часть имеет значения по умолчанию (уровень, золото), поэтому для его создания будем использовать паттерн
*Строитель*. Он отделяет пошаговую сборку сложного объекта от самого объекта: строитель собирает и проверяет значения, а
герой получает их уже готовыми.

Приступим:

```python
from __future__ import annotations

from ..combat.combatant import Combatant
from ..combat.hit import Hit
from ..item.armor import Armor
from ..item.item import Item
from ..item.weapon import Weapon
from ..kit.starter_kit_factory import StarterKitFactory
from .hero_class import HeroClass

MAX_NAME_LENGTH = 20
STARTING_GOLD = 50
HEALTH_PER_LEVEL = 15
DAMAGE_PER_LEVEL = 3
EXPERIENCE_PER_LEVEL = 100


class Hero(Combatant):

    # героя создают только через строителя: Hero.Builder()...build(),
    # поэтому конструктор принимает в себя билдера, из которого и берет значения полей
    def __init__(self, builder: Hero.Builder) -> None:
        self._name: str = builder.name
        self._hero_class: HeroClass = builder.hero_class
        self._level = builder.level
        self._experience = 0
        self._gold = builder.gold
        self._weapon: Weapon = builder.weapon
        self._armor: Armor = builder.armor
        self._inventory: list[Item] = []
        self._health = self.max_health

    # внутренний класс билдер - чтобы удобно и безопасно создавать героев
    class Builder:

        def __init__(self) -> None:
            self.name: str | None = None
            self.hero_class: HeroClass | None = None
            self.weapon: Weapon | None = None
            self.armor: Armor | None = None
            # необязательные поля сразу получают значения по умолчанию
            self.level = 1
            self.gold = STARTING_GOLD

        def set_name(self, name: str) -> Hero.Builder:
            self.name = name
            return self

        def set_hero_class(self, hero_class: HeroClass) -> Hero.Builder:
            self.hero_class = hero_class
            return self

        def set_weapon(self, weapon: Weapon) -> Hero.Builder:
            self.weapon = weapon
            return self

        def set_armor(self, armor: Armor) -> Hero.Builder:
            self.armor = armor
            return self

        # берет оружие и броню из стартового набора
        def set_starter_kit(self, starter_kit: StarterKitFactory) -> Hero.Builder:
            self.weapon = starter_kit.create_weapon()
            self.armor = starter_kit.create_armor()
            return self

        def set_level(self, level: int) -> Hero.Builder:
            self.level = level
            return self

        def set_gold(self, gold: int) -> Hero.Builder:
            self.gold = gold
            return self

        # перед созданием героя проверяем, что все обязательные поля заполнены корректно
        def build(self) -> Hero:
            if self.name is None or not self.name.strip():
                raise ValueError("У героя должно быть имя")
            if len(self.name) > MAX_NAME_LENGTH:
                raise ValueError(f"Имя героя не может быть длиннее {MAX_NAME_LENGTH} символов")
            if self.hero_class is None:
                raise ValueError("Не выбран класс героя")
            if self.weapon is None or self.armor is None:
                raise ValueError("Герою нужны оружие и броня")
            if self.level < 1:
                raise ValueError("Уровень героя не может быть меньше 1")
            if self.gold < 0:
                raise ValueError("Золота не может быть меньше нуля")

            return Hero(self)

    def roll_damage(self) -> Hit:
        hit = self._weapon.roll_damage()
        return Hit(hit.damage + (self._level - 1) * DAMAGE_PER_LEVEL, hit.critical)

    def receive_hit(self, damage: int) -> int:
        reduced_damage = round(damage * (1 - self._armor.defense))
        # хотя бы единицу урона броня пропускает всегда
        dealt = max(1, reduced_damage)
        self._health = max(0, self._health - dealt)
        return dealt

    def is_alive(self) -> bool:
        return self._health > 0

    # поведение героя между экспедициями

    # перед каждой экспедицией герой успевает отдохнуть
    def restore_health(self) -> None:
        self._health = self.max_health

    # начисляет опыт, возвращает количество полученных уровней
    def gain_experience(self, amount: int) -> int:
        self._experience += amount
        levels_gained = 0
        while self._experience >= self.experience_to_next_level:
            self._experience -= self.experience_to_next_level
            self._level += 1
            levels_gained += 1

        return levels_gained

    def add_gold(self, amount: int) -> None:
        self._gold += amount

    def spend_gold(self, amount: int) -> None:
        if amount > self._gold:
            raise ValueError(f"Не хватает золота: нужно {amount}, есть {self._gold}")

        self._gold -= amount

    def add_item(self, item: Item) -> None:
        self._inventory.append(item)

    def remove_item(self, item: Item) -> None:
        if item not in self._inventory:
            raise ValueError(f"В инвентаре нет предмета {item.name}")

        self._inventory.remove(item)

    # экипирует предмет из инвентаря, снятый предмет возвращается в инвентарь
    def equip(self, item: Item) -> None:
        if isinstance(item, Weapon):
            self.remove_item(item)
            self._inventory.append(self._weapon)
            self._weapon = item
        elif isinstance(item, Armor):
            self.remove_item(item)
            self._inventory.append(self._armor)
            self._armor = item
        else:
            raise ValueError(f"Предмет нельзя экипировать: {item.name}")

    # свойства только для чтения

    @property
    def code(self) -> str:
        return self._hero_class.code

    @property
    def name(self) -> str:
        return self._name

    @property
    def health(self) -> int:
        return self._health

    @property
    def max_health(self) -> int:
        return self._hero_class.base_health + (self._level - 1) * HEALTH_PER_LEVEL

    @property
    def hero_class(self) -> HeroClass:
        return self._hero_class

    @property
    def level(self) -> int:
        return self._level

    @property
    def experience(self) -> int:
        return self._experience

    @property
    def experience_to_next_level(self) -> int:
        return self._level * EXPERIENCE_PER_LEVEL

    @property
    def gold(self) -> int:
        return self._gold

    @property
    def weapon(self) -> Weapon:
        return self._weapon

    @property
    def armor(self) -> Armor:
        return self._armor

    @property
    def inventory(self) -> tuple[Item, ...]:
        return tuple(self._inventory)
```

В Python часто обходятся без строителя, передавая параметры конструктору по имени (`Hero(name=..., level=...)`). У нас
он все же к месту: снаряжение приходит из фабрики отдельным шагом, а проверить героя хочется целиком, прежде чем он
появится.

Что в реализации строителя важно:

1. Конструктор героя принимает строителя, а не отдельные значения. Закрыть конструктор в Python нельзя, поэтому
   действует договоренность: героя создают только через строителя.
2. Каждый метод строителя возвращает сам строитель, поэтому вызовы можно выстраивать цепочкой.
3. Все проверки собраны в методе `build`: если обязательное поле не заполнено или заполнено неправильно, герой
   просто не создается.
4. Необязательные поля получают значения по умолчанию в самом строителе.

Без строителя понадобился бы конструктор с шестью параметрами, в котором легко перепутать, например,
уровень и золото (оба - целые числа). А по цепочке вызовов строителя сразу видно, какое значение какому полю
присваивается.

Минус строителя - лишний код: появляется отдельный класс, а поля героя приходится объявлять дважды, в самом герое и в
строителе. Для объекта с двумя-тремя полями это избыточно. Строитель нужен, когда полей много и часть из них
необязательна, или когда объект собирается по шагам и перед созданием его нужно проверить целиком.

## Локации (Фабричный метод / Factory Method)

Одну из основ нашей игры составят локации, куда герой будет совершать вылазки. 
В каждой из локаций живут злые враги (а как иначе). Локаций пусть будет три: лес, склеп и логово дракона. 
Однако в дальнейшем их может быть больше, и враги в них будут другие.

Получается, что у всех локаций есть общая логика вылазки, а от локации к локации меняется только враг, который в ней встречается.
Это идеальный случай для применения паттерна *Фабричный метод*: базовый класс объявляет метод для создания объекта, а какой
именно объект создать, решают подклассы. У нас общая логика вылазки остается в коде абстрактной локации, создание врага
выносится в абстрактный метод, а конкретного врага выбирают лес, склеп и логово дракона.

Начнем с врагов. Интерфейс врага:

```python
from abc import abstractmethod

from ..combat.combatant import Combatant


class Enemy(Combatant):

    @property
    @abstractmethod
    def experience_reward(self) -> int: ...

    @abstractmethod
    def roll_gold_reward(self) -> int: ...
```

Общая логика врагов вынесена в абстрактный класс:

```python
from ..combat.hit import Hit
from ..core.dice import Dice
from .enemy import Enemy


# общая логика для врагов: здоровье, урон в диапазоне, награда
class AbstractEnemy(Enemy):

    def __init__(
        self, code: str, name: str, max_health: int,
        min_damage: int, max_damage: int,
        experience_reward: int, gold_reward: int,
    ) -> None:
        self._code = code
        self._name = name
        self._max_health = max_health
        self._min_damage = min_damage
        self._max_damage = max_damage
        self._experience_reward = experience_reward
        self._gold_reward = gold_reward
        self._health = max_health

    def roll_damage(self) -> Hit:
        return Hit(Dice().roll(self._min_damage, self._max_damage), False)

    def receive_hit(self, damage: int) -> int:
        dealt = max(1, self._reduce_damage(damage))
        self._health = max(0, self._health - dealt)
        return dealt

    # наследники могут уменьшать входящий урон
    def _reduce_damage(self, damage: int) -> int:
        return damage

    def is_alive(self) -> bool:
        return self._health > 0

    def roll_gold_reward(self) -> int:
        return Dice().roll(self._gold_reward // 2, self._gold_reward)

    @property
    def experience_reward(self) -> int:
        return self._experience_reward

    @property
    def code(self) -> str:
        return self._code

    @property
    def name(self) -> str:
        return self._name

    @property
    def health(self) -> int:
        return self._health

    @property
    def max_health(self) -> int:
        return self._max_health
```

Определим злого гоблина, самого слабого врага, живущего в лесу:

```python
from .abstract_enemy import AbstractEnemy


# гоблин - самый простой враг, без особенностей
class Goblin(AbstractEnemy):

    def __init__(self) -> None:
        super().__init__("goblin", "Гоблин", 50, 6, 14, 40, 30)
```

Теперь скелет, он уже чуть страшнее гоблина:

```python
from .abstract_enemy import AbstractEnemy

RESISTANCE = 0.3


# скелет: кости плохо поддаются ударам, поэтому часть урона проходит мимо
class Skeleton(AbstractEnemy):

    def __init__(self) -> None:
        super().__init__("skeleton", "Скелет", 110, 10, 24, 90, 70)

    def _reduce_damage(self, damage: int) -> int:
        return round(damage * (1 - RESISTANCE))
```

И ужасный дракон: он наносит страшный урон огнем, а крепкая чешуя гасит часть урона:

```python
from ..combat.hit import Hit
from ..core.dice import Dice
from .abstract_enemy import AbstractEnemy

RESISTANCE = 0.25
FIRE_BREATH_CHANCE = 0.25
FIRE_BREATH_MULTIPLIER = 2


# дракон защищен чешуей и время от времени дышит огнем
class Dragon(AbstractEnemy):

    def __init__(self) -> None:
        super().__init__("dragon", "Дракон", 260, 16, 22, 500, 400)

    def roll_damage(self) -> Hit:
        bite = super().roll_damage()
        if Dice().chance(FIRE_BREATH_CHANCE):
            return Hit(bite.damage * FIRE_BREATH_MULTIPLIER, True)

        return bite

    def _reduce_damage(self, damage: int) -> int:
        return round(damage * (1 - RESISTANCE))
```

По итогам экспедиции герой получает награды: опыт, золото, добычу и новые уровни. Запишем их так:

```python
from __future__ import annotations

from dataclasses import dataclass

from ..item.item import Item


# итоги экспедиции для героя;
# gold отрицательный, если герой проиграл и потерял золото, loot равен None, если добычи нет
@dataclass(frozen=True)
class Rewards:
    experience: int
    gold: int
    loot: Item | None
    levels_gained: int
```

Ну а общий итог экспедиции фиксируем вот так:

```python
from dataclasses import dataclass

from ..combat.battle_report import BattleReport
from .rewards import Rewards


# отчет об экспедиции: с каким сидом, куда, как прошел бой и что получил герой
@dataclass(frozen=True)
class ExpeditionReport:
    seed: int
    location_code: str
    battle: BattleReport
    rewards: Rewards
```

Дальше давайте определим абстрактную локацию и общую логику экспедиции. Конкретных врагов здесь не создаем - это работа
подклассов:

```python
from __future__ import annotations

from abc import ABC, abstractmethod

from ..combat.battle import Battle
from ..combat.outcome import Outcome
from ..core.dice import Dice
from ..enemy.enemy import Enemy
from ..hero.hero import Hero
from ..item.item import Item
from ..item.item_catalog import ItemCatalog
from .expedition_report import ExpeditionReport
from .rewards import Rewards

LOOT_CHANCE = 0.6
ENCHANT_CHANCE = 0.3


class Location(ABC):

    def __init__(self, code: str, display_name: str, recommended_level: int, catalog: ItemCatalog) -> None:
        self._code = code
        self._display_name = display_name
        self._recommended_level = recommended_level
        self._catalog = catalog

    # создание врага делегируется подклассам для повышения гибкости
    @abstractmethod
    def _create_enemy(self) -> Enemy: ...

    # коды предметов из каталога, которые можно найти в локации
    @property
    @abstractmethod
    def _loot_table(self) -> tuple[str, ...]: ...

    # бизнес-логика, общая для всех локаций:
    # герой встречает врага, сражается с ним и получает награду, если победил
    def explore(self, hero: Hero, seed: int | None = None) -> ExpeditionReport:
        dice = Dice()
        if seed is None:
            seed = dice.next_seed()
        dice.reseed(seed)

        hero.restore_health()
        enemy = self._create_enemy()
        battle = Battle(hero, enemy).run()

        if battle.outcome == Outcome.VICTORY:
            rewards = self._reward(hero, enemy, dice)
        else:
            rewards = self._punish(hero)

        return ExpeditionReport(seed, self._code, battle, rewards)

    def _reward(self, hero: Hero, enemy: Enemy, dice: Dice) -> Rewards:
        experience = enemy.experience_reward
        gold = enemy.roll_gold_reward()
        loot = self._roll_loot(dice)

        levels_gained = hero.gain_experience(experience)
        hero.add_gold(gold)
        if loot is not None:
            hero.add_item(loot)

        return Rewards(experience, gold, loot, levels_gained)

    def _roll_loot(self, dice: Dice) -> Item | None:
        if not dice.chance(LOOT_CHANCE):
            return None

        # добыча - это копия шаблона из каталога, ее можно менять, не боясь испортить шаблон
        loot = self._catalog.create(dice.pick(self._loot_table))
        if dice.chance(ENCHANT_CHANCE):
            loot.enchant(dice.pick(loot.possible_enchantments))

        return loot

    # проигравший герой теряет половину золота
    def _punish(self, hero: Hero) -> Rewards:
        lost_gold = hero.gold // 2
        hero.spend_gold(lost_gold)
        return Rewards(0, -lost_gold, None, 0)

    @property
    def code(self) -> str:
        return self._code

    @property
    def display_name(self) -> str:
        return self._display_name

    @property
    def recommended_level(self) -> int:
        return self._recommended_level
```

Подчеркивание в начале имени (`_create_enemy`) в Python означает, что метод предназначен для использования внутри
класса и его наследников. Это соглашение, а не запрет, но и другим разработчикам, и IDE оно ясно говорит, что снаружи
этот метод вызывать не нужно.

Также обратите внимание, что добыча здесь - копия шаблона из каталога, и зачарование накладывается именно на копию.

Отлично, теперь создаем конкретные локации:

Мистический лес, кишащий злыми, но слабыми гоблинами:

```python
from ..enemy.enemy import Enemy
from ..enemy.goblin import Goblin
from ..item.default_catalog import GOBLIN_CLEAVER, HUNTING_BOW, LEATHER_ARMOR
from ..item.item_catalog import ItemCatalog
from .location import Location


class Forest(Location):

    CODE = "forest"

    def __init__(self, catalog: ItemCatalog) -> None:
        super().__init__(Forest.CODE, "Мистический лес", 1, catalog)

    def _create_enemy(self) -> Enemy:
        return Goblin()

    @property
    def _loot_table(self) -> tuple[str, ...]:
        return (GOBLIN_CLEAVER, HUNTING_BOW, LEATHER_ARMOR)
```

Заброшенный склеп, в котором бродят ожившие скелеты:

```python
from ..enemy.enemy import Enemy
from ..enemy.skeleton import Skeleton
from ..item.default_catalog import BONE_BOW, CRYPT_MAIL, KNIGHT_BLADE, NECROMANCER_STAFF
from ..item.item_catalog import ItemCatalog
from .location import Location


class Crypt(Location):

    CODE = "crypt"

    def __init__(self, catalog: ItemCatalog) -> None:
        super().__init__(Crypt.CODE, "Заброшенный склеп", 3, catalog)

    def _create_enemy(self) -> Enemy:
        return Skeleton()

    @property
    def _loot_table(self) -> tuple[str, ...]:
        return (KNIGHT_BLADE, BONE_BOW, NECROMANCER_STAFF, CRYPT_MAIL)
```

И логово дракона, прибежище жуткого чешуйчатого чудища:

```python
from ..enemy.dragon import Dragon
from ..enemy.enemy import Enemy
from ..item.default_catalog import DAWN_BLADE, DRAGONSCALE_ARMOR
from ..item.item_catalog import ItemCatalog
from .location import Location


class DragonLair(Location):

    CODE = "dragon-lair"

    def __init__(self, catalog: ItemCatalog) -> None:
        super().__init__(DragonLair.CODE, "Логово дракона", 5, catalog)

    def _create_enemy(self) -> Enemy:
        return Dragon()

    @property
    def _loot_table(self) -> tuple[str, ...]:
        return (DAWN_BLADE, DRAGONSCALE_ARMOR)
```

Супер! Теперь если в игре появится еще одна локация, нам нужно будет лишь реализовать для нее фабричный метод в новом
классе, в остальном код экспедиций меняться не будет.

Сам фабричный метод объявлен в базовом классе абстрактным и возвращает абстракцию (`Enemy`), а не конкретного врага.
Вызывает его базовый класс в общем алгоритме `explore`, поэтому подклассы не переписывают сценарий экспедиции, а лишь
определяют, кого герой в ней встретит. В итоге сценарий написан один раз, а сам `Location` не зависит ни от гоблинов, ни
от скелетов, ни от драконов.

Минус в том, что на каждый вид продукта нужен свой подкласс создателя: десять видов врагов - десять классов локаций,
даже если ничем другим локации не отличаются. Применять фабричный метод стоит, когда у класса есть общий алгоритм, а
объект, который в нем создается, должен меняться в наследниках.

Кстати, метод `explore` - это пример еще одного паттерна, *Шаблонного метода*: базовый класс задает шаги
алгоритма, а отдельные шаги переопределяются в наследниках. Фабричный метод часто встречается именно как один из
таких шагов.

Давайте сравним фабричный метод с абстрактной фабрикой из прошлого раздела. Абстрактная фабрика - это отдельный объект,
который передается тому, кому он нужен (композиция - мы кратко касались ее на прошлой лекции). 
Ее методы вызывает внешний код (в нашем случае строитель героя), и создает она целое семейство объектов. 

Фабричный метод - это метод самого класса, который переопределяется в наследниках, и создает он один продукт. 
Но главное отличие в том, что фабричным методом пользуется сам класс, в своей общей бизнес-логике (`explore`). 
Снаружи этот фабричный метод никто не вызывает.

Чтобы всем этим пользоваться, осталось завести реестр локаций:

```python
from __future__ import annotations

from typing import Iterable

from ..core.unknown_code_error import UnknownCodeError
from .location import Location


# все локации игры
class LocationRegistry:

    def __init__(self, locations: Iterable[Location]) -> None:
        self._locations: dict[str, Location] = {}
        for location in locations:
            self._locations[location.code] = location

    def get(self, code: str) -> Location:
        location = self._locations.get(code)
        if location is None:
            raise UnknownCodeError("Локация", code)

        return location

    @property
    def all(self) -> list[Location]:
        return list(self._locations.values())
```

## Лагерь

Между экспедициями герой отдыхает в лагере, где есть торговец (а куда же без него). 
Торговец продает предметы из каталога и скупает ненужное за половину цены:

```python
from __future__ import annotations

from typing import Iterable

from ..core.unknown_code_error import UnknownCodeError
from ..hero.hero import Hero
from ..item.item import Item
from ..item.item_catalog import ItemCatalog

SELL_PRICE_PERCENT = 50


class Merchant:

    def __init__(self, catalog: ItemCatalog, assortment: Iterable[str]) -> None:
        self._catalog = catalog
        self._assortment = tuple(assortment)

    # образцы товаров, чтобы показать их покупателю
    @property
    def showcase(self) -> list[Item]:
        showcase = []
        for code in self._assortment:
            showcase.append(self._catalog.create(code))

        return showcase

    def buy(self, hero: Hero, code: str) -> Item:
        if code not in self._assortment:
            raise UnknownCodeError("Товар", code)

        # покупатель получает собственную копию предмета из каталога
        item = self._catalog.create(code)
        hero.spend_gold(item.price)
        hero.add_item(item)
        return item

    def sell(self, hero: Hero, item: Item) -> int:
        price = self.sell_price(item)
        hero.remove_item(item)
        hero.add_gold(price)
        return price

    def sell_price(self, item: Item) -> int:
        return item.price * SELL_PRICE_PERCENT // 100
```

## Консоль

Вся игровая логика наконец-то готова, осталось научить программу общаться с игроком. Весь вывод соберем в одном классе - он
превращает героя, предметы и отчеты об экспедиции в текст для консоли. 

Отдельный класс нужен потому, что дальше с игрой можно будет работать не только через консоль (спойлер-спойлер), и
привязывать к ней игровую логику не хочется.

Вот он:

```python
from __future__ import annotations

from .combat.battle_event import BattleEvent
from .combat.battle_report import BattleReport
from .combat.event_type import EventType
from .combat.outcome import Outcome
from .combat.participant import Participant
from .combat.side import Side
from .hero.hero import Hero
from .item.item import Item
from .world.expedition_report import ExpeditionReport
from .world.rewards import Rewards


# весь вывод в консоль собран здесь
class ConsoleView:

    def describe(self, item: Item) -> str:
        description = item.name

        if item.enchantments:
            names = []
            for enchantment in item.enchantments:
                names.append(enchantment.name)
            description += " [" + ", ".join(names) + "]"

        return description + " (" + item.description + ")"

    def print_hero(self, hero: Hero) -> None:
        print(
            f"{hero.name}, {hero.hero_class.display_name} {hero.level} ур. | здоровье {hero.max_health} | "
            f"опыт {hero.experience}/{hero.experience_to_next_level} | золото {hero.gold}"
        )
        print("  Оружие: " + self.describe(hero.weapon))
        print("  Броня:  " + self.describe(hero.armor))

    def print_report(self, report: ExpeditionReport, location_name: str) -> None:
        battle = report.battle
        print(f"=== {location_name} (сид {report.seed}) ===")
        print(f"{battle.hero.name} встречает врага: {battle.enemy.name} ({battle.enemy.max_health} здоровья)")

        for event in battle.events:
            self._print_event(event, battle)

        self._print_rewards(battle, report.rewards)

    def _print_event(self, event: BattleEvent, battle: BattleReport) -> None:
        source = self._participant(event.source, battle)
        target = self._participant(event.target, battle)

        if event.type == EventType.HIT:
            print(
                f"[раунд {event.round}] {source.name} -> {target.name}: {event.amount} урона, "
                f"осталось {event.target_health_after}/{target.max_health}"
            )
        elif event.type == EventType.CRITICAL_HIT:
            print(
                f"[раунд {event.round}] {source.name} -> {target.name}: КРИТ! {event.amount} урона, "
                f"осталось {event.target_health_after}/{target.max_health}"
            )
        elif event.type == EventType.DEATH:
            print(f"{target.name} повержен!")

    def _print_rewards(self, battle: BattleReport, rewards: Rewards) -> None:
        if battle.outcome == Outcome.DEFEAT:
            print(f"Поражение... {battle.hero.name} едва уносит ноги и теряет {-rewards.gold} золота")
            return

        print(f"Победа! +{rewards.experience} опыта, +{rewards.gold} золота")
        if rewards.loot is not None:
            print("Добыча: " + self.describe(rewards.loot))
        if rewards.levels_gained > 0:
            print("Новый уровень!")

    def _participant(self, side: Side, battle: BattleReport) -> Participant:
        return battle.hero if side == Side.HERO else battle.enemy
```

Осталось написать игровой цикл - он будет в модуле `main`:

```python
from __future__ import annotations

from typing import Sequence

from .camp.merchant import Merchant
from .combat.outcome import Outcome
from .console_view import ConsoleView
from .hero.hero import Hero
from .hero.hero_class import HeroClass
from .item.default_catalog import CHAINMAIL, ELVEN_BOW, EMBER_STAFF, IRON_SWORD, PLATE_ARMOR, create_default_catalog
from .item.item import Item
from .item.item_catalog import ItemCatalog
from .world.crypt import Crypt
from .world.dragon_lair import DragonLair
from .world.forest import Forest
from .world.location_registry import LocationRegistry

VIEW = ConsoleView()


def main() -> None:
    catalog = create_default_catalog()
    locations = LocationRegistry([
        Forest(catalog),
        Crypt(catalog),
        DragonLair(catalog),
    ])
    merchant = Merchant(catalog, [
        IRON_SWORD, ELVEN_BOW, EMBER_STAFF, CHAINMAIL, PLATE_ARMOR,
    ])

    hero = create_hero(catalog)
    print(f"{hero.name} разбивает лагерь на опушке мистического леса.")

    while True:
        print()
        VIEW.print_hero(hero)
        print("1 - отправиться в экспедицию, 2 - снаряжение, 3 - торговец, 0 - выйти из игры")
        choice = input().strip()
        if choice == "1":
            go_on_expedition(hero, locations)
        elif choice == "2":
            manage_equipment(hero)
        elif choice == "3":
            visit_merchant(hero, merchant)
        elif choice == "0":
            return
        else:
            print("Такого варианта нет")


def create_hero(catalog: ItemCatalog) -> Hero:
    print("Создайте своего героя!")
    while True:
        print("Выберите класс:")
        hero_classes = list(HeroClass)
        for i, hero_class in enumerate(hero_classes):
            kit = hero_class.starter_kit(catalog)
            print(
                f"{i + 1} - {hero_class.display_name} (здоровье {hero_class.base_health}): "
                f"{VIEW.describe(kit.create_weapon())}, {VIEW.describe(kit.create_armor())}"
            )
        class_index = read_choice(len(hero_classes))
        if class_index < 0:
            continue
        hero_class = hero_classes[class_index]

        print("Введите имя:")
        name = input().strip()

        try:
            return (
                Hero.Builder()
                .set_name(name)
                .set_hero_class(hero_class)
                .set_starter_kit(hero_class.starter_kit(catalog))
                .build()
            )
        except ValueError as error:
            print(f"Не получилось создать героя: {error}")


def go_on_expedition(hero: Hero, locations: LocationRegistry) -> None:
    all_locations = locations.all
    print("Куда отправимся?")
    for i, location in enumerate(all_locations):
        print(f"{i + 1} - {location.display_name} (рекомендуемый уровень {location.recommended_level})")
    index = read_choice(len(all_locations))
    if index < 0:
        return

    location = all_locations[index]
    report = location.explore(hero)
    print()
    VIEW.print_report(report, location.display_name)

    if report.battle.outcome == Outcome.VICTORY and location.code == DragonLair.CODE:
        print(f"\nДракон повержен! Барды еще долго будут петь о подвиге героя по имени {hero.name}.")


def manage_equipment(hero: Hero) -> None:
    inventory = hero.inventory
    if not inventory:
        print("Инвентарь пуст")
        return

    print("Что экипировать? (0 - назад)")
    print_items(inventory)
    index = read_choice(len(inventory))
    if index < 0:
        return

    item = inventory[index]
    hero.equip(item)
    print("Экипировано: " + VIEW.describe(item))


def visit_merchant(hero: Hero, merchant: Merchant) -> None:
    print(f'Торговец: "Чего желаешь?" (золото: {hero.gold})')
    print("1 - купить, 2 - продать, 0 - назад")
    choice = input().strip()
    if choice == "1":
        buy(hero, merchant)
    elif choice == "2":
        sell(hero, merchant)


def buy(hero: Hero, merchant: Merchant) -> None:
    showcase = merchant.showcase
    for i, item in enumerate(showcase):
        print(f"{i + 1} - {VIEW.describe(item)}: {item.price} золота")
    index = read_choice(len(showcase))
    if index < 0:
        return

    try:
        item = merchant.buy(hero, showcase[index].code)
        print("Куплено: " + VIEW.describe(item))
    except ValueError as error:
        print(error)


def sell(hero: Hero, merchant: Merchant) -> None:
    inventory = hero.inventory
    if not inventory:
        print("Продавать нечего (экипированные предметы продать нельзя)")
        return

    for i, item in enumerate(inventory):
        print(f"{i + 1} - {VIEW.describe(item)}: {merchant.sell_price(item)} золота")
    index = read_choice(len(inventory))
    if index < 0:
        return

    item = inventory[index]
    price = merchant.sell(hero, item)
    print(f"Продано: {item.name} за {price} золота")


def print_items(items: Sequence[Item]) -> None:
    for i, item in enumerate(items):
        print(f"{i + 1} - {VIEW.describe(item)}")


# читает номер варианта от 1 до count, возвращает индекс или -1, если ввели что-то другое
def read_choice(count: int) -> int:
    try:
        choice = int(input().strip())
        if 1 <= choice <= count:
            return choice - 1
    except ValueError:
        # некорректный ввод обрабатываем так же, как отказ от выбора
        pass

    return -1


if __name__ == "__main__":
    main()
```

Супер! Попробуйте запустить игру и победить дракона :)

В целом по коду программы видно, что мы почти не привязываемся к конкретным реализациям классов, а везде используем
абстракции (`Location`, а не конкретные локации для экспедиций, `StarterKitFactory`, а не конкретные наборы для
получения снаряжения, `Item`, а не конкретные предметы при копировании из каталога).

Таким образом в игру легко добавлять новые локации и врагов, классы героя со снаряжением и предметы.

> **_ДОП. ЗАДАНИЕ:_** Если хотите на практике в этом убедиться - попробуйте создать новую локацию и новый класс
> героя, выдав ему новое оружие и броню (не забудьте добавить их в каталог, а для новой локации - добычу!). Много ли мест в старом коде пришлось поменять?

## Демонстрация паттернов

Чтобы показать все паттерны одним запуском, напишем модуль `demo`:

```python
from __future__ import annotations

from .console_view import ConsoleView
from .core.dice import Dice
from .hero.hero import Hero
from .hero.hero_class import HeroClass
from .item.default_catalog import IRON_SWORD, create_default_catalog
from .item.enchantment import FIERY
from .world.crypt import Crypt
from .world.forest import Forest


def main() -> None:
    catalog = create_default_catalog()
    view = ConsoleView()

    print("===== Одиночка =====")
    first = Dice()
    second = Dice()
    print(f"Dice() возвращает один и тот же объект: {first is second}")

    print()
    print("===== Прототип =====")
    plain_sword = catalog.create(IRON_SWORD)
    fiery_sword = catalog.create(IRON_SWORD)
    fiery_sword.enchant(FIERY)
    print("Первая копия:        " + view.describe(plain_sword))
    print("Вторая копия:        " + view.describe(fiery_sword))
    print("Новая копия шаблона: " + view.describe(catalog.create(IRON_SWORD)))
    print(f"Копии - разные объекты: {plain_sword is not fiery_sword}")

    print()
    print("===== Абстрактная фабрика =====")
    for hero_class in HeroClass:
        kit = hero_class.starter_kit(catalog)
        print(
            f"{hero_class.display_name}: {view.describe(kit.create_weapon())}, "
            f"{view.describe(kit.create_armor())}"
        )

    print()
    print("===== Строитель =====")
    hero = (
        Hero.Builder()
        .set_name("Артур")
        .set_hero_class(HeroClass.WARRIOR)
        .set_starter_kit(HeroClass.WARRIOR.starter_kit(catalog))
        .set_level(3)
        .build()
    )
    view.print_hero(hero)
    try:
        (
            Hero.Builder()
            .set_hero_class(HeroClass.MAGE)
            .set_starter_kit(HeroClass.MAGE.starter_kit(catalog))
            .build()
        )
    except ValueError as error:
        print(f"Герой без имени не создается: {error}")
    try:
        Hero.Builder().set_name("Мерлин").build()
    except ValueError as error:
        print(f"Герой без класса не создается: {error}")

    print()
    print("===== Фабричный метод =====")
    forest = Forest(catalog)
    crypt = Crypt(catalog)
    view.print_report(forest.explore(hero, 7), forest.display_name)
    print()
    view.print_report(crypt.explore(hero, 7), crypt.display_name)


if __name__ == "__main__":
    main()
```

Вывод программы:

```
===== Одиночка =====
Dice() возвращает один и тот же объект: True

===== Прототип =====
Первая копия:        Железный меч (урон 17)
Вторая копия:        Железный меч [Огненное] (урон 23)
Новая копия шаблона: Железный меч (урон 17)
Копии - разные объекты: True

===== Абстрактная фабрика =====
Воин: Тренировочный меч (урон 12), Кольчуга (защита 30%)
Следопыт: Охотничий лук (урон 10, шанс крита 30%), Кожаная броня (защита 20%)
Маг: Посох ученика (урон 10-18), Льняная роба (защита 10%)

===== Строитель =====
Артур, Воин 3 ур. | здоровье 150 | опыт 0/300 | золото 50
  Оружие: Тренировочный меч (урон 12)
  Броня:  Кольчуга (защита 30%)
Герой без имени не создается: У героя должно быть имя
Герой без класса не создается: Не выбран класс героя

===== Фабричный метод =====
=== Мистический лес (сид 7) ===
Артур встречает врага: Гоблин (50 здоровья)
[раунд 1] Артур -> Гоблин: 18 урона, осталось 32/50
[раунд 1] Гоблин -> Артур: 8 урона, осталось 142/150
[раунд 2] Артур -> Гоблин: 18 урона, осталось 14/50
[раунд 2] Гоблин -> Артур: 6 урона, осталось 136/150
[раунд 3] Артур -> Гоблин: 18 урона, осталось 0/50
Гоблин повержен!
Победа! +40 опыта, +27 золота

=== Заброшенный склеп (сид 7) ===
Артур встречает врага: Скелет (110 здоровья)
[раунд 1] Артур -> Скелет: 13 урона, осталось 97/110
[раунд 1] Скелет -> Артур: 10 урона, осталось 140/150
[раунд 2] Артур -> Скелет: 13 урона, осталось 84/110
[раунд 2] Скелет -> Артур: 8 урона, осталось 132/150
[раунд 3] Артур -> Скелет: 13 урона, осталось 71/110
[раунд 3] Скелет -> Артур: 11 урона, осталось 121/150
[раунд 4] Артур -> Скелет: 13 урона, осталось 58/110
[раунд 4] Скелет -> Артур: 14 урона, осталось 107/150
[раунд 5] Артур -> Скелет: 13 урона, осталось 45/110
[раунд 5] Скелет -> Артур: 7 урона, осталось 100/150
[раунд 6] Артур -> Скелет: 13 урона, осталось 32/110
[раунд 6] Скелет -> Артур: 8 урона, осталось 92/150
[раунд 7] Артур -> Скелет: 13 урона, осталось 19/110
[раунд 7] Скелет -> Артур: 16 урона, осталось 76/150
[раунд 8] Артур -> Скелет: 13 урона, осталось 6/110
[раунд 8] Скелет -> Артур: 13 урона, осталось 63/150
[раунд 9] Артур -> Скелет: 13 урона, осталось 0/110
Скелет повержен!
Победа! +90 опыта, +41 золота
Добыча: Клинок рыцаря (урон 23)
```

Не забудьте закоммитить и запушить изменения!

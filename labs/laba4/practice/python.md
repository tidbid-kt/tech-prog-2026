# Лаба 4 — практика на Python

Это практическая часть [лабораторной работы 4](../laba4.md): требования к сдаче и вопросы к защите находятся в основном тексте лабы.

Понадобится Python 3.10 или новее, сторонние библиотеки не нужны. Шаблон репозитория: [tech-prog-labs-2026-python](https://github.com/tidbid-kt/tech-prog-labs-2026-python). Код лабы пишите в пакете `labs/lab3`. Все команды запускаются из корня репозитория: игра - `python -m labs.lab3.main`, демонстрация - `python -m labs.lab3.demo`.

В этой лабе мы продолжим развивать игру из прошлой лабы.

Общая механика остается, однако вокруг нее мы сделаем довольно много нового функционала!

* Добавим новую локацию – проклятый особняк, где врагом станет одно из оружий, захваченное злыми духами;
* Усложним нашу игру, враги нам будут попадаться со случайными усилениями: повышенным уроном, защитой, двойной атакой;
* Научимся сохранять нашу игру и продолжать с сохранения, при этом сделаем это эффективно – с кэшированием;
* Спрячем всю сложность управления игрой за фасадом, что повысит читаемость кода;
* А также добавим выбор стратегии на экспедицию – наш герой сможет вести себя более осторожно или агрессивно, в зависимости от выбора;
* Добавим достижения и журнал экспедиций! Чтобы можно было отслеживать свой прогресс в игре;
* И в конце добавим гибкую систему защиты от урона, применим ее и к героям, и к врагам.

Все это мы сделаем с помощью структурных и поведенческих паттернов.

Четвертая лабораторная делается на основе работы, выполненной в лабораторной 3.

В ходе нее мы будем изменять существующие классы и добавлять новые в соответствующие пакеты.
Папка `labs/lab3` и расположение исходников сохраняются; создавать отдельный модуль `lab4` и копировать
классы не нужно. Дальше мы разберем только новые и измененные классы, остальные остаются как в прошлой лабе.

Новые и измененные классы приведены целиком. Ключевые изменения в коде отмечены короткими комментариями.

## Подготовка: данные и контракты

Наши паттерны потребуют немного подготовки перед введением их в действие! Давайте же эту подготовку приведем.

Старого механизма обработки урона нам теперь не хватает. Раньше мы работали просто с числами, теперь давайте масштабироваться.
Заведем отдельный класс `Damage`. Он будет инкапсулировать в себе результат получения урона
(сколько получили, удалось ли уклонится, сколько было поглощено), эти данные понадобятся нам дальше.

**`labs/lab3/defense/damage.py`**

```python
from dataclasses import dataclass


@dataclass
class Damage:
    amount: int
    evaded: bool = False
    absorbed: int = 0

    def reduce_to(self, amount: int) -> None:
        self.amount = amount

    def evade(self) -> None:
        self.amount = 0
        self.evaded = True

    def absorb(self, value: int) -> None:
        self.amount -= value
        self.absorbed += value
```

И теперь `Combatant.receive_hit` будет возвращать `Damage` вместо `int`. Так мы сможем реализовать отдельные механики защиты от урона.
Получится что-то такое:

**`labs/lab3/combat/combatant.py`**

```python
from __future__ import annotations

from abc import ABC, abstractmethod

from .hit import Hit
from ..defense.damage import Damage


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
    def receive_hit(self, damage: int) -> Damage: ...
```

Для врагов давайте обработку урона в `receive_hit` также сделаем через `Damage`, не зря же мы этот класс добавляли:

**`labs/lab3/enemy/abstract_enemy.py`**

```python
from ..defense.damage import Damage
from ..combat.hit import Hit
from ..core.dice import Dice
from .enemy import Enemy


# общая логика для врагов: здоровье, урон в диапазоне, награда
class AbstractEnemy(Enemy):

    def __init__(
        self,
        code: str,
        name: str,
        max_health: int,
        min_damage: int,
        max_damage: int,
        experience_reward: int,
        gold_reward: int,
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

    def receive_hit(self, damage: int) -> Damage:
        dealt = max(1, self._reduce_damage(damage))
        self._health = max(0, self._health - dealt)
        return Damage(dealt)

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

В `Battle` добавим обработку результата `receive_hit`, с учетом того, что он теперь возвращает `Damage`:

**`labs/lab3/combat/battle.py`**

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
        dealt = defender.receive_hit(hit.damage).amount
        event_type = EventType.CRITICAL_HIT if hit.critical else EventType.HIT
        self._emit(BattleEvent(self._round, event_type, source, target, dealt, defender.health))

        if not defender.is_alive():
            self._emit(BattleEvent(self._round, EventType.DEATH, source, target, 0, 0))

    def _emit(self, event: BattleEvent) -> None:
        self._events.append(event)
```

А еще у нас в игре появятся достижения. Выдаваться они будут с помощью паттерна "наблюдатель", но объявим мы их сразу здесь.

Представлять собой они будут идентификатор, название и описание. Как-то так:

**`labs/lab3/progress/achievement.py`**

```python
from enum import Enum
from ..core.unknown_code_error import UnknownCodeError


class Achievement(Enum):
    FIRST_BLOOD = ("first-blood", "Первая кровь", "одержать первую победу")
    LUCKY = ("lucky", "Счастливчик", "найти зачарованный лут")
    VETERAN = ("veteran", "Ветеран", "достичь 5 уровня")
    DRAGON_SLAYER = ("dragon-slayer", "Драконоборец", "победить дракона")

    def __init__(self, code: str, display_name: str, description: str) -> None:
        self.code = code
        self.display_name = display_name
        self.description = description

    @classmethod
    def by_code(cls, code: str) -> "Achievement":
        for achievement in cls:
            if achievement.code == code:
                return achievement
        raise UnknownCodeError("Достижение", code)
```

В `Hero` добавим хранение достижений, полученных за этого героя, а также возможность восстановления опыта, инвентаря и достижений через
строитель.

Нам это понадобится, когда будем добавлять механику сохранений!

А еще метод отдыха переименуем в `rest` вместо `restore_health`. В нем позже появится дополнительное поведение, поэтому стоит сразу немного
расширить контракт.

Итог выглядит вот так:

**`labs/lab3/hero/hero.py`**

```python
from __future__ import annotations

from ..combat.combatant import Combatant
from ..combat.hit import Hit
from ..item.armor import Armor
from ..item.item import Item
from ..item.weapon import Weapon
from ..kit.starter_kit_factory import StarterKitFactory
from .hero_class import HeroClass
from ..defense.damage import Damage
from ..progress.achievement import Achievement

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
        self._experience = builder.experience
        self._gold = builder.gold
        self._weapon: Weapon = builder.weapon
        self._armor: Armor = builder.armor
        self._inventory = list(builder.inventory)
        self._achievements = list(dict.fromkeys(builder.achievements))
        self.rest()

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
            self.experience = 0
            self.inventory: list[Item] = []
            self.achievements: list[Achievement] = []

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

        def set_experience(self, experience: int) -> Hero.Builder:
            self.experience = experience
            return self

        def set_inventory(self, inventory: list[Item]) -> Hero.Builder:
            self.inventory = inventory
            return self

        def set_achievements(self, achievements: list[Achievement]) -> Hero.Builder:
            self.achievements = achievements
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

            if self.experience < 0:
                raise ValueError("Опыт не может быть отрицательным")
            return Hero(self)

    def roll_damage(self) -> Hit:
        hit = self._weapon.roll_damage()
        return Hit(hit.damage + (self._level - 1) * DAMAGE_PER_LEVEL, hit.critical)

    def receive_hit(self, damage: int) -> Damage:
        reduced_damage = round(damage * (1 - self._armor.defense))
        # хотя бы единицу урона броня пропускает всегда
        dealt = max(1, reduced_damage)
        self._health = max(0, self._health - dealt)
        return Damage(dealt)

    def is_alive(self) -> bool:
        return self._health > 0

    # поведение героя между экспедициями

    # перед каждой экспедицией герой успевает отдохнуть
    def rest(self) -> None:
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

    def unlock(self, achievement: Achievement) -> bool:
        if achievement in self._achievements:
            return False
        self._achievements.append(achievement)
        return True

    @property
    def achievements(self) -> tuple[Achievement, ...]:
        return tuple(self._achievements)
```

В `Location` заменяем вызов `restore_health()` на `rest()` — так герой будет отдыхать перед экспедицией:

**`labs/lab3/world/location.py`**

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

        hero.rest()
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

Чтобы иметь возможность восстановить зачарование по идентификатору (нам это понадобится дальше), добавим в него поиск по этому самому
идентификатору:

**`labs/lab3/item/enchantment.py`**

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


def by_code(code: str) -> Enchantment:
    from ..core.unknown_code_error import UnknownCodeError

    for enchantment in (*FOR_WEAPONS, *FOR_ARMOR):
        if enchantment.code == code:
            return enchantment
    raise UnknownCodeError("Зачарование", code)
```

Супер, предварительные изменения внесены, давайте приступать к паттернам и масштабному расширению функционала!

## Одержимое оружие в проклятом особняке (Адаптер / Adapter)

Представим, что мы хотим ввести в нашу игру новую локацию. Вдохновившись историями про жутких призраков таковой мы решаем сделать
проклятый особняк, в нем на героя будет нападать одержимые предметы!

Писать новую реализацию врага с нуля мы не хотим, было бы круто и удобно использовать код класса `Weapon`, это нам и идейно хорошо подходит,
и часть логики пересекается, и `Weapon`, и `Enemy` умеют наносить урон! Но загвоздка в том, что получать урон оружие не умеет...

Чтобы решить это недоразумение и заставить `Weapon` полноценно соответствовать контракту `Enemy` без изменения самого `Weapon`,
применим замечательный паттерн *Адаптер*.

Он позволяет объектам с несовместимыми интерфейсами работать вместе: адаптер оборачивает нужный объект (класса `Weapon` в нашем случае) и
реализует нужный интерфейс (`Enemy` для нас), переводя обращения к себе в вызовы методов оборачиваемого объекта. А если для части контрактов
методов у оборачиваемого объекта нет — адаптер реализует их сам.

Таким образом мы сможем прокинуть `Weapon` как `Enemy` в новый подкласс `Location` с помощью адаптера, и нам не придется менять ни один из
упомянутых
выше классов!

Давайте скорее реализовывать это чудо, напишем сам адаптер. Назовем его `HauntedWeapon`, чтобы выдержать дух, но можно было назвать и более
технично, например,
`WeaponToEnemyAdapter`, суть была бы та же, но не так зловеще.

И так, код получится вот такой:

**`labs/lab3/enemy/haunted_weapon.py`**

```python
from ..combat.hit import Hit
from ..core.dice import Dice
from ..defense.damage import Damage
from ..item.weapon import Weapon
from .enemy import Enemy


class HauntedWeapon(Enemy):
    def __init__(self, weapon: Weapon) -> None:
        self._weapon = weapon
        self._max_health = weapon.damage * 5
        self._health = self._max_health

    @property
    def code(self) -> str:
        return "haunted-weapon"

    @property
    def name(self) -> str:
        return "Одержимый " + self._weapon.name

    @property
    def health(self) -> int:
        return self._health

    @property
    def max_health(self) -> int:
        return self._max_health

    def is_alive(self) -> bool:
        return self._health > 0

    def roll_damage(self) -> Hit:
        return self._weapon.roll_damage()

    def receive_hit(self, amount: int) -> Damage:
        damage = Damage(amount)
        self._health = max(0, self._health - damage.amount)
        return damage

    @property
    def experience_reward(self) -> int:
        return self._weapon.price // 2

    def roll_gold_reward(self) -> int:
        return Dice().roll(self._weapon.price // 4, self._weapon.price // 2)
```

А теперь объявим и саму локацию — проклятый особняк. В ней будет одно из трех случайных оружий, которые будут взяты из каталога.
Выбранное оружие будет обернуто в адаптер, то есть станет одержимо злым духом, после чего безжалостно нападет на нашего героя.

Благодаря адаптеру мы можем превратить во врага любое оружие, что безусловно разнообразит визиты в это замечательное жилище:

**`labs/lab3/world/haunted_manor.py`**

```python
from ..core.dice import Dice
from ..enemy.enemy import Enemy
from ..enemy.haunted_weapon import HauntedWeapon
from ..item.default_catalog import IRON_SWORD, ELVEN_BOW, EMBER_STAFF
from ..item.item_catalog import ItemCatalog
from .location import Location


class HauntedManor(Location):
    CODE = "haunted-manor"
    HAUNTED_WEAPONS = (IRON_SWORD, ELVEN_BOW, EMBER_STAFF)

    def __init__(self, catalog: ItemCatalog) -> None:
        super().__init__(self.CODE, "Проклятый особняк", 2, catalog)

    def _create_enemy(self) -> Enemy:
        return HauntedWeapon(self._catalog.create_weapon(Dice().pick(self.HAUNTED_WEAPONS)))

    @property
    def _loot_table(self) -> tuple[str, ...]:
        return self.HAUNTED_WEAPONS
```

Просто прелестно! Получилось красиво, расширяемо и интересно. А еще мы сэкономили на реализации метода нанесения урона! Его получилось взять
из оружия подчистую, и менять ничего не пришлось. Остальные методы в целом тоже интересно легли на механику оружия (большинство из них так
или иначе
зависит от его характеристик, здоровье от урона, а награда от ценности).

Важно запомнить, что адаптер стоит применять только если адаптируемый класс близок по своей логике к нужному интерфейсу. Если это не так,
адаптер может решить все задачи самостоятельно и вместо переиспользования у нас получится лишняя зависимость в адаптере. В нашем примере
случай в целом
пограничный – часть логики переиспользуем, часть реализуем сами.

Главный маркер того, что адаптер стоит рассмотреть — готовый класс нужно использовать там, где ожидается другой, но идейно похожий
интерфейс,
а менять сам класс нельзя или не хочется - например, если это чужая библиотека.

## Элитные враги (Декоратор / Decorator)

После расширения списка наших локаций мы поняли – в нашей игре слишком мало челленджа и рандома!

Давайте сделаем вот как – теперь по ходу экспедиции герою могут попадаться усиленные, элитные враги. При входе в локацию с некоторой
вероятностью
враги получают модификаторы. Легендарный (двойной урон), бронированный (уменьшение входящего урона) и обладающий неистовством ветра (
складывающий урон двух атак в один удар).

И причем при совсем уж плохом везении, эти модификаторы еще и должны комбинироваться! То есть в худшем случае на первом же входе в лес перед
нами предстанет
Легендарный бронированный неистовый гоблин и разнесет нашего героя без единого шанса.

Но вместе с риском будет расти и награда, конечно, за таких врагов будет больше опыта и золота.

В целом идея отличная, но тут есть загвоздка. Врагов у нас недавно было 3 вида, сейчас стало 4. В будущем может быть еще больше. Делать
каждому из них подклассы
с модификаторами (неистовый гоблин, неистовый дракон, и так далее) это еще половина беды, настоящая проблема в том, что мы их должны еще и
комбинировать. Если каждый из трех модификаторов может присутствовать или отсутствовать, даже без учета порядка
получится 4 × 2³ = 32 варианта врагов, включая обычных врагов без модификаторов.

То есть через наследование классов врагов эту задачу не решить, слишком много подклассов, и если у нас появятся еще модификаторы, их станет
еще больше.

На помощь нам приходит паттерн декоратор! Он умеет заворачивать в себя любой класс, следующий определенному контракту, и модифицировать его
поведение.
При этом этих декораторов можно комбинировать! И понадобится нам всего 3 класса декораторов, 1 класс базового декоратора и все. И новый
модификатор потребует
всего лишь нового класса декоратора. Плюсом к этому, мы еще и заворачивать их сможем динамически, прямо при выполнении программы.

Получаются сплошные плюсы, давайте смотреть

Еще раз проговорим, про что сам паттерн. *Декоратор* позволяет динамически добавлять объекту новое поведение, оборачивая его
в другие объекты (декораторы) с тем же интерфейсом. Благодаря композиции декоратор расширяет поведение методов класса без
необходимости увеличивать дерево наследования, а декораторы при этом можно комбинировать друг с другом.

Первым делом определим базовый декоратор, он не определяет нового поведения, а выступает основной структурной единицей для дальнейшей
реализации паттерна. В нем через композицию определяется, кого заворачивает декоратор (поле `_wrapped`), все методы базового декоратора
делегируют
вызовы объекту `_wrapped`.

Важно, что `_wrapped` это абстракция – `Enemy` и сам базовый декоратор тоже реализует эту абстракцию. Таким образом на месте `_wrapped` может
быть и другой декоратор.
Но в конце цепочки обязательно будет конкретный враг.

Итак, код получается вот такой:

**`labs/lab3/enemy/enemy_decorator.py`**

```python
from ..combat.hit import Hit
from ..defense.damage import Damage
from .enemy import Enemy


class EnemyDecorator(Enemy):
    def __init__(self, wrapped: Enemy) -> None:
        self._wrapped = wrapped

    @property
    def code(self) -> str:
        return self._wrapped.code

    @property
    def name(self) -> str:
        return self._wrapped.name

    @property
    def health(self) -> int:
        return self._wrapped.health

    @property
    def max_health(self) -> int:
        return self._wrapped.max_health

    @property
    def experience_reward(self) -> int:
        return self._wrapped.experience_reward

    def is_alive(self) -> bool:
        return self._wrapped.is_alive()

    def roll_damage(self) -> Hit:
        return self._wrapped.roll_damage()

    def receive_hit(self, amount: int) -> Damage:
        return self._wrapped.receive_hit(amount)

    def roll_gold_reward(self) -> int:
        return self._wrapped.roll_gold_reward()
```

А дальше уже реализуем конкретных декораторов, наследников базового. Они будут соответствовать нашим модификатором, начнем с легендарного:

**`labs/lab3/enemy/legendary_enemy.py`**

```python
from ..combat.hit import Hit
from .enemy_decorator import EnemyDecorator


class LegendaryEnemy(EnemyDecorator):
    @property
    def name(self) -> str:
        return "Легендарный " + super().name

    def roll_damage(self) -> Hit:
        hit = super().roll_damage()
        return Hit(hit.damage * 2, hit.critical)

    @property
    def experience_reward(self) -> int:
        return super().experience_reward * 2

    def roll_gold_reward(self) -> int:
        return super().roll_gold_reward() * 2
```

Тут важно заметить, что легендарный декоратор во всех своих методах вызывает методы своего суперкласса – базового декоратора. При
нанесении двойного урона он берет за основу именно `super().roll_damage()`. Именно это и позволяет комбинировать декораторов. Мы уже видели,
что
базовый декоратор просто вызывает методы обертки, значит `super().roll_damage()` это `_wrapped.roll_damage()` и если `_wrapped` это враг в еще
одном декораторе –
дальше будет вызвано уже модифицированное им поведение. И так по цепочке пока мы не дойдем до одного из наших конкретных врагов.

Супер, базовый принцип понятен, продолжим реализовывать модификаторы, дальше у нас неистовство ветра:

**`labs/lab3/enemy/windfury_enemy.py`**

```python
from ..combat.hit import Hit
from .enemy_decorator import EnemyDecorator


class WindfuryEnemy(EnemyDecorator):
    @property
    def name(self) -> str:
        return "Неистовый " + super().name

    def roll_damage(self) -> Hit:
        first, second = super().roll_damage(), super().roll_damage()
        return Hit(first.damage + second.damage, first.critical or second.critical)

    @property
    def experience_reward(self) -> int:
        reward = super().experience_reward
        return reward + reward * 50 // 100

    def roll_gold_reward(self) -> int:
        reward = super().roll_gold_reward()
        return reward + reward * 50 // 100
```

Два броска здесь складываются в один удар, поэтому защита применяется к их сумме.

Логика тут точно такая же! Модифицируем поведение, делегируем в `super()`.

Третий и пока финальный декоратор уменьшает входящий урон вдвое, затем передаёт получившийся урон дальше по цепочке декораторов:

**`labs/lab3/enemy/armored_enemy.py`**

```python
from ..defense.damage import Damage
from .enemy_decorator import EnemyDecorator


class ArmoredEnemy(EnemyDecorator):
    @property
    def name(self) -> str:
        return "Бронированный " + super().name

    def receive_hit(self, amount: int) -> Damage:
        if amount < 0:
            raise ValueError("Урон не может быть отрицательным")
        if amount == 0:
            return Damage(0)
        return super().receive_hit(amount - amount // 2)
```

Отлично, все требуемые модификаторы объявлены.
Теперь мы можем добавлять их любому врагу и в любой последовательности.
При этом декоратор реализует тот же интерфейс `Enemy`, что и обычный враг, поэтому ни бой, ни локация не отличают элитного врага
от обычного, и работают с ним точно так же. Прелести `Dependency Inversion Principle`!

Осталось прописать логику, как и когда враг становится элитным. Это часть общей логики экспедиции, поэтому добавим её в локацию:
враг, созданный фабричным методом, с некоторым шансом получает комбинацию модификаторов.

**`labs/lab3/world/location.py`**

```python
from __future__ import annotations

from ..enemy.armored_enemy import ArmoredEnemy
from ..enemy.legendary_enemy import LegendaryEnemy
from ..enemy.windfury_enemy import WindfuryEnemy
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

        hero.rest()
        enemy = self._create_enemy()
        if dice.chance(0.2):
            enemy = self._make_elite(enemy, dice)
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

    def _make_elite(self, enemy: Enemy, dice: Dice) -> Enemy:
        both = dice.chance(0.25)
        if dice.chance(0.5):
            enemy = LegendaryEnemy(enemy)
            if both:
                enemy = WindfuryEnemy(enemy)
        else:
            enemy = WindfuryEnemy(enemy)
            if both:
                enemy = LegendaryEnemy(enemy)
        return ArmoredEnemy(enemy) if dice.chance(0.3) else enemy
```

Получилось красиво и очень расширяемо! Добавить новый модификатор не составит нам ровно никакого труда, главное
аккуратно реализовать передачу вызовов, чтобы комбинирование не развалилось.

Как можно понять из примера, применять декоратор стоит, когда объекту нужно добавлять поведение во время работы программы, а
комбинаций такого поведения слишком много, чтобы заводить на каждую свой подкласс.

## Сохранения (Заместитель / Proxy)

В нашей игре нет кое-чего очень важного! Сейчас при каждом выходе из игры герой просто пропадает. Но со всеми нашими усложнениями
это становится все менее приемлемо. Нам надо сохранять прогресс, чтобы не стартовать с самого начала.

Мы могли бы сделать это через БД, но этим мы займемся сильно позже. Над реализацией, которая нас устроила бы сейчас, подумаем чуть дальше. А
пока
давайте определим контракт для сервиса, который будет заниматься сохранением нашего героя.

Опишем интерфейс хранилища сохранений, нам тут важно иметь возможность найти героя по имени, вывести список всех героев, а также сохранить
текущее состояние нашего персонажа:

**`labs/lab3/storage/hero_repository.py`**

```python
from typing import Protocol
from ..hero.hero import Hero


class HeroRepository(Protocol):
    def find_by_name(self, name: str) -> Hero | None: ...
    def find_all(self) -> list[Hero]: ...
    def save(self, hero: Hero) -> None: ...
```

Контракт репозитория описан через `Protocol`: классу достаточно предоставить указанные методы, наследоваться от протокола не обязательно.

Для хранения сейвов будем использовать простой файл, это не бд, но подход тоже вполне рабочий. В качестве формата сохранения выберем
json! [JSON](https://www.json.org/json-ru.html) — текстовый формат, где объекты состоят из пар "имя — значение",
а массивы хранят списки значений. Это довольно вездесущий формат, и нам он как нельзя кстати подходит.

Для чтения и записи JSON используем стандартный модуль `json`, дополнительные зависимости не нужны.

В файле сохранения будет массив героев со всеми нужными для восстановления объектов данными.
Данные героя и предметов собираем в словари и списки, а при чтении восстанавливаем героя через строителя и предметы через каталог.

Реализуем наконец сам репозиторий:

**`labs/lab3/storage/file_hero_repository.py`**

```python
import json
from pathlib import Path
from ..hero.hero import Hero
from ..hero.hero_class import HeroClass
from ..item.item import Item
from ..item.item_catalog import ItemCatalog
from ..item.enchantment import by_code
from ..progress.achievement import Achievement


class FileHeroRepository:
    def __init__(self, file: Path, catalog: ItemCatalog) -> None:
        self._file = file
        self._catalog = catalog

    def find_by_name(self, name: str) -> Hero | None:
        return next((hero for hero in self.find_all() if hero.name == name), None)

    def find_all(self) -> list[Hero]:
        if not self._file.exists():
            return []
        text = self._file.read_text(encoding="utf-8")
        if not text.strip():
            return []
        saves = json.loads(text)
        if not isinstance(saves, list):
            raise ValueError("В файле должен быть массив героев")
        return [self._restore(save) for save in saves]

    def save(self, hero: Hero) -> None:
        saves = [self._snapshot(other) for other in self.find_all() if other.name != hero.name]
        saves.append(self._snapshot(hero))
        self._file.parent.mkdir(parents=True, exist_ok=True)
        self._file.write_text(json.dumps(saves, ensure_ascii=False, indent=2) + "\n", encoding="utf-8")

    def _snapshot_item(self, item: Item) -> dict:
        return {"code": item.code, "enchantments": [enchantment.code for enchantment in item.enchantments]}

    def _snapshot(self, hero: Hero) -> dict:
        return {
            "name": hero.name,
            "heroClass": hero.hero_class.code,
            "level": hero.level,
            "experience": hero.experience,
            "gold": hero.gold,
            "weapon": self._snapshot_item(hero.weapon),
            "armor": self._snapshot_item(hero.armor),
            "inventory": [self._snapshot_item(item) for item in hero.inventory],
            "achievements": [achievement.code for achievement in hero.achievements],
        }

    def _restore_item(self, save: dict) -> Item:
        item = self._catalog.create(save["code"])
        for code in save["enchantments"]:
            item.enchant(by_code(code))
        return item

    def _restore(self, save: dict) -> Hero:
        return (
            Hero.Builder()
            .set_name(save["name"])
            .set_hero_class(HeroClass.by_code(save["heroClass"]))
            .set_level(save["level"])
            .set_experience(save["experience"])
            .set_gold(save["gold"])
            .set_weapon(self._restore_item(save["weapon"]))
            .set_armor(self._restore_item(save["armor"]))
            .set_inventory([self._restore_item(item) for item in save["inventory"]])
            .set_achievements([Achievement.by_code(code) for code in save["achievements"]])
            .build()
        )
```

Реализация получилась рабочая, но не очень эффективная. На каждый запрос к герою файл читается целиком, а герои собираются
заново. Мы тратим драгоценное время игры на то, чтобы открыть файл, распарсить JSON, собрать объекты, это все в целом быстро, но кажется
что можно сделать чуть эффективнее.

Давайте добавим кэширование! При первом запросе чтения загрузим героев в память, а повторные запросы чтения
будем обслуживать из кэша без обращения к файлу. Мы же можем просто хранить информацию о сохраненных героях в памяти? Можем!

Если говорить простым языком, кэширование, это когда мы временно кладем данные из медленного хранилища в быстрое, чтобы операции чтения с
ними были более
производительны. В нашем примере медленное хранилище – файл, быстрое – объекты в памяти языка. Файл дает нам хранение между перезапусками,
но работать с ним медленно.
Объекты в памяти пропадают между перезапусками, но обращение к ним мгновенное. Поэтому идейно мы хотим один раз прочитать текущие сохранения
из файла,
отправить их в специальный объект (кэш), и потом уже читать оттуда. Но при этом если появляется новое сохранение, мы обязаны записать его в
оба места (иначе кэш разойдется с основным хранилищем). При сохранении файловый репозиторий по-прежнему читает и перезаписывает файл.
Изменения файла другой программой наш кэш не замечает: обновление кэша предусмотрено только через этот заместитель.

Кэширующая логика пригодится нам не только для текущей реализации репозитория, но и для других, если они появятся.
Плюс добавлять ее напрямую в реализацию не очень красиво, смешивается логика, нарушается `SRP`.

Чтобы этого не делать, воспользуемся паттерном *Заместитель*. Он позволяет подставлять вместо настоящего объекта другой объект с тем же
интерфейсом,
который перехватывает обращения и управляет доступом к настоящему объекту.

Реализуем универсального кэширующего заместителя для репозитория сохранений. Он будет оборачивать исходный объект, и в каждый метод
перед реальным чтением сохранений проверять кэш:

**`labs/lab3/storage/caching_hero_repository.py`**

```python
from ..hero.hero import Hero
from .hero_repository import HeroRepository


class CachingHeroRepository:
    def __init__(self, target: HeroRepository) -> None:
        self._target = target
        self._cache: dict[str, Hero] | None = None

    def _cached(self) -> dict[str, Hero]:
        if self._cache is None:
            self._cache = {hero.name: hero for hero in self._target.find_all()}
        return self._cache

    def find_by_name(self, name: str) -> Hero | None:
        return self._cached().get(name)

    def find_all(self) -> list[Hero]:
        return list(self._cached().values())

    def save(self, hero: Hero) -> None:
        self._target.save(hero)
        self._cached()[hero.name] = hero
```

Отлично! Не трогая код работы с файлом, мы смогли модифицировать и оптимизировать его поведение. Кстати, кэш здесь
заполняется лениво: пока героев никто не попросил, файл даже не читается.

Заметим, что заместитель и декоратор очень похожи, они оба оборачивают объект с тем же интерфейсом и позволяют изменять поведение
оборачиваемого класса. Однако заместитель чаще всего не добавляет новое поведение, а управляет доступом к
оборачиваемому объекту. В данном случае управление доступом заключается в кэшировании — вызова настоящего хранилища
не происходит, если нужный герой уже есть в кэше. При этом для клиента заместителя результат остается тем же, тогда
как декоратор поведение модифицирует.

## Фасад: будущее упрощение внутрянки

Фасад предоставляет простой интерфейс к сложной подсистеме. В нашей игре он будет создавать и связывать каталог,
локации, сохранения, и много другое, а консоли предоставит операции создания героя, работы со снаряжением и так далее.

Реализацию опишем в конце: сначала подготовим оставшиеся подсистемы, которые фасад будет объединять.

## Тактика боя (Стратегия / Strategy)

Наконец-то мы переходим к поведенческим паттернам!

Мы хотим разнообразить игровой процесс и добавить тактической глубины! Давайте перед экспедицией дадим возможность выбирать тактику,
от нее будет зависеть, как герой ведет себя в бою. Бьет ли в полную силу, раскрываясь для ответного удара, или уходит в
оборону.

Можно было бы прописать все тактики прямо в `Battle` через условия, однако тогда каждая новая тактика требовала бы
правки этого класса. А еще нарушался бы `SRP`.

Так как в данном случае все изменения сводятся к выбору между разными алгоритми выполнения одного и того же действия
(выбора, что делать в раунде), можно применить паттерн *Стратегия*.

Он определяет семейство алгоритмов, выносит каждый
из них в отдельный класс и делает их взаимозаменяемыми: тот, кто алгоритм использует, работает с общим интерфейсом и
не знает, какая именно реализация ему досталась.

Сначала опишем, что герой вообще может сделать в раунде:

**`labs/lab3/tactic/action.py`**

```python
from enum import Enum


class Action(Enum):
    ATTACK = (1.0, 1.0)
    POWER_STRIKE = (1.5, 1.5)
    DEFEND = (0.5, 0.3)

    def __init__(self, attack_multiplier: float, incoming_multiplier: float) -> None:
        self.attack_multiplier = attack_multiplier
        self.incoming_multiplier = incoming_multiplier
```

Получается, что он может вести себя агрессивно (увеличивается и исходящий, и входящий урон), сбалансированно (ничего не меняется) и
осторожно, уходя в оборону (уменьшается и входящий, и исходящий урон).

Выбирать действия мы будем в зависимости от состояния врага и героя, определим базовый интерфейс тактики:

**`labs/lab3/tactic/tactic.py`**

```python
from abc import ABC, abstractmethod
from ..combat.combatant import Combatant
from .action import Action


class Tactic(ABC):
    @property
    @abstractmethod
    def code(self) -> str: ...

    @property
    @abstractmethod
    def display_name(self) -> str: ...

    @property
    @abstractmethod
    def description(self) -> str: ...

    @abstractmethod
    def choose_action(self, round: int, hero: Combatant, enemy: Combatant) -> Action: ...
```

Реализуем самый простой алгоритм выбора действия - обычный удар каждый раунд:

**`labs/lab3/tactic/balanced_tactic.py`**

```python
from ..combat.combatant import Combatant
from .action import Action
from .tactic import Tactic


class BalancedTactic(Tactic):
    @property
    def code(self) -> str:
        return "balanced"

    @property
    def display_name(self) -> str:
        return "Баланс"

    @property
    def description(self) -> str:
        return "обычный удар каждый раунд"

    def choose_action(self, round: int, hero: Combatant, enemy: Combatant) -> Action:
        return Action.ATTACK
```

Натиск - пока здоровье врага выше, чем у героя, герой бьет безрассудно, нанося и получая в ответ больше урона:

**`labs/lab3/tactic/aggressive_tactic.py`**

```python
from ..combat.combatant import Combatant
from .action import Action
from .tactic import Tactic


class AggressiveTactic(Tactic):
    @property
    def code(self) -> str:
        return "aggressive"

    @property
    def display_name(self) -> str:
        return "Натиск"

    @property
    def description(self) -> str:
        return "совершать безрассудные удары, пока у врага больше здоровья, чем у героя"

    def choose_action(self, round: int, hero: Combatant, enemy: Combatant) -> Action:
        return Action.POWER_STRIKE if enemy.health > hero.health else Action.ATTACK
```

И осторожность - раненый герой через раунд будет уходить в оборону:

**`labs/lab3/tactic/cautious_tactic.py`**

```python
from ..combat.combatant import Combatant
from .action import Action
from .tactic import Tactic


class CautiousTactic(Tactic):
    @property
    def code(self) -> str:
        return "cautious"

    @property
    def display_name(self) -> str:
        return "Осторожность"

    @property
    def description(self) -> str:
        return "при здоровье ниже 40% через раунд уходить в оборону"

    def choose_action(self, round: int, hero: Combatant, enemy: Combatant) -> Action:
        return Action.DEFEND if hero.health < hero.max_health * 0.4 and round % 2 == 0 else Action.ATTACK
```

Также дополним типы ивентов в нашей игре, добавим наши действия, и `EVADE`/`ABSORB`, они нам пригодятся дальше:

**`labs/lab3/combat/event_type.py`**

```python
from enum import Enum


class EventType(Enum):
    HIT = "hit"
    CRITICAL_HIT = "critical_hit"
    DEATH = "death"
    POWER_STRIKE = "power-strike"
    DEFEND = "defend"
    EVADE = "evade"
    ABSORB = "absorb"
```

Теперь обновим сам `Battle`. Тактику он получает через конструктор и каждый раунд спрашивает у нее, что именно делает герой:

**`labs/lab3/combat/battle.py`**

```python
from __future__ import annotations

from .battle_event import BattleEvent
from .battle_report import BattleReport
from .combatant import Combatant
from .event_type import EventType
from .outcome import Outcome
from .participant import Participant
from .side import Side
from ..tactic.action import Action
from ..tactic.tactic import Tactic
from ..tactic.balanced_tactic import BalancedTactic


# бой идет по раундам: сначала бьет герой, потом враг, пока кто-то не погибнет
class Battle:

    def __init__(self, hero: Combatant, enemy: Combatant, tactic: Tactic | None = None) -> None:
        self._hero = hero
        self._tactic = tactic if tactic is not None else BalancedTactic()
        self._enemy = enemy
        self._events: list[BattleEvent] = []
        self._round = 0

    def run(self) -> BattleReport:
        while self._hero.is_alive() and self._enemy.is_alive():
            self._round += 1
            action = self._tactic.choose_action(self._round, self._hero, self._enemy)
            if action == Action.POWER_STRIKE:
                self._emit(BattleEvent(self._round, EventType.POWER_STRIKE, Side.HERO, Side.ENEMY, 0, 0))
            elif action == Action.DEFEND:
                self._emit(BattleEvent(self._round, EventType.DEFEND, Side.HERO, Side.HERO, 0, 0))
            self._strike(self._hero, Side.HERO, self._enemy, Side.ENEMY, action.attack_multiplier)
            if self._enemy.is_alive():
                self._strike(self._enemy, Side.ENEMY, self._hero, Side.HERO, action.incoming_multiplier)

        outcome = Outcome.VICTORY if self._hero.is_alive() else Outcome.DEFEAT
        return BattleReport(
            Participant.of(self._hero), Participant.of(self._enemy), outcome, tuple(self._events)
        )

    def _strike(
        self, attacker: Combatant, source: Side, defender: Combatant, target: Side, multiplier: float
    ) -> None:
        hit = attacker.roll_damage()
        damage = defender.receive_hit(max(1, int(hit.damage * multiplier + 0.5)))
        if damage.evaded:
            self._emit(BattleEvent(self._round, EventType.EVADE, source, target, 0, defender.health))
            return
        if damage.absorbed > 0:
            self._emit(
                BattleEvent(self._round, EventType.ABSORB, source, target, damage.absorbed, defender.health)
            )
        if damage.amount > 0:
            event_type = EventType.CRITICAL_HIT if hit.critical else EventType.HIT
            self._emit(BattleEvent(self._round, event_type, source, target, damage.amount, defender.health))

        if not defender.is_alive():
            self._emit(BattleEvent(self._round, EventType.DEATH, source, target, 0, 0))

    def _emit(self, event: BattleEvent) -> None:
        self._events.append(event)
```

Обратите внимание, что бой ничего не знает о конкретных тактиках: он работает с интерфейсом `Tactic`, и
подставить ему можно любую реализацию. Если появится новая тактика, `Battle` менять не придется.

Ну и в отчете об экспедиции будем запоминать, с какой тактикой герой ее начинал:

**`labs/lab3/world/expedition_report.py`**

```python
from dataclasses import dataclass

from ..combat.battle_report import BattleReport
from .rewards import Rewards


# отчет об экспедиции: с каким сидом, куда, как прошел бой и что получил герой
@dataclass(frozen=True)
class ExpeditionReport:
    seed: int
    location_code: str
    tactic_code: str
    battle: BattleReport
    rewards: Rewards
```

Стратегия позволяет нам соблюдать `Open Closed` принцип, он дает нам возможность расширять поведение нашего класса без его изменения, путем
добавления новых реализаций стратегии. Стратегию можно выбирать как в методе, где она используется, так и в конструкторе, все зависит от
конкретной необходимости. В нашем случае стратегия выбирается в конструкторе, потому что по ходу боя она не меняется.

Применять стратегию стоит, когда одно и то же действие можно выполнить несколькими способами и способ нужно выбирать
или менять во время работы программы.

И давайте сразу обновим `Location`: она будет принимать тактику и передавать дальше в `Battle`.

Получится что-то такое:

**`labs/lab3/world/location.py`**

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
from ..enemy.armored_enemy import ArmoredEnemy
from ..enemy.legendary_enemy import LegendaryEnemy
from ..enemy.windfury_enemy import WindfuryEnemy
from ..tactic.tactic import Tactic
from ..tactic.balanced_tactic import BalancedTactic

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
    def explore(self, hero: Hero, seed: int | None = None, tactic: Tactic | None = None) -> ExpeditionReport:
        dice = Dice()
        if seed is None:
            seed = dice.next_seed()
        dice.reseed(seed)

        tactic = tactic if tactic is not None else BalancedTactic()
        hero.rest()
        enemy = self._create_enemy()
        if dice.chance(0.2):
            enemy = self._make_elite(enemy, dice)
        battle = Battle(hero, enemy, tactic).run()

        if battle.outcome == Outcome.VICTORY:
            rewards = self._reward(hero, enemy, dice)
        else:
            rewards = self._punish(hero)

        return ExpeditionReport(seed, self._code, tactic.code, battle, rewards)

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

    def _make_elite(self, enemy: Enemy, dice: Dice) -> Enemy:
        both = dice.chance(0.25)
        if dice.chance(0.5):
            enemy = LegendaryEnemy(enemy)
            if both:
                enemy = WindfuryEnemy(enemy)
        else:
            enemy = WindfuryEnemy(enemy)
            if both:
                enemy = LegendaryEnemy(enemy)
        return ArmoredEnemy(enemy) if dice.chance(0.3) else enemy
```

Если тактика не передана, используется `BalancedTactic` — обычная атака каждый раунд.

## Достижения и журнал экспедиций (Наблюдатель / Observer)

Теперь реализуем еще два важных и полезных механизма. Добавим в нашу игру достижения и журнал экспедиций!
Журнал будет фиксировать результаты экспедиций (победа/поражение) и его можно будет прочитать в лагере.
Достижения будут выдаваться за определенные события, происходящие в игре.

Заметим, что оба типа функционала реагируют на то, что происходит в игре. И в целом пересекаются! И журналу, и ачивкам надо реагировать на
окончание экспедиции (журнал записывает его, а достижения потенциально выдаются на основе этого результата).

Мы можем захардкодить это прямо в код экспедиции, однако тогда будет довольно сложно расширять обработку этих событий:
добавление обработчиков потребует модификации кода, который управляет экспедициями, это не очень круто.

Вместо этого давайте реализуем этот механизм с помощью паттерна *Наблюдатель*. Он создает механизм подписки: одни объекты
(подписчики/`listeners/observers`) следят за событиями другого объекта (издателя/`publisher`), а издатель оповещает всех подписчиков, когда
событие
происходит, ничего не зная о том, что они делают.

Объявим типы событий, на которые можно подписаться:

**`labs/lab3/event/game_event_type.py`**

```python
from enum import Enum


class GameEventType(Enum):
    VICTORY = "victory"
    DEFEAT = "defeat"
    LEVEL_UP = "level-up"
```

И само событие. Обратите внимание, что это просто данные: что произошло, с каким героем и в какой экспедиции:

**`labs/lab3/event/game_event.py`**

```python
from dataclasses import dataclass
from ..hero.hero import Hero
from ..world.expedition_report import ExpeditionReport
from .game_event_type import GameEventType


@dataclass(frozen=True)
class GameEvent:
    type: GameEventType
    hero: Hero
    expedition: ExpeditionReport
```

Интерфейс для подписчиков содержит всего один метод, который вызывается при наступлении события, и определяет контракт обработки этого
изменения:

**`labs/lab3/event/game_event_listener.py`**

```python
from typing import Protocol
from .game_event import GameEvent


class GameEventListener(Protocol):
    def on_event(self, event: GameEvent) -> None: ...
```

`GameEventListener` тоже описан через `Protocol`: издателю достаточно, чтобы у слушателя был метод `on_event`.

Именно издатель позволяет подписываться на определенный тип события. Также именно он оповещает подписчиков о
наступлении того или иного события:

**`labs/lab3/event/game_event_publisher.py`**

```python
from .game_event import GameEvent
from .game_event_type import GameEventType
from .game_event_listener import GameEventListener


class GameEventPublisher:
    def __init__(self) -> None:
        self._listeners: dict[GameEventType, list[GameEventListener]] = {}

    def subscribe(self, type: GameEventType, listener: GameEventListener) -> None:
        self._listeners.setdefault(type, []).append(listener)

    def unsubscribe(self, type: GameEventType, listener: GameEventListener) -> None:
        listeners = self._listeners.get(type, [])
        if listener in listeners:
            listeners.remove(listener)

    def publish(self, event: GameEvent) -> None:
        for listener in tuple(self._listeners.get(event.type, [])):
            listener.on_event(event)
```

Подписчики хранятся в `_listeners`, причем хранятся по типу ивента, то есть подписаться и реагировать они могут на отдельные типы, а не на
все.
Реакции вызываются в методе `publish`. Когда в нашей системе происходит ивент – мы вызываем издателя, а он, по списку зарегистрированных
подписчиков, вызывает их
`on_event` методы, прокидывая туда информацию о ивенте.

Теперь реализуем двух конкретных подписчиков. Сначала того, который будет выдавать достижения:

**`labs/lab3/progress/achievement_tracker.py`**

```python
from ..event.game_event import GameEvent
from ..event.game_event_type import GameEventType
from .achievement import Achievement


class AchievementTracker:
    def on_event(self, event: GameEvent) -> None:
        hero = event.hero
        if event.type == GameEventType.VICTORY:
            hero.unlock(Achievement.FIRST_BLOOD)
            if event.expedition.battle.enemy.code == "dragon":
                hero.unlock(Achievement.DRAGON_SLAYER)
            loot = event.expedition.rewards.loot
            if loot is not None and loot.enchantments:
                hero.unlock(Achievement.LUCKY)
        elif event.type == GameEventType.LEVEL_UP and hero.level >= 5:
            hero.unlock(Achievement.VETERAN)
```

А теперь второго, он будет записывать экспедиции в журнал:

**`labs/lab3/progress/expedition_journal.py`**

```python
from ..event.game_event import GameEvent
from ..world.expedition_report import ExpeditionReport


class ExpeditionJournal:
    def __init__(self) -> None:
        self._entries: dict[str, list[ExpeditionReport]] = {}

    def on_event(self, event: GameEvent) -> None:
        self._entries.setdefault(event.hero.name, []).append(event.expedition)

    def get_entries(self, hero_name: str) -> tuple[ExpeditionReport, ...]:
        return tuple(self._entries.get(hero_name, []))
```

Отлично, осталось только настроить связи между издателем и подписчиками, а также прописать саму отправку событий, это
мы сделаем чуть позже, в фасаде.

Наблюдатель супер полезный паттерн, если на изменение одного объекта должны реагировать другие, а их состав заранее неизвестен или может
меняться.
Особенно хорош он когда состав подписчиков может меняться в зависимости от конфигурации программы (например, какие-то обработчики
отключаемы).

## Защита (Цепочка обязанностей / Chain of Responsibility)

Последнее новое функциональное требование к нашему коду! Мы очень хотим разнообразить методы защиты от урона для наших героев.
Мы должны уметь уворачиваться, с некоторым шансом полностью нивелируя урон. Использовать магические барьеры, поглощая часть урона,
восстанавливая
барьер в лагере. Ну и защищаться броней. Это в целом у нас уже есть, но мы это чуть адаптируем.

Тут важно, что у нас получается несколько обработчиков урона. Все они могут быть вместе (прокачанный герой может и иметь шанс уворота, и
магический барьер, и хорошую броню).
А могут и не быть! То есть состав непостоянный. А еще в будущем мы легком можем добавить и новых обработчиков.

Теперь займёмся защитой. Сейчас она у каждого бойца своя и зашита прямо в код: герой учитывает броню в методе `receive_hit`, а
скелет и дракон переопределяют метод `_reduce_damage`. Мы же хотим добавить новые механики: следопыт должен иногда
уворачиваться от ударов, а маг - ставить перед экспедицией магический барьер, который забирает на себя урон, пока не
иссякнет. И не факт, что список конечный: через несколько релизов может появиться еще несколько механик защиты.

Обработка всех этих эффектов в одном методе кажется нам излишне сложной. В целом с учетом того, что защита
срабатывает пошагово (сначала уклонение, потом броня, потом барьер), здесь можно применить паттерн *Цепочка
обязанностей*.

Он позволяет передавать запрос по цепочке обработчиков: каждый обработчик сам решает, обработать запрос
и передать его дальше или остановить его обработку. Отправитель запроса при этом не знает, сколько в цепочке звеньев и
какие именно они.

Цепочка наша будет обрабатывать уже объявленный `Damage`. Осталось определить обработчиков, которые изменяют `Damage` и передают дальше.

Объявим базовый класс обработчика. В нем два метода: установка следующего звена для текущего (с помощью него мы и будем
строить цепочку) и обработка урона:

**`labs/lab3/defense/damage_handler.py`**

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from .damage import Damage


class DamageHandler(ABC):
    def __init__(self) -> None:
        self._next: DamageHandler | None = None

    def set_next(self, handler: DamageHandler) -> DamageHandler:
        self._next = handler
        return handler

    @abstractmethod
    def handle(self, damage: Damage) -> None: ...

    def _pass_to_next(self, damage: Damage) -> None:
        if self._next is not None:
            self._next.handle(damage)
```

Теперь конкретный обработчик, который будет хендлить уклонение: если оно сработало, удар по герою не прошел, и передавать урон дальше по
цепочке незачем:

**`labs/lab3/defense/evasion_handler.py`**

```python
from .damage import Damage
from .damage_handler import DamageHandler
from ..core.dice import Dice


class EvasionHandler(DamageHandler):
    def __init__(self, chance: float) -> None:
        super().__init__()
        self._chance = chance

    def handle(self, damage: Damage) -> None:
        if Dice().chance(self._chance):
            damage.evade()
            return
        self._pass_to_next(damage)
```

Броня гасит долю урона и передает то, что осталось, следующему обработчику. Обратите внимание, что это то же самое правило,
что раньше было зашито в герое, скелете и драконе, а теперь оно записано в одном месте:

**`labs/lab3/defense/armor_handler.py`**

```python
from .damage import Damage
from .damage_handler import DamageHandler


class ArmorHandler(DamageHandler):
    def __init__(self, defense: float) -> None:
        super().__init__()
        self._defense = defense

    def handle(self, damage: Damage) -> None:
        damage.reduce_to(max(1, int(damage.amount * (1 - self._defense) + 0.5)))
        self._pass_to_next(damage)
```

И магический барьер, который поглощает урон, пока у него есть запас прочности:

**`labs/lab3/defense/barrier_handler.py`**

```python
from .damage import Damage
from .damage_handler import DamageHandler


class BarrierHandler(DamageHandler):
    def __init__(self, strength: int) -> None:
        super().__init__()
        self._strength = strength

    def handle(self, damage: Damage) -> None:
        absorbed = min(self._strength, damage.amount)
        if absorbed > 0:
            self._strength -= absorbed
            damage.absorb(absorbed)
        self._pass_to_next(damage)
```

Какие звенья будут в цепочке, зависит от героя. Цепочку собирает его класс - так же, как класс сам знает свою
фабрику стартового снаряжения:

**`labs/lab3/hero/hero_class.py`**

```python
from __future__ import annotations

from enum import Enum
from ..defense.armor_handler import ArmorHandler
from ..defense.barrier_handler import BarrierHandler
from ..defense.evasion_handler import EvasionHandler
from ..defense.damage_handler import DamageHandler
from ..item.armor import Armor

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

    def __init__(
        self, code: str, display_name: str, base_health: int, kit_factory: type[StarterKitFactory]
    ) -> None:
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

    def create_defense(self, armor: Armor, level: int) -> DamageHandler:
        evasion = 0.15 if self == HeroClass.RANGER else 0.0
        barrier = 30 if self == HeroClass.MAGE else 0
        if self == HeroClass.WARRIOR and level >= 3:
            barrier = 20
        elif self == HeroClass.RANGER and level >= 5:
            barrier = 15
        elif self == HeroClass.MAGE and level >= 3:
            evasion = 0.10
        armor_handler = ArmorHandler(armor.defense)
        first = armor_handler
        if evasion > 0:
            first = EvasionHandler(evasion)
            first.set_next(armor_handler)
        if barrier > 0:
            armor_handler.set_next(BarrierHandler(barrier))
        return first
```

Обратите внимание, что цепочка строится динамически. Обработчики, которых у класса нет, в цепочку просто не попадают. А еще в
цепочке важен порядок звеньев: если поставить барьер перед броней, он будет поглощать урон до брони и иссякнет быстрее.

С ростом уровня защита дополняется: воин с 3-го уровня получает барьер на 20 единиц, следопыт с 5-го — барьер на 15, а маг с 3-го —
уклонение с вероятностью 10%. У следопыта сохраняется уклонение 15%, у мага — барьер на 30 единиц. Новые способности учитываются при
следующей сборке цепочки, в частности при отдыхе перед экспедицией; урон проходит в порядке "уклонение → броня → барьер".

Теперь сам герой. Урон он больше не считает сам, а передает в первое звено своей цепочки. Перед каждой экспедицией
цепочка собирается заново, поэтому барьер мага восстанавливается, а после смены брони защита сразу учитывает новую
броню:

**`labs/lab3/hero/hero.py`**

```python
from __future__ import annotations

from ..combat.combatant import Combatant
from ..combat.hit import Hit
from ..item.armor import Armor
from ..item.item import Item
from ..item.weapon import Weapon
from ..kit.starter_kit_factory import StarterKitFactory
from .hero_class import HeroClass
from ..defense.damage import Damage
from ..progress.achievement import Achievement

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
        self._experience = builder.experience
        self._gold = builder.gold
        self._weapon: Weapon = builder.weapon
        self._armor: Armor = builder.armor
        self._inventory = list(builder.inventory)
        self._achievements = list(dict.fromkeys(builder.achievements))
        self.rest()

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
            self.experience = 0
            self.inventory: list[Item] = []
            self.achievements: list[Achievement] = []

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

        def set_experience(self, experience: int) -> Hero.Builder:
            self.experience = experience
            return self

        def set_inventory(self, inventory: list[Item]) -> Hero.Builder:
            self.inventory = inventory
            return self

        def set_achievements(self, achievements: list[Achievement]) -> Hero.Builder:
            self.achievements = achievements
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

            if self.experience < 0:
                raise ValueError("Опыт не может быть отрицательным")
            return Hero(self)

    def roll_damage(self) -> Hit:
        hit = self._weapon.roll_damage()
        return Hit(hit.damage + (self._level - 1) * DAMAGE_PER_LEVEL, hit.critical)

    def receive_hit(self, amount: int) -> Damage:
        damage = Damage(amount)
        self._defense.handle(damage)
        self._health = max(0, self._health - damage.amount)
        return damage

    def is_alive(self) -> bool:
        return self._health > 0

    # поведение героя между экспедициями

    # перед каждой экспедицией герой успевает отдохнуть
    def rest(self) -> None:
        self._health = self.max_health
        self._defense = self._hero_class.create_defense(self._armor, self._level)

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
            self._defense = self._hero_class.create_defense(self._armor, self._level)
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

    def unlock(self, achievement: Achievement) -> bool:
        if achievement in self._achievements:
            return False
        self._achievements.append(achievement)
        return True

    @property
    def achievements(self) -> tuple[Achievement, ...]:
        return tuple(self._achievements)
```

У врагов цепочка задается прямо в конструкторе. Общий класс врага пропускает урон через цепочку, если она есть:

**`labs/lab3/enemy/abstract_enemy.py`**

```python
from __future__ import annotations
from ..defense.damage import Damage
from ..defense.damage_handler import DamageHandler
from ..combat.hit import Hit
from ..core.dice import Dice
from .enemy import Enemy


# общая логика для врагов: здоровье, урон в диапазоне, награда
class AbstractEnemy(Enemy):

    def __init__(
        self,
        code: str,
        name: str,
        max_health: int,
        min_damage: int,
        max_damage: int,
        experience_reward: int,
        gold_reward: int,
        defense: DamageHandler | None = None,
    ) -> None:
        self._code = code
        self._name = name
        self._max_health = max_health
        self._min_damage = min_damage
        self._max_damage = max_damage
        self._experience_reward = experience_reward
        self._gold_reward = gold_reward
        self._health = max_health
        self._defense = defense

    def roll_damage(self) -> Hit:
        return Hit(Dice().roll(self._min_damage, self._max_damage), False)

    def receive_hit(self, amount: int) -> Damage:
        damage = Damage(amount)
        if self._defense is not None:
            self._defense.handle(damage)
        self._health = max(0, self._health - damage.amount)
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

У гоблина защиты нет:

**`labs/lab3/enemy/goblin.py`**

```python
from .abstract_enemy import AbstractEnemy


# гоблин - самый простой враг, без особенностей
class Goblin(AbstractEnemy):

    def __init__(self) -> None:
        super().__init__("goblin", "Гоблин", 50, 6, 14, 40, 30)
```

Кости скелета и чешуя дракона работают как броня, поэтому метод `_reduce_damage` им больше не нужен:

**`labs/lab3/enemy/skeleton.py`**

```python
from ..defense.armor_handler import ArmorHandler
from .abstract_enemy import AbstractEnemy

RESISTANCE = 0.3


# скелет: кости плохо поддаются ударам, поэтому часть урона проходит мимо
class Skeleton(AbstractEnemy):

    def __init__(self) -> None:
        super().__init__("skeleton", "Скелет", 110, 10, 24, 90, 70, ArmorHandler(RESISTANCE))
```

**`labs/lab3/enemy/dragon.py`**

```python
from ..defense.armor_handler import ArmorHandler
from ..combat.hit import Hit
from ..core.dice import Dice
from .abstract_enemy import AbstractEnemy

RESISTANCE = 0.25
FIRE_BREATH_CHANCE = 0.25
FIRE_BREATH_MULTIPLIER = 2


# дракон защищен чешуей и время от времени дышит огнем
class Dragon(AbstractEnemy):

    def __init__(self) -> None:
        super().__init__("dragon", "Дракон", 260, 16, 22, 500, 400, ArmorHandler(RESISTANCE))

    def roll_damage(self) -> Hit:
        bite = super().roll_damage()
        if Dice().chance(FIRE_BREATH_CHANCE):
            return Hit(bite.damage * FIRE_BREATH_MULTIPLIER, True)

        return bite
```

Супер! Теперь, чтобы добавить новую механику защиты, достаточно написать новое звено и поставить его в нужное место
цепочки, ни герой, ни враги при этом не меняются.

Применять цепочку стоит, когда запрос обрабатывается в несколько шагов и набор или порядок шагов должен меняться без
переписывания кода, который этот запрос отправляет.

## Игра целиком (Фасад / Facade)

Посмотрите на метод `main` из прошлой лабы: он сам создает каталог, локации и торговца, сам собирает героя строителем
и сам берет у класса фабрику стартового снаряжения. Теперь к этому добавятся тактики, хранилище с заместителем,
издатель событий и подписчики. Если оставить все это в консольном коде, он будет знать обо всех классах игры, а
заменить консоль на что-то другое (спойлер-спойлер) станет очень непросто.

Поэтому давайте воспользуемся паттерном *Фасад*. Он предоставляет простой интерфейс к сложной подсистеме из многих
классов: вся настройка и все связи между этими классами спрятаны внутри фасада, а снаружи видны только несколько
понятных методов.

Наш фасад - это вся игра целиком:

**`labs/lab3/game/game_facade.py`**

```python
from pathlib import Path
from ..camp.merchant import Merchant
from ..combat.outcome import Outcome
from ..core.unknown_code_error import UnknownCodeError
from ..hero.hero import Hero
from ..hero.hero_class import HeroClass
from ..item.default_catalog import (
    create_default_catalog,
    IRON_SWORD,
    ELVEN_BOW,
    EMBER_STAFF,
    CHAINMAIL,
    PLATE_ARMOR,
)
from ..item.item import Item
from ..world.forest import Forest
from ..world.haunted_manor import HauntedManor
from ..world.crypt import Crypt
from ..world.dragon_lair import DragonLair
from ..world.location_registry import LocationRegistry
from ..world.expedition_report import ExpeditionReport
from ..tactic.balanced_tactic import BalancedTactic
from ..tactic.aggressive_tactic import AggressiveTactic
from ..tactic.cautious_tactic import CautiousTactic
from ..tactic.tactic import Tactic
from ..event.game_event import GameEvent
from ..event.game_event_type import GameEventType
from ..event.game_event_publisher import GameEventPublisher
from ..progress.achievement_tracker import AchievementTracker
from ..progress.expedition_journal import ExpeditionJournal
from ..storage.file_hero_repository import FileHeroRepository
from ..storage.caching_hero_repository import CachingHeroRepository


class GameFacade:
    def __init__(self, save_file: Path) -> None:
        self._catalog = create_default_catalog()
        self._locations = LocationRegistry(
            [
                Forest(self._catalog),
                HauntedManor(self._catalog),
                Crypt(self._catalog),
                DragonLair(self._catalog),
            ]
        )
        self.tactics = (BalancedTactic(), AggressiveTactic(), CautiousTactic())
        self._merchant = Merchant(self._catalog, [IRON_SWORD, ELVEN_BOW, EMBER_STAFF, CHAINMAIL, PLATE_ARMOR])
        self._heroes = CachingHeroRepository(FileHeroRepository(save_file, self._catalog))
        self._events = GameEventPublisher()
        self._journal = ExpeditionJournal()
        achievements = AchievementTracker()
        self._events.subscribe(GameEventType.VICTORY, achievements)
        self._events.subscribe(GameEventType.LEVEL_UP, achievements)
        self._events.subscribe(GameEventType.VICTORY, self._journal)
        self._events.subscribe(GameEventType.DEFEAT, self._journal)

    @property
    def hero_classes(self):
        return tuple(HeroClass)

    def get_starter_kit(self, code: str) -> tuple[Item, Item]:
        kit = HeroClass.by_code(code).starter_kit(self._catalog)
        return kit.create_weapon(), kit.create_armor()

    @property
    def locations(self):
        return self._locations.all

    def get_location(self, code: str):
        return self._locations.get(code)

    def get_tactic(self, code: str) -> Tactic:
        for tactic in self.tactics:
            if tactic.code == code:
                return tactic
        raise UnknownCodeError("Тактика", code)

    @property
    def showcase(self):
        return self._merchant.showcase

    def sell_price(self, item: Item) -> int:
        return self._merchant.sell_price(item)

    @property
    def heroes(self) -> list[Hero]:
        return self._heroes.find_all()

    def get_hero(self, name: str) -> Hero:
        hero = self._heroes.find_by_name(name)
        if hero is None:
            raise ValueError("Героя с именем " + name + " нет")
        return hero

    def create_hero(self, name: str, code: str) -> Hero:
        if self._heroes.find_by_name(name) is not None:
            raise ValueError("Герой с именем " + name + " уже есть")
        hero_class = HeroClass.by_code(code)
        hero = (
            Hero.Builder()
            .set_name(name)
            .set_hero_class(hero_class)
            .set_starter_kit(hero_class.starter_kit(self._catalog))
            .build()
        )
        self._heroes.save(hero)
        return hero

    def explore(self, hero_name: str, location_code: str, tactic_code: str) -> ExpeditionReport:
        hero = self.get_hero(hero_name)
        report = self.get_location(location_code).explore(hero, tactic=self.get_tactic(tactic_code))
        result = GameEventType.VICTORY if report.battle.outcome == Outcome.VICTORY else GameEventType.DEFEAT
        self._events.publish(GameEvent(result, hero, report))
        if report.rewards.levels_gained > 0:
            self._events.publish(GameEvent(GameEventType.LEVEL_UP, hero, report))
        self._heroes.save(hero)
        return report

    def equip(self, hero_name: str, index: int) -> Item:
        hero = self.get_hero(hero_name)
        item = hero.inventory[index]
        hero.equip(item)
        self._heroes.save(hero)
        return item

    def buy(self, hero_name: str, code: str) -> Item:
        hero = self.get_hero(hero_name)
        item = self._merchant.buy(hero, code)
        self._heroes.save(hero)
        return item

    def sell(self, hero_name: str, index: int) -> int:
        hero = self.get_hero(hero_name)
        price = self._merchant.sell(hero, hero.inventory[index])
        self._heroes.save(hero)
        return price

    def get_journal(self, name: str):
        return self._journal.get_entries(name)
```

Вот здесь и настраиваются связи между издателем и подписчиками, а также происходит отправка событий после экспедиции.
Одно и то же событие (победу) получают оба подписчика: и выдача достижений, и журнал. Если завтра мы захотим, например,
отправлять уведомления о победах в какой-нибудь мессенджер, достаточно будет написать нового подписчика и подписать его в конструкторе фасада.

Обратите внимание, что для экспедиций и операций со снаряжением клиент передает фасаду имя героя, коды или индексы выбранных объектов.
Фасад сам находит героя в хранилище (тут и работает кэш заместителя) и сохраняет его после изменений.
При этом клиент по-прежнему использует игровые объекты, например для вывода информации: фасад скрывает настройку
и взаимодействие подсистем, но не всю предметную модель.

Адаптер и фасад довольно похожи, ведь оба оборачивают другие объекты. Но адаптер приводит интерфейс одного
объекта к другому, уже существующему интерфейсу, который ожидает клиент, а фасад определяет новый, упрощенный
интерфейс сразу к целой подсистеме.

В реализации фасада важно, чтобы он не был объектом, который знает обо всем и делает все. Поэтому
фасад не должен сам содержать игровую логику: он только связывает между собой классы, которые ее реализуют.
Применять фасад стоит, когда нужно дать простой вход в сложную подсистему или отделить от нее клиентский код.

## Консоль

Вывод в консоль научился показывать новые события боя, особенности классов и достижения:

**`labs/lab3/console_view.py`**

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

        if hero.achievements:
            print("  Достижения: " + ", ".join(a.display_name for a in hero.achievements))

    def summarize(self, report: ExpeditionReport, location_name: str, tactic_name: str) -> str:
        result = "победа" if report.battle.outcome == Outcome.VICTORY else "поражение"
        return f'{location_name}, {report.battle.enemy.name}, тактика "{tactic_name}": {result}'

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
        elif event.type == EventType.EVADE:
            print(f"[раунд {event.round}] {source.name} -> {target.name}: промах, {target.name} уклоняется")
        elif event.type == EventType.ABSORB:
            print(
                f"[раунд {event.round}] {source.name} -> {target.name}: барьер поглощает {event.amount} урона"
            )
        elif event.type == EventType.POWER_STRIKE:
            print(f"[раунд {event.round}] {source.name} бьет в полную силу")
        elif event.type == EventType.DEFEND:
            print(f"[раунд {event.round}] {source.name} уходит в оборону")
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

А игровой цикл в методе `main` теперь работает только с фасадом и хранит текущего героя:

Вместо прямой работы с каталогом, локациями и торговцем:

- создаём `GameFacade` с файлом сохранения и выбираем нового или сохранённого героя;
- перед экспедицией выбираем тактику;
- работам с экспедициями, экипировкой и торговлей через фасад;
- добавляем пункт меню для журнала экспедиций.

**`labs/lab3/main.py`**

```python
from __future__ import annotations

from typing import Sequence
from pathlib import Path
from .game.game_facade import GameFacade

from .combat.outcome import Outcome
from .console_view import ConsoleView
from .hero.hero import Hero
from .hero.hero_class import HeroClass
from .item.item import Item

VIEW = ConsoleView()


def main() -> None:
    save_file = Path("saves") / "heroes.json"
    print("Файл сохранения: " + str(save_file.resolve()))
    game = GameFacade(save_file)
    hero = choose_hero(game)
    print(f"{hero.name} разбивает лагерь на опушке мистического леса.")

    while True:
        print()
        VIEW.print_hero(hero)
        print("1 - отправиться в экспедицию, 2 - снаряжение, 3 - торговец, 4 - журнал, 0 - выйти из игры")
        choice = input().strip()
        if choice == "1":
            go_on_expedition(hero, game)
        elif choice == "2":
            manage_equipment(hero, game)
        elif choice == "3":
            visit_merchant(hero, game)
        elif choice == "4":
            for report in game.get_journal(hero.name):
                print(
                    VIEW.summarize(
                        report,
                        game.get_location(report.location_code).display_name,
                        game.get_tactic(report.tactic_code).display_name,
                    )
                )
        elif choice == "0":
            return
        else:
            print("Такого варианта нет")


def choose_hero(game: GameFacade) -> Hero:
    heroes = game.heroes
    if not heroes:
        return create_hero(game)
    print("Выберите сохраненного героя (0 - создать нового):")
    for i, hero in enumerate(heroes):
        print(f"{i + 1} - {hero.name}, {hero.hero_class.display_name} {hero.level} ур.")
    index = read_choice(len(heroes))
    return heroes[index] if index >= 0 else create_hero(game)


def create_hero(game: GameFacade) -> Hero:
    print("Создайте своего героя!")
    while True:
        print("Выберите класс:")
        hero_classes = list(HeroClass)
        for i, hero_class in enumerate(hero_classes):
            weapon, armor = game.get_starter_kit(hero_class.code)
            print(
                f"{i + 1} - {hero_class.display_name} (здоровье {hero_class.base_health}): "
                f"{VIEW.describe(weapon)}, {VIEW.describe(armor)}"
            )
        class_index = read_choice(len(hero_classes))
        if class_index < 0:
            continue
        hero_class = hero_classes[class_index]

        print("Введите имя:")
        name = input().strip()

        try:
            return game.create_hero(name, hero_class.code)
        except ValueError as error:
            print(f"Не получилось создать героя: {error}")


def go_on_expedition(hero: Hero, game: GameFacade) -> None:
    all_locations = game.locations
    print("Куда отправимся?")
    for i, location in enumerate(all_locations):
        print(f"{i + 1} - {location.display_name} (рекомендуемый уровень {location.recommended_level})")
    index = read_choice(len(all_locations))
    if index < 0:
        return

    location = all_locations[index]
    print("Выберите тактику:")
    for i, tactic in enumerate(game.tactics):
        print(f"{i + 1} - {tactic.display_name}: {tactic.description}")
    tactic_index = read_choice(len(game.tactics))
    if tactic_index < 0:
        return
    report = game.explore(hero.name, location.code, game.tactics[tactic_index].code)
    print()
    VIEW.print_report(report, location.display_name)

    if report.battle.outcome == Outcome.VICTORY and report.battle.enemy.code == "dragon":
        print(f"\nДракон повержен! Барды еще долго будут петь о подвиге героя по имени {hero.name}.")


def manage_equipment(hero: Hero, game: GameFacade) -> None:
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
    game.equip(hero.name, index)
    print("Экипировано: " + VIEW.describe(item))


def visit_merchant(hero: Hero, game: GameFacade) -> None:
    print(f'Торговец: "Чего желаешь?" (золото: {hero.gold})')
    print("1 - купить, 2 - продать, 0 - назад")
    choice = input().strip()
    if choice == "1":
        buy(hero, game)
    elif choice == "2":
        sell(hero, game)


def buy(hero: Hero, game: GameFacade) -> None:
    showcase = game.showcase
    for i, item in enumerate(showcase):
        print(f"{i + 1} - {VIEW.describe(item)}: {item.price} золота")
    index = read_choice(len(showcase))
    if index < 0:
        return

    try:
        item = game.buy(hero.name, showcase[index].code)
        print("Куплено: " + VIEW.describe(item))
    except ValueError as error:
        print(error)


def sell(hero: Hero, game: GameFacade) -> None:
    inventory = hero.inventory
    if not inventory:
        print("Продавать нечего (экипированные предметы продать нельзя)")
        return

    for i, item in enumerate(inventory):
        print(f"{i + 1} - {VIEW.describe(item)}: {game.sell_price(item)} золота")
    index = read_choice(len(inventory))
    if index < 0:
        return

    item = inventory[index]
    price = game.sell(hero.name, index)
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

Супер! Попробуйте запустить игру, сохраниться и вернуться к своему герою :)

## Демонстрация паттернов

Чтобы показать все паттерны одним запуском, напишем демонстрационную программу:

Демонстрацию лабы 3 заменяем новой:

**`labs/lab3/demo.py`**

```python
from pathlib import Path
from tempfile import TemporaryDirectory
from .console_view import ConsoleView
from .core.dice import Dice
from .combat.outcome import Outcome
from .defense.damage import Damage
from .defense.armor_handler import ArmorHandler
from .defense.barrier_handler import BarrierHandler
from .defense.evasion_handler import EvasionHandler
from .enemy.goblin import Goblin
from .enemy.dragon import Dragon
from .enemy.legendary_enemy import LegendaryEnemy
from .enemy.windfury_enemy import WindfuryEnemy
from .enemy.armored_enemy import ArmoredEnemy
from .enemy.haunted_weapon import HauntedWeapon
from .hero.hero import Hero
from .hero.hero_class import HeroClass
from .item.default_catalog import create_default_catalog, ELVEN_BOW
from .tactic.balanced_tactic import BalancedTactic
from .tactic.aggressive_tactic import AggressiveTactic
from .tactic.cautious_tactic import CautiousTactic
from .world.crypt import Crypt
from .world.forest import Forest
from .world.haunted_manor import HauntedManor
from .event.game_event import GameEvent
from .event.game_event_type import GameEventType
from .event.game_event_publisher import GameEventPublisher
from .progress.achievement_tracker import AchievementTracker
from .progress.expedition_journal import ExpeditionJournal
from .storage.file_hero_repository import FileHeroRepository
from .storage.caching_hero_repository import CachingHeroRepository
from .game.game_facade import GameFacade

VIEW = ConsoleView()


def new_hero(name, hero_class, level, catalog):
    return (
        Hero.Builder()
        .set_name(name)
        .set_hero_class(hero_class)
        .set_starter_kit(hero_class.starter_kit(catalog))
        .set_level(level)
        .build()
    )


def print_hits(title, chain, count):
    results = []
    for _ in range(count):
        damage = Damage(30)
        chain.handle(damage)
        if damage.evaded:
            results.append("уклонение")
        elif damage.absorbed:
            results.append(f"поглощено {damage.absorbed}, прошло {damage.amount}")
        else:
            results.append(f"прошло {damage.amount}")
    print(title + ": " + "; ".join(results))


class EventPrinter:
    def on_event(self, event):
        print(f"Событие {event.type.name}: герой {event.hero.name}")


def main():
    catalog = create_default_catalog()
    dice = Dice()
    print("===== Цепочка обязанностей =====")
    print_hits("Броня 30%", ArmorHandler(0.3), 2)
    chain = ArmorHandler(0.3)
    chain.set_next(BarrierHandler(30))
    print_hits("Броня 30% -> барьер 30", chain, 3)
    dice.reseed(3)
    chain = EvasionHandler(0.5)
    chain.set_next(ArmorHandler(0.3)).set_next(BarrierHandler(30))
    print_hits("Уклонение 50% -> броня 30% -> барьер 30", chain, 3)

    print("\n===== Стратегия =====")
    for tactic in (BalancedTactic(), AggressiveTactic(), CautiousTactic()):
        mage = new_hero("Мерлин", HeroClass.MAGE, 2, catalog)
        report = Crypt(catalog).explore(mage, 9, tactic)
        result = "победа" if report.battle.outcome == Outcome.VICTORY else "поражение"
        print(
            f"{tactic.display_name}: {result}, раундов {report.battle.events[-1].round}, "
            f"у героя осталось {mage.health}/{mage.max_health} здоровья"
        )

    print("\n===== Декоратор =====")
    for enemy in (
        Goblin(),
        LegendaryEnemy(Goblin()),
        WindfuryEnemy(Goblin()),
        LegendaryEnemy(WindfuryEnemy(Goblin())),
        ArmoredEnemy(Goblin()),
        ArmoredEnemy(LegendaryEnemy(WindfuryEnemy(Dragon()))),
    ):
        dice.reseed(5)
        hits = ", ".join(str(enemy.roll_damage().damage) for _ in range(3))
        damage = enemy.receive_hit(20)
        print(
            f"{enemy.name}: удары {hits}, награда {enemy.experience_reward} опыта; "
            f"входящий урон 20 -> {damage.amount}, здоровье {enemy.health}/{enemy.max_health}"
        )

    print("\n===== Адаптер =====")
    bow = catalog.create_weapon(ELVEN_BOW)
    enemy = HauntedWeapon(bow)
    print("Предмет: " + VIEW.describe(bow))
    print(f"Враг:    {enemy.name} ({enemy.max_health} здоровья, награда {enemy.experience_reward} опыта)")
    manor = HauntedManor(catalog)
    ranger = new_hero("Леголас", HeroClass.RANGER, 2, catalog)
    VIEW.print_report(manor.explore(ranger, 55, CautiousTactic()), manor.display_name)

    print("\n===== Наблюдатель =====")
    events = GameEventPublisher()
    journal = ExpeditionJournal()
    events.subscribe(GameEventType.VICTORY, AchievementTracker())
    events.subscribe(GameEventType.VICTORY, journal)
    events.subscribe(GameEventType.DEFEAT, journal)
    events.subscribe(GameEventType.VICTORY, EventPrinter())
    warrior = new_hero("Артур", HeroClass.WARRIOR, 1, catalog)
    report = Forest(catalog).explore(warrior, 7)
    result = GameEventType.VICTORY if report.battle.outcome == Outcome.VICTORY else GameEventType.DEFEAT
    events.publish(GameEvent(result, warrior, report))
    print("Достижения героя: " + ", ".join(a.display_name for a in warrior.achievements))
    print("Записей в журнале: " + str(len(journal.get_entries(warrior.name))))

    with TemporaryDirectory() as directory:
        print("\n===== Заместитель =====")
        save_file = Path(directory) / "heroes.json"
        file = FileHeroRepository(save_file, catalog)
        file.save(warrior)
        print("Содержимое файла:")
        print(save_file.read_text(encoding="utf-8"), end="")
        print(
            "Без заместителя два поиска дают один и тот же объект:",
            file.find_by_name("Артур") is file.find_by_name("Артур"),
        )
        cached = CachingHeroRepository(file)
        hero = cached.find_by_name("Артур")
        print("С заместителем два поиска дают один и тот же объект:", hero is cached.find_by_name("Артур"))
        hero.add_gold(100)
        cached.save(hero)
        print("После сохранения через заместитель золото героя в файле:", file.find_by_name("Артур").gold)

        print("\n===== Фасад =====")
        game = GameFacade(Path(directory) / "game.json")
        game.create_hero("Гэндальф", "mage")
        dice.reseed(11)
        report = game.explore("Гэндальф", "forest", "cautious")
        print(
            "Экспедиция через фасад: "
            + VIEW.summarize(
                report,
                game.get_location(report.location_code).display_name,
                game.get_tactic(report.tactic_code).display_name,
            )
        )
        VIEW.print_hero(game.get_hero("Гэндальф"))


if __name__ == "__main__":
    main()
```

Вывод программы:

```text
===== Цепочка обязанностей =====
Броня 30%: прошло 21; прошло 21
Броня 30% -> барьер 30: поглощено 21, прошло 0; поглощено 9, прошло 12; прошло 21
Уклонение 50% -> броня 30% -> барьер 30: уклонение; поглощено 21, прошло 0; уклонение

===== Стратегия =====
Баланс: поражение, раундов 9, у героя осталось 0/100 здоровья
Натиск: поражение, раундов 8, у героя осталось 0/100 здоровья
Осторожность: поражение, раундов 9, у героя осталось 0/100 здоровья

===== Декоратор =====
Гоблин: удары 10, 11, 14, награда 40 опыта; входящий урон 20 -> 20, здоровье 30/50
Легендарный Гоблин: удары 20, 22, 28, награда 80 опыта; входящий урон 20 -> 20, здоровье 30/50
Неистовый Гоблин: удары 21, 20, 22, награда 60 опыта; входящий урон 20 -> 20, здоровье 30/50
Легендарный Неистовый Гоблин: удары 42, 40, 44, награда 120 опыта; входящий урон 20 -> 20, здоровье 30/50
Бронированный Гоблин: удары 10, 11, 14, награда 40 опыта; входящий урон 20 -> 10, здоровье 40/50
Бронированный Легендарный Неистовый Дракон: удары 76, 124, 72, награда 1500 опыта; входящий урон 20 -> 8, здоровье 252/260

===== Адаптер =====
Предмет: Эльфийский лук (урон 14, шанс крита 40%)
Враг:    Одержимый Эльфийский лук (70 здоровья, награда 45 опыта)
=== Проклятый особняк (сид 55) ===
Леголас встречает врага: Одержимый Железный меч (85 здоровья)
[раунд 1] Леголас -> Одержимый Железный меч: КРИТ! 23 урона, осталось 62/85
[раунд 1] Одержимый Железный меч -> Леголас: 14 урона, осталось 101/115
[раунд 2] Леголас -> Одержимый Железный меч: 13 урона, осталось 49/85
[раунд 2] Одержимый Железный меч -> Леголас: промах, Леголас уклоняется
[раунд 3] Леголас -> Одержимый Железный меч: КРИТ! 23 урона, осталось 26/85
[раунд 3] Одержимый Железный меч -> Леголас: промах, Леголас уклоняется
[раунд 4] Леголас -> Одержимый Железный меч: 13 урона, осталось 13/85
[раунд 4] Одержимый Железный меч -> Леголас: 14 урона, осталось 87/115
[раунд 5] Леголас -> Одержимый Железный меч: 13 урона, осталось 0/85
Одержимый Железный меч повержен!
Победа! +40 опыта, +35 золота
Добыча: Эльфийский лук (урон 14, шанс крита 40%)

===== Наблюдатель =====
Событие VICTORY: герой Артур
Достижения героя: Первая кровь
Записей в журнале: 1

===== Заместитель =====
Содержимое файла:
[
  {
    "name": "Артур",
    "heroClass": "warrior",
    "level": 1,
    "experience": 40,
    "gold": 68,
    "weapon": {
      "code": "training-sword",
      "enchantments": []
    },
    "armor": {
      "code": "chainmail",
      "enchantments": []
    },
    "inventory": [
      {
        "code": "goblin-cleaver",
        "enchantments": []
      }
    ],
    "achievements": [
      "first-blood"
    ]
  }
]
Без заместителя два поиска дают один и тот же объект: False
С заместителем два поиска дают один и тот же объект: True
После сохранения через заместитель золото героя в файле: 168

===== Фасад =====
Экспедиция через фасад: Мистический лес, Бронированный Легендарный Гоблин, тактика "Осторожность": поражение
Гэндальф, Маг 1 ур. | здоровье 85 | опыт 0/100 | золото 25
  Оружие: Посох ученика (урон 10-18)
  Броня:  Льняная роба (защита 10%)
```

Не забудьте закоммитить и запушить изменения!

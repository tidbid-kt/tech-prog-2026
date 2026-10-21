# Лаба 3 — практика на C++

Это практическая часть [лабораторной работы 3](../laba3.md): требования к сдаче и вопросы к защите находятся в
основном тексте лабы, а здесь шаг за шагом разобрано эталонное решение.

Понадобится компилятор с поддержкой C++20 (GCC 11+, Clang 14+ или Visual Studio 2022) и CMake 3.20+, сторонние библиотеки не нужны. Шаблон репозитория: [tech-prog-labs-2026-cpp](https://github.com/tidbid-kt/tech-prog-labs-2026-cpp). Код лабы пишите в папке `labs/lab3`.

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

```cpp
#pragma once

#include <cstddef>
#include <cstdint>
#include <random>
#include <vector>

namespace lab3 {

// Одиночка: общий источник случайности на всю игру
class Dice {
public:
    // метод для получения одиночки
    static Dice& getInstance();

    // копировать одиночку нельзя, иначе появился бы второй экземпляр
    Dice(const Dice&) = delete;
    Dice& operator=(const Dice&) = delete;

    // после установки сида все последующие броски повторяются в точности
    void reseed(std::uint32_t seed);

    // случайный сид
    std::uint32_t nextSeed();

    // случайное целое число от min до max включительно
    int roll(int min, int max);

    // возвращает true с вероятностью probability (от 0 до 1)
    bool chance(double probability);

    // случайный элемент списка
    template <typename T>
    const T& pick(const std::vector<T>& options) {
        int index = roll(0, static_cast<int>(options.size()) - 1);
        return options[static_cast<std::size_t>(index)];
    }

private:
    // конструктор приватный - самостоятельно создать экземпляр класса нельзя
    Dice();

    std::mt19937 random_;
};

}  // namespace lab3
```
```cpp
#include "core/dice.hpp"

namespace lab3 {

Dice& Dice::getInstance() {
    // единственный экземпляр - статическая переменная функции:
    // она создается при первом вызове метода и живет до конца программы
    static Dice instance;
    return instance;
}

Dice::Dice() : random_(std::random_device{}()) {}

void Dice::reseed(std::uint32_t seed) {
    random_.seed(seed);
}

std::uint32_t Dice::nextSeed() {
    return static_cast<std::uint32_t>(roll(0, 999999));
}

int Dice::roll(int min, int max) {
    std::uniform_int_distribution<int> distribution(min, max);
    return distribution(random_);
}

bool Dice::chance(double probability) {
    std::uniform_real_distribution<double> distribution(0.0, 1.0);
    return distribution(random_) < probability;
}

}  // namespace lab3
```

Единственный экземпляр здесь - статическая локальная переменная метода `getInstance()` (такую реализацию называют
одиночкой Майерса). Копирующий конструктор и присваивание удалены (`= delete`): без этого можно было бы написать
`Dice copy = Dice::getInstance();` и получить второй экземпляр.

Давайте разберем, что в этой реализации важно.

1. Конструктор приватный, а копирование запрещено. Создать кость самостоятельно нельзя, получить ее можно только через
   `getInstance()`.
2. Экземпляр создается лениво, при первом вызове `getInstance()`, и живет до конца программы. Пока кость никому не
   нужна, ее нет.
3. Со звездочкой заметим, что такая реализация потокобезопасна: начиная с C++11 стандарт гарантирует, что статическая локальная переменная будет
   инициализирована ровно один раз, даже если функцию одновременно вызовут несколько потоков. Тему многопоточности мы здесь затрагивать не будем, но для самостоятельного изучения она очень интересна.

Преимущество одиночки очевидно - у всех составляющих игры (оружия, врагов, локаций) одна и та же кость, поэтому один сид задает все броски. 
А еще кость не нужно передавать через конструкторы каждого предмета и каждого врага - к ней можно
обратиться откуда угодно.

Но тут есть и обратная сторона: одиночка - это глобальное состояние. Лук, посох и враги зависят от игральной кости, однако
по их конструкторам этого не видно, а подменить кость на другую (например, в тестах) не получится. Поэтому применять
одиночку стоит только там, где объект действительно должен быть единственным.

Также давайте сразу заведем исключение, которое будем бросать, если по коду ничего не нашлось (неизвестная локация,
неизвестный предмет и т.д.), в будущем оно нам пригодится:

```cpp
#pragma once

#include <stdexcept>
#include <string>

namespace lab3 {

class UnknownCodeError : public std::runtime_error {
public:
    UnknownCodeError(const std::string& what, const std::string& code)
        : std::runtime_error(what + " с кодом '" + code + "' не существует") {}
};

}  // namespace lab3
```

## Боевая механика

Давайте определим базовые абстракции нашей игры.

Интерфейс сущности, которая умеет атаковать и получать урон - так с точки зрения механики выглядят и герой, и враги:

```cpp
#pragma once

#include <string>

#include "combat/hit.hpp"

namespace lab3 {

// всё, что может участвовать в бою: и герой, и враги
class Combatant {
public:
    virtual ~Combatant() = default;

    // код вида сущности: класс героя или тип врага
    virtual const std::string& getCode() const = 0;

    virtual const std::string& getName() const = 0;

    virtual int getHealth() const = 0;

    virtual int getMaxHealth() const = 0;

    virtual bool isAlive() const = 0;

    // удар, который будет нанесен данной сущностью
    virtual Hit rollDamage() = 0;

    // получение удара, возвращает урон, который прошел на самом деле
    virtual int receiveHit(int damage) = 0;
};

}  // namespace lab3
```

В C++ нет отдельного ключевого слова для интерфейсов: их роль играет класс, в котором все методы чисто виртуальные
(`= 0`). Виртуальный деструктор нужен, чтобы объект наследника корректно удалялся через указатель на базовый класс.

Удар представляет собой вот такую небольшую сущность:

```cpp
#pragma once

namespace lab3 {

// удар: сколько урона и был ли он критическим
struct Hit {
    int damage;
    bool critical;
};

}  // namespace lab3
```

`Hit` - простая структура (`struct`): у нее только открытые поля и нет поведения.

Давайте теперь опишем механику боя. Зафиксируем стороны, типы событий и исход:

```cpp
#pragma once

namespace lab3 {

enum class Side {
    HERO,
    ENEMY
};

}  // namespace lab3
```

```cpp
#pragma once

namespace lab3 {

enum class EventType {
    HIT,
    CRITICAL_HIT,
    DEATH
};

}  // namespace lab3
```

```cpp
#pragma once

namespace lab3 {

enum class Outcome {
    VICTORY,
    DEFEAT
};

}  // namespace lab3
```

Бой делится на раунды. Каждое событие раунда (удар, критический удар, гибель) запишем вот так - тут и пригодятся наши `enum`:

```cpp
#pragma once

#include "combat/event_type.hpp"
#include "combat/side.hpp"

namespace lab3 {

struct BattleEvent {
    int round;
    EventType type;
    Side source;
    Side target;
    int amount;
    int targetHealthAfter;
};

}  // namespace lab3
```

Чтобы в отчете было видно, кто сражался, заведем класс участника боя: в нем все, что мы выводим о герое и его противнике:

```cpp
#pragma once

#include <string>

#include "combat/combatant.hpp"

namespace lab3 {

// участник боя в том виде, в котором он попадает в отчет
struct Participant {
    std::string code;
    std::string name;
    int maxHealth;

    static Participant of(const Combatant& combatant) {
        return {combatant.getCode(), combatant.getName(), combatant.getMaxHealth()};
    }
};

}  // namespace lab3
```

Супер! Теперь можем составить общий отчет о бое, в нем будут два участника (герой и его противник), исход боя, а также информация о событиях, 
получится что-то вот такое:

```cpp
#pragma once

#include <vector>

#include "combat/battle_event.hpp"
#include "combat/outcome.hpp"
#include "combat/participant.hpp"

namespace lab3 {

// результат боя: кто сражался, чем все закончилось и что происходило по ходу дела
struct BattleReport {
    Participant hero;
    Participant enemy;
    Outcome outcome;
    std::vector<BattleEvent> events;
};

}  // namespace lab3
```

А теперь наконец-то определим механику самого боя! В целом уже ясно, что идти она будет по раундам, 
в рамках которых герой и враг будут обмениваться ударами, давайте так и реализуем:

```cpp
#pragma once

#include <vector>

#include "combat/battle_event.hpp"
#include "combat/battle_report.hpp"
#include "combat/combatant.hpp"
#include "combat/side.hpp"

namespace lab3 {

// бой идет по раундам: сначала бьет герой, потом враг, пока кто-то не погибнет
class Battle {
public:
    Battle(Combatant& hero, Combatant& enemy);

    BattleReport run();

private:
    void strike(Combatant& attacker, Side source, Combatant& defender, Side target);

    void emit(const BattleEvent& event);

    Combatant& hero_;
    Combatant& enemy_;
    std::vector<BattleEvent> events_;
    int round_ = 0;
};

}  // namespace lab3
```
```cpp
#include "combat/battle.hpp"

namespace lab3 {

Battle::Battle(Combatant& hero, Combatant& enemy) : hero_(hero), enemy_(enemy) {}

BattleReport Battle::run() {
    while (hero_.isAlive() && enemy_.isAlive()) {
        round_++;
        strike(hero_, Side::HERO, enemy_, Side::ENEMY);
        if (enemy_.isAlive()) {
            strike(enemy_, Side::ENEMY, hero_, Side::HERO);
        }
    }

    Outcome outcome = hero_.isAlive() ? Outcome::VICTORY : Outcome::DEFEAT;
    return {Participant::of(hero_), Participant::of(enemy_), outcome, events_};
}

void Battle::strike(Combatant& attacker, Side source, Combatant& defender, Side target) {
    Hit hit = attacker.rollDamage();
    int dealt = defender.receiveHit(hit.damage);
    EventType type = hit.critical ? EventType::CRITICAL_HIT : EventType::HIT;
    emit({round_, type, source, target, dealt, defender.getHealth()});

    if (!defender.isAlive()) {
        emit({round_, EventType::DEATH, source, target, 0, 0});
    }
}

void Battle::emit(const BattleEvent& event) {
    events_.push_back(event);
}

}  // namespace lab3
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

```cpp
#pragma once

#include <string>
#include <vector>

namespace lab3 {

// зачарование неизменяемо: все его поля константные и не меняются после создания,
// поэтому одно и то же зачарование можно безопасно повесить на несколько предметов
struct Enchantment {
    const std::string code;
    const std::string name;
    const int bonusDamage;
    const double bonusDefense;
    const int bonusPrice;
};

namespace enchantments {

inline const Enchantment SHARP{"sharp", "Острое", 3, 0, 20};
inline const Enchantment FIERY{"fiery", "Огненное", 6, 0, 50};
inline const Enchantment STURDY{"sturdy", "Прочное", 0, 0.05, 20};
inline const Enchantment WARDING{"warding", "Охранное", 0, 0.10, 50};

inline const std::vector<const Enchantment*> FOR_WEAPONS{&SHARP, &FIERY};
inline const std::vector<const Enchantment*> FOR_ARMOR{&STURDY, &WARDING};

}  // namespace enchantments

}  // namespace lab3
```

В базовый класс предмета заложим и контракт `clone`:

```cpp
#pragma once

#include <memory>
#include <string>
#include <vector>

#include "item/enchantment.hpp"

namespace lab3 {

class Item {
public:
    virtual ~Item() = default;

    // каждый наследник по-своему реализует копирование;
    // unique_ptr означает, что вызывающий код становится единственным владельцем новой копии
    virtual std::unique_ptr<Item> clone() const = 0;

    // какие зачарования подходят этому предмету
    virtual const std::vector<const Enchantment*>& getPossibleEnchantments() const = 0;

    // описание характеристик предмета, например "урон 12"
    virtual std::string getDescription() const = 0;

    void enchant(const Enchantment& enchantment);

    const std::string& getCode() const;

    const std::string& getName() const;

    int getPrice() const;

    const std::vector<const Enchantment*>& getEnchantments() const;

protected:
    Item(std::string code, std::string name, int basePrice);

    // конструктор копирования, его вызывают наследники в методе clone();
    // он защищенный, чтобы предмет нельзя было скопировать снаружи в обход clone()
    Item(const Item& source) = default;

    Item& operator=(const Item&) = delete;

private:
    // код шаблона, из которого получен предмет
    std::string code_;
    std::string name_;
    int basePrice_;
    // при копировании предмета вектор копируется целиком, а вот указатели в нем указывают
    // на те же самые зачарования - их копировать и не нужно, они неизменяемые
    std::vector<const Enchantment*> enchantments_;
};

}  // namespace lab3
```
```cpp
#include "item/item.hpp"

#include <algorithm>
#include <stdexcept>
#include <utility>

namespace lab3 {

Item::Item(std::string code, std::string name, int basePrice)
    : code_(std::move(code)), name_(std::move(name)), basePrice_(basePrice) {}

void Item::enchant(const Enchantment& enchantment) {
    const auto& possible = getPossibleEnchantments();
    if (std::find(possible.begin(), possible.end(), &enchantment) == possible.end()) {
        throw std::invalid_argument(
            "Зачарование '" + enchantment.name + "' не подходит для предмета '" + name_ + "'"
        );
    }

    enchantments_.push_back(&enchantment);
}

const std::string& Item::getCode() const {
    return code_;
}

const std::string& Item::getName() const {
    return name_;
}

int Item::getPrice() const {
    int price = basePrice_;
    for (const Enchantment* enchantment : enchantments_) {
        price += enchantment->bonusPrice;
    }

    return price;
}

const std::vector<const Enchantment*>& Item::getEnchantments() const {
    return enchantments_;
}

}  // namespace lab3
```

Обратите внимание на конструктор копирования. В C++ он по умолчанию копирует каждое поле: вектор зачарований
копируется целиком, и у копии получается свой список, а указатели внутри вектора указывают на те же самые зачарования -
их копировать и не нужно, они неизменяемые. Если бы вместо вектора мы хранили указатель на общий список, копия и
оригинал делили бы его (поверхностная копия), и зачарование, наложенное на копию, появилось бы и на шаблоне в каталоге.

Конструктор копирования защищенный (`protected`): вызвать его могут только сами классы предметов. Поэтому снаружи
скопировать предмет можно только через `clone()`, и копия всегда получается нужного класса.

Супер, общий контракт предметов понятен, давайте создадим контракт для оружия:

```cpp
#pragma once

#include <string>
#include <vector>

#include "combat/hit.hpp"
#include "item/item.hpp"

namespace lab3 {

class Weapon : public Item {
public:
    // как именно оружие наносит урон, решают наследники
    virtual Hit rollDamage() const = 0;

    // урон с учетом зачарований
    int getDamage() const;

    const std::vector<const Enchantment*>& getPossibleEnchantments() const override;

protected:
    Weapon(std::string code, std::string name, int basePrice, int damage);

    Weapon(const Weapon& source) = default;

private:
    int damage_;
};

}  // namespace lab3
```
```cpp
#include "item/weapon.hpp"

#include <utility>

namespace lab3 {

Weapon::Weapon(std::string code, std::string name, int basePrice, int damage)
    : Item(std::move(code), std::move(name), basePrice), damage_(damage) {}

int Weapon::getDamage() const {
    int total = damage_;
    for (const Enchantment* enchantment : getEnchantments()) {
        total += enchantment->bonusDamage;
    }

    return total;
}

const std::vector<const Enchantment*>& Weapon::getPossibleEnchantments() const {
    return enchantments::FOR_WEAPONS;
}

}  // namespace lab3
```

Отлично, теперь наше первое оружие, меч. Его урон всегда одинаковый:

```cpp
#pragma once

#include <memory>
#include <string>

#include "item/weapon.hpp"

namespace lab3 {

class Sword : public Weapon {
public:
    Sword(std::string code, std::string name, int basePrice, int damage);

    Hit rollDamage() const override;

    std::unique_ptr<Item> clone() const override;

    std::string getDescription() const override;
};

}  // namespace lab3
```
```cpp
#include "item/sword.hpp"

#include <utility>

namespace lab3 {

Sword::Sword(std::string code, std::string name, int basePrice, int damage)
    : Weapon(std::move(code), std::move(name), basePrice, damage) {}

Hit Sword::rollDamage() const {
    return {getDamage(), false};
}

std::unique_ptr<Item> Sword::clone() const {
    // конструктор копирования Sword копирует все поля, включая поля родителей
    return std::make_unique<Sword>(*this);
}

std::string Sword::getDescription() const {
    return "урон " + std::to_string(getDamage());
}

}  // namespace lab3
```

Лук же с некоторым шансом наносит критический, двойной урон:

```cpp
#pragma once

#include <memory>
#include <string>

#include "item/weapon.hpp"

namespace lab3 {

class Bow : public Weapon {
public:
    Bow(std::string code, std::string name, int basePrice, int damage, double criticalChance);

    Hit rollDamage() const override;

    std::unique_ptr<Item> clone() const override;

    std::string getDescription() const override;

private:
    static constexpr int CRITICAL_MULTIPLIER = 2;

    double criticalChance_;
};

}  // namespace lab3
```
```cpp
#include "item/bow.hpp"

#include <cmath>
#include <utility>

#include "core/dice.hpp"

namespace lab3 {

Bow::Bow(std::string code, std::string name, int basePrice, int damage, double criticalChance)
    : Weapon(std::move(code), std::move(name), basePrice, damage), criticalChance_(criticalChance) {}

Hit Bow::rollDamage() const {
    if (Dice::getInstance().chance(criticalChance_)) {
        return {getDamage() * CRITICAL_MULTIPLIER, true};
    }

    return {getDamage(), false};
}

std::unique_ptr<Item> Bow::clone() const {
    return std::make_unique<Bow>(*this);
}

std::string Bow::getDescription() const {
    return "урон " + std::to_string(getDamage()) +
           ", шанс крита " + std::to_string(std::lround(criticalChance_ * 100)) + "%";
}

}  // namespace lab3
```

Урон магического посоха каждый раз немного разный:

```cpp
#pragma once

#include <memory>
#include <string>

#include "item/weapon.hpp"

namespace lab3 {

// урон посоха непредсказуем: он может быть как больше, так и меньше базового
class Staff : public Weapon {
public:
    Staff(std::string code, std::string name, int basePrice, int damage, double scatter);

    Hit rollDamage() const override;

    std::unique_ptr<Item> clone() const override;

    std::string getDescription() const override;

private:
    int getMinDamage() const;

    int getMaxDamage() const;

    double scatter_;
};

}  // namespace lab3
```
```cpp
#include "item/staff.hpp"

#include <cmath>
#include <utility>

#include "core/dice.hpp"

namespace lab3 {

Staff::Staff(std::string code, std::string name, int basePrice, int damage, double scatter)
    : Weapon(std::move(code), std::move(name), basePrice, damage), scatter_(scatter) {}

Hit Staff::rollDamage() const {
    return {Dice::getInstance().roll(getMinDamage(), getMaxDamage()), false};
}

std::unique_ptr<Item> Staff::clone() const {
    return std::make_unique<Staff>(*this);
}

std::string Staff::getDescription() const {
    return "урон " + std::to_string(getMinDamage()) + "-" + std::to_string(getMaxDamage());
}

int Staff::getMinDamage() const {
    return static_cast<int>(std::lround(getDamage() * (1 - scatter_)));
}

int Staff::getMaxDamage() const {
    return static_cast<int>(std::lround(getDamage() * (1 + scatter_)));
}

}  // namespace lab3
```

Дальше давайте создадим класс для брони, он у нас будет общий для всех ее видов:

```cpp
#pragma once

#include <memory>
#include <string>
#include <vector>

#include "item/item.hpp"

namespace lab3 {

// броня поглощает долю входящего урона
class Armor : public Item {
public:
    Armor(std::string code, std::string name, int basePrice, double defense);

    std::unique_ptr<Item> clone() const override;

    // доля поглощаемого урона с учетом зачарований
    double getDefense() const;

    const std::vector<const Enchantment*>& getPossibleEnchantments() const override;

    std::string getDescription() const override;

private:
    // больше этой доли урона никакая броня не поглощает
    static constexpr double MAX_DEFENSE = 0.75;

    double defense_;
};

}  // namespace lab3
```
```cpp
#include "item/armor.hpp"

#include <algorithm>
#include <cmath>
#include <utility>

namespace lab3 {

Armor::Armor(std::string code, std::string name, int basePrice, double defense)
    : Item(std::move(code), std::move(name), basePrice), defense_(defense) {}

std::unique_ptr<Item> Armor::clone() const {
    return std::make_unique<Armor>(*this);
}

double Armor::getDefense() const {
    double total = defense_;
    for (const Enchantment* enchantment : getEnchantments()) {
        total += enchantment->bonusDefense;
    }

    return std::min(total, MAX_DEFENSE);
}

const std::vector<const Enchantment*>& Armor::getPossibleEnchantments() const {
    return enchantments::FOR_ARMOR;
}

std::string Armor::getDescription() const {
    return "защита " + std::to_string(std::lround(getDefense() * 100)) + "%";
}

}  // namespace lab3
```

Каждый наследник реализует `clone()` одной строкой: `std::make_unique<Sword>(*this)` вызывает конструктор копирования
именно `Sword`, который копирует и его поля, и поля всех родителей. Возвращается `std::unique_ptr<Item>`: тот, кто
вызвал `clone()`, становится единственным владельцем копии и сам решает, что с ней делать дальше.

Теперь давайте напишем каталог: он хранит шаблоны всех предметов игры и выдает по ним конкретные экземпляры оружия и
брони:

```cpp
#pragma once

#include <memory>
#include <string>
#include <vector>

#include "item/armor.hpp"
#include "item/item.hpp"
#include "item/weapon.hpp"

namespace lab3 {

// каталог хранит шаблоны предметов, а наружу отдает только их копии
class ItemCatalog {
public:
    // каталог становится владельцем шаблона
    void add(std::unique_ptr<Item> prototype);

    // каталог не знает, какой конкретно класс у шаблона - копию создает сам шаблон
    std::unique_ptr<Item> create(const std::string& code) const;

    std::unique_ptr<Weapon> createWeapon(const std::string& code) const;

    std::unique_ptr<Armor> createArmor(const std::string& code) const;

    std::vector<std::string> getCodes() const;

private:
    // шаблоны хранятся в порядке добавления, предметов немного, поэтому ищем простым перебором
    std::vector<std::unique_ptr<Item>> prototypes_;
};

}  // namespace lab3
```
```cpp
#include "item/item_catalog.hpp"

#include <stdexcept>
#include <utility>

#include "core/unknown_code_error.hpp"

namespace lab3 {

void ItemCatalog::add(std::unique_ptr<Item> prototype) {
    prototypes_.push_back(std::move(prototype));
}

std::unique_ptr<Item> ItemCatalog::create(const std::string& code) const {
    for (const auto& prototype : prototypes_) {
        if (prototype->getCode() == code) {
            return prototype->clone();
        }
    }

    throw UnknownCodeError("Предмет", code);
}

std::unique_ptr<Weapon> ItemCatalog::createWeapon(const std::string& code) const {
    std::unique_ptr<Item> item = create(code);
    if (auto* weapon = dynamic_cast<Weapon*>(item.get())) {
        // передаем владение копией из unique_ptr<Item> в unique_ptr<Weapon>
        item.release();
        return std::unique_ptr<Weapon>(weapon);
    }

    throw std::invalid_argument("Предмет '" + code + "' не является оружием");
}

std::unique_ptr<Armor> ItemCatalog::createArmor(const std::string& code) const {
    std::unique_ptr<Item> item = create(code);
    if (auto* armor = dynamic_cast<Armor*>(item.get())) {
        item.release();
        return std::unique_ptr<Armor>(armor);
    }

    throw std::invalid_argument("Предмет '" + code + "' не является броней");
}

std::vector<std::string> ItemCatalog::getCodes() const {
    std::vector<std::string> codes;
    for (const auto& prototype : prototypes_) {
        codes.push_back(prototype->getCode());
    }

    return codes;
}

}  // namespace lab3
```

В методе `create` и работает прототип: каталог вызывает `clone()` у шаблона и получает копию, не зная, меч это,
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

```cpp
#pragma once

#include "item/item_catalog.hpp"

namespace lab3 {

namespace item_codes {

// коды шаблонов: по ним предметы ищутся в каталоге
// стартовое снаряжение
inline constexpr const char* TRAINING_SWORD = "training-sword";
inline constexpr const char* HUNTING_BOW = "hunting-bow";
inline constexpr const char* APPRENTICE_STAFF = "apprentice-staff";
inline constexpr const char* CHAINMAIL = "chainmail";
inline constexpr const char* LEATHER_ARMOR = "leather-armor";
inline constexpr const char* LINEN_ROBE = "linen-robe";

// товары торговца
inline constexpr const char* IRON_SWORD = "iron-sword";
inline constexpr const char* ELVEN_BOW = "elven-bow";
inline constexpr const char* EMBER_STAFF = "ember-staff";
inline constexpr const char* PLATE_ARMOR = "plate-armor";

// добыча
inline constexpr const char* GOBLIN_CLEAVER = "goblin-cleaver";
inline constexpr const char* BONE_BOW = "bone-bow";
inline constexpr const char* NECROMANCER_STAFF = "necromancer-staff";
inline constexpr const char* KNIGHT_BLADE = "knight-blade";
inline constexpr const char* CRYPT_MAIL = "crypt-mail";
inline constexpr const char* DAWN_BLADE = "dawn-blade";
inline constexpr const char* DRAGONSCALE_ARMOR = "dragonscale-armor";

}  // namespace item_codes

// все предметы, которые есть в игре
ItemCatalog createDefaultCatalog();

}  // namespace lab3
```
```cpp
#include "item/default_catalog.hpp"

#include <memory>

#include "item/armor.hpp"
#include "item/bow.hpp"
#include "item/staff.hpp"
#include "item/sword.hpp"

namespace lab3 {

ItemCatalog createDefaultCatalog() {
    ItemCatalog catalog;

    // стартовое снаряжение
    catalog.add(std::make_unique<Sword>(item_codes::TRAINING_SWORD, "Тренировочный меч", 20, 12));
    catalog.add(std::make_unique<Bow>(item_codes::HUNTING_BOW, "Охотничий лук", 20, 10, 0.3));
    catalog.add(std::make_unique<Staff>(item_codes::APPRENTICE_STAFF, "Посох ученика", 20, 14, 0.3));
    catalog.add(std::make_unique<Armor>(item_codes::CHAINMAIL, "Кольчуга", 40, 0.3));
    catalog.add(std::make_unique<Armor>(item_codes::LEATHER_ARMOR, "Кожаная броня", 30, 0.2));
    catalog.add(std::make_unique<Armor>(item_codes::LINEN_ROBE, "Льняная роба", 15, 0.1));

    // товары торговца
    catalog.add(std::make_unique<Sword>(item_codes::IRON_SWORD, "Железный меч", 80, 17));
    catalog.add(std::make_unique<Bow>(item_codes::ELVEN_BOW, "Эльфийский лук", 90, 14, 0.4));
    catalog.add(std::make_unique<Staff>(item_codes::EMBER_STAFF, "Тлеющий посох", 90, 20, 0.3));
    catalog.add(std::make_unique<Armor>(item_codes::PLATE_ARMOR, "Латы", 120, 0.4));

    // добыча
    catalog.add(std::make_unique<Sword>(item_codes::GOBLIN_CLEAVER, "Гоблинский тесак", 40, 15));
    catalog.add(std::make_unique<Bow>(item_codes::BONE_BOW, "Костяной лук", 140, 17, 0.4));
    catalog.add(std::make_unique<Staff>(item_codes::NECROMANCER_STAFF, "Посох некроманта", 150, 25, 0.35));
    catalog.add(std::make_unique<Sword>(item_codes::KNIGHT_BLADE, "Клинок рыцаря", 150, 23));
    catalog.add(std::make_unique<Armor>(item_codes::CRYPT_MAIL, "Кольчуга стража склепа", 160, 0.45));
    catalog.add(std::make_unique<Sword>(item_codes::DAWN_BLADE, "Клинок рассвета", 500, 35));
    catalog.add(std::make_unique<Armor>(item_codes::DRAGONSCALE_ARMOR, "Драконья чешуя", 500, 0.55));

    return catalog;
}

}  // namespace lab3
```

## Стартовое снаряжение (Абстрактная фабрика / Abstract Factory)

По логике нашей игры мы хотим выдавать герою стартовое снаряжение в зависимости от его класса.
В целом это можно сделать простыми if'ами в коде основной программы, однако мы знаем, что в дальнейшем в игру будут
добавляться новые классы, поэтому здесь уместно будет применить шаблон *Абстрактная фабрика*.
Она позволяет создавать семейства связанных между собой объектов, не привязываясь к их конкретным классам. Семейство у
нас - это стартовое снаряжение класса.

В роли базовой фабрики выступит стартовый набор снаряжения:

```cpp
#pragma once

#include <memory>

#include "item/armor.hpp"
#include "item/weapon.hpp"

namespace lab3 {

// Абстрактная фабрика: стартовый набор снаряжения
class StarterKitFactory {
public:
    virtual ~StarterKitFactory() = default;

    // возвращает некое оружие
    virtual std::unique_ptr<Weapon> createWeapon() const = 0;

    // возвращает некую броню
    virtual std::unique_ptr<Armor> createArmor() const = 0;
};

}  // namespace lab3
```

И для каждого из классов героя мы создадим свою реализацию этого набора: снаряжение каждого класса - это и есть
семейство объектов.

В наборе для воина будут лежать меч и кольчуга:

```cpp
#pragma once

#include <memory>

#include "item/default_catalog.hpp"
#include "item/item_catalog.hpp"
#include "kit/starter_kit_factory.hpp"

namespace lab3 {

class WarriorKitFactory : public StarterKitFactory {
public:
    explicit WarriorKitFactory(const ItemCatalog& catalog) : catalog_(catalog) {}

    std::unique_ptr<Weapon> createWeapon() const override {
        return catalog_.createWeapon(item_codes::TRAINING_SWORD);
    }

    std::unique_ptr<Armor> createArmor() const override {
        return catalog_.createArmor(item_codes::CHAINMAIL);
    }

private:
    const ItemCatalog& catalog_;
};

}  // namespace lab3
```

В наборе для следопыта - лук и кожаная броня:

```cpp
#pragma once

#include <memory>

#include "item/default_catalog.hpp"
#include "item/item_catalog.hpp"
#include "kit/starter_kit_factory.hpp"

namespace lab3 {

class RangerKitFactory : public StarterKitFactory {
public:
    explicit RangerKitFactory(const ItemCatalog& catalog) : catalog_(catalog) {}

    std::unique_ptr<Weapon> createWeapon() const override {
        return catalog_.createWeapon(item_codes::HUNTING_BOW);
    }

    std::unique_ptr<Armor> createArmor() const override {
        return catalog_.createArmor(item_codes::LEATHER_ARMOR);
    }

private:
    const ItemCatalog& catalog_;
};

}  // namespace lab3
```

Ну а для мага - посох и роба:

```cpp
#pragma once

#include <memory>

#include "item/default_catalog.hpp"
#include "item/item_catalog.hpp"
#include "kit/starter_kit_factory.hpp"

namespace lab3 {

class MageKitFactory : public StarterKitFactory {
public:
    explicit MageKitFactory(const ItemCatalog& catalog) : catalog_(catalog) {}

    std::unique_ptr<Weapon> createWeapon() const override {
        return catalog_.createWeapon(item_codes::APPRENTICE_STAFF);
    }

    std::unique_ptr<Armor> createArmor() const override {
        return catalog_.createArmor(item_codes::LINEN_ROBE);
    }

private:
    const ItemCatalog& catalog_;
};

}  // namespace lab3
```

Перечисление (`enum`) в этом языке не может хранить данные, поэтому классы героя - это набор заранее созданных объектов
класса `HeroClass` с приватным конструктором. Каждый класс будет сам знать свою фабрику стартового снаряжения:

```cpp
#pragma once

#include <memory>
#include <string>
#include <vector>

#include "item/item_catalog.hpp"
#include "kit/starter_kit_factory.hpp"

namespace lab3 {

// в C++ перечисление (enum) не может хранить данные, поэтому классы героя - это
// набор заранее созданных константных объектов, создать новые снаружи нельзя
class HeroClass {
public:
    static const HeroClass WARRIOR;
    static const HeroClass RANGER;
    static const HeroClass MAGE;

    // все классы героя по порядку
    static const std::vector<const HeroClass*>& values();

    static const HeroClass& byCode(const std::string& code);

    // каждый класс сам знает свою фабрику стартового снаряжения
    std::unique_ptr<StarterKitFactory> getStarterKit(const ItemCatalog& catalog) const;

    const std::string& getCode() const;

    const std::string& getDisplayName() const;

    int getBaseHealth() const;

private:
    // указатель на функцию, которая создает фабрику стартового снаряжения
    using StarterKitCreator = std::unique_ptr<StarterKitFactory> (*)(const ItemCatalog&);

    HeroClass(std::string code, std::string displayName, int baseHealth, StarterKitCreator starterKit);

    std::string code_;
    std::string displayName_;
    int baseHealth_;
    StarterKitCreator starterKit_;
};

}  // namespace lab3
```
```cpp
#include "hero/hero_class.hpp"

#include <utility>

#include "core/unknown_code_error.hpp"
#include "kit/mage_kit_factory.hpp"
#include "kit/ranger_kit_factory.hpp"
#include "kit/warrior_kit_factory.hpp"

namespace lab3 {

// лямбда без захвата превращается в обычный указатель на функцию
const HeroClass HeroClass::WARRIOR("warrior", "Воин", 120, [](const ItemCatalog& catalog) {
    return std::unique_ptr<StarterKitFactory>(std::make_unique<WarriorKitFactory>(catalog));
});

const HeroClass HeroClass::RANGER("ranger", "Следопыт", 100, [](const ItemCatalog& catalog) {
    return std::unique_ptr<StarterKitFactory>(std::make_unique<RangerKitFactory>(catalog));
});

const HeroClass HeroClass::MAGE("mage", "Маг", 85, [](const ItemCatalog& catalog) {
    return std::unique_ptr<StarterKitFactory>(std::make_unique<MageKitFactory>(catalog));
});

HeroClass::HeroClass(std::string code, std::string displayName, int baseHealth, StarterKitCreator starterKit)
    : code_(std::move(code)), displayName_(std::move(displayName)), baseHealth_(baseHealth), starterKit_(starterKit) {}

const std::vector<const HeroClass*>& HeroClass::values() {
    static const std::vector<const HeroClass*> all{&WARRIOR, &RANGER, &MAGE};
    return all;
}

const HeroClass& HeroClass::byCode(const std::string& code) {
    for (const HeroClass* heroClass : values()) {
        if (heroClass->code_ == code) {
            return *heroClass;
        }
    }

    throw UnknownCodeError("Класс героя", code);
}

std::unique_ptr<StarterKitFactory> HeroClass::getStarterKit(const ItemCatalog& catalog) const {
    return starterKit_(catalog);
}

const std::string& HeroClass::getCode() const {
    return code_;
}

const std::string& HeroClass::getDisplayName() const {
    return displayName_;
}

int HeroClass::getBaseHealth() const {
    return baseHealth_;
}

}  // namespace lab3
```

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

```cpp
#pragma once

#include <memory>
#include <string>
#include <vector>

#include "combat/combatant.hpp"
#include "combat/hit.hpp"
#include "hero/hero_class.hpp"
#include "item/armor.hpp"
#include "item/item.hpp"
#include "item/weapon.hpp"
#include "kit/starter_kit_factory.hpp"

namespace lab3 {

class Hero : public Combatant {
public:
    // внутренний класс билдер - чтобы удобно и безопасно создавать героев
    class Builder {
    public:
        Builder& setName(std::string name);

        Builder& setHeroClass(const HeroClass& heroClass);

        Builder& setWeapon(std::shared_ptr<Weapon> weapon);

        Builder& setArmor(std::shared_ptr<Armor> armor);

        // берет оружие и броню из стартового набора
        Builder& setStarterKit(const StarterKitFactory& starterKit);

        Builder& setLevel(int level);

        Builder& setGold(int gold);

        // перед созданием героя проверяем, что все обязательные поля заполнены корректно
        Hero build() const;

    private:
        // герою нужен доступ к полям строителя
        friend class Hero;

        std::string name_;
        const HeroClass* heroClass_ = nullptr;
        std::shared_ptr<Weapon> weapon_;
        std::shared_ptr<Armor> armor_;
        // необязательные поля сразу получают значения по умолчанию
        int level_ = 1;
        int gold_ = STARTING_GOLD;
    };

    // героя можно перемещать, но не копировать: копия делила бы с оригиналом все его предметы
    Hero(const Hero&) = delete;
    Hero& operator=(const Hero&) = delete;
    Hero(Hero&&) = default;
    Hero& operator=(Hero&&) = default;

    Hit rollDamage() override;

    int receiveHit(int damage) override;

    bool isAlive() const override;

    // поведение героя между экспедициями

    // перед каждой экспедицией герой успевает отдохнуть
    void restoreHealth();

    // начисляет опыт, возвращает количество полученных уровней
    int gainExperience(int amount);

    void addGold(int amount);

    void spendGold(int amount);

    void addItem(std::shared_ptr<Item> item);

    void removeItem(const std::shared_ptr<Item>& item);

    // экипирует предмет из инвентаря, снятый предмет возвращается в инвентарь
    void equip(const std::shared_ptr<Item>& item);

    // геттеры

    const std::string& getCode() const override;

    const std::string& getName() const override;

    int getHealth() const override;

    int getMaxHealth() const override;

    const HeroClass& getHeroClass() const;

    int getLevel() const;

    int getExperience() const;

    int getExperienceToNextLevel() const;

    int getGold() const;

    const Weapon& getWeapon() const;

    const Armor& getArmor() const;

    const std::vector<std::shared_ptr<Item>>& getInventory() const;

private:
    static constexpr int MAX_NAME_LENGTH = 20;
    static constexpr int STARTING_GOLD = 50;
    static constexpr int HEALTH_PER_LEVEL = 15;
    static constexpr int DAMAGE_PER_LEVEL = 3;
    static constexpr int EXPERIENCE_PER_LEVEL = 100;

    // конструктор приватный и принимает в себя билдера, из которого и берет значения полей
    explicit Hero(const Builder& builder);

    std::string name_;
    const HeroClass* heroClass_;
    int level_;
    int experience_ = 0;
    int gold_;
    int health_;
    // предметы хранятся в shared_ptr: на один и тот же предмет могут ссылаться,
    // например, инвентарь героя и отчет об экспедиции, в котором он выпал
    std::shared_ptr<Weapon> weapon_;
    std::shared_ptr<Armor> armor_;
    std::vector<std::shared_ptr<Item>> inventory_;
};

}  // namespace lab3
```
```cpp
#include "hero/hero.hpp"

#include <algorithm>
#include <cmath>
#include <stdexcept>
#include <utility>

namespace lab3 {

namespace {

int utf8Length(const std::string& text) {
    int length = 0;
    for (unsigned char byte : text) {
        if ((byte & 0xC0) != 0x80) {
            length++;
        }
    }

    return length;
}

bool isBlank(const std::string& text) {
    return text.find_first_not_of(" \t\r\n") == std::string::npos;
}

}  // namespace

// строитель

Hero::Builder& Hero::Builder::setName(std::string name) {
    name_ = std::move(name);
    return *this;
}

Hero::Builder& Hero::Builder::setHeroClass(const HeroClass& heroClass) {
    heroClass_ = &heroClass;
    return *this;
}

Hero::Builder& Hero::Builder::setWeapon(std::shared_ptr<Weapon> weapon) {
    weapon_ = std::move(weapon);
    return *this;
}

Hero::Builder& Hero::Builder::setArmor(std::shared_ptr<Armor> armor) {
    armor_ = std::move(armor);
    return *this;
}

Hero::Builder& Hero::Builder::setStarterKit(const StarterKitFactory& starterKit) {
    weapon_ = starterKit.createWeapon();
    armor_ = starterKit.createArmor();
    return *this;
}

Hero::Builder& Hero::Builder::setLevel(int level) {
    level_ = level;
    return *this;
}

Hero::Builder& Hero::Builder::setGold(int gold) {
    gold_ = gold;
    return *this;
}

Hero Hero::Builder::build() const {
    if (isBlank(name_)) {
        throw std::invalid_argument("У героя должно быть имя");
    }
    if (utf8Length(name_) > MAX_NAME_LENGTH) {
        throw std::invalid_argument(
            "Имя героя не может быть длиннее " + std::to_string(MAX_NAME_LENGTH) + " символов"
        );
    }
    if (heroClass_ == nullptr) {
        throw std::invalid_argument("Не выбран класс героя");
    }
    if (weapon_ == nullptr || armor_ == nullptr) {
        throw std::invalid_argument("Герою нужны оружие и броня");
    }
    if (level_ < 1) {
        throw std::invalid_argument("Уровень героя не может быть меньше 1");
    }
    if (gold_ < 0) {
        throw std::invalid_argument("Золота не может быть меньше нуля");
    }

    return Hero(*this);
}

// герой

Hero::Hero(const Builder& builder)
    : name_(builder.name_),
      heroClass_(builder.heroClass_),
      level_(builder.level_),
      gold_(builder.gold_),
      health_(0),
      weapon_(builder.weapon_),
      armor_(builder.armor_) {
    health_ = getMaxHealth();
}

Hit Hero::rollDamage() {
    Hit hit = weapon_->rollDamage();
    return {hit.damage + (level_ - 1) * DAMAGE_PER_LEVEL, hit.critical};
}

int Hero::receiveHit(int damage) {
    int reducedDamage = static_cast<int>(std::lround(damage * (1 - armor_->getDefense())));
    // хотя бы единицу урона броня пропускает всегда
    int dealt = std::max(1, reducedDamage);
    health_ = std::max(0, health_ - dealt);
    return dealt;
}

bool Hero::isAlive() const {
    return health_ > 0;
}

void Hero::restoreHealth() {
    health_ = getMaxHealth();
}

int Hero::gainExperience(int amount) {
    experience_ += amount;
    int levelsGained = 0;
    while (experience_ >= getExperienceToNextLevel()) {
        experience_ -= getExperienceToNextLevel();
        level_++;
        levelsGained++;
    }

    return levelsGained;
}

void Hero::addGold(int amount) {
    gold_ += amount;
}

void Hero::spendGold(int amount) {
    if (amount > gold_) {
        throw std::runtime_error(
            "Не хватает золота: нужно " + std::to_string(amount) + ", есть " + std::to_string(gold_)
        );
    }

    gold_ -= amount;
}

void Hero::addItem(std::shared_ptr<Item> item) {
    inventory_.push_back(std::move(item));
}

void Hero::removeItem(const std::shared_ptr<Item>& item) {
    auto position = std::find(inventory_.begin(), inventory_.end(), item);
    if (position == inventory_.end()) {
        throw std::invalid_argument("В инвентаре нет предмета " + item->getName());
    }

    inventory_.erase(position);
}

void Hero::equip(const std::shared_ptr<Item>& item) {
    // dynamic_pointer_cast вернет пустой указатель, если предмет другого типа
    if (auto newWeapon = std::dynamic_pointer_cast<Weapon>(item)) {
        removeItem(item);
        inventory_.push_back(weapon_);
        weapon_ = newWeapon;
    } else if (auto newArmor = std::dynamic_pointer_cast<Armor>(item)) {
        removeItem(item);
        inventory_.push_back(armor_);
        armor_ = newArmor;
    } else {
        throw std::invalid_argument("Предмет нельзя экипировать: " + item->getName());
    }
}

const std::string& Hero::getCode() const {
    return heroClass_->getCode();
}

const std::string& Hero::getName() const {
    return name_;
}

int Hero::getHealth() const {
    return health_;
}

int Hero::getMaxHealth() const {
    return heroClass_->getBaseHealth() + (level_ - 1) * HEALTH_PER_LEVEL;
}

const HeroClass& Hero::getHeroClass() const {
    return *heroClass_;
}

int Hero::getLevel() const {
    return level_;
}

int Hero::getExperience() const {
    return experience_;
}

int Hero::getExperienceToNextLevel() const {
    return level_ * EXPERIENCE_PER_LEVEL;
}

int Hero::getGold() const {
    return gold_;
}

const Weapon& Hero::getWeapon() const {
    return *weapon_;
}

const Armor& Hero::getArmor() const {
    return *armor_;
}

const std::vector<std::shared_ptr<Item>>& Hero::getInventory() const {
    return inventory_;
}

}  // namespace lab3
```

Обратите внимание еще на две вещи. Героя нельзя копировать (копирование удалено через `= delete`): копия делила бы с
оригиналом все предметы, на которые указывают `shared_ptr`. А вот перемещать героя можно, поэтому `build()` спокойно
возвращает его по значению.

Что в реализации строителя важно:

1. Конструктор героя приватный и принимает только строителя, поэтому создать героя в обход проверок нельзя.
2. Каждый метод строителя возвращает сам строитель, поэтому вызовы можно выстраивать цепочкой.
3. Все проверки собраны в методе `build`: если обязательное поле не заполнено или заполнено неправильно, герой
   просто не создается.
4. Необязательные поля получают значения по умолчанию в самом строителе.

Без строителя понадобился бы конструктор с шестью параметрами, в котором легко перепутать, например,
уровень и золото (оба - целые числа). А по цепочке вызовов строителя сразу видно, какое значение какому полю
присваивается.

Кроме того, для необязательных параметров пришлось бы заводить несколько конструкторов: 
только все обязательные поля, обязательные + 1 необязательное, обязательные + другое необязательное, все поля. 
Это называется телескопический конструктор, и строитель избавляет от него.

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

```cpp
#pragma once

#include "combat/combatant.hpp"

namespace lab3 {

class Enemy : public Combatant {
public:
    virtual int getExperienceReward() const = 0;

    virtual int rollGoldReward() const = 0;
};

}  // namespace lab3
```

Общая логика врагов вынесена в абстрактный класс:

```cpp
#pragma once

#include <string>

#include "combat/hit.hpp"
#include "enemy/enemy.hpp"

namespace lab3 {

// общая логика для врагов: здоровье, урон в диапазоне, награда
class AbstractEnemy : public Enemy {
public:
    Hit rollDamage() override;

    int receiveHit(int damage) override;

    bool isAlive() const override;

    int rollGoldReward() const override;

    int getExperienceReward() const override;

    const std::string& getCode() const override;

    const std::string& getName() const override;

    int getHealth() const override;

    int getMaxHealth() const override;

protected:
    AbstractEnemy(
        std::string code, std::string name, int maxHealth,
        int minDamage, int maxDamage,
        int experienceReward, int goldReward
    );

    // наследники могут уменьшать входящий урон
    virtual int reduceDamage(int damage) const;

private:
    std::string code_;
    std::string name_;
    int maxHealth_;
    int minDamage_;
    int maxDamage_;
    int experienceReward_;
    int goldReward_;
    int health_;
};

}  // namespace lab3
```
```cpp
#include "enemy/abstract_enemy.hpp"

#include <algorithm>
#include <utility>

#include "core/dice.hpp"

namespace lab3 {

AbstractEnemy::AbstractEnemy(
    std::string code, std::string name, int maxHealth,
    int minDamage, int maxDamage,
    int experienceReward, int goldReward
)
    : code_(std::move(code)),
      name_(std::move(name)),
      maxHealth_(maxHealth),
      minDamage_(minDamage),
      maxDamage_(maxDamage),
      experienceReward_(experienceReward),
      goldReward_(goldReward),
      health_(maxHealth) {}

Hit AbstractEnemy::rollDamage() {
    return {Dice::getInstance().roll(minDamage_, maxDamage_), false};
}

int AbstractEnemy::receiveHit(int damage) {
    int dealt = std::max(1, reduceDamage(damage));
    health_ = std::max(0, health_ - dealt);
    return dealt;
}

int AbstractEnemy::reduceDamage(int damage) const {
    return damage;
}

bool AbstractEnemy::isAlive() const {
    return health_ > 0;
}

int AbstractEnemy::rollGoldReward() const {
    return Dice::getInstance().roll(goldReward_ / 2, goldReward_);
}

int AbstractEnemy::getExperienceReward() const {
    return experienceReward_;
}

const std::string& AbstractEnemy::getCode() const {
    return code_;
}

const std::string& AbstractEnemy::getName() const {
    return name_;
}

int AbstractEnemy::getHealth() const {
    return health_;
}

int AbstractEnemy::getMaxHealth() const {
    return maxHealth_;
}

}  // namespace lab3
```

Определим злого гоблина, самого слабого врага, живущего в лесу:

```cpp
#pragma once

#include "enemy/abstract_enemy.hpp"

namespace lab3 {

// гоблин - самый простой враг, без особенностей
class Goblin : public AbstractEnemy {
public:
    Goblin() : AbstractEnemy("goblin", "Гоблин", 50, 6, 14, 40, 30) {}
};

}  // namespace lab3
```

Теперь скелет, он уже чуть страшнее гоблина:

```cpp
#pragma once

#include <cmath>

#include "enemy/abstract_enemy.hpp"

namespace lab3 {

// скелет: кости плохо поддаются ударам, поэтому часть урона проходит мимо
class Skeleton : public AbstractEnemy {
public:
    Skeleton() : AbstractEnemy("skeleton", "Скелет", 110, 10, 24, 90, 70) {}

protected:
    int reduceDamage(int damage) const override {
        return static_cast<int>(std::lround(damage * (1 - RESISTANCE)));
    }

private:
    static constexpr double RESISTANCE = 0.3;
};

}  // namespace lab3
```

И ужасный дракон: он наносит страшный урон огнем, а крепкая чешуя гасит часть урона:

```cpp
#pragma once

#include "enemy/abstract_enemy.hpp"

namespace lab3 {

// дракон защищен чешуей и время от времени дышит огнем
class Dragon : public AbstractEnemy {
public:
    Dragon();

    Hit rollDamage() override;

protected:
    int reduceDamage(int damage) const override;

private:
    static constexpr double RESISTANCE = 0.25;
    static constexpr double FIRE_BREATH_CHANCE = 0.25;
    static constexpr int FIRE_BREATH_MULTIPLIER = 2;
};

}  // namespace lab3
```
```cpp
#include "enemy/dragon.hpp"

#include <cmath>

#include "core/dice.hpp"

namespace lab3 {

Dragon::Dragon() : AbstractEnemy("dragon", "Дракон", 260, 16, 22, 500, 400) {}

Hit Dragon::rollDamage() {
    Hit bite = AbstractEnemy::rollDamage();
    if (Dice::getInstance().chance(FIRE_BREATH_CHANCE)) {
        return {bite.damage * FIRE_BREATH_MULTIPLIER, true};
    }

    return bite;
}

int Dragon::reduceDamage(int damage) const {
    return static_cast<int>(std::lround(damage * (1 - RESISTANCE)));
}

}  // namespace lab3
```

По итогам экспедиции герой получает награды: опыт, золото, добычу и новые уровни. Запишем их так:

```cpp
#pragma once

#include <memory>

#include "item/item.hpp"

namespace lab3 {

// итоги экспедиции для героя;
// gold отрицательный, если герой проиграл и потерял золото, loot пустой, если добычи нет
struct Rewards {
    int experience;
    int gold;
    std::shared_ptr<Item> loot;
    int levelsGained;
};

}  // namespace lab3
```

Ну а общий итог экспедиции фиксируем вот так:

```cpp
#pragma once

#include <cstdint>
#include <string>

#include "combat/battle_report.hpp"
#include "world/rewards.hpp"

namespace lab3 {

// отчет об экспедиции: с каким сидом, куда, как прошел бой и что получил герой
struct ExpeditionReport {
    std::uint32_t seed;
    std::string locationCode;
    BattleReport battle;
    Rewards rewards;
};

}  // namespace lab3
```

Дальше давайте определим абстрактную локацию и общую логику экспедиции. Конкретных врагов здесь не создаем - это работа
подклассов:

```cpp
#pragma once

#include <cstdint>
#include <memory>
#include <string>
#include <vector>

#include "core/dice.hpp"
#include "enemy/enemy.hpp"
#include "hero/hero.hpp"
#include "item/item_catalog.hpp"
#include "world/expedition_report.hpp"
#include "world/rewards.hpp"

namespace lab3 {

class Location {
public:
    virtual ~Location() = default;

    ExpeditionReport explore(Hero& hero) const;

    // бизнес-логика, общая для всех локаций:
    // герой встречает врага, сражается с ним и получает награду, если победил
    ExpeditionReport explore(Hero& hero, std::uint32_t seed) const;

    const std::string& getCode() const;

    const std::string& getDisplayName() const;

    int getRecommendedLevel() const;

protected:
    Location(std::string code, std::string displayName, int recommendedLevel, const ItemCatalog& catalog);

    // создание врага делегируется подклассам для повышения гибкости
    virtual std::unique_ptr<Enemy> createEnemy() const = 0;

    // коды предметов из каталога, которые можно найти в локации
    virtual const std::vector<std::string>& getLootTable() const = 0;

private:
    static constexpr double LOOT_CHANCE = 0.6;
    static constexpr double ENCHANT_CHANCE = 0.3;

    Rewards reward(Hero& hero, const Enemy& enemy, Dice& dice) const;

    std::shared_ptr<Item> rollLoot(Dice& dice) const;

    // проигравший герой теряет половину золота
    Rewards punish(Hero& hero) const;

    std::string code_;
    std::string displayName_;
    int recommendedLevel_;
    const ItemCatalog& catalog_;
};

}  // namespace lab3
```
```cpp
#include "world/location.hpp"

#include <utility>

#include "combat/battle.hpp"

namespace lab3 {

Location::Location(std::string code, std::string displayName, int recommendedLevel, const ItemCatalog& catalog)
    : code_(std::move(code)), displayName_(std::move(displayName)), recommendedLevel_(recommendedLevel), catalog_(catalog) {}

ExpeditionReport Location::explore(Hero& hero) const {
    return explore(hero, Dice::getInstance().nextSeed());
}

ExpeditionReport Location::explore(Hero& hero, std::uint32_t seed) const {
    Dice& dice = Dice::getInstance();
    dice.reseed(seed);

    hero.restoreHealth();
    std::unique_ptr<Enemy> enemy = createEnemy();
    BattleReport battle = Battle(hero, *enemy).run();

    Rewards rewards = battle.outcome == Outcome::VICTORY
        ? reward(hero, *enemy, dice)
        : punish(hero);

    return {seed, code_, battle, rewards};
}

Rewards Location::reward(Hero& hero, const Enemy& enemy, Dice& dice) const {
    int experience = enemy.getExperienceReward();
    int gold = enemy.rollGoldReward();
    std::shared_ptr<Item> loot = rollLoot(dice);

    int levelsGained = hero.gainExperience(experience);
    hero.addGold(gold);
    if (loot != nullptr) {
        hero.addItem(loot);
    }

    return {experience, gold, loot, levelsGained};
}

std::shared_ptr<Item> Location::rollLoot(Dice& dice) const {
    if (!dice.chance(LOOT_CHANCE)) {
        return nullptr;
    }

    // добыча - это копия шаблона из каталога, ее можно менять, не боясь испортить шаблон
    std::shared_ptr<Item> loot = catalog_.create(dice.pick(getLootTable()));
    if (dice.chance(ENCHANT_CHANCE)) {
        loot->enchant(*dice.pick(loot->getPossibleEnchantments()));
    }

    return loot;
}

Rewards Location::punish(Hero& hero) const {
    int lostGold = hero.getGold() / 2;
    hero.spendGold(lostGold);
    return {0, -lostGold, nullptr, 0};
}

const std::string& Location::getCode() const {
    return code_;
}

const std::string& Location::getDisplayName() const {
    return displayName_;
}

int Location::getRecommendedLevel() const {
    return recommendedLevel_;
}

}  // namespace lab3
```

`createEnemy` возвращает `std::unique_ptr<Enemy>`: владельцем врага становится метод `explore`. Враг живет, пока идет
бой, и удаляется автоматически, когда экспедиция заканчивается.

Также обратите внимание, что добыча здесь - копия шаблона из каталога, и зачарование накладывается именно на копию.

Отлично, теперь создаем конкретные локации:

Мистический лес, кишащий злыми, но слабыми гоблинами:

```cpp
#pragma once

#include <memory>
#include <string>
#include <vector>

#include "enemy/goblin.hpp"
#include "item/default_catalog.hpp"
#include "world/location.hpp"

namespace lab3 {

class Forest : public Location {
public:
    inline static const std::string CODE = "forest";

    explicit Forest(const ItemCatalog& catalog) : Location(CODE, "Мистический лес", 1, catalog) {}

protected:
    std::unique_ptr<Enemy> createEnemy() const override {
        return std::make_unique<Goblin>();
    }

    const std::vector<std::string>& getLootTable() const override {
        static const std::vector<std::string> loot{
            item_codes::GOBLIN_CLEAVER, item_codes::HUNTING_BOW, item_codes::LEATHER_ARMOR};
        return loot;
    }
};

}  // namespace lab3
```

Заброшенный склеп, в котором бродят ожившие скелеты:

```cpp
#pragma once

#include <memory>
#include <string>
#include <vector>

#include "enemy/skeleton.hpp"
#include "item/default_catalog.hpp"
#include "world/location.hpp"

namespace lab3 {

class Crypt : public Location {
public:
    inline static const std::string CODE = "crypt";

    explicit Crypt(const ItemCatalog& catalog) : Location(CODE, "Заброшенный склеп", 3, catalog) {}

protected:
    std::unique_ptr<Enemy> createEnemy() const override {
        return std::make_unique<Skeleton>();
    }

    const std::vector<std::string>& getLootTable() const override {
        static const std::vector<std::string> loot{
            item_codes::KNIGHT_BLADE, item_codes::BONE_BOW, item_codes::NECROMANCER_STAFF, item_codes::CRYPT_MAIL};
        return loot;
    }
};

}  // namespace lab3
```

И логово дракона, прибежище жуткого чешуйчатого чудища:

```cpp
#pragma once

#include <memory>
#include <string>
#include <vector>

#include "enemy/dragon.hpp"
#include "item/default_catalog.hpp"
#include "world/location.hpp"

namespace lab3 {

class DragonLair : public Location {
public:
    inline static const std::string CODE = "dragon-lair";

    explicit DragonLair(const ItemCatalog& catalog) : Location(CODE, "Логово дракона", 5, catalog) {}

protected:
    std::unique_ptr<Enemy> createEnemy() const override {
        return std::make_unique<Dragon>();
    }

    const std::vector<std::string>& getLootTable() const override {
        static const std::vector<std::string> loot{item_codes::DAWN_BLADE, item_codes::DRAGONSCALE_ARMOR};
        return loot;
    }
};

}  // namespace lab3
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

```cpp
#pragma once

#include <memory>
#include <string>
#include <vector>

#include "world/location.hpp"

namespace lab3 {

// все локации игры
class LocationRegistry {
public:
    // реестр становится владельцем локаций
    explicit LocationRegistry(std::vector<std::unique_ptr<Location>> locations);

    const Location& get(const std::string& code) const;

    std::vector<const Location*> getAll() const;

private:
    std::vector<std::unique_ptr<Location>> locations_;
};

}  // namespace lab3
```
```cpp
#include "world/location_registry.hpp"

#include <utility>

#include "core/unknown_code_error.hpp"

namespace lab3 {

LocationRegistry::LocationRegistry(std::vector<std::unique_ptr<Location>> locations)
    : locations_(std::move(locations)) {}

const Location& LocationRegistry::get(const std::string& code) const {
    for (const auto& location : locations_) {
        if (location->getCode() == code) {
            return *location;
        }
    }

    throw UnknownCodeError("Локация", code);
}

std::vector<const Location*> LocationRegistry::getAll() const {
    std::vector<const Location*> all;
    for (const auto& location : locations_) {
        all.push_back(location.get());
    }

    return all;
}

}  // namespace lab3
```

## Лагерь

Между экспедициями герой отдыхает в лагере, где есть торговец (а куда же без него). 
Торговец продает предметы из каталога и скупает ненужное за половину цены:

```cpp
#pragma once

#include <memory>
#include <string>
#include <vector>

#include "hero/hero.hpp"
#include "item/item.hpp"
#include "item/item_catalog.hpp"

namespace lab3 {

class Merchant {
public:
    Merchant(const ItemCatalog& catalog, std::vector<std::string> assortment);

    // образцы товаров, чтобы показать их покупателю
    std::vector<std::unique_ptr<Item>> getShowcase() const;

    std::shared_ptr<Item> buy(Hero& hero, const std::string& code) const;

    int sell(Hero& hero, const std::shared_ptr<Item>& item) const;

    int getSellPrice(const Item& item) const;

private:
    static constexpr int SELL_PRICE_PERCENT = 50;

    const ItemCatalog& catalog_;
    std::vector<std::string> assortment_;
};

}  // namespace lab3
```
```cpp
#include "camp/merchant.hpp"

#include <algorithm>
#include <utility>

#include "core/unknown_code_error.hpp"

namespace lab3 {

Merchant::Merchant(const ItemCatalog& catalog, std::vector<std::string> assortment)
    : catalog_(catalog), assortment_(std::move(assortment)) {}

std::vector<std::unique_ptr<Item>> Merchant::getShowcase() const {
    std::vector<std::unique_ptr<Item>> showcase;
    for (const std::string& code : assortment_) {
        showcase.push_back(catalog_.create(code));
    }

    return showcase;
}

std::shared_ptr<Item> Merchant::buy(Hero& hero, const std::string& code) const {
    if (std::find(assortment_.begin(), assortment_.end(), code) == assortment_.end()) {
        throw UnknownCodeError("Товар", code);
    }

    // покупатель получает собственную копию предмета из каталога
    std::shared_ptr<Item> item = catalog_.create(code);
    hero.spendGold(item->getPrice());
    hero.addItem(item);
    return item;
}

int Merchant::sell(Hero& hero, const std::shared_ptr<Item>& item) const {
    int price = getSellPrice(*item);
    hero.removeItem(item);
    hero.addGold(price);
    return price;
}

int Merchant::getSellPrice(const Item& item) const {
    return item.getPrice() * SELL_PRICE_PERCENT / 100;
}

}  // namespace lab3
```

## Консоль

Вся игровая логика наконец-то готова, осталось научить программу общаться с игроком. Весь вывод соберем в одном классе - он
превращает героя, предметы и отчеты об экспедиции в текст для консоли. 

Отдельный класс нужен потому, что дальше с игрой можно будет работать не только через консоль (спойлер-спойлер), и
привязывать к ней игровую логику не хочется.

Вот он:

```cpp
#pragma once

#include <string>

#include "combat/battle_event.hpp"
#include "combat/battle_report.hpp"
#include "combat/participant.hpp"
#include "combat/side.hpp"
#include "hero/hero.hpp"
#include "item/item.hpp"
#include "world/expedition_report.hpp"
#include "world/rewards.hpp"

namespace lab3 {

// весь вывод в консоль собран здесь
class ConsoleView {
public:
    static void setupConsole();

    std::string describe(const Item& item) const;

    void printHero(const Hero& hero) const;

    void printReport(const ExpeditionReport& report, const std::string& locationName) const;

private:
    void printEvent(const BattleEvent& event, const BattleReport& battle) const;

    void printRewards(const BattleReport& battle, const Rewards& rewards) const;

    const Participant& participant(Side side, const BattleReport& battle) const;
};

}  // namespace lab3
```
```cpp
#include "console_view.hpp"

#include <iostream>

#include "combat/event_type.hpp"
#include "combat/outcome.hpp"

#ifdef _WIN32
#define NOMINMAX
#include <windows.h>
#endif

namespace lab3 {

void ConsoleView::setupConsole() {
#ifdef _WIN32
    SetConsoleOutputCP(CP_UTF8);
    SetConsoleCP(CP_UTF8);
#endif
}

std::string ConsoleView::describe(const Item& item) const {
    std::string description = item.getName();

    const auto& enchantments = item.getEnchantments();
    if (!enchantments.empty()) {
        description += " [";
        for (std::size_t i = 0; i < enchantments.size(); i++) {
            if (i > 0) {
                description += ", ";
            }
            description += enchantments[i]->name;
        }
        description += "]";
    }

    return description + " (" + item.getDescription() + ")";
}

void ConsoleView::printHero(const Hero& hero) const {
    std::cout << hero.getName() << ", " << hero.getHeroClass().getDisplayName() << " " << hero.getLevel()
              << " ур. | здоровье " << hero.getMaxHealth()
              << " | опыт " << hero.getExperience() << "/" << hero.getExperienceToNextLevel()
              << " | золото " << hero.getGold() << "\n";
    std::cout << "  Оружие: " << describe(hero.getWeapon()) << "\n";
    std::cout << "  Броня:  " << describe(hero.getArmor()) << "\n";
}

void ConsoleView::printReport(const ExpeditionReport& report, const std::string& locationName) const {
    const BattleReport& battle = report.battle;
    std::cout << "=== " << locationName << " (сид " << report.seed << ") ===\n";
    std::cout << battle.hero.name << " встречает врага: " << battle.enemy.name
              << " (" << battle.enemy.maxHealth << " здоровья)\n";

    for (const BattleEvent& event : battle.events) {
        printEvent(event, battle);
    }

    printRewards(battle, report.rewards);
}

void ConsoleView::printEvent(const BattleEvent& event, const BattleReport& battle) const {
    const Participant& source = participant(event.source, battle);
    const Participant& target = participant(event.target, battle);

    switch (event.type) {
        case EventType::HIT:
            std::cout << "[раунд " << event.round << "] " << source.name << " -> " << target.name << ": "
                      << event.amount << " урона, осталось "
                      << event.targetHealthAfter << "/" << target.maxHealth << "\n";
            break;
        case EventType::CRITICAL_HIT:
            std::cout << "[раунд " << event.round << "] " << source.name << " -> " << target.name << ": КРИТ! "
                      << event.amount << " урона, осталось "
                      << event.targetHealthAfter << "/" << target.maxHealth << "\n";
            break;
        case EventType::DEATH:
            std::cout << target.name << " повержен!\n";
            break;
    }
}

void ConsoleView::printRewards(const BattleReport& battle, const Rewards& rewards) const {
    if (battle.outcome == Outcome::DEFEAT) {
        std::cout << "Поражение... " << battle.hero.name << " едва уносит ноги и теряет "
                  << -rewards.gold << " золота\n";
        return;
    }

    std::cout << "Победа! +" << rewards.experience << " опыта, +" << rewards.gold << " золота\n";
    if (rewards.loot != nullptr) {
        std::cout << "Добыча: " << describe(*rewards.loot) << "\n";
    }
    if (rewards.levelsGained > 0) {
        std::cout << "Новый уровень!\n";
    }
}

const Participant& ConsoleView::participant(Side side, const BattleReport& battle) const {
    return side == Side::HERO ? battle.hero : battle.enemy;
}

}  // namespace lab3
```

Осталось написать игровой цикл - он будет в функции `main` (файл `main.cpp`):

```cpp
#include <cstdlib>
#include <iostream>
#include <memory>
#include <stdexcept>
#include <string>
#include <vector>

#include "camp/merchant.hpp"
#include "combat/outcome.hpp"
#include "console_view.hpp"
#include "hero/hero.hpp"
#include "hero/hero_class.hpp"
#include "item/default_catalog.hpp"
#include "item/item_catalog.hpp"
#include "world/crypt.hpp"
#include "world/dragon_lair.hpp"
#include "world/forest.hpp"
#include "world/location_registry.hpp"

using namespace lab3;

namespace {

const ConsoleView VIEW;

// читает строку без пробелов по краям; если ввод закончился, завершает программу
std::string readLine() {
    std::string line;
    if (!std::getline(std::cin, line)) {
        std::exit(0);
    }

    std::size_t begin = line.find_first_not_of(" \t\r");
    if (begin == std::string::npos) {
        return "";
    }
    std::size_t end = line.find_last_not_of(" \t\r");
    return line.substr(begin, end - begin + 1);
}

// читает номер варианта от 1 до count, возвращает индекс или -1, если ввели что-то другое
int readChoice(std::size_t count) {
    std::string line = readLine();
    try {
        int choice = std::stoi(line);
        if (choice >= 1 && static_cast<std::size_t>(choice) <= count) {
            return choice - 1;
        }
    } catch (const std::exception&) {
        // некорректный ввод обрабатываем так же, как отказ от выбора
    }

    return -1;
}

Hero createHero(const ItemCatalog& catalog) {
    std::cout << "Создайте своего героя!\n";
    while (true) {
        std::cout << "Выберите класс:\n";
        const auto& heroClasses = HeroClass::values();
        for (std::size_t i = 0; i < heroClasses.size(); i++) {
            auto kit = heroClasses[i]->getStarterKit(catalog);
            std::cout << i + 1 << " - " << heroClasses[i]->getDisplayName()
                      << " (здоровье " << heroClasses[i]->getBaseHealth() << "): "
                      << VIEW.describe(*kit->createWeapon()) << ", " << VIEW.describe(*kit->createArmor()) << "\n";
        }
        int classIndex = readChoice(heroClasses.size());
        if (classIndex < 0) {
            continue;
        }
        const HeroClass& heroClass = *heroClasses[static_cast<std::size_t>(classIndex)];

        std::cout << "Введите имя:\n";
        std::string name = readLine();

        try {
            return Hero::Builder()
                .setName(name)
                .setHeroClass(heroClass)
                .setStarterKit(*heroClass.getStarterKit(catalog))
                .build();
        } catch (const std::invalid_argument& error) {
            std::cout << "Не получилось создать героя: " << error.what() << "\n";
        }
    }
}

void goOnExpedition(Hero& hero, const LocationRegistry& locations) {
    std::vector<const Location*> all = locations.getAll();
    std::cout << "Куда отправимся?\n";
    for (std::size_t i = 0; i < all.size(); i++) {
        std::cout << i + 1 << " - " << all[i]->getDisplayName()
                  << " (рекомендуемый уровень " << all[i]->getRecommendedLevel() << ")\n";
    }
    int index = readChoice(all.size());
    if (index < 0) {
        return;
    }

    const Location& location = *all[static_cast<std::size_t>(index)];
    ExpeditionReport report = location.explore(hero);
    std::cout << "\n";
    VIEW.printReport(report, location.getDisplayName());

    if (report.battle.outcome == Outcome::VICTORY && location.getCode() == DragonLair::CODE) {
        std::cout << "\nДракон повержен! Барды еще долго будут петь о подвиге героя по имени "
                  << hero.getName() << ".\n";
    }
}

void printItems(const std::vector<std::shared_ptr<Item>>& items) {
    for (std::size_t i = 0; i < items.size(); i++) {
        std::cout << i + 1 << " - " << VIEW.describe(*items[i]) << "\n";
    }
}

void manageEquipment(Hero& hero) {
    // копируем список: после equip инвентарь героя изменится
    std::vector<std::shared_ptr<Item>> inventory = hero.getInventory();
    if (inventory.empty()) {
        std::cout << "Инвентарь пуст\n";
        return;
    }

    std::cout << "Что экипировать? (0 - назад)\n";
    printItems(inventory);
    int index = readChoice(inventory.size());
    if (index < 0) {
        return;
    }

    const auto& item = inventory[static_cast<std::size_t>(index)];
    hero.equip(item);
    std::cout << "Экипировано: " << VIEW.describe(*item) << "\n";
}

void buy(Hero& hero, const Merchant& merchant) {
    std::vector<std::unique_ptr<Item>> showcase = merchant.getShowcase();
    for (std::size_t i = 0; i < showcase.size(); i++) {
        std::cout << i + 1 << " - " << VIEW.describe(*showcase[i]) << ": " << showcase[i]->getPrice() << " золота\n";
    }
    int index = readChoice(showcase.size());
    if (index < 0) {
        return;
    }

    try {
        auto item = merchant.buy(hero, showcase[static_cast<std::size_t>(index)]->getCode());
        std::cout << "Куплено: " << VIEW.describe(*item) << "\n";
    } catch (const std::runtime_error& error) {
        std::cout << error.what() << "\n";
    }
}

void sell(Hero& hero, const Merchant& merchant) {
    std::vector<std::shared_ptr<Item>> inventory = hero.getInventory();
    if (inventory.empty()) {
        std::cout << "Продавать нечего (экипированные предметы продать нельзя)\n";
        return;
    }

    for (std::size_t i = 0; i < inventory.size(); i++) {
        std::cout << i + 1 << " - " << VIEW.describe(*inventory[i]) << ": "
                  << merchant.getSellPrice(*inventory[i]) << " золота\n";
    }
    int index = readChoice(inventory.size());
    if (index < 0) {
        return;
    }

    const auto& item = inventory[static_cast<std::size_t>(index)];
    int price = merchant.sell(hero, item);
    std::cout << "Продано: " << item->getName() << " за " << price << " золота\n";
}

void visitMerchant(Hero& hero, const Merchant& merchant) {
    std::cout << "Торговец: \"Чего желаешь?\" (золото: " << hero.getGold() << ")\n";
    std::cout << "1 - купить, 2 - продать, 0 - назад\n";
    std::string choice = readLine();
    if (choice == "1") {
        buy(hero, merchant);
    } else if (choice == "2") {
        sell(hero, merchant);
    }
}

}  // namespace

int main() {
    ConsoleView::setupConsole();

    ItemCatalog catalog = createDefaultCatalog();

    std::vector<std::unique_ptr<Location>> allLocations;
    allLocations.push_back(std::make_unique<Forest>(catalog));
    allLocations.push_back(std::make_unique<Crypt>(catalog));
    allLocations.push_back(std::make_unique<DragonLair>(catalog));
    LocationRegistry locations(std::move(allLocations));

    Merchant merchant(catalog, {item_codes::IRON_SWORD, item_codes::ELVEN_BOW, item_codes::EMBER_STAFF,
                                item_codes::CHAINMAIL, item_codes::PLATE_ARMOR});

    Hero hero = createHero(catalog);
    std::cout << hero.getName() << " разбивает лагерь на опушке мистического леса.\n";

    while (true) {
        std::cout << "\n";
        VIEW.printHero(hero);
        std::cout << "1 - отправиться в экспедицию, 2 - снаряжение, 3 - торговец, 0 - выйти из игры\n";
        std::string choice = readLine();
        if (choice == "1") {
            goOnExpedition(hero, locations);
        } else if (choice == "2") {
            manageEquipment(hero);
        } else if (choice == "3") {
            visitMerchant(hero, merchant);
        } else if (choice == "0") {
            return 0;
        } else {
            std::cout << "Такого варианта нет\n";
        }
    }
}
```

Супер! Попробуйте запустить игру и победить дракона :)

В целом по коду программы видно, что мы почти не привязываемся к конкретным реализациям классов, а везде используем
абстракции (`Location`, а не конкретные локации для экспедиций, `StarterKitFactory`, а не конкретные наборы для
получения снаряжения, `Item`, а не конкретные предметы при копировании из каталога).

Таким образом в игру легко добавлять новые локации и врагов, классы героя со снаряжением и предметы.

> **_ДОП. ЗАДАНИЕ:_** Если хотите на практике в этом убедиться - попробуйте создать новую локацию и новый класс
> героя, выдав ему новое оружие и броню (не забудьте добавить их в каталог, а для новой локации - добычу!). Много ли мест в старом коде пришлось поменять?

## Демонстрация паттернов

Чтобы показать все паттерны одним запуском, напишем отдельную программу со своей функцией `main` (файл
`demo.cpp`):

```cpp
#include <iostream>
#include <stdexcept>

#include "console_view.hpp"
#include "core/dice.hpp"
#include "hero/hero.hpp"
#include "hero/hero_class.hpp"
#include "item/default_catalog.hpp"
#include "item/enchantment.hpp"
#include "item/item_catalog.hpp"
#include "world/crypt.hpp"
#include "world/forest.hpp"

using namespace lab3;

int main() {
    ConsoleView::setupConsole();
    std::cout << std::boolalpha;

    ItemCatalog catalog = createDefaultCatalog();
    ConsoleView view;

    std::cout << "===== Одиночка =====\n";
    Dice& first = Dice::getInstance();
    Dice& second = Dice::getInstance();
    std::cout << "Dice::getInstance() возвращает один и тот же объект: " << (&first == &second) << "\n";

    std::cout << "\n===== Прототип =====\n";
    auto plainSword = catalog.create(item_codes::IRON_SWORD);
    auto fierySword = catalog.create(item_codes::IRON_SWORD);
    fierySword->enchant(enchantments::FIERY);
    std::cout << "Первая копия:        " << view.describe(*plainSword) << "\n";
    std::cout << "Вторая копия:        " << view.describe(*fierySword) << "\n";
    std::cout << "Новая копия шаблона: " << view.describe(*catalog.create(item_codes::IRON_SWORD)) << "\n";
    std::cout << "Копии - разные объекты: " << (plainSword.get() != fierySword.get()) << "\n";

    std::cout << "\n===== Абстрактная фабрика =====\n";
    for (const HeroClass* heroClass : HeroClass::values()) {
        auto kit = heroClass->getStarterKit(catalog);
        std::cout << heroClass->getDisplayName() << ": " << view.describe(*kit->createWeapon()) << ", "
                  << view.describe(*kit->createArmor()) << "\n";
    }

    std::cout << "\n===== Строитель =====\n";
    Hero hero = Hero::Builder()
        .setName("Артур")
        .setHeroClass(HeroClass::WARRIOR)
        .setStarterKit(*HeroClass::WARRIOR.getStarterKit(catalog))
        .setLevel(3)
        .build();
    view.printHero(hero);
    try {
        Hero::Builder()
            .setHeroClass(HeroClass::MAGE)
            .setStarterKit(*HeroClass::MAGE.getStarterKit(catalog))
            .build();
    } catch (const std::invalid_argument& error) {
        std::cout << "Герой без имени не создается: " << error.what() << "\n";
    }
    try {
        Hero::Builder().setName("Мерлин").build();
    } catch (const std::invalid_argument& error) {
        std::cout << "Герой без класса не создается: " << error.what() << "\n";
    }

    std::cout << "\n===== Фабричный метод =====\n";
    Forest forest(catalog);
    Crypt crypt(catalog);
    view.printReport(forest.explore(hero, 7), forest.getDisplayName());
    std::cout << "\n";
    view.printReport(crypt.explore(hero, 7), crypt.getDisplayName());


    return 0;
}
```

Вывод программы:

```
===== Одиночка =====
Dice::getInstance() возвращает один и тот же объект: true

===== Прототип =====
Первая копия:        Железный меч (урон 17)
Вторая копия:        Железный меч [Огненное] (урон 23)
Новая копия шаблона: Железный меч (урон 17)
Копии - разные объекты: true

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
[раунд 1] Гоблин -> Артур: 7 урона, осталось 143/150
[раунд 2] Артур -> Гоблин: 18 урона, осталось 14/50
[раунд 2] Гоблин -> Артур: 8 урона, осталось 135/150
[раунд 3] Артур -> Гоблин: 18 урона, осталось 0/50
Гоблин повержен!
Победа! +40 опыта, +18 золота

=== Заброшенный склеп (сид 7) ===
Артур встречает врага: Скелет (110 здоровья)
[раунд 1] Артур -> Скелет: 13 урона, осталось 97/110
[раунд 1] Скелет -> Артур: 10 урона, осталось 140/150
[раунд 2] Артур -> Скелет: 13 урона, осталось 84/110
[раунд 2] Скелет -> Артур: 13 урона, осталось 127/150
[раунд 3] Артур -> Скелет: 13 урона, осталось 71/110
[раунд 3] Скелет -> Артур: 11 урона, осталось 116/150
[раунд 4] Артур -> Скелет: 13 урона, осталось 58/110
[раунд 4] Скелет -> Артур: 9 урона, осталось 107/150
[раунд 5] Артур -> Скелет: 13 урона, осталось 45/110
[раунд 5] Скелет -> Артур: 9 урона, осталось 98/150
[раунд 6] Артур -> Скелет: 13 урона, осталось 32/110
[раунд 6] Скелет -> Артур: 12 урона, осталось 86/150
[раунд 7] Артур -> Скелет: 13 урона, осталось 19/110
[раунд 7] Скелет -> Артур: 12 урона, осталось 74/150
[раунд 8] Артур -> Скелет: 13 урона, осталось 6/110
[раунд 8] Скелет -> Артур: 15 урона, осталось 59/150
[раунд 9] Артур -> Скелет: 13 урона, осталось 0/110
Скелет повержен!
Победа! +90 опыта, +49 золота
Добыча: Костяной лук (урон 17, шанс крита 40%)
```

Числа в выводе могут отличаться от ваших: алгоритм распределений случайных чисел зависит от стандартной библиотеки.

Не забудьте закоммитить и запушить изменения!

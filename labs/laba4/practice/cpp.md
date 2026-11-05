# Лаба 4 — практика на C++

Это практическая часть [лабораторной работы 4](../laba4.md): требования к сдаче и вопросы к защите находятся в основном тексте лабы.

Понадобится компилятор с поддержкой C++20 (GCC 11+, Clang 14+ или Visual Studio 2022) и CMake 3.20+. Шаблон
репозитория: [tech-prog-labs-2026-cpp](https://github.com/tidbid-kt/tech-prog-labs-2026-cpp). Код лабы пишите в папке `labs/lab3`. Сборка из
корня репозитория: `cmake -S . -B build`, затем `cmake --build build`; игра - `build/labs/lab3/lab3`, демонстрация -
`build/labs/lab3/lab3_demo`. В Visual Studio программы лежат на уровень глубже, в папке конфигурации: например,
`build\labs\lab3\Debug\lab3.exe`.

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

**`labs/lab3/defense/damage.hpp`**

```cpp
#pragma once

namespace lab3 {
class Damage {
    int amount_;
    bool evaded_ = false;
    int absorbed_ = 0;

  public:
    explicit Damage(int amount) : amount_(amount) {}
    void reduceTo(int amount) {
        amount_ = amount;
    }
    void evade() {
        amount_ = 0;
        evaded_ = true;
    }
    void absorb(int value) {
        amount_ -= value;
        absorbed_ += value;
    }
    int getAmount() const {
        return amount_;
    }
    bool isEvaded() const {
        return evaded_;
    }
    int getAbsorbed() const {
        return absorbed_;
    }
};
} // namespace lab3
```

И теперь `Combatant.receiveHit` будет возвращать `Damage` вместо `int`. Так мы сможем реализовать отдельные механики защиты от урона.
Получится что-то такое:

**`labs/lab3/combat/combatant.hpp`**

```cpp
#pragma once
#include "defense/damage.hpp"

#include <string>

#include "combat/hit.hpp"

namespace lab3 {

// всё, что может участвовать в бою: и герой, и враги
class Combatant {
  public:
    virtual ~Combatant() = default;

    // код вида сущности: класс героя или тип врага
    virtual const std::string &getCode() const = 0;

    virtual const std::string &getName() const = 0;

    virtual int getHealth() const = 0;

    virtual int getMaxHealth() const = 0;

    virtual bool isAlive() const = 0;

    // удар, который будет нанесен данной сущностью
    virtual Hit rollDamage() = 0;

    // получение удара, возвращает урон, который прошел на самом деле
    virtual Damage receiveHit(int damage) = 0;
};

} // namespace lab3
```

Для врагов давайте обработку урона в `receiveHit` также сделаем через `Damage`, не зря же мы этот класс добавляли:

**`labs/lab3/enemy/abstract_enemy.hpp`**

```cpp
#include "defense/damage.hpp"
#pragma once

#include <string>

#include "combat/hit.hpp"
#include "enemy/enemy.hpp"

namespace lab3 {

// общая логика для врагов: здоровье, урон в диапазоне, награда
class AbstractEnemy : public Enemy {
  public:
    Hit rollDamage() override;

    Damage receiveHit(int damage) override;

    bool isAlive() const override;

    int rollGoldReward() const override;

    int getExperienceReward() const override;

    const std::string &getCode() const override;

    const std::string &getName() const override;

    int getHealth() const override;

    int getMaxHealth() const override;

  protected:
    AbstractEnemy(std::string code, std::string name, int maxHealth, int minDamage, int maxDamage,
                  int experienceReward, int goldReward);

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

} // namespace lab3
```

**`labs/lab3/enemy/abstract_enemy.cpp`**

```cpp
#include "defense/damage.hpp"
#include "enemy/abstract_enemy.hpp"

#include <algorithm>
#include <utility>

#include "core/dice.hpp"

namespace lab3 {

AbstractEnemy::AbstractEnemy(std::string code, std::string name, int maxHealth, int minDamage, int maxDamage,
                             int experienceReward, int goldReward)
    : code_(std::move(code)), name_(std::move(name)), maxHealth_(maxHealth), minDamage_(minDamage),
      maxDamage_(maxDamage), experienceReward_(experienceReward), goldReward_(goldReward),
      health_(maxHealth) {}

Hit AbstractEnemy::rollDamage() {
    return {Dice::getInstance().roll(minDamage_, maxDamage_), false};
}

Damage AbstractEnemy::receiveHit(int damage) {
    int dealt = std::max(1, reduceDamage(damage));
    health_ = std::max(0, health_ - dealt);
    return Damage(dealt);
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

const std::string &AbstractEnemy::getCode() const {
    return code_;
}

const std::string &AbstractEnemy::getName() const {
    return name_;
}

int AbstractEnemy::getHealth() const {
    return health_;
}

int AbstractEnemy::getMaxHealth() const {
    return maxHealth_;
}

} // namespace lab3
```

В `Battle` добавим обработку результата `receiveHit`, с учетом того, что он теперь возвращает `Damage`:

**`labs/lab3/combat/battle.hpp`**

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
    Battle(Combatant &hero, Combatant &enemy);

    BattleReport run();

  private:
    void strike(Combatant &attacker, Side source, Combatant &defender, Side target);

    void emit(const BattleEvent &event);

    Combatant &hero_;
    Combatant &enemy_;
    std::vector<BattleEvent> events_;
    int round_ = 0;
};

} // namespace lab3
```

**`labs/lab3/combat/battle.cpp`**

```cpp
#include "combat/battle.hpp"

namespace lab3 {

Battle::Battle(Combatant &hero, Combatant &enemy) : hero_(hero), enemy_(enemy) {}

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

void Battle::strike(Combatant &attacker, Side source, Combatant &defender, Side target) {
    Hit hit = attacker.rollDamage();
    int dealt = defender.receiveHit(hit.damage).getAmount();
    EventType type = hit.critical ? EventType::CRITICAL_HIT : EventType::HIT;
    emit({round_, type, source, target, dealt, defender.getHealth()});

    if (!defender.isAlive()) {
        emit({round_, EventType::DEATH, source, target, 0, 0});
    }
}

void Battle::emit(const BattleEvent &event) {
    events_.push_back(event);
}

} // namespace lab3
```

А еще у нас в игре появятся достижения. Выдаваться они будут с помощью паттерна "наблюдатель", но объявим мы их сразу здесь.

Представлять собой они будут идентификатор, название и описание. Как-то так:

**`labs/lab3/progress/achievement.hpp`**

```cpp
#pragma once
#include <string>
#include "core/unknown_code_error.hpp"

namespace lab3 {
struct Achievement {
    std::string code, displayName, description;
};
namespace achievements {
inline const Achievement FIRST_BLOOD{"first-blood", "Первая кровь", "одержать первую победу"};
inline const Achievement LUCKY{"lucky", "Счастливчик", "найти зачарованный лут"};
inline const Achievement VETERAN{"veteran", "Ветеран", "достичь 5 уровня"};
inline const Achievement DRAGON_SLAYER{"dragon-slayer", "Драконоборец", "победить дракона"};
inline const Achievement &byCode(const std::string &code) {
    for (auto a : {&FIRST_BLOOD, &LUCKY, &VETERAN, &DRAGON_SLAYER})
        if (a->code == code)
            return *a;
    throw UnknownCodeError("Достижение", code);
}
} // namespace achievements
} // namespace lab3
```

В `Hero` добавим хранение достижений, полученных за этого героя, а также возможность восстановления опыта, инвентаря и достижений через
строитель.

Нам это понадобится, когда будем добавлять механику сохранений!

А еще метод отдыха переименуем в `rest` вместо `restoreHealth`. В нем позже появится дополнительное поведение, поэтому стоит сразу немного
расширить контракт.

Итог выглядит вот так:

**`labs/lab3/hero/hero.hpp`**

```cpp
#pragma once
#include "progress/achievement.hpp"

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
        Builder &setName(std::string name);

        Builder &setHeroClass(const HeroClass &heroClass);

        Builder &setWeapon(std::shared_ptr<Weapon> weapon);

        Builder &setArmor(std::shared_ptr<Armor> armor);

        // берет оружие и броню из стартового набора
        Builder &setStarterKit(const StarterKitFactory &starterKit);

        Builder &setLevel(int level);

        Builder &setGold(int gold);
        Builder &setExperience(int value) {
            experience_ = value;
            return *this;
        }
        Builder &setInventory(std::vector<std::shared_ptr<Item>> items) {
            inventory_ = std::move(items);
            return *this;
        }
        Builder &setAchievements(std::vector<const Achievement *> items) {
            achievements_ = std::move(items);
            return *this;
        }

        // перед созданием героя проверяем, что все обязательные поля заполнены корректно
        Hero build() const;

      private:
        // герою нужен доступ к полям строителя
        friend class Hero;

        std::string name_;
        const HeroClass *heroClass_ = nullptr;
        std::shared_ptr<Weapon> weapon_;
        std::shared_ptr<Armor> armor_;
        // необязательные поля сразу получают значения по умолчанию
        int level_ = 1;
        int gold_ = STARTING_GOLD;
        int experience_ = 0;
        std::vector<std::shared_ptr<Item>> inventory_;
        std::vector<const Achievement *> achievements_;
    };

    // героя можно перемещать, но не копировать: копия делила бы с оригиналом все его предметы
    Hero(const Hero &) = delete;
    Hero &operator=(const Hero &) = delete;
    Hero(Hero &&) = default;
    Hero &operator=(Hero &&) = default;

    Hit rollDamage() override;

    Damage receiveHit(int damage) override;

    bool isAlive() const override;

    // поведение героя между экспедициями

    // перед каждой экспедицией герой успевает отдохнуть
    void rest();

    // начисляет опыт, возвращает количество полученных уровней
    int gainExperience(int amount);

    void addGold(int amount);

    void spendGold(int amount);

    void addItem(std::shared_ptr<Item> item);

    void removeItem(const std::shared_ptr<Item> &item);

    // экипирует предмет из инвентаря, снятый предмет возвращается в инвентарь
    void equip(const std::shared_ptr<Item> &item);

    // геттеры

    const std::string &getCode() const override;

    const std::string &getName() const override;

    int getHealth() const override;

    int getMaxHealth() const override;

    const HeroClass &getHeroClass() const;

    int getLevel() const;

    int getExperience() const;

    int getExperienceToNextLevel() const;

    int getGold() const;

    const Weapon &getWeapon() const;

    const Armor &getArmor() const;

    const std::vector<std::shared_ptr<Item>> &getInventory() const;
    const std::vector<const Achievement *> &getAchievements() const {
        return achievements_;
    }
    bool unlock(const Achievement &achievement);

  private:
    static constexpr int MAX_NAME_LENGTH = 20;
    static constexpr int STARTING_GOLD = 50;
    static constexpr int HEALTH_PER_LEVEL = 15;
    static constexpr int DAMAGE_PER_LEVEL = 3;
    static constexpr int EXPERIENCE_PER_LEVEL = 100;

    // конструктор приватный и принимает в себя билдера, из которого и берет значения полей
    explicit Hero(const Builder &builder);

    std::string name_;
    const HeroClass *heroClass_;
    int level_;
    int experience_ = 0;
    int gold_;
    int health_;
    // предметы хранятся в shared_ptr: на один и тот же предмет могут ссылаться,
    // например, инвентарь героя и отчет об экспедиции, в котором он выпал
    std::shared_ptr<Weapon> weapon_;
    std::shared_ptr<Armor> armor_;
    std::vector<std::shared_ptr<Item>> inventory_;
    std::vector<const Achievement *> achievements_;
};

} // namespace lab3
```

**`labs/lab3/hero/hero.cpp`**

```cpp
#include "hero/hero.hpp"

#include <algorithm>
#include <cmath>
#include <stdexcept>
#include <utility>

namespace lab3 {

namespace {

int utf8Length(const std::string &text) {
    int length = 0;
    for (unsigned char byte : text) {
        if ((byte & 0xC0) != 0x80) {
            length++;
        }
    }

    return length;
}

bool isBlank(const std::string &text) {
    return text.find_first_not_of(" \t\r\n") == std::string::npos;
}

} // namespace

// строитель

Hero::Builder &Hero::Builder::setName(std::string name) {
    name_ = std::move(name);
    return *this;
}

Hero::Builder &Hero::Builder::setHeroClass(const HeroClass &heroClass) {
    heroClass_ = &heroClass;
    return *this;
}

Hero::Builder &Hero::Builder::setWeapon(std::shared_ptr<Weapon> weapon) {
    weapon_ = std::move(weapon);
    return *this;
}

Hero::Builder &Hero::Builder::setArmor(std::shared_ptr<Armor> armor) {
    armor_ = std::move(armor);
    return *this;
}

Hero::Builder &Hero::Builder::setStarterKit(const StarterKitFactory &starterKit) {
    weapon_ = starterKit.createWeapon();
    armor_ = starterKit.createArmor();
    return *this;
}

Hero::Builder &Hero::Builder::setLevel(int level) {
    level_ = level;
    return *this;
}

Hero::Builder &Hero::Builder::setGold(int gold) {
    gold_ = gold;
    return *this;
}

Hero Hero::Builder::build() const {
    if (isBlank(name_)) {
        throw std::invalid_argument("У героя должно быть имя");
    }
    if (utf8Length(name_) > MAX_NAME_LENGTH) {
        throw std::invalid_argument("Имя героя не может быть длиннее " + std::to_string(MAX_NAME_LENGTH) +
                                    " символов");
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

    if (experience_ < 0)
        throw std::invalid_argument("Опыт не может быть отрицательным");
    return Hero(*this);
}

// герой

Hero::Hero(const Builder &builder)
    : name_(builder.name_), heroClass_(builder.heroClass_), level_(builder.level_),
      experience_(builder.experience_), gold_(builder.gold_), health_(0), weapon_(builder.weapon_),
      armor_(builder.armor_), inventory_(builder.inventory_) {
    for (auto a : builder.achievements_)
        unlock(*a);
    rest();
}

Hit Hero::rollDamage() {
    Hit hit = weapon_->rollDamage();
    return {hit.damage + (level_ - 1) * DAMAGE_PER_LEVEL, hit.critical};
}

Damage Hero::receiveHit(int damage) {
    int reducedDamage = static_cast<int>(std::lround(damage * (1 - armor_->getDefense())));
    // хотя бы единицу урона броня пропускает всегда
    int dealt = std::max(1, reducedDamage);
    health_ = std::max(0, health_ - dealt);
    return Damage(dealt);
}

bool Hero::isAlive() const {
    return health_ > 0;
}

void Hero::rest() {
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
        throw std::runtime_error("Не хватает золота: нужно " + std::to_string(amount) + ", есть " +
                                 std::to_string(gold_));
    }

    gold_ -= amount;
}

void Hero::addItem(std::shared_ptr<Item> item) {
    inventory_.push_back(std::move(item));
}

void Hero::removeItem(const std::shared_ptr<Item> &item) {
    auto position = std::find(inventory_.begin(), inventory_.end(), item);
    if (position == inventory_.end()) {
        throw std::invalid_argument("В инвентаре нет предмета " + item->getName());
    }

    inventory_.erase(position);
}

void Hero::equip(const std::shared_ptr<Item> &item) {
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

const std::string &Hero::getCode() const {
    return heroClass_->getCode();
}

const std::string &Hero::getName() const {
    return name_;
}

int Hero::getHealth() const {
    return health_;
}

int Hero::getMaxHealth() const {
    return heroClass_->getBaseHealth() + (level_ - 1) * HEALTH_PER_LEVEL;
}

const HeroClass &Hero::getHeroClass() const {
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

const Weapon &Hero::getWeapon() const {
    return *weapon_;
}

const Armor &Hero::getArmor() const {
    return *armor_;
}

const std::vector<std::shared_ptr<Item>> &Hero::getInventory() const {
    return inventory_;
}

bool Hero::unlock(const Achievement &a) {
    if (std::find(achievements_.begin(), achievements_.end(), &a) != achievements_.end())
        return false;
    achievements_.push_back(&a);
    return true;
}
} // namespace lab3
```

В `Location` заменяем вызов `restoreHealth()` на `rest()` и открываем подклассам доступ к каталогу через метод `getCatalog()`,
это нам скоро понадобится:

**`labs/lab3/world/location.hpp`**

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

    ExpeditionReport explore(Hero &hero) const;

    // бизнес-логика, общая для всех локаций:
    // герой встречает врага, сражается с ним и получает награду, если победил
    ExpeditionReport explore(Hero &hero, std::uint32_t seed) const;

    const std::string &getCode() const;

    const std::string &getDisplayName() const;

    int getRecommendedLevel() const;

  protected:
    const ItemCatalog &getCatalog() const {
        return catalog_;
    }
    Location(std::string code, std::string displayName, int recommendedLevel, const ItemCatalog &catalog);

    // создание врага делегируется подклассам для повышения гибкости
    virtual std::unique_ptr<Enemy> createEnemy() const = 0;

    // коды предметов из каталога, которые можно найти в локации
    virtual const std::vector<std::string> &getLootTable() const = 0;

  private:
    static constexpr double LOOT_CHANCE = 0.6;
    static constexpr double ENCHANT_CHANCE = 0.3;

    Rewards reward(Hero &hero, const Enemy &enemy, Dice &dice) const;

    std::shared_ptr<Item> rollLoot(Dice &dice) const;

    // проигравший герой теряет половину золота
    Rewards punish(Hero &hero) const;

    std::string code_;
    std::string displayName_;
    int recommendedLevel_;
    const ItemCatalog &catalog_;
};

} // namespace lab3
```

**`labs/lab3/world/location.cpp`**

```cpp
#include "world/location.hpp"

#include <utility>

#include "combat/battle.hpp"

namespace lab3 {

Location::Location(std::string code, std::string displayName, int recommendedLevel,
                   const ItemCatalog &catalog)
    : code_(std::move(code)), displayName_(std::move(displayName)), recommendedLevel_(recommendedLevel),
      catalog_(catalog) {}

ExpeditionReport Location::explore(Hero &hero) const {
    return explore(hero, Dice::getInstance().nextSeed());
}

ExpeditionReport Location::explore(Hero &hero, std::uint32_t seed) const {
    Dice &dice = Dice::getInstance();
    dice.reseed(seed);

    hero.rest();
    std::unique_ptr<Enemy> enemy = createEnemy();
    BattleReport battle = Battle(hero, *enemy).run();

    Rewards rewards = battle.outcome == Outcome::VICTORY ? reward(hero, *enemy, dice) : punish(hero);

    return {seed, code_, battle, rewards};
}

Rewards Location::reward(Hero &hero, const Enemy &enemy, Dice &dice) const {
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

std::shared_ptr<Item> Location::rollLoot(Dice &dice) const {
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

Rewards Location::punish(Hero &hero) const {
    int lostGold = hero.getGold() / 2;
    hero.spendGold(lostGold);
    return {0, -lostGold, nullptr, 0};
}

const std::string &Location::getCode() const {
    return code_;
}

const std::string &Location::getDisplayName() const {
    return displayName_;
}

int Location::getRecommendedLevel() const {
    return recommendedLevel_;
}

} // namespace lab3
```

Чтобы иметь возможность восстановить зачарование по идентификатору (нам это понадобится дальше), добавим в него поиск по этому самому
идентификатору:

**`labs/lab3/item/enchantment.hpp`**

```cpp
#pragma once
#include "core/unknown_code_error.hpp"

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

inline const std::vector<const Enchantment *> FOR_WEAPONS{&SHARP, &FIERY};
inline const std::vector<const Enchantment *> FOR_ARMOR{&STURDY, &WARDING};

inline const Enchantment &byCode(const std::string &code) {
    for (auto e : {&SHARP, &FIERY, &STURDY, &WARDING})
        if (e->code == code)
            return *e;
    throw UnknownCodeError("Зачарование", code);
}
} // namespace enchantments

} // namespace lab3
```

Для воспроизводимых экспедиций используем следующую реализацию `Dice`:

**`labs/lab3/core/dice.hpp`**

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

**`labs/lab3/core/dice.cpp`**

```cpp
#include "core/dice.hpp"

#include <stdexcept>

namespace lab3 {

Dice &Dice::getInstance() {
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
    if (min > max) {
        throw std::invalid_argument("min must not exceed max");
    }
    const auto range = static_cast<std::uint64_t>(static_cast<std::int64_t>(max) - min) + 1;
    const std::uint64_t values = std::uint64_t{1} << 32;
    const auto limit = values - values % range;
    std::uint64_t value;
    do {
        value = random_();
    } while (value >= limit);
    return static_cast<int>(static_cast<std::int64_t>(min) + static_cast<std::int64_t>(value % range));
}

bool Dice::chance(double probability) {
    return static_cast<double>(random_()) / 4294967296.0 < probability;
}

} // namespace lab3
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

**`labs/lab3/enemy/haunted_weapon.hpp`**

```cpp
#pragma once
#include <algorithm>
#include <memory>
#include <utility>
#include "core/dice.hpp"
#include "enemy/enemy.hpp"
#include "item/weapon.hpp"

namespace lab3 {
class HauntedWeapon : public Enemy {
    std::shared_ptr<Weapon> weapon_;
    std::string code_ = "haunted-weapon", name_;
    int maxHealth_, health_;

  public:
    explicit HauntedWeapon(std::shared_ptr<Weapon> weapon)
        : weapon_(std::move(weapon)), name_("Одержимый " + weapon_->getName()),
          maxHealth_(weapon_->getDamage() * 5), health_(maxHealth_) {}
    const std::string &getCode() const override {
        return code_;
    }
    const std::string &getName() const override {
        return name_;
    }
    int getHealth() const override {
        return health_;
    }
    int getMaxHealth() const override {
        return maxHealth_;
    }
    bool isAlive() const override {
        return health_ > 0;
    }
    Hit rollDamage() override {
        return weapon_->rollDamage();
    }
    Damage receiveHit(int amount) override {
        health_ = std::max(0, health_ - amount);
        return Damage(amount);
    }
    int getExperienceReward() const override {
        return weapon_->getPrice() / 2;
    }
    int rollGoldReward() const override {
        return Dice::getInstance().roll(weapon_->getPrice() / 4, weapon_->getPrice() / 2);
    }
};
} // namespace lab3
```

Оружие хранится в `shared_ptr`: адаптер использует уже существующий предмет, а не становится его единственным владельцем.

А теперь объявим и саму локацию — проклятый особняк. В ней будет одно из трех случайных оружий, которые будут взяты из каталога.
Выбранное оружие будет обернуто в адаптер, то есть станет одержимо злым духом, после чего безжалостно нападет на нашего героя.

Благодаря адаптеру мы можем превратить во врага любое оружие, что безусловно разнообразит визиты в это замечательное жилище:

**`labs/lab3/world/haunted_manor.hpp`**

```cpp
#pragma once
#include "world/location.hpp"
#include "enemy/haunted_weapon.hpp"
#include "item/default_catalog.hpp"

namespace lab3 {
class HauntedManor : public Location {
    const std::vector<std::string> weapons_{item_codes::IRON_SWORD, item_codes::ELVEN_BOW,
                                            item_codes::EMBER_STAFF};

  public:
    explicit HauntedManor(const ItemCatalog &catalog)
        : Location("haunted-manor", "Проклятый особняк", 2, catalog) {}

  protected:
    std::unique_ptr<Enemy> createEnemy() const override {
        return std::make_unique<HauntedWeapon>(getCatalog().createWeapon(Dice::getInstance().pick(weapons_)));
    }
    const std::vector<std::string> &getLootTable() const override {
        return weapons_;
    }
};
} // namespace lab3
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
реализации паттерна. В нем через композицию определяется, кого заворачивает декоратор (поле `wrapped_`), все методы базового декоратора
делегируют
вызовы объекту `wrapped_`.

Важно, что `wrapped_` это абстракция – `Enemy` и сам базовый декоратор тоже реализует эту абстракцию. Таким образом на месте `wrapped_`
может
быть и другой декоратор.
Но в конце цепочки обязательно будет конкретный враг.

Итак, код получается вот такой:

**`labs/lab3/enemy/enemy_decorator.hpp`**

```cpp
#pragma once
#include <memory>
#include <utility>
#include "enemy/enemy.hpp"

namespace lab3 {
class EnemyDecorator : public Enemy {
  protected:
    std::unique_ptr<Enemy> wrapped_;

  public:
    explicit EnemyDecorator(std::unique_ptr<Enemy> wrapped) : wrapped_(std::move(wrapped)) {}
    const std::string &getCode() const override {
        return wrapped_->getCode();
    }
    const std::string &getName() const override {
        return wrapped_->getName();
    }
    int getHealth() const override {
        return wrapped_->getHealth();
    }
    int getMaxHealth() const override {
        return wrapped_->getMaxHealth();
    }
    bool isAlive() const override {
        return wrapped_->isAlive();
    }
    Hit rollDamage() override {
        return wrapped_->rollDamage();
    }
    Damage receiveHit(int amount) override {
        return wrapped_->receiveHit(amount);
    }
    int getExperienceReward() const override {
        return wrapped_->getExperienceReward();
    }
    int rollGoldReward() const override {
        return wrapped_->rollGoldReward();
    }
};
} // namespace lab3
```

Декоратор владеет вложенным врагом через `unique_ptr`. При оборачивании передаем владение через `std::move`; оно не копирует самого врага.

А дальше уже реализуем конкретных декораторов, наследников базового. Они будут соответствовать нашим модификатором, начнем с легендарного:

**`labs/lab3/enemy/legendary_enemy.hpp`**

```cpp
#pragma once
#include <stdexcept>
#include "enemy/enemy_decorator.hpp"

namespace lab3 {
class LegendaryEnemy : public EnemyDecorator {
    std::string name_;

  public:
    explicit LegendaryEnemy(std::unique_ptr<Enemy> wrapped)
        : EnemyDecorator(std::move(wrapped)), name_("Легендарный " + wrapped_->getName()) {}
    const std::string &getName() const override {
        return name_;
    }
    Hit rollDamage() override {
        auto hit = wrapped_->rollDamage();
        hit.damage *= 2;
        return hit;
    }
    int getExperienceReward() const override {
        return wrapped_->getExperienceReward() * 2;
    }
    int rollGoldReward() const override {
        return wrapped_->rollGoldReward() * 2;
    }
};
} // namespace lab3
```

Тут важно заметить, что легендарный декоратор берет за основу результат вызова `wrapped_->rollDamage()`.
Именно это и позволяет комбинировать декораторов. Внутри может оказаться не только обычный враг,
но и другая обертка: тогда вызов вернет уже модифицированное ею поведение. И так по цепочке,
пока мы не дойдем до одного из наших конкретных врагов.

Супер, базовый принцип понятен, продолжим реализовывать модификаторы, дальше у нас неистовство ветра:

**`labs/lab3/enemy/windfury_enemy.hpp`**

```cpp
#pragma once
#include <stdexcept>
#include "enemy/enemy_decorator.hpp"

namespace lab3 {
class WindfuryEnemy : public EnemyDecorator {
    std::string name_;

  public:
    explicit WindfuryEnemy(std::unique_ptr<Enemy> wrapped)
        : EnemyDecorator(std::move(wrapped)), name_("Неистовый " + wrapped_->getName()) {}
    const std::string &getName() const override {
        return name_;
    }
    Hit rollDamage() override {
        auto first = wrapped_->rollDamage();
        auto second = wrapped_->rollDamage();
        return {first.damage + second.damage, first.critical || second.critical};
    }
    int getExperienceReward() const override {
        int r = wrapped_->getExperienceReward();
        return r + r * 50 / 100;
    }
    int rollGoldReward() const override {
        int r = wrapped_->rollGoldReward();
        return r + r * 50 / 100;
    }
};
} // namespace lab3
```

Два броска здесь складываются в один удар, поэтому защита применяется к их сумме.

Логика тут точно такая же! Модифицируем поведение, делегируем вызовы вложенному врагу.

Третий и пока финальный декоратор уменьшает входящий урон вдвое, затем передаёт получившийся урон дальше по цепочке декораторов:

**`labs/lab3/enemy/armored_enemy.hpp`**

```cpp
#pragma once
#include <stdexcept>
#include "enemy/enemy_decorator.hpp"

namespace lab3 {
class ArmoredEnemy : public EnemyDecorator {
    std::string name_;

  public:
    explicit ArmoredEnemy(std::unique_ptr<Enemy> wrapped)
        : EnemyDecorator(std::move(wrapped)), name_("Бронированный " + wrapped_->getName()) {}
    const std::string &getName() const override {
        return name_;
    }
    Damage receiveHit(int amount) override {
        if (amount < 0)
            throw std::invalid_argument("Урон не может быть отрицательным");
        return amount == 0 ? Damage(0) : wrapped_->receiveHit(amount - amount / 2);
    }
};
} // namespace lab3
```

Отлично, все требуемые модификаторы объявлены.
Теперь мы можем добавлять их любому врагу и в любой последовательности.
При этом декоратор реализует тот же интерфейс `Enemy`, что и обычный враг, поэтому ни бой, ни локация не отличают элитного врага
от обычного, и работают с ним точно так же. Прелести `Dependency Inversion Principle`!

Осталось прописать логику, как и когда враг становится элитным. Это часть общей логики экспедиции, поэтому добавим её в локацию:
враг, созданный фабричным методом, с некоторым шансом получает комбинацию модификаторов.

**`labs/lab3/world/location.hpp`**

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

    ExpeditionReport explore(Hero &hero) const;

    // бизнес-логика, общая для всех локаций:
    // герой встречает врага, сражается с ним и получает награду, если победил
    ExpeditionReport explore(Hero &hero, std::uint32_t seed) const;

    const std::string &getCode() const;

    const std::string &getDisplayName() const;

    int getRecommendedLevel() const;

  protected:
    const ItemCatalog &getCatalog() const {
        return catalog_;
    }
    Location(std::string code, std::string displayName, int recommendedLevel, const ItemCatalog &catalog);

    // создание врага делегируется подклассам для повышения гибкости
    virtual std::unique_ptr<Enemy> createEnemy() const = 0;

    // коды предметов из каталога, которые можно найти в локации
    virtual const std::vector<std::string> &getLootTable() const = 0;

  private:
    static constexpr double LOOT_CHANCE = 0.6;
    static constexpr double ENCHANT_CHANCE = 0.3;

    Rewards reward(Hero &hero, const Enemy &enemy, Dice &dice) const;

    std::shared_ptr<Item> rollLoot(Dice &dice) const;

    // проигравший герой теряет половину золота
    Rewards punish(Hero &hero) const;

    std::string code_;
    std::string displayName_;
    int recommendedLevel_;
    const ItemCatalog &catalog_;
};

} // namespace lab3
```

**`labs/lab3/world/location.cpp`**

```cpp
#include "enemy/legendary_enemy.hpp"
#include "enemy/windfury_enemy.hpp"
#include "enemy/armored_enemy.hpp"
#include "world/location.hpp"

#include <utility>

#include "combat/battle.hpp"

namespace lab3 {

Location::Location(std::string code, std::string displayName, int recommendedLevel,
                   const ItemCatalog &catalog)
    : code_(std::move(code)), displayName_(std::move(displayName)), recommendedLevel_(recommendedLevel),
      catalog_(catalog) {}

ExpeditionReport Location::explore(Hero &hero) const {
    return explore(hero, Dice::getInstance().nextSeed());
}

ExpeditionReport Location::explore(Hero &hero, std::uint32_t seed) const {
    Dice &dice = Dice::getInstance();
    dice.reseed(seed);

    hero.rest();
    std::unique_ptr<Enemy> enemy = createEnemy();
    if (dice.chance(0.2)) {
        bool both = dice.chance(0.25);
        if (dice.chance(0.5)) {
            enemy = std::make_unique<LegendaryEnemy>(std::move(enemy));
            if (both)
                enemy = std::make_unique<WindfuryEnemy>(std::move(enemy));
        } else {
            enemy = std::make_unique<WindfuryEnemy>(std::move(enemy));
            if (both)
                enemy = std::make_unique<LegendaryEnemy>(std::move(enemy));
        }
        if (dice.chance(0.3))
            enemy = std::make_unique<ArmoredEnemy>(std::move(enemy));
    }
    BattleReport battle = Battle(hero, *enemy).run();

    Rewards rewards = battle.outcome == Outcome::VICTORY ? reward(hero, *enemy, dice) : punish(hero);

    return {seed, code_, battle, rewards};
}

Rewards Location::reward(Hero &hero, const Enemy &enemy, Dice &dice) const {
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

std::shared_ptr<Item> Location::rollLoot(Dice &dice) const {
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

Rewards Location::punish(Hero &hero) const {
    int lostGold = hero.getGold() / 2;
    hero.spendGold(lostGold);
    return {0, -lostGold, nullptr, 0};
}

const std::string &Location::getCode() const {
    return code_;
}

const std::string &Location::getDisplayName() const {
    return displayName_;
}

int Location::getRecommendedLevel() const {
    return recommendedLevel_;
}

} // namespace lab3
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

**`labs/lab3/storage/hero_repository.hpp`**

```cpp
#pragma once
#include <memory>
#include <vector>
#include "hero/hero.hpp"

namespace lab3 {
class HeroRepository {
  public:
    virtual ~HeroRepository() = default;
    virtual std::shared_ptr<Hero> findByName(const std::string &name) = 0;
    virtual std::vector<std::shared_ptr<Hero>> findAll() = 0;
    virtual void save(const std::shared_ptr<Hero> &hero) = 0;
};
} // namespace lab3
```

Репозитории возвращают `shared_ptr<Hero>`, поэтому кэш может возвращать тот же объект героя, а не его копию.

Для хранения сейвов будем использовать простой файл, это не бд, но подход тоже вполне рабочий. В качестве формата сохранения выберем
json! [JSON](https://www.json.org/json-ru.html) — текстовый формат, где объекты состоят из пар "имя — значение",
а массивы хранят списки значений. Это довольно вездесущий формат, и нам он как нельзя кстати подходит.

Для чтения и записи JSON подключим [nlohmann/json](https://json.nlohmann.me/integration/cmake/).
Зависимость зададим в `labs/lab3/CMakeLists.txt`. Полное содержимое файла:

**`labs/lab3/CMakeLists.txt`**

```cmake
if(POLICY CMP0135)
    cmake_policy(SET CMP0135 NEW)
endif()

include(FetchContent)
FetchContent_Declare(nlohmann_json
    URL https://github.com/nlohmann/json/releases/download/v3.11.3/json.tar.xz
    URL_HASH SHA256=d6c65aca6b1ed68e7a182f4757257b107ae403032760ed6ef121c9d55e81757d
)
FetchContent_MakeAvailable(nlohmann_json)

file(GLOB_RECURSE LAB3_SOURCES CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/*.cpp")
list(REMOVE_ITEM LAB3_SOURCES
    "${CMAKE_CURRENT_SOURCE_DIR}/main.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/demo.cpp"
)

# игра
add_executable(lab3 main.cpp ${LAB3_SOURCES})
target_include_directories(lab3 PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}")
target_link_libraries(lab3 PRIVATE nlohmann_json::nlohmann_json)

# демонстрация паттернов - отдельная программа со своей функцией main, собирается, если есть demo.cpp
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/demo.cpp")
    add_executable(lab3_demo demo.cpp ${LAB3_SOURCES})
    target_include_directories(lab3_demo PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}")
    target_link_libraries(lab3_demo PRIVATE nlohmann_json::nlohmann_json)
endif()
```

Из корня репозитория выполните `cmake -S . -B build`, затем
`cmake --build build --config Debug --target lab3 lab3_demo`. При первой настройке CMake загрузит библиотеку;
отдельно устанавливать ее в систему не нужно. Корневой `CMakeLists.txt` остается прежним.

В файле сохранения будет массив героев со всеми нужными для восстановления объектов данными.
В `FileHeroRepository` данные героя и предметов собираются в JSON-объекты, а библиотека преобразует их в текст и обратно.
Самого героя восстанавливаем через строителя, а предметы — через каталог.

Реализуем наконец сам репозиторий:

**`labs/lab3/storage/file_hero_repository.hpp`**

```cpp
#pragma once
#include <filesystem>
#include <fstream>
#include <iterator>
#include <stdexcept>
#include "nlohmann/json.hpp"
#include "storage/hero_repository.hpp"
#include "item/item_catalog.hpp"

namespace lab3 {
class FileHeroRepository : public HeroRepository {
    using Json = nlohmann::ordered_json;
    std::filesystem::path file_;
    const ItemCatalog &catalog_;
    std::string pathText() const {
        auto text = file_.u8string();
        return std::string(text.begin(), text.end());
    }
    static Json snapshotItem(const Item &item) {
        Json codes = Json::array();
        for (auto e : item.getEnchantments())
            codes.push_back(e->code);
        return {{"code", item.getCode()}, {"enchantments", codes}};
    }
    static Json snapshot(const Hero &hero) {
        Json inventory = Json::array(), achievements = Json::array();
        for (auto &item : hero.getInventory())
            inventory.push_back(snapshotItem(*item));
        for (auto a : hero.getAchievements())
            achievements.push_back(a->code);
        return {{"name", hero.getName()},
                {"heroClass", hero.getHeroClass().getCode()},
                {"level", hero.getLevel()},
                {"experience", hero.getExperience()},
                {"gold", hero.getGold()},
                {"weapon", snapshotItem(hero.getWeapon())},
                {"armor", snapshotItem(hero.getArmor())},
                {"inventory", inventory},
                {"achievements", achievements}};
    }
    std::shared_ptr<Item> restoreItem(const Json &save) const {
        std::shared_ptr<Item> item = catalog_.create(save.at("code").get<std::string>());
        for (auto &code : save.at("enchantments"))
            item->enchant(enchantments::byCode(code.get<std::string>()));
        return item;
    }
    std::shared_ptr<Hero> restore(const Json &save) const {
        std::vector<std::shared_ptr<Item>> inventory;
        for (auto &item : save.at("inventory"))
            inventory.push_back(restoreItem(item));
        std::vector<const Achievement *> achievements;
        for (auto &code : save.at("achievements"))
            achievements.push_back(&achievements::byCode(code.get<std::string>()));
        return std::make_shared<Hero>(
            Hero::Builder()
                .setName(save.at("name").get<std::string>())
                .setHeroClass(HeroClass::byCode(save.at("heroClass").get<std::string>()))
                .setLevel(save.at("level").get<int>())
                .setExperience(save.at("experience").get<int>())
                .setGold(save.at("gold").get<int>())
                .setWeapon(std::dynamic_pointer_cast<Weapon>(restoreItem(save.at("weapon"))))
                .setArmor(std::dynamic_pointer_cast<Armor>(restoreItem(save.at("armor"))))
                .setInventory(std::move(inventory))
                .setAchievements(std::move(achievements))
                .build());
    }

  public:
    FileHeroRepository(std::filesystem::path file, const ItemCatalog &catalog)
        : file_(std::move(file)), catalog_(catalog) {}
    std::vector<std::shared_ptr<Hero>> findAll() override {
        if (!std::filesystem::exists(file_))
            return {};
        std::ifstream input(file_, std::ios::binary);
        if (!input)
            throw std::runtime_error("Не удалось открыть сохранение: " + pathText());
        std::string text((std::istreambuf_iterator<char>(input)), {});
        if (input.bad())
            throw std::runtime_error("Ошибка чтения сохранения: " + pathText());
        if (text.find_first_not_of(" \t\r\n") == std::string::npos)
            return {};
        auto saves = Json::parse(text);
        if (!saves.is_array())
            throw std::runtime_error("В файле должен быть массив героев");
        std::vector<std::shared_ptr<Hero>> result;
        for (auto &save : saves)
            result.push_back(restore(save));
        return result;
    }
    std::shared_ptr<Hero> findByName(const std::string &name) override {
        for (auto &h : findAll())
            if (h->getName() == name)
                return h;
        return nullptr;
    }
    void save(const std::shared_ptr<Hero> &hero) override {
        Json saves = Json::array();
        for (auto &h : findAll())
            if (h->getName() != hero->getName())
                saves.push_back(snapshot(*h));
        saves.push_back(snapshot(*hero));
        auto parent = std::filesystem::absolute(file_).parent_path();
        std::filesystem::create_directories(parent);
        std::ofstream output(file_, std::ios::binary | std::ios::trunc);
        output << saves.dump(2) << '\n';
        output.close();
        if (!output)
            throw std::runtime_error("Не удалось записать сохранение: " + pathText());
    }
};
} // namespace lab3
```

Реализация получилась рабочая, но не очень эффективная. На каждый запрос к герою файл читается целиком, а герои собираются
заново. Мы тратим драгоценное время игры на то, чтобы открыть файл, распарсить JSON, собрать объекты, это все в целом быстро, но кажется
что можно сделать чуть эффективнее.

Давайте добавим кэширование! При первом запросе чтения загрузим героев в память, а повторные запросы чтения
будем отдавать из кэша без обращения к файлу. Мы же можем просто хранить информацию о сохраненных героях в памяти? Можем!

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

**`labs/lab3/storage/caching_hero_repository.hpp`**

```cpp
#pragma once
#include "storage/hero_repository.hpp"

namespace lab3 {
class CachingHeroRepository : public HeroRepository {
    std::unique_ptr<HeroRepository> target_;
    std::vector<std::shared_ptr<Hero>> cache_;
    bool loaded_ = false;
    void load() {
        if (!loaded_) {
            cache_ = target_->findAll();
            loaded_ = true;
        }
    }

  public:
    explicit CachingHeroRepository(std::unique_ptr<HeroRepository> target) : target_(std::move(target)) {}
    std::vector<std::shared_ptr<Hero>> findAll() override {
        load();
        return cache_;
    }
    std::shared_ptr<Hero> findByName(const std::string &name) override {
        load();
        for (auto &h : cache_)
            if (h->getName() == name)
                return h;
        return nullptr;
    }
    void save(const std::shared_ptr<Hero> &hero) override {
        target_->save(hero);
        load();
        for (auto &h : cache_)
            if (h->getName() == hero->getName()) {
                h = hero;
                return;
            }
        cache_.push_back(hero);
    }
};
} // namespace lab3
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

Можно было бы прописать все тактики прямо в `Battle` через `switch`, однако тогда каждая новая тактика требовала бы
правки этого класса. А еще нарушался бы `SRP`.

Так как в данном случае все изменения сводятся к выбору между разными алгоритми выполнения одного и того же действия
(выбора, что делать в раунде), можно применить паттерн *Стратегия*.

Он определяет семейство алгоритмов, выносит каждый
из них в отдельный класс и делает их взаимозаменяемыми: тот, кто алгоритм использует, работает с общим интерфейсом и
не знает, какая именно реализация ему досталась.

Сначала опишем, что герой вообще может сделать в раунде:

**`labs/lab3/tactic/action.hpp`**

```cpp
#pragma once

namespace lab3 {
enum class Action { ATTACK, POWER_STRIKE, DEFEND };
inline double attackMultiplier(Action action) {
    return action == Action::POWER_STRIKE ? 1.5 : action == Action::DEFEND ? 0.5 : 1.0;
}
inline double incomingMultiplier(Action action) {
    return action == Action::POWER_STRIKE ? 1.5 : action == Action::DEFEND ? 0.3 : 1.0;
}
} // namespace lab3
```

Получается, что он может вести себя агрессивно (увеличивается и исходящий, и входящий урон), сбалансированно (ничего не меняется) и
осторожно, уходя в оборону (уменьшается и входящий, и исходящий урон).

Выбирать действия мы будем в зависимости от состояния врага и героя, определим базовый интерфейс тактики:

**`labs/lab3/tactic/tactic.hpp`**

```cpp
#pragma once
#include <string>
#include "combat/combatant.hpp"
#include "tactic/action.hpp"

namespace lab3 {
class Tactic {
  public:
    virtual ~Tactic() = default;
    virtual std::string getCode() const = 0;
    virtual std::string getDisplayName() const = 0;
    virtual std::string getDescription() const = 0;
    virtual Action chooseAction(int round, const Combatant &hero, const Combatant &enemy) const = 0;
};
} // namespace lab3
```

Реализуем самый простой алгоритм выбора действия - обычный удар каждый раунд:

**`labs/lab3/tactic/balanced_tactic.hpp`**

```cpp
#pragma once
#include "tactic/tactic.hpp"

namespace lab3 {
class BalancedTactic : public Tactic {
  public:
    std::string getCode() const override {
        return "balanced";
    }
    std::string getDisplayName() const override {
        return "Баланс";
    }
    std::string getDescription() const override {
        return "обычный удар каждый раунд";
    }
    Action chooseAction(int, const Combatant &, const Combatant &) const override {
        return Action::ATTACK;
    }
};
} // namespace lab3
```

Натиск - пока здоровье врага выше, чем у героя, герой бьет безрассудно, нанося и получая в ответ больше урона:

**`labs/lab3/tactic/aggressive_tactic.hpp`**

```cpp
#pragma once
#include "tactic/tactic.hpp"

namespace lab3 {
class AggressiveTactic : public Tactic {
  public:
    std::string getCode() const override {
        return "aggressive";
    }
    std::string getDisplayName() const override {
        return "Натиск";
    }
    std::string getDescription() const override {
        return "совершать безрассудные удары, пока у врага больше здоровья, чем у героя";
    }
    Action chooseAction(int, const Combatant &hero, const Combatant &enemy) const override {
        return enemy.getHealth() > hero.getHealth() ? Action::POWER_STRIKE : Action::ATTACK;
    }
};
} // namespace lab3
```

И осторожность - раненый герой через раунд будет уходить в оборону:

**`labs/lab3/tactic/cautious_tactic.hpp`**

```cpp
#pragma once
#include "tactic/tactic.hpp"

namespace lab3 {
class CautiousTactic : public Tactic {
  public:
    std::string getCode() const override {
        return "cautious";
    }
    std::string getDisplayName() const override {
        return "Осторожность";
    }
    std::string getDescription() const override {
        return "при здоровье ниже 40% через раунд уходить в оборону";
    }
    Action chooseAction(int round, const Combatant &hero, const Combatant &) const override {
        return hero.getHealth() < hero.getMaxHealth() * 0.4 && round % 2 == 0 ? Action::DEFEND
                                                                              : Action::ATTACK;
    }
};
} // namespace lab3
```

Также дополним типы ивентов в нашей игре, добавим наши действия, и `EVADE`/`ABSORB`, они нам пригодятся дальше:

**`labs/lab3/combat/event_type.hpp`**

```cpp
#pragma once

namespace lab3 {

enum class EventType { HIT, CRITICAL_HIT, POWER_STRIKE, DEFEND, EVADE, ABSORB, DEATH };

} // namespace lab3
```

Теперь обновим сам `Battle`. Тактику он получает через конструктор и каждый раунд спрашивает у нее, что именно делает герой:

**`labs/lab3/combat/battle.hpp`**

```cpp
#pragma once
#include "tactic/tactic.hpp"

#include <vector>

#include "combat/battle_event.hpp"
#include "combat/battle_report.hpp"
#include "combat/combatant.hpp"
#include "combat/side.hpp"

namespace lab3 {

// бой идет по раундам: сначала бьет герой, потом враг, пока кто-то не погибнет
class Battle {
  public:
    Battle(Combatant &hero, Combatant &enemy, const Tactic &tactic);

    BattleReport run();

  private:
    void strike(Combatant &attacker, Side source, Combatant &defender, Side target, double multiplier);

    void emit(const BattleEvent &event);

    Combatant &hero_;
    Combatant &enemy_;
    std::vector<BattleEvent> events_;
    int round_ = 0;
    const Tactic &tactic_;
};

} // namespace lab3
```

**`labs/lab3/combat/battle.cpp`**

```cpp
#include <algorithm>
#include <cmath>
#include "combat/battle.hpp"

namespace lab3 {

Battle::Battle(Combatant &hero, Combatant &enemy, const Tactic &tactic)
    : hero_(hero), enemy_(enemy), tactic_(tactic) {}

BattleReport Battle::run() {
    while (hero_.isAlive() && enemy_.isAlive()) {
        round_++;
        Action action = tactic_.chooseAction(round_, hero_, enemy_);
        if (action == Action::POWER_STRIKE)
            emit({round_, EventType::POWER_STRIKE, Side::HERO, Side::ENEMY, 0, 0});
        else if (action == Action::DEFEND)
            emit({round_, EventType::DEFEND, Side::HERO, Side::HERO, 0, 0});
        strike(hero_, Side::HERO, enemy_, Side::ENEMY, attackMultiplier(action));
        if (enemy_.isAlive()) {
            strike(enemy_, Side::ENEMY, hero_, Side::HERO, incomingMultiplier(action));
        }
    }

    Outcome outcome = hero_.isAlive() ? Outcome::VICTORY : Outcome::DEFEAT;
    return {Participant::of(hero_), Participant::of(enemy_), outcome, events_};
}

void Battle::strike(Combatant &attacker, Side source, Combatant &defender, Side target, double multiplier) {
    Hit hit = attacker.rollDamage();
    Damage damage = defender.receiveHit(std::max(1, static_cast<int>(std::lround(hit.damage * multiplier))));
    if (damage.isEvaded()) {
        emit({round_, EventType::EVADE, source, target, 0, defender.getHealth()});
        return;
    }
    if (damage.getAbsorbed() > 0)
        emit({round_, EventType::ABSORB, source, target, damage.getAbsorbed(), defender.getHealth()});
    if (damage.getAmount() > 0)
        emit({round_, hit.critical ? EventType::CRITICAL_HIT : EventType::HIT, source, target,
              damage.getAmount(), defender.getHealth()});

    if (!defender.isAlive()) {
        emit({round_, EventType::DEATH, source, target, 0, 0});
    }
}

void Battle::emit(const BattleEvent &event) {
    events_.push_back(event);
}

} // namespace lab3
```

Обратите внимание, что бой ничего не знает о конкретных тактиках: он работает с интерфейсом `Tactic`, и
подставить ему можно любую реализацию. Если появится новая тактика, `Battle` менять не придется.

Ну и в отчете об экспедиции будем запоминать, с какой тактикой герой ее начинал:

**`labs/lab3/world/expedition_report.hpp`**

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
    std::string tacticCode;
    BattleReport battle;
    Rewards rewards;
};

} // namespace lab3
```

Стратегия позволяет нам соблюдать `Open Closed` принцип, он дает нам возможность расширять поведение нашего класса без его изменения, путем
добавления новых реализаций стратегии. Стратегию можно выбирать как в методе, где она используется, так и в конструкторе, все зависит от
конкретной необходимости. В нашем случае стратегия выбирается в конструкторе, потому что по ходу боя она не меняется.

Применять стратегию стоит, когда одно и то же действие можно выполнить несколькими способами и способ нужно выбирать
или менять во время работы программы.

И давайте сразу обновим `Location`: она будет принимать тактику и передавать дальше в `Battle`.

Получится что-то такое:

**`labs/lab3/world/location.hpp`**

```cpp
#pragma once
#include "tactic/tactic.hpp"

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

    ExpeditionReport explore(Hero &hero) const;
    ExpeditionReport explore(Hero &hero, const Tactic &tactic) const;
    ExpeditionReport explore(Hero &hero, const Tactic &tactic, std::uint32_t seed) const;

    // бизнес-логика, общая для всех локаций:
    // герой встречает врага, сражается с ним и получает награду, если победил
    ExpeditionReport explore(Hero &hero, std::uint32_t seed) const;

    const std::string &getCode() const;

    const std::string &getDisplayName() const;

    int getRecommendedLevel() const;

  protected:
    const ItemCatalog &getCatalog() const {
        return catalog_;
    }
    Location(std::string code, std::string displayName, int recommendedLevel, const ItemCatalog &catalog);

    // создание врага делегируется подклассам для повышения гибкости
    virtual std::unique_ptr<Enemy> createEnemy() const = 0;

    // коды предметов из каталога, которые можно найти в локации
    virtual const std::vector<std::string> &getLootTable() const = 0;

  private:
    static constexpr double LOOT_CHANCE = 0.6;
    static constexpr double ENCHANT_CHANCE = 0.3;

    Rewards reward(Hero &hero, const Enemy &enemy, Dice &dice) const;

    std::shared_ptr<Item> rollLoot(Dice &dice) const;

    // проигравший герой теряет половину золота
    Rewards punish(Hero &hero) const;

    std::string code_;
    std::string displayName_;
    int recommendedLevel_;
    const ItemCatalog &catalog_;
};

} // namespace lab3
```

**`labs/lab3/world/location.cpp`**

```cpp
#include "tactic/balanced_tactic.hpp"
#include "enemy/legendary_enemy.hpp"
#include "enemy/windfury_enemy.hpp"
#include "enemy/armored_enemy.hpp"
#include "world/location.hpp"

#include <utility>

#include "combat/battle.hpp"

namespace lab3 {

Location::Location(std::string code, std::string displayName, int recommendedLevel,
                   const ItemCatalog &catalog)
    : code_(std::move(code)), displayName_(std::move(displayName)), recommendedLevel_(recommendedLevel),
      catalog_(catalog) {}

ExpeditionReport Location::explore(Hero &hero) const {
    return explore(hero, Dice::getInstance().nextSeed());
}

ExpeditionReport Location::explore(Hero &hero, std::uint32_t seed) const {
    return explore(hero, BalancedTactic{}, seed);
}
ExpeditionReport Location::explore(Hero &hero, const Tactic &tactic) const {
    return explore(hero, tactic, Dice::getInstance().nextSeed());
}
ExpeditionReport Location::explore(Hero &hero, const Tactic &tactic, std::uint32_t seed) const {
    Dice &dice = Dice::getInstance();
    dice.reseed(seed);

    hero.rest();
    std::unique_ptr<Enemy> enemy = createEnemy();
    if (dice.chance(0.2)) {
        bool both = dice.chance(0.25);
        if (dice.chance(0.5)) {
            enemy = std::make_unique<LegendaryEnemy>(std::move(enemy));
            if (both)
                enemy = std::make_unique<WindfuryEnemy>(std::move(enemy));
        } else {
            enemy = std::make_unique<WindfuryEnemy>(std::move(enemy));
            if (both)
                enemy = std::make_unique<LegendaryEnemy>(std::move(enemy));
        }
        if (dice.chance(0.3))
            enemy = std::make_unique<ArmoredEnemy>(std::move(enemy));
    }
    BattleReport battle = Battle(hero, *enemy, tactic).run();

    Rewards rewards = battle.outcome == Outcome::VICTORY ? reward(hero, *enemy, dice) : punish(hero);

    return {seed, code_, tactic.getCode(), battle, rewards};
}

Rewards Location::reward(Hero &hero, const Enemy &enemy, Dice &dice) const {
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

std::shared_ptr<Item> Location::rollLoot(Dice &dice) const {
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

Rewards Location::punish(Hero &hero) const {
    int lostGold = hero.getGold() / 2;
    hero.spendGold(lostGold);
    return {0, -lostGold, nullptr, 0};
}

const std::string &Location::getCode() const {
    return code_;
}

const std::string &Location::getDisplayName() const {
    return displayName_;
}

int Location::getRecommendedLevel() const {
    return recommendedLevel_;
}

} // namespace lab3
```

Перегрузки `explore` без параметра тактики используют `BalancedTactic`, что сохраняет прежнее поведение.

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

**`labs/lab3/event/game_event_type.hpp`**

```cpp
#pragma once

namespace lab3 {
enum class GameEventType { VICTORY, DEFEAT, LEVEL_UP };
} // namespace lab3
```

И само событие. Обратите внимание, что это просто данные: что произошло, с каким героем и в какой экспедиции:

**`labs/lab3/event/game_event.hpp`**

```cpp
#pragma once
#include "event/game_event_type.hpp"
#include "hero/hero.hpp"
#include "world/expedition_report.hpp"

namespace lab3 {
struct GameEvent {
    GameEventType type;
    Hero &hero;
    ExpeditionReport expedition;
};
} // namespace lab3
```

Интерфейс для подписчиков содержит всего один метод, который вызывается при наступлении события, и определяет контракт обработки этого
изменения:

**`labs/lab3/event/game_event_listener.hpp`**

```cpp
#pragma once
#include "event/game_event.hpp"

namespace lab3 {
class GameEventListener {
  public:
    virtual ~GameEventListener() = default;
    virtual void onEvent(const GameEvent &event) = 0;
};
} // namespace lab3
```

Именно издатель позволяет подписываться на определенный тип события. Также именно он оповещает подписчиков о
наступлении того или иного события:

**`labs/lab3/event/game_event_publisher.hpp`**

```cpp
#pragma once
#include <map>
#include <vector>
#include <memory>
#include <algorithm>
#include "event/game_event_listener.hpp"

namespace lab3 {
class GameEventPublisher {
    std::map<GameEventType, std::vector<std::shared_ptr<GameEventListener>>> listeners_;

  public:
    void subscribe(GameEventType type, std::shared_ptr<GameEventListener> listener) {
        listeners_[type].push_back(std::move(listener));
    }
    void unsubscribe(GameEventType type, const std::shared_ptr<GameEventListener> &listener) {
        std::erase(listeners_[type], listener);
    }
    void publish(const GameEvent &event) {
        auto listeners = listeners_[event.type];
        for (auto &listener : listeners)
            listener->onEvent(event);
    }
};
} // namespace lab3
```

Подписчики хранятся в `listeners_`, причем хранятся по типу ивента, то есть подписаться и реагировать они могут на отдельные типы, а не на
все.
Реакции вызываются в методе `publish`. Когда в нашей системе происходит ивент – мы вызываем издателя, а он, по списку зарегистрированных
подписчиков, вызывает их
`onEvent` методы, прокидывая туда информацию о ивенте.

Теперь реализуем двух конкретных подписчиков. Сначала того, который будет выдавать достижения:

**`labs/lab3/progress/achievement_tracker.hpp`**

```cpp
#pragma once
#include "event/game_event_listener.hpp"
#include "progress/achievement.hpp"

namespace lab3 {
class AchievementTracker : public GameEventListener {
  public:
    void onEvent(const GameEvent &e) override {
        if (e.type == GameEventType::VICTORY) {
            e.hero.unlock(achievements::FIRST_BLOOD);
            if (e.expedition.battle.enemy.code == "dragon")
                e.hero.unlock(achievements::DRAGON_SLAYER);
            auto loot = e.expedition.rewards.loot;
            if (loot && !loot->getEnchantments().empty())
                e.hero.unlock(achievements::LUCKY);
        } else if (e.type == GameEventType::LEVEL_UP && e.hero.getLevel() >= 5)
            e.hero.unlock(achievements::VETERAN);
    }
};
} // namespace lab3
```

А теперь второго, он будет записывать экспедиции в журнал:

**`labs/lab3/progress/expedition_journal.hpp`**

```cpp
#pragma once
#include <map>
#include "event/game_event_listener.hpp"

namespace lab3 {
class ExpeditionJournal : public GameEventListener {
    std::map<std::string, std::vector<ExpeditionReport>> entries_;

  public:
    void onEvent(const GameEvent &e) override {
        entries_[e.hero.getName()].push_back(e.expedition);
    }
    std::vector<ExpeditionReport> getEntries(const std::string &name) const {
        auto it = entries_.find(name);
        return it == entries_.end() ? std::vector<ExpeditionReport>{} : it->second;
    }
};
} // namespace lab3
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

Теперь займёмся защитой. Сейчас она у каждого бойца своя и зашита прямо в код: герой учитывает броню в методе `receiveHit`, а
скелет и дракон переопределяют метод `reduceDamage`. Мы же хотим добавить новые механики: следопыт должен иногда
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

**`labs/lab3/defense/damage_handler.hpp`**

```cpp
#pragma once
#include <memory>
#include <utility>
#include "defense/damage.hpp"

namespace lab3 {
class DamageHandler {
    std::unique_ptr<DamageHandler> next_;

  public:
    virtual ~DamageHandler() = default;
    DamageHandler &setNext(std::unique_ptr<DamageHandler> next) {
        next_ = std::move(next);
        return *next_;
    }
    virtual void handle(Damage &damage) = 0;

  protected:
    void passToNext(Damage &damage) {
        if (next_)
            next_->handle(damage);
    }
};
} // namespace lab3
```

Каждый обработчик владеет следующим через `unique_ptr`. При пересборке цепочки старые обработчики освобождаются автоматически.

Теперь конкретный обработчик, который будет хендлить уклонение: если оно сработало, удар по герою не прошел, и передавать урон дальше по
цепочке незачем:

**`labs/lab3/defense/evasion_handler.hpp`**

```cpp
#pragma once
#include "core/dice.hpp"
#include "defense/damage_handler.hpp"

namespace lab3 {
class EvasionHandler : public DamageHandler {
    double chance_;

  public:
    explicit EvasionHandler(double chance) : chance_(chance) {}
    void handle(Damage &damage) override {
        if (Dice::getInstance().chance(chance_)) {
            damage.evade();
            return;
        }
        passToNext(damage);
    }
};
} // namespace lab3
```

Броня гасит долю урона и передает то, что осталось, следующему обработчику. Обратите внимание, что это то же самое правило,
что раньше было зашито в герое, скелете и драконе, а теперь оно записано в одном месте:

**`labs/lab3/defense/armor_handler.hpp`**

```cpp
#pragma once
#include <algorithm>
#include <cmath>
#include "defense/damage_handler.hpp"

namespace lab3 {
class ArmorHandler : public DamageHandler {
    double defense_;

  public:
    explicit ArmorHandler(double defense) : defense_(defense) {}
    void handle(Damage &damage) override {
        damage.reduceTo(std::max(1, static_cast<int>(std::lround(damage.getAmount() * (1 - defense_)))));
        passToNext(damage);
    }
};
} // namespace lab3
```

И магический барьер, который поглощает урон, пока у него есть запас прочности:

**`labs/lab3/defense/barrier_handler.hpp`**

```cpp
#pragma once
#include <algorithm>
#include "defense/damage_handler.hpp"

namespace lab3 {
class BarrierHandler : public DamageHandler {
    int strength_;

  public:
    explicit BarrierHandler(int strength) : strength_(strength) {}
    void handle(Damage &damage) override {
        int absorbed = std::min(strength_, damage.getAmount());
        if (absorbed > 0) {
            strength_ -= absorbed;
            damage.absorb(absorbed);
        }
        passToNext(damage);
    }
};
} // namespace lab3
```

Какие звенья будут в цепочке, зависит от героя. Цепочку собирает его класс - так же, как класс сам знает свою
фабрику стартового снаряжения:

**`labs/lab3/hero/hero_class.hpp`**

```cpp
#pragma once
#include "defense/damage_handler.hpp"
#include "item/armor.hpp"

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
    static const std::vector<const HeroClass *> &values();

    static const HeroClass &byCode(const std::string &code);

    // каждый класс сам знает свою фабрику стартового снаряжения
    std::unique_ptr<StarterKitFactory> getStarterKit(const ItemCatalog &catalog) const;

    const std::string &getCode() const;

    const std::string &getDisplayName() const;

    int getBaseHealth() const;
    std::unique_ptr<DamageHandler> createDefense(const Armor &armor, int level) const;

  private:
    // указатель на функцию, которая создает фабрику стартового снаряжения
    using StarterKitCreator = std::unique_ptr<StarterKitFactory> (*)(const ItemCatalog &);

    HeroClass(std::string code, std::string displayName, int baseHealth, StarterKitCreator starterKit);

    std::string code_;
    std::string displayName_;
    int baseHealth_;
    StarterKitCreator starterKit_;
};

} // namespace lab3
```

**`labs/lab3/hero/hero_class.cpp`**

```cpp
#include "defense/armor_handler.hpp"
#include "defense/barrier_handler.hpp"
#include "defense/evasion_handler.hpp"
#include "hero/hero_class.hpp"

#include <utility>

#include "core/unknown_code_error.hpp"
#include "kit/mage_kit_factory.hpp"
#include "kit/ranger_kit_factory.hpp"
#include "kit/warrior_kit_factory.hpp"

namespace lab3 {

// лямбда без захвата превращается в обычный указатель на функцию
const HeroClass HeroClass::WARRIOR("warrior", "Воин", 120, [](const ItemCatalog &catalog) {
    return std::unique_ptr<StarterKitFactory>(std::make_unique<WarriorKitFactory>(catalog));
});

const HeroClass HeroClass::RANGER("ranger", "Следопыт", 100, [](const ItemCatalog &catalog) {
    return std::unique_ptr<StarterKitFactory>(std::make_unique<RangerKitFactory>(catalog));
});

const HeroClass HeroClass::MAGE("mage", "Маг", 85, [](const ItemCatalog &catalog) {
    return std::unique_ptr<StarterKitFactory>(std::make_unique<MageKitFactory>(catalog));
});

HeroClass::HeroClass(std::string code, std::string displayName, int baseHealth, StarterKitCreator starterKit)
    : code_(std::move(code)), displayName_(std::move(displayName)), baseHealth_(baseHealth),
      starterKit_(starterKit) {}

const std::vector<const HeroClass *> &HeroClass::values() {
    static const std::vector<const HeroClass *> all{&WARRIOR, &RANGER, &MAGE};
    return all;
}

const HeroClass &HeroClass::byCode(const std::string &code) {
    for (const HeroClass *heroClass : values()) {
        if (heroClass->code_ == code) {
            return *heroClass;
        }
    }

    throw UnknownCodeError("Класс героя", code);
}

std::unique_ptr<StarterKitFactory> HeroClass::getStarterKit(const ItemCatalog &catalog) const {
    return starterKit_(catalog);
}

const std::string &HeroClass::getCode() const {
    return code_;
}

const std::string &HeroClass::getDisplayName() const {
    return displayName_;
}

int HeroClass::getBaseHealth() const {
    return baseHealth_;
}

std::unique_ptr<DamageHandler> HeroClass::createDefense(const Armor &armor, int level) const {
    double evasion = this == &RANGER ? 0.15 : 0;
    int barrier = this == &MAGE ? 30 : 0;
    if (this == &WARRIOR && level >= 3)
        barrier = 20;
    else if (this == &RANGER && level >= 5)
        barrier = 15;
    else if (this == &MAGE && level >= 3)
        evasion = 0.10;
    auto armorHandler = std::make_unique<ArmorHandler>(armor.getDefense());
    if (barrier > 0)
        armorHandler->setNext(std::make_unique<BarrierHandler>(barrier));
    if (evasion > 0) {
        auto first = std::make_unique<EvasionHandler>(evasion);
        first->setNext(std::move(armorHandler));
        return first;
    }
    return armorHandler;
}
} // namespace lab3
```

Обратите внимание, что цепочка строится динамически. Обработчики, которых у класса нет, в цепочку просто не попадают. А еще в
цепочке важен порядок звеньев: если поставить барьер перед броней, он будет поглощать урон до брони и иссякнет быстрее.

С ростом уровня защита дополняется: воин с 3-го уровня получает барьер на 20 единиц, следопыт с 5-го — барьер на 15, а маг с 3-го —
уклонение с вероятностью 10%. У следопыта сохраняется уклонение 15%, у мага — барьер на 30 единиц. Новые способности учитываются при
следующей сборке цепочки, в частности при отдыхе перед экспедицией; урон проходит в порядке "уклонение → броня → барьер".

Теперь сам герой. Урон он больше не считает сам, а передает в первое звено своей цепочки. Перед каждой экспедицией
цепочка собирается заново, поэтому барьер мага восстанавливается, а после смены брони защита сразу учитывает новую
броню:

**`labs/lab3/hero/hero.hpp`**

```cpp
#pragma once
#include "defense/damage_handler.hpp"
#include "progress/achievement.hpp"

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
        Builder &setName(std::string name);

        Builder &setHeroClass(const HeroClass &heroClass);

        Builder &setWeapon(std::shared_ptr<Weapon> weapon);

        Builder &setArmor(std::shared_ptr<Armor> armor);

        // берет оружие и броню из стартового набора
        Builder &setStarterKit(const StarterKitFactory &starterKit);

        Builder &setLevel(int level);

        Builder &setGold(int gold);
        Builder &setExperience(int value) {
            experience_ = value;
            return *this;
        }
        Builder &setInventory(std::vector<std::shared_ptr<Item>> items) {
            inventory_ = std::move(items);
            return *this;
        }
        Builder &setAchievements(std::vector<const Achievement *> items) {
            achievements_ = std::move(items);
            return *this;
        }

        // перед созданием героя проверяем, что все обязательные поля заполнены корректно
        Hero build() const;

      private:
        // герою нужен доступ к полям строителя
        friend class Hero;

        std::string name_;
        const HeroClass *heroClass_ = nullptr;
        std::shared_ptr<Weapon> weapon_;
        std::shared_ptr<Armor> armor_;
        // необязательные поля сразу получают значения по умолчанию
        int level_ = 1;
        int gold_ = STARTING_GOLD;
        int experience_ = 0;
        std::vector<std::shared_ptr<Item>> inventory_;
        std::vector<const Achievement *> achievements_;
    };

    // героя можно перемещать, но не копировать: копия делила бы с оригиналом все его предметы
    Hero(const Hero &) = delete;
    Hero &operator=(const Hero &) = delete;
    Hero(Hero &&) = default;
    Hero &operator=(Hero &&) = default;

    Hit rollDamage() override;

    Damage receiveHit(int damage) override;

    bool isAlive() const override;

    // поведение героя между экспедициями

    // перед каждой экспедицией герой успевает отдохнуть
    void rest();

    // начисляет опыт, возвращает количество полученных уровней
    int gainExperience(int amount);

    void addGold(int amount);

    void spendGold(int amount);

    void addItem(std::shared_ptr<Item> item);

    void removeItem(const std::shared_ptr<Item> &item);

    // экипирует предмет из инвентаря, снятый предмет возвращается в инвентарь
    void equip(const std::shared_ptr<Item> &item);

    // геттеры

    const std::string &getCode() const override;

    const std::string &getName() const override;

    int getHealth() const override;

    int getMaxHealth() const override;

    const HeroClass &getHeroClass() const;

    int getLevel() const;

    int getExperience() const;

    int getExperienceToNextLevel() const;

    int getGold() const;

    const Weapon &getWeapon() const;

    const Armor &getArmor() const;

    const std::vector<std::shared_ptr<Item>> &getInventory() const;
    const std::vector<const Achievement *> &getAchievements() const {
        return achievements_;
    }
    bool unlock(const Achievement &achievement);

  private:
    static constexpr int MAX_NAME_LENGTH = 20;
    static constexpr int STARTING_GOLD = 50;
    static constexpr int HEALTH_PER_LEVEL = 15;
    static constexpr int DAMAGE_PER_LEVEL = 3;
    static constexpr int EXPERIENCE_PER_LEVEL = 100;

    // конструктор приватный и принимает в себя билдера, из которого и берет значения полей
    explicit Hero(const Builder &builder);

    std::string name_;
    const HeroClass *heroClass_;
    int level_;
    int experience_ = 0;
    int gold_;
    int health_;
    // предметы хранятся в shared_ptr: на один и тот же предмет могут ссылаться,
    // например, инвентарь героя и отчет об экспедиции, в котором он выпал
    std::shared_ptr<Weapon> weapon_;
    std::shared_ptr<Armor> armor_;
    std::vector<std::shared_ptr<Item>> inventory_;
    std::vector<const Achievement *> achievements_;
    std::unique_ptr<DamageHandler> defense_;
};

} // namespace lab3
```

**`labs/lab3/hero/hero.cpp`**

```cpp
#include "hero/hero.hpp"

#include <algorithm>
#include <cmath>
#include <stdexcept>
#include <utility>

namespace lab3 {

namespace {

int utf8Length(const std::string &text) {
    int length = 0;
    for (unsigned char byte : text) {
        if ((byte & 0xC0) != 0x80) {
            length++;
        }
    }

    return length;
}

bool isBlank(const std::string &text) {
    return text.find_first_not_of(" \t\r\n") == std::string::npos;
}

} // namespace

// строитель

Hero::Builder &Hero::Builder::setName(std::string name) {
    name_ = std::move(name);
    return *this;
}

Hero::Builder &Hero::Builder::setHeroClass(const HeroClass &heroClass) {
    heroClass_ = &heroClass;
    return *this;
}

Hero::Builder &Hero::Builder::setWeapon(std::shared_ptr<Weapon> weapon) {
    weapon_ = std::move(weapon);
    return *this;
}

Hero::Builder &Hero::Builder::setArmor(std::shared_ptr<Armor> armor) {
    armor_ = std::move(armor);
    return *this;
}

Hero::Builder &Hero::Builder::setStarterKit(const StarterKitFactory &starterKit) {
    weapon_ = starterKit.createWeapon();
    armor_ = starterKit.createArmor();
    return *this;
}

Hero::Builder &Hero::Builder::setLevel(int level) {
    level_ = level;
    return *this;
}

Hero::Builder &Hero::Builder::setGold(int gold) {
    gold_ = gold;
    return *this;
}

Hero Hero::Builder::build() const {
    if (isBlank(name_)) {
        throw std::invalid_argument("У героя должно быть имя");
    }
    if (utf8Length(name_) > MAX_NAME_LENGTH) {
        throw std::invalid_argument("Имя героя не может быть длиннее " + std::to_string(MAX_NAME_LENGTH) +
                                    " символов");
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

    if (experience_ < 0)
        throw std::invalid_argument("Опыт не может быть отрицательным");
    return Hero(*this);
}

// герой

Hero::Hero(const Builder &builder)
    : name_(builder.name_), heroClass_(builder.heroClass_), level_(builder.level_),
      experience_(builder.experience_), gold_(builder.gold_), health_(0), weapon_(builder.weapon_),
      armor_(builder.armor_), inventory_(builder.inventory_) {
    for (auto a : builder.achievements_)
        unlock(*a);
    rest();
}

Hit Hero::rollDamage() {
    Hit hit = weapon_->rollDamage();
    return {hit.damage + (level_ - 1) * DAMAGE_PER_LEVEL, hit.critical};
}

Damage Hero::receiveHit(int amount) {
    Damage damage(amount);
    defense_->handle(damage);
    health_ = std::max(0, health_ - damage.getAmount());
    return damage;
}

bool Hero::isAlive() const {
    return health_ > 0;
}

void Hero::rest() {
    health_ = getMaxHealth();
    defense_ = heroClass_->createDefense(*armor_, level_);
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
        throw std::runtime_error("Не хватает золота: нужно " + std::to_string(amount) + ", есть " +
                                 std::to_string(gold_));
    }

    gold_ -= amount;
}

void Hero::addItem(std::shared_ptr<Item> item) {
    inventory_.push_back(std::move(item));
}

void Hero::removeItem(const std::shared_ptr<Item> &item) {
    auto position = std::find(inventory_.begin(), inventory_.end(), item);
    if (position == inventory_.end()) {
        throw std::invalid_argument("В инвентаре нет предмета " + item->getName());
    }

    inventory_.erase(position);
}

void Hero::equip(const std::shared_ptr<Item> &item) {
    // dynamic_pointer_cast вернет пустой указатель, если предмет другого типа
    if (auto newWeapon = std::dynamic_pointer_cast<Weapon>(item)) {
        removeItem(item);
        inventory_.push_back(weapon_);
        weapon_ = newWeapon;
    } else if (auto newArmor = std::dynamic_pointer_cast<Armor>(item)) {
        removeItem(item);
        inventory_.push_back(armor_);
        armor_ = newArmor;
        defense_ = heroClass_->createDefense(*armor_, level_);
    } else {
        throw std::invalid_argument("Предмет нельзя экипировать: " + item->getName());
    }
}

const std::string &Hero::getCode() const {
    return heroClass_->getCode();
}

const std::string &Hero::getName() const {
    return name_;
}

int Hero::getHealth() const {
    return health_;
}

int Hero::getMaxHealth() const {
    return heroClass_->getBaseHealth() + (level_ - 1) * HEALTH_PER_LEVEL;
}

const HeroClass &Hero::getHeroClass() const {
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

const Weapon &Hero::getWeapon() const {
    return *weapon_;
}

const Armor &Hero::getArmor() const {
    return *armor_;
}

const std::vector<std::shared_ptr<Item>> &Hero::getInventory() const {
    return inventory_;
}

bool Hero::unlock(const Achievement &a) {
    if (std::find(achievements_.begin(), achievements_.end(), &a) != achievements_.end())
        return false;
    achievements_.push_back(&a);
    return true;
}
} // namespace lab3
```

У врагов цепочка задается прямо в конструкторе. Общий класс врага пропускает урон через цепочку, если она есть:

**`labs/lab3/enemy/abstract_enemy.hpp`**

```cpp
#pragma once
#include "defense/damage_handler.hpp"

#include <string>

#include "combat/hit.hpp"
#include "enemy/enemy.hpp"

namespace lab3 {

// общая логика для врагов: здоровье, урон в диапазоне, награда
class AbstractEnemy : public Enemy {
  public:
    Hit rollDamage() override;

    Damage receiveHit(int damage) override;

    bool isAlive() const override;

    int rollGoldReward() const override;

    int getExperienceReward() const override;

    const std::string &getCode() const override;

    const std::string &getName() const override;

    int getHealth() const override;

    int getMaxHealth() const override;

  protected:
    AbstractEnemy(std::string code, std::string name, int maxHealth, int minDamage, int maxDamage,
                  int experienceReward, int goldReward, std::unique_ptr<DamageHandler> defense = nullptr);

  private:
    std::string code_;
    std::string name_;
    int maxHealth_;
    int minDamage_;
    int maxDamage_;
    int experienceReward_;
    int goldReward_;
    int health_;
    std::unique_ptr<DamageHandler> defense_;
};

} // namespace lab3
```

**`labs/lab3/enemy/abstract_enemy.cpp`**

```cpp
#include "enemy/abstract_enemy.hpp"

#include <algorithm>
#include <utility>

#include "core/dice.hpp"

namespace lab3 {

AbstractEnemy::AbstractEnemy(std::string code, std::string name, int maxHealth, int minDamage, int maxDamage,
                             int experienceReward, int goldReward, std::unique_ptr<DamageHandler> defense)
    : code_(std::move(code)), name_(std::move(name)), maxHealth_(maxHealth), minDamage_(minDamage),
      maxDamage_(maxDamage), experienceReward_(experienceReward), goldReward_(goldReward), health_(maxHealth),
      defense_(std::move(defense)) {}

Hit AbstractEnemy::rollDamage() {
    return {Dice::getInstance().roll(minDamage_, maxDamage_), false};
}

Damage AbstractEnemy::receiveHit(int amount) {
    Damage damage(amount);
    if (defense_)
        defense_->handle(damage);
    health_ = std::max(0, health_ - damage.getAmount());
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

const std::string &AbstractEnemy::getCode() const {
    return code_;
}

const std::string &AbstractEnemy::getName() const {
    return name_;
}

int AbstractEnemy::getHealth() const {
    return health_;
}

int AbstractEnemy::getMaxHealth() const {
    return maxHealth_;
}

} // namespace lab3
```

У гоблина защиты нет:

**`labs/lab3/enemy/goblin.hpp`**

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

Кости скелета и чешуя дракона работают как броня, поэтому метод `reduceDamage` им больше не нужен:

**`labs/lab3/enemy/skeleton.hpp`**

```cpp
#pragma once
#include "defense/armor_handler.hpp"

#include <cmath>

#include "enemy/abstract_enemy.hpp"

namespace lab3 {

// скелет: кости плохо поддаются ударам, поэтому часть урона проходит мимо
class Skeleton : public AbstractEnemy {
  public:
    Skeleton()
        : AbstractEnemy("skeleton", "Скелет", 110, 10, 24, 90, 70,
                        std::make_unique<ArmorHandler>(RESISTANCE)) {}

  private:
    static constexpr double RESISTANCE = 0.3;
};

} // namespace lab3
```

**`labs/lab3/enemy/dragon.hpp`**

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
  private:
    static constexpr double RESISTANCE = 0.25;
    static constexpr double FIRE_BREATH_CHANCE = 0.25;
    static constexpr int FIRE_BREATH_MULTIPLIER = 2;
};

} // namespace lab3
```

**`labs/lab3/enemy/dragon.cpp`**

```cpp
#include "defense/armor_handler.hpp"
#include "enemy/dragon.hpp"

#include <cmath>

#include "core/dice.hpp"

namespace lab3 {

Dragon::Dragon()
    : AbstractEnemy("dragon", "Дракон", 260, 16, 22, 500, 400, std::make_unique<ArmorHandler>(RESISTANCE)) {}

Hit Dragon::rollDamage() {
    Hit bite = AbstractEnemy::rollDamage();
    if (Dice::getInstance().chance(FIRE_BREATH_CHANCE)) {
        return {bite.damage * FIRE_BREATH_MULTIPLIER, true};
    }

    return bite;
}

} // namespace lab3
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

**`labs/lab3/game/game_facade.hpp`**

```cpp
#pragma once
#include "camp/merchant.hpp"
#include "world/location_registry.hpp"
#include "world/forest.hpp"
#include "world/haunted_manor.hpp"
#include "world/crypt.hpp"
#include "world/dragon_lair.hpp"
#include "tactic/balanced_tactic.hpp"
#include "tactic/aggressive_tactic.hpp"
#include "tactic/cautious_tactic.hpp"
#include "storage/file_hero_repository.hpp"
#include "storage/caching_hero_repository.hpp"
#include "event/game_event_publisher.hpp"
#include "progress/achievement_tracker.hpp"
#include "progress/expedition_journal.hpp"
#include "item/default_catalog.hpp"

namespace lab3 {
class GameFacade {
    ItemCatalog catalog_ = createDefaultCatalog();
    LocationRegistry locations_;
    std::vector<std::unique_ptr<Tactic>> tactics_;
    Merchant merchant_;
    CachingHeroRepository heroes_;
    GameEventPublisher events_;
    std::shared_ptr<ExpeditionJournal> journal_ = std::make_shared<ExpeditionJournal>();
    static std::vector<std::unique_ptr<Location>> locations(const ItemCatalog &catalog) {
        std::vector<std::unique_ptr<Location>> result;
        result.push_back(std::make_unique<Forest>(catalog));
        result.push_back(std::make_unique<HauntedManor>(catalog));
        result.push_back(std::make_unique<Crypt>(catalog));
        result.push_back(std::make_unique<DragonLair>(catalog));
        return result;
    }

  public:
    explicit GameFacade(const std::filesystem::path &file)
        : locations_(locations(catalog_)),
          merchant_(catalog_, {item_codes::IRON_SWORD, item_codes::ELVEN_BOW, item_codes::EMBER_STAFF,
                               item_codes::CHAINMAIL, item_codes::PLATE_ARMOR}),
          heroes_(std::make_unique<FileHeroRepository>(file, catalog_)) {
        tactics_.push_back(std::make_unique<BalancedTactic>());
        tactics_.push_back(std::make_unique<AggressiveTactic>());
        tactics_.push_back(std::make_unique<CautiousTactic>());
        auto achievements = std::make_shared<AchievementTracker>();
        events_.subscribe(GameEventType::VICTORY, achievements);
        events_.subscribe(GameEventType::LEVEL_UP, achievements);
        events_.subscribe(GameEventType::VICTORY, journal_);
        events_.subscribe(GameEventType::DEFEAT, journal_);
    }
    GameFacade(const GameFacade &) = delete;
    GameFacade &operator=(const GameFacade &) = delete;
    GameFacade(GameFacade &&) = delete;
    GameFacade &operator=(GameFacade &&) = delete;
    const std::vector<const HeroClass *> &getHeroClasses() const {
        return HeroClass::values();
    }
    std::vector<std::unique_ptr<Item>> getStarterKit(const std::string &code) const {
        auto kit = HeroClass::byCode(code).getStarterKit(catalog_);
        std::vector<std::unique_ptr<Item>> result;
        result.push_back(kit->createWeapon());
        result.push_back(kit->createArmor());
        return result;
    }
    auto getLocations() const {
        return locations_.getAll();
    }
    const Location &getLocation(const std::string &code) const {
        return locations_.get(code);
    }
    std::vector<const Tactic *> getTactics() const {
        std::vector<const Tactic *> result;
        for (auto &t : tactics_)
            result.push_back(t.get());
        return result;
    }
    const Tactic &getTactic(const std::string &code) const {
        for (auto &t : tactics_)
            if (t->getCode() == code)
                return *t;
        throw UnknownCodeError("Тактика", code);
    }
    auto getShowcase() const {
        return merchant_.getShowcase();
    }
    int getSellPrice(const Item &item) const {
        return merchant_.getSellPrice(item);
    }
    auto getHeroes() {
        return heroes_.findAll();
    }
    std::shared_ptr<Hero> getHero(const std::string &name) {
        auto h = heroes_.findByName(name);
        if (!h)
            throw std::invalid_argument("Героя с именем " + name + " нет");
        return h;
    }
    std::shared_ptr<Hero> createHero(const std::string &name, const std::string &code) {
        if (heroes_.findByName(name))
            throw std::invalid_argument("Герой с именем " + name + " уже есть");
        auto &hc = HeroClass::byCode(code);
        auto h = std::make_shared<Hero>(Hero::Builder()
                                            .setName(name)
                                            .setHeroClass(hc)
                                            .setStarterKit(*hc.getStarterKit(catalog_))
                                            .build());
        heroes_.save(h);
        return h;
    }
    ExpeditionReport explore(const std::string &name, const std::string &locationCode,
                             const std::string &tacticCode) {
        auto h = getHero(name);
        auto report = getLocation(locationCode).explore(*h, getTactic(tacticCode));
        auto result =
            report.battle.outcome == Outcome::VICTORY ? GameEventType::VICTORY : GameEventType::DEFEAT;
        events_.publish({result, *h, report});
        if (report.rewards.levelsGained > 0)
            events_.publish({GameEventType::LEVEL_UP, *h, report});
        heroes_.save(h);
        return report;
    }
    std::shared_ptr<Item> equip(const std::string &name, std::size_t index) {
        auto h = getHero(name);
        auto item = h->getInventory().at(index);
        h->equip(item);
        heroes_.save(h);
        return item;
    }
    std::shared_ptr<Item> buy(const std::string &name, const std::string &code) {
        auto h = getHero(name);
        auto item = merchant_.buy(*h, code);
        heroes_.save(h);
        return item;
    }
    int sell(const std::string &name, std::size_t index) {
        auto h = getHero(name);
        int price = merchant_.sell(*h, h->getInventory().at(index));
        heroes_.save(h);
        return price;
    }
    auto getJournal(const std::string &name) const {
        return journal_->getEntries(name);
    }
};
} // namespace lab3
```

Вот здесь и настраиваются связи между издателем и подписчиками, а также происходит отправка событий после экспедиции.
Одно и то же событие (победу) получают оба подписчика: и выдача достижений, и журнал. Если завтра мы захотим, например,
отправлять уведомления о победах в какой-нибудь мессенджер, достаточно будет написать нового подписчика и подписать его в конструкторе
фасада.

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

**`labs/lab3/console_view.hpp`**

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

    std::string describe(const Item &item) const;

    void printHero(const Hero &hero) const;

    std::string summarize(const ExpeditionReport &report, const std::string &locationName,
                          const std::string &tacticName) const;

    void printReport(const ExpeditionReport &report, const std::string &locationName) const;

  private:
    void printEvent(const BattleEvent &event, const BattleReport &battle) const;

    void printRewards(const BattleReport &battle, const Rewards &rewards) const;

    const Participant &participant(Side side, const BattleReport &battle) const;
};

} // namespace lab3
```

**`labs/lab3/console_view.cpp`**

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

std::string ConsoleView::describe(const Item &item) const {
    std::string description = item.getName();

    const auto &enchantments = item.getEnchantments();
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

void ConsoleView::printHero(const Hero &hero) const {
    std::cout << hero.getName() << ", " << hero.getHeroClass().getDisplayName() << " " << hero.getLevel()
              << " ур. | здоровье " << hero.getMaxHealth() << " | опыт " << hero.getExperience() << "/"
              << hero.getExperienceToNextLevel() << " | золото " << hero.getGold() << "\n";
    std::cout << "  Оружие: " << describe(hero.getWeapon()) << "\n";
    std::cout << "  Броня:  " << describe(hero.getArmor()) << "\n";
    if (!hero.getAchievements().empty()) {
        std::cout << "  Достижения: ";
        bool first = true;
        for (auto a : hero.getAchievements()) {
            if (!first)
                std::cout << ", ";
            std::cout << a->displayName;
            first = false;
        }
        std::cout << "\n";
    }
}

void ConsoleView::printReport(const ExpeditionReport &report, const std::string &locationName) const {
    const BattleReport &battle = report.battle;
    std::cout << "=== " << locationName << " (сид " << report.seed << ") ===\n";
    std::cout << battle.hero.name << " встречает врага: " << battle.enemy.name << " ("
              << battle.enemy.maxHealth << " здоровья)\n";

    for (const BattleEvent &event : battle.events) {
        printEvent(event, battle);
    }

    printRewards(battle, report.rewards);
}

void ConsoleView::printEvent(const BattleEvent &event, const BattleReport &battle) const {
    const Participant &source = participant(event.source, battle);
    const Participant &target = participant(event.target, battle);

    switch (event.type) {
    case EventType::HIT:
        std::cout << "[раунд " << event.round << "] " << source.name << " -> " << target.name << ": "
                  << event.amount << " урона, осталось " << event.targetHealthAfter << "/" << target.maxHealth
                  << "\n";
        break;
    case EventType::CRITICAL_HIT:
        std::cout << "[раунд " << event.round << "] " << source.name << " -> " << target.name << ": КРИТ! "
                  << event.amount << " урона, осталось " << event.targetHealthAfter << "/" << target.maxHealth
                  << "\n";
        break;
    case EventType::EVADE:
        std::cout << "[раунд " << event.round << "] " << source.name << " -> " << target.name << ": промах, "
                  << target.name << " уклоняется\n";
        break;
    case EventType::ABSORB:
        std::cout << "[раунд " << event.round << "] " << source.name << " -> " << target.name
                  << ": барьер поглощает " << event.amount << " урона\n";
        break;
    case EventType::POWER_STRIKE:
        std::cout << "[раунд " << event.round << "] " << source.name << " бьет в полную силу\n";
        break;
    case EventType::DEFEND:
        std::cout << "[раунд " << event.round << "] " << source.name << " уходит в оборону\n";
        break;
    case EventType::DEATH:
        std::cout << target.name << " повержен!\n";
        break;
    }
}

void ConsoleView::printRewards(const BattleReport &battle, const Rewards &rewards) const {
    if (battle.outcome == Outcome::DEFEAT) {
        std::cout << "Поражение... " << battle.hero.name << " едва уносит ноги и теряет " << -rewards.gold
                  << " золота\n";
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

const Participant &ConsoleView::participant(Side side, const BattleReport &battle) const {
    return side == Side::HERO ? battle.hero : battle.enemy;
}

std::string ConsoleView::summarize(const ExpeditionReport &report, const std::string &locationName,
                                   const std::string &tacticName) const {
    return locationName + ", " + report.battle.enemy.name + ", тактика \"" + tacticName +
           "\": " + (report.battle.outcome == Outcome::VICTORY ? "победа" : "поражение");
}
} // namespace lab3
```

А игровой цикл в методе `main` теперь работает только с фасадом и хранит текущего героя:

Вместо прямой работы с каталогом, локациями и торговцем:

- создаём `GameFacade` с файлом сохранения и выбираем нового или сохранённого героя;
- перед экспедицией выбираем тактику;
- работам с экспедициями, экипировкой и торговлей через фасад;
- добавляем пункт меню для журнала экспедиций.

**`labs/lab3/main.cpp`**

```cpp
#include "game/game_facade.hpp"
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
    } catch (const std::exception &) {
        // некорректный ввод обрабатываем так же, как отказ от выбора
    }

    return -1;
}

std::shared_ptr<Hero> createHero(GameFacade &game) {
    std::cout << "Создайте своего героя!\n";
    while (true) {
        std::cout << "Выберите класс:\n";
        const auto &heroClasses = HeroClass::values();
        for (std::size_t i = 0; i < heroClasses.size(); i++) {
            auto kit = game.getStarterKit(heroClasses[i]->getCode());
            std::cout << i + 1 << " - " << heroClasses[i]->getDisplayName() << " (здоровье "
                      << heroClasses[i]->getBaseHealth() << "): " << VIEW.describe(*kit[0]) << ", "
                      << VIEW.describe(*kit[1]) << "\n";
        }
        int classIndex = readChoice(heroClasses.size());
        if (classIndex < 0) {
            continue;
        }
        const HeroClass &heroClass = *heroClasses[static_cast<std::size_t>(classIndex)];

        std::cout << "Введите имя:\n";
        std::string name = readLine();

        try {
            return game.createHero(name, heroClass.getCode());
        } catch (const std::invalid_argument &error) {
            std::cout << "Не получилось создать героя: " << error.what() << "\n";
        }
    }
}

std::shared_ptr<Hero> chooseHero(GameFacade &game) {
    auto heroes = game.getHeroes();
    if (heroes.empty())
        return createHero(game);
    std::cout << "Выберите сохраненного героя (0 - создать нового):\n";
    for (std::size_t i = 0; i < heroes.size(); ++i)
        std::cout << i + 1 << " - " << heroes[i]->getName() << ", "
                  << heroes[i]->getHeroClass().getDisplayName() << " " << heroes[i]->getLevel() << " ур.\n";
    int index = readChoice(heroes.size());
    return index >= 0 ? heroes[static_cast<std::size_t>(index)] : createHero(game);
}

void goOnExpedition(Hero &hero, GameFacade &game) {
    std::vector<const Location *> all = game.getLocations();
    std::cout << "Куда отправимся?\n";
    for (std::size_t i = 0; i < all.size(); i++) {
        std::cout << i + 1 << " - " << all[i]->getDisplayName() << " (рекомендуемый уровень "
                  << all[i]->getRecommendedLevel() << ")\n";
    }
    int index = readChoice(all.size());
    if (index < 0) {
        return;
    }

    const Location &location = *all[static_cast<std::size_t>(index)];
    std::cout << "Выберите тактику:\n";
    auto tactics = game.getTactics();
    for (std::size_t i = 0; i < tactics.size(); ++i)
        std::cout << i + 1 << " - " << tactics[i]->getDisplayName() << ": " << tactics[i]->getDescription()
                  << "\n";
    int tacticIndex = readChoice(tactics.size());
    if (tacticIndex < 0)
        return;
    ExpeditionReport report = game.explore(hero.getName(), location.getCode(),
                                           tactics[static_cast<std::size_t>(tacticIndex)]->getCode());
    std::cout << "\n";
    VIEW.printReport(report, location.getDisplayName());

    if (report.battle.outcome == Outcome::VICTORY && report.battle.enemy.code == "dragon") {
        std::cout << "\nДракон повержен! Барды еще долго будут петь о подвиге героя по имени "
                  << hero.getName() << ".\n";
    }
}

void printItems(const std::vector<std::shared_ptr<Item>> &items) {
    for (std::size_t i = 0; i < items.size(); i++) {
        std::cout << i + 1 << " - " << VIEW.describe(*items[i]) << "\n";
    }
}

void manageEquipment(Hero &hero, GameFacade &game) {
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

    const auto &item = inventory[static_cast<std::size_t>(index)];
    game.equip(hero.getName(), static_cast<std::size_t>(index));
    std::cout << "Экипировано: " << VIEW.describe(*item) << "\n";
}

void buy(Hero &hero, GameFacade &game) {
    std::vector<std::unique_ptr<Item>> showcase = game.getShowcase();
    for (std::size_t i = 0; i < showcase.size(); i++) {
        std::cout << i + 1 << " - " << VIEW.describe(*showcase[i]) << ": " << showcase[i]->getPrice()
                  << " золота\n";
    }
    int index = readChoice(showcase.size());
    if (index < 0) {
        return;
    }

    try {
        auto item = game.buy(hero.getName(), showcase[static_cast<std::size_t>(index)]->getCode());
        std::cout << "Куплено: " << VIEW.describe(*item) << "\n";
    } catch (const std::runtime_error &error) {
        std::cout << error.what() << "\n";
    }
}

void sell(Hero &hero, GameFacade &game) {
    std::vector<std::shared_ptr<Item>> inventory = hero.getInventory();
    if (inventory.empty()) {
        std::cout << "Продавать нечего (экипированные предметы продать нельзя)\n";
        return;
    }

    for (std::size_t i = 0; i < inventory.size(); i++) {
        std::cout << i + 1 << " - " << VIEW.describe(*inventory[i]) << ": "
                  << game.getSellPrice(*inventory[i]) << " золота\n";
    }
    int index = readChoice(inventory.size());
    if (index < 0) {
        return;
    }

    const auto &item = inventory[static_cast<std::size_t>(index)];
    int price = game.sell(hero.getName(), static_cast<std::size_t>(index));
    std::cout << "Продано: " << item->getName() << " за " << price << " золота\n";
}

void visitMerchant(Hero &hero, GameFacade &game) {
    std::cout << "Торговец: \"Чего желаешь?\" (золото: " << hero.getGold() << ")\n";
    std::cout << "1 - купить, 2 - продать, 0 - назад\n";
    std::string choice = readLine();
    if (choice == "1") {
        buy(hero, game);
    } else if (choice == "2") {
        sell(hero, game);
    }
}

} // namespace

int main() try {
    ConsoleView::setupConsole();

    std::filesystem::path saveFile = std::filesystem::path("saves") / "heroes.json";
    auto savePath = std::filesystem::absolute(saveFile).u8string();
    std::cout << "Файл сохранения: " << std::string(savePath.begin(), savePath.end()) << "\n";
    GameFacade game(saveFile);
    auto selectedHero = chooseHero(game);
    Hero &hero = *selectedHero;
    std::cout << hero.getName() << " разбивает лагерь на опушке мистического леса.\n";

    while (true) {
        std::cout << "\n";
        VIEW.printHero(hero);
        std::cout
            << "1 - отправиться в экспедицию, 2 - снаряжение, 3 - торговец, 4 - журнал, 0 - выйти из игры\n";
        std::string choice = readLine();
        if (choice == "1") {
            goOnExpedition(hero, game);
        } else if (choice == "2") {
            manageEquipment(hero, game);
        } else if (choice == "3") {
            visitMerchant(hero, game);
        } else if (choice == "4") {
            for (auto &report : game.getJournal(hero.getName()))
                std::cout << VIEW.summarize(report, game.getLocation(report.locationCode).getDisplayName(),
                                            game.getTactic(report.tacticCode).getDisplayName())
                          << "\n";
        } else if (choice == "0") {
            return 0;
        } else {
            std::cout << "Такого варианта нет\n";
        }
    }
}
catch (const std::exception &error) {
    std::cerr << error.what() << "\n";
    return 1;
}
```

Супер! Попробуйте запустить игру, сохраниться и вернуться к своему герою :)

## Демонстрация паттернов

Чтобы показать все паттерны одним запуском, напишем демонстрационную программу:

Демонстрацию лабы 3 заменяем новой:

**`labs/lab3/demo.cpp`**

```cpp
#include <iostream>
#include <fstream>
#include <sstream>
#include <filesystem>
#include <chrono>
#include <memory>
#include <vector>
#include "console_view.hpp"
#include "game/game_facade.hpp"
#include "defense/armor_handler.hpp"
#include "defense/barrier_handler.hpp"
#include "defense/evasion_handler.hpp"
#include "enemy/goblin.hpp"
#include "enemy/dragon.hpp"
#include "enemy/legendary_enemy.hpp"
#include "enemy/windfury_enemy.hpp"
#include "enemy/armored_enemy.hpp"
#include "enemy/haunted_weapon.hpp"
using namespace lab3;
namespace {
const ConsoleView VIEW;
Hero newHero(const std::string &name, const HeroClass &hc, int level, const ItemCatalog &catalog) {
    return Hero::Builder()
        .setName(name)
        .setHeroClass(hc)
        .setStarterKit(*hc.getStarterKit(catalog))
        .setLevel(level)
        .build();
}
void printHits(const std::string &title, DamageHandler &chain, int count) {
    std::cout << title << ": ";
    for (int i = 0; i < count; ++i) {
        if (i)
            std::cout << "; ";
        Damage d(30);
        chain.handle(d);
        if (d.isEvaded())
            std::cout << "уклонение";
        else if (d.getAbsorbed() > 0)
            std::cout << "поглощено " << d.getAbsorbed() << ", прошло " << d.getAmount();
        else
            std::cout << "прошло " << d.getAmount();
    }
    std::cout << "\n";
}
class EventPrinter : public GameEventListener {
  public:
    void onEvent(const GameEvent &e) override {
        std::cout << "Событие VICTORY: герой " << e.hero.getName() << "\n";
    }
};
class TempDirectory {
  public:
    std::filesystem::path path;
    TempDirectory() {
        auto prefix = std::to_string(std::chrono::steady_clock::now().time_since_epoch().count());
        for (int i = 0;; ++i) {
            path = std::filesystem::temp_directory_path() / ("lab-demo-" + prefix + "-" + std::to_string(i));
            if (std::filesystem::create_directory(path))
                break;
        }
    }
    ~TempDirectory() {
        std::error_code error;
        std::filesystem::remove_all(path, error);
    }
};
} // namespace
int main() {
    ConsoleView::setupConsole();
    std::cout << std::boolalpha;
    auto catalog = createDefaultCatalog();
    auto &dice = Dice::getInstance();
    std::cout << "===== Цепочка обязанностей =====\n";
    ArmorHandler armor(0.3);
    printHits("Броня 30%", armor, 2);
    ArmorHandler barrier(0.3);
    barrier.setNext(std::make_unique<BarrierHandler>(30));
    printHits("Броня 30% -> барьер 30", barrier, 3);
    dice.reseed(3);
    EvasionHandler chain(0.5);
    chain.setNext(std::make_unique<ArmorHandler>(0.3)).setNext(std::make_unique<BarrierHandler>(30));
    printHits("Уклонение 50% -> броня 30% -> барьер 30", chain, 3);
    std::cout << "\n===== Стратегия =====\n";
    BalancedTactic balanced;
    AggressiveTactic aggressive;
    CautiousTactic cautious;
    for (const Tactic *tactic : std::vector<const Tactic *>{&balanced, &aggressive, &cautious}) {
        auto mage = newHero("Мерлин", HeroClass::MAGE, 2, catalog);
        auto report = Crypt(catalog).explore(mage, *tactic, 9);
        std::cout << tactic->getDisplayName() << ": "
                  << (report.battle.outcome == Outcome::VICTORY ? "победа" : "поражение") << ", раундов "
                  << report.battle.events.back().round << ", у героя осталось " << mage.getHealth() << "/"
                  << mage.getMaxHealth() << " здоровья\n";
    }
    std::cout << "\n===== Декоратор =====\n";
    std::vector<std::unique_ptr<Enemy>> enemies;
    enemies.push_back(std::make_unique<Goblin>());
    enemies.push_back(std::make_unique<LegendaryEnemy>(std::make_unique<Goblin>()));
    enemies.push_back(std::make_unique<WindfuryEnemy>(std::make_unique<Goblin>()));
    enemies.push_back(
        std::make_unique<LegendaryEnemy>(std::make_unique<WindfuryEnemy>(std::make_unique<Goblin>())));
    enemies.push_back(std::make_unique<ArmoredEnemy>(std::make_unique<Goblin>()));
    enemies.push_back(std::make_unique<ArmoredEnemy>(
        std::make_unique<LegendaryEnemy>(std::make_unique<WindfuryEnemy>(std::make_unique<Dragon>()))));
    for (auto &enemy : enemies) {
        dice.reseed(5);
        std::ostringstream hits;
        for (int i = 0; i < 3; ++i) {
            if (i)
                hits << ", ";
            hits << enemy->rollDamage().damage;
        }
        auto d = enemy->receiveHit(20);
        std::cout << enemy->getName() << ": удары " << hits.str() << ", награда "
                  << enemy->getExperienceReward() << " опыта; входящий урон 20 -> " << d.getAmount()
                  << ", здоровье " << enemy->getHealth() << "/" << enemy->getMaxHealth() << "\n";
    }
    std::cout << "\n===== Адаптер =====\n";
    std::shared_ptr<Weapon> bow = catalog.createWeapon(item_codes::ELVEN_BOW);
    HauntedWeapon haunted(bow);
    std::cout << "Предмет: " << VIEW.describe(*bow) << "\nВраг:    " << haunted.getName() << " ("
              << haunted.getMaxHealth() << " здоровья, награда " << haunted.getExperienceReward()
              << " опыта)\n";
    HauntedManor manor(catalog);
    auto ranger = newHero("Леголас", HeroClass::RANGER, 2, catalog);
    VIEW.printReport(manor.explore(ranger, cautious, 55), manor.getDisplayName());
    std::cout << "\n===== Наблюдатель =====\n";
    GameEventPublisher events;
    auto journal = std::make_shared<ExpeditionJournal>();
    events.subscribe(GameEventType::VICTORY, std::make_shared<AchievementTracker>());
    events.subscribe(GameEventType::VICTORY, journal);
    events.subscribe(GameEventType::DEFEAT, journal);
    events.subscribe(GameEventType::VICTORY, std::make_shared<EventPrinter>());
    auto warrior = std::make_shared<Hero>(newHero("Артур", HeroClass::WARRIOR, 1, catalog));
    auto report = Forest(catalog).explore(*warrior, 1);
    events.publish(
        {report.battle.outcome == Outcome::VICTORY ? GameEventType::VICTORY : GameEventType::DEFEAT, *warrior,
         report});
    std::cout << "Достижения героя: ";
    bool first = true;
    for (auto a : warrior->getAchievements()) {
        if (!first)
            std::cout << ", ";
        std::cout << a->displayName;
        first = false;
    }
    std::cout << "\nЗаписей в журнале: " << journal->getEntries(warrior->getName()).size() << "\n";
    TempDirectory temp;
    std::cout << "\n===== Заместитель =====\n";
    auto path = temp.path / "heroes.json";
    FileHeroRepository file(path, catalog);
    file.save(warrior);
    std::cout << "Содержимое файла:\n";
    std::ifstream input(path, std::ios::binary);
    std::cout << input.rdbuf();
    input.close();
    std::cout << "Без заместителя два поиска дают один и тот же объект: "
              << (file.findByName("Артур") == file.findByName("Артур")) << "\n";
    CachingHeroRepository cached(std::make_unique<FileHeroRepository>(path, catalog));
    auto h = cached.findByName("Артур");
    std::cout << "С заместителем два поиска дают один и тот же объект: " << (h == cached.findByName("Артур"))
              << "\n";
    h->addGold(100);
    cached.save(h);
    std::cout << "После сохранения через заместитель золото героя в файле: "
              << file.findByName("Артур")->getGold() << "\n";
    std::cout << "\n===== Фасад =====\n";
    GameFacade game(temp.path / "game.json");
    game.createHero("Гэндальф", "mage");
    dice.reseed(11);
    auto expedition = game.explore("Гэндальф", "forest", "cautious");
    std::cout << "Экспедиция через фасад: "
              << VIEW.summarize(expedition, game.getLocation(expedition.locationCode).getDisplayName(),
                                game.getTactic(expedition.tacticCode).getDisplayName())
              << "\n";
    VIEW.printHero(*game.getHero("Гэндальф"));
}
```

Вывод программы:

```text
===== Цепочка обязанностей =====
Броня 30%: прошло 21; прошло 21
Броня 30% -> барьер 30: поглощено 21, прошло 0; поглощено 9, прошло 12; прошло 21
Уклонение 50% -> броня 30% -> барьер 30: поглощено 21, прошло 0; уклонение; поглощено 9, прошло 12

===== Стратегия =====
Баланс: поражение, раундов 5, у героя осталось 0/100 здоровья
Натиск: поражение, раундов 3, у героя осталось 0/100 здоровья
Осторожность: поражение, раундов 5, у героя осталось 0/100 здоровья

===== Декоратор =====
Гоблин: удары 14, 9, 13, награда 40 опыта; входящий урон 20 -> 20, здоровье 30/50
Легендарный Гоблин: удары 28, 18, 26, награда 80 опыта; входящий урон 20 -> 20, здоровье 30/50
Неистовый Гоблин: удары 23, 22, 27, награда 60 опыта; входящий урон 20 -> 20, здоровье 30/50
Легендарный Неистовый Гоблин: удары 46, 44, 54, награда 120 опыта; входящий урон 20 -> 20, здоровье 30/50
Бронированный Гоблин: удары 14, 9, 13, награда 40 опыта; входящий урон 20 -> 10, здоровье 40/50
Бронированный Легендарный Неистовый Дракон: удары 104, 78, 116, награда 1500 опыта; входящий урон 20 -> 8, здоровье 252/260

===== Адаптер =====
Предмет: Эльфийский лук (урон 14, шанс крита 40%)
Враг:    Одержимый Эльфийский лук (70 здоровья, награда 45 опыта)
=== Проклятый особняк (сид 55) ===
Леголас встречает врага: Одержимый Эльфийский лук (70 здоровья)
[раунд 1] Леголас -> Одержимый Эльфийский лук: 13 урона, осталось 57/70
[раунд 1] Одержимый Эльфийский лук -> Леголас: 11 урона, осталось 104/115
[раунд 2] Леголас -> Одержимый Эльфийский лук: КРИТ! 23 урона, осталось 34/70
[раунд 2] Одержимый Эльфийский лук -> Леголас: КРИТ! 22 урона, осталось 82/115
[раунд 3] Леголас -> Одержимый Эльфийский лук: 13 урона, осталось 21/70
[раунд 3] Одержимый Эльфийский лук -> Леголас: 11 урона, осталось 71/115
[раунд 4] Леголас -> Одержимый Эльфийский лук: 13 урона, осталось 8/70
[раунд 4] Одержимый Эльфийский лук -> Леголас: 11 урона, осталось 60/115
[раунд 5] Леголас -> Одержимый Эльфийский лук: КРИТ! 23 урона, осталось 0/70
Одержимый Эльфийский лук повержен!
Победа! +45 опыта, +44 золота
Добыча: Тлеющий посох (урон 14-26)

===== Наблюдатель =====
Событие VICTORY: герой Артур
Достижения героя: Первая кровь, Счастливчик
Записей в журнале: 1

===== Заместитель =====
Содержимое файла:
[
  {
    "name": "Артур",
    "heroClass": "warrior",
    "level": 1,
    "experience": 40,
    "gold": 74,
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
        "code": "leather-armor",
        "enchantments": [
          "sturdy"
        ]
      }
    ],
    "achievements": [
      "first-blood",
      "lucky"
    ]
  }
]
Без заместителя два поиска дают один и тот же объект: false
С заместителем два поиска дают один и тот же объект: true
После сохранения через заместитель золото героя в файле: 174

===== Фасад =====
Экспедиция через фасад: Мистический лес, Гоблин, тактика "Осторожность": победа
Гэндальф, Маг 1 ур. | здоровье 85 | опыт 40/100 | золото 72
  Оружие: Посох ученика (урон 10-18)
  Броня:  Льняная роба (защита 10%)
  Достижения: Первая кровь, Счастливчик
```

Не забудьте закоммитить и запушить изменения!

# Лаба 4 — Структурные и поведенческие шаблоны проектирования

В этой лабораторной работе будут рассмотрены структурные и поведенческие шаблоны проектирования.

Структурные паттерны проектирования помогают организовать и упростить построение
связей между объектами и классами. Основная цель структурных паттернов - облегчить работу с компонентами,
делая систему более гибкой и поддерживаемой.

Проще говоря, структурные паттерны связаны с композицией объектов или тем, как сущности могут использовать друг друга.

Поведенческие паттерны проектирования же определяют алгоритмы и способы реализации взаимодействия различных объектов и классов.

# Что нужно сдать к защите

Выполнять лабораторную можно на любом языке программирования.

На защите нужно продемонстрировать работоспособность кода, при этом он должен быть запушен в удалённый репозиторий.

В качестве лабораторной работы мы будем развивать функциональность игры третьей лабораторной работы. Добавим возможность выбора тактики боя
с помощью паттерна "стратегия",
через "декоратор" добавим гибко комбинируемые модификаторы врагов, через "наблюдателя" – ачивки, а еще много всего еще.

Со всем этим нам помогут структурные и поведенческие паттерны!

## Паттерны

В программе должны быть реализованы следующие структурные паттерны:

1. Декоратор (Decorator)
2. Заместитель (Proxy)
3. Адаптер (Adapter)
4. Фасад (Facade)

И следующие поведенческие паттерны:

1. Стратегия (Strategy)
2. Наблюдатель (Observer)
3. Цепочка обязанностей (Chain of responsibility)

# Вопросы к защите

Отвечая на вопросы про конкретные паттерны, покажите, где и как этот паттерн реализован в вашем коде.

1. Что такое структурные паттерны проектирования? В чем их суть?
2. Паттерн Декоратор. Что делает, реализация, плюсы и минусы, когда применять?
3. Паттерн Заместитель. Что делает, реализация, плюсы и минусы, когда применять?
4. Паттерн Адаптер. Что делает, реализация, плюсы и минусы, когда применять?
5. Паттерн Фасад. Что делает, реализация, плюсы и минусы, когда применять?
6. Чем декоратор отличается от заместителя?
7. Чем адаптер отличается от фасада?
8. Что такое поведенческие паттерны проектирования? В чем их суть?
9. Паттерн Стратегия. Что делает, реализация, плюсы и минусы, когда применять?
10. Паттерн Наблюдатель. Что делает, реализация, плюсы и минусы, когда применять?
11. Паттерн Цепочка обязанностей. Что делает, реализация, плюсы и минусы, когда применять?

# Практика

> Если вы пишете не на Java, эта же практика есть на других
>
языках: [Python](practice/python.md), [C++](practice/cpp.md), [Go](practice/go.md), [C#](practice/csharp.md), [Kotlin](practice/kotlin.md), [TypeScript](practice/typescript.md).

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
Базовый пакет `org.misis.tp.lab3` и расположение исходников сохраняются; создавать пакет `lab4` и копировать
классы не нужно. Дальше мы разберем только новые и измененные классы, остальные остаются как в прошлой лабе.

Новые и измененные классы приведены целиком. Ключевые изменения в коде отмечены короткими комментариями.

## Подготовка: данные и контракты

Наши паттерны потребуют немного подготовки перед введением их в действие! Давайте же эту подготовку приведем.

Старого механизма обработки урона нам теперь не хватает. Раньше мы работали просто с числами, теперь давайте масштабироваться.
Заведем отдельный класс `Damage`. Он будет инкапсулировать в себе результат получения урона
(сколько получили, удалось ли уклонится, сколько было поглощено), эти данные понадобятся нам дальше.

```java
package org.misis.tp.lab3.defense;

public class Damage {

    private int amount;
    private boolean evaded;
    private int absorbed;

    public Damage(int amount) {
        this.amount = amount;
    }

    public void reduceTo(int amount) {
        this.amount = amount;
    }

    public void evade() {
        amount = 0;
        evaded = true;
    }

    public void absorb(int value) {
        amount -= value;
        absorbed += value;
    }

    public int getAmount() {
        return amount;
    }

    public boolean isEvaded() {
        return evaded;
    }

    public int getAbsorbed() {
        return absorbed;
    }

}
```

И теперь `Combatant.receiveHit` будет возвращать `Damage` вместо `int`. Так мы сможем реализовать отдельные механики защиты от урона.
Получится что-то такое:

```java
package org.misis.tp.lab3.combat;

import org.misis.tp.lab3.defense.Damage;

public interface Combatant {

    String getCode();

    String getName();

    int getHealth();

    int getMaxHealth();

    boolean isAlive();

    Hit rollDamage();

    // возвращаем Damage вместо int, расширяемся под гибкую обработку нанесенного урона
    Damage receiveHit(int damage);

}
```

Для врагов давайте обработку урона в `receiveHit` также сделаем через `Damage`, не зря же мы этот класс добавляли:

```java
package org.misis.tp.lab3.enemy;

import org.misis.tp.lab3.combat.Hit;
import org.misis.tp.lab3.defense.Damage;
import org.misis.tp.lab3.core.Dice;

// общая логика для врагов
public abstract class AbstractEnemy implements Enemy {

    private final String code;
    private final String name;
    private final int maxHealth;
    private final int minDamage;
    private final int maxDamage;
    private final int experienceReward;
    private final int goldReward;
    private int health;

    protected AbstractEnemy(
        String code, String name, int maxHealth,
        int minDamage, int maxDamage,
        int experienceReward, int goldReward
    ) {
        this.code = code;
        this.name = name;
        this.maxHealth = maxHealth;
        this.minDamage = minDamage;
        this.maxDamage = maxDamage;
        this.experienceReward = experienceReward;
        this.goldReward = goldReward;
        this.health = maxHealth;
    }

    @Override
    public Hit rollDamage() {
        return new Hit(Dice.getInstance().roll(minDamage, maxDamage), false);
    }

    @Override
    public Damage receiveHit(int damage) {
        int dealt = Math.max(1, reduceDamage(damage));
        health = Math.max(0, health - dealt);
        return new Damage(dealt);
    }

    protected int reduceDamage(int damage) {
        return damage;
    }

    @Override
    public boolean isAlive() {
        return health > 0;
    }

    @Override
    public int rollGoldReward() {
        return Dice.getInstance().roll(goldReward / 2, goldReward);
    }

    @Override
    public int getExperienceReward() {
        return experienceReward;
    }

    @Override
    public String getCode() {
        return code;
    }

    @Override
    public String getName() {
        return name;
    }

    @Override
    public int getHealth() {
        return health;
    }

    @Override
    public int getMaxHealth() {
        return maxHealth;
    }

}
```

В `Battle` добавим обработку результата `receiveHit`, с учетом того, что он теперь возвращает `Damage`:

```java
package org.misis.tp.lab3.combat;

import java.util.ArrayList;
import java.util.List;

public class Battle {

    private final Combatant hero;
    private final Combatant enemy;
    private final List<BattleEvent> events = new ArrayList<>();
    private int round;

    public Battle(Combatant hero, Combatant enemy) {
        this.hero = hero;
        this.enemy = enemy;
    }

    public BattleReport run() {
        while (hero.isAlive() && enemy.isAlive()) {
            round++;
            strike(hero, Side.HERO, enemy, Side.ENEMY);
            if (enemy.isAlive()) {
                strike(enemy, Side.ENEMY, hero, Side.HERO);
            }
        }

        Outcome outcome = hero.isAlive() ? Outcome.VICTORY : Outcome.DEFEAT;
        return new BattleReport(Participant.of(hero), Participant.of(enemy), outcome, List.copyOf(events));
    }

    private void strike(Combatant attacker, Side source, Combatant defender, Side target) {
        Hit hit = attacker.rollDamage();
        // полученный урон теперь хранится в Damage
        int dealt = defender.receiveHit(hit.damage()).getAmount();
        EventType type = hit.critical() ? EventType.CRITICAL_HIT : EventType.HIT;
        emit(new BattleEvent(round, type, source, target, dealt, defender.getHealth()));

        if (!defender.isAlive()) {
            emit(new BattleEvent(round, EventType.DEATH, source, target, 0, 0));
        }
    }

    private void emit(BattleEvent event) {
        events.add(event);
    }

}
```

А еще у нас в игре появятся достижения. Выдаваться они будут с помощью паттерна "наблюдатель", но объявим мы их сразу здесь.

Представлять собой они будут идентификатор, название и описание. Как-то так:

```java
package org.misis.tp.lab3.progress;

import org.misis.tp.lab3.core.UnknownCodeException;

public enum Achievement {

    FIRST_BLOOD("first-blood", "Первая кровь", "одержать первую победу"),
    LUCKY("lucky", "Счастливчик", "найти зачарованный лут"),
    VETERAN("veteran", "Ветеран", "достичь 5 уровня"),
    DRAGON_SLAYER("dragon-slayer", "Драконоборец", "победить дракона");

    private final String code;
    private final String displayName;
    private final String description;

    Achievement(String code, String displayName, String description) {
        this.code = code;
        this.displayName = displayName;
        this.description = description;
    }

    public static Achievement byCode(String code) {
        for (Achievement achievement : values()) {
            if (achievement.code.equals(code)) {
                return achievement;
            }
        }

        throw new UnknownCodeException("Достижение", code);
    }

    public String getCode() {
        return code;
    }

    public String getDisplayName() {
        return displayName;
    }

    public String getDescription() {
        return description;
    }

}
```

В `Hero` добавим хранение достижений, полученных за этого героя, а также возможность восстановления опыта, инвентаря и достижений через
строитель.

Нам это понадобится, когда будем добавлять механику сохранений!

А еще метод отдыха переименуем в `rest` вместо `restoreHealth`. В нем позже появится дополнительное поведение, поэтому стоит сразу немного
расширить контракт.

Итог выглядит вот так:

```java
package org.misis.tp.lab3.hero;

import org.misis.tp.lab3.combat.Combatant;
import org.misis.tp.lab3.combat.Hit;
import org.misis.tp.lab3.defense.Damage;
import org.misis.tp.lab3.item.Armor;
import org.misis.tp.lab3.item.Item;
import org.misis.tp.lab3.item.Weapon;
import org.misis.tp.lab3.kit.StarterKitFactory;
import org.misis.tp.lab3.progress.Achievement;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;

public class Hero implements Combatant {

    private static final int MAX_NAME_LENGTH = 20;
    private static final int STARTING_GOLD = 50;
    private static final int HEALTH_PER_LEVEL = 15;
    private static final int DAMAGE_PER_LEVEL = 3;
    private static final int EXPERIENCE_PER_LEVEL = 100;

    private final String name;
    private final HeroClass heroClass;
    private int level;
    private int experience;
    private int gold;
    private int health;
    private Weapon weapon;
    private Armor armor;
    private final List<Item> inventory;
    // достижения сохраняются вместе с героем
    private final List<Achievement> achievements;

    private Hero(Builder builder) {
        this.name = builder.name;
        this.heroClass = builder.heroClass;
        this.level = builder.level;
        this.experience = builder.experience;
        this.gold = builder.gold;
        this.weapon = builder.weapon;
        this.armor = builder.armor;
        this.inventory = new ArrayList<>(builder.inventory);
        this.achievements = new ArrayList<>(builder.achievements);
        rest();
    }

    public static class Builder {

        private String name;
        private HeroClass heroClass;
        private Weapon weapon;
        private Armor armor;
        private int level = 1;
        private int experience = 0;
        private int gold = STARTING_GOLD;
        private List<Item> inventory = List.of();
        private List<Achievement> achievements = List.of();

        public Builder setName(String name) {
            this.name = name;
            return this;
        }

        public Builder setHeroClass(HeroClass heroClass) {
            this.heroClass = heroClass;
            return this;
        }

        public Builder setWeapon(Weapon weapon) {
            this.weapon = weapon;
            return this;
        }

        public Builder setArmor(Armor armor) {
            this.armor = armor;
            return this;
        }

        public Builder setStarterKit(StarterKitFactory starterKit) {
            this.weapon = starterKit.createWeapon();
            this.armor = starterKit.createArmor();
            return this;
        }

        public Builder setLevel(int level) {
            this.level = level;
            return this;
        }

        public Builder setExperience(int experience) {
            this.experience = experience;
            return this;
        }

        public Builder setGold(int gold) {
            this.gold = gold;
            return this;
        }

        public Builder setInventory(List<Item> inventory) {
            this.inventory = List.copyOf(inventory);
            return this;
        }

        public Builder setAchievements(List<Achievement> achievements) {
            this.achievements = List.copyOf(achievements);
            return this;
        }

        public Hero build() {
            if (Objects.isNull(name) || name.isBlank()) {
                throw new IllegalArgumentException("У героя должно быть имя");
            }
            if (name.length() > MAX_NAME_LENGTH) {
                throw new IllegalArgumentException(
                    String.format("Имя героя не может быть длиннее %d символов", MAX_NAME_LENGTH)
                );
            }
            if (Objects.isNull(heroClass)) {
                throw new IllegalArgumentException("Не выбран класс героя");
            }
            if (Objects.isNull(weapon) || Objects.isNull(armor)) {
                throw new IllegalArgumentException("Герою нужны оружие и броня");
            }
            if (level < 1) {
                throw new IllegalArgumentException("Уровень героя не может быть меньше 1");
            }
            if (experience < 0) {
                throw new IllegalArgumentException("Опыта не может быть меньше нуля");
            }
            if (gold < 0) {
                throw new IllegalArgumentException("Золота не может быть меньше нуля");
            }

            return new Hero(this);
        }

    }

    @Override
    public Hit rollDamage() {
        Hit hit = weapon.rollDamage();
        return new Hit(hit.damage() + (level - 1) * DAMAGE_PER_LEVEL, hit.critical());
    }

    @Override
    public Damage receiveHit(int amount) {
        Damage damage = new Damage(amount);
        damage.reduceTo(Math.max(1, (int) Math.round(amount * (1 - armor.getDefense()))));
        health = Math.max(0, health - damage.getAmount());
        return damage;
    }

    @Override
    public boolean isAlive() {
        return health > 0;
    }

    public void rest() {
        health = getMaxHealth();

    }

    public int gainExperience(int amount) {
        experience += amount;
        int levelsGained = 0;
        while (experience >= getExperienceToNextLevel()) {
            experience -= getExperienceToNextLevel();
            level++;
            levelsGained++;
        }

        return levelsGained;
    }

    public void addGold(int amount) {
        gold += amount;
    }

    public void spendGold(int amount) {
        if (amount > gold) {
            throw new IllegalStateException(
                String.format("Не хватает золота: нужно %d, есть %d", amount, gold)
            );
        }

        gold -= amount;
    }

    public void addItem(Item item) {
        inventory.add(item);
    }

    public void removeItem(Item item) {
        if (!inventory.remove(item)) {
            throw new IllegalArgumentException("В инвентаре нет предмета " + item.getName());
        }
    }

    public void equip(Item item) {
        if (item instanceof Weapon newWeapon) {
            removeItem(newWeapon);
            inventory.add(weapon);
            weapon = newWeapon;
        } else if (item instanceof Armor newArmor) {
            removeItem(newArmor);
            inventory.add(armor);
            armor = newArmor;

        } else {
            throw new IllegalArgumentException("Предмет нельзя экипировать: " + item.getName());
        }
    }

    public boolean unlock(Achievement achievement) {
        if (achievements.contains(achievement)) {
            return false;
        }

        achievements.add(achievement);
        return true;
    }

    @Override
    public String getCode() {
        return heroClass.getCode();
    }

    @Override
    public String getName() {
        return name;
    }

    @Override
    public int getHealth() {
        return health;
    }

    @Override
    public int getMaxHealth() {
        return heroClass.getBaseHealth() + (level - 1) * HEALTH_PER_LEVEL;
    }

    public HeroClass getHeroClass() {
        return heroClass;
    }

    public int getLevel() {
        return level;
    }

    public int getExperience() {
        return experience;
    }

    public int getExperienceToNextLevel() {
        return level * EXPERIENCE_PER_LEVEL;
    }

    public int getGold() {
        return gold;
    }

    public Weapon getWeapon() {
        return weapon;
    }

    public Armor getArmor() {
        return armor;
    }

    public List<Item> getInventory() {
        return Collections.unmodifiableList(inventory);
    }

    public List<Achievement> getAchievements() {
        return Collections.unmodifiableList(achievements);
    }

}
```

В `Location` заменяем вызов `restoreHealth()` на `rest()` и открываем подклассам доступ к каталогу через `getCatalog()`,
это нам скоро понадобится:

```java
package org.misis.tp.lab3.world;

import org.misis.tp.lab3.combat.Battle;
import org.misis.tp.lab3.combat.BattleReport;
import org.misis.tp.lab3.combat.Outcome;
import org.misis.tp.lab3.core.Dice;
import org.misis.tp.lab3.enemy.Enemy;
import org.misis.tp.lab3.hero.Hero;
import org.misis.tp.lab3.item.Item;
import org.misis.tp.lab3.item.ItemCatalog;

import java.util.List;

public abstract class Location {

    private static final double LOOT_CHANCE = 0.6;
    private static final double ENCHANT_CHANCE = 0.3;

    private final String code;
    private final String displayName;
    private final int recommendedLevel;
    private final ItemCatalog catalog;

    protected Location(String code, String displayName, int recommendedLevel, ItemCatalog catalog) {
        this.code = code;
        this.displayName = displayName;
        this.recommendedLevel = recommendedLevel;
        this.catalog = catalog;
    }

    // создание врага делегируется подклассам для повышения гибкости
    protected abstract Enemy createEnemy();

    // коды предметов из каталога, которые можно найти в локации
    protected abstract List<String> getLootTable();

    public ExpeditionReport explore(Hero hero) {
        return explore(hero, Dice.getInstance().nextSeed());
    }

    // бизнес-логика, общая для всех локаций:
    // герой встречает врага, сражается с ним и получает награду, если победил
    public ExpeditionReport explore(Hero hero, long seed) {
        Dice dice = Dice.getInstance();
        dice.reseed(seed);

        hero.rest();
        Enemy enemy = createEnemy();
        BattleReport battle = new Battle(hero, enemy).run();

        Rewards rewards = battle.outcome() == Outcome.VICTORY
            ? reward(hero, enemy, dice)
            : punish(hero);

        return new ExpeditionReport(seed, code, battle, rewards);
    }

    private Rewards reward(Hero hero, Enemy enemy, Dice dice) {
        int experience = enemy.getExperienceReward();
        int gold = enemy.rollGoldReward();
        Item loot = rollLoot(dice);

        int levelsGained = hero.gainExperience(experience);
        hero.addGold(gold);
        if (loot != null) {
            hero.addItem(loot);
        }

        return new Rewards(experience, gold, loot, levelsGained);
    }

    private Item rollLoot(Dice dice) {
        if (!dice.chance(LOOT_CHANCE)) {
            return null;
        }

        // добыча - это копия шаблона из каталога, ее можно менять, не боясь испортить шаблон
        Item loot = catalog.create(dice.pick(getLootTable()));
        if (dice.chance(ENCHANT_CHANCE)) {
            loot.enchant(dice.pick(loot.getPossibleEnchantments()));
        }

        return loot;
    }

    // проигравший герой теряет половину золота
    private Rewards punish(Hero hero) {
        int lostGold = hero.getGold() / 2;
        hero.spendGold(lostGold);
        return new Rewards(0, -lostGold, null, 0);
    }

    protected ItemCatalog getCatalog() {
        return catalog;
    }

    public String getCode() {
        return code;
    }

    public String getDisplayName() {
        return displayName;
    }

    public int getRecommendedLevel() {
        return recommendedLevel;
    }

}
```

Чтобы иметь возможность восстановить зачарование по идентификатору (нам это понадобится дальше), добавим в него поиск по этому самому
идентификатору:

```java
package org.misis.tp.lab3.item;

import org.misis.tp.lab3.core.UnknownCodeException;

import java.util.List;

public record Enchantment(String code, String name, int bonusDamage, double bonusDefense, int bonusPrice) {

    public static final Enchantment SHARP = new Enchantment("sharp", "Острое", 3, 0, 20);
    public static final Enchantment FIERY = new Enchantment("fiery", "Огненное", 6, 0, 50);
    public static final Enchantment STURDY = new Enchantment("sturdy", "Прочное", 0, 0.05, 20);
    public static final Enchantment WARDING = new Enchantment("warding", "Охранное", 0, 0.10, 50);

    public static final List<Enchantment> FOR_WEAPONS = List.of(SHARP, FIERY);
    public static final List<Enchantment> FOR_ARMOR = List.of(STURDY, WARDING);

    public static Enchantment byCode(String code) {
        for (Enchantment enchantment : List.of(SHARP, FIERY, STURDY, WARDING)) {
            if (enchantment.code().equals(code)) {
                return enchantment;
            }
        }

        throw new UnknownCodeException("Зачарование", code);
    }

}
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

```java
package org.misis.tp.lab3.enemy;

import org.misis.tp.lab3.combat.Hit;
import org.misis.tp.lab3.core.Dice;
import org.misis.tp.lab3.defense.Damage;
import org.misis.tp.lab3.item.Weapon;

public class HauntedWeapon implements Enemy {

    // наше одержимое оружие должно само придумать, сколько у него будет здоровья (у Weapon этой логики нет)
    // поэтому считаем, что за каждую единицу урона полагается 5 здоровья
    private static final int HEALTH_PER_DAMAGE = 5;

    private final Weapon weapon;
    private final int maxHealth;
    private int health;

    public HauntedWeapon(Weapon weapon) {
        this.weapon = weapon;
        // проставляем здоровье по формуле
        this.maxHealth = weapon.getDamage() * HEALTH_PER_DAMAGE;
        this.health = maxHealth;
    }

    @Override
    public String getCode() {
        return "haunted-weapon";
    }

    // название врага берется из названия оружия
    @Override
    public String getName() {
        return "Одержимый " + weapon.getName();
    }

    // метод нанесения урона полностью переиспользует логику оружия, тут основная ценность адаптера
    @Override
    public Hit rollDamage() {
        return weapon.rollDamage();
    }

    // а вот тут ценность адаптера не такая явная, методы ниже мы реализуем сами почти с нуля
    @Override
    public Damage receiveHit(int amount) {
        Damage damage = new Damage(amount);
        health = Math.max(0, health - damage.getAmount());
        return damage;
    }

    @Override
    public boolean isAlive() {
        return health > 0;
    }

    // награду за победу над одержимым оружием привяжем к его стоимости
    @Override
    public int getExperienceReward() {
        return weapon.getPrice() / 2;
    }

    @Override
    public int rollGoldReward() {
        return Dice.getInstance().roll(weapon.getPrice() / 4, weapon.getPrice() / 2);
    }

    @Override
    public int getHealth() {
        return health;
    }

    @Override
    public int getMaxHealth() {
        return maxHealth;
    }

}
```

А теперь объявим и саму локацию — проклятый особняк. В ней будет одно из трех случайных оружий, которые будут взяты из каталога.
Выбранное оружие будет обернуто в адаптер, то есть станет одержимо злым духом, после чего безжалостно нападет на нашего героя.

Благодаря адаптеру мы можем превратить во врага любое оружие, что безусловно разнообразит визиты в это замечательное жилище:

```java
package org.misis.tp.lab3.world;

import org.misis.tp.lab3.core.Dice;
import org.misis.tp.lab3.enemy.Enemy;
import org.misis.tp.lab3.enemy.HauntedWeapon;
import org.misis.tp.lab3.item.DefaultCatalog;
import org.misis.tp.lab3.item.ItemCatalog;
import org.misis.tp.lab3.item.Weapon;

import java.util.List;

public class HauntedManor extends Location {

    public static final String CODE = "haunted-manor";

    // список оружий, которые могут стать одержимыми
    private static final List<String> HAUNTED_WEAPONS = List.of(
        DefaultCatalog.IRON_SWORD, DefaultCatalog.ELVEN_BOW, DefaultCatalog.EMBER_STAFF
    );

    public HauntedManor(ItemCatalog catalog) {
        super(CODE, "Проклятый особняк", 2, catalog);
    }

    // в фабричном методе выбираем одно из оружий и оборачиваем его в адаптер – оно становится одержимым
    @Override
    protected Enemy createEnemy() {
        Weapon weapon = getCatalog().createWeapon(Dice.getInstance().pick(HAUNTED_WEAPONS));
        return new HauntedWeapon(weapon);
    }

    @Override
    protected List<String> getLootTable() {
        return List.of(DefaultCatalog.IRON_SWORD, DefaultCatalog.ELVEN_BOW, DefaultCatalog.EMBER_STAFF, DefaultCatalog.PLATE_ARMOR);
    }

}
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
атакующий два раза подряд).

И причем при совсем уж плохом везении, эти модификаторы еще и должны комбинироваться! То есть в худшем случае на первом же входе в лес перед
нами предстанет
Легендарный бронированный неистовый гоблин и разнесет нашего героя без единого шанса.

Но вместе с риском будет расти и награда, конечно, за таких врагов будет больше опыта и золота.

В целом идея отличная, но тут есть загвоздка. Врагов у нас недавно было 3 вида, сейчас стало 4. В будущем может быть еще больше. Делать
каждому из них подклассы
с модификаторами (неистовый гоблин, неистовый дракон, и так далее) это еще половина беды, настоящая проблема в том, что мы их должны еще и
комбинировать. Для
текущих требований у нас получится 4 (число классов врагов) * 3 (вариантов первого модификатора) * 2 (вариантов второго модификатора) * 1 (
оставшийся модификатор) = 24 комбинации классов врагов с разными модификаторами.

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
реализации паттерна. В нем через композицию определяется, кого заворачивает декоратор (поле `wrapped`), все методы базового декоратора
делегируют
вызовы объекту `wrapped`.

Важно, что `wrapped` это абстракция – `Enemy` и сам базовый декоратор тоже реализует эту абстракцию. Таким образом на месте `wrapped` может
быть и другой декоратор.
Но в конце цепочки обязательно будет конкретный враг.

Итак, код получается вот такой:

```java
package org.misis.tp.lab3.enemy;

import org.misis.tp.lab3.combat.Hit;
import org.misis.tp.lab3.defense.Damage;

public abstract class EnemyDecorator implements Enemy {

    private final Enemy wrapped;

    protected EnemyDecorator(Enemy wrapped) {
        this.wrapped = wrapped;
    }

    @Override
    public String getCode() {
        return wrapped.getCode();
    }

    @Override
    public String getName() {
        return wrapped.getName();
    }

    @Override
    public int getHealth() {
        return wrapped.getHealth();
    }

    @Override
    public int getMaxHealth() {
        return wrapped.getMaxHealth();
    }

    @Override
    public boolean isAlive() {
        return wrapped.isAlive();
    }

    @Override
    public Hit rollDamage() {
        return wrapped.rollDamage();
    }

    @Override
    public Damage receiveHit(int damage) {
        return wrapped.receiveHit(damage);
    }

    @Override
    public int getExperienceReward() {
        return wrapped.getExperienceReward();
    }

    @Override
    public int rollGoldReward() {
        return wrapped.rollGoldReward();
    }

}
```

А дальше уже реализуем конкретных декораторов, наследников базового. Они будут соответствовать нашим модификатором, начнем с легендарного:

```java
package org.misis.tp.lab3.enemy;

import org.misis.tp.lab3.combat.Hit;

public class LegendaryEnemy extends EnemyDecorator {

    private static final int MULTIPLIER = 2;

    public LegendaryEnemy(Enemy wrapped) {
        super(wrapped);
    }

    // добавляем в имя оборачиваемого объекта префикс `Легендарный`
    @Override
    public String getName() {
        return "Легендарный " + super.getName();
    }

    // удваиваем урон оборачиваемого объекта
    @Override
    public Hit rollDamage() {
        Hit hit = super.rollDamage();
        return new Hit(hit.damage() * MULTIPLIER, hit.critical());
    }

    // удваиваем награду за легендарного врага
    @Override
    public int getExperienceReward() {
        return super.getExperienceReward() * MULTIPLIER;
    }

    @Override
    public int rollGoldReward() {
        return super.rollGoldReward() * MULTIPLIER;
    }

}
```

Тут важно заметить, что легендарный декоратор во всех своих методах вызывает методы своего суперкласса – базового декоратора. При
нанесении двойного урона он берет за основу именно `super.rollDamage()`. Именно это и позволяет комбинировать декораторов. Мы уже видели,
что
базовый декоратор просто вызывает методы обертки, значит `super.rollDamage()` это `wrapped.rollDamage()` и если `wrapped` это враг в еще
одном декораторе –
дальше будет вызвано уже модифицированное им поведение. И так по цепочке пока мы не дойдем до одного из наших конкретных врагов.

Супер, базовый принцип понятен, продолжим реализовывать модификаторы, дальше у нас неистовство ветра:

```java
package org.misis.tp.lab3.enemy;

import org.misis.tp.lab3.combat.Hit;

public class WindfuryEnemy extends EnemyDecorator {

    private static final int REWARD_BONUS_PERCENT = 50;

    public WindfuryEnemy(Enemy wrapped) {
        super(wrapped);
    }

    @Override
    public String getName() {
        return "Неистовый " + super.getName();
    }

    @Override
    public Hit rollDamage() {
        Hit first = super.rollDamage();
        Hit second = super.rollDamage();
        return new Hit(first.damage() + second.damage(), first.critical() || second.critical());
    }

    @Override
    public int getExperienceReward() {
        return withBonus(super.getExperienceReward());
    }

    @Override
    public int rollGoldReward() {
        return withBonus(super.rollGoldReward());
    }

    private int withBonus(int reward) {
        return reward + reward * REWARD_BONUS_PERCENT / 100;
    }

}
```

Логика тут точно такая же! Модифицируем поведение, делегируем в `super`.

Третий и пока финальный декоратор уменьшает входящий урон вдвое, затем передаёт получившийся урон дальше по цепочке декораторов:

```java
package org.misis.tp.lab3.enemy;

import org.misis.tp.lab3.defense.Damage;

public class ArmoredEnemy extends EnemyDecorator {

    public ArmoredEnemy(Enemy wrapped) {
        super(wrapped);
    }

    @Override
    public String getName() {
        return "Бронированный " + super.getName();
    }

    @Override
    public Damage receiveHit(int amount) {
        if (amount < 0) {
            throw new IllegalArgumentException("Урон не может быть отрицательным");
        }
        if (amount == 0) {
            return new Damage(0);
        }

        return super.receiveHit(amount - amount / 2);
    }

}
```

Отлично, все требуемые модификаторы объявлены.
Теперь мы можем добавлять их любому врагу и в любой последовательности.
При этом декоратор реализует тот же интерфейс `Enemy`, что и обычный враг, поэтому ни бой, ни локация не отличают элитного врага
от обычного, и работают с ним точно так же. Прелести `Dependency Inversion Principle`!

Осталось прописать логику, как и когда враг становится элитным. Это часть общей логики экспедиции, поэтому добавим её в локацию:
враг, созданный фабричным методом, с некоторым шансом получает комбинацию модификаторов.

```java
package org.misis.tp.lab3.world;

import org.misis.tp.lab3.combat.Battle;
import org.misis.tp.lab3.combat.BattleReport;
import org.misis.tp.lab3.combat.Outcome;
import org.misis.tp.lab3.core.Dice;
import org.misis.tp.lab3.enemy.ArmoredEnemy;
import org.misis.tp.lab3.enemy.Enemy;
import org.misis.tp.lab3.enemy.LegendaryEnemy;
import org.misis.tp.lab3.enemy.WindfuryEnemy;
import org.misis.tp.lab3.hero.Hero;
import org.misis.tp.lab3.item.Item;
import org.misis.tp.lab3.item.ItemCatalog;

import java.util.List;

public abstract class Location {

    private static final double LOOT_CHANCE = 0.6;
    private static final double ENCHANT_CHANCE = 0.3;
    private static final double ELITE_CHANCE = 0.2;
    private static final double DOUBLE_ELITE_CHANCE = 0.25;
    private static final double ARMORED_CHANCE = 0.3;

    private final String code;
    private final String displayName;
    private final int recommendedLevel;
    private final ItemCatalog catalog;

    protected Location(String code, String displayName, int recommendedLevel, ItemCatalog catalog) {
        this.code = code;
        this.displayName = displayName;
        this.recommendedLevel = recommendedLevel;
        this.catalog = catalog;
    }

    protected abstract Enemy createEnemy();

    protected abstract List<String> getLootTable();

    public ExpeditionReport explore(Hero hero) {
        return explore(hero, Dice.getInstance().nextSeed());
    }

    public ExpeditionReport explore(Hero hero, long seed) {
        Dice dice = Dice.getInstance();
        dice.reseed(seed);

        hero.rest();
        Enemy enemy = createEnemy();
        if (dice.chance(ELITE_CHANCE)) {
            // усиления врага через декораторы
            enemy = makeElite(enemy, dice);
        }
        BattleReport battle = new Battle(hero, enemy).run();

        Rewards rewards = battle.outcome() == Outcome.VICTORY
            ? reward(hero, enemy, dice)
            : punish(hero);

        return new ExpeditionReport(seed, code, battle, rewards);
    }

    // с помощью декораторов накладываем на врагов модификаторы
    private Enemy makeElite(Enemy enemy, Dice dice) {
        boolean both = dice.chance(DOUBLE_ELITE_CHANCE);
        if (dice.chance(0.5)) {
            enemy = new LegendaryEnemy(enemy);
            if (both) {
                enemy = new WindfuryEnemy(enemy);
            }
        } else {
            enemy = new WindfuryEnemy(enemy);
            if (both) {
                enemy = new LegendaryEnemy(enemy);
            }
        }

        return dice.chance(ARMORED_CHANCE) ? new ArmoredEnemy(enemy) : enemy;
    }

    private Rewards reward(Hero hero, Enemy enemy, Dice dice) {
        int experience = enemy.getExperienceReward();
        int gold = enemy.rollGoldReward();
        Item loot = rollLoot(dice);

        int levelsGained = hero.gainExperience(experience);
        hero.addGold(gold);
        if (loot != null) {
            hero.addItem(loot);
        }

        return new Rewards(experience, gold, loot, levelsGained);
    }

    private Item rollLoot(Dice dice) {
        if (!dice.chance(LOOT_CHANCE)) {
            return null;
        }

        Item loot = catalog.create(dice.pick(getLootTable()));
        if (dice.chance(ENCHANT_CHANCE)) {
            loot.enchant(dice.pick(loot.getPossibleEnchantments()));
        }

        return loot;
    }

    private Rewards punish(Hero hero) {
        int lostGold = hero.getGold() / 2;
        hero.spendGold(lostGold);
        return new Rewards(0, -lostGold, null, 0);
    }

    protected ItemCatalog getCatalog() {
        return catalog;
    }

    public String getCode() {
        return code;
    }

    public String getDisplayName() {
        return displayName;
    }

    public int getRecommendedLevel() {
        return recommendedLevel;
    }

}
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

```java
package org.misis.tp.lab3.storage;

import org.misis.tp.lab3.hero.Hero;

import java.util.List;
import java.util.Optional;

public interface HeroRepository {

    Optional<Hero> findByName(String name);

    List<Hero> findAll();

    void save(Hero hero);

}
```

Для хранения сейвов будем использовать простой файл, это не бд, но подход тоже вполне рабочий. В качестве формата сохранения выберем
json! [JSON](https://www.json.org/json-ru.html) — текстовый формат, где объекты состоят из пар "имя — значение",
а массивы хранят списки значений. Это довольно вездесущий формат, и нам он как нельзя кстати подходит.

Для чтения и записи JSON подключим библиотеку [Gson](https://google.github.io/gson/UserGuide.html).
Для подключения найдем файл с зависимостями, называется он `pom.xml` и находится в корне Java-проекта, рядом с папкой `src`.

Добавьте туда вот такую конфигурацию зависимостей:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.misis.tp</groupId>
    <artifactId>tp-lab-work</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.google.code.gson</groupId>
            <artifactId>gson</artifactId>
            <version>2.11.0</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>exec-maven-plugin</artifactId>
                <version>3.5.0</version>
            </plugin>
        </plugins>
    </build>

</project>
```

В IntelliJ IDEA обновите Maven-проект кнопкой **Load Maven Changes** или **Reload All Maven Projects** в окне Maven.
Либо он обновится сам при первом запуске приложения.

Супер, библиотеку подключили (наша первая библиотека, кстати, в следующих лабах будет больше), дальше напишем саму реализацию, которая будет
хранить сохранения в обычном текстовом файлике в формате JSON.

В нашем файле с сохранениями будет массив героев со всеми нужными для восстановления объектов данными.

В `FileHeroRepository` структуры `HeroSave` и `ItemSave` описывают данные сохранения, Gson преобразует их в JSON и обратно. Это приватные
классы,
потому что за пределами этой реализации репозитория они в целом не нужны.

Реализуем наконец сам репозиторий:

```java
package org.misis.tp.lab3.storage;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;
import com.google.gson.Strictness;

import org.misis.tp.lab3.hero.Hero;
import org.misis.tp.lab3.hero.HeroClass;
import org.misis.tp.lab3.item.Armor;
import org.misis.tp.lab3.item.Enchantment;
import org.misis.tp.lab3.item.Item;
import org.misis.tp.lab3.item.ItemCatalog;
import org.misis.tp.lab3.item.Weapon;
import org.misis.tp.lab3.progress.Achievement;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

public class FileHeroRepository implements HeroRepository {

    private final Gson gson = new GsonBuilder()
        .setPrettyPrinting()
        .setStrictness(Strictness.STRICT)
        .create();
    private final Path file;
    private final ItemCatalog catalog;

    public FileHeroRepository(Path file, ItemCatalog catalog) {
        this.file = file;
        this.catalog = catalog;
    }

    @Override
    public Optional<Hero> findByName(String name) {
        return findAll().stream().filter(hero -> hero.getName().equals(name)).findFirst();
    }

    @Override
    public List<Hero> findAll() {
        if (!Files.exists(file)) {
            return List.of();
        }
        String json = read();
        if (json.isBlank()) {
            return List.of();
        }
        HeroSave[] saves = gson.fromJson(json, HeroSave[].class);
        if (saves == null) {
            throw new IllegalArgumentException("Ожидался массив героев: " + file.toAbsolutePath());
        }
        List<Hero> heroes = new ArrayList<>();
        for (HeroSave save : saves) {
            heroes.add(restore(save));
        }
        return heroes;
    }

    @Override
    public void save(Hero hero) {
        List<HeroSave> saves = new ArrayList<>();
        for (Hero saved : findAll()) {
            if (!saved.getName().equals(hero.getName())) {
                saves.add(snapshot(saved));
            }
        }
        saves.add(snapshot(hero));
        write(gson.toJson(saves) + "\n");
    }

    private HeroSave snapshot(Hero hero) {
        return new HeroSave(
            hero.getName(), hero.getHeroClass().getCode(), hero.getLevel(),
            hero.getExperience(), hero.getGold(),
            snapshotItem(hero.getWeapon()), snapshotItem(hero.getArmor()),
            hero.getInventory().stream().map(this::snapshotItem).toList(),
            hero.getAchievements().stream().map(Achievement::getCode).toList()
        );
    }

    private ItemSave snapshotItem(Item item) {
        return new ItemSave(item.getCode(), item.getEnchantments().stream().map(Enchantment::code).toList());
    }

    private Hero restore(HeroSave save) {
        return new Hero.Builder()
            .setName(save.name())
            .setHeroClass(HeroClass.byCode(save.heroClass()))
            .setLevel(save.level())
            .setExperience(save.experience())
            .setGold(save.gold())
            .setWeapon((Weapon) restoreItem(save.weapon()))
            .setArmor((Armor) restoreItem(save.armor()))
            .setInventory(save.inventory().stream().map(this::restoreItem).toList())
            .setAchievements(save.achievements().stream().map(Achievement::byCode).toList())
            .build();
    }

    private Item restoreItem(ItemSave save) {
        Item item = catalog.create(save.code());
        for (String code : save.enchantments()) {
            item.enchant(Enchantment.byCode(code));
        }
        return item;
    }

    private record HeroSave(
        String name, String heroClass, int level, int experience, int gold,
        ItemSave weapon, ItemSave armor, List<ItemSave> inventory, List<String> achievements
    ) {
    }

    private record ItemSave(String code, List<String> enchantments) {
    }

    private String read() {
        try {
            return Files.readString(file, StandardCharsets.UTF_8);
        } catch (IOException e) {
            throw new UncheckedIOException("Не удалось прочитать сохранение: " + file.toAbsolutePath(), e);
        }
    }

    private void write(String text) {
        try {
            Path parent = file.toAbsolutePath().getParent();
            Files.createDirectories(parent);
            Files.writeString(file, text, StandardCharsets.UTF_8);
        } catch (IOException e) {
            throw new UncheckedIOException("Не удалось записать сохранение: " + file.toAbsolutePath(), e);
        }
    }

}
```

Если зависимости `Gson` подсвечиваются красным, попробуйте выполнить **Reload** в окошке Maven, либо перезапустите проект.

Реализация получилась рабочая, но не очень эффективная. На каждый запрос к герою файл читается целиком, а герои собираются
заново. Мы тратим драгоценное время игры на то, чтобы открыть файл, распарсить JSON, собрать объекты, это все в целом быстро, но кажется
что можно сделать чуть эффективнее.

Давайте добавим кэширование! Чтобы работа с файлом происходила только при первом запросе, а в дальнейшем герой
брался из кэша, то есть из памяти класса. Мы же можем просто хранить информацию о сохраненных героях в памяти? Можем!

Если говорить простым языком, кэширование, это когда мы временно кладем данные из медленного хранилища в быстрое, чтобы операции чтения с
ними были более
производительны. В нашем примере медленное хранилище – файл, быстрое – объекты в памяти языка. Файл дает нам хранение между перезапусками,
но работать с ним медленно.
Объекты в памяти пропадают между перезапусками, но обращение к ним мгновенное. Поэтому идейно мы хотим один раз прочитать текущие сохранения
из файла,
отправить их в специальный объект (кэш), и потом уже читать оттуда. Но при этом если появляется новое сохранение, мы обязаны записать его в
оба места сразу (иначе кэш разойдется с основным хранилищем).

Кэширующая логика пригодится нам не только для текущей реализации репозитория, но и для других, если они появятся.
Плюс добавлять ее напрямую в реализацию не очень красиво, смешивается логика, нарушается `SRP`.

Чтобы этого не делать, воспользуемся паттерном *Заместитель*. Он позволяет подставлять вместо настоящего объекта другой объект с тем же
интерфейсом,
который перехватывает обращения и управляет доступом к настоящему объекту.

Реализуем универсального кэширующего заместителя для репозитория сохранений. Он будет оборачивать исходный объект, и в каждый метод
перед реальным чтением сохранений проверять кэш:

```java
package org.misis.tp.lab3.storage;

import org.misis.tp.lab3.hero.Hero;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;

public class CachingHeroRepository implements HeroRepository {

    private final HeroRepository target;
    private Map<String, Hero> cache;

    public CachingHeroRepository(HeroRepository target) {
        this.target = target;
    }

    @Override
    public Optional<Hero> findByName(String name) {
        return Optional.ofNullable(cached().get(name));
    }

    @Override
    public List<Hero> findAll() {
        return List.copyOf(cached().values());
    }

    @Override
    public void save(Hero hero) {
        target.save(hero);
        cached().put(hero.getName(), hero);
    }

    private Map<String, Hero> cached() {
        if (Objects.isNull(cache)) {
            Map<String, Hero> loaded = new LinkedHashMap<>();
            for (Hero hero : target.findAll()) {
                loaded.put(hero.getName(), hero);
            }
            cache = loaded;
        }

        return cache;
    }

}
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

```java
package org.misis.tp.lab3.tactic;

public enum Action {

    ATTACK(1.0, 1.0),
    POWER_STRIKE(1.5, 1.5),
    DEFEND(0.5, 0.3);

    private final double attackMultiplier;
    private final double incomingMultiplier;

    Action(double attackMultiplier, double incomingMultiplier) {
        this.attackMultiplier = attackMultiplier;
        this.incomingMultiplier = incomingMultiplier;
    }

    public double getAttackMultiplier() {
        return attackMultiplier;
    }

    public double getIncomingMultiplier() {
        return incomingMultiplier;
    }

}
```

Получается, что он может вести себя агрессивно (увеличивается и исходящий, и входящий урон), сбалансированно (ничего не меняется) и
осторожно, уходя в оборону (уменьшается и входящий, и исходящий урон).

Выбирать действия мы будем в зависимости от состояния врага и героя, определим базовый интерфейс тактики:

```java
package org.misis.tp.lab3.tactic;

import org.misis.tp.lab3.combat.Combatant;

public interface Tactic {

    String getCode();

    String getDisplayName();

    String getDescription();

    Action chooseAction(int round, Combatant hero, Combatant enemy);

}
```

Реализуем самый простой алгоритм выбора действия - обычный удар каждый раунд:

```java
package org.misis.tp.lab3.tactic;

import org.misis.tp.lab3.combat.Combatant;

public class BalancedTactic implements Tactic {

    @Override
    public String getCode() {
        return "balanced";
    }

    @Override
    public String getDisplayName() {
        return "Баланс";
    }

    @Override
    public String getDescription() {
        return "обычный удар каждый раунд";
    }

    @Override
    public Action chooseAction(int round, Combatant hero, Combatant enemy) {
        return Action.ATTACK;
    }

}
```

Натиск - пока здоровье врага выше, чем у героя, герой бьет безрассудно, нанося и получая в ответ больше урона:

```java
package org.misis.tp.lab3.tactic;

import org.misis.tp.lab3.combat.Combatant;

public class AggressiveTactic implements Tactic {

    @Override
    public String getCode() {
        return "aggressive";
    }

    @Override
    public String getDisplayName() {
        return "Натиск";
    }

    @Override
    public String getDescription() {
        return "совершать безрассудные удары, пока у врага больше здоровья, чем у героя";
    }

    @Override
    public Action chooseAction(int round, Combatant hero, Combatant enemy) {
        return enemy.getHealth() > hero.getHealth() ? Action.POWER_STRIKE : Action.ATTACK;
    }

}
```

И осторожность - раненый герой через раунд будет уходить в оборону:

```java
package org.misis.tp.lab3.tactic;

import org.misis.tp.lab3.combat.Combatant;

public class CautiousTactic implements Tactic {

    private static final double LOW_HEALTH = 0.4;

    @Override
    public String getCode() {
        return "cautious";
    }

    @Override
    public String getDisplayName() {
        return "Осторожность";
    }

    @Override
    public String getDescription() {
        return "при здоровье ниже 40% через раунд уходить в оборону";
    }

    @Override
    public Action chooseAction(int round, Combatant hero, Combatant enemy) {
        boolean wounded = hero.getHealth() < hero.getMaxHealth() * LOW_HEALTH;
        return wounded && round % 2 == 0 ? Action.DEFEND : Action.ATTACK;
    }

}
```

Также дополним типы ивентов в нашей игре, добавим наши действия, и `EVADE`/`ABSORB`, они нам пригодятся дальше:

```java
package org.misis.tp.lab3.combat;

public enum EventType {

    HIT,
    CRITICAL_HIT,

    EVADE,
    ABSORB,
    POWER_STRIKE,
    DEFEND,
    DEATH

}
```

Теперь обновим сам `Battle`. Тактику он получает через конструктор и каждый раунд спрашивает у нее, что именно делает герой:

```java
package org.misis.tp.lab3.combat;

import org.misis.tp.lab3.defense.Damage;
import org.misis.tp.lab3.tactic.Action;
import org.misis.tp.lab3.tactic.Tactic;

import java.util.ArrayList;
import java.util.List;

public class Battle {

    private final Combatant hero;
    private final Combatant enemy;
    private final Tactic tactic;
    private final List<BattleEvent> events = new ArrayList<>();
    private int round;

    // стратегия прокидывается в конструктор и сохраняется в поле, потому что применяется на всю продолжительность боя
    public Battle(Combatant hero, Combatant enemy, Tactic tactic) {
        this.hero = hero;
        this.enemy = enemy;
        this.tactic = tactic;
    }

    public BattleReport run() {
        while (hero.isAlive() && enemy.isAlive()) {
            round++;
            // использование стратегии – действие зависит от того, какой алгоритм выбрал пользователь
            Action action = tactic.chooseAction(round, hero, enemy);
            announce(action);

            strike(hero, Side.HERO, enemy, Side.ENEMY, action.getAttackMultiplier());
            if (enemy.isAlive()) {
                strike(enemy, Side.ENEMY, hero, Side.HERO, action.getIncomingMultiplier());
            }
        }

        Outcome outcome = hero.isAlive() ? Outcome.VICTORY : Outcome.DEFEAT;
        return new BattleReport(Participant.of(hero), Participant.of(enemy), outcome, List.copyOf(events));
    }

    private void announce(Action action) {
        if (action == Action.POWER_STRIKE) {
            emit(new BattleEvent(round, EventType.POWER_STRIKE, Side.HERO, Side.ENEMY, 0, 0));
        } else if (action == Action.DEFEND) {
            emit(new BattleEvent(round, EventType.DEFEND, Side.HERO, Side.HERO, 0, 0));
        }
    }

    private void strike(Combatant attacker, Side source, Combatant defender, Side target, double multiplier) {
        Hit hit = attacker.rollDamage();
        int amount = Math.max(1, (int) Math.round(hit.damage() * multiplier));
        Damage damage = defender.receiveHit(amount);

        if (damage.isEvaded()) {
            emit(new BattleEvent(round, EventType.EVADE, source, target, 0, defender.getHealth()));
            return;
        }
        if (damage.getAbsorbed() > 0) {
            emit(new BattleEvent(round, EventType.ABSORB, source, target, damage.getAbsorbed(), defender.getHealth()));
        }
        if (damage.getAmount() > 0) {
            EventType type = hit.critical() ? EventType.CRITICAL_HIT : EventType.HIT;
            emit(new BattleEvent(round, type, source, target, damage.getAmount(), defender.getHealth()));
        }

        if (!defender.isAlive()) {
            emit(new BattleEvent(round, EventType.DEATH, source, target, 0, 0));
        }
    }

    private void emit(BattleEvent event) {
        events.add(event);
    }

}
```

Обратите внимание, что бой ничего не знает о конкретных тактиках: он работает с интерфейсом `Tactic`, и
подставить ему можно любую реализацию. Если появится новая тактика, `Battle` менять не придется.

Ну и в отчете об экспедиции будем запоминать, с какой тактикой герой ее начинал:

```java
package org.misis.tp.lab3.world;

import org.misis.tp.lab3.combat.BattleReport;

public record ExpeditionReport(long seed, String locationCode, String tacticCode, BattleReport battle, Rewards rewards) {
}
```

Стратегия позволяет нам соблюдать `Open Closed` принцип, он дает нам возможность расширять поведение нашего класса без его изменения, путем
добавления новых реализаций стратегии. Стратегию можно выбирать как в методе, где она используется, так и в конструкторе, все зависит от
конкретной необходимости. В нашем случае стратегия выбирается в конструкторе, потому что по ходу боя она не меняется.

Применять стратегию стоит, когда одно и то же действие можно выполнить несколькими способами и способ нужно выбирать
или менять во время работы программы.

И давайте сразу обновим `Location`: она будет принимать тактику и передавать дальше в `Battle`.

Получится что-то такое:

```java
package org.misis.tp.lab3.world;

import org.misis.tp.lab3.combat.Battle;
import org.misis.tp.lab3.combat.BattleReport;
import org.misis.tp.lab3.combat.Outcome;
import org.misis.tp.lab3.core.Dice;
import org.misis.tp.lab3.enemy.ArmoredEnemy;
import org.misis.tp.lab3.enemy.Enemy;
import org.misis.tp.lab3.enemy.LegendaryEnemy;
import org.misis.tp.lab3.enemy.WindfuryEnemy;
import org.misis.tp.lab3.hero.Hero;
import org.misis.tp.lab3.item.Item;
import org.misis.tp.lab3.item.ItemCatalog;
import org.misis.tp.lab3.tactic.BalancedTactic;
import org.misis.tp.lab3.tactic.Tactic;

import java.util.List;

public abstract class Location {

    private static final double LOOT_CHANCE = 0.6;
    private static final double ENCHANT_CHANCE = 0.3;
    private static final double ELITE_CHANCE = 0.2;
    private static final double DOUBLE_ELITE_CHANCE = 0.25;
    private static final double ARMORED_CHANCE = 0.3;

    private final String code;
    private final String displayName;
    private final int recommendedLevel;
    private final ItemCatalog catalog;

    protected Location(String code, String displayName, int recommendedLevel, ItemCatalog catalog) {
        this.code = code;
        this.displayName = displayName;
        this.recommendedLevel = recommendedLevel;
        this.catalog = catalog;
    }

    protected abstract Enemy createEnemy();

    protected abstract List<String> getLootTable();

    public ExpeditionReport explore(Hero hero) {
        return explore(hero, new BalancedTactic());
    }

    public ExpeditionReport explore(Hero hero, long seed) {
        return explore(hero, new BalancedTactic(), seed);
    }

    public ExpeditionReport explore(Hero hero, Tactic tactic) {
        return explore(hero, tactic, Dice.getInstance().nextSeed());
    }

    public ExpeditionReport explore(Hero hero, Tactic tactic, long seed) {
        Dice dice = Dice.getInstance();
        dice.reseed(seed);

        hero.rest();
        Enemy enemy = createEnemy();
        if (dice.chance(ELITE_CHANCE)) {
            // усиления врага через декораторы
            enemy = makeElite(enemy, dice);
        }
        BattleReport battle = new Battle(hero, enemy, tactic).run();

        Rewards rewards = battle.outcome() == Outcome.VICTORY
            ? reward(hero, enemy, dice)
            : punish(hero);

        return new ExpeditionReport(seed, code, tactic.getCode(), battle, rewards);
    }

    private Enemy makeElite(Enemy enemy, Dice dice) {
        boolean both = dice.chance(DOUBLE_ELITE_CHANCE);
        if (dice.chance(0.5)) {
            enemy = new LegendaryEnemy(enemy);
            if (both) {
                enemy = new WindfuryEnemy(enemy);
            }
        } else {
            enemy = new WindfuryEnemy(enemy);
            if (both) {
                enemy = new LegendaryEnemy(enemy);
            }
        }

        return dice.chance(ARMORED_CHANCE) ? new ArmoredEnemy(enemy) : enemy;
    }

    private Rewards reward(Hero hero, Enemy enemy, Dice dice) {
        int experience = enemy.getExperienceReward();
        int gold = enemy.rollGoldReward();
        Item loot = rollLoot(dice);

        int levelsGained = hero.gainExperience(experience);
        hero.addGold(gold);
        if (loot != null) {
            hero.addItem(loot);
        }

        return new Rewards(experience, gold, loot, levelsGained);
    }

    private Item rollLoot(Dice dice) {
        if (!dice.chance(LOOT_CHANCE)) {
            return null;
        }

        Item loot = catalog.create(dice.pick(getLootTable()));
        if (dice.chance(ENCHANT_CHANCE)) {
            loot.enchant(dice.pick(loot.getPossibleEnchantments()));
        }

        return loot;
    }

    private Rewards punish(Hero hero) {
        int lostGold = hero.getGold() / 2;
        hero.spendGold(lostGold);
        return new Rewards(0, -lostGold, null, 0);
    }

    protected ItemCatalog getCatalog() {
        return catalog;
    }

    public String getCode() {
        return code;
    }

    public String getDisplayName() {
        return displayName;
    }

    public int getRecommendedLevel() {
        return recommendedLevel;
    }

}
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

```java
package org.misis.tp.lab3.event;

public enum GameEventType {

    VICTORY,
    DEFEAT,
    LEVEL_UP

}
```

И само событие. Обратите внимание, что это просто данные: что произошло, с каким героем и в какой экспедиции:

```java
package org.misis.tp.lab3.event;

import org.misis.tp.lab3.hero.Hero;
import org.misis.tp.lab3.world.ExpeditionReport;

public record GameEvent(GameEventType type, Hero hero, ExpeditionReport expedition) {
}
```

Интерфейс для подписчиков содержит всего один метод, который вызывается при наступлении события, и определяет контракт обработки этого
изменения:

```java
package org.misis.tp.lab3.event;

public interface GameEventListener {

    void onEvent(GameEvent event);

}
```

Именно издатель позволяет подписываться на определенный тип события. Также именно он оповещает подписчиков о
наступлении того или иного события:

```java
package org.misis.tp.lab3.event;

import java.util.ArrayList;
import java.util.EnumMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;

public class GameEventPublisher {

    private final Map<GameEventType, List<GameEventListener>> listeners = new EnumMap<>(GameEventType.class);

    public void subscribe(GameEventType type, GameEventListener listener) {
        listeners.computeIfAbsent(type, ignored -> new ArrayList<>()).add(listener);
    }

    public void unsubscribe(GameEventType type, GameEventListener listener) {
        List<GameEventListener> subscribed = listeners.get(type);
        if (Objects.nonNull(subscribed)) {
            subscribed.remove(listener);
        }
    }

    public void publish(GameEvent event) {
        for (GameEventListener listener : List.copyOf(listeners.getOrDefault(event.type(), List.of()))) {
            listener.onEvent(event);
        }
    }

}
```

Подписчики хранятся в `listeners`, причем хранятся по типу ивента, то есть подписаться и реагировать они могут на отдельные типы, а не на
все.
Реакции вызываются в методе `publish`. Когда в нашей системе происходит ивент – мы вызываем издателя, а он, по списку зарегистрированных
подписчиков, вызывает их
`onEvent` методы, прокидывая туда информацию о ивенте.

Теперь реализуем двух конкретных подписчиков. Сначала того, который будет выдавать достижения:

```java
package org.misis.tp.lab3.progress;

import org.misis.tp.lab3.event.GameEvent;
import org.misis.tp.lab3.event.GameEventListener;
import org.misis.tp.lab3.event.GameEventType;
import org.misis.tp.lab3.hero.Hero;
import org.misis.tp.lab3.item.Item;

import java.util.Objects;

public class AchievementTracker implements GameEventListener {

    private static final int VETERAN_LEVEL = 5;

    @Override
    public void onEvent(GameEvent event) {
        Hero hero = event.hero();
        if (event.type() == GameEventType.VICTORY) {
            hero.unlock(Achievement.FIRST_BLOOD);

            if ("dragon".equals(event.expedition().battle().enemy().code())) {
                hero.unlock(Achievement.DRAGON_SLAYER);
            }

            Item loot = event.expedition().rewards().loot();
            if (Objects.nonNull(loot) && !loot.getEnchantments().isEmpty()) {
                hero.unlock(Achievement.LUCKY);
            }
        } else if (event.type() == GameEventType.LEVEL_UP && hero.getLevel() >= VETERAN_LEVEL) {
            hero.unlock(Achievement.VETERAN);
        }
    }

}
```

А теперь второго, он будет записывать экспедиции в журнал:

```java
package org.misis.tp.lab3.progress;

import org.misis.tp.lab3.event.GameEvent;
import org.misis.tp.lab3.event.GameEventListener;
import org.misis.tp.lab3.world.ExpeditionReport;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class ExpeditionJournal implements GameEventListener {

    private final Map<String, List<ExpeditionReport>> entries = new HashMap<>();

    @Override
    public void onEvent(GameEvent event) {
        entries.computeIfAbsent(event.hero().getName(), ignored -> new ArrayList<>()).add(event.expedition());
    }

    public List<ExpeditionReport> getEntries(String heroName) {
        return List.copyOf(entries.getOrDefault(heroName, List.of()));
    }

}
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

```java
package org.misis.tp.lab3.defense;

import java.util.Objects;

public abstract class DamageHandler {

    private DamageHandler next;

    public DamageHandler setNext(DamageHandler next) {
        this.next = next;
        return next;
    }

    public abstract void handle(Damage damage);

    protected void passToNext(Damage damage) {
        if (Objects.nonNull(next)) {
            next.handle(damage);
        }
    }

}
```

Теперь конкретный обработчик, который будет хендлить уклонение: если оно сработало, удар по герою не прошел, и передавать урон дальше по
цепочке незачем:

```java
package org.misis.tp.lab3.defense;

import org.misis.tp.lab3.core.Dice;

public class EvasionHandler extends DamageHandler {

    private final double chance;

    public EvasionHandler(double chance) {
        this.chance = chance;
    }

    @Override
    public void handle(Damage damage) {
        if (Dice.getInstance().chance(chance)) {
            damage.evade();
            return;
        }

        passToNext(damage);
    }

}
```

Броня гасит долю урона и передает то, что осталось, следующему обработчику. Обратите внимание, что это то же самое правило,
что раньше было зашито в герое, скелете и драконе, а теперь оно записано в одном месте:

```java
package org.misis.tp.lab3.defense;

public class ArmorHandler extends DamageHandler {

    private final double defense;

    public ArmorHandler(double defense) {
        this.defense = defense;
    }

    @Override
    public void handle(Damage damage) {
        int reduced = (int) Math.round(damage.getAmount() * (1 - defense));
        damage.reduceTo(Math.max(1, reduced));
        passToNext(damage);
    }

}
```

И магический барьер, который поглощает урон, пока у него есть запас прочности:

```java
package org.misis.tp.lab3.defense;

public class BarrierHandler extends DamageHandler {

    private int strength;

    public BarrierHandler(int strength) {
        this.strength = strength;
    }

    @Override
    public void handle(Damage damage) {
        int absorbed = Math.min(strength, damage.getAmount());
        if (absorbed > 0) {
            strength -= absorbed;
            damage.absorb(absorbed);
        }

        passToNext(damage);
    }

}
```

Какие звенья будут в цепочке, зависит от героя. Цепочку собирает его класс - так же, как класс сам знает свою
фабрику стартового снаряжения:

```java
package org.misis.tp.lab3.hero;

import org.misis.tp.lab3.core.UnknownCodeException;
import org.misis.tp.lab3.defense.ArmorHandler;
import org.misis.tp.lab3.defense.BarrierHandler;
import org.misis.tp.lab3.defense.DamageHandler;
import org.misis.tp.lab3.defense.EvasionHandler;
import org.misis.tp.lab3.item.Armor;
import org.misis.tp.lab3.item.ItemCatalog;
import org.misis.tp.lab3.kit.MageKitFactory;
import org.misis.tp.lab3.kit.RangerKitFactory;
import org.misis.tp.lab3.kit.StarterKitFactory;
import org.misis.tp.lab3.kit.WarriorKitFactory;

import java.util.function.Function;

public enum HeroClass {

    WARRIOR("warrior", "Воин", 120, WarriorKitFactory::new, 0, 0),
    RANGER("ranger", "Следопыт", 100, RangerKitFactory::new, 0.15, 0),
    MAGE("mage", "Маг", 85, MageKitFactory::new, 0, 30);

    private final String code;
    private final String displayName;
    private final int baseHealth;
    private final Function<ItemCatalog, StarterKitFactory> starterKit;
    private final double evasion;
    private final int barrier;

    HeroClass(
        String code, String displayName, int baseHealth,
        Function<ItemCatalog, StarterKitFactory> starterKit,
        double evasion, int barrier
    ) {
        this.code = code;
        this.displayName = displayName;
        this.baseHealth = baseHealth;
        this.starterKit = starterKit;
        this.evasion = evasion;
        this.barrier = barrier;
    }

    public StarterKitFactory getStarterKit(ItemCatalog catalog) {
        return starterKit.apply(catalog);
    }

    public DamageHandler createDefense(Armor armor, int level) {
        double evasion = this.evasion;
        int barrier = this.barrier;
        if (this == WARRIOR && level >= 3) {
            barrier = 20;
        } else if (this == RANGER && level >= 5) {
            barrier = 15;
        } else if (this == MAGE && level >= 3) {
            evasion = 0.10;
        }

        DamageHandler armorHandler = new ArmorHandler(armor.getDefense());
        DamageHandler first = armorHandler;

        if (evasion > 0) {
            first = new EvasionHandler(evasion);
            first.setNext(armorHandler);
        }
        if (barrier > 0) {
            armorHandler.setNext(new BarrierHandler(barrier));
        }

        return first;
    }

    public static HeroClass byCode(String code) {
        for (HeroClass heroClass : values()) {
            if (heroClass.code.equals(code)) {
                return heroClass;
            }
        }

        throw new UnknownCodeException("Класс героя", code);
    }

    public String getCode() {
        return code;
    }

    public String getDisplayName() {
        return displayName;
    }

    public int getBaseHealth() {
        return baseHealth;
    }

    public double getEvasion() {
        return evasion;
    }

    public int getBarrier() {
        return barrier;
    }

}
```

Обратите внимание, что цепочка строится динамически. Обработчики, которых у класса нет, в цепочку просто не попадают. А еще в
цепочке важен порядок звеньев: если поставить барьер перед броней, он будет поглощать урон до брони и иссякнет быстрее.

С ростом уровня защита дополняется: воин с 3-го уровня получает барьер на 20 единиц, следопыт с 5-го — барьер на 15, а маг с 3-го —
уклонение с вероятностью 10%. У следопыта сохраняется уклонение 15%, у мага — барьер на 30 единиц. Новые способности учитываются при
следующей сборке цепочки, в частности при отдыхе перед экспедицией; урон проходит в порядке "уклонение → броня → барьер".

Теперь сам герой. Урон он больше не считает сам, а передает в первое звено своей цепочки. Перед каждой экспедицией
цепочка собирается заново, поэтому барьер мага восстанавливается, а после смены брони защита сразу учитывает новую
броню:

```java
package org.misis.tp.lab3.hero;

import org.misis.tp.lab3.combat.Combatant;
import org.misis.tp.lab3.combat.Hit;
import org.misis.tp.lab3.defense.Damage;
import org.misis.tp.lab3.defense.DamageHandler;
import org.misis.tp.lab3.item.Armor;
import org.misis.tp.lab3.item.Item;
import org.misis.tp.lab3.item.Weapon;
import org.misis.tp.lab3.kit.StarterKitFactory;
import org.misis.tp.lab3.progress.Achievement;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Objects;

public class Hero implements Combatant {

    private static final int MAX_NAME_LENGTH = 20;
    private static final int STARTING_GOLD = 50;
    private static final int HEALTH_PER_LEVEL = 15;
    private static final int DAMAGE_PER_LEVEL = 3;
    private static final int EXPERIENCE_PER_LEVEL = 100;

    private final String name;
    private final HeroClass heroClass;
    private int level;
    private int experience;
    private int gold;
    private int health;
    private Weapon weapon;
    private Armor armor;
    private final List<Item> inventory;
    // достижения сохраняются вместе с героем
    private final List<Achievement> achievements;
    private DamageHandler defense;

    private Hero(Builder builder) {
        this.name = builder.name;
        this.heroClass = builder.heroClass;
        this.level = builder.level;
        this.experience = builder.experience;
        this.gold = builder.gold;
        this.weapon = builder.weapon;
        this.armor = builder.armor;
        this.inventory = new ArrayList<>(builder.inventory);
        this.achievements = new ArrayList<>(builder.achievements);
        rest();
    }

    public static class Builder {

        private String name;
        private HeroClass heroClass;
        private Weapon weapon;
        private Armor armor;
        private int level = 1;
        private int experience = 0;
        private int gold = STARTING_GOLD;
        private List<Item> inventory = List.of();
        private List<Achievement> achievements = List.of();

        public Builder setName(String name) {
            this.name = name;
            return this;
        }

        public Builder setHeroClass(HeroClass heroClass) {
            this.heroClass = heroClass;
            return this;
        }

        public Builder setWeapon(Weapon weapon) {
            this.weapon = weapon;
            return this;
        }

        public Builder setArmor(Armor armor) {
            this.armor = armor;
            return this;
        }

        public Builder setStarterKit(StarterKitFactory starterKit) {
            this.weapon = starterKit.createWeapon();
            this.armor = starterKit.createArmor();
            return this;
        }

        public Builder setLevel(int level) {
            this.level = level;
            return this;
        }

        public Builder setExperience(int experience) {
            this.experience = experience;
            return this;
        }

        public Builder setGold(int gold) {
            this.gold = gold;
            return this;
        }

        public Builder setInventory(List<Item> inventory) {
            this.inventory = List.copyOf(inventory);
            return this;
        }

        public Builder setAchievements(List<Achievement> achievements) {
            this.achievements = List.copyOf(achievements);
            return this;
        }

        public Hero build() {
            if (Objects.isNull(name) || name.isBlank()) {
                throw new IllegalArgumentException("У героя должно быть имя");
            }
            if (name.length() > MAX_NAME_LENGTH) {
                throw new IllegalArgumentException(
                    String.format("Имя героя не может быть длиннее %d символов", MAX_NAME_LENGTH)
                );
            }
            if (Objects.isNull(heroClass)) {
                throw new IllegalArgumentException("Не выбран класс героя");
            }
            if (Objects.isNull(weapon) || Objects.isNull(armor)) {
                throw new IllegalArgumentException("Герою нужны оружие и броня");
            }
            if (level < 1) {
                throw new IllegalArgumentException("Уровень героя не может быть меньше 1");
            }
            if (experience < 0) {
                throw new IllegalArgumentException("Опыта не может быть меньше нуля");
            }
            if (gold < 0) {
                throw new IllegalArgumentException("Золота не может быть меньше нуля");
            }

            return new Hero(this);
        }

    }

    @Override
    public Hit rollDamage() {
        Hit hit = weapon.rollDamage();
        return new Hit(hit.damage() + (level - 1) * DAMAGE_PER_LEVEL, hit.critical());
    }

    @Override
    public Damage receiveHit(int amount) {
        Damage damage = new Damage(amount);
        // применение цепочки защиты
        defense.handle(damage);
        health = Math.max(0, health - damage.getAmount());
        return damage;
    }

    @Override
    public boolean isAlive() {
        return health > 0;
    }

    public void rest() {
        health = getMaxHealth();
        defense = heroClass.createDefense(armor, level);
    }

    public int gainExperience(int amount) {
        experience += amount;
        int levelsGained = 0;
        while (experience >= getExperienceToNextLevel()) {
            experience -= getExperienceToNextLevel();
            level++;
            levelsGained++;
        }

        return levelsGained;
    }

    public void addGold(int amount) {
        gold += amount;
    }

    public void spendGold(int amount) {
        if (amount > gold) {
            throw new IllegalStateException(
                String.format("Не хватает золота: нужно %d, есть %d", amount, gold)
            );
        }

        gold -= amount;
    }

    public void addItem(Item item) {
        inventory.add(item);
    }

    public void removeItem(Item item) {
        if (!inventory.remove(item)) {
            throw new IllegalArgumentException("В инвентаре нет предмета " + item.getName());
        }
    }

    public void equip(Item item) {
        if (item instanceof Weapon newWeapon) {
            removeItem(newWeapon);
            inventory.add(weapon);
            weapon = newWeapon;
        } else if (item instanceof Armor newArmor) {
            removeItem(newArmor);
            inventory.add(armor);
            armor = newArmor;
            defense = heroClass.createDefense(armor, level);
        } else {
            throw new IllegalArgumentException("Предмет нельзя экипировать: " + item.getName());
        }
    }

    public boolean unlock(Achievement achievement) {
        if (achievements.contains(achievement)) {
            return false;
        }

        achievements.add(achievement);
        return true;
    }

    @Override
    public String getCode() {
        return heroClass.getCode();
    }

    @Override
    public String getName() {
        return name;
    }

    @Override
    public int getHealth() {
        return health;
    }

    @Override
    public int getMaxHealth() {
        return heroClass.getBaseHealth() + (level - 1) * HEALTH_PER_LEVEL;
    }

    public HeroClass getHeroClass() {
        return heroClass;
    }

    public int getLevel() {
        return level;
    }

    public int getExperience() {
        return experience;
    }

    public int getExperienceToNextLevel() {
        return level * EXPERIENCE_PER_LEVEL;
    }

    public int getGold() {
        return gold;
    }

    public Weapon getWeapon() {
        return weapon;
    }

    public Armor getArmor() {
        return armor;
    }

    public List<Item> getInventory() {
        return Collections.unmodifiableList(inventory);
    }

    public List<Achievement> getAchievements() {
        return Collections.unmodifiableList(achievements);
    }

}
```

У врагов цепочка задается прямо в конструкторе. Общий класс врага пропускает урон через цепочку, если она есть:

```java
package org.misis.tp.lab3.enemy;

import org.misis.tp.lab3.combat.Hit;
import org.misis.tp.lab3.core.Dice;
import org.misis.tp.lab3.defense.Damage;
import org.misis.tp.lab3.defense.DamageHandler;

import java.util.Objects;

public abstract class AbstractEnemy implements Enemy {

    private final String code;
    private final String name;
    private final int maxHealth;
    private final int minDamage;
    private final int maxDamage;
    private final int experienceReward;
    private final int goldReward;
    private final DamageHandler defense;
    private int health;

    protected AbstractEnemy(
        String code, String name, int maxHealth,
        int minDamage, int maxDamage,
        int experienceReward, int goldReward,
        DamageHandler defense
    ) {
        this.code = code;
        this.name = name;
        this.maxHealth = maxHealth;
        this.minDamage = minDamage;
        this.maxDamage = maxDamage;
        this.experienceReward = experienceReward;
        this.goldReward = goldReward;
        this.defense = defense;
        this.health = maxHealth;
    }

    @Override
    public Hit rollDamage() {
        return new Hit(Dice.getInstance().roll(minDamage, maxDamage), false);
    }

    @Override
    public Damage receiveHit(int amount) {
        Damage damage = new Damage(amount);
        if (Objects.nonNull(defense)) {
            // применение цепочки защиты
            defense.handle(damage);
        }

        health = Math.max(0, health - damage.getAmount());
        return damage;
    }

    @Override
    public boolean isAlive() {
        return health > 0;
    }

    @Override
    public int rollGoldReward() {
        return Dice.getInstance().roll(goldReward / 2, goldReward);
    }

    @Override
    public int getExperienceReward() {
        return experienceReward;
    }

    @Override
    public String getCode() {
        return code;
    }

    @Override
    public String getName() {
        return name;
    }

    @Override
    public int getHealth() {
        return health;
    }

    @Override
    public int getMaxHealth() {
        return maxHealth;
    }

}
```

У гоблина защиты нет:

```java
package org.misis.tp.lab3.enemy;

public class Goblin extends AbstractEnemy {

    public Goblin() {
        super("goblin", "Гоблин", 50, 6, 14, 40, 30, null);
    }

}
```

Кости скелета и чешуя дракона работают как броня, поэтому метод `reduceDamage` им больше не нужен:

```java
package org.misis.tp.lab3.enemy;

import org.misis.tp.lab3.defense.ArmorHandler;

public class Skeleton extends AbstractEnemy {

    private static final double RESISTANCE = 0.3;

    public Skeleton() {
        super("skeleton", "Скелет", 110, 10, 24, 90, 70, new ArmorHandler(RESISTANCE));
    }

}
```

```java
package org.misis.tp.lab3.enemy;

import org.misis.tp.lab3.combat.Hit;
import org.misis.tp.lab3.core.Dice;
import org.misis.tp.lab3.defense.ArmorHandler;

public class Dragon extends AbstractEnemy {

    private static final double RESISTANCE = 0.25;
    private static final double FIRE_BREATH_CHANCE = 0.25;
    private static final int FIRE_BREATH_MULTIPLIER = 2;

    public Dragon() {
        super("dragon", "Дракон", 260, 16, 22, 500, 400, new ArmorHandler(RESISTANCE));
    }

    @Override
    public Hit rollDamage() {
        Hit bite = super.rollDamage();
        if (Dice.getInstance().chance(FIRE_BREATH_CHANCE)) {
            return new Hit(bite.damage() * FIRE_BREATH_MULTIPLIER, true);
        }

        return bite;
    }

}
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

```java
package org.misis.tp.lab3.game;

import org.misis.tp.lab3.camp.Merchant;
import org.misis.tp.lab3.combat.Outcome;
import org.misis.tp.lab3.core.UnknownCodeException;
import org.misis.tp.lab3.event.GameEvent;
import org.misis.tp.lab3.event.GameEventPublisher;
import org.misis.tp.lab3.event.GameEventType;
import org.misis.tp.lab3.hero.Hero;
import org.misis.tp.lab3.hero.HeroClass;
import org.misis.tp.lab3.item.DefaultCatalog;
import org.misis.tp.lab3.item.Item;
import org.misis.tp.lab3.item.ItemCatalog;
import org.misis.tp.lab3.kit.StarterKitFactory;
import org.misis.tp.lab3.progress.AchievementTracker;
import org.misis.tp.lab3.progress.ExpeditionJournal;
import org.misis.tp.lab3.storage.CachingHeroRepository;
import org.misis.tp.lab3.storage.FileHeroRepository;
import org.misis.tp.lab3.storage.HeroRepository;
import org.misis.tp.lab3.tactic.AggressiveTactic;
import org.misis.tp.lab3.tactic.BalancedTactic;
import org.misis.tp.lab3.tactic.CautiousTactic;
import org.misis.tp.lab3.tactic.Tactic;
import org.misis.tp.lab3.world.Crypt;
import org.misis.tp.lab3.world.DragonLair;
import org.misis.tp.lab3.world.ExpeditionReport;
import org.misis.tp.lab3.world.Forest;
import org.misis.tp.lab3.world.HauntedManor;
import org.misis.tp.lab3.world.Location;
import org.misis.tp.lab3.world.LocationRegistry;

import java.nio.file.Path;
import java.util.List;

public class GameFacade {

    private final ItemCatalog catalog;
    private final LocationRegistry locations;
    private final List<Tactic> tactics;
    private final Merchant merchant;
    private final HeroRepository heroes;
    private final GameEventPublisher events;
    private final ExpeditionJournal journal;

    public GameFacade(Path saveFile) {
        catalog = DefaultCatalog.create();
        locations = new LocationRegistry(List.of(
            new Forest(catalog),
            new HauntedManor(catalog),
            new Crypt(catalog),
            new DragonLair(catalog)
        ));
        tactics = List.of(new BalancedTactic(), new AggressiveTactic(), new CautiousTactic());
        merchant = new Merchant(
            catalog, List.of(
            DefaultCatalog.IRON_SWORD, DefaultCatalog.ELVEN_BOW, DefaultCatalog.EMBER_STAFF,
            DefaultCatalog.CHAINMAIL, DefaultCatalog.PLATE_ARMOR
        )
        );
        heroes = new CachingHeroRepository(new FileHeroRepository(saveFile, catalog));

        events = new GameEventPublisher();
        journal = new ExpeditionJournal();
        AchievementTracker achievements = new AchievementTracker();
        events.subscribe(GameEventType.VICTORY, achievements);
        events.subscribe(GameEventType.LEVEL_UP, achievements);
        events.subscribe(GameEventType.VICTORY, journal);
        events.subscribe(GameEventType.DEFEAT, journal);
    }

    public List<HeroClass> getHeroClasses() {
        return List.of(HeroClass.values());
    }

    public List<Item> getStarterKit(String heroClassCode) {
        StarterKitFactory kit = HeroClass.byCode(heroClassCode).getStarterKit(catalog);
        return List.of(kit.createWeapon(), kit.createArmor());
    }

    public List<Location> getLocations() {
        return locations.getAll();
    }

    public Location getLocation(String code) {
        return locations.get(code);
    }

    public List<Tactic> getTactics() {
        return tactics;
    }

    public Tactic getTactic(String code) {
        for (Tactic tactic : tactics) {
            if (tactic.getCode().equals(code)) {
                return tactic;
            }
        }

        throw new UnknownCodeException("Тактика", code);
    }

    public List<Item> getShowcase() {
        return merchant.getShowcase();
    }

    public int getSellPrice(Item item) {
        return merchant.getSellPrice(item);
    }

    public List<Hero> getHeroes() {
        return heroes.findAll();
    }

    public Hero getHero(String name) {
        return heroes.findByName(name)
            .orElseThrow(() -> new IllegalArgumentException("Героя с именем " + name + " нет"));
    }

    public Hero createHero(String name, String heroClassCode) {
        if (heroes.findByName(name).isPresent()) {
            throw new IllegalArgumentException("Герой с именем " + name + " уже есть");
        }

        HeroClass heroClass = HeroClass.byCode(heroClassCode);
        Hero hero = new Hero.Builder()
            .setName(name)
            .setHeroClass(heroClass)
            .setStarterKit(heroClass.getStarterKit(catalog))
            .build();
        heroes.save(hero);
        return hero;
    }

    public ExpeditionReport explore(String heroName, String locationCode, String tacticCode) {
        Hero hero = getHero(heroName);
        ExpeditionReport report = locations.get(locationCode).explore(hero, getTactic(tacticCode));

        GameEventType result = report.battle().outcome() == Outcome.VICTORY ? GameEventType.VICTORY : GameEventType.DEFEAT;
        events.publish(new GameEvent(result, hero, report));
        if (report.rewards().levelsGained() > 0) {
            events.publish(new GameEvent(GameEventType.LEVEL_UP, hero, report));
        }

        heroes.save(hero);
        return report;
    }

    public Item equip(String heroName, int inventoryIndex) {
        Hero hero = getHero(heroName);
        Item item = hero.getInventory().get(inventoryIndex);
        hero.equip(item);
        heroes.save(hero);
        return item;
    }

    public Item buy(String heroName, String itemCode) {
        Hero hero = getHero(heroName);
        Item item = merchant.buy(hero, itemCode);
        heroes.save(hero);
        return item;
    }

    public int sell(String heroName, int inventoryIndex) {
        Hero hero = getHero(heroName);
        int price = merchant.sell(hero, hero.getInventory().get(inventoryIndex));
        heroes.save(hero);
        return price;
    }

    public List<ExpeditionReport> getJournal(String heroName) {
        return journal.getEntries(heroName);
    }

}
```

Вот здесь и настраиваются связи между издателем и подписчиками, а также происходит отправка событий после экспедиции.
Одно и то же событие (победу) получают оба подписчика: и выдача достижений, и журнал. Если завтра мы захотим, например,
отправлять уведомления о победах в какой-нибудь мессенджер, достаточно будет написать нового подписчика и подписать его в конструкторе фасада.

Обратите внимание, что методы фасада принимают имя героя и коды (локации, тактики, предмета), а не сами объекты: тот,
кто пользуется фасадом, может вообще ничего не знать о том, как игра устроена внутри. Каждый метод сам находит героя в
хранилище (тут и работает кеш заместителя) и сохраняет его после изменений.

Адаптер и фасад довольно похожи, ведь оба оборачивают другие объекты. Но адаптер приводит интерфейс одного
объекта к другому, уже существующему интерфейсу, который ожидает клиент, а фасад определяет новый, упрощенный
интерфейс сразу к целой подсистеме.

В реализации фасада важно, чтобы он не был объектом, который знает обо всем и делает все. Поэтому
фасад не должен сам содержать игровую логику: он только связывает между собой классы, которые ее реализуют.
Применять фасад стоит, когда нужно дать простой вход в сложную подсистему или отделить от нее клиентский код.

## Консоль

Вывод в консоль научился показывать новые события боя, особенности классов и достижения:

```java
package org.misis.tp.lab3;

import org.misis.tp.lab3.combat.BattleEvent;
import org.misis.tp.lab3.combat.BattleReport;
import org.misis.tp.lab3.combat.Outcome;
import org.misis.tp.lab3.combat.Participant;
import org.misis.tp.lab3.combat.Side;
import org.misis.tp.lab3.hero.Hero;
import org.misis.tp.lab3.hero.HeroClass;
import org.misis.tp.lab3.item.Enchantment;
import org.misis.tp.lab3.item.Item;
import org.misis.tp.lab3.progress.Achievement;
import org.misis.tp.lab3.world.ExpeditionReport;
import org.misis.tp.lab3.world.Rewards;

import java.util.ArrayList;
import java.util.List;

public class ConsoleView {

    public String describe(Item item) {
        StringBuilder description = new StringBuilder(item.getName());

        List<Enchantment> enchantments = item.getEnchantments();
        if (!enchantments.isEmpty()) {
            List<String> names = new ArrayList<>();
            for (Enchantment enchantment : enchantments) {
                names.add(enchantment.name());
            }
            description.append(" [").append(String.join(", ", names)).append("]");
        }

        return description.append(" (").append(item.getDescription()).append(")").toString();
    }

    public String describe(HeroClass heroClass) {
        if (heroClass.getEvasion() > 0) {
            return String.format("уклонение %d%%", Math.round(heroClass.getEvasion() * 100));
        }
        if (heroClass.getBarrier() > 0) {
            return String.format("магический барьер %d", heroClass.getBarrier());
        }

        return "без особенностей";
    }

    public void printHero(Hero hero) {
        System.out.printf(
            "%s, %s %d ур. | здоровье %d | опыт %d/%d | золото %d%n",
            hero.getName(), hero.getHeroClass().getDisplayName(), hero.getLevel(),
            hero.getMaxHealth(), hero.getExperience(), hero.getExperienceToNextLevel(), hero.getGold()
        );
        System.out.println("  Оружие: " + describe(hero.getWeapon()));
        System.out.println("  Броня:  " + describe(hero.getArmor()));

        List<Achievement> achievements = hero.getAchievements();
        if (!achievements.isEmpty()) {
            List<String> names = new ArrayList<>();
            for (Achievement achievement : achievements) {
                names.add(achievement.getDisplayName());
            }
            System.out.println("  Достижения: " + String.join(", ", names));
        }
    }

    public void printReport(ExpeditionReport report, String locationName) {
        BattleReport battle = report.battle();
        System.out.printf("=== %s (сид %d) ===%n", locationName, report.seed());
        System.out.printf(
            "%s встречает врага: %s (%d здоровья)%n",
            battle.hero().name(), battle.enemy().name(), battle.enemy().maxHealth()
        );

        for (BattleEvent event : battle.events()) {
            printEvent(event, battle);
        }

        printRewards(battle, report.rewards());
    }

    public String summarize(ExpeditionReport report, String locationName, String tacticName) {
        BattleReport battle = report.battle();
        String result = battle.outcome() == Outcome.VICTORY ? "победа" : "поражение";
        return String.format("%s, %s, тактика \"%s\": %s", locationName, battle.enemy().name(), tacticName, result);
    }

    private void printEvent(BattleEvent event, BattleReport battle) {
        Participant source = participant(event.source(), battle);
        Participant target = participant(event.target(), battle);

        switch (event.type()) {
            case HIT -> System.out.printf(
                "[раунд %d] %s -> %s: %d урона, осталось %d/%d%n",
                event.round(), source.name(), target.name(),
                event.amount(), event.targetHealthAfter(), target.maxHealth()
            );
            case CRITICAL_HIT -> System.out.printf(
                "[раунд %d] %s -> %s: КРИТ! %d урона, осталось %d/%d%n",
                event.round(), source.name(), target.name(),
                event.amount(), event.targetHealthAfter(), target.maxHealth()
            );
            case EVADE -> System.out.printf(
                "[раунд %d] %s -> %s: промах, %s уклоняется%n",
                event.round(), source.name(), target.name(), target.name()
            );
            case ABSORB -> System.out.printf(
                "[раунд %d] %s -> %s: барьер поглощает %d урона%n",
                event.round(), source.name(), target.name(), event.amount()
            );
            case POWER_STRIKE -> System.out.printf(
                "[раунд %d] %s бьет в полную силу%n", event.round(), source.name()
            );
            case DEFEND -> System.out.printf(
                "[раунд %d] %s уходит в оборону%n", event.round(), source.name()
            );
            case DEATH -> System.out.printf("%s повержен!%n", target.name());
        }
    }

    private void printRewards(BattleReport battle, Rewards rewards) {
        if (battle.outcome() == Outcome.DEFEAT) {
            System.out.printf(
                "Поражение... %s едва уносит ноги и теряет %d золота%n",
                battle.hero().name(), -rewards.gold()
            );
            return;
        }

        System.out.printf("Победа! +%d опыта, +%d золота%n", rewards.experience(), rewards.gold());
        if (rewards.loot() != null) {
            System.out.println("Добыча: " + describe(rewards.loot()));
        }
        if (rewards.levelsGained() > 0) {
            System.out.println("Новый уровень!");
        }
    }

    private Participant participant(Side side, BattleReport battle) {
        return side == Side.HERO ? battle.hero() : battle.enemy();
    }

}
```

А игровой цикл в методе `main` теперь работает только с фасадом и помнит лишь имя текущего героя:

Вместо прямой работы с каталогом, локациями и торговцем:

- создаём `GameFacade` с файлом сохранения и выбираем нового или сохранённого героя;
- перед экспедицией выбираем тактику;
- работам с экспедициями, экипировкой и торговлей через фасад;
- добавляем пункт меню для журнала экспедиций.

```java
package org.misis.tp.lab3;

import org.misis.tp.lab3.combat.Outcome;
import org.misis.tp.lab3.game.GameFacade;
import org.misis.tp.lab3.hero.Hero;
import org.misis.tp.lab3.hero.HeroClass;
import org.misis.tp.lab3.item.Item;
import org.misis.tp.lab3.tactic.Tactic;
import org.misis.tp.lab3.world.ExpeditionReport;
import org.misis.tp.lab3.world.Location;

import java.io.Console;
import java.nio.charset.Charset;
import java.nio.file.Path;
import java.util.List;
import java.util.Scanner;

public class Main {

    private static final Path SAVE_FILE = Path.of("saves", "heroes.json");
    private static final Scanner INPUT = new Scanner(System.in, inputCharset());
    private static final ConsoleView VIEW = new ConsoleView();

    public static void main(String[] args) {
        System.out.println("Файл сохранения: " + SAVE_FILE.toAbsolutePath().normalize());
        GameFacade game = new GameFacade(SAVE_FILE);
        String heroName = chooseHero(game);

        while (true) {
            System.out.println();
            VIEW.printHero(game.getHero(heroName));
            System.out.println("1 - отправиться в экспедицию, 2 - снаряжение, 3 - торговец, 4 - журнал экспедиций, 0 - выйти из игры");
            switch (INPUT.nextLine().strip()) {
                case "1" -> goOnExpedition(game, heroName);
                case "2" -> manageEquipment(game, heroName);
                case "3" -> visitMerchant(game, heroName);
                case "4" -> printJournal(game, heroName);
                case "0" -> {
                    return;
                }
                default -> System.out.println("Такого варианта нет");
            }
        }
    }

    private static String chooseHero(GameFacade game) {
        List<Hero> saved = game.getHeroes();
        if (!saved.isEmpty()) {
            System.out.println("Выберите героя (0 - создать нового):");
            for (int i = 0; i < saved.size(); i++) {
                Hero hero = saved.get(i);
                System.out.printf(
                    "%d - %s, %s %d ур.%n", i + 1, hero.getName(), hero.getHeroClass().getDisplayName(), hero.getLevel()
                );
            }
            int index = readChoice(saved.size());
            if (index >= 0) {
                String name = saved.get(index).getName();
                System.out.printf("%s возвращается в лагерь на опушке мистического леса.%n", name);
                return name;
            }
        }

        String name = createHero(game);
        System.out.printf("%s разбивает лагерь на опушке мистического леса.%n", name);
        return name;
    }

    private static String createHero(GameFacade game) {
        System.out.println("Создайте своего героя!");
        while (true) {
            System.out.println("Выберите класс:");
            List<HeroClass> heroClasses = game.getHeroClasses();
            for (int i = 0; i < heroClasses.size(); i++) {
                HeroClass heroClass = heroClasses.get(i);
                List<Item> kit = game.getStarterKit(heroClass.getCode());
                System.out.printf(
                    "%d - %s (здоровье %d, %s): %s, %s%n",
                    i + 1, heroClass.getDisplayName(), heroClass.getBaseHealth(), VIEW.describe(heroClass),
                    VIEW.describe(kit.get(0)), VIEW.describe(kit.get(1))
                );
            }
            int classIndex = readChoice(heroClasses.size());
            if (classIndex < 0) {
                continue;
            }

            System.out.println("Введите имя:");
            String name = INPUT.nextLine().strip();

            try {
                return game.createHero(name, heroClasses.get(classIndex).getCode()).getName();
            } catch (IllegalArgumentException e) {
                System.out.println("Не получилось создать героя: " + e.getMessage());
            }
        }
    }

    private static void goOnExpedition(GameFacade game, String heroName) {
        List<Location> locations = game.getLocations();
        System.out.println("Куда отправимся?");
        for (int i = 0; i < locations.size(); i++) {
            System.out.printf(
                "%d - %s (рекомендуемый уровень %d)%n",
                i + 1, locations.get(i).getDisplayName(), locations.get(i).getRecommendedLevel()
            );
        }
        int locationIndex = readChoice(locations.size());
        if (locationIndex < 0) {
            return;
        }

        List<Tactic> tactics = game.getTactics();
        System.out.println("Какой тактики держаться в бою?");
        for (int i = 0; i < tactics.size(); i++) {
            System.out.printf("%d - %s: %s%n", i + 1, tactics.get(i).getDisplayName(), tactics.get(i).getDescription());
        }
        int tacticIndex = readChoice(tactics.size());
        if (tacticIndex < 0) {
            return;
        }

        Location location = locations.get(locationIndex);
        ExpeditionReport report = game.explore(heroName, location.getCode(), tactics.get(tacticIndex).getCode());
        System.out.println();
        VIEW.printReport(report, location.getDisplayName());

        if (report.battle().outcome() == Outcome.VICTORY && "dragon".equals(report.battle().enemy().code())) {
            System.out.printf("%nДракон повержен! Барды еще долго будут петь о подвиге героя по имени %s.%n", heroName);
        }
    }

    private static void manageEquipment(GameFacade game, String heroName) {
        List<Item> inventory = game.getHero(heroName).getInventory();
        if (inventory.isEmpty()) {
            System.out.println("Инвентарь пуст");
            return;
        }

        System.out.println("Что экипировать? (0 - назад)");
        printItems(inventory);
        int index = readChoice(inventory.size());
        if (index < 0) {
            return;
        }

        Item item = game.equip(heroName, index);
        System.out.println("Экипировано: " + VIEW.describe(item));
    }

    private static void visitMerchant(GameFacade game, String heroName) {
        System.out.printf("Торговец: \"Чего желаешь?\" (золото: %d)%n", game.getHero(heroName).getGold());
        System.out.println("1 - купить, 2 - продать, 0 - назад");
        switch (INPUT.nextLine().strip()) {
            case "1" -> buy(game, heroName);
            case "2" -> sell(game, heroName);
            default -> {
            }
        }
    }

    private static void buy(GameFacade game, String heroName) {
        List<Item> showcase = game.getShowcase();
        for (int i = 0; i < showcase.size(); i++) {
            System.out.printf("%d - %s: %d золота%n", i + 1, VIEW.describe(showcase.get(i)), showcase.get(i).getPrice());
        }
        int index = readChoice(showcase.size());
        if (index < 0) {
            return;
        }

        try {
            Item item = game.buy(heroName, showcase.get(index).getCode());
            System.out.println("Куплено: " + VIEW.describe(item));
        } catch (IllegalStateException e) {
            System.out.println(e.getMessage());
        }
    }

    private static void sell(GameFacade game, String heroName) {
        List<Item> inventory = game.getHero(heroName).getInventory();
        if (inventory.isEmpty()) {
            System.out.println("Продавать нечего (экипированные предметы продать нельзя)");
            return;
        }

        for (int i = 0; i < inventory.size(); i++) {
            System.out.printf(
                "%d - %s: %d золота%n", i + 1, VIEW.describe(inventory.get(i)), game.getSellPrice(inventory.get(i))
            );
        }
        int index = readChoice(inventory.size());
        if (index < 0) {
            return;
        }

        String itemName = inventory.get(index).getName();
        int price = game.sell(heroName, index);
        System.out.printf("Продано: %s за %d золота%n", itemName, price);
    }

    private static void printJournal(GameFacade game, String heroName) {
        List<ExpeditionReport> journal = game.getJournal(heroName);
        if (journal.isEmpty()) {
            System.out.println("В этот раз экспедиций еще не было");
            return;
        }

        for (int i = 0; i < journal.size(); i++) {
            ExpeditionReport report = journal.get(i);
            System.out.printf(
                "%d. %s%n", i + 1, VIEW.summarize(
                    report,
                    game.getLocation(report.locationCode()).getDisplayName(),
                    game.getTactic(report.tacticCode()).getDisplayName()
                )
            );
        }
    }

    private static void printItems(List<Item> items) {
        for (int i = 0; i < items.size(); i++) {
            System.out.printf("%d - %s%n", i + 1, VIEW.describe(items.get(i)));
        }
    }

    private static Charset inputCharset() {
        Console console = System.console();
        return console != null ? console.charset() : Charset.defaultCharset();
    }

    private static int readChoice(int count) {
        try {
            int choice = Integer.parseInt(INPUT.nextLine().strip());
            if (choice >= 1 && choice <= count) {
                return choice - 1;
            }
        } catch (NumberFormatException ignored) {
        }

        return -1;
    }

}
```

Супер! Попробуйте запустить игру, сохраниться и вернуться к своему герою :)

## Демонстрация паттернов

Чтобы показать все паттерны одним запуском, напишем класс `Demo`:

Демонстрацию лабы 3 заменяем новой:

```java
package org.misis.tp.lab3;

import org.misis.tp.lab3.combat.BattleEvent;
import org.misis.tp.lab3.core.Dice;
import org.misis.tp.lab3.defense.ArmorHandler;
import org.misis.tp.lab3.defense.BarrierHandler;
import org.misis.tp.lab3.defense.Damage;
import org.misis.tp.lab3.defense.DamageHandler;
import org.misis.tp.lab3.defense.EvasionHandler;
import org.misis.tp.lab3.enemy.ArmoredEnemy;
import org.misis.tp.lab3.enemy.Dragon;
import org.misis.tp.lab3.enemy.Enemy;
import org.misis.tp.lab3.enemy.Goblin;
import org.misis.tp.lab3.enemy.HauntedWeapon;
import org.misis.tp.lab3.enemy.LegendaryEnemy;
import org.misis.tp.lab3.enemy.WindfuryEnemy;
import org.misis.tp.lab3.event.GameEvent;
import org.misis.tp.lab3.event.GameEventPublisher;
import org.misis.tp.lab3.event.GameEventType;
import org.misis.tp.lab3.game.GameFacade;
import org.misis.tp.lab3.hero.Hero;
import org.misis.tp.lab3.hero.HeroClass;
import org.misis.tp.lab3.item.DefaultCatalog;
import org.misis.tp.lab3.item.ItemCatalog;
import org.misis.tp.lab3.item.Weapon;
import org.misis.tp.lab3.progress.Achievement;
import org.misis.tp.lab3.progress.AchievementTracker;
import org.misis.tp.lab3.progress.ExpeditionJournal;
import org.misis.tp.lab3.storage.CachingHeroRepository;
import org.misis.tp.lab3.storage.FileHeroRepository;
import org.misis.tp.lab3.storage.HeroRepository;
import org.misis.tp.lab3.tactic.AggressiveTactic;
import org.misis.tp.lab3.tactic.BalancedTactic;
import org.misis.tp.lab3.tactic.CautiousTactic;
import org.misis.tp.lab3.tactic.Tactic;
import org.misis.tp.lab3.world.Crypt;
import org.misis.tp.lab3.world.ExpeditionReport;
import org.misis.tp.lab3.world.Forest;
import org.misis.tp.lab3.world.HauntedManor;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;

public class Demo {

    private static final ConsoleView VIEW = new ConsoleView();

    public static void main(String[] args) throws IOException {
        ItemCatalog catalog = DefaultCatalog.create();
        Dice dice = Dice.getInstance();

        System.out.println("===== Цепочка обязанностей =====");
        DamageHandler armorOnly = new ArmorHandler(0.3);
        printHits("Броня 30%", armorOnly, 2);

        DamageHandler armorAndBarrier = new ArmorHandler(0.3);
        armorAndBarrier.setNext(new BarrierHandler(30));
        printHits("Броня 30% -> барьер 30", armorAndBarrier, 3);

        dice.reseed(3);
        DamageHandler fullChain = new EvasionHandler(0.5);
        fullChain.setNext(new ArmorHandler(0.3)).setNext(new BarrierHandler(30));
        printHits("Уклонение 50% -> броня 30% -> барьер 30", fullChain, 3);

        System.out.println();
        System.out.println("===== Стратегия =====");
        Crypt crypt = new Crypt(catalog);
        for (Tactic tactic : List.of(new BalancedTactic(), new AggressiveTactic(), new CautiousTactic())) {
            Hero mage = newHero("Мерлин", HeroClass.MAGE, 2, catalog);
            ExpeditionReport report = crypt.explore(mage, tactic, 9);
            System.out.printf(
                "%s: %s, раундов %d, у героя осталось %d/%d здоровья%n",
                tactic.getDisplayName(), describeOutcome(report), countRounds(report),
                mage.getHealth(), mage.getMaxHealth()
            );
        }

        System.out.println();
        System.out.println("===== Декоратор =====");
        List<Enemy> goblins = List.of(
            new Goblin(),
            new LegendaryEnemy(new Goblin()),
            new WindfuryEnemy(new Goblin()),
            new LegendaryEnemy(new WindfuryEnemy(new Goblin())),
            new ArmoredEnemy(new Goblin()),
            new ArmoredEnemy(new LegendaryEnemy(new WindfuryEnemy(new Dragon())))
        );
        for (Enemy goblin : goblins) {
            dice.reseed(5);
            List<String> hits = new ArrayList<>();
            for (int i = 0; i < 3; i++) {
                hits.add(String.valueOf(goblin.rollDamage().damage()));
            }
            Damage received = goblin.receiveHit(20);
            System.out.printf(
                "%s: удары %s, награда %d опыта; входящий урон 20 -> %d, здоровье %d/%d%n",
                goblin.getName(), String.join(", ", hits), goblin.getExperienceReward(),
                received.getAmount(), goblin.getHealth(), goblin.getMaxHealth()
            );
        }

        System.out.println();
        System.out.println("===== Адаптер =====");
        Weapon bow = catalog.createWeapon(DefaultCatalog.ELVEN_BOW);
        Enemy hauntedBow = new HauntedWeapon(bow);
        System.out.println("Предмет: " + VIEW.describe(bow));
        System.out.printf(
            "Враг:    %s (%d здоровья, награда %d опыта)%n",
            hauntedBow.getName(), hauntedBow.getMaxHealth(), hauntedBow.getExperienceReward()
        );
        HauntedManor manor = new HauntedManor(catalog);
        Hero ranger = newHero("Леголас", HeroClass.RANGER, 2, catalog);
        VIEW.printReport(manor.explore(ranger, new CautiousTactic(), 55), manor.getDisplayName());

        System.out.println();
        System.out.println("===== Наблюдатель =====");
        GameEventPublisher events = new GameEventPublisher();
        ExpeditionJournal journal = new ExpeditionJournal();
        AchievementTracker achievements = new AchievementTracker();
        events.subscribe(GameEventType.VICTORY, achievements);
        events.subscribe(GameEventType.VICTORY, journal);
        events.subscribe(GameEventType.DEFEAT, journal);
        events.subscribe(
            GameEventType.VICTORY, event -> System.out.printf(
                "Событие %s: герой %s%n", event.type(), event.hero().getName()
            )
        );

        Hero warrior = newHero("Артур", HeroClass.WARRIOR, 1, catalog);
        Forest forest = new Forest(catalog);
        ExpeditionReport victory = forest.explore(warrior, new BalancedTactic(), 7);
        events.publish(new GameEvent(GameEventType.VICTORY, warrior, victory));
        System.out.println("Достижения героя: " + describeAchievements(warrior));
        System.out.println("Записей в журнале: " + journal.getEntries(warrior.getName()).size());

        System.out.println();
        System.out.println("===== Заместитель =====");
        Path saveFile = Files.createTempFile("heroes", ".json");
        HeroRepository file = new FileHeroRepository(saveFile, catalog);
        file.save(warrior);
        System.out.println("Содержимое файла:");
        System.out.print(Files.readString(saveFile));
        System.out.println(
            "Без заместителя два поиска дают один и тот же объект: "
                + (file.findByName("Артур").get() == file.findByName("Артур").get())
        );
        HeroRepository cached = new CachingHeroRepository(file);
        Hero fromCache = cached.findByName("Артур").get();
        System.out.println(
            "С заместителем два поиска дают один и тот же объект: " + (fromCache == cached.findByName("Артур").get())
        );
        fromCache.addGold(100);
        cached.save(fromCache);
        System.out.println("После сохранения через заместитель золото героя в файле: " + file.findByName("Артур").get().getGold());
        Files.delete(saveFile);

        System.out.println();
        System.out.println("===== Фасад =====");
        Path gameFile = Files.createTempFile("game", ".json");
        GameFacade game = new GameFacade(gameFile);
        game.createHero("Гэндальф", HeroClass.MAGE.getCode());
        dice.reseed(11);
        ExpeditionReport expedition = game.explore("Гэндальф", Forest.CODE, "cautious");
        System.out.println("Экспедиция через фасад: " + VIEW.summarize(
            expedition, game.getLocation(expedition.locationCode()).getDisplayName(),
            game.getTactic(expedition.tacticCode()).getDisplayName()
        ));
        VIEW.printHero(game.getHero("Гэндальф"));
        Files.delete(gameFile);
    }

    private static Hero newHero(String name, HeroClass heroClass, int level, ItemCatalog catalog) {
        return new Hero.Builder()
            .setName(name)
            .setHeroClass(heroClass)
            .setStarterKit(heroClass.getStarterKit(catalog))
            .setLevel(level)
            .build();
    }

    private static void printHits(String title, DamageHandler chain, int count) {
        List<String> results = new ArrayList<>();
        for (int i = 0; i < count; i++) {
            Damage damage = new Damage(30);
            chain.handle(damage);
            if (damage.isEvaded()) {
                results.add("уклонение");
            } else if (damage.getAbsorbed() > 0) {
                results.add(String.format("поглощено %d, прошло %d", damage.getAbsorbed(), damage.getAmount()));
            } else {
                results.add("прошло " + damage.getAmount());
            }
        }
        System.out.printf("%s: %s%n", title, String.join("; ", results));
    }

    private static String describeOutcome(ExpeditionReport report) {
        return switch (report.battle().outcome()) {
            case VICTORY -> "победа";
            case DEFEAT -> "поражение";
        };
    }

    private static int countRounds(ExpeditionReport report) {
        List<BattleEvent> events = report.battle().events();
        return events.get(events.size() - 1).round();
    }

    private static String describeAchievements(Hero hero) {
        List<String> names = new ArrayList<>();
        for (Achievement achievement : hero.getAchievements()) {
            names.add(achievement.getDisplayName());
        }

        return String.join(", ", names);
    }

}
```

Вывод программы:

```
===== Цепочка обязанностей =====
Броня 30%: прошло 21; прошло 21
Броня 30% -> барьер 30: поглощено 21, прошло 0; поглощено 9, прошло 12; прошло 21
Уклонение 50% -> броня 30% -> барьер 30: поглощено 21, прошло 0; уклонение; уклонение

===== Стратегия =====
Баланс: поражение, раундов 8, у героя осталось 0/100 здоровья
Натиск: победа, раундов 7, у героя осталось 12/100 здоровья
Осторожность: победа, раундов 9, у героя осталось 5/100 здоровья

===== Декоратор =====
Гоблин: удары 14, 7, 14, награда 40 опыта; входящий урон 20 -> 20, здоровье 30/50
Легендарный Гоблин: удары 28, 14, 28, награда 80 опыта; входящий урон 20 -> 20, здоровье 30/50
Неистовый Гоблин: удары 21, 25, 23, награда 60 опыта; входящий урон 20 -> 20, здоровье 30/50
Легендарный Неистовый Гоблин: удары 42, 50, 46, награда 120 опыта; входящий урон 20 -> 20, здоровье 30/50
Бронированный Гоблин: удары 14, 7, 14, награда 40 опыта; входящий урон 20 -> 10, здоровье 40/50
Бронированный Легендарный Неистовый Дракон: удары 104, 76, 102, награда 1500 опыта; входящий урон 20 -> 8, здоровье 252/260

===== Адаптер =====
Предмет: Эльфийский лук (урон 14, шанс крита 40%)
Враг:    Одержимый Эльфийский лук (70 здоровья, награда 45 опыта)
=== Проклятый особняк (сид 55) ===
Леголас встречает врага: Одержимый Эльфийский лук (70 здоровья)
[раунд 1] Леголас -> Одержимый Эльфийский лук: 13 урона, осталось 57/70
[раунд 1] Одержимый Эльфийский лук -> Леголас: КРИТ! 22 урона, осталось 93/115
[раунд 2] Леголас -> Одержимый Эльфийский лук: 13 урона, осталось 44/70
[раунд 2] Одержимый Эльфийский лук -> Леголас: КРИТ! 22 урона, осталось 71/115
[раунд 3] Леголас -> Одержимый Эльфийский лук: 13 урона, осталось 31/70
[раунд 3] Одержимый Эльфийский лук -> Леголас: промах, Леголас уклоняется
[раунд 4] Леголас -> Одержимый Эльфийский лук: 13 урона, осталось 18/70
[раунд 4] Одержимый Эльфийский лук -> Леголас: 11 урона, осталось 60/115
[раунд 5] Леголас -> Одержимый Эльфийский лук: 13 урона, осталось 5/70
[раунд 5] Одержимый Эльфийский лук -> Леголас: КРИТ! 22 урона, осталось 38/115
[раунд 6] Леголас уходит в оборону
[раунд 6] Леголас -> Одержимый Эльфийский лук: КРИТ! 12 урона, осталось 0/70
Одержимый Эльфийский лук повержен!
Победа! +45 опыта, +40 золота

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
    "gold": 79,
    "weapon": {
      "code": "training-sword",
      "enchantments": []
    },
    "armor": {
      "code": "chainmail",
      "enchantments": []
    },
    "inventory": [],
    "achievements": [
      "first-blood"
    ]
  }
]
Без заместителя два поиска дают один и тот же объект: false
С заместителем два поиска дают один и тот же объект: true
После сохранения через заместитель золото героя в файле: 179

===== Фасад =====
Экспедиция через фасад: Мистический лес, Гоблин, тактика "Осторожность": победа
Гэндальф, Маг 1 ур. | здоровье 85 | опыт 40/100 | золото 76
  Оружие: Посох ученика (урон 10-18)
  Броня:  Льняная роба (защита 10%)
  Достижения: Первая кровь
```

Не забудьте закоммитить и запушить изменения!

# Лаба 3 — Порождающие шаблоны проектирования

В этой лабораторной работе будут рассмотрены порождающие шаблоны проектирования.

Шаблоны проектирования (OOP design patterns) - это повторяемые решения типичных проблем проектирования в
объектно-ориентированном программировании. Они представляют собой лучшие практики, которые уже доказали свою
эффективность. Паттерны помогают разработчикам создавать архитектуры, которые более гибки, расширяемы и поддерживаемы.

Паттерны не являются готовыми кусками кода, которые можно вставить в программу. Это скорее "рецепты" для решения общих
задач, которые могут быть реализованы по-разному в зависимости от конкретной ситуации.

Паттерны проектирования можно классифицировать на три основные категории:

1. Порождающие паттерны (Creational)
2. Структурные паттерны (Structural)
3. Поведенческие паттерны (Behavioral)

Порождающие паттерны отвечают за создание объектов. Эти паттерны позволяют инкапсулировать логику создания объектов и
избежать привязки к конкретным классам, что делает код более гибким. Они помогают управлять процессом инстанцирования
объектов, делая код более независимым от конкретных классов и объектов.

# Что нужно сдать к защите

Выполнять лабораторную можно на любом языке программирования.

На защите нужно продемонстрировать работоспособность кода, при этом он должен быть запушен в удалённый репозиторий.

В качестве лабораторной работы делается небольшая текстовая RPG, в которой герой ходит в экспедиции: на этой игре будут
строиться и следующие лабораторные работы. Ее реализация шаг за шагом разобрана в разделе "Практика", ссылки на
практику на других языках находятся в начале раздела.

## Паттерны

В программе должны быть реализованы следующие паттерны:

1. Одиночка (Singleton)
2. Строитель (Builder)
3. Фабричный метод (Factory Method)
4. Абстрактная фабрика (Abstract Factory)
5. Прототип (Prototype)

Использование каждого из паттернов *должно быть проиллюстрировано* выводом в консоль, минимальные требования:

1. Показать, что при работе с одиночкой нельзя создать больше одного экземпляра.
2. Продемонстрировать создание объекта при помощи строителя.
3. Создать два подкласса, реализующих фабричный метод, продемонстрировать различия в возвращаемых фабричным методом
   объектах.
4. Создать два подкласса, реализующих методы абстрактной фабрики, продемонстрировать различия в семействах объектов,
   полученных от этих реализаций (семейство включает не менее двух членов, то есть в абстрактной фабрике должно быть как минимум два порождающих метода).
5. Продемонстрировать логику копирования объекта, реализующего паттерн прототип.

# Вопросы к защите

Отвечая на вопросы про конкретные паттерны, покажите, где и как этот паттерн реализован в вашем коде.

1. Что такое шаблоны проектирования? Зачем они нужны? Как их можно классифицировать?
2. Что такое порождающие паттерны проектирования? Какие задачи они решают?
3. Паттерн Одиночка. Что делает, реализация, плюсы и минусы, когда применять?
4. Паттерн Строитель. Что делает, реализация, плюсы и минусы, когда применять? Чем строитель лучше конструктора с большим количеством параметров?
5. Паттерн Прототип. Что делает, реализация, плюсы и минусы, когда применять?
6. Что такое глубокое и поверхностное клонирование? Чем они отличаются? Что важно учитывать при поверхностном
   клонировании?
7. Паттерн Фабричный метод. Что делает, реализация, плюсы и минусы, когда применять?
8. Паттерн Абстрактная фабрика. Что делает, реализация, плюсы и минусы, когда применять?
9. Чем фабричный метод отличается от абстрактной фабрики?

# Практика

> Если вы пишете не на Java, эта же практика есть на других языках: [Python](practice/python.md), [C++](practice/cpp.md), [Go](practice/go.md), [C#](practice/csharp.md), [Kotlin](practice/kotlin.md), [TypeScript](practice/typescript.md).

В этой лабе мы напишем небольшую текстовую игру.

Герой нашей игры разбивает лагерь на опушке мистического леса. Из лагеря он ходит в экспедиции и сражается с врагами. 
За победы получает опыт, золото и добычу, растет в уровнях. Цель нашего героя - победить древнего дракона, притаившегося в своем логове.

## Игральная кость (Одиночка / Singleton)

В нашей игре многое решает случай, поэтому паттерном *одиночка* реализуем генератор
случайности - игральную кость, через которую будут проходить вообще все броски в игре.

*Одиночка* гарантирует, что у класса есть только один экземпляр, и дает к нему глобальную точку доступа. Нам это и
нужно: источник случайности должен быть один на всю программу, тогда, задав ему сид (начальное значение генератора),
можно в точности воспроизвести все броски экспедиции. Если бы каждый класс создавал свой
генератор случайных чисел, наш сид бы слетал.

```java
package org.misis.tp.lab3.core;

import java.util.List;
import java.util.Objects;
import java.util.Random;

// Одиночка: общий источник случайности на всю игру
public class Dice {

    // единственный экземпляр класса
    private static Dice instance;

    private final Random random;

    // конструктор приватный - самостоятельно создать экземпляр класса нельзя
    private Dice() {
        this.random = new Random();
    }

    // метод для получения одиночки
    public static Dice getInstance() {
        if (Objects.isNull(instance)) {
            instance = new Dice();
        }

        return instance;
    }

    // после установки сида все последующие броски повторяются в точности
    public void reseed(long seed) {
        random.setSeed(seed);
    }

    // случайный сид
    public long nextSeed() {
        return random.nextInt(1_000_000);
    }

    // случайное целое число от min до max включительно
    public int roll(int min, int max) {
        return min + random.nextInt(max - min + 1);
    }

    // возвращает true с вероятностью probability (от 0 до 1)
    public boolean chance(double probability) {
        return random.nextDouble() < probability;
    }

    // случайный элемент списка
    public <T> T pick(List<T> options) {
        return options.get(random.nextInt(options.size()));
    }

}
```

Давайте разберем, что в этой реализации важно.

1. Конструктор приватный. Создать кость через `new Dice()` нельзя, получить ее можно только через `getInstance()`.
2. Экземпляр хранится в самом классе и создается лениво, при первом обращении. Пока кость никому не нужна, ее нет.
3. Со звездочкой заметим, что наша реализация не потокобезопасна: если два потока одновременно впервые обратятся к кости, могут создаться два
   объекта. В игре поток один, поэтому нам это не страшно, но в многопоточной программе понадобилась бы синхронизация
   (или создание экземпляра сразу, а не лениво).
   Тему многопоточности мы здесь затрагивать не будем, но для самостоятельного изучения она крайне полезна.

Преимущество одиночки очевидно - у всех составляющих игры (оружия, врагов, локаций) одна и та же кость, поэтому один сид задает все броски. 
А еще кость не нужно передавать через конструкторы каждого предмета и каждого врага - к ней можно
обратиться откуда угодно.

Но тут есть и обратная сторона: одиночка - это глобальное состояние. Лук, посох и враги зависят от игральной кости, однако
по их конструкторам этого не видно, а подменить кость на другую (например, в тестах) не получится. Поэтому применять
одиночку стоит только там, где объект действительно должен быть единственным.

Также давайте сразу заведем исключение, которое будем бросать, если по коду ничего не нашлось (неизвестная локация,
неизвестный предмет и т.д.), в будущем оно нам пригодится:

```java
package org.misis.tp.lab3.core;

public class UnknownCodeException extends RuntimeException {

    public UnknownCodeException(String what, String code) {
        super(String.format("%s с кодом '%s' не существует", what, code));
    }

}
```

## Боевая механика

Давайте определим базовые абстракции нашей игры.

Интерфейс сущности, которая умеет атаковать и получать урон - так с точки зрения механики выглядят и герой, и враги:

```java
package org.misis.tp.lab3.combat;

// всё, что может участвовать в бою: и герой, и враги
public interface Combatant {

    // код вида сущности: класс героя или тип врага
    String getCode();

    String getName();

    int getHealth();

    int getMaxHealth();

    boolean isAlive();

    // удар, который будет нанесен данной сущностью
    Hit rollDamage();

    // получение удара, возвращает урон, который прошел на самом деле
    int receiveHit(int damage);

}
```

Удар представляет собой вот такую небольшую сущность:

```java
package org.misis.tp.lab3.combat;

// удар: сколько урона и был ли он критическим
public record Hit(int damage, boolean critical) {
}
```

`record` - это удобный способ работать с неизменяемыми объектами: Java сама создаст для такого класса конструктор, поля, методы
доступа (`damage()`, `critical()`), а также `equals`, `hashCode` и `toString`.

Давайте теперь опишем механику боя. Зафиксируем стороны, типы событий и исход:

```java
package org.misis.tp.lab3.combat;

public enum Side {

    HERO,
    ENEMY

}
```

```java
package org.misis.tp.lab3.combat;

public enum EventType {

    HIT,
    CRITICAL_HIT,
    DEATH

}
```

```java
package org.misis.tp.lab3.combat;

public enum Outcome {

    VICTORY,
    DEFEAT

}
```

Бой делится на раунды. Каждое событие раунда (удар, критический удар, гибель) запишем вот так - тут и пригодятся наши `enum`:

```java
package org.misis.tp.lab3.combat;

public record BattleEvent(int round, EventType type, Side source, Side target, int amount, int targetHealthAfter) {
}
```

Чтобы в отчете было видно, кто сражался, заведем класс участника боя: в нем все, что мы выводим о герое и его противнике:

```java
package org.misis.tp.lab3.combat;

// участник боя в том виде, в котором он попадает в отчет
public record Participant(String code, String name, int maxHealth) {

    public static Participant of(Combatant combatant) {
        return new Participant(combatant.getCode(), combatant.getName(), combatant.getMaxHealth());
    }

}
```

Супер! Теперь можем составить общий отчет о бое, в нем будут два участника (герой и его противник), исход боя, а также информация о событиях, 
получится что-то вот такое:

```java
package org.misis.tp.lab3.combat;

import java.util.List;

// результат боя: кто сражался, чем все закончилось и что происходило по ходу дела
public record BattleReport(Participant hero, Participant enemy, Outcome outcome, List<BattleEvent> events) {
}
```

А теперь наконец-то определим механику самого боя! В целом уже ясно, что идти она будет по раундам, 
в рамках которых герой и враг будут обмениваться ударами, давайте так и реализуем:

```java
package org.misis.tp.lab3.combat;

import java.util.ArrayList;
import java.util.List;

// бой идет по раундам: сначала бьет герой, потом враг, пока кто-то не погибнет
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
        int dealt = defender.receiveHit(hit.damage());
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

```java
package org.misis.tp.lab3.item;

import java.util.List;

// зачарование неизменяемо: его нельзя поменять после создания,
// поэтому одно и то же зачарование можно безопасно повесить на несколько предметов
public record Enchantment(String code, String name, int bonusDamage, double bonusDefense, int bonusPrice) {

    public static final Enchantment SHARP = new Enchantment("sharp", "Острое", 3, 0, 20);
    public static final Enchantment FIERY = new Enchantment("fiery", "Огненное", 6, 0, 50);
    public static final Enchantment STURDY = new Enchantment("sturdy", "Прочное", 0, 0.05, 20);
    public static final Enchantment WARDING = new Enchantment("warding", "Охранное", 0, 0.10, 50);

    public static final List<Enchantment> FOR_WEAPONS = List.of(SHARP, FIERY);
    public static final List<Enchantment> FOR_ARMOR = List.of(STURDY, WARDING);

}
```

В базовый класс предмета заложим и контракт `copy`:

```java
package org.misis.tp.lab3.item;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public abstract class Item {

    // код шаблона, из которого получен предмет
    private final String code;
    private final String name;
    private final int basePrice;
    private final List<Enchantment> enchantments;

    protected Item(String code, String name, int basePrice) {
        this.code = code;
        this.name = name;
        this.basePrice = basePrice;
        this.enchantments = new ArrayList<>();
    }

    // конструктор копирования, его вызывают наследники в методе copy()
    protected Item(Item source) {
        this.code = source.code;
        this.name = source.name;
        this.basePrice = source.basePrice;
        // список копируем, иначе копия и оригинал делили бы один и тот же объект списка зачарований
        this.enchantments = new ArrayList<>(source.enchantments);
    }

    // каждый наследник по-своему реализует копирование
    public abstract Item copy();

    // какие зачарования подходят этому предмету
    public abstract List<Enchantment> getPossibleEnchantments();

    // описание характеристик предмета, например "урон 12"
    public abstract String getDescription();

    public void enchant(Enchantment enchantment) {
        if (!getPossibleEnchantments().contains(enchantment)) {
            throw new IllegalArgumentException(
                String.format("Зачарование '%s' не подходит для предмета '%s'", enchantment.name(), name)
            );
        }

        enchantments.add(enchantment);
    }

    public String getCode() {
        return code;
    }

    public String getName() {
        return name;
    }

    public int getPrice() {
        int price = basePrice;
        for (Enchantment enchantment : enchantments) {
            price += enchantment.bonusPrice();
        }

        return price;
    }

    public List<Enchantment> getEnchantments() {
        return Collections.unmodifiableList(enchantments);
    }

}
```

Обратите внимание на конструктор копирования. Если бы мы написали `this.enchantments = source.enchantments`, то копия и
оригинал ссылались бы на один и тот же список (поверхностная копия), и зачарование, наложенное на копию, появилось бы и
на шаблоне в каталоге. Поэтому для копии мы создаем новый список. А вот сами зачарования копировать не нужно - они неизменяемые, и копия может ссылаться на те же объекты.

Супер, общий контракт предметов понятен, давайте создадим контракт для оружия:

```java
package org.misis.tp.lab3.item;

import org.misis.tp.lab3.combat.Hit;

import java.util.List;

public abstract class Weapon extends Item {

    private final int damage;

    protected Weapon(String code, String name, int basePrice, int damage) {
        super(code, name, basePrice);
        this.damage = damage;
    }

    protected Weapon(Weapon source) {
        super(source);
        this.damage = source.damage;
    }

    // как именно оружие наносит урон, решают наследники
    public abstract Hit rollDamage();

    @Override
    public abstract Weapon copy();

    // урон с учетом зачарований
    public int getDamage() {
        int total = damage;
        for (Enchantment enchantment : getEnchantments()) {
            total += enchantment.bonusDamage();
        }

        return total;
    }

    @Override
    public List<Enchantment> getPossibleEnchantments() {
        return Enchantment.FOR_WEAPONS;
    }

}
```

Отлично, теперь наше первое оружие, меч. Его урон всегда одинаковый:

```java
package org.misis.tp.lab3.item;

import org.misis.tp.lab3.combat.Hit;

public class Sword extends Weapon {

    public Sword(String code, String name, int basePrice, int damage) {
        super(code, name, basePrice, damage);
    }

    private Sword(Sword source) {
        super(source);
    }

    @Override
    public Hit rollDamage() {
        return new Hit(getDamage(), false);
    }

    @Override
    public Sword copy() {
        return new Sword(this);
    }

    @Override
    public String getDescription() {
        return String.format("урон %d", getDamage());
    }

}
```

Лук же с некоторым шансом наносит критический, двойной урон:

```java
package org.misis.tp.lab3.item;

import org.misis.tp.lab3.combat.Hit;
import org.misis.tp.lab3.core.Dice;

public class Bow extends Weapon {

    private static final int CRITICAL_MULTIPLIER = 2;

    private final double criticalChance;

    public Bow(String code, String name, int basePrice, int damage, double criticalChance) {
        super(code, name, basePrice, damage);
        this.criticalChance = criticalChance;
    }

    private Bow(Bow source) {
        super(source);
        this.criticalChance = source.criticalChance;
    }

    @Override
    public Hit rollDamage() {
        if (Dice.getInstance().chance(criticalChance)) {
            return new Hit(getDamage() * CRITICAL_MULTIPLIER, true);
        }

        return new Hit(getDamage(), false);
    }

    @Override
    public Bow copy() {
        return new Bow(this);
    }

    @Override
    public String getDescription() {
        return String.format("урон %d, шанс крита %d%%", getDamage(), Math.round(criticalChance * 100));
    }

}
```

Урон магического посоха каждый раз немного разный:

```java
package org.misis.tp.lab3.item;

import org.misis.tp.lab3.combat.Hit;
import org.misis.tp.lab3.core.Dice;

// урон посоха непредсказуем: он может быть как больше, так и меньше базового
public class Staff extends Weapon {

    private final double scatter;

    public Staff(String code, String name, int basePrice, int damage, double scatter) {
        super(code, name, basePrice, damage);
        this.scatter = scatter;
    }

    private Staff(Staff source) {
        super(source);
        this.scatter = source.scatter;
    }

    @Override
    public Hit rollDamage() {
        return new Hit(Dice.getInstance().roll(getMinDamage(), getMaxDamage()), false);
    }

    @Override
    public Staff copy() {
        return new Staff(this);
    }

    @Override
    public String getDescription() {
        return String.format("урон %d-%d", getMinDamage(), getMaxDamage());
    }

    private int getMinDamage() {
        return (int) Math.round(getDamage() * (1 - scatter));
    }

    private int getMaxDamage() {
        return (int) Math.round(getDamage() * (1 + scatter));
    }

}
```

Дальше давайте создадим класс для брони, он у нас будет общий для всех ее видов:

```java
package org.misis.tp.lab3.item;

import java.util.List;

// броня поглощает долю входящего урона
public class Armor extends Item {

    // больше этой доли урона никакая броня не поглощает
    private static final double MAX_DEFENSE = 0.75;

    private final double defense;

    public Armor(String code, String name, int basePrice, double defense) {
        super(code, name, basePrice);
        this.defense = defense;
    }

    private Armor(Armor source) {
        super(source);
        this.defense = source.defense;
    }

    @Override
    public Armor copy() {
        return new Armor(this);
    }

    // доля поглощаемого урона с учетом зачарований
    public double getDefense() {
        double total = defense;
        for (Enchantment enchantment : getEnchantments()) {
            total += enchantment.bonusDefense();
        }

        return Math.min(total, MAX_DEFENSE);
    }

    @Override
    public List<Enchantment> getPossibleEnchantments() {
        return Enchantment.FOR_ARMOR;
    }

    @Override
    public String getDescription() {
        return String.format("защита %d%%", Math.round(getDefense() * 100));
    }

}
```

Каждый конкретный предмет (меч, лук, посох, броня) реализует `copy()` через свой приватный конструктор копирования, который сначала вызывает
конструктор копирования родителя, а затем копирует собственные поля. 
Так каждый класс сам отвечает за то, чтобы скопировать все свои поля.

Теперь давайте напишем каталог: он хранит шаблоны всех предметов игры и выдает по ним конкретные экземпляры оружия и
брони:

```java
package org.misis.tp.lab3.item;

import org.misis.tp.lab3.core.UnknownCodeException;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;

// каталог хранит шаблоны предметов, а наружу отдает только их копии
public class ItemCatalog {

    private final Map<String, Item> prototypes = new LinkedHashMap<>();

    public void register(Item prototype) {
        prototypes.put(prototype.getCode(), prototype);
    }

    // каталог не знает, какой конкретно класс у шаблона - копию создает сам шаблон
    public Item create(String code) {
        Item prototype = prototypes.get(code);
        if (Objects.isNull(prototype)) {
            throw new UnknownCodeException("Предмет", code);
        }

        return prototype.copy();
    }

    public Weapon createWeapon(String code) {
        Item item = create(code);
        if (item instanceof Weapon weapon) {
            return weapon;
        }

        throw new IllegalArgumentException(String.format("Предмет '%s' не является оружием", code));
    }

    public Armor createArmor(String code) {
        Item item = create(code);
        if (item instanceof Armor armor) {
            return armor;
        }

        throw new IllegalArgumentException(String.format("Предмет '%s' не является броней", code));
    }

    public List<String> getCodes() {
        return List.copyOf(prototypes.keySet());
    }

}
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

```java
package org.misis.tp.lab3.item;

// все предметы, которые есть в игре
public final class DefaultCatalog {

    // коды шаблонов: по ним предметы ищутся в каталоге
    // стартовое снаряжение
    public static final String TRAINING_SWORD = "training-sword";
    public static final String HUNTING_BOW = "hunting-bow";
    public static final String APPRENTICE_STAFF = "apprentice-staff";
    public static final String CHAINMAIL = "chainmail";
    public static final String LEATHER_ARMOR = "leather-armor";
    public static final String LINEN_ROBE = "linen-robe";

    // товары торговца
    public static final String IRON_SWORD = "iron-sword";
    public static final String ELVEN_BOW = "elven-bow";
    public static final String EMBER_STAFF = "ember-staff";
    public static final String PLATE_ARMOR = "plate-armor";

    // добыча
    public static final String GOBLIN_CLEAVER = "goblin-cleaver";
    public static final String BONE_BOW = "bone-bow";
    public static final String NECROMANCER_STAFF = "necromancer-staff";
    public static final String KNIGHT_BLADE = "knight-blade";
    public static final String CRYPT_MAIL = "crypt-mail";
    public static final String DAWN_BLADE = "dawn-blade";
    public static final String DRAGONSCALE_ARMOR = "dragonscale-armor";

    private DefaultCatalog() {
    }

    public static ItemCatalog create() {
        ItemCatalog catalog = new ItemCatalog();

        // стартовое снаряжение
        catalog.register(new Sword(TRAINING_SWORD, "Тренировочный меч", 20, 12));
        catalog.register(new Bow(HUNTING_BOW, "Охотничий лук", 20, 10, 0.3));
        catalog.register(new Staff(APPRENTICE_STAFF, "Посох ученика", 20, 14, 0.3));
        catalog.register(new Armor(CHAINMAIL, "Кольчуга", 40, 0.3));
        catalog.register(new Armor(LEATHER_ARMOR, "Кожаная броня", 30, 0.2));
        catalog.register(new Armor(LINEN_ROBE, "Льняная роба", 15, 0.1));

        // товары торговца
        catalog.register(new Sword(IRON_SWORD, "Железный меч", 80, 17));
        catalog.register(new Bow(ELVEN_BOW, "Эльфийский лук", 90, 14, 0.4));
        catalog.register(new Staff(EMBER_STAFF, "Тлеющий посох", 90, 20, 0.3));
        catalog.register(new Armor(PLATE_ARMOR, "Латы", 120, 0.4));

        // добыча
        catalog.register(new Sword(GOBLIN_CLEAVER, "Гоблинский тесак", 40, 15));
        catalog.register(new Bow(BONE_BOW, "Костяной лук", 140, 17, 0.4));
        catalog.register(new Staff(NECROMANCER_STAFF, "Посох некроманта", 150, 25, 0.35));
        catalog.register(new Sword(KNIGHT_BLADE, "Клинок рыцаря", 150, 23));
        catalog.register(new Armor(CRYPT_MAIL, "Кольчуга стража склепа", 160, 0.45));
        catalog.register(new Sword(DAWN_BLADE, "Клинок рассвета", 500, 35));
        catalog.register(new Armor(DRAGONSCALE_ARMOR, "Драконья чешуя", 500, 0.55));

        return catalog;
    }

}
```

## Стартовое снаряжение (Абстрактная фабрика / Abstract Factory)

По логике нашей игры мы хотим выдавать герою стартовое снаряжение в зависимости от его класса.
В целом это можно сделать простыми if'ами в коде основной программы, однако мы знаем, что в дальнейшем в игру будут
добавляться новые классы, поэтому здесь уместно будет применить шаблон *Абстрактная фабрика*.
Она позволяет создавать семейства связанных между собой объектов, не привязываясь к их конкретным классам. Семейство у
нас - это стартовое снаряжение класса.

В роли базовой фабрики выступит стартовый набор снаряжения:

```java
package org.misis.tp.lab3.kit;

import org.misis.tp.lab3.item.Armor;
import org.misis.tp.lab3.item.Weapon;

// Абстрактная фабрика: стартовый набор снаряжения
public interface StarterKitFactory {

    // возвращает некое оружие
    Weapon createWeapon();

    // возвращает некую броню
    Armor createArmor();

}
```

И для каждого из классов героя мы создадим свою реализацию этого набора: снаряжение каждого класса - это и есть
семейство объектов.

В наборе для воина будут лежать меч и кольчуга:

```java
package org.misis.tp.lab3.kit;

import org.misis.tp.lab3.item.Armor;
import org.misis.tp.lab3.item.DefaultCatalog;
import org.misis.tp.lab3.item.ItemCatalog;
import org.misis.tp.lab3.item.Weapon;

public class WarriorKitFactory implements StarterKitFactory {

    private final ItemCatalog catalog;

    public WarriorKitFactory(ItemCatalog catalog) {
        this.catalog = catalog;
    }

    @Override
    public Weapon createWeapon() {
        return catalog.createWeapon(DefaultCatalog.TRAINING_SWORD);
    }

    @Override
    public Armor createArmor() {
        return catalog.createArmor(DefaultCatalog.CHAINMAIL);
    }

}
```

В наборе для следопыта - лук и кожаная броня:

```java
package org.misis.tp.lab3.kit;

import org.misis.tp.lab3.item.Armor;
import org.misis.tp.lab3.item.DefaultCatalog;
import org.misis.tp.lab3.item.ItemCatalog;
import org.misis.tp.lab3.item.Weapon;

public class RangerKitFactory implements StarterKitFactory {

    private final ItemCatalog catalog;

    public RangerKitFactory(ItemCatalog catalog) {
        this.catalog = catalog;
    }

    @Override
    public Weapon createWeapon() {
        return catalog.createWeapon(DefaultCatalog.HUNTING_BOW);
    }

    @Override
    public Armor createArmor() {
        return catalog.createArmor(DefaultCatalog.LEATHER_ARMOR);
    }

}
```

Ну а для мага - посох и роба:

```java
package org.misis.tp.lab3.kit;

import org.misis.tp.lab3.item.Armor;
import org.misis.tp.lab3.item.DefaultCatalog;
import org.misis.tp.lab3.item.ItemCatalog;
import org.misis.tp.lab3.item.Weapon;

public class MageKitFactory implements StarterKitFactory {

    private final ItemCatalog catalog;

    public MageKitFactory(ItemCatalog catalog) {
        this.catalog = catalog;
    }

    @Override
    public Weapon createWeapon() {
        return catalog.createWeapon(DefaultCatalog.APPRENTICE_STAFF);
    }

    @Override
    public Armor createArmor() {
        return catalog.createArmor(DefaultCatalog.LINEN_ROBE);
    }

}
```

Классы героя опишем через `enum`, и каждый класс будет сам знать свою фабрику стартового снаряжения:

```java
package org.misis.tp.lab3.hero;

import org.misis.tp.lab3.core.UnknownCodeException;
import org.misis.tp.lab3.item.ItemCatalog;
import org.misis.tp.lab3.kit.MageKitFactory;
import org.misis.tp.lab3.kit.RangerKitFactory;
import org.misis.tp.lab3.kit.StarterKitFactory;
import org.misis.tp.lab3.kit.WarriorKitFactory;

import java.util.function.Function;

public enum HeroClass {

    // WarriorKitFactory::new - ссылка на конструктор фабрики,
    // то есть "функция, которая создает фабрику стартового снаряжения воина"
    WARRIOR("warrior", "Воин", 120, WarriorKitFactory::new),
    RANGER("ranger", "Следопыт", 100, RangerKitFactory::new),
    MAGE("mage", "Маг", 85, MageKitFactory::new);

    private final String code;
    private final String displayName;
    private final int baseHealth;
    private final Function<ItemCatalog, StarterKitFactory> starterKit;

    HeroClass(String code, String displayName, int baseHealth, Function<ItemCatalog, StarterKitFactory> starterKit) {
        this.code = code;
        this.displayName = displayName;
        this.baseHealth = baseHealth;
        this.starterKit = starterKit;
    }

    // каждый класс сам знает свою фабрику стартового снаряжения
    public StarterKitFactory getStarterKit(ItemCatalog catalog) {
        return starterKit.apply(catalog);
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

}
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

```java
package org.misis.tp.lab3.hero;

import org.misis.tp.lab3.combat.Combatant;
import org.misis.tp.lab3.combat.Hit;
import org.misis.tp.lab3.item.Armor;
import org.misis.tp.lab3.item.Item;
import org.misis.tp.lab3.item.Weapon;
import org.misis.tp.lab3.kit.StarterKitFactory;

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

    // конструктор принимает в себя билдера, из которого и берет значения полей
    private Hero(Builder builder) {
        this.name = builder.name;
        this.heroClass = builder.heroClass;
        this.level = builder.level;
        this.experience = 0;
        this.gold = builder.gold;
        this.weapon = builder.weapon;
        this.armor = builder.armor;
        this.inventory = new ArrayList<>();
        this.health = getMaxHealth();
    }

    // внутренний класс билдер - чтобы удобно и безопасно создавать героев
    public static class Builder {

        private String name;
        private HeroClass heroClass;
        private Weapon weapon;
        private Armor armor;
        // необязательные поля сразу получают значения по умолчанию
        private int level = 1;
        private int gold = STARTING_GOLD;

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

        // берет оружие и броню из стартового набора
        public Builder setStarterKit(StarterKitFactory starterKit) {
            this.weapon = starterKit.createWeapon();
            this.armor = starterKit.createArmor();
            return this;
        }

        public Builder setLevel(int level) {
            this.level = level;
            return this;
        }

        public Builder setGold(int gold) {
            this.gold = gold;
            return this;
        }

        // перед созданием героя проверяем, что все обязательные поля заполнены корректно
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
    public int receiveHit(int damage) {
        int reducedDamage = (int) Math.round(damage * (1 - armor.getDefense()));
        // хотя бы единицу урона броня пропускает всегда
        int dealt = Math.max(1, reducedDamage);
        health = Math.max(0, health - dealt);
        return dealt;
    }

    @Override
    public boolean isAlive() {
        return health > 0;
    }

    // поведение героя между экспедициями

    // перед каждой экспедицией герой успевает отдохнуть
    public void restoreHealth() {
        health = getMaxHealth();
    }

    // начисляет опыт, возвращает количество полученных уровней
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

    // экипирует предмет из инвентаря, снятый предмет возвращается в инвентарь
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

    // геттеры

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

}
```

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

```java
package org.misis.tp.lab3.enemy;

import org.misis.tp.lab3.combat.Combatant;

public interface Enemy extends Combatant {

    int getExperienceReward();

    int rollGoldReward();

}
```

Общая логика врагов вынесена в абстрактный класс:

```java
package org.misis.tp.lab3.enemy;

import org.misis.tp.lab3.combat.Hit;
import org.misis.tp.lab3.core.Dice;

// общая логика для врагов: здоровье, урон в диапазоне, награда
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
    public int receiveHit(int damage) {
        int dealt = Math.max(1, reduceDamage(damage));
        health = Math.max(0, health - dealt);
        return dealt;
    }

    // наследники могут уменьшать входящий урон
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

Определим злого гоблина, самого слабого врага, живущего в лесу:

```java
package org.misis.tp.lab3.enemy;

// гоблин - самый простой враг, без особенностей
public class Goblin extends AbstractEnemy {

    public Goblin() {
        super("goblin", "Гоблин", 50, 6, 14, 40, 30);
    }

}
```

Теперь скелет, он уже чуть страшнее гоблина:

```java
package org.misis.tp.lab3.enemy;

// скелет: кости плохо поддаются ударам, поэтому часть урона проходит мимо
public class Skeleton extends AbstractEnemy {

    private static final double RESISTANCE = 0.3;

    public Skeleton() {
        super("skeleton", "Скелет", 110, 10, 24, 90, 70);
    }

    @Override
    protected int reduceDamage(int damage) {
        return (int) Math.round(damage * (1 - RESISTANCE));
    }

}
```

И ужасный дракон: он наносит страшный урон огнем, а крепкая чешуя гасит часть урона:

```java
package org.misis.tp.lab3.enemy;

import org.misis.tp.lab3.combat.Hit;
import org.misis.tp.lab3.core.Dice;

// дракон защищен чешуей и время от времени дышит огнем
public class Dragon extends AbstractEnemy {

    private static final double RESISTANCE = 0.25;
    private static final double FIRE_BREATH_CHANCE = 0.25;
    private static final int FIRE_BREATH_MULTIPLIER = 2;

    public Dragon() {
        super("dragon", "Дракон", 260, 16, 22, 500, 400);
    }

    @Override
    public Hit rollDamage() {
        Hit bite = super.rollDamage();
        if (Dice.getInstance().chance(FIRE_BREATH_CHANCE)) {
            return new Hit(bite.damage() * FIRE_BREATH_MULTIPLIER, true);
        }

        return bite;
    }

    @Override
    protected int reduceDamage(int damage) {
        return (int) Math.round(damage * (1 - RESISTANCE));
    }

}
```

По итогам экспедиции герой получает награды: опыт, золото, добычу и новые уровни. Запишем их так:

```java
package org.misis.tp.lab3.world;

import org.misis.tp.lab3.item.Item;

// итоги экспедиции для героя;
// gold отрицательный, если герой проиграл и потерял золото, loot равен null, если добычи нет
public record Rewards(int experience, int gold, Item loot, int levelsGained) {
}
```

Ну а общий итог экспедиции фиксируем вот так:

```java
package org.misis.tp.lab3.world;

import org.misis.tp.lab3.combat.BattleReport;

// отчет об экспедиции: с каким сидом, куда, как прошел бой и что получил герой
public record ExpeditionReport(long seed, String locationCode, BattleReport battle, Rewards rewards) {
}
```

Дальше давайте определим абстрактную локацию и общую логику экспедиции. Конкретных врагов здесь не создаем - это работа
подклассов:

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

        hero.restoreHealth();
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

Также обратите внимание, что добыча здесь - копия шаблона из каталога, и зачарование накладывается именно на копию.

Отлично, теперь создаем конкретные локации:

Мистический лес, кишащий злыми, но слабыми гоблинами:

```java
package org.misis.tp.lab3.world;

import org.misis.tp.lab3.enemy.Enemy;
import org.misis.tp.lab3.enemy.Goblin;
import org.misis.tp.lab3.item.DefaultCatalog;
import org.misis.tp.lab3.item.ItemCatalog;

import java.util.List;

public class Forest extends Location {

    public static final String CODE = "forest";

    public Forest(ItemCatalog catalog) {
        super(CODE, "Мистический лес", 1, catalog);
    }

    @Override
    protected Enemy createEnemy() {
        return new Goblin();
    }

    @Override
    protected List<String> getLootTable() {
        return List.of(DefaultCatalog.GOBLIN_CLEAVER, DefaultCatalog.HUNTING_BOW, DefaultCatalog.LEATHER_ARMOR);
    }

}
```

Заброшенный склеп, в котором бродят ожившие скелеты:

```java
package org.misis.tp.lab3.world;

import org.misis.tp.lab3.enemy.Enemy;
import org.misis.tp.lab3.enemy.Skeleton;
import org.misis.tp.lab3.item.DefaultCatalog;
import org.misis.tp.lab3.item.ItemCatalog;

import java.util.List;

public class Crypt extends Location {

    public static final String CODE = "crypt";

    public Crypt(ItemCatalog catalog) {
        super(CODE, "Заброшенный склеп", 3, catalog);
    }

    @Override
    protected Enemy createEnemy() {
        return new Skeleton();
    }

    @Override
    protected List<String> getLootTable() {
        return List.of(
            DefaultCatalog.KNIGHT_BLADE, DefaultCatalog.BONE_BOW, DefaultCatalog.NECROMANCER_STAFF, DefaultCatalog.CRYPT_MAIL
        );
    }

}
```

И логово дракона, прибежище жуткого чешуйчатого чудища:

```java
package org.misis.tp.lab3.world;

import org.misis.tp.lab3.enemy.Dragon;
import org.misis.tp.lab3.enemy.Enemy;
import org.misis.tp.lab3.item.DefaultCatalog;
import org.misis.tp.lab3.item.ItemCatalog;

import java.util.List;

public class DragonLair extends Location {

    public static final String CODE = "dragon-lair";

    public DragonLair(ItemCatalog catalog) {
        super(CODE, "Логово дракона", 5, catalog);
    }

    @Override
    protected Enemy createEnemy() {
        return new Dragon();
    }

    @Override
    protected List<String> getLootTable() {
        return List.of(DefaultCatalog.DAWN_BLADE, DefaultCatalog.DRAGONSCALE_ARMOR);
    }

}
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

```java
package org.misis.tp.lab3.world;

import org.misis.tp.lab3.core.UnknownCodeException;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;

// все локации игры
public class LocationRegistry {

    private final Map<String, Location> locations = new LinkedHashMap<>();

    public LocationRegistry(List<Location> locations) {
        for (Location location : locations) {
            this.locations.put(location.getCode(), location);
        }
    }

    public Location get(String code) {
        Location location = locations.get(code);
        if (Objects.isNull(location)) {
            throw new UnknownCodeException("Локация", code);
        }

        return location;
    }

    public List<Location> getAll() {
        return List.copyOf(locations.values());
    }

}
```

## Лагерь

Между экспедициями герой отдыхает в лагере, где есть торговец (а куда же без него). 
Торговец продает предметы из каталога и скупает ненужное за половину цены:

```java
package org.misis.tp.lab3.camp;

import org.misis.tp.lab3.core.UnknownCodeException;
import org.misis.tp.lab3.hero.Hero;
import org.misis.tp.lab3.item.Item;
import org.misis.tp.lab3.item.ItemCatalog;

import java.util.ArrayList;
import java.util.List;

public class Merchant {

    private static final int SELL_PRICE_PERCENT = 50;

    private final ItemCatalog catalog;
    private final List<String> assortment;

    public Merchant(ItemCatalog catalog, List<String> assortment) {
        this.catalog = catalog;
        this.assortment = List.copyOf(assortment);
    }

    // образцы товаров, чтобы показать их покупателю
    public List<Item> getShowcase() {
        List<Item> showcase = new ArrayList<>();
        for (String code : assortment) {
            showcase.add(catalog.create(code));
        }

        return showcase;
    }

    public Item buy(Hero hero, String code) {
        if (!assortment.contains(code)) {
            throw new UnknownCodeException("Товар", code);
        }

        // покупатель получает собственную копию предмета из каталога
        Item item = catalog.create(code);
        hero.spendGold(item.getPrice());
        hero.addItem(item);
        return item;
    }

    public int sell(Hero hero, Item item) {
        int price = getSellPrice(item);
        hero.removeItem(item);
        hero.addGold(price);
        return price;
    }

    public int getSellPrice(Item item) {
        return item.getPrice() * SELL_PRICE_PERCENT / 100;
    }

}
```

## Консоль

Вся игровая логика наконец-то готова, осталось научить программу общаться с игроком. Весь вывод соберем в одном классе - он
превращает героя, предметы и отчеты об экспедиции в текст для консоли. 

Отдельный класс нужен потому, что дальше с игрой можно будет работать не только через консоль (спойлер-спойлер), и
привязывать к ней игровую логику не хочется.

Вот он:

```java
package org.misis.tp.lab3;

import org.misis.tp.lab3.combat.BattleEvent;
import org.misis.tp.lab3.combat.BattleReport;
import org.misis.tp.lab3.combat.Outcome;
import org.misis.tp.lab3.combat.Participant;
import org.misis.tp.lab3.combat.Side;
import org.misis.tp.lab3.hero.Hero;
import org.misis.tp.lab3.item.Enchantment;
import org.misis.tp.lab3.item.Item;
import org.misis.tp.lab3.world.ExpeditionReport;
import org.misis.tp.lab3.world.Rewards;

import java.util.ArrayList;
import java.util.List;

// весь вывод в консоль собран здесь
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

    public void printHero(Hero hero) {
        System.out.printf(
            "%s, %s %d ур. | здоровье %d | опыт %d/%d | золото %d%n",
            hero.getName(), hero.getHeroClass().getDisplayName(), hero.getLevel(),
            hero.getMaxHealth(), hero.getExperience(), hero.getExperienceToNextLevel(), hero.getGold()
        );
        System.out.println("  Оружие: " + describe(hero.getWeapon()));
        System.out.println("  Броня:  " + describe(hero.getArmor()));
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

Осталось написать игровой цикл - он будет в методе `main`:

```java
package org.misis.tp.lab3;

import org.misis.tp.lab3.camp.Merchant;
import org.misis.tp.lab3.combat.Outcome;
import org.misis.tp.lab3.hero.Hero;
import org.misis.tp.lab3.hero.HeroClass;
import org.misis.tp.lab3.item.DefaultCatalog;
import org.misis.tp.lab3.item.Item;
import org.misis.tp.lab3.item.ItemCatalog;
import org.misis.tp.lab3.kit.StarterKitFactory;
import org.misis.tp.lab3.world.Crypt;
import org.misis.tp.lab3.world.DragonLair;
import org.misis.tp.lab3.world.ExpeditionReport;
import org.misis.tp.lab3.world.Forest;
import org.misis.tp.lab3.world.Location;
import org.misis.tp.lab3.world.LocationRegistry;

import java.io.Console;
import java.nio.charset.Charset;
import java.util.List;
import java.util.Scanner;

public class Main {

    private static final Scanner INPUT = new Scanner(System.in, inputCharset());
    private static final ConsoleView VIEW = new ConsoleView();

    public static void main(String[] args) {
        ItemCatalog catalog = DefaultCatalog.create();
        LocationRegistry locations = new LocationRegistry(List.of(
            new Forest(catalog),
            new Crypt(catalog),
            new DragonLair(catalog)
        ));
        Merchant merchant = new Merchant(catalog, List.of(
            DefaultCatalog.IRON_SWORD, DefaultCatalog.ELVEN_BOW, DefaultCatalog.EMBER_STAFF,
            DefaultCatalog.CHAINMAIL, DefaultCatalog.PLATE_ARMOR
        ));

        Hero hero = createHero(catalog);
        System.out.printf("%s разбивает лагерь на опушке мистического леса.%n", hero.getName());

        while (true) {
            System.out.println();
            VIEW.printHero(hero);
            System.out.println("1 - отправиться в экспедицию, 2 - снаряжение, 3 - торговец, 0 - выйти из игры");
            switch (INPUT.nextLine().strip()) {
                case "1" -> goOnExpedition(hero, locations);
                case "2" -> manageEquipment(hero);
                case "3" -> visitMerchant(hero, merchant);
                case "0" -> {
                    return;
                }
                default -> System.out.println("Такого варианта нет");
            }
        }
    }

    private static Hero createHero(ItemCatalog catalog) {
        System.out.println("Создайте своего героя!");
        while (true) {
            System.out.println("Выберите класс:");
            HeroClass[] heroClasses = HeroClass.values();
            for (int i = 0; i < heroClasses.length; i++) {
                StarterKitFactory kit = heroClasses[i].getStarterKit(catalog);
                System.out.printf(
                    "%d - %s (здоровье %d): %s, %s%n",
                    i + 1, heroClasses[i].getDisplayName(), heroClasses[i].getBaseHealth(),
                    VIEW.describe(kit.createWeapon()), VIEW.describe(kit.createArmor())
                );
            }
            int classIndex = readChoice(heroClasses.length);
            if (classIndex < 0) {
                continue;
            }
            HeroClass heroClass = heroClasses[classIndex];

            System.out.println("Введите имя:");
            String name = INPUT.nextLine().strip();

            try {
                return new Hero.Builder()
                    .setName(name)
                    .setHeroClass(heroClass)
                    .setStarterKit(heroClass.getStarterKit(catalog))
                    .build();
            } catch (IllegalArgumentException e) {
                System.out.println("Не получилось создать героя: " + e.getMessage());
            }
        }
    }

    private static void goOnExpedition(Hero hero, LocationRegistry locations) {
        List<Location> all = locations.getAll();
        System.out.println("Куда отправимся?");
        for (int i = 0; i < all.size(); i++) {
            System.out.printf(
                "%d - %s (рекомендуемый уровень %d)%n",
                i + 1, all.get(i).getDisplayName(), all.get(i).getRecommendedLevel()
            );
        }
        int index = readChoice(all.size());
        if (index < 0) {
            return;
        }

        Location location = all.get(index);
        ExpeditionReport report = location.explore(hero);
        System.out.println();
        VIEW.printReport(report, location.getDisplayName());

        if (report.battle().outcome() == Outcome.VICTORY && DragonLair.CODE.equals(location.getCode())) {
            System.out.printf("%nДракон повержен! Барды еще долго будут петь о подвиге героя по имени %s.%n", hero.getName());
        }
    }

    private static void manageEquipment(Hero hero) {
        List<Item> inventory = hero.getInventory();
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

        Item item = inventory.get(index);
        hero.equip(item);
        System.out.println("Экипировано: " + VIEW.describe(item));
    }

    private static void visitMerchant(Hero hero, Merchant merchant) {
        System.out.printf("Торговец: \"Чего желаешь?\" (золото: %d)%n", hero.getGold());
        System.out.println("1 - купить, 2 - продать, 0 - назад");
        switch (INPUT.nextLine().strip()) {
            case "1" -> buy(hero, merchant);
            case "2" -> sell(hero, merchant);
            default -> {
            }
        }
    }

    private static void buy(Hero hero, Merchant merchant) {
        List<Item> showcase = merchant.getShowcase();
        for (int i = 0; i < showcase.size(); i++) {
            System.out.printf("%d - %s: %d золота%n", i + 1, VIEW.describe(showcase.get(i)), showcase.get(i).getPrice());
        }
        int index = readChoice(showcase.size());
        if (index < 0) {
            return;
        }

        try {
            Item item = merchant.buy(hero, showcase.get(index).getCode());
            System.out.println("Куплено: " + VIEW.describe(item));
        } catch (IllegalStateException e) {
            System.out.println(e.getMessage());
        }
    }

    private static void sell(Hero hero, Merchant merchant) {
        List<Item> inventory = hero.getInventory();
        if (inventory.isEmpty()) {
            System.out.println("Продавать нечего (экипированные предметы продать нельзя)");
            return;
        }

        for (int i = 0; i < inventory.size(); i++) {
            System.out.printf(
                "%d - %s: %d золота%n", i + 1, VIEW.describe(inventory.get(i)), merchant.getSellPrice(inventory.get(i))
            );
        }
        int index = readChoice(inventory.size());
        if (index < 0) {
            return;
        }

        Item item = inventory.get(index);
        int price = merchant.sell(hero, item);
        System.out.printf("Продано: %s за %d золота%n", item.getName(), price);
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

    // читает номер варианта от 1 до count, возвращает индекс или -1, если ввели что-то другое
    private static int readChoice(int count) {
        try {
            int choice = Integer.parseInt(INPUT.nextLine().strip());
            if (choice >= 1 && choice <= count) {
                return choice - 1;
            }
        } catch (NumberFormatException ignored) {
            // некорректный ввод обрабатываем так же, как отказ от выбора
        }

        return -1;
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

Чтобы показать все паттерны одним запуском, напишем класс `Demo`:

```java
package org.misis.tp.lab3;

import org.misis.tp.lab3.core.Dice;
import org.misis.tp.lab3.hero.Hero;
import org.misis.tp.lab3.hero.HeroClass;
import org.misis.tp.lab3.item.DefaultCatalog;
import org.misis.tp.lab3.item.Enchantment;
import org.misis.tp.lab3.item.Item;
import org.misis.tp.lab3.item.ItemCatalog;
import org.misis.tp.lab3.kit.StarterKitFactory;
import org.misis.tp.lab3.world.Crypt;
import org.misis.tp.lab3.world.Forest;

public class Demo {

    public static void main(String[] args) {
        ItemCatalog catalog = DefaultCatalog.create();
        ConsoleView view = new ConsoleView();

        System.out.println("===== Одиночка =====");
        Dice first = Dice.getInstance();
        Dice second = Dice.getInstance();
        System.out.println("Dice.getInstance() возвращает один и тот же объект: " + (first == second));

        System.out.println();
        System.out.println("===== Прототип =====");
        Item plainSword = catalog.create(DefaultCatalog.IRON_SWORD);
        Item fierySword = catalog.create(DefaultCatalog.IRON_SWORD);
        fierySword.enchant(Enchantment.FIERY);
        System.out.println("Первая копия:        " + view.describe(plainSword));
        System.out.println("Вторая копия:        " + view.describe(fierySword));
        System.out.println("Новая копия шаблона: " + view.describe(catalog.create(DefaultCatalog.IRON_SWORD)));
        System.out.println("Копии - разные объекты: " + (plainSword != fierySword));

        System.out.println();
        System.out.println("===== Абстрактная фабрика =====");
        for (HeroClass heroClass : HeroClass.values()) {
            StarterKitFactory kit = heroClass.getStarterKit(catalog);
            System.out.printf(
                "%s: %s, %s%n",
                heroClass.getDisplayName(), view.describe(kit.createWeapon()), view.describe(kit.createArmor())
            );
        }

        System.out.println();
        System.out.println("===== Строитель =====");
        Hero hero = new Hero.Builder()
            .setName("Артур")
            .setHeroClass(HeroClass.WARRIOR)
            .setStarterKit(HeroClass.WARRIOR.getStarterKit(catalog))
            .setLevel(3)
            .build();
        view.printHero(hero);
        try {
            new Hero.Builder()
                .setHeroClass(HeroClass.MAGE)
                .setStarterKit(HeroClass.MAGE.getStarterKit(catalog))
                .build();
        } catch (IllegalArgumentException e) {
            System.out.println("Герой без имени не создается: " + e.getMessage());
        }
        try {
            new Hero.Builder()
                .setName("Мерлин")
                .build();
        } catch (IllegalArgumentException e) {
            System.out.println("Герой без класса не создается: " + e.getMessage());
        }

        System.out.println();
        System.out.println("===== Фабричный метод =====");
        Forest forest = new Forest(catalog);
        Crypt crypt = new Crypt(catalog);
        view.printReport(forest.explore(hero, 7), forest.getDisplayName());
        System.out.println();
        view.printReport(crypt.explore(hero, 7), crypt.getDisplayName());
    }

}
```

Вывод программы:

```
===== Одиночка =====
Dice.getInstance() возвращает один и тот же объект: true

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
[раунд 1] Гоблин -> Артур: 9 урона, осталось 141/150
[раунд 2] Артур -> Гоблин: 18 урона, осталось 14/50
[раунд 2] Гоблин -> Артур: 6 урона, осталось 135/150
[раунд 3] Артур -> Гоблин: 18 урона, осталось 0/50
Гоблин повержен!
Победа! +40 опыта, +26 золота
Добыча: Охотничий лук (урон 10, шанс крита 30%)

=== Заброшенный склеп (сид 7) ===
Артур встречает врага: Скелет (110 здоровья)
[раунд 1] Артур -> Скелет: 13 урона, осталось 97/110
[раунд 1] Скелет -> Артур: 8 урона, осталось 142/150
[раунд 2] Артур -> Скелет: 13 урона, осталось 84/110
[раунд 2] Скелет -> Артур: 17 урона, осталось 125/150
[раунд 3] Артур -> Скелет: 13 урона, осталось 71/110
[раунд 3] Скелет -> Артур: 7 урона, осталось 118/150
[раунд 4] Артур -> Скелет: 13 урона, осталось 58/110
[раунд 4] Скелет -> Артур: 10 урона, осталось 108/150
[раунд 5] Артур -> Скелет: 13 урона, осталось 45/110
[раунд 5] Скелет -> Артур: 14 урона, осталось 94/150
[раунд 6] Артур -> Скелет: 13 урона, осталось 32/110
[раунд 6] Скелет -> Артур: 10 урона, осталось 84/150
[раунд 7] Артур -> Скелет: 13 урона, осталось 19/110
[раунд 7] Скелет -> Артур: 16 урона, осталось 68/150
[раунд 8] Артур -> Скелет: 13 урона, осталось 6/110
[раунд 8] Скелет -> Артур: 17 урона, осталось 51/150
[раунд 9] Артур -> Скелет: 13 урона, осталось 0/110
Скелет повержен!
Победа! +90 опыта, +53 золота
Добыча: Кольчуга стража склепа [Охранное] (защита 55%)
```

Не забудьте закоммитить и запушить изменения!

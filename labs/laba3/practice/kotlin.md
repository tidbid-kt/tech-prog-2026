# Лаба 3 — практика на Kotlin

Это практическая часть [лабораторной работы 3](../laba3.md): требования к сдаче и вопросы к защите находятся в
основном тексте лабы, а здесь шаг за шагом разобрано эталонное решение.

Понадобится JDK 17 или новее и IntelliJ IDEA, проект собирается через Maven. Шаблон репозитория: [tech-prog-labs-2026-kotlin](https://github.com/tidbid-kt/tech-prog-labs-2026-kotlin). Код лабы пишите в пакете `labs.lab3`. Запускать программы проще всего зеленой кнопкой рядом с `main` в IDEA, а из терминала - так: игра - `mvn -q compile exec:java -Dexec.mainClass=labs.lab3.MainKt`, демонстрация - `mvn -q compile exec:java -Dexec.mainClass=labs.lab3.Demo`.

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

```kotlin
package labs.lab3.core

import kotlin.random.Random

// Одиночка: общий источник случайности на всю игру.
// object - это класс, у которого может быть только один экземпляр: его создает сам язык при первом обращении
object Dice {

    private var random: Random = Random.Default

    // после установки сида все последующие броски повторяются в точности
    fun reseed(seed: Int) {
        random = Random(seed)
    }

    // случайный сид
    fun nextSeed(): Int = random.nextInt(1_000_000)

    // случайное целое число от min до max включительно
    fun roll(min: Int, max: Int): Int = random.nextInt(min, max + 1)

    // возвращает true с вероятностью probability (от 0 до 1)
    fun chance(probability: Double): Boolean = random.nextDouble() < probability

    // случайный элемент списка
    fun <T> pick(options: List<T>): T = options[random.nextInt(options.size)]
}
```

В Kotlin одиночка встроен в сам язык: объявление `object` создает класс и сразу его единственный экземпляр. Конструктора
у `object` нет вообще, а обращаются к нему просто по имени: `Dice.roll(1, 6)`.

Давайте разберем, что в этой реализации важно.

1. Конструктора у `object` нет, поэтому создать кость самостоятельно нельзя, а обратиться к ней можно только по имени
   `Dice`.
2. Экземпляр создается лениво, при первом обращении к `Dice`. Пока кость никому не нужна, ее нет.
3. Со звездочкой заметим, что такая реализация потокобезопасна: объект создается при инициализации класса, а JVM гарантирует, что класс будет
   инициализирован ровно один раз, даже если к нему одновременно обратятся несколько потоков. Тему многопоточности мы здесь затрагивать не будем, но для самостоятельного изучения она очень интересна.

Преимущество одиночки очевидно - у всех составляющих игры (оружия, врагов, локаций) одна и та же кость, поэтому один сид задает все броски. 
А еще кость не нужно передавать через конструкторы каждого предмета и каждого врага - к ней можно
обратиться откуда угодно.

Но тут есть и обратная сторона: одиночка - это глобальное состояние. Лук, посох и враги зависят от игральной кости, однако
по их конструкторам этого не видно, а подменить кость на другую (например, в тестах) не получится. Поэтому применять
одиночку стоит только там, где объект действительно должен быть единственным.

Также давайте сразу заведем исключение, которое будем бросать, если по коду ничего не нашлось (неизвестная локация,
неизвестный предмет и т.д.), в будущем оно нам пригодится:

```kotlin
package labs.lab3.core

class UnknownCodeException(what: String, code: String) : RuntimeException("$what с кодом '$code' не существует")
```

## Боевая механика

Давайте определим базовые абстракции нашей игры.

Интерфейс сущности, которая умеет атаковать и получать урон - так с точки зрения механики выглядят и герой, и враги:

```kotlin
package labs.lab3.combat

// всё, что может участвовать в бою: и герой, и враги
interface Combatant {

    // код вида сущности: класс героя или тип врага
    val code: String

    val name: String

    val health: Int

    val maxHealth: Int

    val isAlive: Boolean

    // удар, который будет нанесен данной сущностью
    fun rollDamage(): Hit

    // получение удара, возвращает урон, который прошел на самом деле
    fun receiveHit(damage: Int): Int
}
```

Вместо методов-геттеров интерфейс объявляет свойства (`val code: String`): снаружи их читают как поля, а реализовать
свойство можно как обычным полем, так и вычисляемым значением.

Удар представляет собой вот такую небольшую сущность:

```kotlin
package labs.lab3.combat

// удар: сколько урона и был ли он критическим
data class Hit(val damage: Int, val critical: Boolean)
```

`data class` - это короткая запись класса для хранения данных: Kotlin сам создаст для него `equals`, `hashCode`,
`toString` и метод `copy`. Обратите внимание: `copy` у `data class` копирует поверхностно - если бы внутри был
изменяемый список, копия и оригинал делили бы его. Поэтому копирование предметов ниже мы реализуем сами.

Давайте теперь опишем механику боя. Зафиксируем стороны, типы событий и исход:

```kotlin
package labs.lab3.combat

enum class Side {
    HERO,
    ENEMY,
}
```

```kotlin
package labs.lab3.combat

enum class EventType {
    HIT,
    CRITICAL_HIT,
    DEATH,
}
```

```kotlin
package labs.lab3.combat

enum class Outcome {
    VICTORY,
    DEFEAT,
}
```

Бой делится на раунды. Каждое событие раунда (удар, критический удар, гибель) запишем вот так - тут и пригодятся наши `enum`:

```kotlin
package labs.lab3.combat

data class BattleEvent(
    val round: Int,
    val type: EventType,
    val source: Side,
    val target: Side,
    val amount: Int,
    val targetHealthAfter: Int,
)
```

Чтобы в отчете было видно, кто сражался, заведем класс участника боя: в нем все, что мы выводим о герое и его противнике:

```kotlin
package labs.lab3.combat

// участник боя в том виде, в котором он попадает в отчет
data class Participant(val code: String, val name: String, val maxHealth: Int) {

    companion object {
        fun of(combatant: Combatant): Participant =
            Participant(combatant.code, combatant.name, combatant.maxHealth)
    }
}
```

Супер! Теперь можем составить общий отчет о бое, в нем будут два участника (герой и его противник), исход боя, а также информация о событиях, 
получится что-то вот такое:

```kotlin
package labs.lab3.combat

// результат боя: кто сражался, чем все закончилось и что происходило по ходу дела
data class BattleReport(
    val hero: Participant,
    val enemy: Participant,
    val outcome: Outcome,
    val events: List<BattleEvent>,
)
```

А теперь наконец-то определим механику самого боя! В целом уже ясно, что идти она будет по раундам, 
в рамках которых герой и враг будут обмениваться ударами, давайте так и реализуем:

```kotlin
package labs.lab3.combat

// бой идет по раундам: сначала бьет герой, потом враг, пока кто-то не погибнет
class Battle(private val hero: Combatant, private val enemy: Combatant) {

    private val events = mutableListOf<BattleEvent>()
    private var round = 0

    fun run(): BattleReport {
        while (hero.isAlive && enemy.isAlive) {
            round++
            strike(hero, Side.HERO, enemy, Side.ENEMY)
            if (enemy.isAlive) {
                strike(enemy, Side.ENEMY, hero, Side.HERO)
            }
        }

        val outcome = if (hero.isAlive) Outcome.VICTORY else Outcome.DEFEAT
        return BattleReport(Participant.of(hero), Participant.of(enemy), outcome, events.toList())
    }

    private fun strike(attacker: Combatant, source: Side, defender: Combatant, target: Side) {
        val hit = attacker.rollDamage()
        val dealt = defender.receiveHit(hit.damage)
        val type = if (hit.critical) EventType.CRITICAL_HIT else EventType.HIT
        emit(BattleEvent(round, type, source, target, dealt, defender.health))

        if (!defender.isAlive) {
            emit(BattleEvent(round, EventType.DEATH, source, target, 0, 0))
        }
    }

    private fun emit(event: BattleEvent) {
        events.add(event)
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

```kotlin
package labs.lab3.item

// зачарование неизменяемо: все свойства объявлены через val и не меняются после создания,
// поэтому одно и то же зачарование можно безопасно повесить на несколько предметов
data class Enchantment(
    val code: String,
    val name: String,
    val bonusDamage: Int,
    val bonusDefense: Double,
    val bonusPrice: Int,
) {

    companion object {
        val SHARP = Enchantment("sharp", "Острое", 3, 0.0, 20)
        val FIERY = Enchantment("fiery", "Огненное", 6, 0.0, 50)
        val STURDY = Enchantment("sturdy", "Прочное", 0, 0.05, 20)
        val WARDING = Enchantment("warding", "Охранное", 0, 0.10, 50)

        val FOR_WEAPONS = listOf(SHARP, FIERY)
        val FOR_ARMOR = listOf(STURDY, WARDING)
    }
}
```

В базовый класс предмета заложим и контракт `copy`:

```kotlin
package labs.lab3.item

abstract class Item(
    // код шаблона, из которого получен предмет
    val code: String,
    val name: String,
    private val basePrice: Int,
) {

    private val enchantmentList = mutableListOf<Enchantment>()

    // конструктор копирования, его вызывают наследники в методе copy()
    protected constructor(source: Item) : this(source.code, source.name, source.basePrice) {
        // зачарования переносим в собственный список копии, иначе копия и оригинал делили бы один и тот же список
        enchantmentList.addAll(source.enchantmentList)
    }

    val price: Int
        get() = basePrice + enchantmentList.sumOf { it.bonusPrice }

    // наружу список отдается только для чтения
    val enchantments: List<Enchantment>
        get() = enchantmentList.toList()

    // какие зачарования подходят этому предмету
    abstract val possibleEnchantments: List<Enchantment>

    // описание характеристик предмета, например "урон 12"
    abstract val description: String

    // каждый наследник по-своему реализует копирование
    abstract fun copy(): Item

    fun enchant(enchantment: Enchantment) {
        require(enchantment in possibleEnchantments) {
            "Зачарование '${enchantment.name}' не подходит для предмета '$name'"
        }

        enchantmentList.add(enchantment)
    }
}
```

Обратите внимание на конструктор копирования. Если бы копия получила тот же самый список, что и оригинал
(`enchantmentList = source.enchantmentList`), они ссылались бы на один и тот же объект (поверхностная копия), и
зачарование, наложенное на копию, появилось бы и на шаблоне в каталоге. Поэтому у копии свой список, в который мы
переносим зачарования оригинала. А вот сами зачарования копировать не нужно - они неизменяемые, и копия может ссылаться на те же объекты.

Супер, общий контракт предметов понятен, давайте создадим контракт для оружия:

```kotlin
package labs.lab3.item

import labs.lab3.combat.Hit

abstract class Weapon : Item {

    private val baseDamage: Int

    protected constructor(code: String, name: String, basePrice: Int, damage: Int) : super(code, name, basePrice) {
        baseDamage = damage
    }

    protected constructor(source: Weapon) : super(source) {
        baseDamage = source.baseDamage
    }

    // урон с учетом зачарований
    val damage: Int
        get() = baseDamage + enchantments.sumOf { it.bonusDamage }

    override val possibleEnchantments: List<Enchantment>
        get() = Enchantment.FOR_WEAPONS

    // как именно оружие наносит урон, решают наследники
    abstract fun rollDamage(): Hit

    abstract override fun copy(): Weapon
}
```

Отлично, теперь наше первое оружие, меч. Его урон всегда одинаковый:

```kotlin
package labs.lab3.item

import labs.lab3.combat.Hit

class Sword : Weapon {

    constructor(code: String, name: String, basePrice: Int, damage: Int) : super(code, name, basePrice, damage)

    private constructor(source: Sword) : super(source)

    override val description: String
        get() = "урон $damage"

    override fun rollDamage(): Hit = Hit(damage, false)

    override fun copy(): Sword = Sword(this)
}
```

Лук же с некоторым шансом наносит критический, двойной урон:

```kotlin
package labs.lab3.item

import labs.lab3.combat.Hit
import labs.lab3.core.Dice
import kotlin.math.roundToInt

class Bow : Weapon {

    private val criticalChance: Double

    constructor(code: String, name: String, basePrice: Int, damage: Int, criticalChance: Double) :
        super(code, name, basePrice, damage) {
        this.criticalChance = criticalChance
    }

    private constructor(source: Bow) : super(source) {
        criticalChance = source.criticalChance
    }

    override val description: String
        get() = "урон $damage, шанс крита ${(criticalChance * 100).roundToInt()}%"

    override fun rollDamage(): Hit {
        if (Dice.chance(criticalChance)) {
            return Hit(damage * CRITICAL_MULTIPLIER, true)
        }

        return Hit(damage, false)
    }

    override fun copy(): Bow = Bow(this)

    private companion object {
        const val CRITICAL_MULTIPLIER = 2
    }
}
```

Урон магического посоха каждый раз немного разный:

```kotlin
package labs.lab3.item

import labs.lab3.combat.Hit
import labs.lab3.core.Dice
import kotlin.math.roundToInt

// урон посоха непредсказуем: он может быть как больше, так и меньше базового
class Staff : Weapon {

    private val scatter: Double

    constructor(code: String, name: String, basePrice: Int, damage: Int, scatter: Double) :
        super(code, name, basePrice, damage) {
        this.scatter = scatter
    }

    private constructor(source: Staff) : super(source) {
        scatter = source.scatter
    }

    private val minDamage: Int
        get() = (damage * (1 - scatter)).roundToInt()

    private val maxDamage: Int
        get() = (damage * (1 + scatter)).roundToInt()

    override val description: String
        get() = "урон $minDamage-$maxDamage"

    override fun rollDamage(): Hit = Hit(Dice.roll(minDamage, maxDamage), false)

    override fun copy(): Staff = Staff(this)
}
```

Дальше давайте создадим класс для брони, он у нас будет общий для всех ее видов:

```kotlin
package labs.lab3.item

import kotlin.math.min
import kotlin.math.roundToInt

// броня поглощает долю входящего урона
class Armor : Item {

    private val baseDefense: Double

    constructor(code: String, name: String, basePrice: Int, defense: Double) : super(code, name, basePrice) {
        baseDefense = defense
    }

    private constructor(source: Armor) : super(source) {
        baseDefense = source.baseDefense
    }

    // доля поглощаемого урона с учетом зачарований
    val defense: Double
        get() = min(baseDefense + enchantments.sumOf { it.bonusDefense }, MAX_DEFENSE)

    override val possibleEnchantments: List<Enchantment>
        get() = Enchantment.FOR_ARMOR

    override val description: String
        get() = "защита ${(defense * 100).roundToInt()}%"

    override fun copy(): Armor = Armor(this)

    private companion object {
        // больше этой доли урона никакая броня не поглощает
        const val MAX_DEFENSE = 0.75
    }
}
```

Каждый конкретный предмет (меч, лук, посох, броня) реализует `copy()` через свой приватный конструктор копирования, который сначала вызывает
конструктор копирования родителя, а затем копирует собственные поля. 
Так каждый класс сам отвечает за то, чтобы скопировать все свои поля.

Теперь давайте напишем каталог: он хранит шаблоны всех предметов игры и выдает по ним конкретные экземпляры оружия и
брони:

```kotlin
package labs.lab3.item

import labs.lab3.core.UnknownCodeException

// каталог хранит шаблоны предметов, а наружу отдает только их копии
class ItemCatalog {

    // LinkedHashMap запоминает порядок добавления
    private val prototypes = LinkedHashMap<String, Item>()

    val codes: List<String>
        get() = prototypes.keys.toList()

    fun register(prototype: Item) {
        prototypes[prototype.code] = prototype
    }

    // каталог не знает, какой конкретно класс у шаблона - копию создает сам шаблон
    fun create(code: String): Item {
        val prototype = prototypes[code] ?: throw UnknownCodeException("Предмет", code)
        return prototype.copy()
    }

    // as? - безопасное приведение: вернет null, если предмет другого типа
    fun createWeapon(code: String): Weapon =
        create(code) as? Weapon ?: throw IllegalArgumentException("Предмет '$code' не является оружием")

    fun createArmor(code: String): Armor =
        create(code) as? Armor ?: throw IllegalArgumentException("Предмет '$code' не является броней")
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

```kotlin
package labs.lab3.item

// все предметы, которые есть в игре
object DefaultCatalog {

    // коды шаблонов: по ним предметы ищутся в каталоге
    // стартовое снаряжение
    const val TRAINING_SWORD = "training-sword"
    const val HUNTING_BOW = "hunting-bow"
    const val APPRENTICE_STAFF = "apprentice-staff"
    const val CHAINMAIL = "chainmail"
    const val LEATHER_ARMOR = "leather-armor"
    const val LINEN_ROBE = "linen-robe"

    // товары торговца
    const val IRON_SWORD = "iron-sword"
    const val ELVEN_BOW = "elven-bow"
    const val EMBER_STAFF = "ember-staff"
    const val PLATE_ARMOR = "plate-armor"

    // добыча
    const val GOBLIN_CLEAVER = "goblin-cleaver"
    const val BONE_BOW = "bone-bow"
    const val NECROMANCER_STAFF = "necromancer-staff"
    const val KNIGHT_BLADE = "knight-blade"
    const val CRYPT_MAIL = "crypt-mail"
    const val DAWN_BLADE = "dawn-blade"
    const val DRAGONSCALE_ARMOR = "dragonscale-armor"

    fun create(): ItemCatalog {
        val catalog = ItemCatalog()

        // стартовое снаряжение
        catalog.register(Sword(TRAINING_SWORD, "Тренировочный меч", 20, 12))
        catalog.register(Bow(HUNTING_BOW, "Охотничий лук", 20, 10, 0.3))
        catalog.register(Staff(APPRENTICE_STAFF, "Посох ученика", 20, 14, 0.3))
        catalog.register(Armor(CHAINMAIL, "Кольчуга", 40, 0.3))
        catalog.register(Armor(LEATHER_ARMOR, "Кожаная броня", 30, 0.2))
        catalog.register(Armor(LINEN_ROBE, "Льняная роба", 15, 0.1))

        // товары торговца
        catalog.register(Sword(IRON_SWORD, "Железный меч", 80, 17))
        catalog.register(Bow(ELVEN_BOW, "Эльфийский лук", 90, 14, 0.4))
        catalog.register(Staff(EMBER_STAFF, "Тлеющий посох", 90, 20, 0.3))
        catalog.register(Armor(PLATE_ARMOR, "Латы", 120, 0.4))

        // добыча
        catalog.register(Sword(GOBLIN_CLEAVER, "Гоблинский тесак", 40, 15))
        catalog.register(Bow(BONE_BOW, "Костяной лук", 140, 17, 0.4))
        catalog.register(Staff(NECROMANCER_STAFF, "Посох некроманта", 150, 25, 0.35))
        catalog.register(Sword(KNIGHT_BLADE, "Клинок рыцаря", 150, 23))
        catalog.register(Armor(CRYPT_MAIL, "Кольчуга стража склепа", 160, 0.45))
        catalog.register(Sword(DAWN_BLADE, "Клинок рассвета", 500, 35))
        catalog.register(Armor(DRAGONSCALE_ARMOR, "Драконья чешуя", 500, 0.55))

        return catalog
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

```kotlin
package labs.lab3.kit

import labs.lab3.item.Armor
import labs.lab3.item.Weapon

// Абстрактная фабрика: стартовый набор снаряжения
interface StarterKitFactory {

    // возвращает некое оружие
    fun createWeapon(): Weapon

    // возвращает некую броню
    fun createArmor(): Armor
}
```

И для каждого из классов героя мы создадим свою реализацию этого набора: снаряжение каждого класса - это и есть
семейство объектов.

В наборе для воина будут лежать меч и кольчуга:

```kotlin
package labs.lab3.kit

import labs.lab3.item.Armor
import labs.lab3.item.DefaultCatalog
import labs.lab3.item.ItemCatalog
import labs.lab3.item.Weapon

class WarriorKitFactory(private val catalog: ItemCatalog) : StarterKitFactory {

    override fun createWeapon(): Weapon = catalog.createWeapon(DefaultCatalog.TRAINING_SWORD)

    override fun createArmor(): Armor = catalog.createArmor(DefaultCatalog.CHAINMAIL)
}
```

В наборе для следопыта - лук и кожаная броня:

```kotlin
package labs.lab3.kit

import labs.lab3.item.Armor
import labs.lab3.item.DefaultCatalog
import labs.lab3.item.ItemCatalog
import labs.lab3.item.Weapon

class RangerKitFactory(private val catalog: ItemCatalog) : StarterKitFactory {

    override fun createWeapon(): Weapon = catalog.createWeapon(DefaultCatalog.HUNTING_BOW)

    override fun createArmor(): Armor = catalog.createArmor(DefaultCatalog.LEATHER_ARMOR)
}
```

Ну а для мага - посох и роба:

```kotlin
package labs.lab3.kit

import labs.lab3.item.Armor
import labs.lab3.item.DefaultCatalog
import labs.lab3.item.ItemCatalog
import labs.lab3.item.Weapon

class MageKitFactory(private val catalog: ItemCatalog) : StarterKitFactory {

    override fun createWeapon(): Weapon = catalog.createWeapon(DefaultCatalog.APPRENTICE_STAFF)

    override fun createArmor(): Armor = catalog.createArmor(DefaultCatalog.LINEN_ROBE)
}
```

Классы героя опишем через `enum`, и каждый класс будет сам знать свою фабрику стартового снаряжения:

```kotlin
package labs.lab3.hero

import labs.lab3.core.UnknownCodeException
import labs.lab3.item.ItemCatalog
import labs.lab3.kit.MageKitFactory
import labs.lab3.kit.RangerKitFactory
import labs.lab3.kit.StarterKitFactory
import labs.lab3.kit.WarriorKitFactory

enum class HeroClass(
    val code: String,
    val displayName: String,
    val baseHealth: Int,
    // функция, которая создает фабрику стартового снаряжения
    private val starterKit: (ItemCatalog) -> StarterKitFactory,
) {
    // ::WarriorKitFactory - ссылка на конструктор фабрики набора воина
    WARRIOR("warrior", "Воин", 120, ::WarriorKitFactory),
    RANGER("ranger", "Следопыт", 100, ::RangerKitFactory),
    MAGE("mage", "Маг", 85, ::MageKitFactory);

    // каждый класс сам знает свою фабрику стартового снаряжения
    fun getStarterKit(catalog: ItemCatalog): StarterKitFactory = starterKit(catalog)

    companion object {
        fun byCode(code: String): HeroClass =
            entries.firstOrNull { it.code == code } ?: throw UnknownCodeException("Класс героя", code)
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

```kotlin
package labs.lab3.hero

import labs.lab3.combat.Combatant
import labs.lab3.combat.Hit
import labs.lab3.item.Armor
import labs.lab3.item.Item
import labs.lab3.item.Weapon
import labs.lab3.kit.StarterKitFactory
import kotlin.math.max
import kotlin.math.roundToInt

// конструктор приватный и принимает в себя билдера, из которого и берет значения полей
class Hero private constructor(builder: Builder) : Combatant {

    override val name: String = builder.name!!
    val heroClass: HeroClass = builder.heroClass!!

    // private set - читать свойство можно снаружи, а менять - только самому герою
    var level: Int = builder.level
        private set
    var experience: Int = 0
        private set
    var gold: Int = builder.gold
        private set
    var weapon: Weapon = builder.weapon!!
        private set
    var armor: Armor = builder.armor!!
        private set
    override var health: Int = 0
        private set

    private val inventoryList = mutableListOf<Item>()

    init {
        health = maxHealth
    }

    // внутренний класс билдер - чтобы удобно и безопасно создавать героев
    class Builder {

        internal var name: String? = null
        internal var heroClass: HeroClass? = null
        internal var weapon: Weapon? = null
        internal var armor: Armor? = null
        // необязательные поля сразу получают значения по умолчанию
        internal var level: Int = 1
        internal var gold: Int = STARTING_GOLD

        // apply выполняет блок и возвращает сам объект, поэтому вызовы можно выстраивать цепочкой
        fun setName(name: String) = apply { this.name = name }

        fun setHeroClass(heroClass: HeroClass) = apply { this.heroClass = heroClass }

        fun setWeapon(weapon: Weapon) = apply { this.weapon = weapon }

        fun setArmor(armor: Armor) = apply { this.armor = armor }

        // берет оружие и броню из стартового набора
        fun setStarterKit(starterKit: StarterKitFactory) = apply {
            weapon = starterKit.createWeapon()
            armor = starterKit.createArmor()
        }

        fun setLevel(level: Int) = apply { this.level = level }

        fun setGold(gold: Int) = apply { this.gold = gold }

        // перед созданием героя проверяем, что все обязательные поля заполнены корректно;
        // require бросает IllegalArgumentException с указанным сообщением, если условие не выполнено
        fun build(): Hero {
            val name = name
            require(!name.isNullOrBlank()) { "У героя должно быть имя" }
            require(name.length <= MAX_NAME_LENGTH) { "Имя героя не может быть длиннее $MAX_NAME_LENGTH символов" }
            require(heroClass != null) { "Не выбран класс героя" }
            require(weapon != null && armor != null) { "Герою нужны оружие и броня" }
            require(level >= 1) { "Уровень героя не может быть меньше 1" }
            require(gold >= 0) { "Золота не может быть меньше нуля" }

            return Hero(this)
        }
    }

    override val code: String
        get() = heroClass.code

    override val maxHealth: Int
        get() = heroClass.baseHealth + (level - 1) * HEALTH_PER_LEVEL

    override val isAlive: Boolean
        get() = health > 0

    val experienceToNextLevel: Int
        get() = level * EXPERIENCE_PER_LEVEL

    val inventory: List<Item>
        get() = inventoryList.toList()

    override fun rollDamage(): Hit {
        val hit = weapon.rollDamage()
        return hit.copy(damage = hit.damage + (level - 1) * DAMAGE_PER_LEVEL)
    }

    override fun receiveHit(damage: Int): Int {
        val reducedDamage = (damage * (1 - armor.defense)).roundToInt()
        // хотя бы единицу урона броня пропускает всегда
        val dealt = max(1, reducedDamage)
        health = max(0, health - dealt)
        return dealt
    }

    // поведение героя между экспедициями

    // перед каждой экспедицией герой успевает отдохнуть
    fun restoreHealth() {
        health = maxHealth
    }

    // начисляет опыт, возвращает количество полученных уровней
    fun gainExperience(amount: Int): Int {
        experience += amount
        var levelsGained = 0
        while (experience >= experienceToNextLevel) {
            experience -= experienceToNextLevel
            level++
            levelsGained++
        }

        return levelsGained
    }

    fun addGold(amount: Int) {
        gold += amount
    }

    fun spendGold(amount: Int) {
        // check бросает IllegalStateException, если условие не выполнено
        check(amount <= gold) { "Не хватает золота: нужно $amount, есть $gold" }
        gold -= amount
    }

    fun addItem(item: Item) {
        inventoryList.add(item)
    }

    fun removeItem(item: Item) {
        require(inventoryList.remove(item)) { "В инвентаре нет предмета ${item.name}" }
    }

    // экипирует предмет из инвентаря, снятый предмет возвращается в инвентарь
    fun equip(item: Item) {
        when (item) {
            is Weapon -> {
                removeItem(item)
                inventoryList.add(weapon)
                weapon = item
            }
            is Armor -> {
                removeItem(item)
                inventoryList.add(armor)
                armor = item
            }
            else -> throw IllegalArgumentException("Предмет нельзя экипировать: ${item.name}")
        }
    }

    private companion object {
        const val MAX_NAME_LENGTH = 20
        const val STARTING_GOLD = 50
        const val HEALTH_PER_LEVEL = 15
        const val DAMAGE_PER_LEVEL = 3
        const val EXPERIENCE_PER_LEVEL = 100
    }
}
```

В Kotlin строитель нужен реже, чем в Java: у конструкторов есть именованные параметры и значения по умолчанию
(`Hero(name = "Артур", level = 3)`), так что телескопических конструкторов здесь не бывает. У нас строитель оправдан
тем, что снаряжение берется из фабрики отдельным шагом. Для проверок удобны функции `require` и
`check`: если условие не выполнено, они бросают исключение с указанным сообщением.

Что в реализации строителя важно:

1. Конструктор героя приватный и принимает только строителя, поэтому создать героя в обход проверок нельзя.
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

```kotlin
package labs.lab3.enemy

import labs.lab3.combat.Combatant

interface Enemy : Combatant {

    val experienceReward: Int

    fun rollGoldReward(): Int
}
```

Общая логика врагов вынесена в абстрактный класс:

```kotlin
package labs.lab3.enemy

import labs.lab3.combat.Hit
import labs.lab3.core.Dice
import kotlin.math.max

// общая логика для врагов: здоровье, урон в диапазоне, награда
abstract class AbstractEnemy(
    override val code: String,
    override val name: String,
    override val maxHealth: Int,
    private val minDamage: Int,
    private val maxDamage: Int,
    override val experienceReward: Int,
    private val goldReward: Int,
) : Enemy {

    // final - наследники не смогут переопределить свойство, а значит, менять здоровье может только этот класс
    final override var health: Int = maxHealth
        private set

    override val isAlive: Boolean
        get() = health > 0

    // методы с override сами по себе открыты, поэтому наследники смогут переопределить и этот метод
    override fun rollDamage(): Hit = Hit(Dice.roll(minDamage, maxDamage), false)

    override fun receiveHit(damage: Int): Int {
        val dealt = max(1, reduceDamage(damage))
        health = max(0, health - dealt)
        return dealt
    }

    override fun rollGoldReward(): Int = Dice.roll(goldReward / 2, goldReward)

    // наследники могут уменьшать входящий урон
    protected open fun reduceDamage(damage: Int): Int = damage
}
```

В Kotlin классы и методы по умолчанию закрыты для наследования. Абстрактные классы и методы открыты сами, обычный
метод, который наследникам разрешено переопределять, помечают словом `open` (как `reduceDamage`), а методы с `override`
остаются открытыми, пока их не пометят `final`.

Определим злого гоблина, самого слабого врага, живущего в лесу:

```kotlin
package labs.lab3.enemy

// гоблин - самый простой враг, без особенностей
class Goblin : AbstractEnemy("goblin", "Гоблин", 50, 6, 14, 40, 30)
```

Теперь скелет, он уже чуть страшнее гоблина:

```kotlin
package labs.lab3.enemy

import kotlin.math.roundToInt

// скелет: кости плохо поддаются ударам, поэтому часть урона проходит мимо
class Skeleton : AbstractEnemy("skeleton", "Скелет", 110, 10, 24, 90, 70) {

    override fun reduceDamage(damage: Int): Int = (damage * (1 - RESISTANCE)).roundToInt()

    private companion object {
        const val RESISTANCE = 0.3
    }
}
```

И ужасный дракон: он наносит страшный урон огнем, а крепкая чешуя гасит часть урона:

```kotlin
package labs.lab3.enemy

import labs.lab3.combat.Hit
import labs.lab3.core.Dice
import kotlin.math.roundToInt

// дракон защищен чешуей и время от времени дышит огнем
class Dragon : AbstractEnemy("dragon", "Дракон", 260, 16, 22, 500, 400) {

    override fun rollDamage(): Hit {
        val bite = super.rollDamage()
        if (Dice.chance(FIRE_BREATH_CHANCE)) {
            return Hit(bite.damage * FIRE_BREATH_MULTIPLIER, true)
        }

        return bite
    }

    override fun reduceDamage(damage: Int): Int = (damage * (1 - RESISTANCE)).roundToInt()

    private companion object {
        const val RESISTANCE = 0.25
        const val FIRE_BREATH_CHANCE = 0.25
        const val FIRE_BREATH_MULTIPLIER = 2
    }
}
```

По итогам экспедиции герой получает награды: опыт, золото, добычу и новые уровни. Запишем их так:

```kotlin
package labs.lab3.world

import labs.lab3.item.Item

// итоги экспедиции для героя;
// gold отрицательный, если герой проиграл и потерял золото, loot равен null, если добычи нет
data class Rewards(val experience: Int, val gold: Int, val loot: Item?, val levelsGained: Int)
```

Ну а общий итог экспедиции фиксируем вот так:

```kotlin
package labs.lab3.world

import labs.lab3.combat.BattleReport

// отчет об экспедиции: с каким сидом, куда, как прошел бой и что получил герой
data class ExpeditionReport(val seed: Int, val locationCode: String, val battle: BattleReport, val rewards: Rewards)
```

Дальше давайте определим абстрактную локацию и общую логику экспедиции. Конкретных врагов здесь не создаем - это работа
подклассов:

```kotlin
package labs.lab3.world

import labs.lab3.combat.Battle
import labs.lab3.combat.Outcome
import labs.lab3.core.Dice
import labs.lab3.enemy.Enemy
import labs.lab3.hero.Hero
import labs.lab3.item.Item
import labs.lab3.item.ItemCatalog

abstract class Location(
    val code: String,
    val displayName: String,
    val recommendedLevel: Int,
    private val catalog: ItemCatalog,
) {

    // коды предметов из каталога, которые можно найти в локации
    protected abstract val lootTable: List<String>

    // создание врага делегируется подклассам для повышения гибкости
    protected abstract fun createEnemy(): Enemy

    // бизнес-логика, общая для всех локаций:
    // герой встречает врага, сражается с ним и получает награду, если победил;
    // если сид не передан, берется случайный
    fun explore(hero: Hero, seed: Int = Dice.nextSeed()): ExpeditionReport {
        Dice.reseed(seed)

        hero.restoreHealth()
        val enemy = createEnemy()
        val battle = Battle(hero, enemy).run()

        val rewards = if (battle.outcome == Outcome.VICTORY) reward(hero, enemy) else punish(hero)

        return ExpeditionReport(seed, code, battle, rewards)
    }

    private fun reward(hero: Hero, enemy: Enemy): Rewards {
        val experience = enemy.experienceReward
        val gold = enemy.rollGoldReward()
        val loot = rollLoot()

        val levelsGained = hero.gainExperience(experience)
        hero.addGold(gold)
        if (loot != null) {
            hero.addItem(loot)
        }

        return Rewards(experience, gold, loot, levelsGained)
    }

    private fun rollLoot(): Item? {
        if (!Dice.chance(LOOT_CHANCE)) {
            return null
        }

        // добыча - это копия шаблона из каталога, ее можно менять, не боясь испортить шаблон
        val loot = catalog.create(Dice.pick(lootTable))
        if (Dice.chance(ENCHANT_CHANCE)) {
            loot.enchant(Dice.pick(loot.possibleEnchantments))
        }

        return loot
    }

    // проигравший герой теряет половину золота
    private fun punish(hero: Hero): Rewards {
        val lostGold = hero.gold / 2
        hero.spendGold(lostGold)
        return Rewards(0, -lostGold, null, 0)
    }

    private companion object {
        const val LOOT_CHANCE = 0.6
        const val ENCHANT_CHANCE = 0.3
    }
}
```

Также обратите внимание, что добыча здесь - копия шаблона из каталога, и зачарование накладывается именно на копию.

Отлично, теперь создаем конкретные локации:

Мистический лес, кишащий злыми, но слабыми гоблинами:

```kotlin
package labs.lab3.world

import labs.lab3.enemy.Enemy
import labs.lab3.enemy.Goblin
import labs.lab3.item.DefaultCatalog
import labs.lab3.item.ItemCatalog

class Forest(catalog: ItemCatalog) : Location(CODE, "Мистический лес", 1, catalog) {

    override val lootTable = listOf(
        DefaultCatalog.GOBLIN_CLEAVER, DefaultCatalog.HUNTING_BOW, DefaultCatalog.LEATHER_ARMOR,
    )

    override fun createEnemy(): Enemy = Goblin()

    companion object {
        const val CODE = "forest"
    }
}
```

Заброшенный склеп, в котором бродят ожившие скелеты:

```kotlin
package labs.lab3.world

import labs.lab3.enemy.Enemy
import labs.lab3.enemy.Skeleton
import labs.lab3.item.DefaultCatalog
import labs.lab3.item.ItemCatalog

class Crypt(catalog: ItemCatalog) : Location(CODE, "Заброшенный склеп", 3, catalog) {

    override val lootTable = listOf(
        DefaultCatalog.KNIGHT_BLADE, DefaultCatalog.BONE_BOW, DefaultCatalog.NECROMANCER_STAFF, DefaultCatalog.CRYPT_MAIL,
    )

    override fun createEnemy(): Enemy = Skeleton()

    companion object {
        const val CODE = "crypt"
    }
}
```

И логово дракона, прибежище жуткого чешуйчатого чудища:

```kotlin
package labs.lab3.world

import labs.lab3.enemy.Dragon
import labs.lab3.enemy.Enemy
import labs.lab3.item.DefaultCatalog
import labs.lab3.item.ItemCatalog

class DragonLair(catalog: ItemCatalog) : Location(CODE, "Логово дракона", 5, catalog) {

    override val lootTable = listOf(DefaultCatalog.DAWN_BLADE, DefaultCatalog.DRAGONSCALE_ARMOR)

    override fun createEnemy(): Enemy = Dragon()

    companion object {
        const val CODE = "dragon-lair"
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

```kotlin
package labs.lab3.world

import labs.lab3.core.UnknownCodeException

// все локации игры
class LocationRegistry(locations: List<Location>) {

    // associateBy строит словарь "код -> локация" и сохраняет порядок
    private val locations = locations.associateBy { it.code }

    val all: List<Location>
        get() = locations.values.toList()

    fun get(code: String): Location = locations[code] ?: throw UnknownCodeException("Локация", code)
}
```

## Лагерь

Между экспедициями герой отдыхает в лагере, где есть торговец (а куда же без него). 
Торговец продает предметы из каталога и скупает ненужное за половину цены:

```kotlin
package labs.lab3.camp

import labs.lab3.core.UnknownCodeException
import labs.lab3.hero.Hero
import labs.lab3.item.Item
import labs.lab3.item.ItemCatalog

class Merchant(private val catalog: ItemCatalog, private val assortment: List<String>) {

    // образцы товаров, чтобы показать их покупателю
    val showcase: List<Item>
        get() = assortment.map { catalog.create(it) }

    fun buy(hero: Hero, code: String): Item {
        if (code !in assortment) {
            throw UnknownCodeException("Товар", code)
        }

        // покупатель получает собственную копию предмета из каталога
        val item = catalog.create(code)
        hero.spendGold(item.price)
        hero.addItem(item)
        return item
    }

    fun sell(hero: Hero, item: Item): Int {
        val price = getSellPrice(item)
        hero.removeItem(item)
        hero.addGold(price)
        return price
    }

    fun getSellPrice(item: Item): Int = item.price * SELL_PRICE_PERCENT / 100

    private companion object {
        const val SELL_PRICE_PERCENT = 50
    }
}
```

## Консоль

Вся игровая логика наконец-то готова, осталось научить программу общаться с игроком. Весь вывод соберем в одном классе - он
превращает героя, предметы и отчеты об экспедиции в текст для консоли. 

Отдельный класс нужен потому, что дальше с игрой можно будет работать не только через консоль (спойлер-спойлер), и
привязывать к ней игровую логику не хочется.

Вот он:

```kotlin
package labs.lab3

import labs.lab3.combat.BattleEvent
import labs.lab3.combat.BattleReport
import labs.lab3.combat.EventType
import labs.lab3.combat.Outcome
import labs.lab3.combat.Participant
import labs.lab3.combat.Side
import labs.lab3.hero.Hero
import labs.lab3.item.Item
import labs.lab3.world.ExpeditionReport
import labs.lab3.world.Rewards

// весь вывод в консоль собран здесь
class ConsoleView {

    fun describe(item: Item): String {
        var description = item.name

        val enchantments = item.enchantments
        if (enchantments.isNotEmpty()) {
            val names = mutableListOf<String>()
            for (enchantment in enchantments) {
                names.add(enchantment.name)
            }
            description += " [" + names.joinToString(", ") + "]"
        }

        return "$description (${item.description})"
    }

    fun printHero(hero: Hero) {
        println(
            "${hero.name}, ${hero.heroClass.displayName} ${hero.level} ур. | здоровье ${hero.maxHealth} | " +
                "опыт ${hero.experience}/${hero.experienceToNextLevel} | золото ${hero.gold}"
        )
        println("  Оружие: " + describe(hero.weapon))
        println("  Броня:  " + describe(hero.armor))
    }

    fun printReport(report: ExpeditionReport, locationName: String) {
        val battle = report.battle
        println("=== $locationName (сид ${report.seed}) ===")
        println("${battle.hero.name} встречает врага: ${battle.enemy.name} (${battle.enemy.maxHealth} здоровья)")

        for (event in battle.events) {
            printEvent(event, battle)
        }

        printRewards(battle, report.rewards)
    }

    private fun printEvent(event: BattleEvent, battle: BattleReport) {
        val source = participant(event.source, battle)
        val target = participant(event.target, battle)

        when (event.type) {
            EventType.HIT -> println(
                "[раунд ${event.round}] ${source.name} -> ${target.name}: ${event.amount} урона, " +
                    "осталось ${event.targetHealthAfter}/${target.maxHealth}"
            )
            EventType.CRITICAL_HIT -> println(
                "[раунд ${event.round}] ${source.name} -> ${target.name}: КРИТ! ${event.amount} урона, " +
                    "осталось ${event.targetHealthAfter}/${target.maxHealth}"
            )
            EventType.DEATH -> println("${target.name} повержен!")
        }
    }

    private fun printRewards(battle: BattleReport, rewards: Rewards) {
        if (battle.outcome == Outcome.DEFEAT) {
            println("Поражение... ${battle.hero.name} едва уносит ноги и теряет ${-rewards.gold} золота")
            return
        }

        println("Победа! +${rewards.experience} опыта, +${rewards.gold} золота")
        rewards.loot?.let { println("Добыча: " + describe(it)) }
        if (rewards.levelsGained > 0) {
            println("Новый уровень!")
        }
    }

    private fun participant(side: Side, battle: BattleReport): Participant =
        if (side == Side.HERO) battle.hero else battle.enemy
}
```

Осталось написать игровой цикл - он будет в функции `main` (файл `Main.kt`):

```kotlin
package labs.lab3

import labs.lab3.camp.Merchant
import labs.lab3.combat.Outcome
import labs.lab3.hero.Hero
import labs.lab3.hero.HeroClass
import labs.lab3.item.DefaultCatalog
import labs.lab3.item.Item
import labs.lab3.item.ItemCatalog
import labs.lab3.world.Crypt
import labs.lab3.world.DragonLair
import labs.lab3.world.Forest
import labs.lab3.world.LocationRegistry
import java.nio.charset.Charset
import kotlin.system.exitProcess

private val view = ConsoleView()

private val input = System.`in`.bufferedReader(System.console()?.charset() ?: Charset.defaultCharset())

fun main() {
    val catalog = DefaultCatalog.create()
    val locations = LocationRegistry(
        listOf(
            Forest(catalog),
            Crypt(catalog),
            DragonLair(catalog),
        )
    )
    val merchant = Merchant(
        catalog,
        listOf(
            DefaultCatalog.IRON_SWORD, DefaultCatalog.ELVEN_BOW, DefaultCatalog.EMBER_STAFF,
            DefaultCatalog.CHAINMAIL, DefaultCatalog.PLATE_ARMOR,
        ),
    )

    val hero = createHero(catalog)
    println("${hero.name} разбивает лагерь на опушке мистического леса.")

    while (true) {
        println()
        view.printHero(hero)
        println("1 - отправиться в экспедицию, 2 - снаряжение, 3 - торговец, 0 - выйти из игры")
        when (readInput()) {
            "1" -> goOnExpedition(hero, locations)
            "2" -> manageEquipment(hero)
            "3" -> visitMerchant(hero, merchant)
            "0" -> return
            else -> println("Такого варианта нет")
        }
    }
}

private fun createHero(catalog: ItemCatalog): Hero {
    println("Создайте своего героя!")
    while (true) {
        println("Выберите класс:")
        val heroClasses = HeroClass.entries
        heroClasses.forEachIndexed { i, heroClass ->
            val kit = heroClass.getStarterKit(catalog)
            println(
                "${i + 1} - ${heroClass.displayName} (здоровье ${heroClass.baseHealth}): " +
                    "${view.describe(kit.createWeapon())}, ${view.describe(kit.createArmor())}"
            )
        }
        val classIndex = readChoice(heroClasses.size)
        if (classIndex < 0) {
            continue
        }
        val heroClass = heroClasses[classIndex]

        println("Введите имя:")
        val name = readInput()

        try {
            return Hero.Builder()
                .setName(name)
                .setHeroClass(heroClass)
                .setStarterKit(heroClass.getStarterKit(catalog))
                .build()
        } catch (e: IllegalArgumentException) {
            println("Не получилось создать героя: " + e.message)
        }
    }
}

private fun goOnExpedition(hero: Hero, locations: LocationRegistry) {
    val all = locations.all
    println("Куда отправимся?")
    all.forEachIndexed { i, location ->
        println("${i + 1} - ${location.displayName} (рекомендуемый уровень ${location.recommendedLevel})")
    }
    val index = readChoice(all.size)
    if (index < 0) {
        return
    }

    val location = all[index]
    val report = location.explore(hero)
    println()
    view.printReport(report, location.displayName)

    if (report.battle.outcome == Outcome.VICTORY && location.code == DragonLair.CODE) {
        println("\nДракон повержен! Барды еще долго будут петь о подвиге героя по имени ${hero.name}.")
    }
}

private fun manageEquipment(hero: Hero) {
    val inventory = hero.inventory
    if (inventory.isEmpty()) {
        println("Инвентарь пуст")
        return
    }

    println("Что экипировать? (0 - назад)")
    printItems(inventory)
    val index = readChoice(inventory.size)
    if (index < 0) {
        return
    }

    val item = inventory[index]
    hero.equip(item)
    println("Экипировано: " + view.describe(item))
}

private fun visitMerchant(hero: Hero, merchant: Merchant) {
    println("Торговец: \"Чего желаешь?\" (золото: ${hero.gold})")
    println("1 - купить, 2 - продать, 0 - назад")
    when (readInput()) {
        "1" -> buy(hero, merchant)
        "2" -> sell(hero, merchant)
    }
}

private fun buy(hero: Hero, merchant: Merchant) {
    val showcase = merchant.showcase
    showcase.forEachIndexed { i, item ->
        println("${i + 1} - ${view.describe(item)}: ${item.price} золота")
    }
    val index = readChoice(showcase.size)
    if (index < 0) {
        return
    }

    try {
        val item = merchant.buy(hero, showcase[index].code)
        println("Куплено: " + view.describe(item))
    } catch (e: IllegalStateException) {
        println(e.message)
    }
}

private fun sell(hero: Hero, merchant: Merchant) {
    val inventory = hero.inventory
    if (inventory.isEmpty()) {
        println("Продавать нечего (экипированные предметы продать нельзя)")
        return
    }

    inventory.forEachIndexed { i, item ->
        println("${i + 1} - ${view.describe(item)}: ${merchant.getSellPrice(item)} золота")
    }
    val index = readChoice(inventory.size)
    if (index < 0) {
        return
    }

    val item = inventory[index]
    val price = merchant.sell(hero, item)
    println("Продано: ${item.name} за $price золота")
}

private fun printItems(items: List<Item>) {
    items.forEachIndexed { i, item -> println("${i + 1} - ${view.describe(item)}") }
}

// читает строку без пробелов по краям; если ввод закончился, завершает программу
private fun readInput(): String = input.readLine()?.trim() ?: exitProcess(0)

// читает номер варианта от 1 до count, возвращает индекс или -1, если ввели что-то другое
private fun readChoice(count: Int): Int {
    val choice = readInput().toIntOrNull() ?: return -1
    return if (choice in 1..count) choice - 1 else -1
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

Чтобы показать все паттерны одним запуском, напишем объект `Demo` со своей функцией `main`. Вторую функцию `main` верхнего уровня в том же пакете объявить нельзя, поэтому она лежит внутри
`object`:

```kotlin
package labs.lab3

import labs.lab3.core.Dice
import labs.lab3.hero.Hero
import labs.lab3.hero.HeroClass
import labs.lab3.item.DefaultCatalog
import labs.lab3.item.Enchantment
import labs.lab3.world.Crypt
import labs.lab3.world.Forest

object Demo {

    @JvmStatic
    fun main(args: Array<String>) {
        val catalog = DefaultCatalog.create()
        val view = ConsoleView()

        println("===== Одиночка =====")
        val first = Dice
        val second = Dice
        println("Dice - один и тот же объект: ${first === second}")

        println()
        println("===== Прототип =====")
        val plainSword = catalog.create(DefaultCatalog.IRON_SWORD)
        val fierySword = catalog.create(DefaultCatalog.IRON_SWORD)
        fierySword.enchant(Enchantment.FIERY)
        println("Первая копия:        " + view.describe(plainSword))
        println("Вторая копия:        " + view.describe(fierySword))
        println("Новая копия шаблона: " + view.describe(catalog.create(DefaultCatalog.IRON_SWORD)))
        println("Копии - разные объекты: ${plainSword !== fierySword}")

        println()
        println("===== Абстрактная фабрика =====")
        for (heroClass in HeroClass.entries) {
            val kit = heroClass.getStarterKit(catalog)
            println("${heroClass.displayName}: ${view.describe(kit.createWeapon())}, ${view.describe(kit.createArmor())}")
        }

        println()
        println("===== Строитель =====")
        val hero = Hero.Builder()
            .setName("Артур")
            .setHeroClass(HeroClass.WARRIOR)
            .setStarterKit(HeroClass.WARRIOR.getStarterKit(catalog))
            .setLevel(3)
            .build()
        view.printHero(hero)
        try {
            Hero.Builder()
                .setHeroClass(HeroClass.MAGE)
                .setStarterKit(HeroClass.MAGE.getStarterKit(catalog))
                .build()
        } catch (e: IllegalArgumentException) {
            println("Герой без имени не создается: " + e.message)
        }
        try {
            Hero.Builder()
                .setName("Мерлин")
                .build()
        } catch (e: IllegalArgumentException) {
            println("Герой без класса не создается: " + e.message)
        }

        println()
        println("===== Фабричный метод =====")
        val forest = Forest(catalog)
        val crypt = Crypt(catalog)
        view.printReport(forest.explore(hero, 7), forest.displayName)
        println()
        view.printReport(crypt.explore(hero, 7), crypt.displayName)

    }
}
```

Вывод программы:

```
===== Одиночка =====
Dice - один и тот же объект: true

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
[раунд 1] Гоблин -> Артур: 10 урона, осталось 140/150
[раунд 2] Артур -> Гоблин: 18 урона, осталось 14/50
[раунд 2] Гоблин -> Артур: 8 урона, осталось 132/150
[раунд 3] Артур -> Гоблин: 18 урона, осталось 0/50
Гоблин повержен!
Победа! +40 опыта, +23 золота
Добыча: Охотничий лук (урон 10, шанс крита 30%)

=== Заброшенный склеп (сид 7) ===
Артур встречает врага: Скелет (110 здоровья)
[раунд 1] Артур -> Скелет: 13 урона, осталось 97/110
[раунд 1] Скелет -> Артур: 15 урона, осталось 135/150
[раунд 2] Артур -> Скелет: 13 урона, осталось 84/110
[раунд 2] Скелет -> Артур: 17 урона, осталось 118/150
[раунд 3] Артур -> Скелет: 13 урона, осталось 71/110
[раунд 3] Скелет -> Артур: 9 урона, осталось 109/150
[раунд 4] Артур -> Скелет: 13 урона, осталось 58/110
[раунд 4] Скелет -> Артур: 15 урона, осталось 94/150
[раунд 5] Артур -> Скелет: 13 урона, осталось 45/110
[раунд 5] Скелет -> Артур: 8 урона, осталось 86/150
[раунд 6] Артур -> Скелет: 13 урона, осталось 32/110
[раунд 6] Скелет -> Артур: 14 урона, осталось 72/150
[раунд 7] Артур -> Скелет: 13 урона, осталось 19/110
[раунд 7] Скелет -> Артур: 15 урона, осталось 57/150
[раунд 8] Артур -> Скелет: 13 урона, осталось 6/110
[раунд 8] Скелет -> Артур: 17 урона, осталось 40/150
[раунд 9] Артур -> Скелет: 13 урона, осталось 0/110
Скелет повержен!
Победа! +90 опыта, +45 золота
Добыча: Костяной лук (урон 17, шанс крита 40%)
```

Не забудьте закоммитить и запушить изменения!

# Лаба 4 — практика на Kotlin

Это практическая часть [лабораторной работы 4](../laba4.md): требования к сдаче и вопросы к защите находятся в основном тексте лабы.

Понадобится JDK 17 или новее и IntelliJ IDEA, проект собирается через Maven. Шаблон репозитория: [tech-prog-labs-2026-kotlin](https://github.com/tidbid-kt/tech-prog-labs-2026-kotlin). Код лабы пишите в пакете `labs.lab3`. Запускать программы проще всего зеленой кнопкой рядом с `main` в IDEA, а из терминала - так: игра - `mvn -q compile exec:java -Dexec.mainClass=labs.lab3.MainKt`, демонстрация - `mvn -q compile exec:java -Dexec.mainClass=labs.lab3.Demo`.

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
Базовый пакет `labs.lab3` и расположение исходников сохраняются; создавать пакет `lab4` и копировать
классы не нужно. Дальше мы разберем только новые и измененные классы, остальные остаются как в прошлой лабе.

Новые и измененные классы приведены целиком. Ключевые изменения в коде отмечены короткими комментариями.

## Подготовка: данные и контракты

Наши паттерны потребуют немного подготовки перед введением их в действие! Давайте же эту подготовку приведем.

Старого механизма обработки урона нам теперь не хватает. Раньше мы работали просто с числами, теперь давайте масштабироваться.
Заведем отдельный класс `Damage`. Он будет инкапсулировать в себе результат получения урона
(сколько получили, удалось ли уклонится, сколько было поглощено), эти данные понадобятся нам дальше.

**`src/main/kotlin/labs/lab3/defense/Damage.kt`**

```kotlin
package labs.lab3.defense

class Damage(var amount: Int) {
    var evaded = false
        private set
    var absorbed = 0
        private set
    fun reduceTo(amount: Int) { this.amount = amount }
    fun evade() { amount = 0; evaded = true }
    fun absorb(value: Int) { amount -= value; absorbed += value }
}
```

И теперь `Combatant.receiveHit` будет возвращать `Damage` вместо `Int`. Так мы сможем реализовать отдельные механики защиты от урона.
Получится что-то такое:

**`src/main/kotlin/labs/lab3/combat/Combatant.kt`**

```kotlin
package labs.lab3.combat

import labs.lab3.defense.Damage

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
    fun receiveHit(damage: Int): Damage
}
```

Для врагов давайте обработку урона в `receiveHit` также сделаем через `Damage`, не зря же мы этот класс добавляли:

**`src/main/kotlin/labs/lab3/enemy/AbstractEnemy.kt`**

```kotlin
package labs.lab3.enemy

import labs.lab3.defense.Damage

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

    override fun receiveHit(damage: Int): Damage {
        val dealt = max(1, reduceDamage(damage))
        health = max(0, health - dealt)
        return Damage(dealt)
    }

    override fun rollGoldReward(): Int = Dice.roll(goldReward / 2, goldReward)

    // наследники могут уменьшать входящий урон
    protected open fun reduceDamage(damage: Int): Int = damage
}
```

В `Battle` добавим обработку результата `receiveHit`, с учетом того, что он теперь возвращает `Damage`:

**`src/main/kotlin/labs/lab3/combat/Battle.kt`**

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
        val dealt = defender.receiveHit(hit.damage).amount
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

А еще у нас в игре появятся достижения. Выдаваться они будут с помощью паттерна "наблюдатель", но объявим мы их сразу здесь.

Представлять собой они будут идентификатор, название и описание. Как-то так:

**`src/main/kotlin/labs/lab3/progress/Achievement.kt`**

```kotlin
package labs.lab3.progress

import labs.lab3.core.UnknownCodeException

enum class Achievement(val code: String, val displayName: String, val description: String) {
    FIRST_BLOOD("first-blood", "Первая кровь", "одержать первую победу"),
    LUCKY("lucky", "Счастливчик", "найти зачарованный лут"),
    VETERAN("veteran", "Ветеран", "достичь 5 уровня"),
    DRAGON_SLAYER("dragon-slayer", "Драконоборец", "победить дракона");

    companion object {
        fun byCode(code: String): Achievement =
            entries.firstOrNull { it.code == code } ?: throw UnknownCodeException("Достижение", code)
    }
}
```

В `Hero` добавим хранение достижений, полученных за этого героя, а также возможность восстановления опыта, инвентаря и достижений через
строитель.

Нам это понадобится, когда будем добавлять механику сохранений!

А еще метод отдыха переименуем в `rest` вместо `restoreHealth`. В нем позже появится дополнительное поведение, поэтому стоит сразу немного
расширить контракт.

Итог выглядит вот так:

**`src/main/kotlin/labs/lab3/hero/Hero.kt`**

```kotlin
package labs.lab3.hero

import labs.lab3.defense.Damage
import labs.lab3.progress.Achievement
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
    var experience: Int = builder.experience
        private set
    var gold: Int = builder.gold
        private set
    var weapon: Weapon = builder.weapon!!
        private set
    var armor: Armor = builder.armor!!
        private set
    override var health: Int = 0
        private set

    private val inventoryList = builder.inventory.toMutableList()
    private val achievementList = builder.achievements.distinct().toMutableList()
    val achievements: List<Achievement> get() = achievementList.toList()

    init { rest() }

    // внутренний класс билдер - чтобы удобно и безопасно создавать героев
    class Builder {

        internal var name: String? = null
        internal var heroClass: HeroClass? = null
        internal var weapon: Weapon? = null
        internal var armor: Armor? = null
        // необязательные поля сразу получают значения по умолчанию
        internal var level: Int = 1
        internal var gold: Int = STARTING_GOLD
        internal var experience = 0
        internal var inventory: List<Item> = emptyList()
        internal var achievements: List<Achievement> = emptyList()

        fun setExperience(experience: Int) = apply { this.experience = experience }
        fun setInventory(inventory: List<Item>) = apply { this.inventory = inventory.toList() }
        fun setAchievements(achievements: List<Achievement>) = apply { this.achievements = achievements.toList() }

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

            require(experience >= 0) { "Опыт не может быть отрицательным" }
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

    override fun receiveHit(damage: Int): Damage {
        val reducedDamage = (damage * (1 - armor.defense)).roundToInt()
        // хотя бы единицу урона броня пропускает всегда
        val dealt = max(1, reducedDamage)
        health = max(0, health - dealt)
        return Damage(dealt)
    }

    // поведение героя между экспедициями

    // перед каждой экспедицией герой успевает отдохнуть
    fun rest() {
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

    fun unlock(achievement: Achievement): Boolean {
        if (achievement in achievementList) return false
        achievementList.add(achievement)
        return true
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

В `Location` заменяем вызов `restoreHealth()` на `rest()` и открываем подклассам доступ к каталогу через свойство `catalog`,
это нам скоро понадобится:

**`src/main/kotlin/labs/lab3/world/Location.kt`**

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
    protected val catalog: ItemCatalog,
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

        hero.rest()
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

Чтобы иметь возможность восстановить зачарование по идентификатору (нам это понадобится дальше), добавим в него поиск по этому самому
идентификатору:

**`src/main/kotlin/labs/lab3/item/Enchantment.kt`**

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
        fun byCode(code: String): Enchantment =
            (FOR_WEAPONS + FOR_ARMOR).firstOrNull { it.code == code }
                ?: throw labs.lab3.core.UnknownCodeException("Зачарование", code)

        val SHARP = Enchantment("sharp", "Острое", 3, 0.0, 20)
        val FIERY = Enchantment("fiery", "Огненное", 6, 0.0, 50)
        val STURDY = Enchantment("sturdy", "Прочное", 0, 0.05, 20)
        val WARDING = Enchantment("warding", "Охранное", 0, 0.10, 50)

        val FOR_WEAPONS = listOf(SHARP, FIERY)
        val FOR_ARMOR = listOf(STURDY, WARDING)
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

**`src/main/kotlin/labs/lab3/enemy/HauntedWeapon.kt`**

```kotlin
package labs.lab3.enemy

import labs.lab3.item.Weapon
import labs.lab3.core.Dice
import labs.lab3.defense.Damage
class HauntedWeapon(private val weapon: Weapon) : Enemy {
    override val code = "haunted-weapon"
    override val name get() = "Одержимый " + weapon.name
    override val maxHealth = weapon.damage * 5
    override var health = maxHealth
        private set
    override val isAlive get() = health > 0
    override val experienceReward get() = weapon.price / 2
    override fun rollGoldReward() = Dice.roll(weapon.price / 4, weapon.price / 2)
    override fun rollDamage() = weapon.rollDamage()
    override fun receiveHit(amount: Int): Damage {
        val damage = Damage(amount)
        health = maxOf(0, health - damage.amount)
        return damage
    }
}
```

А теперь объявим и саму локацию — проклятый особняк. В ней будет одно из трех случайных оружий, которые будут взяты из каталога.
Выбранное оружие будет обернуто в адаптер, то есть станет одержимо злым духом, после чего безжалостно нападет на нашего героя.

Благодаря адаптеру мы можем превратить во врага любое оружие, что безусловно разнообразит визиты в это замечательное жилище:

**`src/main/kotlin/labs/lab3/world/HauntedManor.kt`**

```kotlin
package labs.lab3.world

import labs.lab3.core.Dice
import labs.lab3.enemy.HauntedWeapon
import labs.lab3.item.DefaultCatalog
import labs.lab3.item.ItemCatalog
class HauntedManor(catalog: ItemCatalog) : Location(CODE, "Проклятый особняк", 2, catalog) {
    override val lootTable = listOf(DefaultCatalog.IRON_SWORD, DefaultCatalog.ELVEN_BOW, DefaultCatalog.EMBER_STAFF)
    override fun createEnemy() = HauntedWeapon(catalog.createWeapon(Dice.pick(lootTable)))
    companion object { const val CODE = "haunted-manor" }
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
комбинировать. Если использовать все три модификатора и учитывать их порядок, у нас получится 4 (число классов врагов) * 3 (вариантов первого модификатора) * 2 (вариантов второго модификатора) * 1 (
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

**`src/main/kotlin/labs/lab3/enemy/EnemyDecorator.kt`**

```kotlin
package labs.lab3.enemy

abstract class EnemyDecorator(protected val wrapped: Enemy) : Enemy by wrapped
```

Запись `Enemy by wrapped` поручает Kotlin сгенерировать делегирование методов и свойств вложенному врагу. В конкретных декораторах остаются только нужные переопределения.

А дальше уже реализуем конкретных декораторов, наследников базового. Они будут соответствовать нашим модификатором, начнем с легендарного:

**`src/main/kotlin/labs/lab3/enemy/LegendaryEnemy.kt`**

```kotlin
package labs.lab3.enemy

import labs.lab3.combat.Hit
class LegendaryEnemy(wrapped: Enemy) : EnemyDecorator(wrapped) {
    override val name get() = "Легендарный " + wrapped.name
    override fun rollDamage(): Hit = wrapped.rollDamage().let { it.copy(damage = it.damage * 2) }
    override val experienceReward get() = wrapped.experienceReward * 2
    override fun rollGoldReward() = wrapped.rollGoldReward() * 2
}
```

Тут важно заметить, что легендарный декоратор берет за основу результат вызова `wrapped.rollDamage()`.
Именно это и позволяет комбинировать декораторов. Внутри может оказаться не только обычный враг,
но и другая обертка: тогда вызов вернет уже модифицированное ею поведение. И так по цепочке,
пока мы не дойдем до одного из наших конкретных врагов.

Супер, базовый принцип понятен, продолжим реализовывать модификаторы, дальше у нас неистовство ветра:

**`src/main/kotlin/labs/lab3/enemy/WindfuryEnemy.kt`**

```kotlin
package labs.lab3.enemy

import labs.lab3.combat.Hit
class WindfuryEnemy(wrapped: Enemy) : EnemyDecorator(wrapped) {
    override val name get() = "Неистовый " + wrapped.name
    override fun rollDamage(): Hit {
        val first = wrapped.rollDamage()
        val second = wrapped.rollDamage()
        return Hit(first.damage + second.damage, first.critical || second.critical)
    }
    override val experienceReward get() = withBonus(wrapped.experienceReward)
    override fun rollGoldReward() = withBonus(wrapped.rollGoldReward())
    private fun withBonus(reward: Int) = reward + reward * 50 / 100
}
```

Два броска здесь складываются в один удар, поэтому защита применяется к их сумме.

Логика тут точно такая же! Модифицируем поведение, делегируем вызовы вложенному врагу.

Третий и пока финальный декоратор уменьшает входящий урон вдвое, затем передаёт получившийся урон дальше по цепочке декораторов:

**`src/main/kotlin/labs/lab3/enemy/ArmoredEnemy.kt`**

```kotlin
package labs.lab3.enemy

import labs.lab3.defense.Damage
class ArmoredEnemy(wrapped: Enemy) : EnemyDecorator(wrapped) {
    override val name get() = "Бронированный " + wrapped.name
    override fun receiveHit(amount: Int): Damage {
        require(amount >= 0) { "Урон не может быть отрицательным" }
        return if (amount == 0) Damage(0) else wrapped.receiveHit(amount - amount / 2)
    }
}
```

Отлично, все требуемые модификаторы объявлены.
Теперь мы можем добавлять их любому врагу и в любой последовательности.
При этом декоратор реализует тот же интерфейс `Enemy`, что и обычный враг, поэтому ни бой, ни локация не отличают элитного врага
от обычного, и работают с ним точно так же. Прелести `Dependency Inversion Principle`!

Осталось прописать логику, как и когда враг становится элитным. Это часть общей логики экспедиции, поэтому добавим её в локацию:
враг, созданный фабричным методом, с некоторым шансом получает комбинацию модификаторов.

**`src/main/kotlin/labs/lab3/world/Location.kt`**

```kotlin
package labs.lab3.world
import labs.lab3.enemy.ArmoredEnemy
import labs.lab3.enemy.LegendaryEnemy
import labs.lab3.enemy.WindfuryEnemy

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
    protected val catalog: ItemCatalog,
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

        hero.rest()
        var enemy = createEnemy()
        if (Dice.chance(0.2)) enemy = makeElite(enemy)
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

    private fun makeElite(original: Enemy): Enemy {
        var enemy = original
        val both = Dice.chance(0.25)
        if (Dice.chance(0.5)) {
            enemy = LegendaryEnemy(enemy)
            if (both) enemy = WindfuryEnemy(enemy)
        } else {
            enemy = WindfuryEnemy(enemy)
            if (both) enemy = LegendaryEnemy(enemy)
        }
        return if (Dice.chance(0.3)) ArmoredEnemy(enemy) else enemy
    }

    private companion object {
        const val LOOT_CHANCE = 0.6
        const val ENCHANT_CHANCE = 0.3
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

**`src/main/kotlin/labs/lab3/storage/HeroRepository.kt`**

```kotlin
package labs.lab3.storage

import labs.lab3.hero.Hero
interface HeroRepository {
    fun findByName(name: String): Hero?
    fun findAll(): List<Hero>
    fun save(hero: Hero)
}
```

Для хранения сейвов будем использовать простой файл, это не бд, но подход тоже вполне рабочий. В качестве формата сохранения выберем
json! [JSON](https://www.json.org/json-ru.html) — текстовый формат, где объекты состоят из пар "имя — значение",
а массивы хранят списки значений. Это довольно вездесущий формат, и нам он как нельзя кстати подходит.

Для чтения и записи JSON подключим библиотеку [Gson](https://google.github.io/gson/UserGuide.html).
Файл зависимостей `pom.xml` находится в корне Kotlin-проекта, рядом с папкой `src`. Полное содержимое файла:

**`pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.misis.tp</groupId>
    <artifactId>tp-lab-work-kotlin</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <kotlin.version>2.4.21</kotlin.version>
        <kotlin.compiler.jvmTarget>17</kotlin.compiler.jvmTarget>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.jetbrains.kotlin</groupId>
            <artifactId>kotlin-stdlib</artifactId>
            <version>${kotlin.version}</version>
        </dependency>
        <dependency>
            <groupId>com.google.code.gson</groupId>
            <artifactId>gson</artifactId>
            <version>2.11.0</version>
        </dependency>
    </dependencies>

    <build>
        <sourceDirectory>src/main/kotlin</sourceDirectory>
        <plugins>
            <plugin>
                <groupId>org.jetbrains.kotlin</groupId>
                <artifactId>kotlin-maven-plugin</artifactId>
                <version>${kotlin.version}</version>
                <executions>
                    <execution>
                        <id>compile</id>
                        <goals>
                            <goal>compile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
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
Из корня проекта сборка запускается командой `mvn compile`.

В файле сохранения будет массив героев со всеми нужными для восстановления объектов данными.
В `FileHeroRepository` приватные классы данных `HeroSave` и `ItemSave` описывают данные сохранения,
а Gson преобразует их в JSON и обратно.

Реализуем наконец сам репозиторий:

**`src/main/kotlin/labs/lab3/storage/FileHeroRepository.kt`**

```kotlin
package labs.lab3.storage

import com.google.gson.GsonBuilder
import com.google.gson.Strictness
import labs.lab3.hero.Hero
import labs.lab3.hero.HeroClass
import labs.lab3.item.Item
import labs.lab3.item.Weapon
import labs.lab3.item.Armor
import labs.lab3.item.ItemCatalog
import labs.lab3.item.Enchantment
import labs.lab3.progress.Achievement
import java.nio.file.Files
import java.nio.file.Path
import java.nio.charset.StandardCharsets.UTF_8

class FileHeroRepository(private val file: Path, private val catalog: ItemCatalog) : HeroRepository {
    private val gson = GsonBuilder().setPrettyPrinting().setStrictness(Strictness.STRICT).create()
    override fun findByName(name: String) = findAll().firstOrNull { it.name == name }
    override fun findAll(): List<Hero> {
        if (!Files.exists(file)) return emptyList()
        val text = Files.readString(file, UTF_8)
        if (text.isBlank()) return emptyList()
        val saves = requireNotNull(gson.fromJson(text, Array<HeroSave>::class.java)) { "В файле должен быть массив героев" }
        return saves.map { restore(it) }
    }
    override fun save(hero: Hero) {
        val saves = findAll().filter { it.name != hero.name }.map { snapshot(it) } + snapshot(hero)
        file.toAbsolutePath().parent?.let { Files.createDirectories(it) }
        Files.writeString(file, gson.toJson(saves) + "\n", UTF_8)
    }
    private fun snapshotItem(item: Item) = ItemSave(item.code, item.enchantments.map { it.code })
    private fun snapshot(hero: Hero) = HeroSave(hero.name, hero.heroClass.code, hero.level, hero.experience, hero.gold,
        snapshotItem(hero.weapon), snapshotItem(hero.armor), hero.inventory.map { snapshotItem(it) }, hero.achievements.map { it.code })
    private fun restoreItem(save: ItemSave): Item = catalog.create(save.code).also { item ->
        save.enchantments.forEach { item.enchant(Enchantment.byCode(it)) }
    }
    private fun restore(save: HeroSave): Hero = Hero.Builder().setName(save.name)
        .setHeroClass(HeroClass.byCode(save.heroClass)).setLevel(save.level).setExperience(save.experience).setGold(save.gold)
        .setWeapon(restoreItem(save.weapon) as Weapon).setArmor(restoreItem(save.armor) as Armor)
        .setInventory(save.inventory.map { restoreItem(it) }).setAchievements(save.achievements.map { Achievement.byCode(it) }).build()

    private data class ItemSave(val code: String, val enchantments: List<String>)
    private data class HeroSave(val name: String, val heroClass: String, val level: Int, val experience: Int, val gold: Int,
        val weapon: ItemSave, val armor: ItemSave, val inventory: List<ItemSave>, val achievements: List<String>)
}
```

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

**`src/main/kotlin/labs/lab3/storage/CachingHeroRepository.kt`**

```kotlin
package labs.lab3.storage

import labs.lab3.hero.Hero
class CachingHeroRepository(private val target: HeroRepository) : HeroRepository {
    private var cache: MutableMap<String, Hero>? = null
    private fun cached(): MutableMap<String, Hero> {
        if (cache == null) cache = target.findAll().associateBy { it.name }.toMutableMap()
        return cache!!
    }
    override fun findByName(name: String) = cached()[name]
    override fun findAll() = cached().values.toList()
    override fun save(hero: Hero) { target.save(hero); cached()[hero.name] = hero }
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

Можно было бы прописать все тактики прямо в `Battle` через `when`, однако тогда каждая новая тактика требовала бы
правки этого класса. А еще нарушался бы `SRP`.

Так как в данном случае все изменения сводятся к выбору между разными алгоритми выполнения одного и того же действия
(выбора, что делать в раунде), можно применить паттерн *Стратегия*.

Он определяет семейство алгоритмов, выносит каждый
из них в отдельный класс и делает их взаимозаменяемыми: тот, кто алгоритм использует, работает с общим интерфейсом и
не знает, какая именно реализация ему досталась.

Сначала опишем, что герой вообще может сделать в раунде:

**`src/main/kotlin/labs/lab3/tactic/Action.kt`**

```kotlin
package labs.lab3.tactic

enum class Action(val attackMultiplier: Double, val incomingMultiplier: Double) {
    ATTACK(1.0, 1.0), POWER_STRIKE(1.5, 1.5), DEFEND(0.5, 0.3)
}
```

Получается, что он может вести себя агрессивно (увеличивается и исходящий, и входящий урон), сбалансированно (ничего не меняется) и
осторожно, уходя в оборону (уменьшается и входящий, и исходящий урон).

Выбирать действия мы будем в зависимости от состояния врага и героя, определим базовый интерфейс тактики:

**`src/main/kotlin/labs/lab3/tactic/Tactic.kt`**

```kotlin
package labs.lab3.tactic

import labs.lab3.combat.Combatant
interface Tactic {
    val code: String
    val displayName: String
    val description: String
    fun chooseAction(round: Int, hero: Combatant, enemy: Combatant): Action
}
```

Реализуем самый простой алгоритм выбора действия - обычный удар каждый раунд:

**`src/main/kotlin/labs/lab3/tactic/BalancedTactic.kt`**

```kotlin
package labs.lab3.tactic

import labs.lab3.combat.Combatant
class BalancedTactic : Tactic {
    override val code = "balanced"
    override val displayName = "Баланс"
    override val description = "обычный удар каждый раунд"
    override fun chooseAction(round: Int, hero: Combatant, enemy: Combatant) = Action.ATTACK
}
```

Натиск - пока здоровье врага выше, чем у героя, герой бьет безрассудно, нанося и получая в ответ больше урона:

**`src/main/kotlin/labs/lab3/tactic/AggressiveTactic.kt`**

```kotlin
package labs.lab3.tactic

import labs.lab3.combat.Combatant
class AggressiveTactic : Tactic {
    override val code = "aggressive"
    override val displayName = "Натиск"
    override val description = "совершать безрассудные удары, пока у врага больше здоровья, чем у героя"
    override fun chooseAction(round: Int, hero: Combatant, enemy: Combatant) = if (enemy.health > hero.health) Action.POWER_STRIKE else Action.ATTACK
}
```

И осторожность - раненый герой через раунд будет уходить в оборону:

**`src/main/kotlin/labs/lab3/tactic/CautiousTactic.kt`**

```kotlin
package labs.lab3.tactic

import labs.lab3.combat.Combatant
class CautiousTactic : Tactic {
    override val code = "cautious"
    override val displayName = "Осторожность"
    override val description = "при здоровье ниже 40% через раунд уходить в оборону"
    override fun chooseAction(round: Int, hero: Combatant, enemy: Combatant) = if (hero.health < hero.maxHealth * 0.4 && round % 2 == 0) Action.DEFEND else Action.ATTACK
}
```

Также дополним типы ивентов в нашей игре, добавим наши действия, и `EVADE`/`ABSORB`, они нам пригодятся дальше:

**`src/main/kotlin/labs/lab3/combat/EventType.kt`**

```kotlin
package labs.lab3.combat

enum class EventType {
    HIT,
    CRITICAL_HIT,
    POWER_STRIKE, DEFEND, EVADE, ABSORB,
    DEATH,
}
```

Вывод новых событий боя:

**`src/main/kotlin/labs/lab3/ConsoleView.kt`**

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
        if (hero.achievements.isNotEmpty()) println("  Достижения: " + hero.achievements.joinToString { it.displayName })
    }

    fun summarize(report: ExpeditionReport, locationName: String, tacticName: String): String {
        val result = if (report.battle.outcome == Outcome.VICTORY) "победа" else "поражение"
        return "$locationName, ${report.battle.enemy.name}, тактика \"$tacticName\": $result"
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
            EventType.EVADE -> println("[раунд ${event.round}] ${source.name} -> ${target.name}: промах, ${target.name} уклоняется")
            EventType.ABSORB -> println("[раунд ${event.round}] ${source.name} -> ${target.name}: барьер поглощает ${event.amount} урона")
            EventType.POWER_STRIKE -> println("[раунд ${event.round}] ${source.name} бьет в полную силу")
            EventType.DEFEND -> println("[раунд ${event.round}] ${source.name} уходит в оборону")
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

Теперь обновим сам `Battle`. Тактику он получает через конструктор и каждый раунд спрашивает у нее, что именно делает герой:

**`src/main/kotlin/labs/lab3/combat/Battle.kt`**

```kotlin
package labs.lab3.combat

import labs.lab3.tactic.*
import kotlin.math.roundToInt

// бой идет по раундам: сначала бьет герой, потом враг, пока кто-то не погибнет
class Battle(private val hero: Combatant, private val enemy: Combatant, private val tactic: Tactic = BalancedTactic()) {

    private val events = mutableListOf<BattleEvent>()
    private var round = 0

    fun run(): BattleReport {
        while (hero.isAlive && enemy.isAlive) {
            round++
            val action = tactic.chooseAction(round, hero, enemy)
            when (action) {
                Action.POWER_STRIKE -> emit(BattleEvent(round, EventType.POWER_STRIKE, Side.HERO, Side.ENEMY, 0, 0))
                Action.DEFEND -> emit(BattleEvent(round, EventType.DEFEND, Side.HERO, Side.HERO, 0, 0))
                else -> Unit
            }
            strike(hero, Side.HERO, enemy, Side.ENEMY, action.attackMultiplier)
            if (enemy.isAlive) {
                strike(enemy, Side.ENEMY, hero, Side.HERO, action.incomingMultiplier)
            }
        }

        val outcome = if (hero.isAlive) Outcome.VICTORY else Outcome.DEFEAT
        return BattleReport(Participant.of(hero), Participant.of(enemy), outcome, events.toList())
    }

    private fun strike(attacker: Combatant, source: Side, defender: Combatant, target: Side, multiplier: Double) {
        val hit = attacker.rollDamage()
        val damage = defender.receiveHit(maxOf(1, (hit.damage * multiplier).roundToInt()))
        if (damage.evaded) {
            emit(BattleEvent(round, EventType.EVADE, source, target, 0, defender.health))
            return
        }
        if (damage.absorbed > 0) emit(BattleEvent(round, EventType.ABSORB, source, target, damage.absorbed, defender.health))
        if (damage.amount > 0) {
            val type = if (hit.critical) EventType.CRITICAL_HIT else EventType.HIT
            emit(BattleEvent(round, type, source, target, damage.amount, defender.health))
        }

        if (!defender.isAlive) {
            emit(BattleEvent(round, EventType.DEATH, source, target, 0, 0))
        }
    }

    private fun emit(event: BattleEvent) {
        events.add(event)
    }
}
```

Обратите внимание, что бой ничего не знает о конкретных тактиках: он работает с интерфейсом `Tactic`, и
подставить ему можно любую реализацию. Если появится новая тактика, `Battle` менять не придется.

Ну и в отчете об экспедиции будем запоминать, с какой тактикой герой ее начинал:

**`src/main/kotlin/labs/lab3/world/ExpeditionReport.kt`**

```kotlin
package labs.lab3.world

import labs.lab3.combat.BattleReport

// отчет об экспедиции: с каким сидом, куда, как прошел бой и что получил герой
data class ExpeditionReport(val seed: Int, val locationCode: String,
    val tacticCode: String, val battle: BattleReport, val rewards: Rewards)
```

Стратегия позволяет нам соблюдать `Open Closed` принцип, он дает нам возможность расширять поведение нашего класса без его изменения, путем
добавления новых реализаций стратегии. Стратегию можно выбирать как в методе, где она используется, так и в конструкторе, все зависит от
конкретной необходимости. В нашем случае стратегия выбирается в конструкторе, потому что по ходу боя она не меняется.

Применять стратегию стоит, когда одно и то же действие можно выполнить несколькими способами и способ нужно выбирать
или менять во время работы программы.

И давайте сразу обновим `Location`: она будет принимать тактику и передавать дальше в `Battle`.

Получится что-то такое:

**`src/main/kotlin/labs/lab3/world/Location.kt`**

```kotlin
package labs.lab3.world

import labs.lab3.tactic.*
import labs.lab3.enemy.ArmoredEnemy
import labs.lab3.enemy.LegendaryEnemy
import labs.lab3.enemy.WindfuryEnemy
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
    protected val catalog: ItemCatalog,
) {

    // коды предметов из каталога, которые можно найти в локации
    protected abstract val lootTable: List<String>

    // создание врага делегируется подклассам для повышения гибкости
    protected abstract fun createEnemy(): Enemy

    // бизнес-логика, общая для всех локаций:
    // герой встречает врага, сражается с ним и получает награду, если победил;
    // если сид не передан, берется случайный
    fun explore(hero: Hero, seed: Int = Dice.nextSeed(), tactic: Tactic = BalancedTactic()): ExpeditionReport {
        Dice.reseed(seed)

        hero.rest()
        var enemy = createEnemy()
        if (Dice.chance(0.2)) enemy = makeElite(enemy)
        val battle = Battle(hero, enemy, tactic).run()

        val rewards = if (battle.outcome == Outcome.VICTORY) reward(hero, enemy) else punish(hero)

        return ExpeditionReport(seed, code, tactic.code, battle, rewards)
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

    private fun makeElite(original: Enemy): Enemy {
        var enemy = original
        val both = Dice.chance(0.25)
        if (Dice.chance(0.5)) {
            enemy = LegendaryEnemy(enemy)
            if (both) enemy = WindfuryEnemy(enemy)
        } else {
            enemy = WindfuryEnemy(enemy)
            if (both) enemy = LegendaryEnemy(enemy)
        }
        return if (Dice.chance(0.3)) ArmoredEnemy(enemy) else enemy
    }

    private companion object {
        const val LOOT_CHANCE = 0.6
        const val ENCHANT_CHANCE = 0.3
    }
}
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

**`src/main/kotlin/labs/lab3/event/GameEventType.kt`**

```kotlin
package labs.lab3.event

enum class GameEventType { VICTORY, DEFEAT, LEVEL_UP }
```

И само событие. Обратите внимание, что это просто данные: что произошло, с каким героем и в какой экспедиции:

**`src/main/kotlin/labs/lab3/event/GameEvent.kt`**

```kotlin
package labs.lab3.event

import labs.lab3.hero.Hero
import labs.lab3.world.ExpeditionReport
data class GameEvent(val type: GameEventType, val hero: Hero, val expedition: ExpeditionReport)
```

Интерфейс для подписчиков содержит всего один метод, который вызывается при наступлении события, и определяет контракт обработки этого
изменения:

**`src/main/kotlin/labs/lab3/event/GameEventListener.kt`**

```kotlin
package labs.lab3.event

fun interface GameEventListener { fun onEvent(event: GameEvent) }
```

`fun interface` задает интерфейс с одним абстрактным методом: слушателя можно написать отдельным классом или передать лямбду.

Именно издатель позволяет подписываться на определенный тип события. Также именно он оповещает подписчиков о
наступлении того или иного события:

**`src/main/kotlin/labs/lab3/event/GameEventPublisher.kt`**

```kotlin
package labs.lab3.event

class GameEventPublisher {
    private val listeners = mutableMapOf<GameEventType, MutableList<GameEventListener>>()
    fun subscribe(type: GameEventType, listener: GameEventListener) {
        listeners.getOrPut(type) { mutableListOf() }.add(listener)
    }
    fun unsubscribe(type: GameEventType, listener: GameEventListener) { listeners[type]?.remove(listener) }
    fun publish(event: GameEvent) { listeners[event.type]?.toList()?.forEach { it.onEvent(event) } }
}
```

Подписчики хранятся в `listeners`, причем хранятся по типу ивента, то есть подписаться и реагировать они могут на отдельные типы, а не на
все.
Реакции вызываются в методе `publish`. Когда в нашей системе происходит ивент – мы вызываем издателя, а он, по списку зарегистрированных
подписчиков, вызывает их
`onEvent` методы, прокидывая туда информацию о ивенте.

Теперь реализуем двух конкретных подписчиков. Сначала того, который будет выдавать достижения:

**`src/main/kotlin/labs/lab3/progress/AchievementTracker.kt`**

```kotlin
package labs.lab3.progress

import labs.lab3.event.*
class AchievementTracker : GameEventListener {
    override fun onEvent(event: GameEvent) {
        val hero = event.hero
        if (event.type == GameEventType.VICTORY) {
            hero.unlock(Achievement.FIRST_BLOOD)
            if (event.expedition.battle.enemy.code == "dragon") hero.unlock(Achievement.DRAGON_SLAYER)
            if (event.expedition.rewards.loot?.enchantments?.isNotEmpty() == true) hero.unlock(Achievement.LUCKY)
        } else if (event.type == GameEventType.LEVEL_UP && hero.level >= 5) hero.unlock(Achievement.VETERAN)
    }
}
```

А теперь второго, он будет записывать экспедиции в журнал:

**`src/main/kotlin/labs/lab3/progress/ExpeditionJournal.kt`**

```kotlin
package labs.lab3.progress

import labs.lab3.event.*
import labs.lab3.world.ExpeditionReport
class ExpeditionJournal : GameEventListener {
    private val entries = mutableMapOf<String, MutableList<ExpeditionReport>>()
    override fun onEvent(event: GameEvent) { entries.getOrPut(event.hero.name) { mutableListOf() }.add(event.expedition) }
    fun getEntries(name: String): List<ExpeditionReport> = entries[name]?.toList() ?: emptyList()
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

**`src/main/kotlin/labs/lab3/defense/DamageHandler.kt`**

```kotlin
package labs.lab3.defense

abstract class DamageHandler {
    private var next: DamageHandler? = null
    fun setNext(next: DamageHandler): DamageHandler { this.next = next; return next }
    abstract fun handle(damage: Damage)
    protected fun passToNext(damage: Damage) { next?.handle(damage) }
}
```

Теперь конкретный обработчик, который будет хендлить уклонение: если оно сработало, удар по герою не прошел, и передавать урон дальше по
цепочке незачем:

**`src/main/kotlin/labs/lab3/defense/EvasionHandler.kt`**

```kotlin
package labs.lab3.defense

import labs.lab3.core.Dice
class EvasionHandler(private val chance: Double) : DamageHandler() {
    override fun handle(damage: Damage) {
        if (Dice.chance(chance)) { damage.evade(); return }
        passToNext(damage)
    }
}
```

Броня гасит долю урона и передает то, что осталось, следующему обработчику. Обратите внимание, что это то же самое правило,
что раньше было зашито в герое, скелете и драконе, а теперь оно записано в одном месте:

**`src/main/kotlin/labs/lab3/defense/ArmorHandler.kt`**

```kotlin
package labs.lab3.defense

import kotlin.math.roundToInt
class ArmorHandler(private val defense: Double) : DamageHandler() {
    override fun handle(damage: Damage) {
        damage.reduceTo(maxOf(1, (damage.amount * (1 - defense)).roundToInt()))
        passToNext(damage)
    }
}
```

И магический барьер, который поглощает урон, пока у него есть запас прочности:

**`src/main/kotlin/labs/lab3/defense/BarrierHandler.kt`**

```kotlin
package labs.lab3.defense

class BarrierHandler(private var strength: Int) : DamageHandler() {
    override fun handle(damage: Damage) {
        val absorbed = minOf(strength, damage.amount)
        if (absorbed > 0) { strength -= absorbed; damage.absorb(absorbed) }
        passToNext(damage)
    }
}
```

Какие звенья будут в цепочке, зависит от героя. Цепочку собирает его класс - так же, как класс сам знает свою
фабрику стартового снаряжения:

**`src/main/kotlin/labs/lab3/hero/HeroClass.kt`**

```kotlin
package labs.lab3.hero

import labs.lab3.defense.*
import labs.lab3.item.Armor
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

    fun createDefense(armor: Armor, level: Int): DamageHandler {
        var evasion = if (this == RANGER) 0.15 else 0.0
        var barrier = if (this == MAGE) 30 else 0
        when {
            this == WARRIOR && level >= 3 -> barrier = 20
            this == RANGER && level >= 5 -> barrier = 15
            this == MAGE && level >= 3 -> evasion = 0.10
        }
        val armorHandler = ArmorHandler(armor.defense)
        var first: DamageHandler = armorHandler
        if (evasion > 0) {
            first = EvasionHandler(evasion)
            first.setNext(armorHandler)
        }
        if (barrier > 0) armorHandler.setNext(BarrierHandler(barrier))
        return first
    }

    companion object {
        fun byCode(code: String): HeroClass =
            entries.firstOrNull { it.code == code } ?: throw UnknownCodeException("Класс героя", code)
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

**`src/main/kotlin/labs/lab3/hero/Hero.kt`**

```kotlin
package labs.lab3.hero

import labs.lab3.defense.Damage
import labs.lab3.defense.DamageHandler
import labs.lab3.progress.Achievement
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
    var experience: Int = builder.experience
        private set
    var gold: Int = builder.gold
        private set
    var weapon: Weapon = builder.weapon!!
        private set
    var armor: Armor = builder.armor!!
        private set
    override var health: Int = 0
        private set

    private val inventoryList = builder.inventory.toMutableList()
    private val achievementList = builder.achievements.distinct().toMutableList()
    private lateinit var defense: DamageHandler
    val achievements: List<Achievement> get() = achievementList.toList()

    init { rest() }

    // внутренний класс билдер - чтобы удобно и безопасно создавать героев
    class Builder {

        internal var name: String? = null
        internal var heroClass: HeroClass? = null
        internal var weapon: Weapon? = null
        internal var armor: Armor? = null
        // необязательные поля сразу получают значения по умолчанию
        internal var level: Int = 1
        internal var gold: Int = STARTING_GOLD
        internal var experience = 0
        internal var inventory: List<Item> = emptyList()
        internal var achievements: List<Achievement> = emptyList()

        fun setExperience(experience: Int) = apply { this.experience = experience }
        fun setInventory(inventory: List<Item>) = apply { this.inventory = inventory.toList() }
        fun setAchievements(achievements: List<Achievement>) = apply { this.achievements = achievements.toList() }

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

            require(experience >= 0) { "Опыт не может быть отрицательным" }
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

    override fun receiveHit(amount: Int): Damage {
        val damage = Damage(amount)
        defense.handle(damage)
        health = maxOf(0, health - damage.amount)
        return damage
    }

    // поведение героя между экспедициями

    // перед каждой экспедицией герой успевает отдохнуть
    fun rest() {
        health = maxHealth
        defense = heroClass.createDefense(armor, level)
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
                defense = heroClass.createDefense(armor, level)
            }
            else -> throw IllegalArgumentException("Предмет нельзя экипировать: ${item.name}")
        }
    }

    fun unlock(achievement: Achievement): Boolean {
        if (achievement in achievementList) return false
        achievementList.add(achievement)
        return true
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

У врагов цепочка задается прямо в конструкторе. Общий класс врага пропускает урон через цепочку, если она есть:

**`src/main/kotlin/labs/lab3/enemy/AbstractEnemy.kt`**

```kotlin
package labs.lab3.enemy

import labs.lab3.defense.Damage
import labs.lab3.defense.DamageHandler
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
    private val defense: DamageHandler? = null,
) : Enemy {

    // final - наследники не смогут переопределить свойство, а значит, менять здоровье может только этот класс
    final override var health: Int = maxHealth
        private set

    override val isAlive: Boolean
        get() = health > 0

    // методы с override сами по себе открыты, поэтому наследники смогут переопределить и этот метод
    override fun rollDamage(): Hit = Hit(Dice.roll(minDamage, maxDamage), false)

    override fun receiveHit(amount: Int): Damage {
        val damage = Damage(amount)
        defense?.handle(damage)
        health = maxOf(0, health - damage.amount)
        return damage
    }

    override fun rollGoldReward(): Int = Dice.roll(goldReward / 2, goldReward)

}
```

У гоблина защиты нет:

**`src/main/kotlin/labs/lab3/enemy/Goblin.kt`**

```kotlin
package labs.lab3.enemy

// гоблин - самый простой враг, без особенностей
class Goblin : AbstractEnemy("goblin", "Гоблин", 50, 6, 14, 40, 30)
```

Кости скелета и чешуя дракона работают как броня, поэтому метод `reduceDamage` им больше не нужен:

**`src/main/kotlin/labs/lab3/enemy/Skeleton.kt`**

```kotlin
package labs.lab3.enemy

import labs.lab3.defense.ArmorHandler

import kotlin.math.roundToInt

// скелет: кости плохо поддаются ударам, поэтому часть урона проходит мимо
class Skeleton : AbstractEnemy("skeleton", "Скелет", 110, 10, 24, 90, 70, ArmorHandler(RESISTANCE)) {


    private companion object {
        const val RESISTANCE = 0.3
    }
}
```

**`src/main/kotlin/labs/lab3/enemy/Dragon.kt`**

```kotlin
package labs.lab3.enemy

import labs.lab3.defense.ArmorHandler

import labs.lab3.combat.Hit
import labs.lab3.core.Dice
import kotlin.math.roundToInt

// дракон защищен чешуей и время от времени дышит огнем
class Dragon : AbstractEnemy("dragon", "Дракон", 260, 16, 22, 500, 400, ArmorHandler(RESISTANCE)) {

    override fun rollDamage(): Hit {
        val bite = super.rollDamage()
        if (Dice.chance(FIRE_BREATH_CHANCE)) {
            return Hit(bite.damage * FIRE_BREATH_MULTIPLIER, true)
        }

        return bite
    }


    private companion object {
        const val RESISTANCE = 0.25
        const val FIRE_BREATH_CHANCE = 0.25
        const val FIRE_BREATH_MULTIPLIER = 2
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

**`src/main/kotlin/labs/lab3/game/GameFacade.kt`**

```kotlin
package labs.lab3.game

import java.nio.file.Path
import labs.lab3.camp.Merchant
import labs.lab3.combat.Outcome
import labs.lab3.core.UnknownCodeException
import labs.lab3.hero.Hero
import labs.lab3.hero.HeroClass
import labs.lab3.item.DefaultCatalog
import labs.lab3.item.Item
import labs.lab3.world.*
import labs.lab3.tactic.*
import labs.lab3.event.*
import labs.lab3.progress.*
import labs.lab3.storage.*

class GameFacade(saveFile: Path) {
    private val catalog = DefaultCatalog.create()
    private val registry = LocationRegistry(listOf(Forest(catalog), HauntedManor(catalog), Crypt(catalog), DragonLair(catalog)))
    val tactics: List<Tactic> = listOf(BalancedTactic(), AggressiveTactic(), CautiousTactic())
    private val merchant = Merchant(catalog, listOf(DefaultCatalog.IRON_SWORD, DefaultCatalog.ELVEN_BOW,
        DefaultCatalog.EMBER_STAFF, DefaultCatalog.CHAINMAIL, DefaultCatalog.PLATE_ARMOR))
    private val heroesRepository: HeroRepository = CachingHeroRepository(FileHeroRepository(saveFile, catalog))
    private val events = GameEventPublisher()
    private val journal = ExpeditionJournal()
    init {
        val achievements = AchievementTracker()
        events.subscribe(GameEventType.VICTORY, achievements)
        events.subscribe(GameEventType.LEVEL_UP, achievements)
        events.subscribe(GameEventType.VICTORY, journal)
        events.subscribe(GameEventType.DEFEAT, journal)
    }
    val heroClasses get() = HeroClass.entries
    fun getStarterKit(code: String): List<Item> = HeroClass.byCode(code).getStarterKit(catalog).let {
        listOf(it.createWeapon(), it.createArmor())
    }
    val locations get() = registry.all
    fun getLocation(code: String) = registry.get(code)
    fun getTactic(code: String): Tactic = tactics.firstOrNull { it.code == code } ?: throw UnknownCodeException("Тактика", code)
    val showcase get() = merchant.showcase
    fun getSellPrice(item: Item) = merchant.getSellPrice(item)
    val heroes get() = heroesRepository.findAll()
    fun getHero(name: String): Hero = requireNotNull(heroesRepository.findByName(name)) { "Героя с именем $name нет" }
    fun createHero(name: String, code: String): Hero {
        require(heroesRepository.findByName(name) == null) { "Герой с именем $name уже есть" }
        val heroClass = HeroClass.byCode(code)
        val hero = Hero.Builder().setName(name).setHeroClass(heroClass).setStarterKit(heroClass.getStarterKit(catalog)).build()
        heroesRepository.save(hero)
        return hero
    }
    fun explore(name: String, locationCode: String, tacticCode: String): ExpeditionReport {
        val hero = getHero(name)
        val report = getLocation(locationCode).explore(hero, tactic = getTactic(tacticCode))
        val result = if (report.battle.outcome == Outcome.VICTORY) GameEventType.VICTORY else GameEventType.DEFEAT
        events.publish(GameEvent(result, hero, report))
        if (report.rewards.levelsGained > 0) events.publish(GameEvent(GameEventType.LEVEL_UP, hero, report))
        heroesRepository.save(hero)
        return report
    }
    fun equip(name: String, index: Int): Item {
        val hero = getHero(name)
        val item = hero.inventory[index]
        hero.equip(item)
        heroesRepository.save(hero)
        return item
    }
    fun buy(name: String, code: String): Item {
        val hero = getHero(name)
        val item = merchant.buy(hero, code)
        heroesRepository.save(hero)
        return item
    }
    fun sell(name: String, index: Int): Int {
        val hero = getHero(name)
        val price = merchant.sell(hero, hero.inventory[index])
        heroesRepository.save(hero)
        return price
    }
    fun getJournal(name: String) = journal.getEntries(name)
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

А игровой цикл в методе `main` теперь работает только с фасадом и хранит текущего героя:

Вместо прямой работы с каталогом, локациями и торговцем:

- создаём `GameFacade` с файлом сохранения и выбираем нового или сохранённого героя;
- перед экспедицией выбираем тактику;
- работам с экспедициями, экипировкой и торговлей через фасад;
- добавляем пункт меню для журнала экспедиций.

**`src/main/kotlin/labs/lab3/Main.kt`**

```kotlin
package labs.lab3

import labs.lab3.game.GameFacade
import java.nio.file.Path
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
    val saveFile = Path.of("saves", "heroes.json")
    println("Файл сохранения: " + saveFile.toAbsolutePath().normalize())
    val game = GameFacade(saveFile)
    val hero = chooseHero(game)
    println("${hero.name} разбивает лагерь на опушке мистического леса.")

    while (true) {
        println()
        view.printHero(hero)
        println("1 - отправиться в экспедицию, 2 - снаряжение, 3 - торговец, 4 - журнал, 0 - выйти из игры")
        when (readInput()) {
            "1" -> goOnExpedition(hero, game)
            "2" -> manageEquipment(hero, game)
            "3" -> visitMerchant(hero, game)
            "4" -> game.getJournal(hero.name).forEach {
                println(view.summarize(it, game.getLocation(it.locationCode).displayName, game.getTactic(it.tacticCode).displayName))
            }
            "0" -> return
            else -> println("Такого варианта нет")
        }
    }
}

private fun chooseHero(game: GameFacade): Hero {
    val heroes = game.heroes
    if (heroes.isEmpty()) return createHero(game)
    println("Выберите сохраненного героя (0 - создать нового):")
    heroes.forEachIndexed { i, hero -> println("${i + 1} - ${hero.name}, ${hero.heroClass.displayName} ${hero.level} ур.") }
    val index = readChoice(heroes.size)
    return if (index >= 0) heroes[index] else createHero(game)
}

private fun createHero(game: GameFacade): Hero {
    println("Создайте своего героя!")
    while (true) {
        println("Выберите класс:")
        val heroClasses = HeroClass.entries
        heroClasses.forEachIndexed { i, heroClass ->
            val kit = game.getStarterKit(heroClass.code)
            println(
                "${i + 1} - ${heroClass.displayName} (здоровье ${heroClass.baseHealth}): " +
                    "${view.describe(kit[0])}, ${view.describe(kit[1])}"
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
            return game.createHero(name, heroClass.code)
        } catch (e: IllegalArgumentException) {
            println("Не получилось создать героя: " + e.message)
        }
    }
}

private fun goOnExpedition(hero: Hero, game: GameFacade) {
    val all = game.locations
    println("Куда отправимся?")
    all.forEachIndexed { i, location ->
        println("${i + 1} - ${location.displayName} (рекомендуемый уровень ${location.recommendedLevel})")
    }
    val index = readChoice(all.size)
    if (index < 0) {
        return
    }

    val location = all[index]
    println("Выберите тактику:")
    game.tactics.forEachIndexed { i, tactic -> println("${i + 1} - ${tactic.displayName}: ${tactic.description}") }
    val tacticIndex = readChoice(game.tactics.size)
    if (tacticIndex < 0) return
    val report = game.explore(hero.name, location.code, game.tactics[tacticIndex].code)
    println()
    view.printReport(report, location.displayName)

    if (report.battle.outcome == Outcome.VICTORY && report.battle.enemy.code == "dragon") {
        println("\nДракон повержен! Барды еще долго будут петь о подвиге героя по имени ${hero.name}.")
    }
}

private fun manageEquipment(hero: Hero, game: GameFacade) {
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
    game.equip(hero.name, index)
    println("Экипировано: " + view.describe(item))
}

private fun visitMerchant(hero: Hero, game: GameFacade) {
    println("Торговец: \"Чего желаешь?\" (золото: ${hero.gold})")
    println("1 - купить, 2 - продать, 0 - назад")
    when (readInput()) {
        "1" -> buy(hero, game)
        "2" -> sell(hero, game)
    }
}

private fun buy(hero: Hero, game: GameFacade) {
    val showcase = game.showcase
    showcase.forEachIndexed { i, item ->
        println("${i + 1} - ${view.describe(item)}: ${item.price} золота")
    }
    val index = readChoice(showcase.size)
    if (index < 0) {
        return
    }

    try {
        val item = game.buy(hero.name, showcase[index].code)
        println("Куплено: " + view.describe(item))
    } catch (e: IllegalStateException) {
        println(e.message)
    }
}

private fun sell(hero: Hero, game: GameFacade) {
    val inventory = hero.inventory
    if (inventory.isEmpty()) {
        println("Продавать нечего (экипированные предметы продать нельзя)")
        return
    }

    inventory.forEachIndexed { i, item ->
        println("${i + 1} - ${view.describe(item)}: ${game.getSellPrice(item)} золота")
    }
    val index = readChoice(inventory.size)
    if (index < 0) {
        return
    }

    val item = inventory[index]
    val price = game.sell(hero.name, index)
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

Супер! Попробуйте запустить игру, сохраниться и вернуться к своему герою :)

## Демонстрация паттернов

Чтобы показать все паттерны одним запуском, напишем объект `Demo`:

Демонстрацию лабы 3 заменяем новой:

**`src/main/kotlin/labs/lab3/Demo.kt`**

```kotlin
package labs.lab3

import labs.lab3.core.Dice
import labs.lab3.combat.Outcome
import labs.lab3.defense.*
import labs.lab3.enemy.*
import labs.lab3.hero.*
import labs.lab3.item.*
import labs.lab3.tactic.*
import labs.lab3.world.*
import labs.lab3.event.*
import labs.lab3.progress.*
import labs.lab3.storage.*
import labs.lab3.game.GameFacade
import java.nio.file.Files

object Demo {
    private val view = ConsoleView()
    private fun newHero(name: String, heroClass: HeroClass, level: Int, catalog: ItemCatalog) =
        Hero.Builder().setName(name).setHeroClass(heroClass).setStarterKit(heroClass.getStarterKit(catalog)).setLevel(level).build()

    private fun printHits(title: String, chain: DamageHandler, count: Int) {
        val results = (1..count).map {
            val damage = Damage(30)
            chain.handle(damage)
            when {
                damage.evaded -> "уклонение"
                damage.absorbed > 0 -> "поглощено ${damage.absorbed}, прошло ${damage.amount}"
                else -> "прошло ${damage.amount}"
            }
        }
        println("$title: " + results.joinToString("; "))
    }

    @JvmStatic
    fun main(args: Array<String>) {
        val catalog = DefaultCatalog.create()
        println("===== Цепочка обязанностей =====")
        printHits("Броня 30%", ArmorHandler(0.3), 2)
        val armor = ArmorHandler(0.3)
        armor.setNext(BarrierHandler(30))
        printHits("Броня 30% -> барьер 30", armor, 3)
        Dice.reseed(3)
        val chain = EvasionHandler(0.5)
        chain.setNext(ArmorHandler(0.3)).setNext(BarrierHandler(30))
        printHits("Уклонение 50% -> броня 30% -> барьер 30", chain, 3)

        println("\n===== Стратегия =====")
        for (tactic in listOf(BalancedTactic(), AggressiveTactic(), CautiousTactic())) {
            val mage = newHero("Мерлин", HeroClass.MAGE, 2, catalog)
            val report = Crypt(catalog).explore(mage, 9, tactic)
            val result = if (report.battle.outcome == Outcome.VICTORY) "победа" else "поражение"
            println("${tactic.displayName}: $result, раундов ${report.battle.events.last().round}, у героя осталось ${mage.health}/${mage.maxHealth} здоровья")
        }
        println("\n===== Декоратор =====")
        for (enemy in listOf(Goblin(), LegendaryEnemy(Goblin()), WindfuryEnemy(Goblin()),
            LegendaryEnemy(WindfuryEnemy(Goblin())), ArmoredEnemy(Goblin()), ArmoredEnemy(LegendaryEnemy(WindfuryEnemy(Dragon()))))) {
            Dice.reseed(5)
            val hits = (1..3).map { enemy.rollDamage().damage }.joinToString()
            val damage = enemy.receiveHit(20)
            println("${enemy.name}: удары $hits, награда ${enemy.experienceReward} опыта; входящий урон 20 -> ${damage.amount}, здоровье ${enemy.health}/${enemy.maxHealth}")
        }
        println("\n===== Адаптер =====")
        val bow = catalog.createWeapon(DefaultCatalog.ELVEN_BOW)
        val haunted = HauntedWeapon(bow)
        println("Предмет: " + view.describe(bow))
        println("Враг:    ${haunted.name} (${haunted.maxHealth} здоровья, награда ${haunted.experienceReward} опыта)")
        val manor = HauntedManor(catalog)
        view.printReport(manor.explore(newHero("Леголас", HeroClass.RANGER, 2, catalog), 55, CautiousTactic()), manor.displayName)

        println("\n===== Наблюдатель =====")
        val events = GameEventPublisher()
        val journal = ExpeditionJournal()
        events.subscribe(GameEventType.VICTORY, AchievementTracker())
        events.subscribe(GameEventType.VICTORY, journal)
        events.subscribe(GameEventType.DEFEAT, journal)
        events.subscribe(GameEventType.VICTORY) { println("Событие ${it.type}: герой ${it.hero.name}") }
        val warrior = newHero("Артур", HeroClass.WARRIOR, 1, catalog)
        val report = Forest(catalog).explore(warrior, 7)
        val result = if (report.battle.outcome == Outcome.VICTORY) GameEventType.VICTORY else GameEventType.DEFEAT
        events.publish(GameEvent(result, warrior, report))
        println("Достижения героя: " + warrior.achievements.joinToString { it.displayName })
        println("Записей в журнале: " + journal.getEntries(warrior.name).size)

        println("\n===== Заместитель =====")
        val filePath = Files.createTempFile("heroes", ".json")
        try {
            val file = FileHeroRepository(filePath, catalog)
            file.save(warrior)
            println("Содержимое файла:")
            print(Files.readString(filePath))
            println("Без заместителя два поиска дают один и тот же объект: " + (file.findByName("Артур") === file.findByName("Артур")))
            val cached = CachingHeroRepository(file)
            val hero = cached.findByName("Артур")!!
            println("С заместителем два поиска дают один и тот же объект: " + (hero === cached.findByName("Артур")))
            hero.addGold(100)
            cached.save(hero)
            println("После сохранения через заместитель золото героя в файле: " + file.findByName("Артур")!!.gold)
        } finally { Files.deleteIfExists(filePath) }

        println("\n===== Фасад =====")
        val gameFile = Files.createTempFile("game", ".json")
        try {
            val game = GameFacade(gameFile)
            game.createHero("Гэндальф", "mage")
            Dice.reseed(11)
            val expedition = game.explore("Гэндальф", "forest", "cautious")
            println("Экспедиция через фасад: " + view.summarize(expedition,
                game.getLocation(expedition.locationCode).displayName, game.getTactic(expedition.tacticCode).displayName))
            view.printHero(game.getHero("Гэндальф"))
        } finally { Files.deleteIfExists(gameFile) }
    }
}
```

Вывод программы:

```text
===== Цепочка обязанностей =====
Броня 30%: прошло 21; прошло 21
Броня 30% -> барьер 30: поглощено 21, прошло 0; поглощено 9, прошло 12; прошло 21
Уклонение 50% -> броня 30% -> барьер 30: поглощено 21, прошло 0; поглощено 9, прошло 12; уклонение

===== Стратегия =====
Баланс: поражение, раундов 3, у героя осталось 0/100 здоровья
Натиск: поражение, раундов 2, у героя осталось 0/100 здоровья
Осторожность: поражение, раундов 3, у героя осталось 0/100 здоровья

===== Декоратор =====
Гоблин: удары 14, 9, 13, награда 40 опыта; входящий урон 20 -> 20, здоровье 30/50
Легендарный Гоблин: удары 28, 18, 26, награда 80 опыта; входящий урон 20 -> 20, здоровье 30/50
Неистовый Гоблин: удары 23, 22, 15, награда 60 опыта; входящий урон 20 -> 20, здоровье 30/50
Легендарный Неистовый Гоблин: удары 46, 44, 30, награда 120 опыта; входящий урон 20 -> 20, здоровье 30/50
Бронированный Гоблин: удары 14, 9, 13, награда 40 опыта; входящий урон 20 -> 10, здоровье 40/50
Бронированный Легендарный Неистовый Дракон: удары 66, 82, 78, награда 1500 опыта; входящий урон 20 -> 8, здоровье 252/260

===== Адаптер =====
Предмет: Эльфийский лук (урон 14, шанс крита 40%)
Враг:    Одержимый Эльфийский лук (70 здоровья, награда 45 опыта)
=== Проклятый особняк (сид 55) ===
Леголас встречает врага: Одержимый Железный меч (85 здоровья)
[раунд 1] Леголас -> Одержимый Железный меч: 13 урона, осталось 72/85
[раунд 1] Одержимый Железный меч -> Леголас: 14 урона, осталось 101/115
[раунд 2] Леголас -> Одержимый Железный меч: КРИТ! 23 урона, осталось 49/85
[раунд 2] Одержимый Железный меч -> Леголас: 14 урона, осталось 87/115
[раунд 3] Леголас -> Одержимый Железный меч: 13 урона, осталось 36/85
[раунд 3] Одержимый Железный меч -> Леголас: 14 урона, осталось 73/115
[раунд 4] Леголас -> Одержимый Железный меч: 13 урона, осталось 23/85
[раунд 4] Одержимый Железный меч -> Леголас: 14 урона, осталось 59/115
[раунд 5] Леголас -> Одержимый Железный меч: 13 урона, осталось 10/85
[раунд 5] Одержимый Железный меч -> Леголас: 14 урона, осталось 45/115
[раунд 6] Леголас уходит в оборону
[раунд 6] Леголас -> Одержимый Железный меч: 7 урона, осталось 3/85
[раунд 6] Одержимый Железный меч -> Леголас: 4 урона, осталось 41/115
[раунд 7] Леголас -> Одержимый Железный меч: 13 урона, осталось 0/85
Одержимый Железный меч повержен!
Победа! +40 опыта, +30 золота
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
Гэндальф, Маг 1 ур. | здоровье 85 | опыт 40/100 | золото 69
  Оружие: Посох ученика (урон 10-18)
  Броня:  Льняная роба (защита 10%)
  Достижения: Первая кровь
```

Не забудьте закоммитить и запушить изменения!

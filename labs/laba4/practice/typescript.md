# Лаба 4 — практика на TypeScript

Это практическая часть [лабораторной работы 4](../laba4.md): требования к сдаче и вопросы к защите находятся в основном тексте лабы.

Понадобится Node.js 22 или новее. Шаблон репозитория: [tech-prog-labs-2026-typescript](https://github.com/tidbid-kt/tech-prog-labs-2026-typescript), после клонирования установите компилятор TypeScript командой `npm install`. Код лабы пишите в папке `labs/lab3`. Все команды запускаются из корня репозитория: сборка - `npm run build`, затем игра - `node dist/labs/lab3/main.js`, демонстрация - `node dist/labs/lab3/demo.js`.

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

**`labs/lab3/defense/damage.ts`**

```typescript
export class Damage {
    evaded = false;
    absorbed = 0;
    constructor(public amount: number) {}
    reduceTo(amount: number): void {
        this.amount = amount;
    }
    evade(): void {
        this.amount = 0;
        this.evaded = true;
    }
    absorb(value: number): void {
        this.amount -= value;
        this.absorbed += value;
    }
}
```

И теперь `Combatant.receiveHit` будет возвращать `Damage` вместо `number`. Так мы сможем реализовать отдельные механики защиты от урона.
Получится что-то такое:

**`labs/lab3/combat/combatant.ts`**

```typescript
import {Damage} from "../defense/damage";
import {Hit} from "./hit";

// всё, что может участвовать в бою: и герой, и враги
export interface Combatant {

    // код вида сущности: класс героя или тип врага
    readonly code: string;

    readonly name: string;

    readonly health: number;

    readonly maxHealth: number;

    isAlive(): boolean;

    // удар, который будет нанесен данной сущностью
    rollDamage(): Hit;

    // получение удара, возвращает урон, который прошел на самом деле
    receiveHit(damage: number): Damage;
}
```

Для врагов давайте обработку урона в `receiveHit` также сделаем через `Damage`, не зря же мы этот класс добавляли:

**`labs/lab3/enemy/abstract-enemy.ts`**

```typescript
import {Damage} from "../defense/damage";
import {Hit} from "../combat/hit";
import {Dice} from "../core/dice";
import {Enemy} from "./enemy";

// общая логика для врагов: здоровье, урон в диапазоне, награда
export abstract class AbstractEnemy implements Enemy {

    private currentHealth: number;

    protected constructor(
        readonly code: string,
        readonly name: string,
        readonly maxHealth: number,
        private readonly minDamage: number,
        private readonly maxDamage: number,
        readonly experienceReward: number,
        private readonly goldReward: number,
    ) {
        this.currentHealth = maxHealth;
    }

    get health(): number {
        return this.currentHealth;
    }

    isAlive(): boolean {
        return this.currentHealth > 0;
    }

    rollDamage(): Hit {
        return {damage : Dice.getInstance().roll(this.minDamage, this.maxDamage), critical : false};
    }

    receiveHit(damage: number): Damage {
        const dealt = Math.max(1, this.reduceDamage(damage));
        this.currentHealth = Math.max(0, this.currentHealth - dealt);
        return new Damage(dealt);
    }

    rollGoldReward(): number {
        return Dice.getInstance().roll(Math.floor(this.goldReward / 2), this.goldReward);
    }

    // наследники могут уменьшать входящий урон
    protected reduceDamage(damage: number): number {
        return damage;
    }
}
```

В `Battle` добавим обработку результата `receiveHit`, с учетом того, что он теперь возвращает `Damage`:

**`labs/lab3/combat/battle.ts`**

```typescript
import {BattleEvent} from "./battle-event";
import {BattleReport} from "./battle-report";
import {Combatant} from "./combatant";
import {EventType} from "./event-type";
import {participantOf} from "./participant";
import {Side} from "./side";

// бой идет по раундам: сначала бьет герой, потом враг, пока кто-то не погибнет
export class Battle {

    private readonly events: BattleEvent[] = [];
    private round = 0;

    constructor(private readonly hero: Combatant, private readonly enemy: Combatant) {}

    run(): BattleReport {
        while (this.hero.isAlive() && this.enemy.isAlive()) {
            this.round++;
            this.strike(this.hero, "hero", this.enemy, "enemy");
            if (this.enemy.isAlive()) {
                this.strike(this.enemy, "enemy", this.hero, "hero");
            }
        }

        return {
            hero : participantOf(this.hero),
            enemy : participantOf(this.enemy),
            outcome : this.hero.isAlive() ? "victory" : "defeat",
            events : [...this.events ],
        };
    }

    private strike(attacker: Combatant, source: Side, defender: Combatant, target: Side): void {
        const hit = attacker.rollDamage();
        const dealt = defender.receiveHit(hit.damage).amount;
        const type: EventType = hit.critical ? "critical-hit" : "hit";
        this.emit(
            {round : this.round, type, source, target, amount : dealt, targetHealthAfter : defender.health});

        if (!defender.isAlive()) {
            this.emit(
                {round : this.round, type : "death", source, target, amount : 0, targetHealthAfter : 0});
        }
    }

    private emit(event: BattleEvent): void {
        this.events.push(event);
    }
}
```

А еще у нас в игре появятся достижения. Выдаваться они будут с помощью паттерна "наблюдатель", но объявим мы их сразу здесь.

Представлять собой они будут идентификатор, название и описание. Как-то так:

**`labs/lab3/progress/achievement.ts`**

```typescript
import {UnknownCodeError} from "../core/unknown-code-error";
export class Achievement {
    static readonly FIRST_BLOOD = new Achievement("first-blood", "Первая кровь", "одержать первую победу");
    static readonly LUCKY = new Achievement("lucky", "Счастливчик", "найти зачарованный лут");
    static readonly VETERAN = new Achievement("veteran", "Ветеран", "достичь 5 уровня");
    static readonly DRAGON_SLAYER = new Achievement("dragon-slayer", "Драконоборец", "победить дракона");
    static readonly values =
        [ Achievement.FIRST_BLOOD, Achievement.LUCKY, Achievement.VETERAN, Achievement.DRAGON_SLAYER ];
    private constructor(readonly code: string, readonly displayName: string, readonly description: string) {}
    static byCode(code: string): Achievement {
        const result = this.values.find(a => a.code === code);
        if (!result)
            throw new UnknownCodeError("Достижение", code);
        return result;
    }
}
```

В `Hero` добавим хранение достижений, полученных за этого героя, а также возможность восстановления опыта, инвентаря и достижений через
строитель.

Нам это понадобится, когда будем добавлять механику сохранений!

А еще метод отдыха переименуем в `rest` вместо `restoreHealth`. В нем позже появится дополнительное поведение, поэтому стоит сразу немного
расширить контракт.

Итог выглядит вот так:

**`labs/lab3/hero/hero.ts`**

```typescript
import {Damage} from "../defense/damage";
import {Achievement} from "../progress/achievement";
import {Combatant} from "../combat/combatant";
import {Hit} from "../combat/hit";
import {Armor} from "../item/armor";
import {Item} from "../item/item";
import {Weapon} from "../item/weapon";
import {StarterKitFactory} from "../kit/starter-kit-factory";
import {HeroClass} from "./hero-class";

const MAX_NAME_LENGTH = 20;
const STARTING_GOLD = 50;
const HEALTH_PER_LEVEL = 15;
const DAMAGE_PER_LEVEL = 3;
const EXPERIENCE_PER_LEVEL = 100;

// значения, которые строитель передает в конструктор героя
interface HeroParameters {
    readonly name: string;
    readonly heroClass: HeroClass;
    readonly weapon: Weapon;
    readonly armor: Armor;
    readonly level: number;
    readonly gold: number;
    readonly experience: number;
    readonly inventory: readonly Item[];
    readonly achievements: readonly Achievement[];
}

export class Hero implements Combatant {

    // внутренний класс билдер - чтобы удобно и безопасно создавать героев
    static readonly Builder = class {

        private name: string|undefined;
        private heroClass: HeroClass|undefined;
        private weapon: Weapon|undefined;
        private armor: Armor|undefined;
        // необязательные поля сразу получают значения по умолчанию
        private level = 1;
        private gold = STARTING_GOLD;
        private experience = 0;
        private inventory: readonly Item[] = [];
        private achievements: readonly Achievement[] = [];
        setExperience(experience: number): this {
            this.experience = experience;
            return this;
        }
        setInventory(inventory: readonly Item[]): this {
            this.inventory = [...inventory ];
            return this;
        }
        setAchievements(achievements: readonly Achievement[]): this {
            this.achievements = [...achievements ];
            return this;
        }

        setName(name: string): this {
            this.name = name;
            return this;
        }

        setHeroClass(heroClass: HeroClass): this {
            this.heroClass = heroClass;
            return this;
        }

        setWeapon(weapon: Weapon): this {
            this.weapon = weapon;
            return this;
        }

        setArmor(armor: Armor): this {
            this.armor = armor;
            return this;
        }

        // берет оружие и броню из стартового набора
        setStarterKit(starterKit: StarterKitFactory): this {
            this.weapon = starterKit.createWeapon();
            this.armor = starterKit.createArmor();
            return this;
        }

        setLevel(level: number): this {
            this.level = level;
            return this;
        }

        setGold(gold: number): this {
            this.gold = gold;
            return this;
        }

        // перед созданием героя проверяем, что все обязательные поля заполнены корректно
        build(): Hero {
            if (this.name === undefined || this.name.trim() === "") {
                throw new Error("У героя должно быть имя");
            }
            if (this.name.length > MAX_NAME_LENGTH) {
                throw new Error(`Имя героя не может быть длиннее ${MAX_NAME_LENGTH} символов`);
            }
            if (this.heroClass === undefined) {
                throw new Error("Не выбран класс героя");
            }
            if (this.weapon === undefined || this.armor === undefined) {
                throw new Error("Герою нужны оружие и броня");
            }
            if (this.level < 1) {
                throw new Error("Уровень героя не может быть меньше 1");
            }
            if (this.gold < 0) {
                throw new Error("Золота не может быть меньше нуля");
            }

            if (this.experience < 0)
                throw new Error("Опыт не может быть отрицательным");
            return new Hero({
                name : this.name,
                heroClass : this.heroClass,
                weapon : this.weapon,
                armor : this.armor,
                level : this.level,
                gold : this.gold,
                experience : this.experience,
                inventory : this.inventory,
                achievements : this.achievements,
            });
        }
    };

    readonly name: string;
    readonly heroClass: HeroClass;
    private currentLevel: number;
    private currentExperience = 0;
    private currentGold: number;
    private currentHealth = 0;
    private readonly achievementList: Achievement[];
    private currentWeapon: Weapon;
    private currentArmor: Armor;
    private readonly inventoryList: Item[] = [];

    // конструктор приватный: создать героя можно только через строителя
    private constructor(parameters: HeroParameters) {
        this.name = parameters.name;
        this.heroClass = parameters.heroClass;
        this.currentLevel = parameters.level;
        this.currentGold = parameters.gold;
        this.currentWeapon = parameters.weapon;
        this.currentArmor = parameters.armor;
        this.currentExperience = parameters.experience;
        this.inventoryList.push(...parameters.inventory);
        this.achievementList = [...new Set(parameters.achievements) ];
        this.rest();
    }

    // свойства только для чтения: снаружи их можно прочитать, а поменять может только сам герой

    get code(): string {
        return this.heroClass.code;
    }

    get health(): number {
        return this.currentHealth;
    }

    get maxHealth(): number {
        return this.heroClass.baseHealth + (this.currentLevel - 1) * HEALTH_PER_LEVEL;
    }

    get level(): number {
        return this.currentLevel;
    }

    get experience(): number {
        return this.currentExperience;
    }

    get experienceToNextLevel(): number {
        return this.currentLevel * EXPERIENCE_PER_LEVEL;
    }

    get gold(): number {
        return this.currentGold;
    }

    get weapon(): Weapon {
        return this.currentWeapon;
    }

    get armor(): Armor {
        return this.currentArmor;
    }

    get inventory(): readonly Item[] {
        return [...this.inventoryList ];
    }

    rollDamage(): Hit {
        const hit = this.currentWeapon.rollDamage();
        return {damage : hit.damage + (this.currentLevel - 1) * DAMAGE_PER_LEVEL, critical : hit.critical};
    }

    receiveHit(damage: number): Damage {
        const reducedDamage = Math.round(damage * (1 - this.currentArmor.defense));
        // хотя бы единицу урона броня пропускает всегда
        const dealt = Math.max(1, reducedDamage);
        this.currentHealth = Math.max(0, this.currentHealth - dealt);
        return new Damage(dealt);
    }

    isAlive(): boolean {
        return this.currentHealth > 0;
    }

    // поведение героя между экспедициями

    // перед каждой экспедицией герой успевает отдохнуть
    rest(): void {
        this.currentHealth = this.maxHealth;
    }

    // начисляет опыт, возвращает количество полученных уровней
    gainExperience(amount: number): number {
        this.currentExperience += amount;
        let levelsGained = 0;
        while (this.currentExperience >= this.experienceToNextLevel) {
            this.currentExperience -= this.experienceToNextLevel;
            this.currentLevel++;
            levelsGained++;
        }

        return levelsGained;
    }

    addGold(amount: number): void {
        this.currentGold += amount;
    }

    spendGold(amount: number): void {
        if (amount > this.currentGold) {
            throw new Error(`Не хватает золота: нужно ${amount}, есть ${this.currentGold}`);
        }

        this.currentGold -= amount;
    }

    addItem(item: Item): void {
        this.inventoryList.push(item);
    }

    removeItem(item: Item): void {
        const index = this.inventoryList.indexOf(item);
        if (index < 0) {
            throw new Error(`В инвентаре нет предмета ${item.name}`);
        }

        this.inventoryList.splice(index, 1);
    }

    // экипирует предмет из инвентаря, снятый предмет возвращается в инвентарь
    equip(item: Item): void {
        if (item instanceof Weapon) {
            this.removeItem(item);
            this.inventoryList.push(this.currentWeapon);
            this.currentWeapon = item;
        } else if (item instanceof Armor) {
            this.removeItem(item);
            this.inventoryList.push(this.currentArmor);
            this.currentArmor = item;
        } else {
            throw new Error(`Предмет нельзя экипировать: ${item.name}`);
        }
    }
    get achievements(): readonly Achievement[] {
        return [...this.achievementList ];
    }
    unlock(achievement: Achievement): boolean {
        if (this.achievementList.includes(achievement))
            return false;
        this.achievementList.push(achievement);
        return true;
    }
}
```

В `Location` заменяем вызов `restoreHealth()` на `rest()` и открываем подклассам доступ к каталогу через поле `catalog`,
это нам скоро понадобится:

**`labs/lab3/world/location.ts`**

```typescript
import {Battle} from "../combat/battle";
import {Dice} from "../core/dice";
import {Enemy} from "../enemy/enemy";
import {Hero} from "../hero/hero";
import {Item} from "../item/item";
import {ItemCatalog} from "../item/item-catalog";
import {ExpeditionReport} from "./expedition-report";
import {Rewards} from "./rewards";

const LOOT_CHANCE = 0.6;
const ENCHANT_CHANCE = 0.3;

export abstract class Location {

    protected constructor(
        readonly code: string,
        readonly displayName: string,
        readonly recommendedLevel: number,
        protected readonly catalog: ItemCatalog,
    ) {}

    // коды предметов из каталога, которые можно найти в локации
    protected abstract get lootTable(): readonly string[];

    // создание врага делегируется подклассам для повышения гибкости
    protected abstract createEnemy(): Enemy;

    // бизнес-логика, общая для всех локаций:
    // герой встречает врага, сражается с ним и получает награду, если победил;
    // если сид не передан, берется случайный
    explore(hero: Hero, seed: number = Dice.getInstance().nextSeed()): ExpeditionReport {
        const dice = Dice.getInstance();
        dice.reseed(seed);

        hero.rest();
        const enemy = this.createEnemy();
        const battle = new Battle(hero, enemy).run();

        const rewards = battle.outcome === "victory" ? this.reward(hero, enemy, dice) : this.punish(hero);

        return {seed, locationCode : this.code, battle, rewards};
    }

    private reward(hero: Hero, enemy: Enemy, dice: Dice): Rewards {
        const experience = enemy.experienceReward;
        const gold = enemy.rollGoldReward();
        const loot = this.rollLoot(dice);

        const levelsGained = hero.gainExperience(experience);
        hero.addGold(gold);
        if (loot !== null) {
            hero.addItem(loot);
        }

        return {experience, gold, loot, levelsGained};
    }

    private rollLoot(dice: Dice): Item|null {
        if (!dice.chance(LOOT_CHANCE)) {
            return null;
        }

        // добыча - это копия шаблона из каталога, ее можно менять, не боясь испортить шаблон
        const loot = this.catalog.create(dice.pick(this.lootTable));
        if (dice.chance(ENCHANT_CHANCE)) {
            loot.enchant(dice.pick(loot.possibleEnchantments));
        }

        return loot;
    }

    // проигравший герой теряет половину золота
    private punish(hero: Hero): Rewards {
        const lostGold = Math.floor(hero.gold / 2);
        hero.spendGold(lostGold);
        return {experience : 0, gold : -lostGold, loot : null, levelsGained : 0};
    }
}
```

Чтобы иметь возможность восстановить зачарование по идентификатору (нам это понадобится дальше), добавим в него поиск по этому самому
идентификатору:

**`labs/lab3/item/enchantment.ts`**

```typescript
import {UnknownCodeError} from "../core/unknown-code-error";
// зачарование неизменяемо: поля readonly нельзя поменять в коде, а Object.freeze запрещает менять объект
// и во время работы программы, поэтому одно и то же зачарование можно безопасно повесить на несколько
// предметов
export interface Enchantment {
    readonly code: string;
    readonly name: string;
    readonly bonusDamage: number;
    readonly bonusDefense: number;
    readonly bonusPrice: number;
}

export const SHARP: Enchantment =
    Object.freeze({code : "sharp", name : "Острое", bonusDamage : 3, bonusDefense : 0, bonusPrice : 20});
export const FIERY: Enchantment =
    Object.freeze({code : "fiery", name : "Огненное", bonusDamage : 6, bonusDefense : 0, bonusPrice : 50});
export const STURDY: Enchantment =
    Object.freeze({code : "sturdy", name : "Прочное", bonusDamage : 0, bonusDefense : 0.05, bonusPrice : 20});
export const WARDING: Enchantment = Object.freeze(
    {code : "warding", name : "Охранное", bonusDamage : 0, bonusDefense : 0.1, bonusPrice : 50});

export const FOR_WEAPONS: readonly Enchantment[] = [ SHARP, FIERY ];
export const FOR_ARMOR: readonly Enchantment[] = [ STURDY, WARDING ];

export function enchantmentByCode(code: string): Enchantment {
    const result = [...FOR_WEAPONS, ...FOR_ARMOR ].find(e => e.code === code);
    if (!result)
        throw new UnknownCodeError("Зачарование", code);
    return result;
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

**`labs/lab3/enemy/haunted-weapon.ts`**

```typescript
import {Dice} from "../core/dice";
import {Enemy} from "./enemy";
import {Weapon} from "../item/weapon";
import {Damage} from "../defense/damage";
export class HauntedWeapon implements Enemy {
    readonly code = "haunted-weapon";
    readonly maxHealth: number;
    private currentHealth: number;
    constructor(private readonly weapon: Weapon) {
        this.maxHealth = weapon.damage * 5;
        this.currentHealth = this.maxHealth;
    }
    get name() {
        return "Одержимый " + this.weapon.name;
    }
    get health() {
        return this.currentHealth;
    }
    isAlive() {
        return this.health > 0;
    }
    rollDamage() {
        return this.weapon.rollDamage();
    }
    get experienceReward() {
        return Math.floor(this.weapon.price / 2);
    }
    rollGoldReward() {
        return Dice.getInstance().roll(Math.floor(this.weapon.price / 4), Math.floor(this.weapon.price / 2));
    }
    receiveHit(amount: number): Damage {
        const damage = new Damage(amount);
        this.currentHealth = Math.max(0, this.health - damage.amount);
        return damage;
    }
}
```

А теперь объявим и саму локацию — проклятый особняк. В ней будет одно из трех случайных оружий, которые будут взяты из каталога.
Выбранное оружие будет обернуто в адаптер, то есть станет одержимо злым духом, после чего безжалостно нападет на нашего героя.

Благодаря адаптеру мы можем превратить во врага любое оружие, что безусловно разнообразит визиты в это замечательное жилище:

**`labs/lab3/world/haunted-manor.ts`**

```typescript
import {Dice} from "../core/dice";
import {HauntedWeapon} from "../enemy/haunted-weapon";
import {IRON_SWORD, ELVEN_BOW, EMBER_STAFF} from "../item/default-catalog";
import {ItemCatalog} from "../item/item-catalog";
import {Location} from "./location";
export class HauntedManor extends Location {
    static readonly CODE = "haunted-manor";
    constructor(catalog: ItemCatalog) {
        super(HauntedManor.CODE, "Проклятый особняк", 2, catalog);
    }
    protected override get lootTable() {
        return [ IRON_SWORD, ELVEN_BOW, EMBER_STAFF ];
    }
    protected override createEnemy() {
        return new HauntedWeapon(this.catalog.createWeapon(Dice.getInstance().pick(this.lootTable)));
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
реализации паттерна. В нем через композицию определяется, кого заворачивает декоратор (поле `wrapped`), все методы базового декоратора
делегируют
вызовы объекту `wrapped`.

Важно, что `wrapped` это абстракция – `Enemy` и сам базовый декоратор тоже реализует эту абстракцию. Таким образом на месте `wrapped` может
быть и другой декоратор.
Но в конце цепочки обязательно будет конкретный враг.

Итак, код получается вот такой:

**`labs/lab3/enemy/enemy-decorator.ts`**

```typescript
import {Enemy} from "./enemy";
import {Damage} from "../defense/damage";
export abstract class EnemyDecorator implements Enemy {
    constructor(protected readonly wrapped: Enemy) {}
    get code() {
        return this.wrapped.code;
    }
    get name() {
        return this.wrapped.name;
    }
    get health() {
        return this.wrapped.health;
    }
    get maxHealth() {
        return this.wrapped.maxHealth;
    }
    get experienceReward() {
        return this.wrapped.experienceReward;
    }
    isAlive() {
        return this.wrapped.isAlive();
    }
    rollDamage() {
        return this.wrapped.rollDamage();
    }
    receiveHit(amount: number): Damage {
        return this.wrapped.receiveHit(amount);
    }
    rollGoldReward() {
        return this.wrapped.rollGoldReward();
    }
}
```

А дальше уже реализуем конкретных декораторов, наследников базового. Они будут соответствовать нашим модификатором, начнем с легендарного:

**`labs/lab3/enemy/legendary-enemy.ts`**

```typescript
import {EnemyDecorator} from "./enemy-decorator";
export class LegendaryEnemy extends EnemyDecorator {
    override get name() {
        return "Легендарный " + super.name;
    }
    override rollDamage() {
        const hit = super.rollDamage();
        return {damage : hit.damage * 2, critical : hit.critical};
    }
    override get experienceReward() {
        return super.experienceReward * 2;
    }
    override rollGoldReward() {
        return super.rollGoldReward() * 2;
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

**`labs/lab3/enemy/windfury-enemy.ts`**

```typescript
import {EnemyDecorator} from "./enemy-decorator";
export class WindfuryEnemy extends EnemyDecorator {
    override get name() {
        return "Неистовый " + super.name;
    }
    override rollDamage() {
        const first = super.rollDamage(), second = super.rollDamage();
        return {damage : first.damage + second.damage, critical : first.critical || second.critical};
    }
    override get experienceReward() {
        return this.withBonus(super.experienceReward);
    }
    override rollGoldReward() {
        return this.withBonus(super.rollGoldReward());
    }
    private withBonus(reward: number) {
        return reward + Math.floor(reward * 50 / 100);
    }
}
```

Два броска здесь складываются в один удар, поэтому защита применяется к их сумме.

Логика тут точно такая же! Модифицируем поведение, делегируем в `super`.

Третий и пока финальный декоратор уменьшает входящий урон вдвое, затем передаёт получившийся урон дальше по цепочке декораторов:

**`labs/lab3/enemy/armored-enemy.ts`**

```typescript
import {Damage} from "../defense/damage";
import {EnemyDecorator} from "./enemy-decorator";
export class ArmoredEnemy extends EnemyDecorator {
    override get name() {
        return "Бронированный " + super.name;
    }
    override receiveHit(amount: number): Damage {
        if (amount < 0)
            throw new Error("Урон не может быть отрицательным");
        return amount === 0 ? new Damage(0) : super.receiveHit(amount - Math.floor(amount / 2));
    }
}
```

Отлично, все требуемые модификаторы объявлены.
Теперь мы можем добавлять их любому врагу и в любой последовательности.
При этом декоратор реализует тот же интерфейс `Enemy`, что и обычный враг, поэтому ни бой, ни локация не отличают элитного врага
от обычного, и работают с ним точно так же. Прелести `Dependency Inversion Principle`!

Осталось прописать логику, как и когда враг становится элитным. Это часть общей логики экспедиции, поэтому добавим её в локацию:
враг, созданный фабричным методом, с некоторым шансом получает комбинацию модификаторов.

**`labs/lab3/world/location.ts`**

```typescript
import {ArmoredEnemy} from "../enemy/armored-enemy";
import {LegendaryEnemy} from "../enemy/legendary-enemy";
import {WindfuryEnemy} from "../enemy/windfury-enemy";
import {Battle} from "../combat/battle";
import {Dice} from "../core/dice";
import {Enemy} from "../enemy/enemy";
import {Hero} from "../hero/hero";
import {Item} from "../item/item";
import {ItemCatalog} from "../item/item-catalog";
import {ExpeditionReport} from "./expedition-report";
import {Rewards} from "./rewards";

const LOOT_CHANCE = 0.6;
const ENCHANT_CHANCE = 0.3;

export abstract class Location {

    protected constructor(
        readonly code: string,
        readonly displayName: string,
        readonly recommendedLevel: number,
        protected readonly catalog: ItemCatalog,
    ) {}

    // коды предметов из каталога, которые можно найти в локации
    protected abstract get lootTable(): readonly string[];

    // создание врага делегируется подклассам для повышения гибкости
    protected abstract createEnemy(): Enemy;

    // бизнес-логика, общая для всех локаций:
    // герой встречает врага, сражается с ним и получает награду, если победил;
    // если сид не передан, берется случайный
    explore(hero: Hero, seed: number = Dice.getInstance().nextSeed()): ExpeditionReport {
        const dice = Dice.getInstance();
        dice.reseed(seed);

        hero.rest();
        let enemy = this.createEnemy();
        if (dice.chance(0.2))
            enemy = this.makeElite(enemy, dice);
        const battle = new Battle(hero, enemy).run();

        const rewards = battle.outcome === "victory" ? this.reward(hero, enemy, dice) : this.punish(hero);

        return {seed, locationCode : this.code, battle, rewards};
    }

    private reward(hero: Hero, enemy: Enemy, dice: Dice): Rewards {
        const experience = enemy.experienceReward;
        const gold = enemy.rollGoldReward();
        const loot = this.rollLoot(dice);

        const levelsGained = hero.gainExperience(experience);
        hero.addGold(gold);
        if (loot !== null) {
            hero.addItem(loot);
        }

        return {experience, gold, loot, levelsGained};
    }

    private rollLoot(dice: Dice): Item|null {
        if (!dice.chance(LOOT_CHANCE)) {
            return null;
        }

        // добыча - это копия шаблона из каталога, ее можно менять, не боясь испортить шаблон
        const loot = this.catalog.create(dice.pick(this.lootTable));
        if (dice.chance(ENCHANT_CHANCE)) {
            loot.enchant(dice.pick(loot.possibleEnchantments));
        }

        return loot;
    }

    // проигравший герой теряет половину золота
    private punish(hero: Hero): Rewards {
        const lostGold = Math.floor(hero.gold / 2);
        hero.spendGold(lostGold);
        return {experience : 0, gold : -lostGold, loot : null, levelsGained : 0};
    }
    private makeElite(enemy: Enemy, dice: Dice): Enemy {
        const both = dice.chance(0.25);
        if (dice.chance(0.5)) {
            enemy = new LegendaryEnemy(enemy);
            if (both)
                enemy = new WindfuryEnemy(enemy);
        } else {
            enemy = new WindfuryEnemy(enemy);
            if (both)
                enemy = new LegendaryEnemy(enemy);
        }
        return dice.chance(0.3) ? new ArmoredEnemy(enemy) : enemy;
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

**`labs/lab3/storage/hero-repository.ts`**

```typescript
import {Hero} from "../hero/hero";
export interface HeroRepository {
    findByName(name: string): Hero|undefined;
    findAll(): Hero[];
    save(hero: Hero): void;
}
```

Для хранения сейвов будем использовать простой файл, это не бд, но подход тоже вполне рабочий. В качестве формата сохранения выберем
json! [JSON](https://www.json.org/json-ru.html) — текстовый формат, где объекты состоят из пар "имя — значение",
а массивы хранят списки значений. Это довольно вездесущий формат, и нам он как нельзя кстати подходит.

Для чтения и записи JSON используем встроенные `JSON.stringify` и `JSON.parse`, дополнительные зависимости не нужны.

В файле сохранения будет массив героев со всеми нужными для восстановления объектов данными.
Интерфейсы `HeroSave` и `ItemSave` описывают данные сохранения. Из JSON получаются обычные объекты с данными; героя восстанавливаем через строителя, а предметы — через каталог.

Реализуем наконец сам репозиторий:

**`labs/lab3/storage/file-hero-repository.ts`**

```typescript
import {existsSync, readFileSync, writeFileSync, mkdirSync} from "node:fs";
import {dirname, resolve} from "node:path";
import {Hero} from "../hero/hero";
import {HeroClass} from "../hero/hero-class";
import {Item} from "../item/item";
import {Armor} from "../item/armor";
import {Weapon} from "../item/weapon";
import {ItemCatalog} from "../item/item-catalog";
import {enchantmentByCode} from "../item/enchantment";
import {Achievement} from "../progress/achievement";
import {HeroRepository} from "./hero-repository";
interface ItemSave {
    code: string;
    enchantments: string[];
}
interface HeroSave {
    name: string;
    heroClass: string;
    level: number;
    experience: number;
    gold: number;
    weapon: ItemSave;
    armor: ItemSave;
    inventory: ItemSave[];
    achievements: string[];
}
export class FileHeroRepository implements HeroRepository {
    constructor(private readonly file: string, private readonly catalog: ItemCatalog) {}
    findByName(name: string) {
        return this.findAll().find(hero => hero.name === name);
    }
    findAll(): Hero[] {
        if (!existsSync(this.file))
            return [];
        const text = readFileSync(this.file, "utf8");
        if (!text.trim())
            return [];
        const saves: HeroSave[] = JSON.parse(text);
        if (!Array.isArray(saves))
            throw new Error("В файле должен быть массив героев");
        return saves.map(save => this.restore(save));
    }
    save(hero: Hero): void {
        const saves =
            this.findAll().filter(other => other.name !== hero.name).map(other => this.snapshot(other));
        saves.push(this.snapshot(hero));
        mkdirSync(dirname(resolve(this.file)), {recursive : true});
        writeFileSync(this.file, JSON.stringify(saves, null, 2) + "\n", "utf8");
    }
    private snapshotItem(item: Item): ItemSave {
        return {code : item.code, enchantments : item.enchantments.map(e => e.code)};
    }
    private snapshot(hero: Hero): HeroSave {
        return {
            name : hero.name,
            heroClass : hero.heroClass.code,
            level : hero.level,
            experience : hero.experience,
            gold : hero.gold,
            weapon : this.snapshotItem(hero.weapon),
            armor : this.snapshotItem(hero.armor),
            inventory : hero.inventory.map(item => this.snapshotItem(item)),
            achievements : hero.achievements.map(a => a.code)
        };
    }
    private restoreItem(save: ItemSave): Item {
        const item = this.catalog.create(save.code);
        for (const code of save.enchantments)
            item.enchant(enchantmentByCode(code));
        return item;
    }
    private restore(save: HeroSave): Hero {
        const weapon = this.restoreItem(save.weapon), armor = this.restoreItem(save.armor);
        if (!(weapon instanceof Weapon) || !(armor instanceof Armor))
            throw new Error("Некорректное снаряжение героя");
        return new Hero.Builder()
            .setName(save.name)
            .setHeroClass(HeroClass.byCode(save.heroClass))
            .setLevel(save.level)
            .setExperience(save.experience)
            .setGold(save.gold)
            .setWeapon(weapon)
            .setArmor(armor)
            .setInventory(save.inventory.map(item => this.restoreItem(item)))
            .setAchievements(save.achievements.map(code => Achievement.byCode(code)))
            .build();
    }
}
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

**`labs/lab3/storage/caching-hero-repository.ts`**

```typescript
import {Hero} from "../hero/hero";
import {HeroRepository} from "./hero-repository";
export class CachingHeroRepository implements HeroRepository {
    private cache: Map<string, Hero>|undefined;
    constructor(private readonly target: HeroRepository) {}
    private cached(): Map<string, Hero> {
        if (!this.cache)
            this.cache = new Map(this.target.findAll().map(hero => [hero.name, hero]));
        return this.cache;
    }
    findByName(name: string) {
        return this.cached().get(name);
    }
    findAll() {
        return [...this.cached().values() ];
    }
    save(hero: Hero): void {
        this.target.save(hero);
        this.cached().set(hero.name, hero);
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

**`labs/lab3/tactic/action.ts`**

```typescript
export class Action {
    static readonly ATTACK = new Action(1.0, 1.0);
    static readonly POWER_STRIKE = new Action(1.5, 1.5);
    static readonly DEFEND = new Action(0.5, 0.3);
    private constructor(readonly attackMultiplier: number, readonly incomingMultiplier: number) {}
}
```

Получается, что он может вести себя агрессивно (увеличивается и исходящий, и входящий урон), сбалансированно (ничего не меняется) и
осторожно, уходя в оборону (уменьшается и входящий, и исходящий урон).

Выбирать действия мы будем в зависимости от состояния врага и героя, определим базовый интерфейс тактики:

**`labs/lab3/tactic/tactic.ts`**

```typescript
import {Combatant} from "../combat/combatant";
import {Action} from "./action";
export interface Tactic {
    readonly code: string;
    readonly displayName: string;
    readonly description: string;
    chooseAction(round: number, hero: Combatant, enemy: Combatant): Action;
}
```

Реализуем самый простой алгоритм выбора действия - обычный удар каждый раунд:

**`labs/lab3/tactic/balanced-tactic.ts`**

```typescript
import {Combatant} from "../combat/combatant";
import {Tactic} from "./tactic";
import {Action} from "./action";
export class BalancedTactic implements Tactic {
    readonly code = "balanced";
    readonly displayName = "Баланс";
    readonly description = "обычный удар каждый раунд";
    chooseAction(round: number, hero: Combatant, enemy: Combatant): Action {
        return Action.ATTACK;
    }
}
```

Натиск - пока здоровье врага выше, чем у героя, герой бьет безрассудно, нанося и получая в ответ больше урона:

**`labs/lab3/tactic/aggressive-tactic.ts`**

```typescript
import {Combatant} from "../combat/combatant";
import {Tactic} from "./tactic";
import {Action} from "./action";
export class AggressiveTactic implements Tactic {
    readonly code = "aggressive";
    readonly displayName = "Натиск";
    readonly description = "совершать безрассудные удары, пока у врага больше здоровья, чем у героя";
    chooseAction(round: number, hero: Combatant, enemy: Combatant): Action {
        return enemy.health > hero.health ? Action.POWER_STRIKE : Action.ATTACK;
    }
}
```

И осторожность - раненый герой через раунд будет уходить в оборону:

**`labs/lab3/tactic/cautious-tactic.ts`**

```typescript
import {Combatant} from "../combat/combatant";
import {Tactic} from "./tactic";
import {Action} from "./action";
export class CautiousTactic implements Tactic {
    readonly code = "cautious";
    readonly displayName = "Осторожность";
    readonly description = "при здоровье ниже 40% через раунд уходить в оборону";
    chooseAction(round: number, hero: Combatant, enemy: Combatant): Action {
        return hero.health < hero.maxHealth * 0.4 && round % 2 === 0 ? Action.DEFEND : Action.ATTACK;
    }
}
```

Также дополним типы ивентов в нашей игре, добавим наши действия, и `evade`/`absorb`, они нам пригодятся дальше:

**`labs/lab3/combat/event-type.ts`**

```typescript
export type EventType = "hit"|"critical-hit"|"death"|"power-strike"|"defend"|"evade"|"absorb";
```

Теперь обновим сам `Battle`. Тактику он получает через конструктор и каждый раунд спрашивает у нее, что именно делает герой:

**`labs/lab3/combat/battle.ts`**

```typescript
import {Action} from "../tactic/action";
import {Tactic} from "../tactic/tactic";
import {BalancedTactic} from "../tactic/balanced-tactic";
import {BattleEvent} from "./battle-event";
import {BattleReport} from "./battle-report";
import {Combatant} from "./combatant";
import {EventType} from "./event-type";
import {participantOf} from "./participant";
import {Side} from "./side";

// бой идет по раундам: сначала бьет герой, потом враг, пока кто-то не погибнет
export class Battle {

    private readonly events: BattleEvent[] = [];
    private round = 0;

    constructor(private readonly hero: Combatant, private readonly enemy: Combatant,
                private readonly tactic: Tactic = new BalancedTactic()) {}

    run(): BattleReport {
        while (this.hero.isAlive() && this.enemy.isAlive()) {
            this.round++;
            const action = this.tactic.chooseAction(this.round, this.hero, this.enemy);
            if (action !== Action.ATTACK)
                this.emit({
                    round : this.round,
                    type : action === Action.POWER_STRIKE ? "power-strike" : "defend",
                    source : "hero",
                    target : action === Action.POWER_STRIKE ? "enemy" : "hero",
                    amount : 0,
                    targetHealthAfter : 0
                });
            this.strike(this.hero, "hero", this.enemy, "enemy", action.attackMultiplier);
            if (this.enemy.isAlive()) {
                this.strike(this.enemy, "enemy", this.hero, "hero", action.incomingMultiplier);
            }
        }

        return {
            hero : participantOf(this.hero),
            enemy : participantOf(this.enemy),
            outcome : this.hero.isAlive() ? "victory" : "defeat",
            events : [...this.events ],
        };
    }

    private strike(attacker: Combatant, source: Side, defender: Combatant, target: Side,
                   multiplier: number): void {
        const hit = attacker.rollDamage();
        const damage = defender.receiveHit(Math.max(1, Math.round(hit.damage * multiplier)));
        if (damage.evaded) {
            this.emit({
                round : this.round,
                type : "evade",
                source,
                target,
                amount : 0,
                targetHealthAfter : defender.health
            });
            return;
        }
        if (damage.absorbed > 0)
            this.emit({
                round : this.round,
                type : "absorb",
                source,
                target,
                amount : damage.absorbed,
                targetHealthAfter : defender.health
            });
        if (damage.amount > 0)
            this.emit({
                round : this.round,
                type : hit.critical ? "critical-hit" : "hit",
                source,
                target,
                amount : damage.amount,
                targetHealthAfter : defender.health
            });

        if (!defender.isAlive()) {
            this.emit(
                {round : this.round, type : "death", source, target, amount : 0, targetHealthAfter : 0});
        }
    }

    private emit(event: BattleEvent): void {
        this.events.push(event);
    }
}
```

Обратите внимание, что бой ничего не знает о конкретных тактиках: он работает с интерфейсом `Tactic`, и
подставить ему можно любую реализацию. Если появится новая тактика, `Battle` менять не придется.

Ну и в отчете об экспедиции будем запоминать, с какой тактикой герой ее начинал:

**`labs/lab3/world/expedition-report.ts`**

```typescript
import {BattleReport} from "../combat/battle-report";
import {Rewards} from "./rewards";

// отчет об экспедиции: с каким сидом, куда, как прошел бой и что получил герой
export interface ExpeditionReport {
    readonly seed: number;
    readonly locationCode: string;
    readonly tacticCode: string;
    readonly battle: BattleReport;
    readonly rewards: Rewards;
}
```

Стратегия позволяет нам соблюдать `Open Closed` принцип, он дает нам возможность расширять поведение нашего класса без его изменения, путем
добавления новых реализаций стратегии. Стратегию можно выбирать как в методе, где она используется, так и в конструкторе, все зависит от
конкретной необходимости. В нашем случае стратегия выбирается в конструкторе, потому что по ходу боя она не меняется.

Применять стратегию стоит, когда одно и то же действие можно выполнить несколькими способами и способ нужно выбирать
или менять во время работы программы.

И давайте сразу обновим `Location`: она будет принимать тактику и передавать дальше в `Battle`.

Получится что-то такое:

**`labs/lab3/world/location.ts`**

```typescript
import {ArmoredEnemy} from "../enemy/armored-enemy";
import {LegendaryEnemy} from "../enemy/legendary-enemy";
import {WindfuryEnemy} from "../enemy/windfury-enemy";
import {Tactic} from "../tactic/tactic";
import {BalancedTactic} from "../tactic/balanced-tactic";
import {Battle} from "../combat/battle";
import {Dice} from "../core/dice";
import {Enemy} from "../enemy/enemy";
import {Hero} from "../hero/hero";
import {Item} from "../item/item";
import {ItemCatalog} from "../item/item-catalog";
import {ExpeditionReport} from "./expedition-report";
import {Rewards} from "./rewards";

const LOOT_CHANCE = 0.6;
const ENCHANT_CHANCE = 0.3;

export abstract class Location {

    protected constructor(
        readonly code: string,
        readonly displayName: string,
        readonly recommendedLevel: number,
        protected readonly catalog: ItemCatalog,
    ) {}

    // коды предметов из каталога, которые можно найти в локации
    protected abstract get lootTable(): readonly string[];

    // создание врага делегируется подклассам для повышения гибкости
    protected abstract createEnemy(): Enemy;

    // бизнес-логика, общая для всех локаций:
    // герой встречает врага, сражается с ним и получает награду, если победил;
    // если сид не передан, берется случайный
    explore(hero: Hero, seed: number = Dice.getInstance().nextSeed(),
            tactic: Tactic = new BalancedTactic()): ExpeditionReport {
        const dice = Dice.getInstance();
        dice.reseed(seed);

        hero.rest();
        let enemy = this.createEnemy();
        if (dice.chance(0.2))
            enemy = this.makeElite(enemy, dice);
        const battle = new Battle(hero, enemy, tactic).run();

        const rewards = battle.outcome === "victory" ? this.reward(hero, enemy, dice) : this.punish(hero);

        return {seed, locationCode : this.code, tacticCode : tactic.code, battle, rewards};
    }

    private reward(hero: Hero, enemy: Enemy, dice: Dice): Rewards {
        const experience = enemy.experienceReward;
        const gold = enemy.rollGoldReward();
        const loot = this.rollLoot(dice);

        const levelsGained = hero.gainExperience(experience);
        hero.addGold(gold);
        if (loot !== null) {
            hero.addItem(loot);
        }

        return {experience, gold, loot, levelsGained};
    }

    private rollLoot(dice: Dice): Item|null {
        if (!dice.chance(LOOT_CHANCE)) {
            return null;
        }

        // добыча - это копия шаблона из каталога, ее можно менять, не боясь испортить шаблон
        const loot = this.catalog.create(dice.pick(this.lootTable));
        if (dice.chance(ENCHANT_CHANCE)) {
            loot.enchant(dice.pick(loot.possibleEnchantments));
        }

        return loot;
    }

    // проигравший герой теряет половину золота
    private punish(hero: Hero): Rewards {
        const lostGold = Math.floor(hero.gold / 2);
        hero.spendGold(lostGold);
        return {experience : 0, gold : -lostGold, loot : null, levelsGained : 0};
    }
    private makeElite(enemy: Enemy, dice: Dice): Enemy {
        const both = dice.chance(0.25);
        if (dice.chance(0.5)) {
            enemy = new LegendaryEnemy(enemy);
            if (both)
                enemy = new WindfuryEnemy(enemy);
        } else {
            enemy = new WindfuryEnemy(enemy);
            if (both)
                enemy = new LegendaryEnemy(enemy);
        }
        return dice.chance(0.3) ? new ArmoredEnemy(enemy) : enemy;
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

**`labs/lab3/event/game-event-type.ts`**

```typescript
export type GameEventType = "victory"|"defeat"|"level-up";
```

И само событие. Обратите внимание, что это просто данные: что произошло, с каким героем и в какой экспедиции:

**`labs/lab3/event/game-event.ts`**

```typescript
import {Hero} from "../hero/hero";
import {ExpeditionReport} from "../world/expedition-report";
import {GameEventType} from "./game-event-type";
export interface GameEvent {
    readonly type: GameEventType;
    readonly hero: Hero;
    readonly expedition: ExpeditionReport;
}
```

Интерфейс для подписчиков содержит всего один метод, который вызывается при наступлении события, и определяет контракт обработки этого
изменения:

**`labs/lab3/event/game-event-listener.ts`**

```typescript
import {GameEvent} from "./game-event";
export interface GameEventListener {
    onEvent(event: GameEvent): void;
}
```

Интерфейсы TypeScript описывают форму объектов. Издателю подходит любой объект с методом `onEvent`, независимо от его класса.

Именно издатель позволяет подписываться на определенный тип события. Также именно он оповещает подписчиков о
наступлении того или иного события:

**`labs/lab3/event/game-event-publisher.ts`**

```typescript
import {GameEvent} from "./game-event";
import {GameEventType} from "./game-event-type";
import {GameEventListener} from "./game-event-listener";
export class GameEventPublisher {
    private readonly listeners = new Map<GameEventType, GameEventListener[]>();
    subscribe(type: GameEventType, listener: GameEventListener): void {
        const list = this.listeners.get(type) ?? [];
        list.push(listener);
        this.listeners.set(type, list);
    }
    unsubscribe(type: GameEventType, listener: GameEventListener): void {
        const list = this.listeners.get(type);
        const index = list?.indexOf(listener) ?? -1;
        if (list && index >= 0)
            list.splice(index, 1);
    }
    publish(event: GameEvent): void {
        for (const listener of [...this.listeners.get(event.type) ?? []])
            listener.onEvent(event);
    }
}
```

Подписчики хранятся в `listeners`, причем хранятся по типу ивента, то есть подписаться и реагировать они могут на отдельные типы, а не на
все.
Реакции вызываются в методе `publish`. Когда в нашей системе происходит ивент – мы вызываем издателя, а он, по списку зарегистрированных
подписчиков, вызывает их
`onEvent` методы, прокидывая туда информацию о ивенте.

Теперь реализуем двух конкретных подписчиков. Сначала того, который будет выдавать достижения:

**`labs/lab3/progress/achievement-tracker.ts`**

```typescript
import {GameEvent} from "../event/game-event";
import {GameEventListener} from "../event/game-event-listener";
import {Achievement} from "./achievement";
export class AchievementTracker implements GameEventListener {
    onEvent(event: GameEvent): void {
        if (event.type === "victory") {
            event.hero.unlock(Achievement.FIRST_BLOOD);
            if (event.expedition.battle.enemy.code === "dragon")
                event.hero.unlock(Achievement.DRAGON_SLAYER);
            if (event.expedition.rewards.loot?.enchantments.length)
                event.hero.unlock(Achievement.LUCKY);
        } else if (event.type === "level-up" && event.hero.level >= 5)
            event.hero.unlock(Achievement.VETERAN);
    }
}
```

А теперь второго, он будет записывать экспедиции в журнал:

**`labs/lab3/progress/expedition-journal.ts`**

```typescript
import {GameEvent} from "../event/game-event";
import {GameEventListener} from "../event/game-event-listener";
import {ExpeditionReport} from "../world/expedition-report";
export class ExpeditionJournal implements GameEventListener {
    private readonly entries = new Map<string, ExpeditionReport[]>();
    onEvent(event: GameEvent): void {
        const list = this.entries.get(event.hero.name) ?? [];
        list.push(event.expedition);
        this.entries.set(event.hero.name, list);
    }
    getEntries(name: string): readonly ExpeditionReport[] {
        return [...this.entries.get(name) ?? [] ];
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

**`labs/lab3/defense/damage-handler.ts`**

```typescript
import {Damage} from "./damage";
export abstract class DamageHandler {
    private next: DamageHandler|undefined;
    setNext(next: DamageHandler): DamageHandler {
        this.next = next;
        return next;
    }
    abstract handle(damage: Damage): void;
    protected passToNext(damage: Damage): void {
        this.next?.handle(damage);
    }
}
```

Теперь конкретный обработчик, который будет хендлить уклонение: если оно сработало, удар по герою не прошел, и передавать урон дальше по
цепочке незачем:

**`labs/lab3/defense/evasion-handler.ts`**

```typescript
import {Damage} from "./damage";
import {DamageHandler} from "./damage-handler";
import {Dice} from "../core/dice";
export class EvasionHandler extends DamageHandler {
    constructor(private chance: number) {
        super();
    }
    override handle(damage: Damage): void {
        if (Dice.getInstance().chance(this.chance)) {
            damage.evade();
            return;
        }
        this.passToNext(damage);
    }
}
```

Броня гасит долю урона и передает то, что осталось, следующему обработчику. Обратите внимание, что это то же самое правило,
что раньше было зашито в герое, скелете и драконе, а теперь оно записано в одном месте:

**`labs/lab3/defense/armor-handler.ts`**

```typescript
import {Damage} from "./damage";
import {DamageHandler} from "./damage-handler";
export class ArmorHandler extends DamageHandler {
    constructor(private defense: number) {
        super();
    }
    override handle(damage: Damage): void {
        damage.reduceTo(Math.max(1, Math.round(damage.amount * (1 - this.defense))));
        this.passToNext(damage);
    }
}
```

И магический барьер, который поглощает урон, пока у него есть запас прочности:

**`labs/lab3/defense/barrier-handler.ts`**

```typescript
import {Damage} from "./damage";
import {DamageHandler} from "./damage-handler";
export class BarrierHandler extends DamageHandler {
    constructor(private strength: number) {
        super();
    }
    override handle(damage: Damage): void {
        const absorbed = Math.min(this.strength, damage.amount);
        if (absorbed > 0) {
            this.strength -= absorbed;
            damage.absorb(absorbed);
        }
        this.passToNext(damage);
    }
}
```

Какие звенья будут в цепочке, зависит от героя. Цепочку собирает его класс - так же, как класс сам знает свою
фабрику стартового снаряжения:

**`labs/lab3/hero/hero-class.ts`**

```typescript
import {Armor} from "../item/armor";
import {DamageHandler} from "../defense/damage-handler";
import {ArmorHandler} from "../defense/armor-handler";
import {BarrierHandler} from "../defense/barrier-handler";
import {EvasionHandler} from "../defense/evasion-handler";
import {UnknownCodeError} from "../core/unknown-code-error";
import {ItemCatalog} from "../item/item-catalog";
import {MageKitFactory} from "../kit/mage-kit-factory";
import {RangerKitFactory} from "../kit/ranger-kit-factory";
import {StarterKitFactory} from "../kit/starter-kit-factory";
import {WarriorKitFactory} from "../kit/warrior-kit-factory";

// перечисление (enum) в TypeScript не может хранить данные и методы, поэтому классы героя - это
// набор заранее созданных объектов, а создать новые снаружи нельзя: конструктор приватный
export class HeroClass {

    // стрелочная функция, которая создает фабрику стартового снаряжения
    static readonly WARRIOR =
        new HeroClass("warrior", "Воин", 120, catalog => new WarriorKitFactory(catalog));
    static readonly RANGER =
        new HeroClass("ranger", "Следопыт", 100, catalog => new RangerKitFactory(catalog));
    static readonly MAGE = new HeroClass("mage", "Маг", 85, catalog => new MageKitFactory(catalog));

    // все классы героя по порядку
    static readonly values: readonly HeroClass[] = [ HeroClass.WARRIOR, HeroClass.RANGER, HeroClass.MAGE ];

    private constructor(
        readonly code: string,
        readonly displayName: string,
        readonly baseHealth: number,
        private readonly starterKit: (catalog: ItemCatalog) => StarterKitFactory,
    ) {}

    static byCode(code: string): HeroClass {
        const heroClass = HeroClass.values.find(candidate => candidate.code === code);
        if (heroClass === undefined) {
            throw new UnknownCodeError("Класс героя", code);
        }

        return heroClass;
    }

    // каждый класс сам знает свою фабрику стартового снаряжения
    getStarterKit(catalog: ItemCatalog): StarterKitFactory {
        return this.starterKit(catalog);
    }
    createDefense(armor: Armor, level: number): DamageHandler {
        let evasion = this === HeroClass.RANGER ? 0.15 : 0;
        let barrier = this === HeroClass.MAGE ? 30 : 0;
        if (this === HeroClass.WARRIOR && level >= 3)
            barrier = 20;
        else if (this === HeroClass.RANGER && level >= 5)
            barrier = 15;
        else if (this === HeroClass.MAGE && level >= 3)
            evasion = 0.10;
        const armorHandler = new ArmorHandler(armor.defense);
        let first: DamageHandler = armorHandler;
        if (evasion > 0) {
            first = new EvasionHandler(evasion);
            first.setNext(armorHandler);
        }
        if (barrier > 0)
            armorHandler.setNext(new BarrierHandler(barrier));
        return first;
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

**`labs/lab3/hero/hero.ts`**

```typescript
import {Damage} from "../defense/damage";
import {DamageHandler} from "../defense/damage-handler";
import {Achievement} from "../progress/achievement";
import {Combatant} from "../combat/combatant";
import {Hit} from "../combat/hit";
import {Armor} from "../item/armor";
import {Item} from "../item/item";
import {Weapon} from "../item/weapon";
import {StarterKitFactory} from "../kit/starter-kit-factory";
import {HeroClass} from "./hero-class";

const MAX_NAME_LENGTH = 20;
const STARTING_GOLD = 50;
const HEALTH_PER_LEVEL = 15;
const DAMAGE_PER_LEVEL = 3;
const EXPERIENCE_PER_LEVEL = 100;

// значения, которые строитель передает в конструктор героя
interface HeroParameters {
    readonly name: string;
    readonly heroClass: HeroClass;
    readonly weapon: Weapon;
    readonly armor: Armor;
    readonly level: number;
    readonly gold: number;
    readonly experience: number;
    readonly inventory: readonly Item[];
    readonly achievements: readonly Achievement[];
}

export class Hero implements Combatant {

    // внутренний класс билдер - чтобы удобно и безопасно создавать героев
    static readonly Builder = class {

        private name: string|undefined;
        private heroClass: HeroClass|undefined;
        private weapon: Weapon|undefined;
        private armor: Armor|undefined;
        // необязательные поля сразу получают значения по умолчанию
        private level = 1;
        private gold = STARTING_GOLD;
        private experience = 0;
        private inventory: readonly Item[] = [];
        private achievements: readonly Achievement[] = [];
        setExperience(experience: number): this {
            this.experience = experience;
            return this;
        }
        setInventory(inventory: readonly Item[]): this {
            this.inventory = [...inventory ];
            return this;
        }
        setAchievements(achievements: readonly Achievement[]): this {
            this.achievements = [...achievements ];
            return this;
        }

        setName(name: string): this {
            this.name = name;
            return this;
        }

        setHeroClass(heroClass: HeroClass): this {
            this.heroClass = heroClass;
            return this;
        }

        setWeapon(weapon: Weapon): this {
            this.weapon = weapon;
            return this;
        }

        setArmor(armor: Armor): this {
            this.armor = armor;
            return this;
        }

        // берет оружие и броню из стартового набора
        setStarterKit(starterKit: StarterKitFactory): this {
            this.weapon = starterKit.createWeapon();
            this.armor = starterKit.createArmor();
            return this;
        }

        setLevel(level: number): this {
            this.level = level;
            return this;
        }

        setGold(gold: number): this {
            this.gold = gold;
            return this;
        }

        // перед созданием героя проверяем, что все обязательные поля заполнены корректно
        build(): Hero {
            if (this.name === undefined || this.name.trim() === "") {
                throw new Error("У героя должно быть имя");
            }
            if (this.name.length > MAX_NAME_LENGTH) {
                throw new Error(`Имя героя не может быть длиннее ${MAX_NAME_LENGTH} символов`);
            }
            if (this.heroClass === undefined) {
                throw new Error("Не выбран класс героя");
            }
            if (this.weapon === undefined || this.armor === undefined) {
                throw new Error("Герою нужны оружие и броня");
            }
            if (this.level < 1) {
                throw new Error("Уровень героя не может быть меньше 1");
            }
            if (this.gold < 0) {
                throw new Error("Золота не может быть меньше нуля");
            }

            if (this.experience < 0)
                throw new Error("Опыт не может быть отрицательным");
            return new Hero({
                name : this.name,
                heroClass : this.heroClass,
                weapon : this.weapon,
                armor : this.armor,
                level : this.level,
                gold : this.gold,
                experience : this.experience,
                inventory : this.inventory,
                achievements : this.achievements,
            });
        }
    };

    readonly name: string;
    readonly heroClass: HeroClass;
    private currentLevel: number;
    private currentExperience = 0;
    private currentGold: number;
    private currentHealth = 0;
    private defense!: DamageHandler;
    private readonly achievementList: Achievement[];
    private currentWeapon: Weapon;
    private currentArmor: Armor;
    private readonly inventoryList: Item[] = [];

    // конструктор приватный: создать героя можно только через строителя
    private constructor(parameters: HeroParameters) {
        this.name = parameters.name;
        this.heroClass = parameters.heroClass;
        this.currentLevel = parameters.level;
        this.currentGold = parameters.gold;
        this.currentWeapon = parameters.weapon;
        this.currentArmor = parameters.armor;
        this.currentExperience = parameters.experience;
        this.inventoryList.push(...parameters.inventory);
        this.achievementList = [...new Set(parameters.achievements) ];
        this.rest();
    }

    // свойства только для чтения: снаружи их можно прочитать, а поменять может только сам герой

    get code(): string {
        return this.heroClass.code;
    }

    get health(): number {
        return this.currentHealth;
    }

    get maxHealth(): number {
        return this.heroClass.baseHealth + (this.currentLevel - 1) * HEALTH_PER_LEVEL;
    }

    get level(): number {
        return this.currentLevel;
    }

    get experience(): number {
        return this.currentExperience;
    }

    get experienceToNextLevel(): number {
        return this.currentLevel * EXPERIENCE_PER_LEVEL;
    }

    get gold(): number {
        return this.currentGold;
    }

    get weapon(): Weapon {
        return this.currentWeapon;
    }

    get armor(): Armor {
        return this.currentArmor;
    }

    get inventory(): readonly Item[] {
        return [...this.inventoryList ];
    }

    rollDamage(): Hit {
        const hit = this.currentWeapon.rollDamage();
        return {damage : hit.damage + (this.currentLevel - 1) * DAMAGE_PER_LEVEL, critical : hit.critical};
    }

    receiveHit(amount: number): Damage {
        const damage = new Damage(amount);
        this.defense.handle(damage);
        this.currentHealth = Math.max(0, this.currentHealth - damage.amount);
        return damage;
    }

    isAlive(): boolean {
        return this.currentHealth > 0;
    }

    // поведение героя между экспедициями

    // перед каждой экспедицией герой успевает отдохнуть
    rest(): void {
        this.currentHealth = this.maxHealth;
        this.defense = this.heroClass.createDefense(this.armor, this.level);
    }

    // начисляет опыт, возвращает количество полученных уровней
    gainExperience(amount: number): number {
        this.currentExperience += amount;
        let levelsGained = 0;
        while (this.currentExperience >= this.experienceToNextLevel) {
            this.currentExperience -= this.experienceToNextLevel;
            this.currentLevel++;
            levelsGained++;
        }

        return levelsGained;
    }

    addGold(amount: number): void {
        this.currentGold += amount;
    }

    spendGold(amount: number): void {
        if (amount > this.currentGold) {
            throw new Error(`Не хватает золота: нужно ${amount}, есть ${this.currentGold}`);
        }

        this.currentGold -= amount;
    }

    addItem(item: Item): void {
        this.inventoryList.push(item);
    }

    removeItem(item: Item): void {
        const index = this.inventoryList.indexOf(item);
        if (index < 0) {
            throw new Error(`В инвентаре нет предмета ${item.name}`);
        }

        this.inventoryList.splice(index, 1);
    }

    // экипирует предмет из инвентаря, снятый предмет возвращается в инвентарь
    equip(item: Item): void {
        if (item instanceof Weapon) {
            this.removeItem(item);
            this.inventoryList.push(this.currentWeapon);
            this.currentWeapon = item;
        } else if (item instanceof Armor) {
            this.removeItem(item);
            this.inventoryList.push(this.currentArmor);
            this.currentArmor = item;
            this.defense = this.heroClass.createDefense(this.armor, this.level);
        } else {
            throw new Error(`Предмет нельзя экипировать: ${item.name}`);
        }
    }
    get achievements(): readonly Achievement[] {
        return [...this.achievementList ];
    }
    unlock(achievement: Achievement): boolean {
        if (this.achievementList.includes(achievement))
            return false;
        this.achievementList.push(achievement);
        return true;
    }
}
```

У врагов цепочка задается прямо в конструкторе. Общий класс врага пропускает урон через цепочку, если она есть:

**`labs/lab3/enemy/abstract-enemy.ts`**

```typescript
import {Damage} from "../defense/damage";
import {DamageHandler} from "../defense/damage-handler";
import {Hit} from "../combat/hit";
import {Dice} from "../core/dice";
import {Enemy} from "./enemy";

// общая логика для врагов: здоровье, урон в диапазоне, награда
export abstract class AbstractEnemy implements Enemy {

    private currentHealth: number;

    protected constructor(
        readonly code: string,
        readonly name: string,
        readonly maxHealth: number,
        private readonly minDamage: number,
        private readonly maxDamage: number,
        readonly experienceReward: number,
        private readonly goldReward: number,
        private readonly defense?: DamageHandler,
    ) {
        this.currentHealth = maxHealth;
    }

    get health(): number {
        return this.currentHealth;
    }

    isAlive(): boolean {
        return this.currentHealth > 0;
    }

    rollDamage(): Hit {
        return {damage : Dice.getInstance().roll(this.minDamage, this.maxDamage), critical : false};
    }

    receiveHit(amount: number): Damage {
        const damage = new Damage(amount);
        this.defense?.handle(damage);
        this.currentHealth = Math.max(0, this.currentHealth - damage.amount);
        return damage;
    }

    rollGoldReward(): number {
        return Dice.getInstance().roll(Math.floor(this.goldReward / 2), this.goldReward);
    }
}
```

У гоблина защиты нет:

**`labs/lab3/enemy/goblin.ts`**

```typescript
import { AbstractEnemy } from "./abstract-enemy";

// гоблин - самый простой враг, без особенностей
export class Goblin extends AbstractEnemy {

    constructor() {
        super("goblin", "Гоблин", 50, 6, 14, 40, 30);
    }
}
```

Кости скелета и чешуя дракона работают как броня, поэтому метод `reduceDamage` им больше не нужен:

**`labs/lab3/enemy/skeleton.ts`**

```typescript
import {ArmorHandler} from "../defense/armor-handler";
import {AbstractEnemy} from "./abstract-enemy";

const RESISTANCE = 0.3;

// скелет: кости плохо поддаются ударам, поэтому часть урона проходит мимо
export class Skeleton extends AbstractEnemy {

    constructor() {
        super("skeleton", "Скелет", 110, 10, 24, 90, 70, new ArmorHandler(RESISTANCE));
    }
}
```

**`labs/lab3/enemy/dragon.ts`**

```typescript
import {ArmorHandler} from "../defense/armor-handler";
import {Hit} from "../combat/hit";
import {Dice} from "../core/dice";
import {AbstractEnemy} from "./abstract-enemy";

const RESISTANCE = 0.25;
const FIRE_BREATH_CHANCE = 0.25;
const FIRE_BREATH_MULTIPLIER = 2;

// дракон защищен чешуей и время от времени дышит огнем
export class Dragon extends AbstractEnemy {

    constructor() {
        super("dragon", "Дракон", 260, 16, 22, 500, 400, new ArmorHandler(RESISTANCE));
    }

    override rollDamage(): Hit {
        const bite = super.rollDamage();
        if (Dice.getInstance().chance(FIRE_BREATH_CHANCE)) {
            return {damage : bite.damage * FIRE_BREATH_MULTIPLIER, critical : true};
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

**`labs/lab3/game/game-facade.ts`**

```typescript
import {Merchant} from "../camp/merchant";
import {Hero} from "../hero/hero";
import {HeroClass} from "../hero/hero-class";
import {Item} from "../item/item";
import {Forest} from "../world/forest";
import {Crypt} from "../world/crypt";
import {DragonLair} from "../world/dragon-lair";
import {HauntedManor} from "../world/haunted-manor";
import {LocationRegistry} from "../world/location-registry";
import {Tactic} from "../tactic/tactic";
import {BalancedTactic} from "../tactic/balanced-tactic";
import {AggressiveTactic} from "../tactic/aggressive-tactic";
import {CautiousTactic} from "../tactic/cautious-tactic";
import {GameEventPublisher} from "../event/game-event-publisher";
import {AchievementTracker} from "../progress/achievement-tracker";
import {ExpeditionJournal} from "../progress/expedition-journal";
import {FileHeroRepository} from "../storage/file-hero-repository";
import {CachingHeroRepository} from "../storage/caching-hero-repository";
import {
    createDefaultCatalog,
    IRON_SWORD,
    ELVEN_BOW,
    EMBER_STAFF,
    CHAINMAIL,
    PLATE_ARMOR
} from "../item/default-catalog";
export class GameFacade {
    private readonly catalog = createDefaultCatalog();
    private readonly registry = new LocationRegistry([
        new Forest(this.catalog), new HauntedManor(this.catalog), new Crypt(this.catalog),
        new DragonLair(this.catalog)
    ]);
    readonly tactics: readonly Tactic[] =
        [ new BalancedTactic(), new AggressiveTactic(), new CautiousTactic() ];
    private readonly merchant =
        new Merchant(this.catalog, [ IRON_SWORD, ELVEN_BOW, EMBER_STAFF, CHAINMAIL, PLATE_ARMOR ]);
    private readonly repository: CachingHeroRepository;
    private readonly events = new GameEventPublisher();
    private readonly journal = new ExpeditionJournal();
    constructor(saveFile: string) {
        this.repository = new CachingHeroRepository(new FileHeroRepository(saveFile, this.catalog));
        const achievements = new AchievementTracker();
        this.events.subscribe("victory", achievements);
        this.events.subscribe("level-up", achievements);
        this.events.subscribe("victory", this.journal);
        this.events.subscribe("defeat", this.journal);
    }
    get heroClasses() {
        return HeroClass.values;
    }
    getStarterKit(code: string): readonly Item[] {
        const kit = HeroClass.byCode(code).getStarterKit(this.catalog);
        return [ kit.createWeapon(), kit.createArmor() ];
    }
    get locations() {
        return this.registry.all;
    }
    getLocation(code: string) {
        return this.registry.get(code);
    }
    getTactic(code: string): Tactic {
        const tactic = this.tactics.find(t => t.code === code);
        if (!tactic)
            throw new Error("Неизвестная тактика: " + code);
        return tactic;
    }
    get showcase() {
        return this.merchant.showcase;
    }
    getSellPrice(item: Item) {
        return this.merchant.getSellPrice(item);
    }
    get heroes() {
        return this.repository.findAll();
    }
    getHero(name: string): Hero {
        const hero = this.repository.findByName(name);
        if (!hero)
            throw new Error("Героя с именем " + name + " нет");
        return hero;
    }
    createHero(name: string, code: string): Hero {
        if (this.repository.findByName(name))
            throw new Error("Герой с именем " + name + " уже есть");
        const heroClass = HeroClass.byCode(code);
        const hero = new Hero.Builder()
                         .setName(name)
                         .setHeroClass(heroClass)
                         .setStarterKit(heroClass.getStarterKit(this.catalog))
                         .build();
        this.repository.save(hero);
        return hero;
    }
    explore(name: string, locationCode: string, tacticCode: string) {
        const hero = this.getHero(name);
        const expedition =
            this.getLocation(locationCode).explore(hero, undefined, this.getTactic(tacticCode));
        this.events.publish({type : expedition.battle.outcome, hero, expedition});
        if (expedition.rewards.levelsGained > 0)
            this.events.publish({type : "level-up", hero, expedition});
        this.repository.save(hero);
        return expedition;
    }
    equip(name: string, index: number): Item {
        const hero = this.getHero(name), item = hero.inventory[index];
        hero.equip(item);
        this.repository.save(hero);
        return item;
    }
    buy(name: string, code: string): Item {
        const hero = this.getHero(name), item = this.merchant.buy(hero, code);
        this.repository.save(hero);
        return item;
    }
    sell(name: string, index: number): number {
        const hero = this.getHero(name), price = this.merchant.sell(hero, hero.inventory[index]);
        this.repository.save(hero);
        return price;
    }
    getJournal(name: string) {
        return this.journal.getEntries(name);
    }
}
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

**`labs/lab3/console-view.ts`**

```typescript
import {BattleEvent} from "./combat/battle-event";
import {BattleReport} from "./combat/battle-report";
import {Participant} from "./combat/participant";
import {Side} from "./combat/side";
import {Hero} from "./hero/hero";
import {Item} from "./item/item";
import {ExpeditionReport} from "./world/expedition-report";
import {Rewards} from "./world/rewards";

// весь вывод в консоль собран здесь
export class ConsoleView {

    describe(item: Item): string {
        let description = item.name;

        const enchantments = item.enchantments;
        if (enchantments.length > 0) {
            const names: string[] = [];
            for (const enchantment of enchantments) {
                names.push(enchantment.name);
            }
            description += ` [${names.join(", ")}]`;
        }

        return `${description} (${item.description})`;
    }

    printHero(hero: Hero): void {
        console.log(
            `${hero.name}, ${hero.heroClass.displayName} ${hero.level} ур. | здоровье ${hero.maxHealth} | ` +
                `опыт ${hero.experience}/${hero.experienceToNextLevel} | золото ${hero.gold}`,
        );
        console.log("  Оружие: " + this.describe(hero.weapon));
        console.log("  Броня:  " + this.describe(hero.armor));
        if (hero.achievements.length)
            console.log("  Достижения: " + hero.achievements.map(a => a.displayName).join(", "));
    }

    summarize(report: ExpeditionReport, locationName: string, tacticName: string): string {
        const result = report.battle.outcome === "victory" ? "победа" : "поражение";
        return `${locationName}, ${report.battle.enemy.name}, тактика "${tacticName}": ${result}`;
    }

    printReport(report: ExpeditionReport, locationName: string): void {
        const battle = report.battle;
        console.log(`=== ${locationName} (сид ${report.seed}) ===`);
        console.log(
            `${battle.hero.name} встречает врага: ${battle.enemy.name} (${battle.enemy.maxHealth} здоровья)`);

        for (const event of battle.events) {
            this.printEvent(event, battle);
        }

        this.printRewards(battle, report.rewards);
    }

    private printEvent(event: BattleEvent, battle: BattleReport): void {
        const source = this.participant(event.source, battle);
        const target = this.participant(event.target, battle);

        switch (event.type) {
        case "hit":
            console.log(
                `[раунд ${event.round}] ${source.name} -> ${target.name}: ${event.amount} урона, ` +
                    `осталось ${event.targetHealthAfter}/${target.maxHealth}`,
            );
            break;
        case "critical-hit":
            console.log(
                `[раунд ${event.round}] ${source.name} -> ${target.name}: КРИТ! ${event.amount} урона, ` +
                    `осталось ${event.targetHealthAfter}/${target.maxHealth}`,
            );
            break;
        case "evade":
            console.log(
                `[раунд ${event.round}] ${source.name} -> ${target.name}: промах, ${target.name} уклоняется`);
            break;
        case "absorb":
            console.log(`[раунд ${event.round}] ${source.name} -> ${target.name}: барьер поглощает ${
                event.amount} урона`);
            break;
        case "power-strike":
            console.log(`[раунд ${event.round}] ${source.name} бьет в полную силу`);
            break;
        case "defend":
            console.log(`[раунд ${event.round}] ${source.name} уходит в оборону`);
            break;
        case "death":
            console.log(`${target.name} повержен!`);
            break;
        }
    }

    private printRewards(battle: BattleReport, rewards: Rewards): void {
        if (battle.outcome === "defeat") {
            console.log(
                `Поражение... ${battle.hero.name} едва уносит ноги и теряет ${- rewards.gold} золота`);
            return;
        }

        console.log(`Победа! +${rewards.experience} опыта, +${rewards.gold} золота`);
        if (rewards.loot !== null) {
            console.log("Добыча: " + this.describe(rewards.loot));
        }
        if (rewards.levelsGained > 0) {
            console.log("Новый уровень!");
        }
    }

    private participant(side: Side, battle: BattleReport): Participant {
        return side === "hero" ? battle.hero : battle.enemy;
    }
}
```

А игровой цикл в методе `main` теперь работает только с фасадом и хранит текущего героя:

Вместо прямой работы с каталогом, локациями и торговцем:

- создаём `GameFacade` с файлом сохранения и выбираем нового или сохранённого героя;
- перед экспедицией выбираем тактику;
- работам с экспедициями, экипировкой и торговлей через фасад;
- добавляем пункт меню для журнала экспедиций.

**`labs/lab3/main.ts`**

```typescript
import {GameFacade} from "./game/game-facade";
import {join, resolve} from "node:path";
import * as readline from "node:readline";
import {Merchant} from "./camp/merchant";
import {ConsoleView} from "./console-view";
import {Hero} from "./hero/hero";
import {HeroClass} from "./hero/hero-class";
import {
    createDefaultCatalog,
    CHAINMAIL,
    ELVEN_BOW,
    EMBER_STAFF,
    IRON_SWORD,
    PLATE_ARMOR
} from "./item/default-catalog";
import {Item} from "./item/item";
import {ItemCatalog} from "./item/item-catalog";
import {Crypt} from "./world/crypt";
import {DragonLair} from "./world/dragon-lair";
import {Forest} from "./world/forest";
import {LocationRegistry} from "./world/location-registry";

const view = new ConsoleView();

// в Node.js консоль читается асинхронно: строки ввода приходят по одной, и мы ждем каждую через await
const lines = readline.createInterface({input : process.stdin})[Symbol.asyncIterator]();

async function main(): Promise<void> {
    const saveFile = join("saves", "heroes.json");
    console.log("Файл сохранения: " + resolve(saveFile));
    const game = new GameFacade(saveFile);
    const hero = await chooseHero(game);
    console.log(`${hero.name} разбивает лагерь на опушке мистического леса.`);

    while (true) {
        console.log();
        view.printHero(hero);
        console.log(
            "1 - отправиться в экспедицию, 2 - снаряжение, 3 - торговец, 4 - журнал, 0 - выйти из игры");
        switch (await readLine()) {
        case "1":
            await goOnExpedition(hero, game);
            break;
        case "2":
            await manageEquipment(hero, game);
            break;
        case "3":
            await visitMerchant(hero, game);
            break;
        case "4":
            for (const report of game.getJournal(hero.name))
                console.log(view.summarize(report, game.getLocation(report.locationCode).displayName,
                                           game.getTactic(report.tacticCode).displayName));
            break;
        case "0":
            process.exit(0);
        default:
            console.log("Такого варианта нет");
        }
    }
}

async function chooseHero(game: GameFacade): Promise<Hero> {
    const heroes = game.heroes;
    if (!heroes.length)
        return createHero(game);
    console.log("Выберите сохраненного героя (0 - создать нового):");
    heroes.forEach(
        (hero, i) => console.log(`${i + 1} - ${hero.name}, ${hero.heroClass.displayName} ${hero.level} ур.`));
    const index = await readChoice(heroes.length);
    return index >= 0 ? heroes[index] : createHero(game);
}

async function createHero(game: GameFacade): Promise<Hero> {
    console.log("Создайте своего героя!");
    while (true) {
        console.log("Выберите класс:");
        const heroClasses = HeroClass.values;
        heroClasses.forEach((heroClass, i) => {
            const kit = game.getStarterKit(heroClass.code);
            console.log(
                `${i + 1} - ${heroClass.displayName} (здоровье ${heroClass.baseHealth}): ` +
                    `${view.describe(kit[0])}, ${view.describe(kit[1])}`,
            );
        });
        const classIndex = await readChoice(heroClasses.length);
        if (classIndex < 0) {
            continue;
        }
        const heroClass = heroClasses[classIndex];

        console.log("Введите имя:");
        const name = await readLine();

        try {
            return game.createHero(name, heroClass.code);
        } catch (error) {
            console.log("Не получилось создать героя: " + (error as Error).message);
        }
    }
}

async function goOnExpedition(hero: Hero, game: GameFacade): Promise<void> {
    const all = game.locations;
    console.log("Куда отправимся?");
    all.forEach((location, i) => {
        console.log(
            `${i + 1} - ${location.displayName} (рекомендуемый уровень ${location.recommendedLevel})`);
    });
    const index = await readChoice(all.length);
    if (index < 0) {
        return;
    }

    const location = all[index];
    console.log("Выберите тактику:");
    game.tactics.forEach((tactic, i) =>
                             console.log(`${i + 1} - ${tactic.displayName}: ${tactic.description}`));
    const tacticIndex = await readChoice(game.tactics.length);
    if (tacticIndex < 0)
        return;
    const report = game.explore(hero.name, location.code, game.tactics[tacticIndex].code);
    console.log();
    view.printReport(report, location.displayName);

    if (report.battle.outcome === "victory" && report.battle.enemy.code === "dragon") {
        console.log(`\nДракон повержен! Барды еще долго будут петь о подвиге героя по имени ${hero.name}.`);
    }
}

async function manageEquipment(hero: Hero, game: GameFacade): Promise<void> {
    const inventory = hero.inventory;
    if (inventory.length === 0) {
        console.log("Инвентарь пуст");
        return;
    }

    console.log("Что экипировать? (0 - назад)");
    printItems(inventory);
    const index = await readChoice(inventory.length);
    if (index < 0) {
        return;
    }

    const item = inventory[index];
    game.equip(hero.name, index);
    console.log("Экипировано: " + view.describe(item));
}

async function visitMerchant(hero: Hero, game: GameFacade): Promise<void> {
    console.log(`Торговец: "Чего желаешь?" (золото: ${hero.gold})`);
    console.log("1 - купить, 2 - продать, 0 - назад");
    switch (await readLine()) {
    case "1":
        await buy(hero, game);
        break;
    case "2":
        await sell(hero, game);
        break;
    }
}

async function buy(hero: Hero, game: GameFacade): Promise<void> {
    const showcase = game.showcase;
    showcase.forEach(
        (item, i) => { console.log(`${i + 1} - ${view.describe(item)}: ${item.price} золота`); });
    const index = await readChoice(showcase.length);
    if (index < 0) {
        return;
    }

    try {
        const item = game.buy(hero.name, showcase[index].code);
        console.log("Куплено: " + view.describe(item));
    } catch (error) {
        console.log((error as Error).message);
    }
}

async function sell(hero: Hero, game: GameFacade): Promise<void> {
    const inventory = hero.inventory;
    if (inventory.length === 0) {
        console.log("Продавать нечего (экипированные предметы продать нельзя)");
        return;
    }

    inventory.forEach(
        (item,
         i) => { console.log(`${i + 1} - ${view.describe(item)}: ${game.getSellPrice(item)} золота`); });
    const index = await readChoice(inventory.length);
    if (index < 0) {
        return;
    }

    const item = inventory[index];
    const price = game.sell(hero.name, index);
    console.log(`Продано: ${item.name} за ${price} золота`);
}

function printItems(items: readonly Item[]): void {
    items.forEach((item, i) => console.log(`${i + 1} - ${view.describe(item)}`));
}

// читает строку без пробелов по краям; если ввод закончился, завершает программу
async function readLine(): Promise<string> {
    const line = await lines.next();
    if (line.done) {
        process.exit(0);
    }

    return line.value.trim();
}

// читает номер варианта от 1 до count, возвращает индекс или -1, если ввели что-то другое
async function readChoice(count: number): Promise<number> {
    const choice = Number(await readLine());
    if (Number.isInteger(choice) && choice >= 1 && choice <= count) {
        return choice - 1;
    }

    return -1;
}

void main();
```

Супер! Попробуйте запустить игру, сохраниться и вернуться к своему герою :)

## Демонстрация паттернов

Чтобы показать все паттерны одним запуском, напишем демонстрационную программу:

Демонстрацию лабы 3 заменяем новой:

**`labs/lab3/demo.ts`**

```typescript
import {Dice} from "./core/dice";
import {ConsoleView} from "./console-view";
import {Damage} from "./defense/damage";
import {DamageHandler} from "./defense/damage-handler";
import {ArmorHandler} from "./defense/armor-handler";
import {BarrierHandler} from "./defense/barrier-handler";
import {EvasionHandler} from "./defense/evasion-handler";
import {Hero} from "./hero/hero";
import {HeroClass} from "./hero/hero-class";
import {ItemCatalog} from "./item/item-catalog";
import {Goblin} from "./enemy/goblin";
import {Dragon} from "./enemy/dragon";
import {LegendaryEnemy} from "./enemy/legendary-enemy";
import {WindfuryEnemy} from "./enemy/windfury-enemy";
import {ArmoredEnemy} from "./enemy/armored-enemy";
import {HauntedWeapon} from "./enemy/haunted-weapon";
import {BalancedTactic} from "./tactic/balanced-tactic";
import {AggressiveTactic} from "./tactic/aggressive-tactic";
import {CautiousTactic} from "./tactic/cautious-tactic";
import {Crypt} from "./world/crypt";
import {Forest} from "./world/forest";
import {HauntedManor} from "./world/haunted-manor";
import {GameEventPublisher} from "./event/game-event-publisher";
import {AchievementTracker} from "./progress/achievement-tracker";
import {ExpeditionJournal} from "./progress/expedition-journal";
import {FileHeroRepository} from "./storage/file-hero-repository";
import {CachingHeroRepository} from "./storage/caching-hero-repository";
import {GameFacade} from "./game/game-facade";
import {createDefaultCatalog, ELVEN_BOW} from "./item/default-catalog";
import {mkdtempSync, readFileSync, rmSync} from "node:fs";
import {join} from "node:path";
import {tmpdir} from "node:os";
const view = new ConsoleView();
const catalog = createDefaultCatalog();
const dice = Dice.getInstance();
function newHero(name: string, heroClass: HeroClass, level: number): Hero {
    return new Hero.Builder()
        .setName(name)
        .setHeroClass(heroClass)
        .setStarterKit(heroClass.getStarterKit(catalog))
        .setLevel(level)
        .build();
}
function printHits(title: string, chain: DamageHandler, count: number): void {
    const results: string[] = [];
    for (let i = 0; i < count; i++) {
        const damage = new Damage(30);
        chain.handle(damage);
        results.push(damage.evaded         ? "уклонение"
                     : damage.absorbed > 0 ? `поглощено ${damage.absorbed}, прошло ${damage.amount}`
                                           : `прошло ${damage.amount}`);
    }
    console.log(title + ": " + results.join("; "));
}
console.log("===== Цепочка обязанностей =====");
printHits("Броня 30%", new ArmorHandler(0.3), 2);
const armor = new ArmorHandler(0.3);
armor.setNext(new BarrierHandler(30));
printHits("Броня 30% -> барьер 30", armor, 3);
dice.reseed(3);
const chain = new EvasionHandler(0.5);
chain.setNext(new ArmorHandler(0.3)).setNext(new BarrierHandler(30));
printHits("Уклонение 50% -> броня 30% -> барьер 30", chain, 3);
console.log("\n===== Стратегия =====");
for (const tactic of [new BalancedTactic(), new AggressiveTactic(), new CautiousTactic()]) {
    const mage = newHero("Мерлин", HeroClass.MAGE, 2), report = new Crypt(catalog).explore(mage, 9, tactic);
    const result = report.battle.outcome === "victory" ? "победа" : "поражение";
    console.log(`${tactic.displayName}: ${result}, раундов ${
        report.battle.events.at(-1)!.round}, у героя осталось ${mage.health}/${mage.maxHealth} здоровья`);
}
console.log("\n===== Декоратор =====");
for (const enemy of [new Goblin(), new LegendaryEnemy(new Goblin()), new WindfuryEnemy(new Goblin()),
                     new LegendaryEnemy(new WindfuryEnemy(new Goblin())), new ArmoredEnemy(new Goblin()),
                     new ArmoredEnemy(new LegendaryEnemy(new WindfuryEnemy(new Dragon())))]) {
    dice.reseed(5);
    const hits = Array.from({length : 3}, () => enemy.rollDamage().damage).join(", ");
    const damage = enemy.receiveHit(20);
    console.log(`${enemy.name}: удары ${hits}, награда ${enemy.experienceReward} опыта; входящий урон 20 -> ${
        damage.amount}, здоровье ${enemy.health}/${enemy.maxHealth}`);
}
console.log("\n===== Адаптер =====");
const bow = catalog.createWeapon(ELVEN_BOW), haunted = new HauntedWeapon(bow);
console.log("Предмет: " + view.describe(bow));
console.log(
    `Враг:    ${haunted.name} (${haunted.maxHealth} здоровья, награда ${haunted.experienceReward} опыта)`);
const manor = new HauntedManor(catalog);
view.printReport(manor.explore(newHero("Леголас", HeroClass.RANGER, 2), 55, new CautiousTactic()),
                 manor.displayName);
console.log("\n===== Наблюдатель =====");
const events = new GameEventPublisher(), journal = new ExpeditionJournal();
events.subscribe("victory", new AchievementTracker());
events.subscribe("victory", journal);
events.subscribe("defeat", journal);
events.subscribe("victory",
                 {onEvent : event => console.log(`Событие ${event.type}: герой ${event.hero.name}`)});
const warrior = newHero("Артур", HeroClass.WARRIOR, 1), report = new Forest(catalog).explore(warrior, 7);
events.publish({type : report.battle.outcome, hero : warrior, expedition : report});
console.log("Достижения героя: " + warrior.achievements.map(a => a.displayName).join(", "));
console.log("Записей в журнале: " + journal.getEntries(warrior.name).length);
const directory = mkdtempSync(join(tmpdir(), "lab-demo-"));
try {
    console.log("\n===== Заместитель =====");
    const path = join(directory, "heroes.json"), file = new FileHeroRepository(path, catalog);
    file.save(warrior);
    console.log("Содержимое файла:");
    process.stdout.write(readFileSync(path, "utf8"));
    console.log("Без заместителя два поиска дают один и тот же объект: " +
                (file.findByName("Артур") === file.findByName("Артур")));
    const cached = new CachingHeroRepository(file), hero = cached.findByName("Артур")!;
    console.log("С заместителем два поиска дают один и тот же объект: " +
                (hero === cached.findByName("Артур")));
    hero.addGold(100);
    cached.save(hero);
    console.log("После сохранения через заместитель золото героя в файле: " + file.findByName("Артур")!.gold);
    console.log("\n===== Фасад =====");
    const game = new GameFacade(join(directory, "game.json"));
    game.createHero("Гэндальф", "mage");
    dice.reseed(11);
    const expedition = game.explore("Гэндальф", "forest", "cautious");
    console.log("Экспедиция через фасад: " +
                view.summarize(expedition, game.getLocation(expedition.locationCode).displayName,
                               game.getTactic(expedition.tacticCode).displayName));
    view.printHero(game.getHero("Гэндальф"));
} finally {
    rmSync(directory, {recursive : true});
}
```

Вывод программы:

```text
===== Цепочка обязанностей =====
Броня 30%: прошло 21; прошло 21
Броня 30% -> барьер 30: поглощено 21, прошло 0; поглощено 9, прошло 12; прошло 21
Уклонение 50% -> броня 30% -> барьер 30: поглощено 21, прошло 0; уклонение; уклонение

===== Стратегия =====
Баланс: поражение, раундов 4, у героя осталось 0/100 здоровья
Натиск: поражение, раундов 3, у героя осталось 0/100 здоровья
Осторожность: поражение, раундов 5, у героя осталось 0/100 здоровья

===== Декоратор =====
Гоблин: удары 12, 12, 7, награда 40 опыта; входящий урон 20 -> 20, здоровье 30/50
Легендарный Гоблин: удары 24, 24, 14, награда 80 опыта; входящий урон 20 -> 20, здоровье 30/50
Неистовый Гоблин: удары 24, 18, 17, награда 60 опыта; входящий урон 20 -> 20, здоровье 30/50
Легендарный Неистовый Гоблин: удары 48, 36, 34, награда 120 опыта; входящий урон 20 -> 20, здоровье 30/50
Бронированный Гоблин: удары 12, 12, 7, награда 40 опыта; входящий урон 20 -> 10, здоровье 40/50
Бронированный Легендарный Неистовый Дракон: удары 74, 74, 130, награда 1500 опыта; входящий урон 20 -> 8, здоровье 252/260

===== Адаптер =====
Предмет: Эльфийский лук (урон 14, шанс крита 40%)
Враг:    Одержимый Эльфийский лук (70 здоровья, награда 45 опыта)
=== Проклятый особняк (сид 55) ===
Леголас встречает врага: Одержимый Эльфийский лук (70 здоровья)
[раунд 1] Леголас -> Одержимый Эльфийский лук: 13 урона, осталось 57/70
[раунд 1] Одержимый Эльфийский лук -> Леголас: промах, Леголас уклоняется
[раунд 2] Леголас -> Одержимый Эльфийский лук: КРИТ! 23 урона, осталось 34/70
[раунд 2] Одержимый Эльфийский лук -> Леголас: 11 урона, осталось 104/115
[раунд 3] Леголас -> Одержимый Эльфийский лук: 13 урона, осталось 21/70
[раунд 3] Одержимый Эльфийский лук -> Леголас: КРИТ! 22 урона, осталось 82/115
[раунд 4] Леголас -> Одержимый Эльфийский лук: 13 урона, осталось 8/70
[раунд 4] Одержимый Эльфийский лук -> Леголас: промах, Леголас уклоняется
[раунд 5] Леголас -> Одержимый Эльфийский лук: 13 урона, осталось 0/70
Одержимый Эльфийский лук повержен!
Победа! +45 опыта, +32 золота
Добыча: Железный меч [Острое] (урон 20)

===== Наблюдатель =====
Событие victory: герой Артур
Достижения героя: Первая кровь
Записей в журнале: 1

===== Заместитель =====
Содержимое файла:
[
  {
    "name": "Артур",
    "heroClass": "warrior",
    "level": 2,
    "experience": 20,
    "gold": 130,
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
Без заместителя два поиска дают один и тот же объект: false
С заместителем два поиска дают один и тот же объект: true
После сохранения через заместитель золото героя в файле: 230

===== Фасад =====
Экспедиция через фасад: Мистический лес, Гоблин, тактика "Осторожность": победа
Гэндальф, Маг 1 ур. | здоровье 85 | опыт 40/100 | золото 68
  Оружие: Посох ученика (урон 10-18)
  Броня:  Льняная роба (защита 10%)
  Достижения: Первая кровь
```

Не забудьте закоммитить и запушить изменения!

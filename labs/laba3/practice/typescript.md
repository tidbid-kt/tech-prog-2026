# Лаба 3 — практика на TypeScript

Это практическая часть [лабораторной работы 3](../laba3.md): требования к сдаче и вопросы к защите находятся в
основном тексте лабы, а здесь шаг за шагом разобрано эталонное решение.

Понадобится Node.js 22 или новее. Шаблон репозитория: [tech-prog-labs-2026-typescript](https://github.com/tidbid-kt/tech-prog-labs-2026-typescript), после клонирования установите компилятор TypeScript командой `npm install`. Код лабы пишите в папке `labs/lab3`.

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

```typescript
// Одиночка: общий источник случайности на всю игру
export class Dice {

    // единственный экземпляр класса
    private static instance: Dice | undefined;

    // состояние генератора случайных чисел
    private state: number;

    // конструктор приватный - самостоятельно создать экземпляр класса нельзя
    private constructor() {
        this.state = Math.floor(Math.random() * 2 ** 32);
    }

    // метод для получения одиночки
    static getInstance(): Dice {
        if (Dice.instance === undefined) {
            Dice.instance = new Dice();
        }

        return Dice.instance;
    }

    // после установки сида все последующие броски повторяются в точности
    reseed(seed: number): void {
        this.state = seed >>> 0;
    }

    // случайный сид
    nextSeed(): number {
        return this.roll(0, 999_999);
    }

    // случайное целое число от min до max включительно
    roll(min: number, max: number): number {
        return min + Math.floor(this.next() * (max - min + 1));
    }

    // возвращает true с вероятностью probability (от 0 до 1)
    chance(probability: number): boolean {
        return this.next() < probability;
    }

    // случайный элемент массива
    pick<T>(options: readonly T[]): T {
        return options[Math.floor(this.next() * options.length)];
    }

    // сид для встроенного Math.random задать нельзя, поэтому генератор написан свой (алгоритм mulberry32):
    // каждый вызов возвращает следующее число от 0 до 1 в последовательности, которую полностью определяет сид
    private next(): number {
        this.state = (this.state + 0x6d2b79f5) >>> 0;
        let t = this.state;
        t = Math.imul(t ^ (t >>> 15), t | 1);
        t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
        return ((t ^ (t >>> 14)) >>> 0) / 2 ** 32;
    }
}
```

Модификатор `private` в TypeScript проверяет компилятор: в получившемся JavaScript его уже нет, но код с `new Dice()`
просто не скомпилируется. И еще одна особенность: задать сид встроенному `Math.random` нельзя, поэтому генератор
случайных чисел мы пишем сами - это небольшой известный алгоритм mulberry32.

Давайте разберем, что в этой реализации важно.

1. Конструктор приватный. Создать кость через `new Dice()` нельзя, получить ее можно только через `getInstance()`.
2. Экземпляр хранится в самом классе и создается лениво, при первом обращении. Пока кость никому не нужна, ее нет.
3. Со звездочкой заметим, что JavaScript выполняет наш код в одном потоке, поэтому ситуации, когда два потока одновременно создают кость и
   получают два разных экземпляра, здесь не возникает. Тему многопоточности мы здесь затрагивать не будем, но для самостоятельного изучения она очень интересна.

Преимущество одиночки очевидно - у всех составляющих игры (оружия, врагов, локаций) одна и та же кость, поэтому один сид задает все броски. 
А еще кость не нужно передавать через конструкторы каждого предмета и каждого врага - к ней можно
обратиться откуда угодно.

Но тут есть и обратная сторона: одиночка - это глобальное состояние. Лук, посох и враги зависят от игральной кости, однако
по их конструкторам этого не видно, а подменить кость на другую (например, в тестах) не получится. Поэтому применять
одиночку стоит только там, где объект действительно должен быть единственным.

Также давайте сразу заведем исключение, которое будем бросать, если по коду ничего не нашлось (неизвестная локация,
неизвестный предмет и т.д.), в будущем оно нам пригодится:

```typescript
export class UnknownCodeError extends Error {

    constructor(what: string, code: string) {
        super(`${what} с кодом '${code}' не существует`);
    }
}
```

## Боевая механика

Давайте определим базовые абстракции нашей игры.

Интерфейс сущности, которая умеет атаковать и получать урон - так с точки зрения механики выглядят и герой, и враги:

```typescript
import { Hit } from "./hit";

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
    receiveHit(damage: number): number;
}
```

Свойства интерфейса помечены `readonly`: снаружи их можно только читать. В классах они реализованы обычными полями или
геттерами (`get health()`).

Удар представляет собой вот такую небольшую сущность:

```typescript
// удар: сколько урона и был ли он критическим
export interface Hit {
    readonly damage: number;
    readonly critical: boolean;
}
```

`Hit` - это просто описание формы объекта (`interface`): заводить класс для таких данных в TypeScript не нужно, объект
создается литералом `{ damage: 10, critical: false }`. Стороны боя, типы событий и исход описаны объединением строк
(`"hero" | "enemy"`): компилятор проверит, что других значений не бывает.

Давайте теперь опишем механику боя. Зафиксируем стороны, типы событий и исход:

```typescript
export type Side = "hero" | "enemy";
```

```typescript
export type EventType = "hit" | "critical-hit" | "death";
```

```typescript
export type Outcome = "victory" | "defeat";
```

Бой делится на раунды. Каждое событие раунда (удар, критический удар, гибель) запишем вот так - тут и пригодятся наши объединения строк:

```typescript
import { EventType } from "./event-type";
import { Side } from "./side";

export interface BattleEvent {
    readonly round: number;
    readonly type: EventType;
    readonly source: Side;
    readonly target: Side;
    readonly amount: number;
    readonly targetHealthAfter: number;
}
```

Чтобы в отчете было видно, кто сражался, заведем тип участника боя: в нем все, что мы выводим о герое и его противнике:

```typescript
import { Combatant } from "./combatant";

// участник боя в том виде, в котором он попадает в отчет
export interface Participant {
    readonly code: string;
    readonly name: string;
    readonly maxHealth: number;
}

export function participantOf(combatant: Combatant): Participant {
    return { code: combatant.code, name: combatant.name, maxHealth: combatant.maxHealth };
}
```

Супер! Теперь можем составить общий отчет о бое, в нем будут два участника (герой и его противник), исход боя, а также информация о событиях, 
получится что-то вот такое:

```typescript
import { BattleEvent } from "./battle-event";
import { Outcome } from "./outcome";
import { Participant } from "./participant";

// результат боя: кто сражался, чем все закончилось и что происходило по ходу дела
export interface BattleReport {
    readonly hero: Participant;
    readonly enemy: Participant;
    readonly outcome: Outcome;
    readonly events: readonly BattleEvent[];
}
```

А теперь наконец-то определим механику самого боя! В целом уже ясно, что идти она будет по раундам, 
в рамках которых герой и враг будут обмениваться ударами, давайте так и реализуем:

```typescript
import { BattleEvent } from "./battle-event";
import { BattleReport } from "./battle-report";
import { Combatant } from "./combatant";
import { EventType } from "./event-type";
import { participantOf } from "./participant";
import { Side } from "./side";

// бой идет по раундам: сначала бьет герой, потом враг, пока кто-то не погибнет
export class Battle {

    private readonly events: BattleEvent[] = [];
    private round = 0;

    constructor(private readonly hero: Combatant, private readonly enemy: Combatant) {
    }

    run(): BattleReport {
        while (this.hero.isAlive() && this.enemy.isAlive()) {
            this.round++;
            this.strike(this.hero, "hero", this.enemy, "enemy");
            if (this.enemy.isAlive()) {
                this.strike(this.enemy, "enemy", this.hero, "hero");
            }
        }

        return {
            hero: participantOf(this.hero),
            enemy: participantOf(this.enemy),
            outcome: this.hero.isAlive() ? "victory" : "defeat",
            events: [...this.events],
        };
    }

    private strike(attacker: Combatant, source: Side, defender: Combatant, target: Side): void {
        const hit = attacker.rollDamage();
        const dealt = defender.receiveHit(hit.damage);
        const type: EventType = hit.critical ? "critical-hit" : "hit";
        this.emit({ round: this.round, type, source, target, amount: dealt, targetHealthAfter: defender.health });

        if (!defender.isAlive()) {
            this.emit({ round: this.round, type: "death", source, target, amount: 0, targetHealthAfter: 0 });
        }
    }

    private emit(event: BattleEvent): void {
        this.events.push(event);
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

```typescript
// зачарование неизменяемо: поля readonly нельзя поменять в коде, а Object.freeze запрещает менять объект
// и во время работы программы, поэтому одно и то же зачарование можно безопасно повесить на несколько предметов
export interface Enchantment {
    readonly code: string;
    readonly name: string;
    readonly bonusDamage: number;
    readonly bonusDefense: number;
    readonly bonusPrice: number;
}

export const SHARP: Enchantment = Object.freeze({ code: "sharp", name: "Острое", bonusDamage: 3, bonusDefense: 0, bonusPrice: 20 });
export const FIERY: Enchantment = Object.freeze({ code: "fiery", name: "Огненное", bonusDamage: 6, bonusDefense: 0, bonusPrice: 50 });
export const STURDY: Enchantment = Object.freeze({ code: "sturdy", name: "Прочное", bonusDamage: 0, bonusDefense: 0.05, bonusPrice: 20 });
export const WARDING: Enchantment = Object.freeze({ code: "warding", name: "Охранное", bonusDamage: 0, bonusDefense: 0.1, bonusPrice: 50 });

export const FOR_WEAPONS: readonly Enchantment[] = [SHARP, FIERY];
export const FOR_ARMOR: readonly Enchantment[] = [STURDY, WARDING];
```

В базовый класс предмета заложим и контракт `copy`:

```typescript
import { Enchantment } from "./enchantment";

export abstract class Item {

    private enchantmentList: Enchantment[] = [];

    protected constructor(
        // код шаблона, из которого получен предмет
        readonly code: string,
        readonly name: string,
        private readonly basePrice: number,
    ) {
    }

    // какие зачарования подходят этому предмету
    abstract get possibleEnchantments(): readonly Enchantment[];

    // описание характеристик предмета, например "урон 12"
    abstract get description(): string;

    // тип this означает, что копия меча - это меч, копия брони - броня и т.д.
    copy(): this {
        // Object.create создает объект с тем же прототипом, то есть того же класса, что и оригинал,
        // а Object.assign копирует в него все поля, поэтому переопределять copy наследникам не нужно
        const clone: this = Object.assign(Object.create(Object.getPrototypeOf(this)), this);
        // но копия поверхностная: поля копии ссылаются на те же объекты, что и у оригинала,
        // поэтому массив зачарований заменяем новым, иначе копия и оригинал делили бы его
        clone.enchantmentList = [...this.enchantmentList];
        return clone;
    }

    get price(): number {
        return this.basePrice + this.enchantmentList.reduce((sum, enchantment) => sum + enchantment.bonusPrice, 0);
    }

    get enchantments(): readonly Enchantment[] {
        return [...this.enchantmentList];
    }

    enchant(enchantment: Enchantment): void {
        if (!this.possibleEnchantments.includes(enchantment)) {
            throw new Error(`Зачарование '${enchantment.name}' не подходит для предмета '${this.name}'`);
        }

        this.enchantmentList.push(enchantment);
    }
}
```

Обратите внимание на метод `copy`. `Object.create(Object.getPrototypeOf(this))` создает объект того же класса, что и
оригинал, а `Object.assign` копирует в него все поля. Но копия получается поверхностной: поля копии ссылаются на те же
объекты, что и у оригинала. Если бы мы на этом остановились, копия и шаблон делили бы один массив зачарований, и
зачарование, наложенное на копию, появилось бы и на шаблоне в каталоге. Поэтому массив мы заменяем новым. А вот сами
зачарования копировать не нужно - они неизменяемые.

Встроенная функция `structuredClone` здесь не подойдет: она копирует только данные и теряет класс объекта, у такой
копии не было бы методов.

Супер, общий контракт предметов понятен, давайте создадим контракт для оружия:

```typescript
import { Hit } from "../combat/hit";
import { Enchantment, FOR_WEAPONS } from "./enchantment";
import { Item } from "./item";

export abstract class Weapon extends Item {

    protected constructor(code: string, name: string, basePrice: number, private readonly baseDamage: number) {
        super(code, name, basePrice);
    }

    // урон с учетом зачарований
    get damage(): number {
        return this.baseDamage + this.enchantments.reduce((sum, enchantment) => sum + enchantment.bonusDamage, 0);
    }

    override get possibleEnchantments(): readonly Enchantment[] {
        return FOR_WEAPONS;
    }

    // как именно оружие наносит урон, решают наследники
    abstract rollDamage(): Hit;
}
```

Отлично, теперь наше первое оружие, меч. Его урон всегда одинаковый:

```typescript
import { Hit } from "../combat/hit";
import { Weapon } from "./weapon";

export class Sword extends Weapon {

    constructor(code: string, name: string, basePrice: number, damage: number) {
        super(code, name, basePrice, damage);
    }

    override get description(): string {
        return `урон ${this.damage}`;
    }

    override rollDamage(): Hit {
        return { damage: this.damage, critical: false };
    }
}
```

Лук же с некоторым шансом наносит критический, двойной урон:

```typescript
import { Hit } from "../combat/hit";
import { Dice } from "../core/dice";
import { Weapon } from "./weapon";

const CRITICAL_MULTIPLIER = 2;

export class Bow extends Weapon {

    constructor(code: string, name: string, basePrice: number, damage: number, private readonly criticalChance: number) {
        super(code, name, basePrice, damage);
    }

    override get description(): string {
        return `урон ${this.damage}, шанс крита ${Math.round(this.criticalChance * 100)}%`;
    }

    override rollDamage(): Hit {
        if (Dice.getInstance().chance(this.criticalChance)) {
            return { damage: this.damage * CRITICAL_MULTIPLIER, critical: true };
        }

        return { damage: this.damage, critical: false };
    }
}
```

Урон магического посоха каждый раз немного разный:

```typescript
import { Hit } from "../combat/hit";
import { Dice } from "../core/dice";
import { Weapon } from "./weapon";

// урон посоха непредсказуем: он может быть как больше, так и меньше базового
export class Staff extends Weapon {

    constructor(code: string, name: string, basePrice: number, damage: number, private readonly scatter: number) {
        super(code, name, basePrice, damage);
    }

    override get description(): string {
        return `урон ${this.minDamage}-${this.maxDamage}`;
    }

    override rollDamage(): Hit {
        return { damage: Dice.getInstance().roll(this.minDamage, this.maxDamage), critical: false };
    }

    private get minDamage(): number {
        return Math.round(this.damage * (1 - this.scatter));
    }

    private get maxDamage(): number {
        return Math.round(this.damage * (1 + this.scatter));
    }
}
```

Дальше давайте создадим класс для брони, он у нас будет общий для всех ее видов:

```typescript
import { Enchantment, FOR_ARMOR } from "./enchantment";
import { Item } from "./item";

// больше этой доли урона никакая броня не поглощает
const MAX_DEFENSE = 0.75;

// броня поглощает долю входящего урона
export class Armor extends Item {

    constructor(code: string, name: string, basePrice: number, private readonly baseDefense: number) {
        super(code, name, basePrice);
    }

    // доля поглощаемого урона с учетом зачарований
    get defense(): number {
        const total = this.baseDefense + this.enchantments.reduce((sum, enchantment) => sum + enchantment.bonusDefense, 0);
        return Math.min(total, MAX_DEFENSE);
    }

    override get possibleEnchantments(): readonly Enchantment[] {
        return FOR_ARMOR;
    }

    override get description(): string {
        return `защита ${Math.round(this.defense * 100)}%`;
    }
}
```

Наследникам переопределять `copy` не нужно: копия сама получается объектом нужного класса. Тип возвращаемого значения
`this` сообщает компилятору то же самое: копия меча - это меч, копия брони - броня. Переопределять `copy` пришлось бы,
только если бы у наследника появились собственные изменяемые поля.

Теперь давайте напишем каталог: он хранит шаблоны всех предметов игры и выдает по ним конкретные экземпляры оружия и
брони:

```typescript
import { UnknownCodeError } from "../core/unknown-code-error";
import { Armor } from "./armor";
import { Item } from "./item";
import { Weapon } from "./weapon";

// каталог хранит шаблоны предметов, а наружу отдает только их копии
export class ItemCatalog {

    // Map запоминает порядок добавления
    private readonly prototypes = new Map<string, Item>();

    get codes(): string[] {
        return [...this.prototypes.keys()];
    }

    register(prototype: Item): void {
        this.prototypes.set(prototype.code, prototype);
    }

    // каталог не знает, какой конкретно класс у шаблона - копию создает сам шаблон
    create(code: string): Item {
        const prototype = this.prototypes.get(code);
        if (prototype === undefined) {
            throw new UnknownCodeError("Предмет", code);
        }

        return prototype.copy();
    }

    createWeapon(code: string): Weapon {
        const item = this.create(code);
        if (!(item instanceof Weapon)) {
            throw new Error(`Предмет '${code}' не является оружием`);
        }

        return item;
    }

    createArmor(code: string): Armor {
        const item = this.create(code);
        if (!(item instanceof Armor)) {
            throw new Error(`Предмет '${code}' не является броней`);
        }

        return item;
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

```typescript
import { Armor } from "./armor";
import { Bow } from "./bow";
import { ItemCatalog } from "./item-catalog";
import { Staff } from "./staff";
import { Sword } from "./sword";

// коды шаблонов: по ним предметы ищутся в каталоге
// стартовое снаряжение
export const TRAINING_SWORD = "training-sword";
export const HUNTING_BOW = "hunting-bow";
export const APPRENTICE_STAFF = "apprentice-staff";
export const CHAINMAIL = "chainmail";
export const LEATHER_ARMOR = "leather-armor";
export const LINEN_ROBE = "linen-robe";

// товары торговца
export const IRON_SWORD = "iron-sword";
export const ELVEN_BOW = "elven-bow";
export const EMBER_STAFF = "ember-staff";
export const PLATE_ARMOR = "plate-armor";

// добыча
export const GOBLIN_CLEAVER = "goblin-cleaver";
export const BONE_BOW = "bone-bow";
export const NECROMANCER_STAFF = "necromancer-staff";
export const KNIGHT_BLADE = "knight-blade";
export const CRYPT_MAIL = "crypt-mail";
export const DAWN_BLADE = "dawn-blade";
export const DRAGONSCALE_ARMOR = "dragonscale-armor";

// все предметы, которые есть в игре
export function createDefaultCatalog(): ItemCatalog {
    const catalog = new ItemCatalog();

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
```

## Стартовое снаряжение (Абстрактная фабрика / Abstract Factory)

По логике нашей игры мы хотим выдавать герою стартовое снаряжение в зависимости от его класса.
В целом это можно сделать простыми if'ами в коде основной программы, однако мы знаем, что в дальнейшем в игру будут
добавляться новые классы, поэтому здесь уместно будет применить шаблон *Абстрактная фабрика*.
Она позволяет создавать семейства связанных между собой объектов, не привязываясь к их конкретным классам. Семейство у
нас - это стартовое снаряжение класса.

В роли базовой фабрики выступит стартовый набор снаряжения:

```typescript
import { Armor } from "../item/armor";
import { Weapon } from "../item/weapon";

// Абстрактная фабрика: стартовый набор снаряжения
export interface StarterKitFactory {

    // возвращает некое оружие
    createWeapon(): Weapon;

    // возвращает некую броню
    createArmor(): Armor;
}
```

И для каждого из классов героя мы создадим свою реализацию этого набора: снаряжение каждого класса - это и есть
семейство объектов.

В наборе для воина будут лежать меч и кольчуга:

```typescript
import { Armor } from "../item/armor";
import { CHAINMAIL, TRAINING_SWORD } from "../item/default-catalog";
import { ItemCatalog } from "../item/item-catalog";
import { Weapon } from "../item/weapon";
import { StarterKitFactory } from "./starter-kit-factory";

export class WarriorKitFactory implements StarterKitFactory {

    constructor(private readonly catalog: ItemCatalog) {
    }

    createWeapon(): Weapon {
        return this.catalog.createWeapon(TRAINING_SWORD);
    }

    createArmor(): Armor {
        return this.catalog.createArmor(CHAINMAIL);
    }
}
```

В наборе для следопыта - лук и кожаная броня:

```typescript
import { Armor } from "../item/armor";
import { HUNTING_BOW, LEATHER_ARMOR } from "../item/default-catalog";
import { ItemCatalog } from "../item/item-catalog";
import { Weapon } from "../item/weapon";
import { StarterKitFactory } from "./starter-kit-factory";

export class RangerKitFactory implements StarterKitFactory {

    constructor(private readonly catalog: ItemCatalog) {
    }

    createWeapon(): Weapon {
        return this.catalog.createWeapon(HUNTING_BOW);
    }

    createArmor(): Armor {
        return this.catalog.createArmor(LEATHER_ARMOR);
    }
}
```

Ну а для мага - посох и роба:

```typescript
import { Armor } from "../item/armor";
import { APPRENTICE_STAFF, LINEN_ROBE } from "../item/default-catalog";
import { ItemCatalog } from "../item/item-catalog";
import { Weapon } from "../item/weapon";
import { StarterKitFactory } from "./starter-kit-factory";

export class MageKitFactory implements StarterKitFactory {

    constructor(private readonly catalog: ItemCatalog) {
    }

    createWeapon(): Weapon {
        return this.catalog.createWeapon(APPRENTICE_STAFF);
    }

    createArmor(): Armor {
        return this.catalog.createArmor(LINEN_ROBE);
    }
}
```

Перечисление (`enum`) в этом языке не может хранить данные, поэтому классы героя - это набор заранее созданных объектов
класса `HeroClass` с приватным конструктором. Каждый класс будет сам знать свою фабрику стартового снаряжения:

```typescript
import { UnknownCodeError } from "../core/unknown-code-error";
import { ItemCatalog } from "../item/item-catalog";
import { MageKitFactory } from "../kit/mage-kit-factory";
import { RangerKitFactory } from "../kit/ranger-kit-factory";
import { StarterKitFactory } from "../kit/starter-kit-factory";
import { WarriorKitFactory } from "../kit/warrior-kit-factory";

// перечисление (enum) в TypeScript не может хранить данные и методы, поэтому классы героя - это
// набор заранее созданных объектов, а создать новые снаружи нельзя: конструктор приватный
export class HeroClass {

    // стрелочная функция, которая создает фабрику стартового снаряжения
    static readonly WARRIOR = new HeroClass("warrior", "Воин", 120, catalog => new WarriorKitFactory(catalog));
    static readonly RANGER = new HeroClass("ranger", "Следопыт", 100, catalog => new RangerKitFactory(catalog));
    static readonly MAGE = new HeroClass("mage", "Маг", 85, catalog => new MageKitFactory(catalog));

    // все классы героя по порядку
    static readonly values: readonly HeroClass[] = [HeroClass.WARRIOR, HeroClass.RANGER, HeroClass.MAGE];

    private constructor(
        readonly code: string,
        readonly displayName: string,
        readonly baseHealth: number,
        private readonly starterKit: (catalog: ItemCatalog) => StarterKitFactory,
    ) {
    }

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

```typescript
import { Combatant } from "../combat/combatant";
import { Hit } from "../combat/hit";
import { Armor } from "../item/armor";
import { Item } from "../item/item";
import { Weapon } from "../item/weapon";
import { StarterKitFactory } from "../kit/starter-kit-factory";
import { HeroClass } from "./hero-class";

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
}

export class Hero implements Combatant {

    // внутренний класс билдер - чтобы удобно и безопасно создавать героев
    static readonly Builder = class {

        private name: string | undefined;
        private heroClass: HeroClass | undefined;
        private weapon: Weapon | undefined;
        private armor: Armor | undefined;
        // необязательные поля сразу получают значения по умолчанию
        private level = 1;
        private gold = STARTING_GOLD;

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

            return new Hero({
                name: this.name,
                heroClass: this.heroClass,
                weapon: this.weapon,
                armor: this.armor,
                level: this.level,
                gold: this.gold,
            });
        }
    };

    readonly name: string;
    readonly heroClass: HeroClass;
    private currentLevel: number;
    private currentExperience = 0;
    private currentGold: number;
    private currentHealth: number;
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
        this.currentHealth = this.maxHealth;
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
        return [...this.inventoryList];
    }

    rollDamage(): Hit {
        const hit = this.currentWeapon.rollDamage();
        return { damage: hit.damage + (this.currentLevel - 1) * DAMAGE_PER_LEVEL, critical: hit.critical };
    }

    receiveHit(damage: number): number {
        const reducedDamage = Math.round(damage * (1 - this.currentArmor.defense));
        // хотя бы единицу урона броня пропускает всегда
        const dealt = Math.max(1, reducedDamage);
        this.currentHealth = Math.max(0, this.currentHealth - dealt);
        return dealt;
    }

    isAlive(): boolean {
        return this.currentHealth > 0;
    }

    // поведение героя между экспедициями

    // перед каждой экспедицией герой успевает отдохнуть
    restoreHealth(): void {
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
}
```

Строитель объявлен как статическое поле `Hero.Builder`, в котором лежит класс: так он находится внутри класса героя и
может вызвать его приватный конструктор. Методы строителя возвращают тип `this`, поэтому вызовы можно выстраивать
цепочкой. Часто в TypeScript обходятся без строителя и передают в конструктор объект с параметрами
(`new Hero({ name, level })`). Строитель же позволяет добавить снаряжение отдельным шагом и проверить все поля в одном
месте, в `build()`.

Что в реализации строителя важно:

1. Конструктор героя приватный, его вызывает только строитель, поэтому создать героя в обход проверок нельзя.
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

```typescript
import { Combatant } from "../combat/combatant";

export interface Enemy extends Combatant {

    readonly experienceReward: number;

    rollGoldReward(): number;
}
```

Общая логика врагов вынесена в абстрактный класс:

```typescript
import { Hit } from "../combat/hit";
import { Dice } from "../core/dice";
import { Enemy } from "./enemy";

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
        return { damage: Dice.getInstance().roll(this.minDamage, this.maxDamage), critical: false };
    }

    receiveHit(damage: number): number {
        const dealt = Math.max(1, this.reduceDamage(damage));
        this.currentHealth = Math.max(0, this.currentHealth - dealt);
        return dealt;
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

Определим злого гоблина, самого слабого врага, живущего в лесу:

```typescript
import { AbstractEnemy } from "./abstract-enemy";

// гоблин - самый простой враг, без особенностей
export class Goblin extends AbstractEnemy {

    constructor() {
        super("goblin", "Гоблин", 50, 6, 14, 40, 30);
    }
}
```

Теперь скелет, он уже чуть страшнее гоблина:

```typescript
import { AbstractEnemy } from "./abstract-enemy";

const RESISTANCE = 0.3;

// скелет: кости плохо поддаются ударам, поэтому часть урона проходит мимо
export class Skeleton extends AbstractEnemy {

    constructor() {
        super("skeleton", "Скелет", 110, 10, 24, 90, 70);
    }

    protected override reduceDamage(damage: number): number {
        return Math.round(damage * (1 - RESISTANCE));
    }
}
```

И ужасный дракон: он наносит страшный урон огнем, а крепкая чешуя гасит часть урона:

```typescript
import { Hit } from "../combat/hit";
import { Dice } from "../core/dice";
import { AbstractEnemy } from "./abstract-enemy";

const RESISTANCE = 0.25;
const FIRE_BREATH_CHANCE = 0.25;
const FIRE_BREATH_MULTIPLIER = 2;

// дракон защищен чешуей и время от времени дышит огнем
export class Dragon extends AbstractEnemy {

    constructor() {
        super("dragon", "Дракон", 260, 16, 22, 500, 400);
    }

    override rollDamage(): Hit {
        const bite = super.rollDamage();
        if (Dice.getInstance().chance(FIRE_BREATH_CHANCE)) {
            return { damage: bite.damage * FIRE_BREATH_MULTIPLIER, critical: true };
        }

        return bite;
    }

    protected override reduceDamage(damage: number): number {
        return Math.round(damage * (1 - RESISTANCE));
    }
}
```

По итогам экспедиции герой получает награды: опыт, золото, добычу и новые уровни. Запишем их так:

```typescript
import { Item } from "../item/item";

// итоги экспедиции для героя;
// gold отрицательный, если герой проиграл и потерял золото, loot равен null, если добычи нет
export interface Rewards {
    readonly experience: number;
    readonly gold: number;
    readonly loot: Item | null;
    readonly levelsGained: number;
}
```

Ну а общий итог экспедиции фиксируем вот так:

```typescript
import { BattleReport } from "../combat/battle-report";
import { Rewards } from "./rewards";

// отчет об экспедиции: с каким сидом, куда, как прошел бой и что получил герой
export interface ExpeditionReport {
    readonly seed: number;
    readonly locationCode: string;
    readonly battle: BattleReport;
    readonly rewards: Rewards;
}
```

Дальше давайте определим абстрактную локацию и общую логику экспедиции. Конкретных врагов здесь не создаем - это работа
подклассов:

```typescript
import { Battle } from "../combat/battle";
import { Dice } from "../core/dice";
import { Enemy } from "../enemy/enemy";
import { Hero } from "../hero/hero";
import { Item } from "../item/item";
import { ItemCatalog } from "../item/item-catalog";
import { ExpeditionReport } from "./expedition-report";
import { Rewards } from "./rewards";

const LOOT_CHANCE = 0.6;
const ENCHANT_CHANCE = 0.3;

export abstract class Location {

    protected constructor(
        readonly code: string,
        readonly displayName: string,
        readonly recommendedLevel: number,
        private readonly catalog: ItemCatalog,
    ) {
    }

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

        hero.restoreHealth();
        const enemy = this.createEnemy();
        const battle = new Battle(hero, enemy).run();

        const rewards = battle.outcome === "victory" ? this.reward(hero, enemy, dice) : this.punish(hero);

        return { seed, locationCode: this.code, battle, rewards };
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

        return { experience, gold, loot, levelsGained };
    }

    private rollLoot(dice: Dice): Item | null {
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
        return { experience: 0, gold: -lostGold, loot: null, levelsGained: 0 };
    }
}
```

Также обратите внимание, что добыча здесь - копия шаблона из каталога, и зачарование накладывается именно на копию.

Отлично, теперь создаем конкретные локации:

Мистический лес, кишащий злыми, но слабыми гоблинами:

```typescript
import { Enemy } from "../enemy/enemy";
import { Goblin } from "../enemy/goblin";
import { GOBLIN_CLEAVER, HUNTING_BOW, LEATHER_ARMOR } from "../item/default-catalog";
import { ItemCatalog } from "../item/item-catalog";
import { Location } from "./location";

export class Forest extends Location {

    static readonly CODE = "forest";

    constructor(catalog: ItemCatalog) {
        super(Forest.CODE, "Мистический лес", 1, catalog);
    }

    protected override get lootTable(): readonly string[] {
        return [GOBLIN_CLEAVER, HUNTING_BOW, LEATHER_ARMOR];
    }

    protected override createEnemy(): Enemy {
        return new Goblin();
    }
}
```

Заброшенный склеп, в котором бродят ожившие скелеты:

```typescript
import { Enemy } from "../enemy/enemy";
import { Skeleton } from "../enemy/skeleton";
import { BONE_BOW, CRYPT_MAIL, KNIGHT_BLADE, NECROMANCER_STAFF } from "../item/default-catalog";
import { ItemCatalog } from "../item/item-catalog";
import { Location } from "./location";

export class Crypt extends Location {

    static readonly CODE = "crypt";

    constructor(catalog: ItemCatalog) {
        super(Crypt.CODE, "Заброшенный склеп", 3, catalog);
    }

    protected override get lootTable(): readonly string[] {
        return [KNIGHT_BLADE, BONE_BOW, NECROMANCER_STAFF, CRYPT_MAIL];
    }

    protected override createEnemy(): Enemy {
        return new Skeleton();
    }
}
```

И логово дракона, прибежище жуткого чешуйчатого чудища:

```typescript
import { Dragon } from "../enemy/dragon";
import { Enemy } from "../enemy/enemy";
import { DAWN_BLADE, DRAGONSCALE_ARMOR } from "../item/default-catalog";
import { ItemCatalog } from "../item/item-catalog";
import { Location } from "./location";

export class DragonLair extends Location {

    static readonly CODE = "dragon-lair";

    constructor(catalog: ItemCatalog) {
        super(DragonLair.CODE, "Логово дракона", 5, catalog);
    }

    protected override get lootTable(): readonly string[] {
        return [DAWN_BLADE, DRAGONSCALE_ARMOR];
    }

    protected override createEnemy(): Enemy {
        return new Dragon();
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

```typescript
import { UnknownCodeError } from "../core/unknown-code-error";
import { Location } from "./location";

// все локации игры
export class LocationRegistry {

    private readonly locations = new Map<string, Location>();

    constructor(locations: readonly Location[]) {
        for (const location of locations) {
            this.locations.set(location.code, location);
        }
    }

    get all(): Location[] {
        return [...this.locations.values()];
    }

    get(code: string): Location {
        const location = this.locations.get(code);
        if (location === undefined) {
            throw new UnknownCodeError("Локация", code);
        }

        return location;
    }
}
```

## Лагерь

Между экспедициями герой отдыхает в лагере, где есть торговец (а куда же без него). 
Торговец продает предметы из каталога и скупает ненужное за половину цены:

```typescript
import { UnknownCodeError } from "../core/unknown-code-error";
import { Hero } from "../hero/hero";
import { Item } from "../item/item";
import { ItemCatalog } from "../item/item-catalog";

const SELL_PRICE_PERCENT = 50;

export class Merchant {

    constructor(private readonly catalog: ItemCatalog, private readonly assortment: readonly string[]) {
    }

    // образцы товаров, чтобы показать их покупателю
    get showcase(): Item[] {
        return this.assortment.map(code => this.catalog.create(code));
    }

    buy(hero: Hero, code: string): Item {
        if (!this.assortment.includes(code)) {
            throw new UnknownCodeError("Товар", code);
        }

        // покупатель получает собственную копию предмета из каталога
        const item = this.catalog.create(code);
        hero.spendGold(item.price);
        hero.addItem(item);
        return item;
    }

    sell(hero: Hero, item: Item): number {
        const price = this.getSellPrice(item);
        hero.removeItem(item);
        hero.addGold(price);
        return price;
    }

    getSellPrice(item: Item): number {
        return Math.floor(item.price * SELL_PRICE_PERCENT / 100);
    }
}
```

## Консоль

Вся игровая логика наконец-то готова, осталось научить программу общаться с игроком. Весь вывод соберем в одном классе - он
превращает героя, предметы и отчеты об экспедиции в текст для консоли. 

Отдельный класс нужен потому, что дальше с игрой можно будет работать не только через консоль (спойлер-спойлер), и
привязывать к ней игровую логику не хочется.

Вот он:

```typescript
import { BattleEvent } from "./combat/battle-event";
import { BattleReport } from "./combat/battle-report";
import { Participant } from "./combat/participant";
import { Side } from "./combat/side";
import { Hero } from "./hero/hero";
import { Item } from "./item/item";
import { ExpeditionReport } from "./world/expedition-report";
import { Rewards } from "./world/rewards";

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
    }

    printReport(report: ExpeditionReport, locationName: string): void {
        const battle = report.battle;
        console.log(`=== ${locationName} (сид ${report.seed}) ===`);
        console.log(`${battle.hero.name} встречает врага: ${battle.enemy.name} (${battle.enemy.maxHealth} здоровья)`);

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
            case "death":
                console.log(`${target.name} повержен!`);
                break;
        }
    }

    private printRewards(battle: BattleReport, rewards: Rewards): void {
        if (battle.outcome === "defeat") {
            console.log(`Поражение... ${battle.hero.name} едва уносит ноги и теряет ${-rewards.gold} золота`);
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

Осталось написать игровой цикл - он будет в функции `main` (файл `main.ts`). Консоль в Node.js
читается асинхронно, поэтому функции, которые читают ввод, помечены `async`, а каждую строку ввода мы ждем через
`await`:

```typescript
import * as readline from "node:readline";
import { Merchant } from "./camp/merchant";
import { ConsoleView } from "./console-view";
import { Hero } from "./hero/hero";
import { HeroClass } from "./hero/hero-class";
import { createDefaultCatalog, CHAINMAIL, ELVEN_BOW, EMBER_STAFF, IRON_SWORD, PLATE_ARMOR } from "./item/default-catalog";
import { Item } from "./item/item";
import { ItemCatalog } from "./item/item-catalog";
import { Crypt } from "./world/crypt";
import { DragonLair } from "./world/dragon-lair";
import { Forest } from "./world/forest";
import { LocationRegistry } from "./world/location-registry";

const view = new ConsoleView();

// в Node.js консоль читается асинхронно: строки ввода приходят по одной, и мы ждем каждую через await
const lines = readline.createInterface({ input: process.stdin })[Symbol.asyncIterator]();

async function main(): Promise<void> {
    const catalog = createDefaultCatalog();
    const locations = new LocationRegistry([
        new Forest(catalog),
        new Crypt(catalog),
        new DragonLair(catalog),
    ]);
    const merchant = new Merchant(catalog, [IRON_SWORD, ELVEN_BOW, EMBER_STAFF, CHAINMAIL, PLATE_ARMOR]);

    const hero = await createHero(catalog);
    console.log(`${hero.name} разбивает лагерь на опушке мистического леса.`);

    while (true) {
        console.log();
        view.printHero(hero);
        console.log("1 - отправиться в экспедицию, 2 - снаряжение, 3 - торговец, 0 - выйти из игры");
        switch (await readLine()) {
            case "1":
                await goOnExpedition(hero, locations);
                break;
            case "2":
                await manageEquipment(hero);
                break;
            case "3":
                await visitMerchant(hero, merchant);
                break;
            case "0":
                process.exit(0);
            default:
                console.log("Такого варианта нет");
        }
    }
}

async function createHero(catalog: ItemCatalog): Promise<Hero> {
    console.log("Создайте своего героя!");
    while (true) {
        console.log("Выберите класс:");
        const heroClasses = HeroClass.values;
        heroClasses.forEach((heroClass, i) => {
            const kit = heroClass.getStarterKit(catalog);
            console.log(
                `${i + 1} - ${heroClass.displayName} (здоровье ${heroClass.baseHealth}): ` +
                `${view.describe(kit.createWeapon())}, ${view.describe(kit.createArmor())}`,
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
            return new Hero.Builder()
                .setName(name)
                .setHeroClass(heroClass)
                .setStarterKit(heroClass.getStarterKit(catalog))
                .build();
        } catch (error) {
            console.log("Не получилось создать героя: " + (error as Error).message);
        }
    }
}

async function goOnExpedition(hero: Hero, locations: LocationRegistry): Promise<void> {
    const all = locations.all;
    console.log("Куда отправимся?");
    all.forEach((location, i) => {
        console.log(`${i + 1} - ${location.displayName} (рекомендуемый уровень ${location.recommendedLevel})`);
    });
    const index = await readChoice(all.length);
    if (index < 0) {
        return;
    }

    const location = all[index];
    const report = location.explore(hero);
    console.log();
    view.printReport(report, location.displayName);

    if (report.battle.outcome === "victory" && location.code === DragonLair.CODE) {
        console.log(`\nДракон повержен! Барды еще долго будут петь о подвиге героя по имени ${hero.name}.`);
    }
}

async function manageEquipment(hero: Hero): Promise<void> {
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
    hero.equip(item);
    console.log("Экипировано: " + view.describe(item));
}

async function visitMerchant(hero: Hero, merchant: Merchant): Promise<void> {
    console.log(`Торговец: "Чего желаешь?" (золото: ${hero.gold})`);
    console.log("1 - купить, 2 - продать, 0 - назад");
    switch (await readLine()) {
        case "1":
            await buy(hero, merchant);
            break;
        case "2":
            await sell(hero, merchant);
            break;
    }
}

async function buy(hero: Hero, merchant: Merchant): Promise<void> {
    const showcase = merchant.showcase;
    showcase.forEach((item, i) => {
        console.log(`${i + 1} - ${view.describe(item)}: ${item.price} золота`);
    });
    const index = await readChoice(showcase.length);
    if (index < 0) {
        return;
    }

    try {
        const item = merchant.buy(hero, showcase[index].code);
        console.log("Куплено: " + view.describe(item));
    } catch (error) {
        console.log((error as Error).message);
    }
}

async function sell(hero: Hero, merchant: Merchant): Promise<void> {
    const inventory = hero.inventory;
    if (inventory.length === 0) {
        console.log("Продавать нечего (экипированные предметы продать нельзя)");
        return;
    }

    inventory.forEach((item, i) => {
        console.log(`${i + 1} - ${view.describe(item)}: ${merchant.getSellPrice(item)} золота`);
    });
    const index = await readChoice(inventory.length);
    if (index < 0) {
        return;
    }

    const item = inventory[index];
    const price = merchant.sell(hero, item);
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

Супер! Попробуйте запустить игру и победить дракона :)

В целом по коду программы видно, что мы почти не привязываемся к конкретным реализациям классов, а везде используем
абстракции (`Location`, а не конкретные локации для экспедиций, `StarterKitFactory`, а не конкретные наборы для
получения снаряжения, `Item`, а не конкретные предметы при копировании из каталога).

Таким образом в игру легко добавлять новые локации и врагов, классы героя со снаряжением и предметы.

> **_ДОП. ЗАДАНИЕ:_** Если хотите на практике в этом убедиться - попробуйте создать новую локацию и новый класс
> героя, выдав ему новое оружие и броню (не забудьте добавить их в каталог, а для новой локации - добычу!). Много ли мест в старом коде пришлось поменять?

## Демонстрация паттернов

Чтобы показать все паттерны одним запуском, напишем отдельную программу (файл `demo.ts`):

```typescript
import { ConsoleView } from "./console-view";
import { Dice } from "./core/dice";
import { Hero } from "./hero/hero";
import { HeroClass } from "./hero/hero-class";
import { createDefaultCatalog, IRON_SWORD } from "./item/default-catalog";
import { FIERY } from "./item/enchantment";
import { Crypt } from "./world/crypt";
import { Forest } from "./world/forest";

function main(): void {
    const catalog = createDefaultCatalog();
    const view = new ConsoleView();

    console.log("===== Одиночка =====");
    const first = Dice.getInstance();
    const second = Dice.getInstance();
    console.log(`Dice.getInstance() возвращает один и тот же объект: ${first === second}`);

    console.log();
    console.log("===== Прототип =====");
    const plainSword = catalog.create(IRON_SWORD);
    const fierySword = catalog.create(IRON_SWORD);
    fierySword.enchant(FIERY);
    console.log("Первая копия:        " + view.describe(plainSword));
    console.log("Вторая копия:        " + view.describe(fierySword));
    console.log("Новая копия шаблона: " + view.describe(catalog.create(IRON_SWORD)));
    console.log(`Копии - разные объекты: ${plainSword !== fierySword}`);

    console.log();
    console.log("===== Абстрактная фабрика =====");
    for (const heroClass of HeroClass.values) {
        const kit = heroClass.getStarterKit(catalog);
        console.log(`${heroClass.displayName}: ${view.describe(kit.createWeapon())}, ${view.describe(kit.createArmor())}`);
    }

    console.log();
    console.log("===== Строитель =====");
    const hero = new Hero.Builder()
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
    } catch (error) {
        console.log("Герой без имени не создается: " + (error as Error).message);
    }
    try {
        new Hero.Builder()
            .setName("Мерлин")
            .build();
    } catch (error) {
        console.log("Герой без класса не создается: " + (error as Error).message);
    }

    console.log();
    console.log("===== Фабричный метод =====");
    const forest = new Forest(catalog);
    const crypt = new Crypt(catalog);
    view.printReport(forest.explore(hero, 7), forest.displayName);
    console.log();
    view.printReport(crypt.explore(hero, 7), crypt.displayName);

}

main();
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
[раунд 1] Гоблин -> Артур: 4 урона, осталось 146/150
[раунд 2] Артур -> Гоблин: 18 урона, осталось 14/50
[раунд 2] Гоблин -> Артур: 4 урона, осталось 142/150
[раунд 3] Артур -> Гоблин: 18 урона, осталось 0/50
Гоблин повержен!
Победа! +40 опыта, +30 золота

=== Заброшенный склеп (сид 7) ===
Артур встречает врага: Скелет (110 здоровья)
[раунд 1] Артур -> Скелет: 13 урона, осталось 97/110
[раунд 1] Скелет -> Артур: 7 урона, осталось 143/150
[раунд 2] Артур -> Скелет: 13 урона, осталось 84/110
[раунд 2] Скелет -> Артур: 7 урона, осталось 136/150
[раунд 3] Артур -> Скелет: 13 урона, осталось 71/110
[раунд 3] Скелет -> Артур: 17 урона, осталось 119/150
[раунд 4] Артур -> Скелет: 13 урона, осталось 58/110
[раунд 4] Скелет -> Артур: 14 урона, осталось 105/150
[раунд 5] Артур -> Скелет: 13 урона, осталось 45/110
[раунд 5] Скелет -> Артур: 12 урона, осталось 93/150
[раунд 6] Артур -> Скелет: 13 урона, осталось 32/110
[раунд 6] Скелет -> Артур: 11 урона, осталось 82/150
[раунд 7] Артур -> Скелет: 13 урона, осталось 19/110
[раунд 7] Скелет -> Артур: 11 урона, осталось 71/150
[раунд 8] Артур -> Скелет: 13 урона, осталось 6/110
[раунд 8] Скелет -> Артур: 9 урона, осталось 62/150
[раунд 9] Артур -> Скелет: 13 урона, осталось 0/110
Скелет повержен!
Победа! +90 опыта, +54 золота
```

Не забудьте закоммитить и запушить изменения!

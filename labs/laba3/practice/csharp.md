# Лаба 3 — практика на C#

Это практическая часть [лабораторной работы 3](../laba3.md): требования к сдаче и вопросы к защите находятся в
основном тексте лабы, а здесь шаг за шагом разобрано эталонное решение.

Понадобится .NET SDK 10 или новее, сторонние библиотеки не нужны. Шаблон репозитория: [tech-prog-labs-2026-csharp](https://github.com/tidbid-kt/tech-prog-labs-2026-csharp). Код лабы пишите в проекте `labs/lab3`.

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

```csharp
namespace Lab3.Core;

// Одиночка: общий источник случайности на всю игру
public sealed class Dice
{
    // единственный экземпляр класса
    private static Dice? instance;

    private Random random = new();

    // конструктор приватный - самостоятельно создать экземпляр класса нельзя
    private Dice()
    {
    }

    // свойство для получения одиночки: оператор ??= создаст экземпляр только при первом обращении
    public static Dice Instance => instance ??= new Dice();

    // после установки сида все последующие броски повторяются в точности
    public void Reseed(int seed)
    {
        random = new Random(seed);
    }

    // случайный сид
    public int NextSeed()
    {
        return random.Next(1_000_000);
    }

    // случайное целое число от min до max включительно
    public int Roll(int min, int max)
    {
        return random.Next(min, max + 1);
    }

    // возвращает true с вероятностью probability (от 0 до 1)
    public bool Chance(double probability)
    {
        return random.NextDouble() < probability;
    }

    // случайный элемент списка
    public T Pick<T>(IReadOnlyList<T> options)
    {
        return options[random.Next(options.Count)];
    }
}
```

Вместо метода `getInstance()` в C# принято делать статическое свойство `Instance`. Оператор `??=` присваивает значение,
только если переменная равна `null`, поэтому экземпляр создается при первом обращении к свойству.

Давайте разберем, что в этой реализации важно.

1. Конструктор приватный. Создать кость через `new Dice()` нельзя, получить ее можно только через `Dice.Instance`.
2. Экземпляр хранится в самом классе и создается лениво, при первом обращении. Пока кость никому не нужна, ее нет.
3. Со звездочкой заметим, что наша реализация не потокобезопасна: если два потока одновременно впервые обратятся к кости, могут создаться два
   объекта. В игре поток один, поэтому нам это не страшно, но в многопоточной программе понадобилась бы синхронизация
   (или создание экземпляра сразу, а не лениво).
   В C# для этого удобно использовать класс `Lazy<Dice>`, который потокобезопасно создает объект при первом обращении.
   Тему многопоточности мы здесь затрагивать не будем, но для самостоятельного изучения она очень интересна.

Преимущество одиночки очевидно - у всех составляющих игры (оружия, врагов, локаций) одна и та же кость, поэтому один сид задает все броски. 
А еще кость не нужно передавать через конструкторы каждого предмета и каждого врага - к ней можно
обратиться откуда угодно.

Но тут есть и обратная сторона: одиночка - это глобальное состояние. Лук, посох и враги зависят от игральной кости, однако
по их конструкторам этого не видно, а подменить кость на другую (например, в тестах) не получится. Поэтому применять
одиночку стоит только там, где объект действительно должен быть единственным.

Также давайте сразу заведем исключение, которое будем бросать, если по коду ничего не нашлось (неизвестная локация,
неизвестный предмет и т.д.), в будущем оно нам пригодится:

```csharp
namespace Lab3.Core;

public sealed class UnknownCodeException(string what, string code)
    : Exception($"{what} с кодом '{code}' не существует");
```

## Боевая механика

Давайте определим базовые абстракции нашей игры.

Интерфейс сущности, которая умеет атаковать и получать урон - так с точки зрения механики выглядят и герой, и враги:

```csharp
namespace Lab3.Combat;

// всё, что может участвовать в бою: и герой, и враги
public interface ICombatant
{
    // код вида сущности: класс героя или тип врага
    string Code { get; }

    string Name { get; }

    int Health { get; }

    int MaxHealth { get; }

    bool IsAlive { get; }

    // удар, который будет нанесен данной сущностью
    Hit RollDamage();

    // получение удара, возвращает урон, который прошел на самом деле
    int ReceiveHit(int damage);
}
```

Интерфейсы в C# по соглашению называют с буквы `I`: `ICombatant`. Простые значения (код, имя, здоровье) здесь не
методы, а свойства (`string Code { get; }`): снаружи их читают как поля, но поменять не могут.

Удар представляет собой вот такую небольшую сущность:

```csharp
namespace Lab3.Combat;

// удар: сколько урона и был ли он критическим
public readonly record struct Hit(int Damage, bool Critical);
```

`record` - это короткая запись класса, который сравнивается по значению: C# сам создаст для него конструктор,
свойства, а также `Equals`, `GetHashCode` и `ToString`. `record struct` - то же самое, но для структуры.

Давайте теперь опишем механику боя. Зафиксируем стороны, типы событий и исход:

```csharp
namespace Lab3.Combat;

public enum Side
{
    Hero,
    Enemy
}
```

```csharp
namespace Lab3.Combat;

public enum EventType
{
    Hit,
    CriticalHit,
    Death
}
```

```csharp
namespace Lab3.Combat;

public enum Outcome
{
    Victory,
    Defeat
}
```

Бой делится на раунды. Каждое событие раунда (удар, критический удар, гибель) запишем вот так - тут и пригодятся наши `enum`:

```csharp
namespace Lab3.Combat;

public sealed record BattleEvent(int Round, EventType Type, Side Source, Side Target, int Amount, int TargetHealthAfter);
```

Чтобы в отчете было видно, кто сражался, заведем класс участника боя: в нем все, что мы выводим о герое и его противнике:

```csharp
namespace Lab3.Combat;

// участник боя в том виде, в котором он попадает в отчет
public sealed record Participant(string Code, string Name, int MaxHealth)
{
    public static Participant Of(ICombatant combatant)
    {
        return new Participant(combatant.Code, combatant.Name, combatant.MaxHealth);
    }
}
```

Супер! Теперь можем составить общий отчет о бое, в нем будут два участника (герой и его противник), исход боя, а также информация о событиях, 
получится что-то вот такое:

```csharp
namespace Lab3.Combat;

// результат боя: кто сражался, чем все закончилось и что происходило по ходу дела
public sealed record BattleReport(Participant Hero, Participant Enemy, Outcome Outcome, IReadOnlyList<BattleEvent> Events);
```

А теперь наконец-то определим механику самого боя! В целом уже ясно, что идти она будет по раундам, 
в рамках которых герой и враг будут обмениваться ударами, давайте так и реализуем:

```csharp
namespace Lab3.Combat;

// бой идет по раундам: сначала бьет герой, потом враг, пока кто-то не погибнет
public sealed class Battle
{
    private readonly ICombatant hero;
    private readonly ICombatant enemy;
    private readonly List<BattleEvent> events = new();
    private int round;

    public Battle(ICombatant hero, ICombatant enemy)
    {
        this.hero = hero;
        this.enemy = enemy;
    }

    public BattleReport Run()
    {
        while (hero.IsAlive && enemy.IsAlive)
        {
            round++;
            Strike(hero, Side.Hero, enemy, Side.Enemy);
            if (enemy.IsAlive)
            {
                Strike(enemy, Side.Enemy, hero, Side.Hero);
            }
        }

        Outcome outcome = hero.IsAlive ? Outcome.Victory : Outcome.Defeat;
        return new BattleReport(Participant.Of(hero), Participant.Of(enemy), outcome, events.ToList());
    }

    private void Strike(ICombatant attacker, Side source, ICombatant defender, Side target)
    {
        Hit hit = attacker.RollDamage();
        int dealt = defender.ReceiveHit(hit.Damage);
        EventType type = hit.Critical ? EventType.CriticalHit : EventType.Hit;
        Emit(new BattleEvent(round, type, source, target, dealt, defender.Health));

        if (!defender.IsAlive)
        {
            Emit(new BattleEvent(round, EventType.Death, source, target, 0, 0));
        }
    }

    private void Emit(BattleEvent battleEvent)
    {
        events.Add(battleEvent);
    }
}
```

Обратите внимание: `Battle` работает только с интерфейсом `ICombatant`, ничего не зная о конкретных героях и
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

```csharp
namespace Lab3.Items;

// зачарование неизменяемо (record с init-свойствами нельзя поменять после создания),
// поэтому одно и то же зачарование можно безопасно повесить на несколько предметов
public sealed record Enchantment(string Code, string Name, int BonusDamage, double BonusDefense, int BonusPrice)
{
    public static readonly Enchantment Sharp = new("sharp", "Острое", 3, 0, 20);
    public static readonly Enchantment Fiery = new("fiery", "Огненное", 6, 0, 50);
    public static readonly Enchantment Sturdy = new("sturdy", "Прочное", 0, 0.05, 20);
    public static readonly Enchantment Warding = new("warding", "Охранное", 0, 0.10, 50);

    public static readonly IReadOnlyList<Enchantment> ForWeapons = [Sharp, Fiery];
    public static readonly IReadOnlyList<Enchantment> ForArmor = [Sturdy, Warding];
}
```

В базовый класс предмета заложим и контракт `Copy`:

```csharp
namespace Lab3.Items;

public abstract class Item
{
    private readonly List<Enchantment> enchantments;

    protected Item(string code, string name, int basePrice)
    {
        Code = code;
        Name = name;
        BasePrice = basePrice;
        enchantments = new List<Enchantment>();
    }

    // конструктор копирования, его вызывают наследники в методе Copy()
    protected Item(Item source)
    {
        Code = source.Code;
        Name = source.Name;
        BasePrice = source.BasePrice;
        // список копируем, иначе копия и оригинал делили бы один и тот же объект списка зачарований
        enchantments = new List<Enchantment>(source.enchantments);
    }

    // код шаблона, из которого получен предмет
    public string Code { get; }

    public string Name { get; }

    private int BasePrice { get; }

    public int Price => BasePrice + enchantments.Sum(enchantment => enchantment.BonusPrice);

    public IReadOnlyList<Enchantment> Enchantments => enchantments.AsReadOnly();

    // какие зачарования подходят этому предмету
    public abstract IReadOnlyList<Enchantment> PossibleEnchantments { get; }

    // описание характеристик предмета, например "урон 12"
    public abstract string Description { get; }

    // каждый наследник по-своему реализует копирование
    public abstract Item Copy();

    public void Enchant(Enchantment enchantment)
    {
        if (!PossibleEnchantments.Contains(enchantment))
        {
            throw new ArgumentException($"Зачарование '{enchantment.Name}' не подходит для предмета '{Name}'");
        }

        enchantments.Add(enchantment);
    }
}
```

Обратите внимание на конструктор копирования. Если бы мы написали `this.enchantments = source.enchantments`, то копия и
оригинал ссылались бы на один и тот же список (поверхностная копия), и зачарование, наложенное на копию, появилось бы и
на шаблоне в каталоге. Поэтому для копии мы создаем новый список. А вот сами зачарования копировать не нужно - они неизменяемые, и копия может ссылаться на те же объекты.

Супер, общий контракт предметов понятен, давайте создадим контракт для оружия:

```csharp
using Lab3.Combat;

namespace Lab3.Items;

public abstract class Weapon : Item
{
    private readonly int damage;

    protected Weapon(string code, string name, int basePrice, int damage)
        : base(code, name, basePrice)
    {
        this.damage = damage;
    }

    protected Weapon(Weapon source)
        : base(source)
    {
        damage = source.damage;
    }

    // урон с учетом зачарований
    public int Damage => damage + Enchantments.Sum(enchantment => enchantment.BonusDamage);

    public override IReadOnlyList<Enchantment> PossibleEnchantments => Enchantment.ForWeapons;

    // как именно оружие наносит урон, решают наследники
    public abstract Hit RollDamage();

    public abstract override Weapon Copy();
}
```

Отлично, теперь наше первое оружие, меч. Его урон всегда одинаковый:

```csharp
using Lab3.Combat;

namespace Lab3.Items;

public sealed class Sword : Weapon
{
    public Sword(string code, string name, int basePrice, int damage)
        : base(code, name, basePrice, damage)
    {
    }

    private Sword(Sword source)
        : base(source)
    {
    }

    public override string Description => $"урон {Damage}";

    public override Hit RollDamage()
    {
        return new Hit(Damage, false);
    }

    public override Sword Copy()
    {
        return new Sword(this);
    }
}
```

Лук же с некоторым шансом наносит критический, двойной урон:

```csharp
using Lab3.Combat;
using Lab3.Core;

namespace Lab3.Items;

public sealed class Bow : Weapon
{
    private const int CriticalMultiplier = 2;

    private readonly double criticalChance;

    public Bow(string code, string name, int basePrice, int damage, double criticalChance)
        : base(code, name, basePrice, damage)
    {
        this.criticalChance = criticalChance;
    }

    private Bow(Bow source)
        : base(source)
    {
        criticalChance = source.criticalChance;
    }

    public override string Description => $"урон {Damage}, шанс крита {Math.Round(criticalChance * 100)}%";

    public override Hit RollDamage()
    {
        if (Dice.Instance.Chance(criticalChance))
        {
            return new Hit(Damage * CriticalMultiplier, true);
        }

        return new Hit(Damage, false);
    }

    public override Bow Copy()
    {
        return new Bow(this);
    }
}
```

Урон магического посоха каждый раз немного разный:

```csharp
using Lab3.Combat;
using Lab3.Core;

namespace Lab3.Items;

// урон посоха непредсказуем: он может быть как больше, так и меньше базового
public sealed class Staff : Weapon
{
    private readonly double scatter;

    public Staff(string code, string name, int basePrice, int damage, double scatter)
        : base(code, name, basePrice, damage)
    {
        this.scatter = scatter;
    }

    private Staff(Staff source)
        : base(source)
    {
        scatter = source.scatter;
    }

    public override string Description => $"урон {MinDamage}-{MaxDamage}";

    private int MinDamage => (int)Math.Round(Damage * (1 - scatter), MidpointRounding.AwayFromZero);

    private int MaxDamage => (int)Math.Round(Damage * (1 + scatter), MidpointRounding.AwayFromZero);

    public override Hit RollDamage()
    {
        return new Hit(Dice.Instance.Roll(MinDamage, MaxDamage), false);
    }

    public override Staff Copy()
    {
        return new Staff(this);
    }
}
```

Дальше давайте создадим класс для брони, он у нас будет общий для всех ее видов:

```csharp
namespace Lab3.Items;

// броня поглощает долю входящего урона
public sealed class Armor : Item
{
    // больше этой доли урона никакая броня не поглощает
    private const double MaxDefense = 0.75;

    private readonly double defense;

    public Armor(string code, string name, int basePrice, double defense)
        : base(code, name, basePrice)
    {
        this.defense = defense;
    }

    private Armor(Armor source)
        : base(source)
    {
        defense = source.defense;
    }

    // доля поглощаемого урона с учетом зачарований
    public double Defense => Math.Min(defense + Enchantments.Sum(enchantment => enchantment.BonusDefense), MaxDefense);

    public override IReadOnlyList<Enchantment> PossibleEnchantments => Enchantment.ForArmor;

    public override string Description => $"защита {Math.Round(Defense * 100)}%";

    public override Armor Copy()
    {
        return new Armor(this);
    }
}
```

Каждый конкретный предмет (меч, лук, посох, броня) реализует `Copy()` через свой приватный конструктор копирования, который сначала вызывает
конструктор копирования родителя, а затем копирует собственные поля. 
Так каждый класс сам отвечает за то, чтобы скопировать все свои поля.

Обратите внимание, что `Sword.Copy()` возвращает `Sword`, а не `Item`: начиная с C# 9 переопределенный метод может
возвращать более конкретный тип, чем в базовом классе.

Теперь давайте напишем каталог: он хранит шаблоны всех предметов игры и выдает по ним конкретные экземпляры оружия и
брони:

```csharp
using Lab3.Core;

namespace Lab3.Items;

// каталог хранит шаблоны предметов, а наружу отдает только их копии
public sealed class ItemCatalog
{
    // шаблоны хранятся в порядке добавления, предметов немного, поэтому ищем простым перебором
    private readonly List<Item> prototypes = new();

    public IReadOnlyList<string> Codes => prototypes.Select(prototype => prototype.Code).ToList();

    public void Register(Item prototype)
    {
        prototypes.Add(prototype);
    }

    // каталог не знает, какой конкретно класс у шаблона - копию создает сам шаблон
    public Item Create(string code)
    {
        Item? prototype = prototypes.FirstOrDefault(candidate => candidate.Code == code);
        if (prototype is null)
        {
            throw new UnknownCodeException("Предмет", code);
        }

        return prototype.Copy();
    }

    public Weapon CreateWeapon(string code)
    {
        return Create(code) as Weapon
            ?? throw new ArgumentException($"Предмет '{code}' не является оружием");
    }

    public Armor CreateArmor(string code)
    {
        return Create(code) as Armor
            ?? throw new ArgumentException($"Предмет '{code}' не является броней");
    }
}
```

В методе `Create` и работает прототип: каталог вызывает `Copy()` у шаблона и получает копию, не зная, меч это,
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

```csharp
namespace Lab3.Items;

// все предметы, которые есть в игре
public static class DefaultCatalog
{
    // коды шаблонов: по ним предметы ищутся в каталоге
    // стартовое снаряжение
    public const string TrainingSword = "training-sword";
    public const string HuntingBow = "hunting-bow";
    public const string ApprenticeStaff = "apprentice-staff";
    public const string Chainmail = "chainmail";
    public const string LeatherArmor = "leather-armor";
    public const string LinenRobe = "linen-robe";

    // товары торговца
    public const string IronSword = "iron-sword";
    public const string ElvenBow = "elven-bow";
    public const string EmberStaff = "ember-staff";
    public const string PlateArmor = "plate-armor";

    // добыча
    public const string GoblinCleaver = "goblin-cleaver";
    public const string BoneBow = "bone-bow";
    public const string NecromancerStaff = "necromancer-staff";
    public const string KnightBlade = "knight-blade";
    public const string CryptMail = "crypt-mail";
    public const string DawnBlade = "dawn-blade";
    public const string DragonscaleArmor = "dragonscale-armor";

    public static ItemCatalog Create()
    {
        var catalog = new ItemCatalog();

        // стартовое снаряжение
        catalog.Register(new Sword(TrainingSword, "Тренировочный меч", 20, 12));
        catalog.Register(new Bow(HuntingBow, "Охотничий лук", 20, 10, 0.3));
        catalog.Register(new Staff(ApprenticeStaff, "Посох ученика", 20, 14, 0.3));
        catalog.Register(new Armor(Chainmail, "Кольчуга", 40, 0.3));
        catalog.Register(new Armor(LeatherArmor, "Кожаная броня", 30, 0.2));
        catalog.Register(new Armor(LinenRobe, "Льняная роба", 15, 0.1));

        // товары торговца
        catalog.Register(new Sword(IronSword, "Железный меч", 80, 17));
        catalog.Register(new Bow(ElvenBow, "Эльфийский лук", 90, 14, 0.4));
        catalog.Register(new Staff(EmberStaff, "Тлеющий посох", 90, 20, 0.3));
        catalog.Register(new Armor(PlateArmor, "Латы", 120, 0.4));

        // добыча
        catalog.Register(new Sword(GoblinCleaver, "Гоблинский тесак", 40, 15));
        catalog.Register(new Bow(BoneBow, "Костяной лук", 140, 17, 0.4));
        catalog.Register(new Staff(NecromancerStaff, "Посох некроманта", 150, 25, 0.35));
        catalog.Register(new Sword(KnightBlade, "Клинок рыцаря", 150, 23));
        catalog.Register(new Armor(CryptMail, "Кольчуга стража склепа", 160, 0.45));
        catalog.Register(new Sword(DawnBlade, "Клинок рассвета", 500, 35));
        catalog.Register(new Armor(DragonscaleArmor, "Драконья чешуя", 500, 0.55));

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

```csharp
using Lab3.Items;

namespace Lab3.Kits;

// Абстрактная фабрика: стартовый набор снаряжения
public interface IStarterKitFactory
{
    // возвращает некое оружие
    Weapon CreateWeapon();

    // возвращает некую броню
    Armor CreateArmor();
}
```

И для каждого из классов героя мы создадим свою реализацию этого набора: снаряжение каждого класса - это и есть
семейство объектов.

В наборе для воина будут лежать меч и кольчуга:

```csharp
using Lab3.Items;

namespace Lab3.Kits;

public sealed class WarriorKitFactory : IStarterKitFactory
{
    private readonly ItemCatalog catalog;

    public WarriorKitFactory(ItemCatalog catalog)
    {
        this.catalog = catalog;
    }

    public Weapon CreateWeapon()
    {
        return catalog.CreateWeapon(DefaultCatalog.TrainingSword);
    }

    public Armor CreateArmor()
    {
        return catalog.CreateArmor(DefaultCatalog.Chainmail);
    }
}
```

В наборе для следопыта - лук и кожаная броня:

```csharp
using Lab3.Items;

namespace Lab3.Kits;

public sealed class RangerKitFactory : IStarterKitFactory
{
    private readonly ItemCatalog catalog;

    public RangerKitFactory(ItemCatalog catalog)
    {
        this.catalog = catalog;
    }

    public Weapon CreateWeapon()
    {
        return catalog.CreateWeapon(DefaultCatalog.HuntingBow);
    }

    public Armor CreateArmor()
    {
        return catalog.CreateArmor(DefaultCatalog.LeatherArmor);
    }
}
```

Ну а для мага - посох и роба:

```csharp
using Lab3.Items;

namespace Lab3.Kits;

public sealed class MageKitFactory : IStarterKitFactory
{
    private readonly ItemCatalog catalog;

    public MageKitFactory(ItemCatalog catalog)
    {
        this.catalog = catalog;
    }

    public Weapon CreateWeapon()
    {
        return catalog.CreateWeapon(DefaultCatalog.ApprenticeStaff);
    }

    public Armor CreateArmor()
    {
        return catalog.CreateArmor(DefaultCatalog.LinenRobe);
    }
}
```

Перечисление (`enum`) в этом языке не может хранить данные, поэтому классы героя - это набор заранее созданных объектов
класса `HeroClass` с приватным конструктором. Каждый класс будет сам знать свою фабрику стартового снаряжения:

```csharp
using Lab3.Core;
using Lab3.Items;
using Lab3.Kits;

namespace Lab3.Heroes;

// перечисление (enum) в C# не может хранить данные, поэтому классы героя - это
// набор заранее созданных объектов, а создать новые снаружи нельзя: конструктор приватный
public sealed class HeroClass
{
    // лямбда - функция, которая создает фабрику стартового снаряжения
    public static readonly HeroClass Warrior = new("warrior", "Воин", 120, catalog => new WarriorKitFactory(catalog));
    public static readonly HeroClass Ranger = new("ranger", "Следопыт", 100, catalog => new RangerKitFactory(catalog));
    public static readonly HeroClass Mage = new("mage", "Маг", 85, catalog => new MageKitFactory(catalog));

    // все классы героя по порядку
    public static readonly IReadOnlyList<HeroClass> Values = [Warrior, Ranger, Mage];

    private readonly Func<ItemCatalog, IStarterKitFactory> starterKit;

    private HeroClass(string code, string displayName, int baseHealth, Func<ItemCatalog, IStarterKitFactory> starterKit)
    {
        Code = code;
        DisplayName = displayName;
        BaseHealth = baseHealth;
        this.starterKit = starterKit;
    }

    public string Code { get; }

    public string DisplayName { get; }

    public int BaseHealth { get; }

    public static HeroClass ByCode(string code)
    {
        return Values.FirstOrDefault(heroClass => heroClass.Code == code)
            ?? throw new UnknownCodeException("Класс героя", code);
    }

    // каждый класс сам знает свою фабрику стартового снаряжения
    public IStarterKitFactory GetStarterKit(ItemCatalog catalog)
    {
        return starterKit(catalog);
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

```csharp
using Lab3.Combat;
using Lab3.Items;
using Lab3.Kits;

namespace Lab3.Heroes;

public sealed class Hero : ICombatant
{
    private const int MaxNameLength = 20;
    private const int StartingGold = 50;
    private const int HealthPerLevel = 15;
    private const int DamagePerLevel = 3;
    private const int ExperiencePerLevel = 100;

    private readonly List<Item> inventory = new();

    // конструктор приватный и принимает в себя билдера, из которого и берет значения полей
    private Hero(Builder builder)
    {
        Name = builder.Name!;
        HeroClass = builder.HeroClass!;
        Level = builder.Level;
        Gold = builder.Gold;
        Weapon = builder.Weapon!;
        Armor = builder.Armor!;
        Health = MaxHealth;
    }

    // внутренний класс билдер - чтобы удобно и безопасно создавать героев
    public sealed class Builder
    {
        internal string? Name { get; private set; }

        internal HeroClass? HeroClass { get; private set; }

        internal Weapon? Weapon { get; private set; }

        internal Armor? Armor { get; private set; }

        // необязательные поля сразу получают значения по умолчанию
        internal int Level { get; private set; } = 1;

        internal int Gold { get; private set; } = StartingGold;

        public Builder SetName(string name)
        {
            Name = name;
            return this;
        }

        public Builder SetHeroClass(HeroClass heroClass)
        {
            HeroClass = heroClass;
            return this;
        }

        public Builder SetWeapon(Weapon weapon)
        {
            Weapon = weapon;
            return this;
        }

        public Builder SetArmor(Armor armor)
        {
            Armor = armor;
            return this;
        }

        // берет оружие и броню из стартового набора
        public Builder SetStarterKit(IStarterKitFactory starterKit)
        {
            Weapon = starterKit.CreateWeapon();
            Armor = starterKit.CreateArmor();
            return this;
        }

        public Builder SetLevel(int level)
        {
            Level = level;
            return this;
        }

        public Builder SetGold(int gold)
        {
            Gold = gold;
            return this;
        }

        // перед созданием героя проверяем, что все обязательные поля заполнены корректно
        public Hero Build()
        {
            if (string.IsNullOrWhiteSpace(Name))
            {
                throw new ArgumentException("У героя должно быть имя");
            }
            if (Name.Length > MaxNameLength)
            {
                throw new ArgumentException($"Имя героя не может быть длиннее {MaxNameLength} символов");
            }
            if (HeroClass is null)
            {
                throw new ArgumentException("Не выбран класс героя");
            }
            if (Weapon is null || Armor is null)
            {
                throw new ArgumentException("Герою нужны оружие и броня");
            }
            if (Level < 1)
            {
                throw new ArgumentException("Уровень героя не может быть меньше 1");
            }
            if (Gold < 0)
            {
                throw new ArgumentException("Золота не может быть меньше нуля");
            }

            return new Hero(this);
        }
    }

    // свойства героя: читать их можно снаружи, а менять - только самому герою

    public string Code => HeroClass.Code;

    public string Name { get; }

    public HeroClass HeroClass { get; }

    public int Level { get; private set; }

    public int Experience { get; private set; }

    public int ExperienceToNextLevel => Level * ExperiencePerLevel;

    public int Gold { get; private set; }

    public int Health { get; private set; }

    public int MaxHealth => HeroClass.BaseHealth + (Level - 1) * HealthPerLevel;

    public bool IsAlive => Health > 0;

    public Weapon Weapon { get; private set; }

    public Armor Armor { get; private set; }

    public IReadOnlyList<Item> Inventory => inventory.AsReadOnly();

    public Hit RollDamage()
    {
        Hit hit = Weapon.RollDamage();
        return hit with { Damage = hit.Damage + (Level - 1) * DamagePerLevel };
    }

    public int ReceiveHit(int damage)
    {
        int reducedDamage = (int)Math.Round(damage * (1 - Armor.Defense), MidpointRounding.AwayFromZero);
        // хотя бы единицу урона броня пропускает всегда
        int dealt = Math.Max(1, reducedDamage);
        Health = Math.Max(0, Health - dealt);
        return dealt;
    }

    // поведение героя между экспедициями

    // перед каждой экспедицией герой успевает отдохнуть
    public void RestoreHealth()
    {
        Health = MaxHealth;
    }

    // начисляет опыт, возвращает количество полученных уровней
    public int GainExperience(int amount)
    {
        Experience += amount;
        int levelsGained = 0;
        while (Experience >= ExperienceToNextLevel)
        {
            Experience -= ExperienceToNextLevel;
            Level++;
            levelsGained++;
        }

        return levelsGained;
    }

    public void AddGold(int amount)
    {
        Gold += amount;
    }

    public void SpendGold(int amount)
    {
        if (amount > Gold)
        {
            throw new InvalidOperationException($"Не хватает золота: нужно {amount}, есть {Gold}");
        }

        Gold -= amount;
    }

    public void AddItem(Item item)
    {
        inventory.Add(item);
    }

    public void RemoveItem(Item item)
    {
        if (!inventory.Remove(item))
        {
            throw new ArgumentException($"В инвентаре нет предмета {item.Name}");
        }
    }

    // экипирует предмет из инвентаря, снятый предмет возвращается в инвентарь
    public void Equip(Item item)
    {
        switch (item)
        {
            case Weapon newWeapon:
                RemoveItem(newWeapon);
                inventory.Add(Weapon);
                Weapon = newWeapon;
                break;
            case Armor newArmor:
                RemoveItem(newArmor);
                inventory.Add(Armor);
                Armor = newArmor;
                break;
            default:
                throw new ArgumentException($"Предмет нельзя экипировать: {item.Name}");
        }
    }
}
```

В C# объекты часто создают через инициализатор (`new Hero { Name = ..., Level = ... }`), а обязательные свойства
помечают словом `required`. Но инициализатор заполняет свойства уже созданного объекта, так что проверить все поля
вместе до его появления не выйдет - это и берет на себя строитель.

Что в реализации строителя важно:

1. Конструктор героя приватный и принимает только строителя, поэтому создать героя в обход проверок нельзя.
2. Каждый метод строителя возвращает сам строитель, поэтому вызовы можно выстраивать цепочкой.
3. Все проверки собраны в методе `Build`: если обязательное поле не заполнено или заполнено неправильно, герой
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

```csharp
using Lab3.Combat;

namespace Lab3.Enemies;

public interface IEnemy : ICombatant
{
    int ExperienceReward { get; }

    int RollGoldReward();
}
```

Общая логика врагов вынесена в абстрактный класс:

```csharp
using Lab3.Combat;
using Lab3.Core;

namespace Lab3.Enemies;

// общая логика для врагов: здоровье, урон в диапазоне, награда
public abstract class AbstractEnemy : IEnemy
{
    private readonly int minDamage;
    private readonly int maxDamage;
    private readonly int goldReward;

    protected AbstractEnemy(
        string code, string name, int maxHealth,
        int minDamage, int maxDamage,
        int experienceReward, int goldReward)
    {
        Code = code;
        Name = name;
        MaxHealth = maxHealth;
        Health = maxHealth;
        ExperienceReward = experienceReward;
        this.minDamage = minDamage;
        this.maxDamage = maxDamage;
        this.goldReward = goldReward;
    }

    public string Code { get; }

    public string Name { get; }

    public int Health { get; private set; }

    public int MaxHealth { get; }

    public bool IsAlive => Health > 0;

    public int ExperienceReward { get; }

    // virtual - наследники могут переопределить этот метод
    public virtual Hit RollDamage()
    {
        return new Hit(Dice.Instance.Roll(minDamage, maxDamage), false);
    }

    public int ReceiveHit(int damage)
    {
        int dealt = Math.Max(1, ReduceDamage(damage));
        Health = Math.Max(0, Health - dealt);
        return dealt;
    }

    public int RollGoldReward()
    {
        return Dice.Instance.Roll(goldReward / 2, goldReward);
    }

    // наследники могут уменьшать входящий урон
    protected virtual int ReduceDamage(int damage)
    {
        return damage;
    }
}
```

Определим злого гоблина, самого слабого врага, живущего в лесу:

```csharp
namespace Lab3.Enemies;

// гоблин - самый простой враг, без особенностей
public sealed class Goblin : AbstractEnemy
{
    public Goblin()
        : base("goblin", "Гоблин", 50, 6, 14, 40, 30)
    {
    }
}
```

Теперь скелет, он уже чуть страшнее гоблина:

```csharp
namespace Lab3.Enemies;

// скелет: кости плохо поддаются ударам, поэтому часть урона проходит мимо
public sealed class Skeleton : AbstractEnemy
{
    private const double Resistance = 0.3;

    public Skeleton()
        : base("skeleton", "Скелет", 110, 10, 24, 90, 70)
    {
    }

    protected override int ReduceDamage(int damage)
    {
        return (int)Math.Round(damage * (1 - Resistance), MidpointRounding.AwayFromZero);
    }
}
```

И ужасный дракон: он наносит страшный урон огнем, а крепкая чешуя гасит часть урона:

```csharp
using Lab3.Combat;
using Lab3.Core;

namespace Lab3.Enemies;

// дракон защищен чешуей и время от времени дышит огнем
public sealed class Dragon : AbstractEnemy
{
    private const double Resistance = 0.25;
    private const double FireBreathChance = 0.25;
    private const int FireBreathMultiplier = 2;

    public Dragon()
        : base("dragon", "Дракон", 260, 16, 22, 500, 400)
    {
    }

    public override Hit RollDamage()
    {
        Hit bite = base.RollDamage();
        if (Dice.Instance.Chance(FireBreathChance))
        {
            return new Hit(bite.Damage * FireBreathMultiplier, true);
        }

        return bite;
    }

    protected override int ReduceDamage(int damage)
    {
        return (int)Math.Round(damage * (1 - Resistance), MidpointRounding.AwayFromZero);
    }
}
```

По итогам экспедиции герой получает награды: опыт, золото, добычу и новые уровни. Запишем их так:

```csharp
using Lab3.Items;

namespace Lab3.World;

// итоги экспедиции для героя;
// Gold отрицательный, если герой проиграл и потерял золото, Loot равен null, если добычи нет
public sealed record Rewards(int Experience, int Gold, Item? Loot, int LevelsGained);
```

Ну а общий итог экспедиции фиксируем вот так:

```csharp
using Lab3.Combat;

namespace Lab3.World;

// отчет об экспедиции: с каким сидом, куда, как прошел бой и что получил герой
public sealed record ExpeditionReport(int Seed, string LocationCode, BattleReport Battle, Rewards Rewards);
```

Дальше давайте определим абстрактную локацию и общую логику экспедиции. Конкретных врагов здесь не создаем - это работа
подклассов:

```csharp
using Lab3.Combat;
using Lab3.Core;
using Lab3.Enemies;
using Lab3.Heroes;
using Lab3.Items;

namespace Lab3.World;

public abstract class Location
{
    private const double LootChance = 0.6;
    private const double EnchantChance = 0.3;

    private readonly ItemCatalog catalog;

    protected Location(string code, string displayName, int recommendedLevel, ItemCatalog catalog)
    {
        Code = code;
        DisplayName = displayName;
        RecommendedLevel = recommendedLevel;
        this.catalog = catalog;
    }

    public string Code { get; }

    public string DisplayName { get; }

    public int RecommendedLevel { get; }

    // коды предметов из каталога, которые можно найти в локации
    protected abstract IReadOnlyList<string> LootTable { get; }

    // создание врага делегируется подклассам для повышения гибкости
    protected abstract IEnemy CreateEnemy();

    public ExpeditionReport Explore(Hero hero)
    {
        return Explore(hero, Dice.Instance.NextSeed());
    }

    // бизнес-логика, общая для всех локаций:
    // герой встречает врага, сражается с ним и получает награду, если победил
    public ExpeditionReport Explore(Hero hero, int seed)
    {
        Dice dice = Dice.Instance;
        dice.Reseed(seed);

        hero.RestoreHealth();
        IEnemy enemy = CreateEnemy();
        BattleReport battle = new Battle(hero, enemy).Run();

        Rewards rewards = battle.Outcome == Outcome.Victory
            ? Reward(hero, enemy, dice)
            : Punish(hero);

        return new ExpeditionReport(seed, Code, battle, rewards);
    }

    private Rewards Reward(Hero hero, IEnemy enemy, Dice dice)
    {
        int experience = enemy.ExperienceReward;
        int gold = enemy.RollGoldReward();
        Item? loot = RollLoot(dice);

        int levelsGained = hero.GainExperience(experience);
        hero.AddGold(gold);
        if (loot is not null)
        {
            hero.AddItem(loot);
        }

        return new Rewards(experience, gold, loot, levelsGained);
    }

    private Item? RollLoot(Dice dice)
    {
        if (!dice.Chance(LootChance))
        {
            return null;
        }

        // добыча - это копия шаблона из каталога, ее можно менять, не боясь испортить шаблон
        Item loot = catalog.Create(dice.Pick(LootTable));
        if (dice.Chance(EnchantChance))
        {
            loot.Enchant(dice.Pick(loot.PossibleEnchantments));
        }

        return loot;
    }

    // проигравший герой теряет половину золота
    private static Rewards Punish(Hero hero)
    {
        int lostGold = hero.Gold / 2;
        hero.SpendGold(lostGold);
        return new Rewards(0, -lostGold, null, 0);
    }
}
```

Также обратите внимание, что добыча здесь - копия шаблона из каталога, и зачарование накладывается именно на копию.

Отлично, теперь создаем конкретные локации:

Мистический лес, кишащий злыми, но слабыми гоблинами:

```csharp
using Lab3.Enemies;
using Lab3.Items;

namespace Lab3.World;

public sealed class Forest : Location
{
    public const string LocationCode = "forest";

    public Forest(ItemCatalog catalog)
        : base(LocationCode, "Мистический лес", 1, catalog)
    {
    }

    protected override IReadOnlyList<string> LootTable { get; } =
    [
        DefaultCatalog.GoblinCleaver, DefaultCatalog.HuntingBow, DefaultCatalog.LeatherArmor,
    ];

    protected override IEnemy CreateEnemy()
    {
        return new Goblin();
    }
}
```

Заброшенный склеп, в котором бродят ожившие скелеты:

```csharp
using Lab3.Enemies;
using Lab3.Items;

namespace Lab3.World;

public sealed class Crypt : Location
{
    public const string LocationCode = "crypt";

    public Crypt(ItemCatalog catalog)
        : base(LocationCode, "Заброшенный склеп", 3, catalog)
    {
    }

    protected override IReadOnlyList<string> LootTable { get; } =
    [
        DefaultCatalog.KnightBlade, DefaultCatalog.BoneBow, DefaultCatalog.NecromancerStaff, DefaultCatalog.CryptMail,
    ];

    protected override IEnemy CreateEnemy()
    {
        return new Skeleton();
    }
}
```

И логово дракона, прибежище жуткого чешуйчатого чудища:

```csharp
using Lab3.Enemies;
using Lab3.Items;

namespace Lab3.World;

public sealed class DragonLair : Location
{
    public const string LocationCode = "dragon-lair";

    public DragonLair(ItemCatalog catalog)
        : base(LocationCode, "Логово дракона", 5, catalog)
    {
    }

    protected override IReadOnlyList<string> LootTable { get; } =
    [
        DefaultCatalog.DawnBlade, DefaultCatalog.DragonscaleArmor,
    ];

    protected override IEnemy CreateEnemy()
    {
        return new Dragon();
    }
}
```

Супер! Теперь если в игре появится еще одна локация, нам нужно будет лишь реализовать для нее фабричный метод в новом
классе, в остальном код экспедиций меняться не будет.

Сам фабричный метод объявлен в базовом классе абстрактным и возвращает абстракцию (`IEnemy`), а не конкретного врага.
Вызывает его базовый класс в общем алгоритме `Explore`, поэтому подклассы не переписывают сценарий экспедиции, а лишь
определяют, кого герой в ней встретит. В итоге сценарий написан один раз, а сам `Location` не зависит ни от гоблинов, ни
от скелетов, ни от драконов.

Минус в том, что на каждый вид продукта нужен свой подкласс создателя: десять видов врагов - десять классов локаций,
даже если ничем другим локации не отличаются. Применять фабричный метод стоит, когда у класса есть общий алгоритм, а
объект, который в нем создается, должен меняться в наследниках.

Кстати, метод `Explore` - это пример еще одного паттерна, *Шаблонного метода*: базовый класс задает шаги
алгоритма, а отдельные шаги переопределяются в наследниках. Фабричный метод часто встречается именно как один из
таких шагов.

Давайте сравним фабричный метод с абстрактной фабрикой из прошлого раздела. Абстрактная фабрика - это отдельный объект,
который передается тому, кому он нужен (композиция - мы кратко касались ее на прошлой лекции). 
Ее методы вызывает внешний код (в нашем случае строитель героя), и создает она целое семейство объектов. 

Фабричный метод - это метод самого класса, который переопределяется в наследниках, и создает он один продукт. 
Но главное отличие в том, что фабричным методом пользуется сам класс, в своей общей бизнес-логике (`Explore`). 
Снаружи этот фабричный метод никто не вызывает.

Чтобы всем этим пользоваться, осталось завести реестр локаций:

```csharp
using Lab3.Core;

namespace Lab3.World;

// все локации игры
public sealed class LocationRegistry
{
    private readonly List<Location> locations;

    public LocationRegistry(IEnumerable<Location> locations)
    {
        this.locations = locations.ToList();
    }

    public IReadOnlyList<Location> All => locations.AsReadOnly();

    public Location Get(string code)
    {
        return locations.FirstOrDefault(location => location.Code == code)
            ?? throw new UnknownCodeException("Локация", code);
    }
}
```

## Лагерь

Между экспедициями герой отдыхает в лагере, где есть торговец (а куда же без него). 
Торговец продает предметы из каталога и скупает ненужное за половину цены:

```csharp
using Lab3.Core;
using Lab3.Heroes;
using Lab3.Items;

namespace Lab3.Camp;

public sealed class Merchant
{
    private const int SellPricePercent = 50;

    private readonly ItemCatalog catalog;
    private readonly List<string> assortment;

    public Merchant(ItemCatalog catalog, IEnumerable<string> assortment)
    {
        this.catalog = catalog;
        this.assortment = assortment.ToList();
    }

    // образцы товаров, чтобы показать их покупателю
    public IReadOnlyList<Item> Showcase => assortment.Select(catalog.Create).ToList();

    public Item Buy(Hero hero, string code)
    {
        if (!assortment.Contains(code))
        {
            throw new UnknownCodeException("Товар", code);
        }

        // покупатель получает собственную копию предмета из каталога
        Item item = catalog.Create(code);
        hero.SpendGold(item.Price);
        hero.AddItem(item);
        return item;
    }

    public int Sell(Hero hero, Item item)
    {
        int price = GetSellPrice(item);
        hero.RemoveItem(item);
        hero.AddGold(price);
        return price;
    }

    public int GetSellPrice(Item item)
    {
        return item.Price * SellPricePercent / 100;
    }
}
```

## Консоль

Вся игровая логика наконец-то готова, осталось научить программу общаться с игроком. Весь вывод соберем в одном классе - он
превращает героя, предметы и отчеты об экспедиции в текст для консоли. 

Отдельный класс нужен потому, что дальше с игрой можно будет работать не только через консоль (спойлер-спойлер), и
привязывать к ней игровую логику не хочется.

Вот он:

```csharp
using Lab3.Combat;
using Lab3.Heroes;
using Lab3.Items;
using Lab3.World;

namespace Lab3;

// весь вывод в консоль собран здесь
public sealed class ConsoleView
{
    public string Describe(Item item)
    {
        string description = item.Name;

        if (item.Enchantments.Count > 0)
        {
            var names = new List<string>();
            foreach (Enchantment enchantment in item.Enchantments)
            {
                names.Add(enchantment.Name);
            }
            description += " [" + string.Join(", ", names) + "]";
        }

        return description + " (" + item.Description + ")";
    }

    public void PrintHero(Hero hero)
    {
        Console.WriteLine(
            $"{hero.Name}, {hero.HeroClass.DisplayName} {hero.Level} ур. | здоровье {hero.MaxHealth} | " +
            $"опыт {hero.Experience}/{hero.ExperienceToNextLevel} | золото {hero.Gold}");
        Console.WriteLine("  Оружие: " + Describe(hero.Weapon));
        Console.WriteLine("  Броня:  " + Describe(hero.Armor));
    }

    public void PrintReport(ExpeditionReport report, string locationName)
    {
        BattleReport battle = report.Battle;
        Console.WriteLine($"=== {locationName} (сид {report.Seed}) ===");
        Console.WriteLine($"{battle.Hero.Name} встречает врага: {battle.Enemy.Name} ({battle.Enemy.MaxHealth} здоровья)");

        foreach (BattleEvent battleEvent in battle.Events)
        {
            PrintEvent(battleEvent, battle);
        }

        PrintRewards(battle, report.Rewards);
    }

    private void PrintEvent(BattleEvent battleEvent, BattleReport battle)
    {
        Participant source = GetParticipant(battleEvent.Source, battle);
        Participant target = GetParticipant(battleEvent.Target, battle);

        switch (battleEvent.Type)
        {
            case EventType.Hit:
                Console.WriteLine(
                    $"[раунд {battleEvent.Round}] {source.Name} -> {target.Name}: {battleEvent.Amount} урона, " +
                    $"осталось {battleEvent.TargetHealthAfter}/{target.MaxHealth}");
                break;
            case EventType.CriticalHit:
                Console.WriteLine(
                    $"[раунд {battleEvent.Round}] {source.Name} -> {target.Name}: КРИТ! {battleEvent.Amount} урона, " +
                    $"осталось {battleEvent.TargetHealthAfter}/{target.MaxHealth}");
                break;
            case EventType.Death:
                Console.WriteLine($"{target.Name} повержен!");
                break;
        }
    }

    private void PrintRewards(BattleReport battle, Rewards rewards)
    {
        if (battle.Outcome == Outcome.Defeat)
        {
            Console.WriteLine($"Поражение... {battle.Hero.Name} едва уносит ноги и теряет {-rewards.Gold} золота");
            return;
        }

        Console.WriteLine($"Победа! +{rewards.Experience} опыта, +{rewards.Gold} золота");
        if (rewards.Loot is not null)
        {
            Console.WriteLine("Добыча: " + Describe(rewards.Loot));
        }
        if (rewards.LevelsGained > 0)
        {
            Console.WriteLine("Новый уровень!");
        }
    }

    private static Participant GetParticipant(Side side, BattleReport battle)
    {
        return side == Side.Hero ? battle.Hero : battle.Enemy;
    }
}
```

Осталось написать игровой цикл. Запускать программу будет точка входа `Program.cs`, а сама игра
находится в классе `Game`:

```csharp
using System.Text;
using Lab3;

Console.OutputEncoding = Encoding.UTF8;
Console.InputEncoding = Encoding.UTF8;

// точка входа: без аргументов запускается игра, с аргументом demo - демонстрация паттернов
if (args.Length > 0 && args[0] == "demo")
{
    Demo.Run();
}
else
{
    Game.Run();
}
```
```csharp
using Lab3.Camp;
using Lab3.Combat;
using Lab3.Heroes;
using Lab3.Items;
using Lab3.Kits;
using Lab3.World;

namespace Lab3;

public static class Game
{
    private static readonly ConsoleView View = new();

    public static void Run()
    {
        ItemCatalog catalog = DefaultCatalog.Create();
        var locations = new LocationRegistry([
            new Forest(catalog),
            new Crypt(catalog),
            new DragonLair(catalog),
        ]);
        var merchant = new Merchant(catalog, [
            DefaultCatalog.IronSword, DefaultCatalog.ElvenBow, DefaultCatalog.EmberStaff,
            DefaultCatalog.Chainmail, DefaultCatalog.PlateArmor,
        ]);

        Hero hero = CreateHero(catalog);
        Console.WriteLine($"{hero.Name} разбивает лагерь на опушке мистического леса.");

        while (true)
        {
            Console.WriteLine();
            View.PrintHero(hero);
            Console.WriteLine("1 - отправиться в экспедицию, 2 - снаряжение, 3 - торговец, 0 - выйти из игры");
            switch (ReadLine())
            {
                case "1":
                    GoOnExpedition(hero, locations);
                    break;
                case "2":
                    ManageEquipment(hero);
                    break;
                case "3":
                    VisitMerchant(hero, merchant);
                    break;
                case "0":
                    return;
                default:
                    Console.WriteLine("Такого варианта нет");
                    break;
            }
        }
    }

    private static Hero CreateHero(ItemCatalog catalog)
    {
        Console.WriteLine("Создайте своего героя!");
        while (true)
        {
            Console.WriteLine("Выберите класс:");
            IReadOnlyList<HeroClass> heroClasses = HeroClass.Values;
            for (int i = 0; i < heroClasses.Count; i++)
            {
                IStarterKitFactory kit = heroClasses[i].GetStarterKit(catalog);
                Console.WriteLine(
                    $"{i + 1} - {heroClasses[i].DisplayName} (здоровье {heroClasses[i].BaseHealth}): " +
                    $"{View.Describe(kit.CreateWeapon())}, {View.Describe(kit.CreateArmor())}");
            }
            int classIndex = ReadChoice(heroClasses.Count);
            if (classIndex < 0)
            {
                continue;
            }
            HeroClass heroClass = heroClasses[classIndex];

            Console.WriteLine("Введите имя:");
            string name = ReadLine();

            try
            {
                return new Hero.Builder()
                    .SetName(name)
                    .SetHeroClass(heroClass)
                    .SetStarterKit(heroClass.GetStarterKit(catalog))
                    .Build();
            }
            catch (ArgumentException e)
            {
                Console.WriteLine("Не получилось создать героя: " + e.Message);
            }
        }
    }

    private static void GoOnExpedition(Hero hero, LocationRegistry locations)
    {
        IReadOnlyList<Location> all = locations.All;
        Console.WriteLine("Куда отправимся?");
        for (int i = 0; i < all.Count; i++)
        {
            Console.WriteLine($"{i + 1} - {all[i].DisplayName} (рекомендуемый уровень {all[i].RecommendedLevel})");
        }
        int index = ReadChoice(all.Count);
        if (index < 0)
        {
            return;
        }

        Location location = all[index];
        ExpeditionReport report = location.Explore(hero);
        Console.WriteLine();
        View.PrintReport(report, location.DisplayName);

        if (report.Battle.Outcome == Outcome.Victory && location.Code == DragonLair.LocationCode)
        {
            Console.WriteLine($"\nДракон повержен! Барды еще долго будут петь о подвиге героя по имени {hero.Name}.");
        }
    }

    private static void ManageEquipment(Hero hero)
    {
        // копируем список: после Equip инвентарь героя изменится
        List<Item> inventory = hero.Inventory.ToList();
        if (inventory.Count == 0)
        {
            Console.WriteLine("Инвентарь пуст");
            return;
        }

        Console.WriteLine("Что экипировать? (0 - назад)");
        PrintItems(inventory);
        int index = ReadChoice(inventory.Count);
        if (index < 0)
        {
            return;
        }

        Item item = inventory[index];
        hero.Equip(item);
        Console.WriteLine("Экипировано: " + View.Describe(item));
    }

    private static void VisitMerchant(Hero hero, Merchant merchant)
    {
        Console.WriteLine($"Торговец: \"Чего желаешь?\" (золото: {hero.Gold})");
        Console.WriteLine("1 - купить, 2 - продать, 0 - назад");
        switch (ReadLine())
        {
            case "1":
                Buy(hero, merchant);
                break;
            case "2":
                Sell(hero, merchant);
                break;
        }
    }

    private static void Buy(Hero hero, Merchant merchant)
    {
        IReadOnlyList<Item> showcase = merchant.Showcase;
        for (int i = 0; i < showcase.Count; i++)
        {
            Console.WriteLine($"{i + 1} - {View.Describe(showcase[i])}: {showcase[i].Price} золота");
        }
        int index = ReadChoice(showcase.Count);
        if (index < 0)
        {
            return;
        }

        try
        {
            Item item = merchant.Buy(hero, showcase[index].Code);
            Console.WriteLine("Куплено: " + View.Describe(item));
        }
        catch (InvalidOperationException e)
        {
            Console.WriteLine(e.Message);
        }
    }

    private static void Sell(Hero hero, Merchant merchant)
    {
        List<Item> inventory = hero.Inventory.ToList();
        if (inventory.Count == 0)
        {
            Console.WriteLine("Продавать нечего (экипированные предметы продать нельзя)");
            return;
        }

        for (int i = 0; i < inventory.Count; i++)
        {
            Console.WriteLine($"{i + 1} - {View.Describe(inventory[i])}: {merchant.GetSellPrice(inventory[i])} золота");
        }
        int index = ReadChoice(inventory.Count);
        if (index < 0)
        {
            return;
        }

        Item item = inventory[index];
        int price = merchant.Sell(hero, item);
        Console.WriteLine($"Продано: {item.Name} за {price} золота");
    }

    private static void PrintItems(IReadOnlyList<Item> items)
    {
        for (int i = 0; i < items.Count; i++)
        {
            Console.WriteLine($"{i + 1} - {View.Describe(items[i])}");
        }
    }

    // читает строку без пробелов по краям; если ввод закончился, завершает программу
    private static string ReadLine()
    {
        string? line = Console.ReadLine();
        if (line is null)
        {
            Environment.Exit(0);
        }

        return line.Trim();
    }

    // читает номер варианта от 1 до count, возвращает индекс или -1, если ввели что-то другое
    private static int ReadChoice(int count)
    {
        if (int.TryParse(ReadLine(), out int choice) && choice >= 1 && choice <= count)
        {
            return choice - 1;
        }

        return -1;
    }
}
```

Супер! Попробуйте запустить игру и победить дракона :)

В целом по коду программы видно, что мы почти не привязываемся к конкретным реализациям классов, а везде используем
абстракции (`Location`, а не конкретные локации для экспедиций, `IStarterKitFactory`, а не конкретные наборы для
получения снаряжения, `Item`, а не конкретные предметы при копировании из каталога).

Таким образом в игру легко добавлять новые локации и врагов, классы героя со снаряжением и предметы.

> **_ДОП. ЗАДАНИЕ:_** Если хотите на практике в этом убедиться - попробуйте создать новую локацию и новый класс
> героя, выдав ему новое оружие и броню (не забудьте добавить их в каталог, а для новой локации - добычу!). Много ли мест в старом коде пришлось поменять?

## Демонстрация паттернов

Чтобы показать все паттерны одним запуском, напишем класс `Demo`. Его запускает `Program.cs`, если передать аргумент
`demo`:

```csharp
using Lab3.Core;
using Lab3.Heroes;
using Lab3.Items;
using Lab3.Kits;
using Lab3.World;

namespace Lab3;

public static class Demo
{
    public static void Run()
    {
        ItemCatalog catalog = DefaultCatalog.Create();
        var view = new ConsoleView();

        Console.WriteLine("===== Одиночка =====");
        Dice first = Dice.Instance;
        Dice second = Dice.Instance;
        Console.WriteLine($"Dice.Instance возвращает один и тот же объект: {ReferenceEquals(first, second)}");

        Console.WriteLine();
        Console.WriteLine("===== Прототип =====");
        Item plainSword = catalog.Create(DefaultCatalog.IronSword);
        Item fierySword = catalog.Create(DefaultCatalog.IronSword);
        fierySword.Enchant(Enchantment.Fiery);
        Console.WriteLine("Первая копия:        " + view.Describe(plainSword));
        Console.WriteLine("Вторая копия:        " + view.Describe(fierySword));
        Console.WriteLine("Новая копия шаблона: " + view.Describe(catalog.Create(DefaultCatalog.IronSword)));
        Console.WriteLine($"Копии - разные объекты: {!ReferenceEquals(plainSword, fierySword)}");

        Console.WriteLine();
        Console.WriteLine("===== Абстрактная фабрика =====");
        foreach (HeroClass heroClass in HeroClass.Values)
        {
            IStarterKitFactory kit = heroClass.GetStarterKit(catalog);
            Console.WriteLine($"{heroClass.DisplayName}: {view.Describe(kit.CreateWeapon())}, {view.Describe(kit.CreateArmor())}");
        }

        Console.WriteLine();
        Console.WriteLine("===== Строитель =====");
        Hero hero = new Hero.Builder()
            .SetName("Артур")
            .SetHeroClass(HeroClass.Warrior)
            .SetStarterKit(HeroClass.Warrior.GetStarterKit(catalog))
            .SetLevel(3)
            .Build();
        view.PrintHero(hero);
        try
        {
            new Hero.Builder()
                .SetHeroClass(HeroClass.Mage)
                .SetStarterKit(HeroClass.Mage.GetStarterKit(catalog))
                .Build();
        }
        catch (ArgumentException e)
        {
            Console.WriteLine("Герой без имени не создается: " + e.Message);
        }
        try
        {
            new Hero.Builder()
                .SetName("Мерлин")
                .Build();
        }
        catch (ArgumentException e)
        {
            Console.WriteLine("Герой без класса не создается: " + e.Message);
        }

        Console.WriteLine();
        Console.WriteLine("===== Фабричный метод =====");
        var forest = new Forest(catalog);
        var crypt = new Crypt(catalog);
        view.PrintReport(forest.Explore(hero, 7), forest.DisplayName);
        Console.WriteLine();
        view.PrintReport(crypt.Explore(hero, 7), crypt.DisplayName);

    }
}
```

Вывод программы:

```
===== Одиночка =====
Dice.Instance возвращает один и тот же объект: True

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
[раунд 1] Гоблин -> Артур: 6 урона, осталось 144/150
[раунд 2] Артур -> Гоблин: 18 урона, осталось 14/50
[раунд 2] Гоблин -> Артур: 9 урона, осталось 135/150
[раунд 3] Артур -> Гоблин: 18 урона, осталось 0/50
Гоблин повержен!
Победа! +40 опыта, +25 золота
Добыча: Охотничий лук (урон 10, шанс крита 30%)

=== Заброшенный склеп (сид 7) ===
Артур встречает врага: Скелет (110 здоровья)
[раунд 1] Артур -> Скелет: 13 урона, осталось 97/110
[раунд 1] Скелет -> Артур: 11 урона, осталось 139/150
[раунд 2] Артур -> Скелет: 13 урона, осталось 84/110
[раунд 2] Скелет -> Артур: 16 урона, осталось 123/150
[раунд 3] Артур -> Скелет: 13 урона, осталось 71/110
[раунд 3] Скелет -> Артур: 13 урона, осталось 110/150
[раунд 4] Артур -> Скелет: 13 урона, осталось 58/110
[раунд 4] Скелет -> Артур: 7 урона, осталось 103/150
[раунд 5] Артур -> Скелет: 13 урона, осталось 45/110
[раунд 5] Скелет -> Артур: 11 урона, осталось 92/150
[раунд 6] Артур -> Скелет: 13 урона, осталось 32/110
[раунд 6] Скелет -> Артур: 14 урона, осталось 78/150
[раунд 7] Артур -> Скелет: 13 урона, осталось 19/110
[раунд 7] Скелет -> Артур: 7 урона, осталось 71/150
[раунд 8] Артур -> Скелет: 13 урона, осталось 6/110
[раунд 8] Скелет -> Артур: 17 урона, осталось 54/150
[раунд 9] Артур -> Скелет: 13 урона, осталось 0/110
Скелет повержен!
Победа! +90 опыта, +65 золота
```

Не забудьте закоммитить и запушить изменения!

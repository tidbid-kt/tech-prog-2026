# Лаба 4 — практика на C#

Это практическая часть [лабораторной работы 4](../laba4.md): требования к сдаче и вопросы к защите находятся в основном тексте лабы.

Понадобится .NET SDK 10 или новее, сторонние библиотеки не нужны. Шаблон репозитория: [tech-prog-labs-2026-csharp](https://github.com/tidbid-kt/tech-prog-labs-2026-csharp). Код лабы пишите в проекте `labs/lab3`. Все команды запускаются из корня репозитория: игра - `dotnet run --project labs/lab3`, демонстрация - `dotnet run --project labs/lab3 -- demo`.

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

**`labs/lab3/Defense/Damage.cs`**

```csharp
namespace Lab3.Defense;

public class Damage {

    private int amount;
    private bool evaded;
    private int absorbed;

    public Damage(int amount) {
        this.amount = amount;
    }

    public void ReduceTo(int amount) {
        this.amount = amount;
    }

    public void Evade() {
        amount = 0;
        evaded = true;
    }

    public void Absorb(int value) {
        amount -= value;
        absorbed += value;
    }

    public int Amount => amount;

    public bool IsEvaded => evaded;

    public int Absorbed => absorbed;
}
```

И теперь `ICombatant.ReceiveHit` будет возвращать `Damage` вместо `int`. Так мы сможем реализовать отдельные механики защиты от урона.
Получится что-то такое:

**`labs/lab3/Combat/ICombatant.cs`**

```csharp
using Lab3.Defense;
namespace Lab3.Combat;

// всё, что может участвовать в бою: и герой, и враги
public interface ICombatant {
    // код вида сущности: класс героя или тип врага
    string Code { get; }

    string Name { get; }

    int Health { get; }

    int MaxHealth { get; }

    bool IsAlive { get; }

    // удар, который будет нанесен данной сущностью
    Hit RollDamage();

    // получение удара, возвращает урон, который прошел на самом деле
    Damage ReceiveHit(int damage);
}
```

Для врагов давайте обработку урона в `ReceiveHit` также сделаем через `Damage`, не зря же мы этот класс добавляли:

**`labs/lab3/Enemies/AbstractEnemy.cs`**

```csharp
using Lab3.Defense;
using Lab3.Combat;
using Lab3.Core;

namespace Lab3.Enemies;

// общая логика для врагов: здоровье, урон в диапазоне, награда
public abstract class AbstractEnemy : IEnemy {
    private readonly int minDamage;
    private readonly int maxDamage;
    private readonly int goldReward;

    protected AbstractEnemy(string code, string name, int maxHealth, int minDamage, int maxDamage,
                            int experienceReward, int goldReward) {
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
    public virtual Hit RollDamage() {
        return new Hit(Dice.Instance.Roll(minDamage, maxDamage), false);
    }

    public Damage ReceiveHit(int damage) {
        int dealt = Math.Max(1, ReduceDamage(damage));
        Health = Math.Max(0, Health - dealt);
        return new Damage(dealt);
    }

    public int RollGoldReward() {
        return Dice.Instance.Roll(goldReward / 2, goldReward);
    }

    // наследники могут уменьшать входящий урон
    protected virtual int ReduceDamage(int damage) {
        return damage;
    }
}
```

В `Battle` добавим обработку результата `ReceiveHit`, с учетом того, что он теперь возвращает `Damage`:

**`labs/lab3/Combat/Battle.cs`**

```csharp
namespace Lab3.Combat;

// бой идет по раундам: сначала бьет герой, потом враг, пока кто-то не погибнет
public sealed class Battle {
    private readonly ICombatant hero;
    private readonly ICombatant enemy;
    private readonly List<BattleEvent> events = new();
    private int round;

    public Battle(ICombatant hero, ICombatant enemy) {
        this.hero = hero;
        this.enemy = enemy;
    }

    public BattleReport Run() {
        while (hero.IsAlive && enemy.IsAlive) {
            round++;
            Strike(hero, Side.Hero, enemy, Side.Enemy);
            if (enemy.IsAlive) {
                Strike(enemy, Side.Enemy, hero, Side.Hero);
            }
        }

        Outcome outcome = hero.IsAlive ? Outcome.Victory : Outcome.Defeat;
        return new BattleReport(Participant.Of(hero), Participant.Of(enemy), outcome, events.ToList());
    }

    private void Strike(ICombatant attacker, Side source, ICombatant defender, Side target) {
        Hit hit = attacker.RollDamage();
        int dealt = defender.ReceiveHit(hit.Damage).Amount;
        EventType type = hit.Critical ? EventType.CriticalHit : EventType.Hit;
        Emit(new BattleEvent(round, type, source, target, dealt, defender.Health));

        if (!defender.IsAlive) {
            Emit(new BattleEvent(round, EventType.Death, source, target, 0, 0));
        }
    }

    private void Emit(BattleEvent battleEvent) {
        events.Add(battleEvent);
    }
}
```

А еще у нас в игре появятся достижения. Выдаваться они будут с помощью паттерна "наблюдатель", но объявим мы их сразу здесь.

Представлять собой они будут идентификатор, название и описание. Как-то так:

**`labs/lab3/Progress/Achievement.cs`**

```csharp
using Lab3.Core;
namespace Lab3.Progress;
public sealed record Achievement(string Code, string DisplayName, string Description) {
    public static readonly Achievement FirstBlood =
        new("first-blood", "Первая кровь", "одержать первую победу");
    public static readonly Achievement Lucky = new("lucky", "Счастливчик", "найти зачарованный лут");
    public static readonly Achievement Veteran = new("veteran", "Ветеран", "достичь 5 уровня");
    public static readonly Achievement DragonSlayer =
        new("dragon-slayer", "Драконоборец", "победить дракона");
    public static readonly IReadOnlyList<Achievement> Values = [FirstBlood, Lucky, Veteran, DragonSlayer];
    public static Achievement ByCode(string code) =>
        Values.FirstOrDefault(a => a.Code == code) ?? throw new UnknownCodeException("Достижение", code);
}
```

В `Hero` добавим хранение достижений, полученных за этого героя, а также возможность восстановления опыта, инвентаря и достижений через
строитель.

Нам это понадобится, когда будем добавлять механику сохранений!

А еще метод отдыха переименуем в `Rest` вместо `RestoreHealth`. В нем позже появится дополнительное поведение, поэтому стоит сразу немного
расширить контракт.

Итог выглядит вот так:

**`labs/lab3/Heroes/Hero.cs`**

```csharp
using Lab3.Defense;
using Lab3.Progress;
using Lab3.Combat;
using Lab3.Items;
using Lab3.Kits;

namespace Lab3.Heroes;

public sealed class Hero : ICombatant {
    private const int MaxNameLength = 20;
    private const int StartingGold = 50;
    private const int HealthPerLevel = 15;
    private const int DamagePerLevel = 3;
    private const int ExperiencePerLevel = 100;

    private readonly List<Item> inventory = new();
    private readonly List<Achievement> achievements;
    public IReadOnlyList<Achievement> Achievements => achievements.AsReadOnly();

    // конструктор приватный и принимает в себя билдера, из которого и берет значения полей
    private Hero(Builder builder) {
        Name = builder.Name!;
        HeroClass = builder.HeroClass!;
        Level = builder.Level;
        Gold = builder.Gold;
        Weapon = builder.Weapon!;
        Armor = builder.Armor!;
        Experience = builder.Experience;
        inventory.AddRange(builder.Inventory);
        achievements = builder.Achievements.Distinct().ToList();
        Rest();
    }

    // внутренний класс билдер - чтобы удобно и безопасно создавать героев
    public sealed class Builder {
        internal string? Name { get; private set; }

        internal HeroClass? HeroClass { get; private set; }

        internal Weapon? Weapon { get; private set; }

        internal Armor? Armor { get; private set; }

        // необязательные поля сразу получают значения по умолчанию
        internal int Level { get; private set; } = 1;

        internal int Gold { get; private set; } = StartingGold;

        internal int Experience { get; private set; }
        internal List<Item> Inventory { get; private set; } = new();
        internal List<Achievement> Achievements { get; private set; } = new();
        public Builder SetExperience(int value) {
            Experience = value;
            return this;
        }
        public Builder SetInventory(IEnumerable<Item> items) {
            Inventory = items.ToList();
            return this;
        }
        public Builder SetAchievements(IEnumerable<Achievement> items) {
            Achievements = items.ToList();
            return this;
        }

        public Builder SetName(string name) {
            Name = name;
            return this;
        }

        public Builder SetHeroClass(HeroClass heroClass) {
            HeroClass = heroClass;
            return this;
        }

        public Builder SetWeapon(Weapon weapon) {
            Weapon = weapon;
            return this;
        }

        public Builder SetArmor(Armor armor) {
            Armor = armor;
            return this;
        }

        // берет оружие и броню из стартового набора
        public Builder SetStarterKit(IStarterKitFactory starterKit) {
            Weapon = starterKit.CreateWeapon();
            Armor = starterKit.CreateArmor();
            return this;
        }

        public Builder SetLevel(int level) {
            Level = level;
            return this;
        }

        public Builder SetGold(int gold) {
            Gold = gold;
            return this;
        }

        // перед созданием героя проверяем, что все обязательные поля заполнены корректно
        public Hero Build() {
            if (string.IsNullOrWhiteSpace(Name)) {
                throw new ArgumentException("У героя должно быть имя");
            }
            if (Name.Length > MaxNameLength) {
                throw new ArgumentException($"Имя героя не может быть длиннее {MaxNameLength} символов");
            }
            if (HeroClass is null) {
                throw new ArgumentException("Не выбран класс героя");
            }
            if (Weapon is null || Armor is null) {
                throw new ArgumentException("Герою нужны оружие и броня");
            }
            if (Level < 1) {
                throw new ArgumentException("Уровень героя не может быть меньше 1");
            }
            if (Gold < 0) {
                throw new ArgumentException("Золота не может быть меньше нуля");
            }

            if (Experience < 0)
                throw new ArgumentException("Опыт не может быть отрицательным");
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

    public Hit RollDamage() {
        Hit hit = Weapon.RollDamage();
        return hit with { Damage = hit.Damage + (Level - 1) * DamagePerLevel };
    }

    public Damage ReceiveHit(int damage) {
        int reducedDamage = (int)Math.Round(damage * (1 - Armor.Defense), MidpointRounding.AwayFromZero);
        // хотя бы единицу урона броня пропускает всегда
        int dealt = Math.Max(1, reducedDamage);
        Health = Math.Max(0, Health - dealt);
        return new Damage(dealt);
    }

    // поведение героя между экспедициями

    // перед каждой экспедицией герой успевает отдохнуть
    public void Rest() {
        Health = MaxHealth;
    }

    // начисляет опыт, возвращает количество полученных уровней
    public int GainExperience(int amount) {
        Experience += amount;
        int levelsGained = 0;
        while (Experience >= ExperienceToNextLevel) {
            Experience -= ExperienceToNextLevel;
            Level++;
            levelsGained++;
        }

        return levelsGained;
    }

    public void AddGold(int amount) {
        Gold += amount;
    }

    public void SpendGold(int amount) {
        if (amount > Gold) {
            throw new InvalidOperationException($"Не хватает золота: нужно {amount}, есть {Gold}");
        }

        Gold -= amount;
    }

    public void AddItem(Item item) {
        inventory.Add(item);
    }

    public void RemoveItem(Item item) {
        if (!inventory.Remove(item)) {
            throw new ArgumentException($"В инвентаре нет предмета {item.Name}");
        }
    }

    // экипирует предмет из инвентаря, снятый предмет возвращается в инвентарь
    public void Equip(Item item) {
        switch (item) {
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
    public bool Unlock(Achievement achievement) {
        if (achievements.Contains(achievement))
            return false;
        achievements.Add(achievement);
        return true;
    }
}
```

В `Location` заменяем вызов `RestoreHealth()` на `Rest()` и открываем подклассам доступ к каталогу через поле `catalog`,
это нам скоро понадобится:

**`labs/lab3/World/Location.cs`**

```csharp
using Lab3.Combat;
using Lab3.Core;
using Lab3.Enemies;
using Lab3.Heroes;
using Lab3.Items;

namespace Lab3.World;

public abstract class Location {
    private const double LootChance = 0.6;
    private const double EnchantChance = 0.3;

    protected readonly ItemCatalog catalog;

    protected Location(string code, string displayName, int recommendedLevel, ItemCatalog catalog) {
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

    public ExpeditionReport Explore(Hero hero) {
        return Explore(hero, Dice.Instance.NextSeed());
    }

    // бизнес-логика, общая для всех локаций:
    // герой встречает врага, сражается с ним и получает награду, если победил
    public ExpeditionReport Explore(Hero hero, int seed) {
        Dice dice = Dice.Instance;
        dice.Reseed(seed);

        hero.Rest();
        IEnemy enemy = CreateEnemy();
        BattleReport battle = new Battle(hero, enemy).Run();

        Rewards rewards = battle.Outcome == Outcome.Victory ? Reward(hero, enemy, dice) : Punish(hero);

        return new ExpeditionReport(seed, Code, battle, rewards);
    }

    private Rewards Reward(Hero hero, IEnemy enemy, Dice dice) {
        int experience = enemy.ExperienceReward;
        int gold = enemy.RollGoldReward();
        Item? loot = RollLoot(dice);

        int levelsGained = hero.GainExperience(experience);
        hero.AddGold(gold);
        if (loot is not null) {
            hero.AddItem(loot);
        }

        return new Rewards(experience, gold, loot, levelsGained);
    }

    private Item? RollLoot(Dice dice) {
        if (!dice.Chance(LootChance)) {
            return null;
        }

        // добыча - это копия шаблона из каталога, ее можно менять, не боясь испортить шаблон
        Item loot = catalog.Create(dice.Pick(LootTable));
        if (dice.Chance(EnchantChance)) {
            loot.Enchant(dice.Pick(loot.PossibleEnchantments));
        }

        return loot;
    }

    // проигравший герой теряет половину золота
    private static Rewards Punish(Hero hero) {
        int lostGold = hero.Gold / 2;
        hero.SpendGold(lostGold);
        return new Rewards(0, -lostGold, null, 0);
    }
}
```

Чтобы иметь возможность восстановить зачарование по идентификатору (нам это понадобится дальше), добавим в него поиск по этому самому
идентификатору:

**`labs/lab3/Items/Enchantment.cs`**

```csharp
namespace Lab3.Items;

// зачарование неизменяемо (record с init-свойствами нельзя поменять после создания),
// поэтому одно и то же зачарование можно безопасно повесить на несколько предметов
public sealed record Enchantment(string Code, string Name, int BonusDamage, double BonusDefense,
                                 int BonusPrice) {
    public static readonly Enchantment Sharp = new("sharp", "Острое", 3, 0, 20);
    public static readonly Enchantment Fiery = new("fiery", "Огненное", 6, 0, 50);
    public static readonly Enchantment Sturdy = new("sturdy", "Прочное", 0, 0.05, 20);
    public static readonly Enchantment Warding = new("warding", "Охранное", 0, 0.10, 50);

    public static readonly IReadOnlyList<Enchantment> ForWeapons = [Sharp, Fiery];
    public static readonly IReadOnlyList<Enchantment> ForArmor = [Sturdy, Warding];
    public static Enchantment ByCode(string code) =>
        ForWeapons.Concat(ForArmor).FirstOrDefault(e => e.Code == code) ??
        throw new Lab3.Core.UnknownCodeException("Зачарование", code);
}
```

Супер, предварительные изменения внесены, давайте приступать к паттернам и масштабному расширению функционала!

## Одержимое оружие в проклятом особняке (Адаптер / Adapter)

Представим, что мы хотим ввести в нашу игру новую локацию. Вдохновившись историями про жутких призраков таковой мы решаем сделать
проклятый особняк, в нем на героя будет нападать одержимые предметы!

Писать новую реализацию врага с нуля мы не хотим, было бы круто и удобно использовать код класса `Weapon`, это нам и идейно хорошо подходит,
и часть логики пересекается, и `Weapon`, и `IEnemy` умеют наносить урон! Но загвоздка в том, что получать урон оружие не умеет...

Чтобы решить это недоразумение и заставить `Weapon` полноценно соответствовать контракту `IEnemy` без изменения самого `Weapon`,
применим замечательный паттерн *Адаптер*.

Он позволяет объектам с несовместимыми интерфейсами работать вместе: адаптер оборачивает нужный объект (класса `Weapon` в нашем случае) и
реализует нужный интерфейс (`IEnemy` для нас), переводя обращения к себе в вызовы методов оборачиваемого объекта. А если для части контрактов
методов у оборачиваемого объекта нет — адаптер реализует их сам.

Таким образом мы сможем прокинуть `Weapon` как `IEnemy` в новый подкласс `Location` с помощью адаптера, и нам не придется менять ни один из
упомянутых
выше классов!

Давайте скорее реализовывать это чудо, напишем сам адаптер. Назовем его `HauntedWeapon`, чтобы выдержать дух, но можно было назвать и более
технично, например,
`WeaponToEnemyAdapter`, суть была бы та же, но не так зловеще.

И так, код получится вот такой:

**`labs/lab3/Enemies/HauntedWeapon.cs`**

```csharp
using Lab3.Combat;
using Lab3.Core;
using Lab3.Defense;
using Lab3.Items;

namespace Lab3.Enemies;

public class HauntedWeapon : IEnemy {

    // наше одержимое оружие должно само придумать, сколько у него будет здоровья (у Weapon этой логики нет)
    // поэтому считаем, что за каждую единицу урона полагается 5 здоровья
    private const int HEALTH_PER_DAMAGE = 5;

    private readonly Weapon weapon;
    private readonly int maxHealth;
    private int health;

    public HauntedWeapon(Weapon weapon) {
        this.weapon = weapon;
        // проставляем здоровье по формуле
        this.maxHealth = weapon.Damage * HEALTH_PER_DAMAGE;
        this.health = maxHealth;
    }

    public string Code => "haunted-weapon";

    // название врага берется из названия оружия
    public string Name => "Одержимый " + weapon.Name;

    // метод нанесения урона полностью переиспользует логику оружия, тут основная ценность адаптера
    public Hit RollDamage() {
        return weapon.RollDamage();
    }

    // а вот тут ценность адаптера не такая явная, методы ниже мы реализуем сами почти с нуля
    public Damage ReceiveHit(int amount) {
        Damage damage = new Damage(amount);
        health = Math.Max(0, health - damage.Amount);
        return damage;
    }

    public bool IsAlive => health > 0;

    // награду за победу над одержимым оружием привяжем к его стоимости
    public int ExperienceReward => weapon.Price / 2;

    public int RollGoldReward() {
        return Dice.Instance.Roll(weapon.Price / 4, weapon.Price / 2);
    }

    public int Health => health;

    public int MaxHealth => maxHealth;
}
```

А теперь объявим и саму локацию — проклятый особняк. В ней будет одно из трех случайных оружий, которые будут взяты из каталога.
Выбранное оружие будет обернуто в адаптер, то есть станет одержимо злым духом, после чего безжалостно нападет на нашего героя.

Благодаря адаптеру мы можем превратить во врага любое оружие, что безусловно разнообразит визиты в это замечательное жилище:

**`labs/lab3/World/HauntedManor.cs`**

```csharp
using Lab3.Core;
using Lab3.Enemies;
using Lab3.Items;
namespace Lab3.World;
public sealed class HauntedManor : Location {
    public const string LocationCode = "haunted-manor";
    public HauntedManor(ItemCatalog catalog) : base(LocationCode, "Проклятый особняк", 2, catalog) {}
    protected override IReadOnlyList<string>
        LootTable => [DefaultCatalog.IronSword, DefaultCatalog.ElvenBow, DefaultCatalog.EmberStaff];
    protected override
        IEnemy CreateEnemy() => new HauntedWeapon(catalog.CreateWeapon(Dice.Instance.Pick(LootTable)));
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

Важно, что `wrapped` это абстракция – `IEnemy` и сам базовый декоратор тоже реализует эту абстракцию. Таким образом на месте `wrapped` может
быть и другой декоратор.
Но в конце цепочки обязательно будет конкретный враг.

Итак, код получается вот такой:

**`labs/lab3/Enemies/EnemyDecorator.cs`**

```csharp
using Lab3.Combat;
using Lab3.Defense;

namespace Lab3.Enemies;

public abstract class EnemyDecorator : IEnemy {

    private readonly IEnemy wrapped;

    protected EnemyDecorator(IEnemy wrapped) {
        this.wrapped = wrapped;
    }

    public virtual string Code => wrapped.Code;

    public virtual string Name => wrapped.Name;

    public virtual int Health => wrapped.Health;

    public virtual int MaxHealth => wrapped.MaxHealth;

    public virtual bool IsAlive => wrapped.IsAlive;

    public virtual Hit RollDamage() {
        return wrapped.RollDamage();
    }

    public virtual Damage ReceiveHit(int damage) {
        return wrapped.ReceiveHit(damage);
    }

    public virtual int ExperienceReward => wrapped.ExperienceReward;

    public virtual int RollGoldReward() {
        return wrapped.RollGoldReward();
    }
}
```

А дальше уже реализуем конкретных декораторов, наследников базового. Они будут соответствовать нашим модификатором, начнем с легендарного:

**`labs/lab3/Enemies/LegendaryEnemy.cs`**

```csharp
using Lab3.Combat;

namespace Lab3.Enemies;

public class LegendaryEnemy : EnemyDecorator {

    private const int MULTIPLIER = 2;

    public LegendaryEnemy(IEnemy wrapped) : base(wrapped) {}

    // добавляем в имя оборачиваемого объекта префикс `Легендарный`
    public override string Name => "Легендарный " + base.Name;

    // удваиваем урон оборачиваемого объекта
    public override Hit RollDamage() {
        Hit hit = base.RollDamage();
        return new Hit(hit.Damage * MULTIPLIER, hit.Critical);
    }

    // удваиваем награду за легендарного врага
    public override int ExperienceReward => base.ExperienceReward * MULTIPLIER;

    public override int RollGoldReward() {
        return base.RollGoldReward() * MULTIPLIER;
    }
}
```

Тут важно заметить, что легендарный декоратор во всех своих методах вызывает методы своего суперкласса – базового декоратора. При
нанесении двойного урона он берет за основу именно `base.RollDamage()`. Именно это и позволяет комбинировать декораторов. Мы уже видели,
что
базовый декоратор просто вызывает методы обертки, значит `base.RollDamage()` это `wrapped.RollDamage()` и если `wrapped` это враг в еще
одном декораторе –
дальше будет вызвано уже модифицированное им поведение. И так по цепочке пока мы не дойдем до одного из наших конкретных врагов.

Супер, базовый принцип понятен, продолжим реализовывать модификаторы, дальше у нас неистовство ветра:

**`labs/lab3/Enemies/WindfuryEnemy.cs`**

```csharp
using Lab3.Combat;

namespace Lab3.Enemies;

public class WindfuryEnemy : EnemyDecorator {

    private const int REWARD_BONUS_PERCENT = 50;

    public WindfuryEnemy(IEnemy wrapped) : base(wrapped) {}

    public override string Name => "Неистовый " + base.Name;

    public override Hit RollDamage() {
        Hit first = base.RollDamage();
        Hit second = base.RollDamage();
        return new Hit(first.Damage + second.Damage, first.Critical || second.Critical);
    }

    public override int ExperienceReward => WithBonus(base.ExperienceReward);

    public override int RollGoldReward() {
        return WithBonus(base.RollGoldReward());
    }

    private int WithBonus(int reward) {
        return reward + reward * REWARD_BONUS_PERCENT / 100;
    }
}
```

Два броска здесь складываются в один удар, поэтому защита применяется к их сумме.

Логика тут точно такая же! Модифицируем поведение, делегируем в `base`.

Третий и пока финальный декоратор уменьшает входящий урон вдвое, затем передаёт получившийся урон дальше по цепочке декораторов:

**`labs/lab3/Enemies/ArmoredEnemy.cs`**

```csharp
using Lab3.Defense;

namespace Lab3.Enemies;

public class ArmoredEnemy : EnemyDecorator {

    public ArmoredEnemy(IEnemy wrapped) : base(wrapped) {}

    public override string Name => "Бронированный " + base.Name;

    public override Damage ReceiveHit(int amount) {
        if (amount < 0) {
            throw new ArgumentException("Урон не может быть отрицательным");
        }
        if (amount == 0) {
            return new Damage(0);
        }

        return base.ReceiveHit(amount - amount / 2);
    }
}
```

Отлично, все требуемые модификаторы объявлены.
Теперь мы можем добавлять их любому врагу и в любой последовательности.
При этом декоратор реализует тот же интерфейс `IEnemy`, что и обычный враг, поэтому ни бой, ни локация не отличают элитного врага
от обычного, и работают с ним точно так же. Прелести `Dependency Inversion Principle`!

Осталось прописать логику, как и когда враг становится элитным. Это часть общей логики экспедиции, поэтому добавим её в локацию:
враг, созданный фабричным методом, с некоторым шансом получает комбинацию модификаторов.

**`labs/lab3/World/Location.cs`**

```csharp
using Lab3.Combat;
using Lab3.Core;
using Lab3.Enemies;
using Lab3.Heroes;
using Lab3.Items;

namespace Lab3.World;

public abstract class Location {
    private const double LootChance = 0.6;
    private const double EnchantChance = 0.3;

    protected readonly ItemCatalog catalog;

    protected Location(string code, string displayName, int recommendedLevel, ItemCatalog catalog) {
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

    public ExpeditionReport Explore(Hero hero) {
        return Explore(hero, Dice.Instance.NextSeed());
    }

    // бизнес-логика, общая для всех локаций:
    // герой встречает врага, сражается с ним и получает награду, если победил
    public ExpeditionReport Explore(Hero hero, int seed) {
        Dice dice = Dice.Instance;
        dice.Reseed(seed);

        hero.Rest();
        IEnemy enemy = CreateEnemy();
        if (dice.Chance(0.2))
            enemy = MakeElite(enemy, dice);
        BattleReport battle = new Battle(hero, enemy).Run();

        Rewards rewards = battle.Outcome == Outcome.Victory ? Reward(hero, enemy, dice) : Punish(hero);

        return new ExpeditionReport(seed, Code, battle, rewards);
    }

    private Rewards Reward(Hero hero, IEnemy enemy, Dice dice) {
        int experience = enemy.ExperienceReward;
        int gold = enemy.RollGoldReward();
        Item? loot = RollLoot(dice);

        int levelsGained = hero.GainExperience(experience);
        hero.AddGold(gold);
        if (loot is not null) {
            hero.AddItem(loot);
        }

        return new Rewards(experience, gold, loot, levelsGained);
    }

    private Item? RollLoot(Dice dice) {
        if (!dice.Chance(LootChance)) {
            return null;
        }

        // добыча - это копия шаблона из каталога, ее можно менять, не боясь испортить шаблон
        Item loot = catalog.Create(dice.Pick(LootTable));
        if (dice.Chance(EnchantChance)) {
            loot.Enchant(dice.Pick(loot.PossibleEnchantments));
        }

        return loot;
    }

    // проигравший герой теряет половину золота
    private static Rewards Punish(Hero hero) {
        int lostGold = hero.Gold / 2;
        hero.SpendGold(lostGold);
        return new Rewards(0, -lostGold, null, 0);
    }
    private static IEnemy MakeElite(IEnemy enemy, Dice dice) {
        bool both = dice.Chance(0.25);
        if (dice.Chance(0.5)) {
            enemy = new LegendaryEnemy(enemy);
            if (both)
                enemy = new WindfuryEnemy(enemy);
        } else {
            enemy = new WindfuryEnemy(enemy);
            if (both)
                enemy = new LegendaryEnemy(enemy);
        }
        return dice.Chance(0.3) ? new ArmoredEnemy(enemy) : enemy;
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

**`labs/lab3/Storage/IHeroRepository.cs`**

```csharp
using Lab3.Heroes;
namespace Lab3.Storage;
public interface IHeroRepository {
    Hero? FindByName(string name);
    IReadOnlyList<Hero> FindAll();
    void Save(Hero hero);
}
```

Для хранения сейвов будем использовать простой файл, это не бд, но подход тоже вполне рабочий. В качестве формата сохранения выберем
json! [JSON](https://www.json.org/json-ru.html) — текстовый формат, где объекты состоят из пар "имя — значение",
а массивы хранят списки значений. Это довольно вездесущий формат, и нам он как нельзя кстати подходит.

Для чтения и записи JSON используем встроенный `System.Text.Json`, дополнительные зависимости не нужны.

В файле сохранения будет массив героев со всеми нужными для восстановления объектов данными.
В `FileHeroRepository` приватные записи `HeroSave` и `ItemSave` описывают данные сохранения. Сериализатор преобразует их в JSON и обратно; героя восстанавливаем через строителя, а предметы — через каталог.

Реализуем наконец сам репозиторий:

**`labs/lab3/Storage/FileHeroRepository.cs`**

```csharp
using System.Text;
using System.Text.Json;
using System.Text.Encodings.Web;
using Lab3.Heroes;
using Lab3.Items;
using Lab3.Progress;
namespace Lab3.Storage;
public sealed class FileHeroRepository(string file, ItemCatalog catalog) : IHeroRepository {
    private static readonly JsonSerializerOptions Options =
        new() { WriteIndented = true, PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
                Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping };
    public Hero? FindByName(string name) => FindAll().FirstOrDefault(hero => hero.Name == name);
    public IReadOnlyList<Hero> FindAll() {
        if (!File.Exists(file))
            return [];
        string text = File.ReadAllText(file, Encoding.UTF8);
        if (string.IsNullOrWhiteSpace(text))
            return [];
        var saves = JsonSerializer.Deserialize<List<HeroSave>>(text, Options) ??
                    throw new InvalidDataException("В файле должен быть массив героев");
        return saves.Select(Restore).ToArray();
    }
    public void Save(Hero hero) {
        var saves = FindAll().Where(other => other.Name != hero.Name).Select(Snapshot).ToList();
        saves.Add(Snapshot(hero));
        Directory.CreateDirectory(Path.GetDirectoryName(Path.GetFullPath(file))!);
        File.WriteAllText(file, JsonSerializer.Serialize(saves, Options) + "\n", new UTF8Encoding(false));
    }
    private static ItemSave SnapshotItem(Item item) => new(item.Code,
                                                           item.Enchantments.Select(e => e.Code).ToList());
    private static HeroSave Snapshot(Hero hero) => new(hero.Name, hero.HeroClass.Code, hero.Level,
                                                       hero.Experience, hero.Gold, SnapshotItem(hero.Weapon),
                                                       SnapshotItem(hero.Armor),
                                                       hero.Inventory.Select(SnapshotItem).ToList(),
                                                       hero.Achievements.Select(a => a.Code).ToList());
    private Item RestoreItem(ItemSave save) {
        Item item = catalog.Create(save.Code);
        foreach (string code in save.Enchantments)
            item.Enchant(Enchantment.ByCode(code));
        return item;
    }
    private Hero Restore(HeroSave save) => new Hero.Builder()
                                               .SetName(save.Name)
                                               .SetHeroClass(HeroClass.ByCode(save.HeroClass))
                                               .SetLevel(save.Level)
                                               .SetExperience(save.Experience)
                                               .SetGold(save.Gold)
                                               .SetWeapon((Weapon)RestoreItem(save.Weapon))
                                               .SetArmor((Armor)RestoreItem(save.Armor))
                                               .SetInventory(save.Inventory.Select(RestoreItem))
                                               .SetAchievements(save.Achievements.Select(Achievement.ByCode))
                                               .Build();
    private sealed record ItemSave(string Code, List<string> Enchantments);
    private sealed record HeroSave(string Name, string HeroClass, int Level, int Experience, int Gold,
                                   ItemSave Weapon, ItemSave Armor, List<ItemSave> Inventory,
                                   List<string> Achievements);
}
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

**`labs/lab3/Storage/CachingHeroRepository.cs`**

```csharp
using Lab3.Heroes;
namespace Lab3.Storage;
public sealed class CachingHeroRepository(IHeroRepository target) : IHeroRepository {
    private Dictionary<string, Hero>? cache;
    private Dictionary<string, Hero> Cached() => cache ??= target.FindAll().ToDictionary(hero => hero.Name);
    public Hero? FindByName(string name) => Cached().GetValueOrDefault(name);
    public IReadOnlyList<Hero> FindAll() => Cached().Values.ToArray();
    public void Save(Hero hero) {
        target.Save(hero);
        Cached()[hero.Name] = hero;
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

**`labs/lab3/Tactics/Action.cs`**

```csharp
namespace Lab3.Tactics;
public sealed record Action(double AttackMultiplier, double IncomingMultiplier) {
    public static readonly Action Attack = new(1.0, 1.0);
    public static readonly Action PowerStrike = new(1.5, 1.5);
    public static readonly Action Defend = new(0.5, 0.3);
}
```

Получается, что он может вести себя агрессивно (увеличивается и исходящий, и входящий урон), сбалансированно (ничего не меняется) и
осторожно, уходя в оборону (уменьшается и входящий, и исходящий урон).

Выбирать действия мы будем в зависимости от состояния врага и героя, определим базовый интерфейс тактики:

**`labs/lab3/Tactics/ITactic.cs`**

```csharp
using Lab3.Combat;
namespace Lab3.Tactics;
public interface ITactic {
    string Code { get; }
    string DisplayName { get; }
    string Description { get; }
    Action ChooseAction(int round, ICombatant hero, ICombatant enemy);
}
```

Реализуем самый простой алгоритм выбора действия - обычный удар каждый раунд:

**`labs/lab3/Tactics/BalancedTactic.cs`**

```csharp
using Lab3.Combat;
namespace Lab3.Tactics;
public sealed class BalancedTactic : ITactic {
    public string Code => "balanced";
    public string DisplayName => "Баланс";
    public string Description => "обычный удар каждый раунд";
    public Action ChooseAction(int round, ICombatant hero, ICombatant enemy) => Action.Attack;
}
```

Натиск - пока здоровье врага выше, чем у героя, герой бьет безрассудно, нанося и получая в ответ больше урона:

**`labs/lab3/Tactics/AggressiveTactic.cs`**

```csharp
using Lab3.Combat;
namespace Lab3.Tactics;
public sealed class AggressiveTactic : ITactic {
    public string Code => "aggressive";
    public string DisplayName => "Натиск";
    public string Description => "совершать безрассудные удары, пока у врага больше здоровья, чем у героя";
    public Action ChooseAction(int round, ICombatant hero,
                               ICombatant enemy) => enemy.Health > hero.Health? Action.PowerStrike
        : Action.Attack;
}
```

И осторожность - раненый герой через раунд будет уходить в оборону:

**`labs/lab3/Tactics/CautiousTactic.cs`**

```csharp
using Lab3.Combat;
namespace Lab3.Tactics;
public sealed class CautiousTactic : ITactic {
    public string Code => "cautious";
    public string DisplayName => "Осторожность";
    public string Description => "при здоровье ниже 40% через раунд уходить в оборону";
    public Action ChooseAction(int round, ICombatant hero,
                               ICombatant enemy) => hero.Health < hero.MaxHealth * 0.4 && round % 2 == 0
                                                        ? Action.Defend
                                                        : Action.Attack;
}
```

Также дополним типы ивентов в нашей игре, добавим наши действия, и `Evade`/`Absorb`, они нам пригодятся дальше:

**`labs/lab3/Combat/EventType.cs`**

```csharp
namespace Lab3.Combat;

public enum EventType { Hit, CriticalHit, PowerStrike, Defend, Evade, Absorb, Death }
```

Теперь обновим сам `Battle`. Тактику он получает через конструктор и каждый раунд спрашивает у нее, что именно делает герой:

**`labs/lab3/Combat/Battle.cs`**

```csharp
using Lab3.Defense;
using Lab3.Tactics;
using Action = Lab3.Tactics.Action;
namespace Lab3.Combat;

// бой идет по раундам: сначала бьет герой, потом враг, пока кто-то не погибнет
public sealed class Battle {
    private readonly ICombatant hero;
    private readonly ICombatant enemy;
    private readonly List<BattleEvent> events = new();
    private int round;
    private readonly ITactic tactic;

    public Battle(ICombatant hero, ICombatant enemy, ITactic? tactic = null) {
        this.hero = hero;
        this.enemy = enemy;
        this.tactic = tactic ?? new BalancedTactic();
    }

    public BattleReport Run() {
        while (hero.IsAlive && enemy.IsAlive) {
            round++;
            Action action = tactic.ChooseAction(round, hero, enemy);
            if (action == Action.PowerStrike)
                Emit(new BattleEvent(round, EventType.PowerStrike, Side.Hero, Side.Enemy, 0, 0));
            else if (action == Action.Defend)
                Emit(new BattleEvent(round, EventType.Defend, Side.Hero, Side.Hero, 0, 0));
            Strike(hero, Side.Hero, enemy, Side.Enemy, action.AttackMultiplier);
            if (enemy.IsAlive) {
                Strike(enemy, Side.Enemy, hero, Side.Hero, action.IncomingMultiplier);
            }
        }

        Outcome outcome = hero.IsAlive ? Outcome.Victory : Outcome.Defeat;
        return new BattleReport(Participant.Of(hero), Participant.Of(enemy), outcome, events.ToList());
    }

    private void Strike(ICombatant attacker, Side source, ICombatant defender, Side target,
                        double multiplier) {
        Hit hit = attacker.RollDamage();
        Damage damage = defender.ReceiveHit(
            Math.Max(1, (int)Math.Round(hit.Damage * multiplier, MidpointRounding.AwayFromZero)));
        if (damage.IsEvaded) {
            Emit(new BattleEvent(round, EventType.Evade, source, target, 0, defender.Health));
            return;
        }
        if (damage.Absorbed > 0)
            Emit(new BattleEvent(round, EventType.Absorb, source, target, damage.Absorbed, defender.Health));
        if (damage.Amount > 0)
            Emit(new BattleEvent(round, hit.Critical ? EventType.CriticalHit : EventType.Hit, source, target,
                                 damage.Amount, defender.Health));

        if (!defender.IsAlive) {
            Emit(new BattleEvent(round, EventType.Death, source, target, 0, 0));
        }
    }

    private void Emit(BattleEvent battleEvent) {
        events.Add(battleEvent);
    }
}
```

Обратите внимание, что бой ничего не знает о конкретных тактиках: он работает с интерфейсом `ITactic`, и
подставить ему можно любую реализацию. Если появится новая тактика, `Battle` менять не придется.

Ну и в отчете об экспедиции будем запоминать, с какой тактикой герой ее начинал:

**`labs/lab3/World/ExpeditionReport.cs`**

```csharp
using Lab3.Combat;

namespace Lab3.World;

// отчет об экспедиции: с каким сидом, куда, как прошел бой и что получил герой
public sealed record ExpeditionReport(int Seed, string LocationCode, string TacticCode, BattleReport Battle,
                                      Rewards Rewards);
```

Стратегия позволяет нам соблюдать `Open Closed` принцип, он дает нам возможность расширять поведение нашего класса без его изменения, путем
добавления новых реализаций стратегии. Стратегию можно выбирать как в методе, где она используется, так и в конструкторе, все зависит от
конкретной необходимости. В нашем случае стратегия выбирается в конструкторе, потому что по ходу боя она не меняется.

Применять стратегию стоит, когда одно и то же действие можно выполнить несколькими способами и способ нужно выбирать
или менять во время работы программы.

И давайте сразу обновим `Location`: она будет принимать тактику и передавать дальше в `Battle`.

Получится что-то такое:

**`labs/lab3/World/Location.cs`**

```csharp
using Lab3.Tactics;
using Lab3.Combat;
using Lab3.Core;
using Lab3.Enemies;
using Lab3.Heroes;
using Lab3.Items;

namespace Lab3.World;

public abstract class Location {
    private const double LootChance = 0.6;
    private const double EnchantChance = 0.3;

    protected readonly ItemCatalog catalog;

    protected Location(string code, string displayName, int recommendedLevel, ItemCatalog catalog) {
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

    public ExpeditionReport Explore(Hero hero) {
        return Explore(hero, new BalancedTactic(), Dice.Instance.NextSeed());
    }

    // бизнес-логика, общая для всех локаций:
    // герой встречает врага, сражается с ним и получает награду, если победил
    public ExpeditionReport Explore(Hero hero, int seed) => Explore(hero, new BalancedTactic(), seed);
    public ExpeditionReport Explore(Hero hero, ITactic tactic) => Explore(hero, tactic,
                                                                          Dice.Instance.NextSeed());
    public ExpeditionReport Explore(Hero hero, ITactic tactic, int seed) {
        Dice dice = Dice.Instance;
        dice.Reseed(seed);

        hero.Rest();
        IEnemy enemy = CreateEnemy();
        if (dice.Chance(0.2))
            enemy = MakeElite(enemy, dice);
        BattleReport battle = new Battle(hero, enemy, tactic).Run();

        Rewards rewards = battle.Outcome == Outcome.Victory ? Reward(hero, enemy, dice) : Punish(hero);

        return new ExpeditionReport(seed, Code, tactic.Code, battle, rewards);
    }

    private Rewards Reward(Hero hero, IEnemy enemy, Dice dice) {
        int experience = enemy.ExperienceReward;
        int gold = enemy.RollGoldReward();
        Item? loot = RollLoot(dice);

        int levelsGained = hero.GainExperience(experience);
        hero.AddGold(gold);
        if (loot is not null) {
            hero.AddItem(loot);
        }

        return new Rewards(experience, gold, loot, levelsGained);
    }

    private Item? RollLoot(Dice dice) {
        if (!dice.Chance(LootChance)) {
            return null;
        }

        // добыча - это копия шаблона из каталога, ее можно менять, не боясь испортить шаблон
        Item loot = catalog.Create(dice.Pick(LootTable));
        if (dice.Chance(EnchantChance)) {
            loot.Enchant(dice.Pick(loot.PossibleEnchantments));
        }

        return loot;
    }

    // проигравший герой теряет половину золота
    private static Rewards Punish(Hero hero) {
        int lostGold = hero.Gold / 2;
        hero.SpendGold(lostGold);
        return new Rewards(0, -lostGold, null, 0);
    }
    private static IEnemy MakeElite(IEnemy enemy, Dice dice) {
        bool both = dice.Chance(0.25);
        if (dice.Chance(0.5)) {
            enemy = new LegendaryEnemy(enemy);
            if (both)
                enemy = new WindfuryEnemy(enemy);
        } else {
            enemy = new WindfuryEnemy(enemy);
            if (both)
                enemy = new LegendaryEnemy(enemy);
        }
        return dice.Chance(0.3) ? new ArmoredEnemy(enemy) : enemy;
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

**`labs/lab3/Events/GameEventType.cs`**

```csharp
namespace Lab3.Events;
public enum GameEventType { Victory, Defeat, LevelUp }
```

И само событие. Обратите внимание, что это просто данные: что произошло, с каким героем и в какой экспедиции:

**`labs/lab3/Events/GameEvent.cs`**

```csharp
using Lab3.Heroes;
using Lab3.World;
namespace Lab3.Events;
public sealed record GameEvent(GameEventType Type, Hero Hero, ExpeditionReport Expedition);
```

Интерфейс для подписчиков содержит всего один метод, который вызывается при наступлении события, и определяет контракт обработки этого
изменения:

**`labs/lab3/Events/IGameEventListener.cs`**

```csharp
namespace Lab3.Events;
public interface IGameEventListener {
    void OnEvent(GameEvent gameEvent);
}
```

Именно издатель позволяет подписываться на определенный тип события. Также именно он оповещает подписчиков о
наступлении того или иного события:

**`labs/lab3/Events/GameEventPublisher.cs`**

```csharp
namespace Lab3.Events;
public sealed class GameEventPublisher {
    private readonly Dictionary<GameEventType, List<IGameEventListener>> listeners = new();
    public void Subscribe(GameEventType type, IGameEventListener listener) {
        if (!listeners.TryGetValue(type, out var list))
            listeners[type] = list = new();
        list.Add(listener);
    }
    public void Unsubscribe(GameEventType type, IGameEventListener listener) {
        if (listeners.TryGetValue(type, out var list))
            list.Remove(listener);
    }
    public void Publish(GameEvent gameEvent) {
        if (listeners.TryGetValue(gameEvent.Type, out var list))
            foreach (var listener in list.ToArray())
                listener.OnEvent(gameEvent);
    }
}
```

Подписчики хранятся в `listeners`, причем хранятся по типу ивента, то есть подписаться и реагировать они могут на отдельные типы, а не на
все.
Реакции вызываются в методе `Publish`. Когда в нашей системе происходит ивент – мы вызываем издателя, а он, по списку зарегистрированных
подписчиков, вызывает их
`OnEvent` методы, прокидывая туда информацию о ивенте.

Теперь реализуем двух конкретных подписчиков. Сначала того, который будет выдавать достижения:

**`labs/lab3/Progress/AchievementTracker.cs`**

```csharp
using Lab3.Events;
namespace Lab3.Progress;
public sealed class AchievementTracker : IGameEventListener {
    public void OnEvent(GameEvent e) {
        if (e.Type == GameEventType.Victory) {
            e.Hero.Unlock(Achievement.FirstBlood);
            if (e.Expedition.Battle.Enemy.Code == "dragon")
                e.Hero.Unlock(Achievement.DragonSlayer);
            if (e.Expedition.Rewards.Loot?.Enchantments.Count > 0)
                e.Hero.Unlock(Achievement.Lucky);
        } else if (e.Type == GameEventType.LevelUp && e.Hero.Level >= 5)
            e.Hero.Unlock(Achievement.Veteran);
    }
}
```

А теперь второго, он будет записывать экспедиции в журнал:

**`labs/lab3/Progress/ExpeditionJournal.cs`**

```csharp
using Lab3.Events;
using Lab3.World;
namespace Lab3.Progress;
public sealed class ExpeditionJournal : IGameEventListener {
    private readonly Dictionary<string, List<ExpeditionReport>> entries = new();
    public void OnEvent(GameEvent e) {
        if (!entries.TryGetValue(e.Hero.Name, out var list))
            entries[e.Hero.Name] = list = new();
        list.Add(e.Expedition);
    }
    public IReadOnlyList<ExpeditionReport> GetEntries(string name) => entries.TryGetValue(name, out var list)
                                                                          ? list.ToArray()
                                                                          : [];
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

Теперь займёмся защитой. Сейчас она у каждого бойца своя и зашита прямо в код: герой учитывает броню в методе `ReceiveHit`, а
скелет и дракон переопределяют метод `ReduceDamage`. Мы же хотим добавить новые механики: следопыт должен иногда
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

**`labs/lab3/Defense/DamageHandler.cs`**

```csharp
namespace Lab3.Defense;

public abstract class DamageHandler {

    private DamageHandler? next;

    public DamageHandler SetNext(DamageHandler next) {
        this.next = next;
        return next;
    }

    public abstract void Handle(Damage damage);

    protected void PassToNext(Damage damage) {
        if (next is not null) {
            next.Handle(damage);
        }
    }
}
```

Теперь конкретный обработчик, который будет хендлить уклонение: если оно сработало, удар по герою не прошел, и передавать урон дальше по
цепочке незачем:

**`labs/lab3/Defense/EvasionHandler.cs`**

```csharp
using Lab3.Core;

namespace Lab3.Defense;

public class EvasionHandler : DamageHandler {

    private readonly double chance;

    public EvasionHandler(double chance) {
        this.chance = chance;
    }

    public override void Handle(Damage damage) {
        if (Dice.Instance.Chance(chance)) {
            damage.Evade();
            return;
        }

        PassToNext(damage);
    }
}
```

Броня гасит долю урона и передает то, что осталось, следующему обработчику. Обратите внимание, что это то же самое правило,
что раньше было зашито в герое, скелете и драконе, а теперь оно записано в одном месте:

**`labs/lab3/Defense/ArmorHandler.cs`**

```csharp
namespace Lab3.Defense;

public class ArmorHandler : DamageHandler {

    private readonly double defense;

    public ArmorHandler(double defense) {
        this.defense = defense;
    }

    public override void Handle(Damage damage) {
        int reduced = (int)Math.Round(damage.Amount * (1 - defense), MidpointRounding.AwayFromZero);
        damage.ReduceTo(Math.Max(1, reduced));
        PassToNext(damage);
    }
}
```

И магический барьер, который поглощает урон, пока у него есть запас прочности:

**`labs/lab3/Defense/BarrierHandler.cs`**

```csharp
namespace Lab3.Defense;

public class BarrierHandler : DamageHandler {

    private int strength;

    public BarrierHandler(int strength) {
        this.strength = strength;
    }

    public override void Handle(Damage damage) {
        int absorbed = Math.Min(strength, damage.Amount);
        if (absorbed > 0) {
            strength -= absorbed;
            damage.Absorb(absorbed);
        }

        PassToNext(damage);
    }
}
```

Какие звенья будут в цепочке, зависит от героя. Цепочку собирает его класс - так же, как класс сам знает свою
фабрику стартового снаряжения:

**`labs/lab3/Heroes/HeroClass.cs`**

```csharp
using Lab3.Defense;
using Lab3.Core;
using Lab3.Items;
using Lab3.Kits;

namespace Lab3.Heroes;

// перечисление (enum) в C# не может хранить данные, поэтому классы героя - это
// набор заранее созданных объектов, а создать новые снаружи нельзя: конструктор приватный
public sealed class HeroClass {
    // лямбда - функция, которая создает фабрику стартового снаряжения
    public static readonly HeroClass Warrior =
        new("warrior", "Воин", 120, catalog => new WarriorKitFactory(catalog));
    public static readonly HeroClass Ranger =
        new("ranger", "Следопыт", 100, catalog => new RangerKitFactory(catalog));
    public static readonly HeroClass Mage = new("mage", "Маг", 85, catalog => new MageKitFactory(catalog));

    // все классы героя по порядку
    public static readonly IReadOnlyList<HeroClass> Values = [Warrior, Ranger, Mage];

    private readonly Func<ItemCatalog, IStarterKitFactory> starterKit;

    private HeroClass(string code, string displayName, int baseHealth,
                      Func<ItemCatalog, IStarterKitFactory> starterKit) {
        Code = code;
        DisplayName = displayName;
        BaseHealth = baseHealth;
        this.starterKit = starterKit;
    }

    public string Code { get; }

    public string DisplayName { get; }

    public int BaseHealth { get; }

    public static HeroClass ByCode(string code) {
        return Values.FirstOrDefault(heroClass => heroClass.Code == code) ??
               throw new UnknownCodeException("Класс героя", code);
    }

    // каждый класс сам знает свою фабрику стартового снаряжения
    public IStarterKitFactory GetStarterKit(ItemCatalog catalog) {
        return starterKit(catalog);
    }
    public DamageHandler CreateDefense(Armor armor, int level) {
        double evasion = this == Ranger ? 0.15 : 0;
        int barrier = this == Mage ? 30 : 0;
        if (this == Warrior && level >= 3)
            barrier = 20;
        else if (this == Ranger && level >= 5)
            barrier = 15;
        else if (this == Mage && level >= 3)
            evasion = 0.10;
        DamageHandler armorHandler = new ArmorHandler(armor.Defense), first = armorHandler;
        if (evasion > 0) {
            first = new EvasionHandler(evasion);
            first.SetNext(armorHandler);
        }
        if (barrier > 0)
            armorHandler.SetNext(new BarrierHandler(barrier));
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

**`labs/lab3/Heroes/Hero.cs`**

```csharp
using Lab3.Defense;
using Lab3.Progress;
using Lab3.Combat;
using Lab3.Items;
using Lab3.Kits;

namespace Lab3.Heroes;

public sealed class Hero : ICombatant {
    private const int MaxNameLength = 20;
    private const int StartingGold = 50;
    private const int HealthPerLevel = 15;
    private const int DamagePerLevel = 3;
    private const int ExperiencePerLevel = 100;

    private readonly List<Item> inventory = new();
    private readonly List<Achievement> achievements;
    private DamageHandler defense = null!;
    public IReadOnlyList<Achievement> Achievements => achievements.AsReadOnly();

    // конструктор приватный и принимает в себя билдера, из которого и берет значения полей
    private Hero(Builder builder) {
        Name = builder.Name!;
        HeroClass = builder.HeroClass!;
        Level = builder.Level;
        Gold = builder.Gold;
        Weapon = builder.Weapon!;
        Armor = builder.Armor!;
        Experience = builder.Experience;
        inventory.AddRange(builder.Inventory);
        achievements = builder.Achievements.Distinct().ToList();
        Rest();
    }

    // внутренний класс билдер - чтобы удобно и безопасно создавать героев
    public sealed class Builder {
        internal string? Name { get; private set; }

        internal HeroClass? HeroClass { get; private set; }

        internal Weapon? Weapon { get; private set; }

        internal Armor? Armor { get; private set; }

        // необязательные поля сразу получают значения по умолчанию
        internal int Level { get; private set; } = 1;

        internal int Gold { get; private set; } = StartingGold;

        internal int Experience { get; private set; }
        internal List<Item> Inventory { get; private set; } = new();
        internal List<Achievement> Achievements { get; private set; } = new();
        public Builder SetExperience(int value) {
            Experience = value;
            return this;
        }
        public Builder SetInventory(IEnumerable<Item> items) {
            Inventory = items.ToList();
            return this;
        }
        public Builder SetAchievements(IEnumerable<Achievement> items) {
            Achievements = items.ToList();
            return this;
        }

        public Builder SetName(string name) {
            Name = name;
            return this;
        }

        public Builder SetHeroClass(HeroClass heroClass) {
            HeroClass = heroClass;
            return this;
        }

        public Builder SetWeapon(Weapon weapon) {
            Weapon = weapon;
            return this;
        }

        public Builder SetArmor(Armor armor) {
            Armor = armor;
            return this;
        }

        // берет оружие и броню из стартового набора
        public Builder SetStarterKit(IStarterKitFactory starterKit) {
            Weapon = starterKit.CreateWeapon();
            Armor = starterKit.CreateArmor();
            return this;
        }

        public Builder SetLevel(int level) {
            Level = level;
            return this;
        }

        public Builder SetGold(int gold) {
            Gold = gold;
            return this;
        }

        // перед созданием героя проверяем, что все обязательные поля заполнены корректно
        public Hero Build() {
            if (string.IsNullOrWhiteSpace(Name)) {
                throw new ArgumentException("У героя должно быть имя");
            }
            if (Name.Length > MaxNameLength) {
                throw new ArgumentException($"Имя героя не может быть длиннее {MaxNameLength} символов");
            }
            if (HeroClass is null) {
                throw new ArgumentException("Не выбран класс героя");
            }
            if (Weapon is null || Armor is null) {
                throw new ArgumentException("Герою нужны оружие и броня");
            }
            if (Level < 1) {
                throw new ArgumentException("Уровень героя не может быть меньше 1");
            }
            if (Gold < 0) {
                throw new ArgumentException("Золота не может быть меньше нуля");
            }

            if (Experience < 0)
                throw new ArgumentException("Опыт не может быть отрицательным");
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

    public Hit RollDamage() {
        Hit hit = Weapon.RollDamage();
        return hit with { Damage = hit.Damage + (Level - 1) * DamagePerLevel };
    }

    public Damage ReceiveHit(int amount) {
        Damage damage = new(amount);
        defense.Handle(damage);
        Health = Math.Max(0, Health - damage.Amount);
        return damage;
    }

    // поведение героя между экспедициями

    // перед каждой экспедицией герой успевает отдохнуть
    public void Rest() {
        Health = MaxHealth;
        defense = HeroClass.CreateDefense(Armor, Level);
    }

    // начисляет опыт, возвращает количество полученных уровней
    public int GainExperience(int amount) {
        Experience += amount;
        int levelsGained = 0;
        while (Experience >= ExperienceToNextLevel) {
            Experience -= ExperienceToNextLevel;
            Level++;
            levelsGained++;
        }

        return levelsGained;
    }

    public void AddGold(int amount) {
        Gold += amount;
    }

    public void SpendGold(int amount) {
        if (amount > Gold) {
            throw new InvalidOperationException($"Не хватает золота: нужно {amount}, есть {Gold}");
        }

        Gold -= amount;
    }

    public void AddItem(Item item) {
        inventory.Add(item);
    }

    public void RemoveItem(Item item) {
        if (!inventory.Remove(item)) {
            throw new ArgumentException($"В инвентаре нет предмета {item.Name}");
        }
    }

    // экипирует предмет из инвентаря, снятый предмет возвращается в инвентарь
    public void Equip(Item item) {
        switch (item) {
        case Weapon newWeapon:
            RemoveItem(newWeapon);
            inventory.Add(Weapon);
            Weapon = newWeapon;
            break;
        case Armor newArmor:
            RemoveItem(newArmor);
            inventory.Add(Armor);
            Armor = newArmor;
            defense = HeroClass.CreateDefense(Armor, Level);
            break;
        default:
            throw new ArgumentException($"Предмет нельзя экипировать: {item.Name}");
        }
    }
    public bool Unlock(Achievement achievement) {
        if (achievements.Contains(achievement))
            return false;
        achievements.Add(achievement);
        return true;
    }
}
```

У врагов цепочка задается прямо в конструкторе. Общий класс врага пропускает урон через цепочку, если она есть:

**`labs/lab3/Enemies/AbstractEnemy.cs`**

```csharp
using Lab3.Defense;
using Lab3.Combat;
using Lab3.Core;

namespace Lab3.Enemies;

// общая логика для врагов: здоровье, урон в диапазоне, награда
public abstract class AbstractEnemy : IEnemy {
    private readonly int minDamage;
    private readonly int maxDamage;
    private readonly int goldReward;
    private readonly DamageHandler? defense;

    protected AbstractEnemy(string code, string name, int maxHealth, int minDamage, int maxDamage,
                            int experienceReward, int goldReward, DamageHandler? defense = null) {
        Code = code;
        Name = name;
        MaxHealth = maxHealth;
        Health = maxHealth;
        ExperienceReward = experienceReward;
        this.minDamage = minDamage;
        this.maxDamage = maxDamage;
        this.goldReward = goldReward;
        this.defense = defense;
    }

    public string Code { get; }

    public string Name { get; }

    public int Health { get; private set; }

    public int MaxHealth { get; }

    public bool IsAlive => Health > 0;

    public int ExperienceReward { get; }

    // virtual - наследники могут переопределить этот метод
    public virtual Hit RollDamage() {
        return new Hit(Dice.Instance.Roll(minDamage, maxDamage), false);
    }

    public Damage ReceiveHit(int amount) {
        Damage damage = new(amount);
        defense?.Handle(damage);
        Health = Math.Max(0, Health - damage.Amount);
        return damage;
    }

    public int RollGoldReward() {
        return Dice.Instance.Roll(goldReward / 2, goldReward);
    }
}
```

У гоблина защиты нет:

**`labs/lab3/Enemies/Goblin.cs`**

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

Кости скелета и чешуя дракона работают как броня, поэтому метод `ReduceDamage` им больше не нужен:

**`labs/lab3/Enemies/Skeleton.cs`**

```csharp
using Lab3.Defense;
namespace Lab3.Enemies;

// скелет: кости плохо поддаются ударам, поэтому часть урона проходит мимо
public sealed class Skeleton : AbstractEnemy {
    private const double Resistance = 0.3;

    public Skeleton() : base("skeleton", "Скелет", 110, 10, 24, 90, 70, new ArmorHandler(Resistance)) {}
}
```

**`labs/lab3/Enemies/Dragon.cs`**

```csharp
using Lab3.Defense;
using Lab3.Combat;
using Lab3.Core;

namespace Lab3.Enemies;

// дракон защищен чешуей и время от времени дышит огнем
public sealed class Dragon : AbstractEnemy {
    private const double Resistance = 0.25;
    private const double FireBreathChance = 0.25;
    private const int FireBreathMultiplier = 2;

    public Dragon() : base("dragon", "Дракон", 260, 16, 22, 500, 400, new ArmorHandler(Resistance)) {}

    public override Hit RollDamage() {
        Hit bite = base.RollDamage();
        if (Dice.Instance.Chance(FireBreathChance)) {
            return new Hit(bite.Damage * FireBreathMultiplier, true);
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

Посмотрите на метод `Game.Run` из прошлой лабы: он сам создает каталог, локации и торговца, сам собирает героя строителем
и сам берет у класса фабрику стартового снаряжения. Теперь к этому добавятся тактики, хранилище с заместителем,
издатель событий и подписчики. Если оставить все это в консольном коде, он будет знать обо всех классах игры, а
заменить консоль на что-то другое (спойлер-спойлер) станет очень непросто.

Поэтому давайте воспользуемся паттерном *Фасад*. Он предоставляет простой интерфейс к сложной подсистеме из многих
классов: вся настройка и все связи между этими классами спрятаны внутри фасада, а снаружи видны только несколько
понятных методов.

Наш фасад - это вся игра целиком:

**`labs/lab3/GameServices/GameFacade.cs`**

```csharp
using Lab3.Camp;
using Lab3.Combat;
using Lab3.Heroes;
using Lab3.Items;
using Lab3.World;
using Lab3.Tactics;
using Lab3.Events;
using Lab3.Progress;
using Lab3.Storage;
namespace Lab3.GameServices;
public sealed class GameFacade {
    private readonly ItemCatalog catalog = DefaultCatalog.Create();
    private readonly LocationRegistry registry;
    private readonly Merchant merchant;
    private readonly IHeroRepository heroes;
    private readonly GameEventPublisher events = new();
    private readonly ExpeditionJournal journal = new();
    public IReadOnlyList<ITactic> Tactics { get; } =
        [new BalancedTactic(), new AggressiveTactic(), new CautiousTactic()];
    public GameFacade(string saveFile) {
        registry = new LocationRegistry(
            [new Forest(catalog), new HauntedManor(catalog), new Crypt(catalog), new DragonLair(catalog)]);
        merchant = new Merchant(catalog, [
            DefaultCatalog.IronSword, DefaultCatalog.ElvenBow, DefaultCatalog.EmberStaff,
            DefaultCatalog.Chainmail, DefaultCatalog.PlateArmor
        ]);
        heroes = new CachingHeroRepository(new FileHeroRepository(saveFile, catalog));
        var achievements = new AchievementTracker();
        events.Subscribe(GameEventType.Victory, achievements);
        events.Subscribe(GameEventType.LevelUp, achievements);
        events.Subscribe(GameEventType.Victory, journal);
        events.Subscribe(GameEventType.Defeat, journal);
    }
    public IReadOnlyList<HeroClass> HeroClasses => HeroClass.Values;
    public IReadOnlyList<Item> GetStarterKit(string code) {
        var kit = HeroClass.ByCode(code).GetStarterKit(catalog);
        return [kit.CreateWeapon(), kit.CreateArmor()];
    }
    public IReadOnlyList<Location> Locations => registry.All;
    public Location GetLocation(string code) => registry.Get(code);
    public ITactic GetTactic(string code) => Tactics.FirstOrDefault(t => t.Code == code) ??
                                             throw new ArgumentException("Неизвестная тактика: " + code);
    public IReadOnlyList<Item> Showcase => merchant.Showcase;
    public int GetSellPrice(Item item) => merchant.GetSellPrice(item);
    public IReadOnlyList<Hero> Heroes => heroes.FindAll();
    public Hero GetHero(string name) => heroes.FindByName(name) ??
                                        throw new ArgumentException("Героя с именем " + name + " нет");
    public Hero CreateHero(string name, string code) {
        if (heroes.FindByName(name) is not null)
            throw new ArgumentException("Герой с именем " + name + " уже есть");
        var heroClass = HeroClass.ByCode(code);
        var hero = new Hero.Builder()
                       .SetName(name)
                       .SetHeroClass(heroClass)
                       .SetStarterKit(heroClass.GetStarterKit(catalog))
                       .Build();
        heroes.Save(hero);
        return hero;
    }
    public ExpeditionReport Explore(string name, string locationCode, string tacticCode) {
        var hero = GetHero(name);
        var report = GetLocation(locationCode).Explore(hero, GetTactic(tacticCode));
        var result = report.Battle.Outcome == Outcome.Victory ? GameEventType.Victory : GameEventType.Defeat;
        events.Publish(new GameEvent(result, hero, report));
        if (report.Rewards.LevelsGained > 0)
            events.Publish(new GameEvent(GameEventType.LevelUp, hero, report));
        heroes.Save(hero);
        return report;
    }
    public Item Equip(string name, int index) {
        var hero = GetHero(name);
        var item = hero.Inventory[index];
        hero.Equip(item);
        heroes.Save(hero);
        return item;
    }
    public Item Buy(string name, string code) {
        var hero = GetHero(name);
        var item = merchant.Buy(hero, code);
        heroes.Save(hero);
        return item;
    }
    public int Sell(string name, int index) {
        var hero = GetHero(name);
        int price = merchant.Sell(hero, hero.Inventory[index]);
        heroes.Save(hero);
        return price;
    }
    public IReadOnlyList<ExpeditionReport> GetJournal(string name) => journal.GetEntries(name);
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

**`labs/lab3/ConsoleView.cs`**

```csharp
using Lab3.Combat;
using Lab3.Heroes;
using Lab3.Items;
using Lab3.World;

namespace Lab3;

// весь вывод в консоль собран здесь
public sealed class ConsoleView {
    public string Describe(Item item) {
        string description = item.Name;

        if (item.Enchantments.Count > 0) {
            var names = new List<string>();
            foreach (Enchantment enchantment in item.Enchantments) {
                names.Add(enchantment.Name);
            }
            description += " [" + string.Join(", ", names) + "]";
        }

        return description + " (" + item.Description + ")";
    }

    public void PrintHero(Hero hero) {
        Console.WriteLine(
            $"{hero.Name}, {hero.HeroClass.DisplayName} {hero.Level} ур. | здоровье {hero.MaxHealth} | " +
            $"опыт {hero.Experience}/{hero.ExperienceToNextLevel} | золото {hero.Gold}");
        Console.WriteLine("  Оружие: " + Describe(hero.Weapon));
        Console.WriteLine("  Броня:  " + Describe(hero.Armor));
        if (hero.Achievements.Count > 0)
            Console.WriteLine("  Достижения: " +
                              string.Join(", ", hero.Achievements.Select(a => a.DisplayName)));
    }

    public string Summarize(ExpeditionReport report, string locationName, string tacticName) {
        string result = report.Battle.Outcome == Outcome.Victory ? "победа" : "поражение";
        return $"{locationName}, {report.Battle.Enemy.Name}, тактика \"{tacticName}\": {result}";
    }

    public void PrintReport(ExpeditionReport report, string locationName) {
        BattleReport battle = report.Battle;
        Console.WriteLine($"=== {locationName} (сид {report.Seed}) ===");
        Console.WriteLine(
            $"{battle.Hero.Name} встречает врага: {battle.Enemy.Name} ({battle.Enemy.MaxHealth} здоровья)");

        foreach (BattleEvent battleEvent in battle.Events) {
            PrintEvent(battleEvent, battle);
        }

        PrintRewards(battle, report.Rewards);
    }

    private void PrintEvent(BattleEvent battleEvent, BattleReport battle) {
        Participant source = GetParticipant(battleEvent.Source, battle);
        Participant target = GetParticipant(battleEvent.Target, battle);

        switch (battleEvent.Type) {
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
        case EventType.Evade:
            Console.WriteLine(
                $"[раунд {battleEvent.Round}] {source.Name} -> {target.Name}: промах, {target.Name} уклоняется");
            break;
        case EventType.Absorb:
            Console.WriteLine(
                $"[раунд {battleEvent.Round}] {source.Name} -> {target.Name}: барьер поглощает {battleEvent.Amount} урона");
            break;
        case EventType.PowerStrike:
            Console.WriteLine($"[раунд {battleEvent.Round}] {source.Name} бьет в полную силу");
            break;
        case EventType.Defend:
            Console.WriteLine($"[раунд {battleEvent.Round}] {source.Name} уходит в оборону");
            break;
        case EventType.Death:
            Console.WriteLine($"{target.Name} повержен!");
            break;
        }
    }

    private void PrintRewards(BattleReport battle, Rewards rewards) {
        if (battle.Outcome == Outcome.Defeat) {
            Console.WriteLine(
                $"Поражение... {battle.Hero.Name} едва уносит ноги и теряет {-rewards.Gold} золота");
            return;
        }

        Console.WriteLine($"Победа! +{rewards.Experience} опыта, +{rewards.Gold} золота");
        if (rewards.Loot is not null) {
            Console.WriteLine("Добыча: " + Describe(rewards.Loot));
        }
        if (rewards.LevelsGained > 0) {
            Console.WriteLine("Новый уровень!");
        }
    }

    private static Participant GetParticipant(Side side, BattleReport battle) {
        return side == Side.Hero ? battle.Hero : battle.Enemy;
    }
}
```

А игровой цикл в методе `Game.Run` теперь работает только с фасадом и хранит текущего героя:

Вместо прямой работы с каталогом, локациями и торговцем:

- создаём `GameFacade` с файлом сохранения и выбираем нового или сохранённого героя;
- перед экспедицией выбираем тактику;
- работам с экспедициями, экипировкой и торговлей через фасад;
- добавляем пункт меню для журнала экспедиций.

**`labs/lab3/Program.cs`**

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

**`labs/lab3/Game.cs`**

```csharp
using Lab3.GameServices;
using Lab3.Camp;
using Lab3.Combat;
using Lab3.Heroes;
using Lab3.Items;
using Lab3.Kits;
using Lab3.World;

namespace Lab3;

public static class Game {
    private static readonly ConsoleView View = new();

    public static void Run() {
        string saveFile = Path.Combine("saves", "heroes.json");
        Console.WriteLine("Файл сохранения: " + Path.GetFullPath(saveFile));
        var game = new GameFacade(saveFile);
        Hero hero = ChooseHero(game);
        Console.WriteLine($"{hero.Name} разбивает лагерь на опушке мистического леса.");

        while (true) {
            Console.WriteLine();
            View.PrintHero(hero);
            Console.WriteLine(
                "1 - отправиться в экспедицию, 2 - снаряжение, 3 - торговец, 4 - журнал, 0 - выйти из игры");
            switch (ReadLine()) {
            case "1":
                GoOnExpedition(hero, game);
                break;
            case "2":
                ManageEquipment(hero, game);
                break;
            case "3":
                VisitMerchant(hero, game);
                break;
            case "4":
                foreach (var report in game.GetJournal(hero.Name))
                    Console.WriteLine(View.Summarize(report,
                                                     game.GetLocation(report.LocationCode).DisplayName,
                                                     game.GetTactic(report.TacticCode).DisplayName));
                break;
            case "0":
                return;
            default:
                Console.WriteLine("Такого варианта нет");
                break;
            }
        }
    }

    private static Hero ChooseHero(GameFacade game) {
        var heroes = game.Heroes;
        if (heroes.Count == 0)
            return CreateHero(game);
        Console.WriteLine("Выберите сохраненного героя (0 - создать нового):");
        for (int i = 0; i < heroes.Count; i++)
            Console.WriteLine(
                $"{i + 1} - {heroes[i].Name}, {heroes[i].HeroClass.DisplayName} {heroes[i].Level} ур.");
        int index = ReadChoice(heroes.Count);
        return index >= 0 ? heroes[index] : CreateHero(game);
    }

    private static Hero CreateHero(GameFacade game) {
        Console.WriteLine("Создайте своего героя!");
        while (true) {
            Console.WriteLine("Выберите класс:");
            IReadOnlyList<HeroClass> heroClasses = HeroClass.Values;
            for (int i = 0; i < heroClasses.Count; i++) {
                var kit = game.GetStarterKit(heroClasses[i].Code);
                Console.WriteLine(
                    $"{i + 1} - {heroClasses[i].DisplayName} (здоровье {heroClasses[i].BaseHealth}): " +
                    $"{View.Describe(kit[0])}, {View.Describe(kit[1])}");
            }
            int classIndex = ReadChoice(heroClasses.Count);
            if (classIndex < 0) {
                continue;
            }
            HeroClass heroClass = heroClasses[classIndex];

            Console.WriteLine("Введите имя:");
            string name = ReadLine();

            try {
                return game.CreateHero(name, heroClass.Code);
            } catch (ArgumentException e) {
                Console.WriteLine("Не получилось создать героя: " + e.Message);
            }
        }
    }

    private static void GoOnExpedition(Hero hero, GameFacade game) {
        IReadOnlyList<Location> all = game.Locations;
        Console.WriteLine("Куда отправимся?");
        for (int i = 0; i < all.Count; i++) {
            Console.WriteLine(
                $"{i + 1} - {all[i].DisplayName} (рекомендуемый уровень {all[i].RecommendedLevel})");
        }
        int index = ReadChoice(all.Count);
        if (index < 0) {
            return;
        }

        Location location = all[index];
        Console.WriteLine("Выберите тактику:");
        for (int i = 0; i < game.Tactics.Count; i++)
            Console.WriteLine($"{i + 1} - {game.Tactics[i].DisplayName}: {game.Tactics[i].Description}");
        int tacticIndex = ReadChoice(game.Tactics.Count);
        if (tacticIndex < 0)
            return;
        ExpeditionReport report = game.Explore(hero.Name, location.Code, game.Tactics[tacticIndex].Code);
        Console.WriteLine();
        View.PrintReport(report, location.DisplayName);

        if (report.Battle.Outcome == Outcome.Victory && report.Battle.Enemy.Code == "dragon") {
            Console.WriteLine(
                $"\nДракон повержен! Барды еще долго будут петь о подвиге героя по имени {hero.Name}.");
        }
    }

    private static void ManageEquipment(Hero hero, GameFacade game) {
        // копируем список: после Equip инвентарь героя изменится
        List<Item> inventory = hero.Inventory.ToList();
        if (inventory.Count == 0) {
            Console.WriteLine("Инвентарь пуст");
            return;
        }

        Console.WriteLine("Что экипировать? (0 - назад)");
        PrintItems(inventory);
        int index = ReadChoice(inventory.Count);
        if (index < 0) {
            return;
        }

        Item item = inventory[index];
        game.Equip(hero.Name, index);
        Console.WriteLine("Экипировано: " + View.Describe(item));
    }

    private static void VisitMerchant(Hero hero, GameFacade game) {
        Console.WriteLine($"Торговец: \"Чего желаешь?\" (золото: {hero.Gold})");
        Console.WriteLine("1 - купить, 2 - продать, 0 - назад");
        switch (ReadLine()) {
        case "1":
            Buy(hero, game);
            break;
        case "2":
            Sell(hero, game);
            break;
        }
    }

    private static void Buy(Hero hero, GameFacade game) {
        IReadOnlyList<Item> showcase = game.Showcase;
        for (int i = 0; i < showcase.Count; i++) {
            Console.WriteLine($"{i + 1} - {View.Describe(showcase[i])}: {showcase[i].Price} золота");
        }
        int index = ReadChoice(showcase.Count);
        if (index < 0) {
            return;
        }

        try {
            Item item = game.Buy(hero.Name, showcase[index].Code);
            Console.WriteLine("Куплено: " + View.Describe(item));
        } catch (InvalidOperationException e) {
            Console.WriteLine(e.Message);
        }
    }

    private static void Sell(Hero hero, GameFacade game) {
        List<Item> inventory = hero.Inventory.ToList();
        if (inventory.Count == 0) {
            Console.WriteLine("Продавать нечего (экипированные предметы продать нельзя)");
            return;
        }

        for (int i = 0; i < inventory.Count; i++) {
            Console.WriteLine(
                $"{i + 1} - {View.Describe(inventory[i])}: {game.GetSellPrice(inventory[i])} золота");
        }
        int index = ReadChoice(inventory.Count);
        if (index < 0) {
            return;
        }

        Item item = inventory[index];
        int price = game.Sell(hero.Name, index);
        Console.WriteLine($"Продано: {item.Name} за {price} золота");
    }

    private static void PrintItems(IReadOnlyList<Item> items) {
        for (int i = 0; i < items.Count; i++) {
            Console.WriteLine($"{i + 1} - {View.Describe(items[i])}");
        }
    }

    // читает строку без пробелов по краям; если ввод закончился, завершает программу
    private static string ReadLine() {
        string? line = Console.ReadLine();
        if (line is null) {
            Environment.Exit(0);
        }

        return line.Trim();
    }

    // читает номер варианта от 1 до count, возвращает индекс или -1, если ввели что-то другое
    private static int ReadChoice(int count) {
        if (int.TryParse(ReadLine(), out int choice) && choice >= 1 && choice <= count) {
            return choice - 1;
        }

        return -1;
    }
}
```

Супер! Попробуйте запустить игру, сохраниться и вернуться к своему герою :)

## Демонстрация паттернов

Чтобы показать все паттерны одним запуском, напишем класс `Demo`:

Демонстрацию лабы 3 заменяем новой:

**`labs/lab3/Demo.cs`**

```csharp
using Lab3.Core;
using Lab3.Combat;
using Lab3.Defense;
using Lab3.Enemies;
using Lab3.Heroes;
using Lab3.Items;
using Lab3.Tactics;
using Lab3.World;
using Lab3.Events;
using Lab3.Progress;
using Lab3.Storage;
using Lab3.GameServices;
namespace Lab3;
public static class Demo {
    private static readonly ConsoleView View = new();
    private static Hero NewHero(string name, HeroClass heroClass, int level,
                                ItemCatalog catalog) => new Hero.Builder()
                                                            .SetName(name)
                                                            .SetHeroClass(heroClass)
                                                            .SetStarterKit(heroClass.GetStarterKit(catalog))
                                                            .SetLevel(level)
                                                            .Build();
    private static void PrintHits(string title, DamageHandler chain, int count) {
        var results = new List<string>();
        for (int i = 0; i < count; i++) {
            var damage = new Damage(30);
            chain.Handle(damage);
            results.Add(damage.IsEvaded       ? "уклонение"
                        : damage.Absorbed > 0 ? $"поглощено {damage.Absorbed}, прошло {damage.Amount}"
                                              : $"прошло {damage.Amount}");
        }
        Console.WriteLine(title + ": " + string.Join("; ", results));
    }
    private sealed class EventPrinter : IGameEventListener {
        public void OnEvent(GameEvent e) => Console.WriteLine($"Событие {e.Type}: герой {e.Hero.Name}");
    }
    public static void Run() {
        var catalog = DefaultCatalog.Create();
        var dice = Dice.Instance;
        Console.WriteLine("===== Цепочка обязанностей =====");
        PrintHits("Броня 30%", new ArmorHandler(0.3), 2);
        var armor = new ArmorHandler(0.3);
        armor.SetNext(new BarrierHandler(30));
        PrintHits("Броня 30% -> барьер 30", armor, 3);
        dice.Reseed(3);
        var chain = new EvasionHandler(0.5);
        chain.SetNext(new ArmorHandler(0.3)).SetNext(new BarrierHandler(30));
        PrintHits("Уклонение 50% -> броня 30% -> барьер 30", chain, 3);
        Console.WriteLine("\n===== Стратегия =====");
        foreach (ITactic tactic in new ITactic[] { new BalancedTactic(), new AggressiveTactic(),
                                                   new CautiousTactic() }) {
            var mage = NewHero("Мерлин", HeroClass.Mage, 2, catalog);
            var report = new Crypt(catalog).Explore(mage, tactic, 9);
            string result = report.Battle.Outcome == Outcome.Victory ? "победа" : "поражение";
            Console.WriteLine(
                $"{tactic.DisplayName}: {result}, раундов {report.Battle.Events.Last().Round}, у героя осталось {mage.Health}/{mage.MaxHealth} здоровья");
        }
        Console.WriteLine("\n===== Декоратор =====");
        foreach (IEnemy enemy in new IEnemy[] {
                     new Goblin(), new LegendaryEnemy(new Goblin()), new WindfuryEnemy(new Goblin()),
                     new LegendaryEnemy(new WindfuryEnemy(new Goblin())), new ArmoredEnemy(new Goblin()),
                     new ArmoredEnemy(new LegendaryEnemy(new WindfuryEnemy(new Dragon())))
                 }) {
            dice.Reseed(5);
            string hits = string.Join(", ", Enumerable.Range(0, 3).Select(
                                                _ => enemy.RollDamage().Damage));
            var damage = enemy.ReceiveHit(20);
            Console.WriteLine(
                $"{enemy.Name}: удары {hits}, награда {enemy.ExperienceReward} опыта; входящий урон 20 -> {damage.Amount}, здоровье {enemy.Health}/{enemy.MaxHealth}");
        }
        Console.WriteLine("\n===== Адаптер =====");
        var bow = catalog.CreateWeapon(DefaultCatalog.ElvenBow);
        var haunted = new HauntedWeapon(bow);
        Console.WriteLine("Предмет: " + View.Describe(bow));
        Console.WriteLine(
            $"Враг:    {haunted.Name} ({haunted.MaxHealth} здоровья, награда {haunted.ExperienceReward} опыта)");
        var manor = new HauntedManor(catalog);
        View.PrintReport(
            manor.Explore(NewHero("Леголас", HeroClass.Ranger, 2, catalog), new CautiousTactic(), 55),
            manor.DisplayName);
        Console.WriteLine("\n===== Наблюдатель =====");
        var events = new GameEventPublisher();
        var journal = new ExpeditionJournal();
        events.Subscribe(GameEventType.Victory, new AchievementTracker());
        events.Subscribe(GameEventType.Victory, journal);
        events.Subscribe(GameEventType.Defeat, journal);
        events.Subscribe(GameEventType.Victory, new EventPrinter());
        var warrior = NewHero("Артур", HeroClass.Warrior, 1, catalog);
        var expedition = new Forest(catalog).Explore(warrior, 7);
        events.Publish(new GameEvent(expedition.Battle.Outcome == Outcome.Victory ? GameEventType.Victory
                                                                                  : GameEventType.Defeat,
                                     warrior, expedition));
        Console.WriteLine("Достижения героя: " +
                          string.Join(", ", warrior.Achievements.Select(a => a.DisplayName)));
        Console.WriteLine("Записей в журнале: " + journal.GetEntries(warrior.Name).Count);
        string directory = Path.Combine(Path.GetTempPath(), "lab-demo-" + Guid.NewGuid());
        Directory.CreateDirectory(directory);
        try {
            Console.WriteLine("\n===== Заместитель =====");
            string path = Path.Combine(directory, "heroes.json");
            var file = new FileHeroRepository(path, catalog);
            file.Save(warrior);
            Console.WriteLine("Содержимое файла:");
            Console.Write(File.ReadAllText(path));
            Console.WriteLine("Без заместителя два поиска дают один и тот же объект: " +
                              ReferenceEquals(file.FindByName("Артур"), file.FindByName("Артур")));
            var cached = new CachingHeroRepository(file);
            var hero = cached.FindByName("Артур")!;
            Console.WriteLine("С заместителем два поиска дают один и тот же объект: " +
                              ReferenceEquals(hero, cached.FindByName("Артур")));
            hero.AddGold(100);
            cached.Save(hero);
            Console.WriteLine("После сохранения через заместитель золото героя в файле: " +
                              file.FindByName("Артур")!.Gold);
            Console.WriteLine("\n===== Фасад =====");
            var game = new GameFacade(Path.Combine(directory, "game.json"));
            game.CreateHero("Гэндальф", "mage");
            dice.Reseed(11);
            var report = game.Explore("Гэндальф", "forest", "cautious");
            Console.WriteLine("Экспедиция через фасад: " +
                              View.Summarize(report, game.GetLocation(report.LocationCode).DisplayName,
                                             game.GetTactic(report.TacticCode).DisplayName));
            View.PrintHero(game.GetHero("Гэндальф"));
        } finally {
            Directory.Delete(directory, true);
        }
    }
}
```

Вывод программы:

```text
===== Цепочка обязанностей =====
Броня 30%: прошло 21; прошло 21
Броня 30% -> барьер 30: поглощено 21, прошло 0; поглощено 9, прошло 12; прошло 21
Уклонение 50% -> броня 30% -> барьер 30: уклонение; поглощено 21, прошло 0; поглощено 9, прошло 12

===== Стратегия =====
Баланс: победа, раундов 9, у героя осталось 5/100 здоровья
Натиск: победа, раундов 8, у героя осталось 17/100 здоровья
Осторожность: поражение, раундов 9, у героя осталось 0/100 здоровья

===== Декоратор =====
Гоблин: удары 9, 8, 8, награда 40 опыта; входящий урон 20 -> 20, здоровье 30/50
Легендарный Гоблин: удары 18, 16, 16, награда 80 опыта; входящий урон 20 -> 20, здоровье 30/50
Неистовый Гоблин: удары 17, 19, 24, награда 60 опыта; входящий урон 20 -> 20, здоровье 30/50
Легендарный Неистовый Гоблин: удары 34, 38, 48, награда 120 опыта; входящий урон 20 -> 20, здоровье 30/50
Бронированный Гоблин: удары 9, 8, 8, награда 40 опыта; входящий урон 20 -> 10, здоровье 40/50
Бронированный Легендарный Неистовый Дракон: удары 70, 72, 124, награда 1500 опыта; входящий урон 20 -> 8, здоровье 252/260

===== Адаптер =====
Предмет: Эльфийский лук (урон 14, шанс крита 40%)
Враг:    Одержимый Эльфийский лук (70 здоровья, награда 45 опыта)
=== Проклятый особняк (сид 55) ===
Леголас встречает врага: Одержимый Эльфийский лук (70 здоровья)
[раунд 1] Леголас -> Одержимый Эльфийский лук: КРИТ! 23 урона, осталось 47/70
[раунд 1] Одержимый Эльфийский лук -> Леголас: промах, Леголас уклоняется
[раунд 2] Леголас -> Одержимый Эльфийский лук: 13 урона, осталось 34/70
[раунд 2] Одержимый Эльфийский лук -> Леголас: промах, Леголас уклоняется
[раунд 3] Леголас -> Одержимый Эльфийский лук: 13 урона, осталось 21/70
[раунд 3] Одержимый Эльфийский лук -> Леголас: 11 урона, осталось 104/115
[раунд 4] Леголас -> Одержимый Эльфийский лук: 13 урона, осталось 8/70
[раунд 4] Одержимый Эльфийский лук -> Леголас: промах, Леголас уклоняется
[раунд 5] Леголас -> Одержимый Эльфийский лук: 13 урона, осталось 0/70
Одержимый Эльфийский лук повержен!
Победа! +45 опыта, +31 золота

===== Наблюдатель =====
Событие Victory: герой Артур
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
    "gold": 75,
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
После сохранения через заместитель золото героя в файле: 175

===== Фасад =====
Экспедиция через фасад: Мистический лес, Гоблин, тактика "Осторожность": победа
Гэндальф, Маг 1 ур. | здоровье 85 | опыт 40/100 | золото 68
  Оружие: Посох ученика (урон 10-18)
  Броня:  Льняная роба (защита 10%)
  Достижения: Первая кровь
```

Не забудьте закоммитить и запушить изменения!

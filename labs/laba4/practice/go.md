# Лаба 4 — практика на Go

Это практическая часть [лабораторной работы 4](../laba4.md): требования к сдаче и вопросы к защите находятся в основном тексте лабы.

Понадобится Go 1.22 или новее, сторонние библиотеки не нужны. Шаблон репозитория: [tech-prog-labs-2026-go](https://github.com/tidbid-kt/tech-prog-labs-2026-go). Код лабы пишите в папке `labs/lab3`. Все команды запускаются из корня репозитория: игра - `go run ./labs/lab3`, демонстрация - `go run ./labs/lab3/cmd/demo`.

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

В ходе нее мы будем изменять существующие типы и добавлять новые в соответствующие пакеты.
Папка `labs/lab3` и расположение исходников сохраняются; создавать отдельный модуль `lab4` и копировать
типы не нужно. Дальше мы разберем только новые и измененные типы, остальные остаются как в прошлой лабе.

Новые и измененные типы приведены целиком. Ключевые изменения в коде отмечены короткими комментариями.

## Подготовка: данные и контракты

Наши паттерны потребуют немного подготовки перед введением их в действие! Давайте же эту подготовку приведем.

Старого механизма обработки урона нам теперь не хватает. Раньше мы работали просто с числами, теперь давайте масштабироваться.
Заведем отдельную структуру `Damage`. Она будет инкапсулировать в себе результат получения урона
(сколько получили, удалось ли уклонится, сколько было поглощено), эти данные понадобятся нам дальше.

**`labs/lab3/defense/damage.go`**

```go
package defense

type Damage struct {
	Amount   int
	Evaded   bool
	Absorbed int
}

func (d *Damage) ReduceTo(amount int) { d.Amount = amount }
func (d *Damage) Evade()              { d.Amount = 0; d.Evaded = true }
func (d *Damage) Absorb(value int)    { d.Amount -= value; d.Absorbed += value }
```

И теперь `Combatant.ReceiveHit` будет возвращать `Damage` вместо `int`. Так мы сможем реализовать отдельные механики защиты от урона.
Получится что-то такое:

**`labs/lab3/combat/combatant.go`**

```go
package combat

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/defense"
)

// Combatant - всё, что может участвовать в бою: и герой, и враги
type Combatant interface {
	// Code - код вида сущности: класс героя или тип врага
	Code() string
	Name() string
	Health() int
	MaxHealth() int
	IsAlive() bool
	// RollDamage - удар, который будет нанесен данной сущностью
	RollDamage() Hit
	// ReceiveHit - получение удара, возвращает урон, который прошел на самом деле
	ReceiveHit(damage int) defense.Damage
}
```

Для врагов давайте обработку урона в `ReceiveHit` также сделаем через `Damage`, не зря же мы эту структуру добавляли:

**`labs/lab3/enemy/base_enemy.go`**

```go
package enemy

import (
"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/defense"

	"math"

	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
)

// baseEnemy - общая логика для врагов: здоровье, урон в диапазоне, сопротивление и награда.
// Конкретные враги встраивают эту структуру в себя
type baseEnemy struct {
	code      string
	name      string
	maxHealth int
	minDamage int
	maxDamage int
	// доля урона, которую враг игнорирует
	resistance       float64
	experienceReward int
	goldReward       int
	health           int
}

func newBaseEnemy(code, name string, maxHealth, minDamage, maxDamage int, resistance float64, experienceReward, goldReward int) baseEnemy {
	return baseEnemy{code, name, maxHealth, minDamage, maxDamage, resistance, experienceReward, goldReward, maxHealth}
}

func (e *baseEnemy) RollDamage() combat.Hit {
	return combat.Hit{Damage: core.GetInstance().Roll(e.minDamage, e.maxDamage), Critical: false}
}

func (e *baseEnemy) ReceiveHit(damage int) defense.Damage {
	reducedDamage := int(math.Round(float64(damage) * (1 - e.resistance)))
	dealt := max(1, reducedDamage)
	e.health = max(0, e.health-dealt)
	return defense.Damage{Amount: dealt}
}

func (e *baseEnemy) IsAlive() bool { return e.health > 0 }

func (e *baseEnemy) RollGoldReward() int {
	return core.GetInstance().Roll(e.goldReward/2, e.goldReward)
}

func (e *baseEnemy) ExperienceReward() int { return e.experienceReward }

func (e *baseEnemy) Code() string { return e.code }

func (e *baseEnemy) Name() string { return e.name }

func (e *baseEnemy) Health() int { return e.health }

func (e *baseEnemy) MaxHealth() int { return e.maxHealth }
```

В `Battle` добавим обработку результата `ReceiveHit`, с учетом того, что он теперь возвращает `Damage`:

**`labs/lab3/combat/battle.go`**

```go
package combat

// Battle - бой идет по раундам: сначала бьет герой, потом враг, пока кто-то не погибнет
type Battle struct {
	hero   Combatant
	enemy  Combatant
	events []BattleEvent
	round  int
}

func NewBattle(hero, enemy Combatant) *Battle {
	return &Battle{hero: hero, enemy: enemy}
}

func (b *Battle) Run() BattleReport {
	for b.hero.IsAlive() && b.enemy.IsAlive() {
		b.round++
		b.strike(b.hero, SideHero, b.enemy, SideEnemy)
		if b.enemy.IsAlive() {
			b.strike(b.enemy, SideEnemy, b.hero, SideHero)
		}
	}

	outcome := Defeat
	if b.hero.IsAlive() {
		outcome = Victory
	}

	return BattleReport{
		Hero:    ParticipantOf(b.hero),
		Enemy:   ParticipantOf(b.enemy),
		Outcome: outcome,
		Events:  b.events,
	}
}

func (b *Battle) strike(attacker Combatant, source Side, defender Combatant, target Side) {
	hit := attacker.RollDamage()
	dealt := defender.ReceiveHit(hit.Damage).Amount
	eventType := EventHit
	if hit.Critical {
		eventType = EventCriticalHit
	}
	b.emit(BattleEvent{b.round, eventType, source, target, dealt, defender.Health()})

	if !defender.IsAlive() {
		b.emit(BattleEvent{b.round, EventDeath, source, target, 0, 0})
	}
}

func (b *Battle) emit(event BattleEvent) {
	b.events = append(b.events, event)
}
```

**`labs/lab3/combat/battle_report.go`**

```go
package combat

// BattleReport - результат боя: кто сражался, чем все закончилось и что происходило по ходу дела
type BattleReport struct {
	Hero    Participant
	Enemy   Participant
	Outcome Outcome
	Events  []BattleEvent
}
```

А еще у нас в игре появятся достижения. Выдаваться они будут с помощью паттерна "наблюдатель", но объявим мы их сразу здесь.

Представлять собой они будут идентификатор, название и описание. Как-то так:

**`labs/lab3/progress/achievement.go`**

```go
package progress

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
)

type Achievement struct{ code, displayName, description string }

var (
	FirstBlood   = &Achievement{"first-blood", "Первая кровь", "одержать первую победу"}
	Lucky        = &Achievement{"lucky", "Счастливчик", "найти зачарованный лут"}
	Veteran      = &Achievement{"veteran", "Ветеран", "достичь 5 уровня"}
	DragonSlayer = &Achievement{"dragon-slayer", "Драконоборец", "победить дракона"}
)

func (a *Achievement) Code() string        { return a.code }
func (a *Achievement) DisplayName() string { return a.displayName }
func (a *Achievement) Description() string { return a.description }
func ByCode(code string) (*Achievement, error) {
	for _, a := range []*Achievement{FirstBlood, Lucky, Veteran, DragonSlayer} {
		if a.code == code {
			return a, nil
		}
	}
	return nil, &core.UnknownCodeError{What: "Достижение", Code: code}
}
```

В `Hero` добавим хранение достижений, полученных за этого героя, а также возможность восстановления опыта, инвентаря и достижений через
строитель.

Нам это понадобится, когда будем добавлять механику сохранений!

А еще метод отдыха переименуем в `Rest` вместо `RestoreHealth`. В нем позже появится дополнительное поведение, поэтому стоит сразу немного
расширить контракт.

Итог выглядит вот так:

**`labs/lab3/hero/hero.go`**

```go
package hero

import (
"math"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/defense"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/progress"

	"errors"
	"fmt"
	"slices"
	"strings"
	"unicode/utf8"

	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/kit"
)

const (
	maxNameLength      = 20
	startingGold       = 50
	healthPerLevel     = 15
	damagePerLevel     = 3
	experiencePerLevel = 100
)

// Hero - герой. Все поля неэкспортируемые, поэтому вне пакета hero собрать героя можно только через строителя
type Hero struct {
	name         string
	heroClass    *HeroClass
	level        int
	experience   int
	gold         int
	health       int
	weapon       item.Weapon
	armor        *item.Armor
	inventory    []item.Item
	achievements []*progress.Achievement
}

// Builder - строитель, чтобы удобно и безопасно создавать героев
type Builder struct {
	name         string
	heroClass    *HeroClass
	weapon       item.Weapon
	armor        *item.Armor
	level        int
	gold         int
	experience   int
	inventory    []item.Item
	achievements []*progress.Achievement
	// первая ошибка, случившаяся при заполнении полей; ее вернет Build
	err error
}

// NewBuilder создает строителя: необязательные поля сразу получают значения по умолчанию
func NewBuilder() *Builder {
	return &Builder{level: 1, gold: startingGold}
}

func (b *Builder) SetExperience(value int) *Builder { b.experience = value; return b }
func (b *Builder) SetInventory(items []item.Item) *Builder {
	b.inventory = slices.Clone(items)
	return b
}
func (b *Builder) SetAchievements(items []*progress.Achievement) *Builder {
	b.achievements = slices.Clone(items)
	return b
}

func (b *Builder) SetName(name string) *Builder {
	b.name = name
	return b
}

func (b *Builder) SetHeroClass(heroClass *HeroClass) *Builder {
	b.heroClass = heroClass
	return b
}

func (b *Builder) SetWeapon(weapon item.Weapon) *Builder {
	b.weapon = weapon
	return b
}

func (b *Builder) SetArmor(armor *item.Armor) *Builder {
	b.armor = armor
	return b
}

// SetStarterKit берет оружие и броню из стартового набора
func (b *Builder) SetStarterKit(starterKit kit.StarterKitFactory) *Builder {
	weapon, err := starterKit.CreateWeapon()
	if err != nil {
		b.fail(err)
		return b
	}
	armor, err := starterKit.CreateArmor()
	if err != nil {
		b.fail(err)
		return b
	}

	b.weapon = weapon
	b.armor = armor
	return b
}

func (b *Builder) SetLevel(level int) *Builder {
	b.level = level
	return b
}

func (b *Builder) SetGold(gold int) *Builder {
	b.gold = gold
	return b
}

// Build проверяет, что все обязательные поля заполнены корректно, и только тогда создает героя
func (b *Builder) Build() (*Hero, error) {
	if b.err != nil {
		return nil, b.err
	}
	if strings.TrimSpace(b.name) == "" {
		return nil, errors.New("у героя должно быть имя")
	}
	if utf8.RuneCountInString(b.name) > maxNameLength {
		return nil, fmt.Errorf("имя героя не может быть длиннее %d символов", maxNameLength)
	}
	if b.heroClass == nil {
		return nil, errors.New("не выбран класс героя")
	}
	if b.weapon == nil || b.armor == nil {
		return nil, errors.New("герою нужны оружие и броня")
	}
	if b.level < 1 {
		return nil, errors.New("уровень героя не может быть меньше 1")
	}
	if b.gold < 0 {
		return nil, errors.New("золота не может быть меньше нуля")
	}

	if b.experience < 0 {
		return nil, errors.New("опыт не может быть отрицательным")
	}
	hero := &Hero{
		experience: b.experience, inventory: slices.Clone(b.inventory),
		name:      b.name,
		heroClass: b.heroClass,
		level:     b.level,
		gold:      b.gold,
		weapon:    b.weapon,
		armor:     b.armor,
	}
	for _, a := range b.achievements {
		hero.Unlock(a)
	}
	hero.Rest()
	return hero, nil
}

func (b *Builder) fail(err error) {
	if b.err == nil {
		b.err = err
	}
}

func (h *Hero) RollDamage() combat.Hit {
	hit := h.weapon.RollDamage()
	return combat.Hit{Damage: hit.Damage + (h.level-1)*damagePerLevel, Critical: hit.Critical}
}

func (h *Hero) ReceiveHit(damage int) defense.Damage {
	reducedDamage := int(math.Round(float64(damage) * (1 - h.armor.Defense())))
	// хотя бы единицу урона броня пропускает всегда
	dealt := max(1, reducedDamage)
	h.health = max(0, h.health-dealt)
	return defense.Damage{Amount: dealt}
}

func (h *Hero) IsAlive() bool {
	return h.health > 0
}

// поведение героя между экспедициями

// Rest - перед каждой экспедицией герой успевает отдохнуть
func (h *Hero) Rest() {
	h.health = h.MaxHealth()
}

// GainExperience начисляет опыт и возвращает количество полученных уровней
func (h *Hero) GainExperience(amount int) int {
	h.experience += amount
	levelsGained := 0
	for h.experience >= h.ExperienceToNextLevel() {
		h.experience -= h.ExperienceToNextLevel()
		h.level++
		levelsGained++
	}

	return levelsGained
}

func (h *Hero) AddGold(amount int) {
	h.gold += amount
}

func (h *Hero) SpendGold(amount int) error {
	if amount > h.gold {
		return fmt.Errorf("не хватает золота: нужно %d, есть %d", amount, h.gold)
	}

	h.gold -= amount
	return nil
}

func (h *Hero) AddItem(it item.Item) {
	h.inventory = append(h.inventory, it)
}

func (h *Hero) RemoveItem(it item.Item) error {
	index := slices.Index(h.inventory, it)
	if index < 0 {
		return fmt.Errorf("в инвентаре нет предмета %s", it.Name())
	}

	h.inventory = slices.Delete(h.inventory, index, index+1)
	return nil
}

// Equip экипирует предмет из инвентаря, снятый предмет возвращается в инвентарь
func (h *Hero) Equip(it item.Item) error {
	// switch по типу: в каждой ветке equipment имеет конкретный тип
	switch equipment := it.(type) {
	case item.Weapon:
		if err := h.RemoveItem(equipment); err != nil {
			return err
		}
		h.inventory = append(h.inventory, h.weapon)
		h.weapon = equipment
	case *item.Armor:
		if err := h.RemoveItem(equipment); err != nil {
			return err
		}
		h.inventory = append(h.inventory, h.armor)
		h.armor = equipment
	default:
		return fmt.Errorf("предмет нельзя экипировать: %s", it.Name())
	}

	return nil
}

// геттеры

func (h *Hero) Code() string { return h.heroClass.Code() }

func (h *Hero) Name() string { return h.name }

func (h *Hero) Health() int { return h.health }

func (h *Hero) MaxHealth() int {
	return h.heroClass.BaseHealth() + (h.level-1)*healthPerLevel
}

func (h *Hero) HeroClass() *HeroClass { return h.heroClass }

func (h *Hero) Level() int { return h.level }

func (h *Hero) Experience() int { return h.experience }

func (h *Hero) ExperienceToNextLevel() int { return h.level * experiencePerLevel }

func (h *Hero) Gold() int { return h.gold }

func (h *Hero) Weapon() item.Weapon { return h.weapon }

func (h *Hero) Armor() *item.Armor { return h.armor }

// Inventory возвращает копию списка предметов, чтобы инвентарь нельзя было поменять снаружи
func (h *Hero) Inventory() []item.Item {
	return slices.Clone(h.inventory)
}

func (h *Hero) Unlock(a *progress.Achievement) bool {
	if slices.Contains(h.achievements, a) {
		return false
	}
	h.achievements = append(h.achievements, a)
	return true
}
func (h *Hero) Achievements() []*progress.Achievement { return slices.Clone(h.achievements) }
```

В `Location` заменяем вызов `RestoreHealth()` на `Rest()` — так герой будет отдыхать перед экспедицией:

**`labs/lab3/world/location.go`**

```go
package world

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/enemy"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/hero"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
)

const (
	lootChance    = 0.6
	enchantChance = 0.3
)

// Location - локация. Каждая конкретная локация сама решает, какого врага создать (фабричный метод CreateEnemy)
type Location interface {
	Code() string
	DisplayName() string
	RecommendedLevel() int
	Catalog() *item.ItemCatalog
	// CreateEnemy - создание врага делегируется конкретным локациям для повышения гибкости
	CreateEnemy() enemy.Enemy
	// LootTable - коды предметов из каталога, которые можно найти в локации
	LootTable() []string
}

// baseLocation - общие данные всех локаций, конкретные локации встраивают эту структуру в себя
type baseLocation struct {
	code             string
	displayName      string
	recommendedLevel int
	catalog          *item.ItemCatalog
}

func (l *baseLocation) Code() string { return l.code }

func (l *baseLocation) DisplayName() string { return l.displayName }

func (l *baseLocation) RecommendedLevel() int { return l.recommendedLevel }

func (l *baseLocation) Catalog() *item.ItemCatalog { return l.catalog }

// Explore отправляет героя в экспедицию со случайным сидом
func Explore(location Location, h *hero.Hero) (ExpeditionReport, error) {
	return ExploreWithSeed(location, h, core.GetInstance().NextSeed())
}

// ExploreWithSeed - бизнес-логика, общая для всех локаций:
// герой встречает врага, сражается с ним и получает награду, если победил
func ExploreWithSeed(location Location, h *hero.Hero, seed uint64) (ExpeditionReport, error) {
	core.GetInstance().Reseed(seed)

	h.Rest()
	foe := location.CreateEnemy()
	battle := combat.NewBattle(h, foe).Run()

	var rewards Rewards
	if battle.Outcome == combat.Victory {
		var err error
		rewards, err = reward(location, h, foe)
		if err != nil {
			return ExpeditionReport{}, err
		}
	} else {
		rewards = punish(h)
	}

	return ExpeditionReport{Seed: seed, LocationCode: location.Code(), Battle: battle, Rewards: rewards}, nil
}

func reward(location Location, h *hero.Hero, foe enemy.Enemy) (Rewards, error) {
	experience := foe.ExperienceReward()
	gold := foe.RollGoldReward()
	loot, err := rollLoot(location)
	if err != nil {
		return Rewards{}, err
	}

	levelsGained := h.GainExperience(experience)
	h.AddGold(gold)
	if loot != nil {
		h.AddItem(loot)
	}

	return Rewards{Experience: experience, Gold: gold, Loot: loot, LevelsGained: levelsGained}, nil
}

func rollLoot(location Location) (item.Item, error) {
	dice := core.GetInstance()
	if !dice.Chance(lootChance) {
		return nil, nil
	}

	// добыча - это копия шаблона из каталога, ее можно менять, не боясь испортить шаблон
	table := location.LootTable()
	loot, err := location.Catalog().Create(table[dice.Index(len(table))])
	if err != nil {
		return nil, err
	}
	if dice.Chance(enchantChance) {
		possible := loot.PossibleEnchantments()
		if err := loot.Enchant(possible[dice.Index(len(possible))]); err != nil {
			return nil, err
		}
	}

	return loot, nil
}

// проигравший герой теряет половину золота
func punish(h *hero.Hero) Rewards {
	lostGold := h.Gold() / 2
	// потратить половину имеющегося золота можно всегда, поэтому ошибку здесь не проверяем
	_ = h.SpendGold(lostGold)
	return Rewards{Experience: 0, Gold: -lostGold, Loot: nil, LevelsGained: 0}
}
```

Чтобы иметь возможность восстановить зачарование по идентификатору (нам это понадобится дальше), добавим в него поиск по этому самому
идентификатору:

**`labs/lab3/item/enchantment.go`**

```go
package item

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
)

// Enchantment - зачарование. Все его поля неэкспортируемые и меняются только при создании,
// поэтому зачарование неизменяемо, и одно и то же зачарование можно безопасно повесить на несколько предметов
type Enchantment struct {
	code         string
	name         string
	bonusDamage  int
	bonusDefense float64
	bonusPrice   int
}

var (
	Sharp   = &Enchantment{"sharp", "Острое", 3, 0, 20}
	Fiery   = &Enchantment{"fiery", "Огненное", 6, 0, 50}
	Sturdy  = &Enchantment{"sturdy", "Прочное", 0, 0.05, 20}
	Warding = &Enchantment{"warding", "Охранное", 0, 0.10, 50}

	ForWeapons = []*Enchantment{Sharp, Fiery}
	ForArmor   = []*Enchantment{Sturdy, Warding}
)

func (e *Enchantment) Code() string          { return e.code }
func (e *Enchantment) Name() string          { return e.name }
func (e *Enchantment) BonusDamage() int      { return e.bonusDamage }
func (e *Enchantment) BonusDefense() float64 { return e.bonusDefense }
func (e *Enchantment) BonusPrice() int       { return e.bonusPrice }

func EnchantmentByCode(code string) (*Enchantment, error) {
	for _, e := range []*Enchantment{Sharp, Fiery, Sturdy, Warding} {
		if e.Code() == code {
			return e, nil
		}
	}
	return nil, &core.UnknownCodeError{What: "Зачарование", Code: code}
}
```

Супер, предварительные изменения внесены, давайте приступать к паттернам и масштабному расширению функционала!

## Одержимое оружие в проклятом особняке (Адаптер / Adapter)

Представим, что мы хотим ввести в нашу игру новую локацию. Вдохновившись историями про жутких призраков таковой мы решаем сделать
проклятый особняк, в нем на героя будет нападать одержимые предметы!

Писать новую реализацию врага с нуля мы не хотим, было бы круто и удобно использовать готовое оружие с интерфейсом `Weapon`, это нам и идейно хорошо подходит,
и часть логики пересекается, и `Weapon`, и `Enemy` умеют наносить урон! Но загвоздка в том, что получать урон оружие не умеет...

Чтобы решить это недоразумение и заставить `Weapon` полноценно соответствовать контракту `Enemy` без изменения самого `Weapon`,
применим замечательный паттерн *Адаптер*.

Он позволяет объектам с несовместимыми интерфейсами работать вместе: адаптер оборачивает нужный объект (оружие с интерфейсом `Weapon` в нашем случае) и
реализует нужный интерфейс (`Enemy` для нас), переводя обращения к себе в вызовы методов оборачиваемого объекта. А если для части контрактов
методов у оборачиваемого объекта нет — адаптер реализует их сам.

Таким образом мы сможем прокинуть `Weapon` как `Enemy` в новую реализацию `Location` с помощью адаптера, и нам не придется менять ни один из
упомянутых
выше типов!

Давайте скорее реализовывать это чудо, напишем сам адаптер. Назовем его `HauntedWeapon`, чтобы выдержать дух, но можно было назвать и более
технично, например,
`WeaponToEnemyAdapter`, суть была бы та же, но не так зловеще.

И так, код получится вот такой:

**`labs/lab3/enemy/haunted_weapon.go`**

```go
package enemy

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/defense"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
)

type HauntedWeapon struct {
	weapon            item.Weapon
	health, maxHealth int
}

func NewHauntedWeapon(weapon item.Weapon) *HauntedWeapon {
	hp := weapon.Damage() * 5
	return &HauntedWeapon{weapon, hp, hp}
}
func (e *HauntedWeapon) Code() string           { return "haunted-weapon" }
func (e *HauntedWeapon) Name() string           { return "Одержимый " + e.weapon.Name() }
func (e *HauntedWeapon) Health() int            { return e.health }
func (e *HauntedWeapon) MaxHealth() int         { return e.maxHealth }
func (e *HauntedWeapon) IsAlive() bool          { return e.health > 0 }
func (e *HauntedWeapon) RollDamage() combat.Hit { return e.weapon.RollDamage() }
func (e *HauntedWeapon) ExperienceReward() int  { return e.weapon.Price() / 2 }
func (e *HauntedWeapon) RollGoldReward() int {
	return core.GetInstance().Roll(e.weapon.Price()/4, e.weapon.Price()/2)
}
func (e *HauntedWeapon) ReceiveHit(amount int) defense.Damage {
	e.health = max(0, e.health-amount)
	return defense.Damage{Amount: amount}
}
```

А теперь объявим и саму локацию — проклятый особняк. В ней будет одно из трех случайных оружий, которые будут взяты из каталога.
Выбранное оружие будет обернуто в адаптер, то есть станет одержимо злым духом, после чего безжалостно нападет на нашего героя.

Благодаря адаптеру мы можем превратить во врага любое оружие, что безусловно разнообразит визиты в это замечательное жилище:

**`labs/lab3/world/haunted_manor.go`**

```go
package world

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/enemy"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
)

const HauntedManorCode = "haunted-manor"

type HauntedManor struct{ baseLocation }

func NewHauntedManor(catalog *item.ItemCatalog) *HauntedManor {
	return &HauntedManor{baseLocation{HauntedManorCode, "Проклятый особняк", 2, catalog}}
}
func (l *HauntedManor) LootTable() []string {
	return []string{item.IronSword, item.ElvenBow, item.EmberStaff}
}
func (l *HauntedManor) CreateEnemy() enemy.Enemy {
	table := l.LootTable()
	weapon, err := l.catalog.CreateWeapon(table[core.GetInstance().Index(len(table))])
	if err != nil {
		panic(err)
	}
	return enemy.NewHauntedWeapon(weapon)
}
```

Просто прелестно! Получилось красиво, расширяемо и интересно. А еще мы сэкономили на реализации метода нанесения урона! Его получилось взять
из оружия подчистую, и менять ничего не пришлось. Остальные методы в целом тоже интересно легли на механику оружия (большинство из них так
или иначе
зависит от его характеристик, здоровье от урона, а награда от ценности).

Важно запомнить, что адаптер стоит применять только если адаптируемый тип близок по своей логике к нужному интерфейсу. Если это не так,
адаптер может решить все задачи самостоятельно и вместо переиспользования у нас получится лишняя зависимость в адаптере. В нашем примере
случай в целом
пограничный – часть логики переиспользуем, часть реализуем сами.

Главный маркер того, что адаптер стоит рассмотреть — готовый тип нужно использовать там, где ожидается другой, но идейно похожий
интерфейс,
а менять сам тип нельзя или не хочется - например, если это чужая библиотека.

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
каждому из них отдельные типы
с модификаторами (неистовый гоблин, неистовый дракон, и так далее) это еще половина беды, настоящая проблема в том, что мы их должны еще и
комбинировать. Если каждый из трех модификаторов может присутствовать или отсутствовать, даже без учета порядка
получится 4 × 2³ = 32 варианта врагов, включая обычных врагов без модификаторов.

То есть отдельный тип врага на каждую комбинацию — плохое решение: слишком много типов, и если у нас появятся еще модификаторы, их станет
еще больше.

На помощь нам приходит паттерн декоратор! Он умеет заворачивать в себя любой объект, реализующий нужный интерфейс, и модифицировать его
поведение.
При этом этих декораторов можно комбинировать! И понадобится нам всего 3 структуры декораторов, 1 структура базового декоратора и все. И новый
модификатор потребует
всего лишь нового типа декоратора. Плюсом к этому, мы еще и заворачивать их сможем динамически, прямо при выполнении программы.

Получаются сплошные плюсы, давайте смотреть

Еще раз проговорим, про что сам паттерн. *Декоратор* позволяет динамически добавлять объекту новое поведение, оборачивая его
в другие объекты (декораторы) с тем же интерфейсом. Благодаря композиции декоратор расширяет поведение методов типа без
необходимости создавать отдельный тип для каждой комбинации, а декораторы при этом можно комбинировать друг с другом.

Первым делом определим базовый декоратор, он не определяет нового поведения, а выступает основной структурной единицей для дальнейшей
реализации паттерна. В нем через композицию определяется, кого заворачивает декоратор (встроенный интерфейс `Enemy`), все методы базового декоратора
делегируют
вызовы объекту `Enemy`.

Важно, что `Enemy` это абстракция – `Enemy` и сам базовый декоратор тоже реализует эту абстракцию. Таким образом на месте `Enemy` может
быть и другой декоратор.
Но в конце цепочки обязательно будет конкретный враг.

Итак, код получается вот такой:

**`labs/lab3/enemy/enemy_decorator.go`**

```go
package enemy

type EnemyDecorator struct{ Enemy }

func NewEnemyDecorator(wrapped Enemy) EnemyDecorator { return EnemyDecorator{wrapped} }
```

Встраивание `Enemy` дает обертке методы вложенного врага. Чтобы поменять поведение, конкретная обертка объявляет свой метод с тем же именем.

А дальше уже реализуем конкретных декораторов, со встроенным базовым декоратором. Они будут соответствовать нашим модификатором, начнем с легендарного:

**`labs/lab3/enemy/legendary_enemy.go`**

```go
package enemy

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
)

type LegendaryEnemy struct{ EnemyDecorator }

func NewLegendaryEnemy(wrapped Enemy) *LegendaryEnemy {
	return &LegendaryEnemy{NewEnemyDecorator(wrapped)}
}
func (e *LegendaryEnemy) Name() string           { return "Легендарный " + e.Enemy.Name() }
func (e *LegendaryEnemy) RollDamage() combat.Hit { h := e.Enemy.RollDamage(); h.Damage *= 2; return h }
func (e *LegendaryEnemy) ExperienceReward() int  { return e.Enemy.ExperienceReward() * 2 }
func (e *LegendaryEnemy) RollGoldReward() int    { return e.Enemy.RollGoldReward() * 2 }
```

Тут важно заметить, что легендарный декоратор берет за основу результат вызова `e.Enemy.RollDamage()`.
Именно это и позволяет комбинировать декораторов. Внутри может оказаться не только обычный враг,
но и другая обертка: тогда вызов вернет уже модифицированное ею поведение. И так по цепочке,
пока мы не дойдем до одного из наших конкретных врагов.

Супер, базовый принцип понятен, продолжим реализовывать модификаторы, дальше у нас неистовство ветра:

**`labs/lab3/enemy/windfury_enemy.go`**

```go
package enemy

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
)

type WindfuryEnemy struct{ EnemyDecorator }

func NewWindfuryEnemy(wrapped Enemy) *WindfuryEnemy {
	return &WindfuryEnemy{NewEnemyDecorator(wrapped)}
}
func (e *WindfuryEnemy) Name() string { return "Неистовый " + e.Enemy.Name() }
func (e *WindfuryEnemy) RollDamage() combat.Hit {
	first, second := e.Enemy.RollDamage(), e.Enemy.RollDamage()
	return combat.Hit{Damage: first.Damage + second.Damage, Critical: first.Critical || second.Critical}
}
func (e *WindfuryEnemy) ExperienceReward() int { r := e.Enemy.ExperienceReward(); return r + r*50/100 }
func (e *WindfuryEnemy) RollGoldReward() int   { r := e.Enemy.RollGoldReward(); return r + r*50/100 }
```

Два броска здесь складываются в один удар, поэтому защита применяется к их сумме.

Логика тут точно такая же! Модифицируем поведение, делегируем вызовы вложенному врагу.

Третий и пока финальный декоратор уменьшает входящий урон вдвое, затем передаёт получившийся урон дальше по цепочке декораторов:

**`labs/lab3/enemy/armored_enemy.go`**

```go
package enemy

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/defense"
)

type ArmoredEnemy struct{ EnemyDecorator }

func NewArmoredEnemy(wrapped Enemy) *ArmoredEnemy { return &ArmoredEnemy{NewEnemyDecorator(wrapped)} }
func (e *ArmoredEnemy) Name() string              { return "Бронированный " + e.Enemy.Name() }
func (e *ArmoredEnemy) ReceiveHit(amount int) defense.Damage {
	if amount <= 0 {
		return defense.Damage{}
	}
	return e.Enemy.ReceiveHit(amount - amount/2)
}
```

Отлично, все требуемые модификаторы объявлены.
Теперь мы можем добавлять их любому врагу и в любой последовательности.
При этом декоратор реализует тот же интерфейс `Enemy`, что и обычный враг, поэтому ни бой, ни локация не отличают элитного врага
от обычного, и работают с ним точно так же. Прелести `Dependency Inversion Principle`!

Осталось прописать логику, как и когда враг становится элитным. Это часть общей логики экспедиции, поэтому добавим её в локацию:
враг, созданный фабричным методом, с некоторым шансом получает комбинацию модификаторов.

**`labs/lab3/world/location.go`**

```go
package world

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/enemy"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/hero"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
)

const (
	lootChance    = 0.6
	enchantChance = 0.3
)

// Location - локация. Каждая конкретная локация сама решает, какого врага создать (фабричный метод CreateEnemy)
type Location interface {
	Code() string
	DisplayName() string
	RecommendedLevel() int
	Catalog() *item.ItemCatalog
	// CreateEnemy - создание врага делегируется конкретным локациям для повышения гибкости
	CreateEnemy() enemy.Enemy
	// LootTable - коды предметов из каталога, которые можно найти в локации
	LootTable() []string
}

// baseLocation - общие данные всех локаций, конкретные локации встраивают эту структуру в себя
type baseLocation struct {
	code             string
	displayName      string
	recommendedLevel int
	catalog          *item.ItemCatalog
}

func (l *baseLocation) Code() string { return l.code }

func (l *baseLocation) DisplayName() string { return l.displayName }

func (l *baseLocation) RecommendedLevel() int { return l.recommendedLevel }

func (l *baseLocation) Catalog() *item.ItemCatalog { return l.catalog }

// Explore отправляет героя в экспедицию со случайным сидом
func Explore(location Location, h *hero.Hero) (ExpeditionReport, error) {
	return ExploreWithSeed(location, h, core.GetInstance().NextSeed())
}

// ExploreWithSeed - бизнес-логика, общая для всех локаций:
// герой встречает врага, сражается с ним и получает награду, если победил
func ExploreWithSeed(location Location, h *hero.Hero, seed uint64) (ExpeditionReport, error) {
	core.GetInstance().Reseed(seed)

	h.Rest()
	foe := location.CreateEnemy()
	if core.GetInstance().Chance(0.2) { foe = makeElite(foe) }
	battle := combat.NewBattle(h, foe).Run()

	var rewards Rewards
	if battle.Outcome == combat.Victory {
		var err error
		rewards, err = reward(location, h, foe)
		if err != nil {
			return ExpeditionReport{}, err
		}
	} else {
		rewards = punish(h)
	}

	return ExpeditionReport{Seed: seed, LocationCode: location.Code(), Battle: battle, Rewards: rewards}, nil
}

func reward(location Location, h *hero.Hero, foe enemy.Enemy) (Rewards, error) {
	experience := foe.ExperienceReward()
	gold := foe.RollGoldReward()
	loot, err := rollLoot(location)
	if err != nil {
		return Rewards{}, err
	}

	levelsGained := h.GainExperience(experience)
	h.AddGold(gold)
	if loot != nil {
		h.AddItem(loot)
	}

	return Rewards{Experience: experience, Gold: gold, Loot: loot, LevelsGained: levelsGained}, nil
}

func rollLoot(location Location) (item.Item, error) {
	dice := core.GetInstance()
	if !dice.Chance(lootChance) {
		return nil, nil
	}

	// добыча - это копия шаблона из каталога, ее можно менять, не боясь испортить шаблон
	table := location.LootTable()
	loot, err := location.Catalog().Create(table[dice.Index(len(table))])
	if err != nil {
		return nil, err
	}
	if dice.Chance(enchantChance) {
		possible := loot.PossibleEnchantments()
		if err := loot.Enchant(possible[dice.Index(len(possible))]); err != nil {
			return nil, err
		}
	}

	return loot, nil
}

// проигравший герой теряет половину золота
func punish(h *hero.Hero) Rewards {
	lostGold := h.Gold() / 2
	// потратить половину имеющегося золота можно всегда, поэтому ошибку здесь не проверяем
	_ = h.SpendGold(lostGold)
	return Rewards{Experience: 0, Gold: -lostGold, Loot: nil, LevelsGained: 0}
}

func makeElite(foe enemy.Enemy) enemy.Enemy {
	dice := core.GetInstance()
	both := dice.Chance(0.25)
	if dice.Chance(0.5) {
		foe = enemy.NewLegendaryEnemy(foe)
		if both {
			foe = enemy.NewWindfuryEnemy(foe)
		}
	} else {
		foe = enemy.NewWindfuryEnemy(foe)
		if both {
			foe = enemy.NewLegendaryEnemy(foe)
		}
	}
	if dice.Chance(0.3) {
		foe = enemy.NewArmoredEnemy(foe)
	}
	return foe
}
```

Получилось красиво и очень расширяемо! Добавить новый модификатор не составит нам ровно никакого труда, главное
аккуратно реализовать передачу вызовов, чтобы комбинирование не развалилось.

Как можно понять из примера, применять декоратор стоит, когда объекту нужно добавлять поведение во время работы программы, а
комбинаций такого поведения слишком много, чтобы заводить на каждую свой тип.

## Сохранения (Заместитель / Proxy)

В нашей игре нет кое-чего очень важного! Сейчас при каждом выходе из игры герой просто пропадает. Но со всеми нашими усложнениями
это становится все менее приемлемо. Нам надо сохранять прогресс, чтобы не стартовать с самого начала.

Мы могли бы сделать это через БД, но этим мы займемся сильно позже. Над реализацией, которая нас устроила бы сейчас, подумаем чуть дальше. А
пока
давайте определим контракт для сервиса, который будет заниматься сохранением нашего героя.

Опишем интерфейс хранилища сохранений, нам тут важно иметь возможность найти героя по имени, вывести список всех героев, а также сохранить
текущее состояние нашего персонажа:

**`labs/lab3/storage/hero_repository.go`**

```go
package storage

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/hero"
)

type HeroRepository interface {
	FindByName(string) (*hero.Hero, error)
	FindAll() ([]*hero.Hero, error)
	Save(*hero.Hero) error
}
```

Для хранения сейвов будем использовать простой файл, это не бд, но подход тоже вполне рабочий. В качестве формата сохранения выберем
json! [JSON](https://www.json.org/json-ru.html) — текстовый формат, где объекты состоят из пар "имя — значение",
а массивы хранят списки значений. Это довольно вездесущий формат, и нам он как нельзя кстати подходит.

Для чтения и записи JSON используем стандартный пакет `encoding/json`, дополнительные зависимости не нужны.

В файле сохранения будет массив героев со всеми нужными для восстановления объектов данными.
Структуры `heroSave` и `itemSave` описывают данные сохранения. Их поля экспортируемые, чтобы JSON-кодек мог с ними работать, а теги `json` задают имена полей в файле.

Реализуем наконец сам репозиторий:

**`labs/lab3/storage/file_hero_repository.go`**

```go
package storage

import (
	"bytes"
	"encoding/json"
	"errors"
	"fmt"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/hero"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/progress"
	"os"
	"path/filepath"
)

type FileHeroRepository struct {
	file    string
	catalog *item.ItemCatalog
}

func NewFileHeroRepository(file string, catalog *item.ItemCatalog) *FileHeroRepository {
	return &FileHeroRepository{file, catalog}
}

type itemSave struct {
	Code         string   `json:"code"`
	Enchantments []string `json:"enchantments"`
}
type heroSave struct {
	Name         string     `json:"name"`
	HeroClass    string     `json:"heroClass"`
	Level        int        `json:"level"`
	Experience   int        `json:"experience"`
	Gold         int        `json:"gold"`
	Weapon       itemSave   `json:"weapon"`
	Armor        itemSave   `json:"armor"`
	Inventory    []itemSave `json:"inventory"`
	Achievements []string   `json:"achievements"`
}

func (r *FileHeroRepository) FindByName(name string) (*hero.Hero, error) {
	list, err := r.FindAll()
	if err != nil {
		return nil, err
	}
	for _, h := range list {
		if h.Name() == name {
			return h, nil
		}
	}
	return nil, nil
}
func (r *FileHeroRepository) FindAll() ([]*hero.Hero, error) {
	data, err := os.ReadFile(r.file)
	if errors.Is(err, os.ErrNotExist) {
		return []*hero.Hero{}, nil
	}
	if err != nil {
		return nil, err
	}
	if len(bytes.TrimSpace(data)) == 0 {
		return []*hero.Hero{}, nil
	}
	var saves []heroSave
	if err = json.Unmarshal(data, &saves); err != nil {
		return nil, err
	}
	if saves == nil {
		return nil, fmt.Errorf("в файле должен быть массив героев")
	}
	result := make([]*hero.Hero, 0, len(saves))
	for _, save := range saves {
		h, err := r.restore(save)
		if err != nil {
			return nil, err
		}
		result = append(result, h)
	}
	return result, nil
}
func (r *FileHeroRepository) Save(h *hero.Hero) error {
	heroes, err := r.FindAll()
	if err != nil {
		return err
	}
	saves := []heroSave{}
	for _, other := range heroes {
		if other.Name() != h.Name() {
			saves = append(saves, snapshot(other))
		}
	}
	saves = append(saves, snapshot(h))
	data, err := json.MarshalIndent(saves, "", "  ")
	if err != nil {
		return err
	}
	if err = os.MkdirAll(filepath.Dir(r.file), 0755); err != nil {
		return err
	}
	return os.WriteFile(r.file, append(data, '\n'), 0600)
}
func snapshotItem(it item.Item) itemSave {
	codes := []string{}
	for _, e := range it.Enchantments() {
		codes = append(codes, e.Code())
	}
	return itemSave{it.Code(), codes}
}
func snapshot(h *hero.Hero) heroSave {
	inv := []itemSave{}
	for _, it := range h.Inventory() {
		inv = append(inv, snapshotItem(it))
	}
	ach := []string{}
	for _, a := range h.Achievements() {
		ach = append(ach, a.Code())
	}
	return heroSave{h.Name(), h.HeroClass().Code(), h.Level(), h.Experience(), h.Gold(), snapshotItem(h.Weapon()), snapshotItem(h.Armor()), inv, ach}
}
func (r *FileHeroRepository) restoreItem(s itemSave) (item.Item, error) {
	it, err := r.catalog.Create(s.Code)
	if err != nil {
		return nil, err
	}
	for _, code := range s.Enchantments {
		e, err := item.EnchantmentByCode(code)
		if err != nil {
			return nil, err
		}
		if err = it.Enchant(e); err != nil {
			return nil, err
		}
	}
	return it, nil
}
func (r *FileHeroRepository) restore(s heroSave) (*hero.Hero, error) {
	hc, err := hero.HeroClassByCode(s.HeroClass)
	if err != nil {
		return nil, err
	}
	w, err := r.restoreItem(s.Weapon)
	if err != nil {
		return nil, err
	}
	a, err := r.restoreItem(s.Armor)
	if err != nil {
		return nil, err
	}
	weapon, ok := w.(item.Weapon)
	if !ok {
		return nil, fmt.Errorf("сохраненный предмет не оружие")
	}
	armor, ok := a.(*item.Armor)
	if !ok {
		return nil, fmt.Errorf("сохраненный предмет не броня")
	}
	inv := []item.Item{}
	for _, save := range s.Inventory {
		it, err := r.restoreItem(save)
		if err != nil {
			return nil, err
		}
		inv = append(inv, it)
	}
	ach := []*progress.Achievement{}
	for _, code := range s.Achievements {
		v, err := progress.ByCode(code)
		if err != nil {
			return nil, err
		}
		ach = append(ach, v)
	}
	return hero.NewBuilder().SetName(s.Name).SetHeroClass(hc).SetLevel(s.Level).SetExperience(s.Experience).SetGold(s.Gold).SetWeapon(weapon).SetArmor(armor).SetInventory(inv).SetAchievements(ach).Build()
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

**`labs/lab3/storage/caching_hero_repository.go`**

```go
package storage

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/hero"
	"slices"
)

type CachingHeroRepository struct {
	target HeroRepository
	cache  []*hero.Hero
	loaded bool
}

func NewCachingHeroRepository(target HeroRepository) *CachingHeroRepository {
	return &CachingHeroRepository{target: target}
}
func (r *CachingHeroRepository) FindAll() ([]*hero.Hero, error) {
	if !r.loaded {
		list, err := r.target.FindAll()
		if err != nil {
			return nil, err
		}
		r.cache = list
		r.loaded = true
	}
	return slices.Clone(r.cache), nil
}
func (r *CachingHeroRepository) FindByName(name string) (*hero.Hero, error) {
	list, err := r.FindAll()
	if err != nil {
		return nil, err
	}
	for _, h := range list {
		if h.Name() == name {
			return h, nil
		}
	}
	return nil, nil
}
func (r *CachingHeroRepository) Save(h *hero.Hero) error {
	if err := r.target.Save(h); err != nil {
		return err
	}
	if _, err := r.FindAll(); err != nil {
		return err
	}
	for i, other := range r.cache {
		if other.Name() == h.Name() {
			r.cache[i] = h
			return nil
		}
	}
	r.cache = append(r.cache, h)
	return nil
}
```

Отлично! Не трогая код работы с файлом, мы смогли модифицировать и оптимизировать его поведение. Кстати, кэш здесь
заполняется лениво: пока героев никто не попросил, файл даже не читается.

Заметим, что заместитель и декоратор очень похожи, они оба оборачивают объект с тем же интерфейсом и позволяют изменять поведение
оборачиваемого объекта. Однако заместитель чаще всего не добавляет новое поведение, а управляет доступом к
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
правки этой реализации. А еще нарушался бы `SRP`.

Так как в данном случае все изменения сводятся к выбору между разными алгоритми выполнения одного и того же действия
(выбора, что делать в раунде), можно применить паттерн *Стратегия*.

Он определяет семейство алгоритмов, выносит каждый
из них в отдельную реализацию интерфейса и делает их взаимозаменяемыми: тот, кто алгоритм использует, работает с общим интерфейсом и
не знает, какая именно реализация ему досталась.

Сначала опишем, что герой вообще может сделать в раунде:

**`labs/lab3/tactic/action.go`**

```go
package tactic

type Action struct{ AttackMultiplier, IncomingMultiplier float64 }

var Attack = Action{1, 1}
var PowerStrike = Action{1.5, 1.5}
var Defend = Action{0.5, 0.3}
```

Получается, что он может вести себя агрессивно (увеличивается и исходящий, и входящий урон), сбалансированно (ничего не меняется) и
осторожно, уходя в оборону (уменьшается и входящий, и исходящий урон).

Выбирать действия мы будем в зависимости от состояния врага и героя, определим базовый интерфейс тактики:

**`labs/lab3/tactic/tactic.go`**

```go
package tactic

// Для выбора действия достаточно здоровья участников: пакет не зависит от реализации боя.
type Combatant interface {
	Health() int
	MaxHealth() int
}
type Tactic interface {
	Code() string
	DisplayName() string
	Description() string
	ChooseAction(round int, hero, enemy Combatant) Action
}
```

В пакете `tactic` объявлен небольшой интерфейс `Combatant` с двумя методами здоровья — именно они нужны тактике для выбора действия.

Реализуем самый простой алгоритм выбора действия - обычный удар каждый раунд:

**`labs/lab3/tactic/balanced_tactic.go`**

```go
package tactic

type BalancedTactic struct{}

func (BalancedTactic) Code() string                                         { return "balanced" }
func (BalancedTactic) DisplayName() string                                  { return "Баланс" }
func (BalancedTactic) Description() string                                  { return "обычный удар каждый раунд" }
func (BalancedTactic) ChooseAction(round int, hero, enemy Combatant) Action { return Attack }
```

Натиск - пока здоровье врага выше, чем у героя, герой бьет безрассудно, нанося и получая в ответ больше урона:

**`labs/lab3/tactic/aggressive_tactic.go`**

```go
package tactic

type AggressiveTactic struct{}

func (AggressiveTactic) Code() string        { return "aggressive" }
func (AggressiveTactic) DisplayName() string { return "Натиск" }
func (AggressiveTactic) Description() string {
	return "совершать безрассудные удары, пока у врага больше здоровья, чем у героя"
}
func (AggressiveTactic) ChooseAction(round int, hero, enemy Combatant) Action {
	if enemy.Health() > hero.Health() {
		return PowerStrike
	}
	return Attack
}
```

И осторожность - раненый герой через раунд будет уходить в оборону:

**`labs/lab3/tactic/cautious_tactic.go`**

```go
package tactic

type CautiousTactic struct{}

func (CautiousTactic) Code() string        { return "cautious" }
func (CautiousTactic) DisplayName() string { return "Осторожность" }
func (CautiousTactic) Description() string {
	return "при здоровье ниже 40% через раунд уходить в оборону"
}
func (CautiousTactic) ChooseAction(round int, hero, enemy Combatant) Action {
	if float64(hero.Health()) < float64(hero.MaxHealth())*0.4 && round%2 == 0 {
		return Defend
	}
	return Attack
}
```

Также дополним типы ивентов в нашей игре, добавим наши действия, и `EventEvade`/`EventAbsorb`, они нам пригодятся дальше:

**`labs/lab3/combat/event_type.go`**

```go
package combat

type EventType int

const (
	EventHit EventType = iota
	EventCriticalHit
	EventDeath
)

const (
	EventPowerStrike EventType = iota + 3
	EventDefend
	EventEvade
	EventAbsorb
)
```

Теперь обновим сам `Battle`. Тактику он получает через конструктор и каждый раунд спрашивает у нее, что именно делает герой:

**`labs/lab3/combat/battle.go`**

```go
package combat

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/tactic"
	"math"
)

// Battle - бой идет по раундам: сначала бьет герой, потом враг, пока кто-то не погибнет
type Battle struct {
	hero   Combatant
	enemy  Combatant
	events []BattleEvent
	round  int
	tactic tactic.Tactic
}

func NewBattle(hero, enemy Combatant, strategy tactic.Tactic) *Battle {
	return &Battle{hero: hero, enemy: enemy, tactic: strategy}
}

func (b *Battle) Run() BattleReport {
	for b.hero.IsAlive() && b.enemy.IsAlive() {
		b.round++
		action := b.tactic.ChooseAction(b.round, b.hero, b.enemy)
		if action == tactic.PowerStrike {
			b.emit(BattleEvent{b.round, EventPowerStrike, SideHero, SideEnemy, 0, 0})
		} else if action == tactic.Defend {
			b.emit(BattleEvent{b.round, EventDefend, SideHero, SideHero, 0, 0})
		}
		b.strike(b.hero, SideHero, b.enemy, SideEnemy, action.AttackMultiplier)
		if b.enemy.IsAlive() {
			b.strike(b.enemy, SideEnemy, b.hero, SideHero, action.IncomingMultiplier)
		}
	}

	outcome := Defeat
	if b.hero.IsAlive() {
		outcome = Victory
	}

	return BattleReport{
		Hero:    ParticipantOf(b.hero),
		Enemy:   ParticipantOf(b.enemy),
		Outcome: outcome,
		Events:  b.events,
	}
}

func (b *Battle) strike(attacker Combatant, source Side, defender Combatant, target Side, multiplier float64) {
	hit := attacker.RollDamage()
	d := defender.ReceiveHit(max(1, int(math.Round(float64(hit.Damage)*multiplier))))
	if d.Evaded {
		b.emit(BattleEvent{b.round, EventEvade, source, target, 0, defender.Health()})
		return
	}
	if d.Absorbed > 0 {
		b.emit(BattleEvent{b.round, EventAbsorb, source, target, d.Absorbed, defender.Health()})
	}
	if d.Amount > 0 {
		eventType := EventHit
		if hit.Critical {
			eventType = EventCriticalHit
		}
		b.emit(BattleEvent{b.round, eventType, source, target, d.Amount, defender.Health()})
	}

	if !defender.IsAlive() {
		b.emit(BattleEvent{b.round, EventDeath, source, target, 0, 0})
	}
}

func (b *Battle) emit(event BattleEvent) {
	b.events = append(b.events, event)
}
```

Обратите внимание, что бой ничего не знает о конкретных тактиках: он работает с интерфейсом `Tactic`, и
подставить ему можно любую реализацию. Если появится новая тактика, `Battle` менять не придется.

Ну и в отчете об экспедиции будем запоминать, с какой тактикой герой ее начинал:

**`labs/lab3/world/expedition_report.go`**

```go
package world

import "github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"

// ExpeditionReport - отчет об экспедиции: с каким сидом, куда, как прошел бой и что получил герой
type ExpeditionReport struct {
	Seed         uint64
	LocationCode string
	TacticCode   string
	Battle       combat.BattleReport
	Rewards      Rewards
}
```

Стратегия позволяет нам соблюдать `Open Closed` принцип, он дает нам возможность расширять поведение нашей реализации без ее изменения, путем
добавления новых реализаций стратегии. Стратегию можно выбирать как в методе, где она используется, так и в конструкторе, все зависит от
конкретной необходимости. В нашем случае стратегия выбирается в конструкторе, потому что по ходу боя она не меняется.

Применять стратегию стоит, когда одно и то же действие можно выполнить несколькими способами и способ нужно выбирать
или менять во время работы программы.

И давайте сразу обновим `Location`: она будет принимать тактику и передавать дальше в `Battle`.

Получится что-то такое:

**`labs/lab3/world/location.go`**

```go
package world

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/tactic"

	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/enemy"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/hero"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
)

const (
	lootChance    = 0.6
	enchantChance = 0.3
)

// Location - локация. Каждая конкретная локация сама решает, какого врага создать (фабричный метод CreateEnemy)
type Location interface {
	Code() string
	DisplayName() string
	RecommendedLevel() int
	Catalog() *item.ItemCatalog
	// CreateEnemy - создание врага делегируется конкретным локациям для повышения гибкости
	CreateEnemy() enemy.Enemy
	// LootTable - коды предметов из каталога, которые можно найти в локации
	LootTable() []string
}

// baseLocation - общие данные всех локаций, конкретные локации встраивают эту структуру в себя
type baseLocation struct {
	code             string
	displayName      string
	recommendedLevel int
	catalog          *item.ItemCatalog
}

func (l *baseLocation) Code() string { return l.code }

func (l *baseLocation) DisplayName() string { return l.displayName }

func (l *baseLocation) RecommendedLevel() int { return l.recommendedLevel }

func (l *baseLocation) Catalog() *item.ItemCatalog { return l.catalog }

// Explore отправляет героя в экспедицию со случайным сидом
func Explore(location Location, h *hero.Hero) (ExpeditionReport, error) {
	return ExploreWithSeed(location, h, core.GetInstance().NextSeed())
}

// ExploreWithSeed - бизнес-логика, общая для всех локаций:
// герой встречает врага, сражается с ним и получает награду, если победил
func ExploreWithSeed(location Location, h *hero.Hero, seed uint64) (ExpeditionReport, error) {
	return ExploreWithTactic(location, h, tactic.BalancedTactic{}, seed)
}

func ExploreWithTactic(location Location, h *hero.Hero, strategy tactic.Tactic, seed uint64) (ExpeditionReport, error) {
	core.GetInstance().Reseed(seed)

	h.Rest()
	foe := location.CreateEnemy()
	if core.GetInstance().Chance(0.2) {
		foe = makeElite(foe)
	}
	battle := combat.NewBattle(h, foe, strategy).Run()

	var rewards Rewards
	if battle.Outcome == combat.Victory {
		var err error
		rewards, err = reward(location, h, foe)
		if err != nil {
			return ExpeditionReport{}, err
		}
	} else {
		rewards = punish(h)
	}

	return ExpeditionReport{Seed: seed, LocationCode: location.Code(), TacticCode: strategy.Code(), Battle: battle, Rewards: rewards}, nil
}

func reward(location Location, h *hero.Hero, foe enemy.Enemy) (Rewards, error) {
	experience := foe.ExperienceReward()
	gold := foe.RollGoldReward()
	loot, err := rollLoot(location)
	if err != nil {
		return Rewards{}, err
	}

	levelsGained := h.GainExperience(experience)
	h.AddGold(gold)
	if loot != nil {
		h.AddItem(loot)
	}

	return Rewards{Experience: experience, Gold: gold, Loot: loot, LevelsGained: levelsGained}, nil
}

func rollLoot(location Location) (item.Item, error) {
	dice := core.GetInstance()
	if !dice.Chance(lootChance) {
		return nil, nil
	}

	// добыча - это копия шаблона из каталога, ее можно менять, не боясь испортить шаблон
	table := location.LootTable()
	loot, err := location.Catalog().Create(table[dice.Index(len(table))])
	if err != nil {
		return nil, err
	}
	if dice.Chance(enchantChance) {
		possible := loot.PossibleEnchantments()
		if err := loot.Enchant(possible[dice.Index(len(possible))]); err != nil {
			return nil, err
		}
	}

	return loot, nil
}

// проигравший герой теряет половину золота
func punish(h *hero.Hero) Rewards {
	lostGold := h.Gold() / 2
	// потратить половину имеющегося золота можно всегда, поэтому ошибку здесь не проверяем
	_ = h.SpendGold(lostGold)
	return Rewards{Experience: 0, Gold: -lostGold, Loot: nil, LevelsGained: 0}
}

func makeElite(foe enemy.Enemy) enemy.Enemy {
	dice := core.GetInstance()
	both := dice.Chance(0.25)
	if dice.Chance(0.5) {
		foe = enemy.NewLegendaryEnemy(foe)
		if both {
			foe = enemy.NewWindfuryEnemy(foe)
		}
	} else {
		foe = enemy.NewWindfuryEnemy(foe)
		if both {
			foe = enemy.NewLegendaryEnemy(foe)
		}
	}
	if dice.Chance(0.3) {
		foe = enemy.NewArmoredEnemy(foe)
	}
	return foe
}
```

`Explore` и `ExploreWithSeed` используют `BalancedTactic`, а `ExploreWithTactic` принимает выбранную тактику.

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

**`labs/lab3/event/game_event_type.go`**

```go
package event

type GameEventType string

const (
	Victory GameEventType = "victory"
	Defeat  GameEventType = "defeat"
	LevelUp GameEventType = "level-up"
)
```

И само событие. Обратите внимание, что это просто данные: что произошло, с каким героем и в какой экспедиции:

**`labs/lab3/event/game_event.go`**

```go
package event

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/hero"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/world"
)

type GameEvent struct {
	Type       GameEventType
	Hero       *hero.Hero
	Expedition world.ExpeditionReport
}
```

Интерфейс для подписчиков содержит всего один метод, который вызывается при наступлении события, и определяет контракт обработки этого
изменения:

**`labs/lab3/event/game_event_listener.go`**

```go
package event

type GameEventListener interface{ OnEvent(GameEvent) }
type ListenerFunc func(GameEvent)

func (f ListenerFunc) OnEvent(e GameEvent) { f(e) }
```

`ListenerFunc` позволяет использовать обычную функцию как слушателя.

Именно издатель позволяет подписываться на определенный тип события. Также именно он оповещает подписчиков о
наступлении того или иного события:

**`labs/lab3/event/game_event_publisher.go`**

```go
package event

type subscription struct {
	id       int
	listener GameEventListener
}
type GameEventPublisher struct {
	listeners map[GameEventType][]subscription
	nextID    int
}

func (p *GameEventPublisher) Subscribe(t GameEventType, l GameEventListener) int {
	if p.listeners == nil {
		p.listeners = make(map[GameEventType][]subscription)
	}
	p.nextID++
	p.listeners[t] = append(p.listeners[t], subscription{p.nextID, l})
	return p.nextID
}
func (p *GameEventPublisher) Unsubscribe(t GameEventType, id int) {
	for i, s := range p.listeners[t] {
		if s.id == id {
			p.listeners[t] = append(p.listeners[t][:i], p.listeners[t][i+1:]...)
			return
		}
	}
}
func (p *GameEventPublisher) Publish(e GameEvent) {
	for _, s := range append([]subscription(nil), p.listeners[e.Type]...) {
		s.listener.OnEvent(e)
	}
}
```

`Subscribe` возвращает номер подписки: его можно передать в `Unsubscribe`, когда слушатель больше не нужен.

Подписчики хранятся в `listeners`, причем хранятся по типу ивента, то есть подписаться и реагировать они могут на отдельные типы, а не на
все.
Реакции вызываются в методе `Publish`. Когда в нашей системе происходит ивент – мы вызываем издателя, а он, по списку зарегистрированных
подписчиков, вызывает их
`OnEvent` методы, прокидывая туда информацию о ивенте.

Теперь реализуем двух конкретных подписчиков. Сначала того, который будет выдавать достижения:

**`labs/lab3/event/achievement_tracker.go`**

```go
package event

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/progress"
)

type AchievementTracker struct{}

func (*AchievementTracker) OnEvent(e GameEvent) {
	if e.Type == Victory {
		e.Hero.Unlock(progress.FirstBlood)
		if e.Expedition.Battle.Enemy.Code == "dragon" {
			e.Hero.Unlock(progress.DragonSlayer)
		}
		if loot := e.Expedition.Rewards.Loot; loot != nil && len(loot.Enchantments()) > 0 {
			e.Hero.Unlock(progress.Lucky)
		}
	} else if e.Type == LevelUp && e.Hero.Level() >= 5 {
		e.Hero.Unlock(progress.Veteran)
	}
}
```

А теперь второго, он будет записывать экспедиции в журнал:

**`labs/lab3/event/expedition_journal.go`**

```go
package event

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/world"
	"slices"
)

type ExpeditionJournal struct {
	entries map[string][]world.ExpeditionReport
}

func (j *ExpeditionJournal) OnEvent(e GameEvent) {
	if j.entries == nil {
		j.entries = make(map[string][]world.ExpeditionReport)
	}
	j.entries[e.Hero.Name()] = append(j.entries[e.Hero.Name()], e.Expedition)
}
func (j *ExpeditionJournal) Entries(name string) []world.ExpeditionReport {
	return slices.Clone(j.entries[name])
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
скелет и дракон передают свое сопротивление в `baseEnemy`. Мы же хотим добавить новые механики: следопыт должен иногда
уворачиваться от ударов, а маг - ставить перед экспедицией магический барьер, который забирает на себя урон, пока не
иссякнет. И не факт, что список конечный: через несколько релизов может появиться еще несколько механик защиты.

Обработка всех этих эффектов в одном методе кажется нам излишне сложной. В целом с учетом того, что защита
срабатывает пошагово (сначала уклонение, потом броня, потом барьер), здесь можно применить паттерн *Цепочка
обязанностей*.

Он позволяет передавать запрос по цепочке обработчиков: каждый обработчик сам решает, обработать запрос
и передать его дальше или остановить его обработку. Отправитель запроса при этом не знает, сколько в цепочке звеньев и
какие именно они.

Цепочка наша будет обрабатывать уже объявленный `Damage`. Осталось определить обработчиков, которые изменяют `Damage` и передают дальше.

Объявим интерфейс обработчика и общую структуру `baseHandler`. Контракт содержит два метода: установка следующего звена для текущего (с помощью него мы и будем
строить цепочку) и обработка урона:

**`labs/lab3/defense/damage_handler.go`**

```go
package defense

type DamageHandler interface {
	Handle(*Damage)
	SetNext(DamageHandler) DamageHandler
}
type baseHandler struct{ next DamageHandler }

func (h *baseHandler) SetNext(next DamageHandler) DamageHandler { h.next = next; return next }
func (h *baseHandler) passToNext(d *Damage) {
	if h.next != nil {
		h.next.Handle(d)
	}
}
```

Теперь конкретный обработчик, который будет хендлить уклонение: если оно сработало, удар по герою не прошел, и передавать урон дальше по
цепочке незачем:

**`labs/lab3/defense/evasion_handler.go`**

```go
package defense

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
)

type EvasionHandler struct {
	baseHandler
	chance float64
}

func NewEvasionHandler(chance float64) *EvasionHandler { return &EvasionHandler{chance: chance} }
func (h *EvasionHandler) Handle(d *Damage) {
	if core.GetInstance().Chance(h.chance) {
		d.Evade()
		return
	}
	h.passToNext(d)
}
```

Броня гасит долю урона и передает то, что осталось, следующему обработчику. Обратите внимание, что это то же самое правило,
что раньше было зашито в герое, скелете и драконе, а теперь оно записано в одном месте:

**`labs/lab3/defense/armor_handler.go`**

```go
package defense

import (
	"math"
)

type ArmorHandler struct {
	baseHandler
	defense float64
}

func NewArmorHandler(defense float64) *ArmorHandler { return &ArmorHandler{defense: defense} }
func (h *ArmorHandler) Handle(d *Damage) {
	d.ReduceTo(max(1, int(math.Round(float64(d.Amount)*(1-h.defense)))))
	h.passToNext(d)
}
```

И магический барьер, который поглощает урон, пока у него есть запас прочности:

**`labs/lab3/defense/barrier_handler.go`**

```go
package defense

type BarrierHandler struct {
	baseHandler
	strength int
}

func NewBarrierHandler(strength int) *BarrierHandler { return &BarrierHandler{strength: strength} }
func (h *BarrierHandler) Handle(d *Damage) {
	absorbed := min(h.strength, d.Amount)
	if absorbed > 0 {
		h.strength -= absorbed
		d.Absorb(absorbed)
	}
	h.passToNext(d)
}
```

Какие звенья будут в цепочке, зависит от героя. Цепочку собирает его класс - так же, как класс сам знает свою
фабрику стартового снаряжения:

**`labs/lab3/hero/hero_class.go`**

```go
package hero

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/defense"

	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/kit"
)

// HeroClass - класс героя. Поля неэкспортируемые, поэтому классы героя - это только значения,
// заранее созданные в этом пакете (Warrior, Ranger, Mage)
type HeroClass struct {
	code        string
	displayName string
	baseHealth  int
	// функция, которая создает фабрику стартового снаряжения
	starterKit func(catalog *item.ItemCatalog) kit.StarterKitFactory
}

var (
	Warrior = &HeroClass{"warrior", "Воин", 120, kit.NewWarriorKitFactory}
	Ranger  = &HeroClass{"ranger", "Следопыт", 100, kit.NewRangerKitFactory}
	Mage    = &HeroClass{"mage", "Маг", 85, kit.NewMageKitFactory}
)

// HeroClasses возвращает все классы героя по порядку
func HeroClasses() []*HeroClass {
	return []*HeroClass{Warrior, Ranger, Mage}
}

func HeroClassByCode(code string) (*HeroClass, error) {
	for _, heroClass := range HeroClasses() {
		if heroClass.code == code {
			return heroClass, nil
		}
	}

	return nil, &core.UnknownCodeError{What: "Класс героя", Code: code}
}

// StarterKit - каждый класс сам знает свою фабрику стартового снаряжения
func (c *HeroClass) StarterKit(catalog *item.ItemCatalog) kit.StarterKitFactory {
	return c.starterKit(catalog)
}

func (c *HeroClass) Code() string { return c.code }

func (c *HeroClass) DisplayName() string { return c.displayName }

func (c *HeroClass) BaseHealth() int { return c.baseHealth }

func (c *HeroClass) CreateDefense(armor *item.Armor, level int) defense.DamageHandler {
	evasion, barrier := 0.0, 0
	if c == Ranger {
		evasion = 0.15
	}
	if c == Mage {
		barrier = 30
	}
	if c == Warrior && level >= 3 {
		barrier = 20
	} else if c == Ranger && level >= 5 {
		barrier = 15
	} else if c == Mage && level >= 3 {
		evasion = 0.10
	}
	armorHandler := defense.NewArmorHandler(armor.Defense())
	var first defense.DamageHandler = armorHandler
	if evasion > 0 {
		first = defense.NewEvasionHandler(evasion)
		first.SetNext(armorHandler)
	}
	if barrier > 0 {
		armorHandler.SetNext(defense.NewBarrierHandler(barrier))
	}
	return first
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

**`labs/lab3/hero/hero.go`**

```go
package hero

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/defense"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/progress"

	"errors"
	"fmt"
	"slices"
	"strings"
	"unicode/utf8"

	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/kit"
)

const (
	maxNameLength      = 20
	startingGold       = 50
	healthPerLevel     = 15
	damagePerLevel     = 3
	experiencePerLevel = 100
)

// Hero - герой. Все поля неэкспортируемые, поэтому вне пакета hero собрать героя можно только через строителя
type Hero struct {
	name         string
	heroClass    *HeroClass
	level        int
	experience   int
	gold         int
	health       int
	weapon       item.Weapon
	armor        *item.Armor
	inventory    []item.Item
	achievements []*progress.Achievement
	defense      defense.DamageHandler
}

// Builder - строитель, чтобы удобно и безопасно создавать героев
type Builder struct {
	name         string
	heroClass    *HeroClass
	weapon       item.Weapon
	armor        *item.Armor
	level        int
	gold         int
	experience   int
	inventory    []item.Item
	achievements []*progress.Achievement
	// первая ошибка, случившаяся при заполнении полей; ее вернет Build
	err error
}

// NewBuilder создает строителя: необязательные поля сразу получают значения по умолчанию
func NewBuilder() *Builder {
	return &Builder{level: 1, gold: startingGold}
}

func (b *Builder) SetExperience(value int) *Builder { b.experience = value; return b }
func (b *Builder) SetInventory(items []item.Item) *Builder {
	b.inventory = slices.Clone(items)
	return b
}
func (b *Builder) SetAchievements(items []*progress.Achievement) *Builder {
	b.achievements = slices.Clone(items)
	return b
}

func (b *Builder) SetName(name string) *Builder {
	b.name = name
	return b
}

func (b *Builder) SetHeroClass(heroClass *HeroClass) *Builder {
	b.heroClass = heroClass
	return b
}

func (b *Builder) SetWeapon(weapon item.Weapon) *Builder {
	b.weapon = weapon
	return b
}

func (b *Builder) SetArmor(armor *item.Armor) *Builder {
	b.armor = armor
	return b
}

// SetStarterKit берет оружие и броню из стартового набора
func (b *Builder) SetStarterKit(starterKit kit.StarterKitFactory) *Builder {
	weapon, err := starterKit.CreateWeapon()
	if err != nil {
		b.fail(err)
		return b
	}
	armor, err := starterKit.CreateArmor()
	if err != nil {
		b.fail(err)
		return b
	}

	b.weapon = weapon
	b.armor = armor
	return b
}

func (b *Builder) SetLevel(level int) *Builder {
	b.level = level
	return b
}

func (b *Builder) SetGold(gold int) *Builder {
	b.gold = gold
	return b
}

// Build проверяет, что все обязательные поля заполнены корректно, и только тогда создает героя
func (b *Builder) Build() (*Hero, error) {
	if b.err != nil {
		return nil, b.err
	}
	if strings.TrimSpace(b.name) == "" {
		return nil, errors.New("у героя должно быть имя")
	}
	if utf8.RuneCountInString(b.name) > maxNameLength {
		return nil, fmt.Errorf("имя героя не может быть длиннее %d символов", maxNameLength)
	}
	if b.heroClass == nil {
		return nil, errors.New("не выбран класс героя")
	}
	if b.weapon == nil || b.armor == nil {
		return nil, errors.New("герою нужны оружие и броня")
	}
	if b.level < 1 {
		return nil, errors.New("уровень героя не может быть меньше 1")
	}
	if b.gold < 0 {
		return nil, errors.New("золота не может быть меньше нуля")
	}

	if b.experience < 0 {
		return nil, errors.New("опыт не может быть отрицательным")
	}
	hero := &Hero{
		experience: b.experience, inventory: slices.Clone(b.inventory),
		name:      b.name,
		heroClass: b.heroClass,
		level:     b.level,
		gold:      b.gold,
		weapon:    b.weapon,
		armor:     b.armor,
	}
	for _, a := range b.achievements {
		hero.Unlock(a)
	}
	hero.Rest()
	return hero, nil
}

func (b *Builder) fail(err error) {
	if b.err == nil {
		b.err = err
	}
}

func (h *Hero) RollDamage() combat.Hit {
	hit := h.weapon.RollDamage()
	return combat.Hit{Damage: hit.Damage + (h.level-1)*damagePerLevel, Critical: hit.Critical}
}

func (h *Hero) ReceiveHit(amount int) defense.Damage {
	d := defense.Damage{Amount: amount}
	h.defense.Handle(&d)
	h.health = max(0, h.health-d.Amount)
	return d
}

func (h *Hero) IsAlive() bool {
	return h.health > 0
}

// поведение героя между экспедициями

// Rest - перед каждой экспедицией герой успевает отдохнуть
func (h *Hero) Rest() {
	h.health = h.MaxHealth()
	h.defense = h.heroClass.CreateDefense(h.armor, h.level)
}

// GainExperience начисляет опыт и возвращает количество полученных уровней
func (h *Hero) GainExperience(amount int) int {
	h.experience += amount
	levelsGained := 0
	for h.experience >= h.ExperienceToNextLevel() {
		h.experience -= h.ExperienceToNextLevel()
		h.level++
		levelsGained++
	}

	return levelsGained
}

func (h *Hero) AddGold(amount int) {
	h.gold += amount
}

func (h *Hero) SpendGold(amount int) error {
	if amount > h.gold {
		return fmt.Errorf("не хватает золота: нужно %d, есть %d", amount, h.gold)
	}

	h.gold -= amount
	return nil
}

func (h *Hero) AddItem(it item.Item) {
	h.inventory = append(h.inventory, it)
}

func (h *Hero) RemoveItem(it item.Item) error {
	index := slices.Index(h.inventory, it)
	if index < 0 {
		return fmt.Errorf("в инвентаре нет предмета %s", it.Name())
	}

	h.inventory = slices.Delete(h.inventory, index, index+1)
	return nil
}

// Equip экипирует предмет из инвентаря, снятый предмет возвращается в инвентарь
func (h *Hero) Equip(it item.Item) error {
	// switch по типу: в каждой ветке equipment имеет конкретный тип
	switch equipment := it.(type) {
	case item.Weapon:
		if err := h.RemoveItem(equipment); err != nil {
			return err
		}
		h.inventory = append(h.inventory, h.weapon)
		h.weapon = equipment
	case *item.Armor:
		if err := h.RemoveItem(equipment); err != nil {
			return err
		}
		h.inventory = append(h.inventory, h.armor)
		h.armor = equipment
		h.defense = h.heroClass.CreateDefense(h.armor, h.level)
	default:
		return fmt.Errorf("предмет нельзя экипировать: %s", it.Name())
	}

	return nil
}

// геттеры

func (h *Hero) Code() string { return h.heroClass.Code() }

func (h *Hero) Name() string { return h.name }

func (h *Hero) Health() int { return h.health }

func (h *Hero) MaxHealth() int {
	return h.heroClass.BaseHealth() + (h.level-1)*healthPerLevel
}

func (h *Hero) HeroClass() *HeroClass { return h.heroClass }

func (h *Hero) Level() int { return h.level }

func (h *Hero) Experience() int { return h.experience }

func (h *Hero) ExperienceToNextLevel() int { return h.level * experiencePerLevel }

func (h *Hero) Gold() int { return h.gold }

func (h *Hero) Weapon() item.Weapon { return h.weapon }

func (h *Hero) Armor() *item.Armor { return h.armor }

// Inventory возвращает копию списка предметов, чтобы инвентарь нельзя было поменять снаружи
func (h *Hero) Inventory() []item.Item {
	return slices.Clone(h.inventory)
}

func (h *Hero) Unlock(a *progress.Achievement) bool {
	if slices.Contains(h.achievements, a) {
		return false
	}
	h.achievements = append(h.achievements, a)
	return true
}
func (h *Hero) Achievements() []*progress.Achievement { return slices.Clone(h.achievements) }
```

У врагов цепочка задается прямо в конструкторе. Общая структура врага пропускает урон через цепочку, если она есть:

**`labs/lab3/enemy/base_enemy.go`**

```go
package enemy

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/defense"

	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
)

// baseEnemy - общая логика для врагов: здоровье, урон в диапазоне, защита и награда.
// Конкретные враги встраивают эту структуру в себя
type baseEnemy struct {
	code      string
	name      string
	maxHealth int
	minDamage int
	maxDamage int
	// цепочка защиты
	defense          defense.DamageHandler
	experienceReward int
	goldReward       int
	health           int
}

func newBaseEnemy(code, name string, maxHealth, minDamage, maxDamage int, protection defense.DamageHandler, experienceReward, goldReward int) baseEnemy {
	return baseEnemy{code, name, maxHealth, minDamage, maxDamage, protection, experienceReward, goldReward, maxHealth}
}

func (e *baseEnemy) RollDamage() combat.Hit {
	return combat.Hit{Damage: core.GetInstance().Roll(e.minDamage, e.maxDamage), Critical: false}
}

func (e *baseEnemy) ReceiveHit(amount int) defense.Damage {
	d := defense.Damage{Amount: amount}
	if e.defense != nil {
		e.defense.Handle(&d)
	}
	e.health = max(0, e.health-d.Amount)
	return d
}

func (e *baseEnemy) IsAlive() bool { return e.health > 0 }

func (e *baseEnemy) RollGoldReward() int {
	return core.GetInstance().Roll(e.goldReward/2, e.goldReward)
}

func (e *baseEnemy) ExperienceReward() int { return e.experienceReward }

func (e *baseEnemy) Code() string { return e.code }

func (e *baseEnemy) Name() string { return e.name }

func (e *baseEnemy) Health() int { return e.health }

func (e *baseEnemy) MaxHealth() int { return e.maxHealth }
```

У гоблина защиты нет:

**`labs/lab3/enemy/goblin.go`**

```go
package enemy

// Goblin - самый простой враг, без особенностей
type Goblin struct {
	baseEnemy
}

func NewGoblin() *Goblin {
	return &Goblin{newBaseEnemy("goblin", "Гоблин", 50, 6, 14, nil, 40, 30)}
}
```

Кости скелета и чешуя дракона работают как броня, поэтому передадим им обработчик брони вместо числового сопротивления:

**`labs/lab3/enemy/skeleton.go`**

```go
package enemy

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/defense"
)

// Skeleton - кости плохо поддаются ударам, поэтому часть урона проходит мимо
type Skeleton struct {
	baseEnemy
}

func NewSkeleton() *Skeleton {
	return &Skeleton{newBaseEnemy("skeleton", "Скелет", 110, 10, 24, defense.NewArmorHandler(0.3), 90, 70)}
}
```

**`labs/lab3/enemy/dragon.go`**

```go
package enemy

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/defense"

	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
)

const (
	fireBreathChance     = 0.25
	fireBreathMultiplier = 2
)

// Dragon - дракон защищен чешуей и время от времени дышит огнем
type Dragon struct {
	baseEnemy
}

func NewDragon() *Dragon {
	return &Dragon{newBaseEnemy("dragon", "Дракон", 260, 16, 22, defense.NewArmorHandler(0.25), 500, 400)}
}

// RollDamage заменяет метод встроенной структуры: бой вызывает его через интерфейс, поэтому сработает именно этот метод
func (d *Dragon) RollDamage() combat.Hit {
	bite := d.baseEnemy.RollDamage()
	if core.GetInstance().Chance(fireBreathChance) {
		return combat.Hit{Damage: bite.Damage * fireBreathMultiplier, Critical: true}
	}

	return bite
}
```

Супер! Теперь, чтобы добавить новую механику защиты, достаточно написать новое звено и поставить его в нужное место
цепочки, ни герой, ни враги при этом не меняются.

Применять цепочку стоит, когда запрос обрабатывается в несколько шагов и набор или порядок шагов должен меняться без
переписывания кода, который этот запрос отправляет.

## Игра целиком (Фасад / Facade)

Посмотрите на метод `main` из прошлой лабы: он сам создает каталог, локации и торговца, сам собирает героя строителем
и сам берет у класса фабрику стартового снаряжения. Теперь к этому добавятся тактики, хранилище с заместителем,
издатель событий и подписчики. Если оставить все это в консольном коде, он будет знать обо всех типах игры, а
заменить консоль на что-то другое (спойлер-спойлер) станет очень непросто.

Поэтому давайте воспользуемся паттерном *Фасад*. Он предоставляет простой интерфейс к сложной подсистеме из многих
компонентов: вся настройка и все связи между этими компонентами спрятаны внутри фасада, а снаружи видны только несколько
понятных методов.

Наш фасад - это вся игра целиком:

**`labs/lab3/game/game_facade.go`**

```go
package game

import (
	"fmt"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/camp"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/event"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/hero"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/storage"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/tactic"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/world"
	"slices"
)

type GameFacade struct {
	catalog   *item.ItemCatalog
	locations *world.LocationRegistry
	merchant  *camp.Merchant
	tactics   []tactic.Tactic
	heroes    storage.HeroRepository
	events    event.GameEventPublisher
	journal   *event.ExpeditionJournal
}

func NewGameFacade(saveFile string) *GameFacade {
	catalog := item.NewDefaultCatalog()
	g := &GameFacade{catalog: catalog, locations: world.NewLocationRegistry(world.NewForest(catalog), world.NewHauntedManor(catalog), world.NewCrypt(catalog), world.NewDragonLair(catalog)),
		merchant: camp.NewMerchant(catalog, item.IronSword, item.ElvenBow, item.EmberStaff, item.Chainmail, item.PlateArmor),
		tactics:  []tactic.Tactic{tactic.BalancedTactic{}, tactic.AggressiveTactic{}, tactic.CautiousTactic{}},
		heroes:   storage.NewCachingHeroRepository(storage.NewFileHeroRepository(saveFile, catalog)), journal: &event.ExpeditionJournal{}}
	a := &event.AchievementTracker{}
	g.events.Subscribe(event.Victory, a)
	g.events.Subscribe(event.LevelUp, a)
	g.events.Subscribe(event.Victory, g.journal)
	g.events.Subscribe(event.Defeat, g.journal)
	return g
}
func (g *GameFacade) HeroClasses() []*hero.HeroClass { return hero.HeroClasses() }
func (g *GameFacade) StarterKit(code string) ([]item.Item, error) {
	hc, err := hero.HeroClassByCode(code)
	if err != nil {
		return nil, err
	}
	kit := hc.StarterKit(g.catalog)
	w, err := kit.CreateWeapon()
	if err != nil {
		return nil, err
	}
	a, err := kit.CreateArmor()
	if err != nil {
		return nil, err
	}
	return []item.Item{w, a}, nil
}
func (g *GameFacade) Locations() []world.Location                  { return g.locations.All() }
func (g *GameFacade) Location(code string) (world.Location, error) { return g.locations.Get(code) }
func (g *GameFacade) Tactics() []tactic.Tactic                     { return slices.Clone(g.tactics) }
func (g *GameFacade) Tactic(code string) (tactic.Tactic, error) {
	for _, t := range g.tactics {
		if t.Code() == code {
			return t, nil
		}
	}
	return nil, &core.UnknownCodeError{What: "Тактика", Code: code}
}
func (g *GameFacade) Showcase() ([]item.Item, error) { return g.merchant.Showcase() }
func (g *GameFacade) SellPrice(it item.Item) int     { return g.merchant.SellPrice(it) }
func (g *GameFacade) Heroes() ([]*hero.Hero, error)  { return g.heroes.FindAll() }
func (g *GameFacade) Hero(name string) (*hero.Hero, error) {
	h, err := g.heroes.FindByName(name)
	if err != nil {
		return nil, err
	}
	if h == nil {
		return nil, fmt.Errorf("героя с именем %s нет", name)
	}
	return h, nil
}
func (g *GameFacade) CreateHero(name, code string) (*hero.Hero, error) {
	found, err := g.heroes.FindByName(name)
	if err != nil {
		return nil, err
	}
	if found != nil {
		return nil, fmt.Errorf("герой с именем %s уже есть", name)
	}
	hc, err := hero.HeroClassByCode(code)
	if err != nil {
		return nil, err
	}
	h, err := hero.NewBuilder().SetName(name).SetHeroClass(hc).SetStarterKit(hc.StarterKit(g.catalog)).Build()
	if err != nil {
		return nil, err
	}
	return h, g.heroes.Save(h)
}
func (g *GameFacade) Explore(name, locationCode, tacticCode string) (world.ExpeditionReport, error) {
	h, err := g.Hero(name)
	if err != nil {
		return world.ExpeditionReport{}, err
	}
	l, err := g.Location(locationCode)
	if err != nil {
		return world.ExpeditionReport{}, err
	}
	t, err := g.Tactic(tacticCode)
	if err != nil {
		return world.ExpeditionReport{}, err
	}
	report, err := world.ExploreWithTactic(l, h, t, core.GetInstance().NextSeed())
	if err != nil {
		return report, err
	}
	result := event.Defeat
	if report.Battle.Outcome == combat.Victory {
		result = event.Victory
	}
	g.events.Publish(event.GameEvent{Type: result, Hero: h, Expedition: report})
	if report.Rewards.LevelsGained > 0 {
		g.events.Publish(event.GameEvent{Type: event.LevelUp, Hero: h, Expedition: report})
	}
	return report, g.heroes.Save(h)
}
func (g *GameFacade) Equip(name string, index int) error {
	h, err := g.Hero(name)
	if err != nil {
		return err
	}
	inv := h.Inventory()
	if index < 0 || index >= len(inv) {
		return fmt.Errorf("нет предмета с таким номером")
	}
	if err = h.Equip(inv[index]); err != nil {
		return err
	}
	return g.heroes.Save(h)
}
func (g *GameFacade) Buy(name, code string) (item.Item, error) {
	h, err := g.Hero(name)
	if err != nil {
		return nil, err
	}
	it, err := g.merchant.Buy(h, code)
	if err != nil {
		return nil, err
	}
	return it, g.heroes.Save(h)
}
func (g *GameFacade) Sell(name string, index int) (int, error) {
	h, err := g.Hero(name)
	if err != nil {
		return 0, err
	}
	inv := h.Inventory()
	if index < 0 || index >= len(inv) {
		return 0, fmt.Errorf("нет предмета с таким номером")
	}
	price, err := g.merchant.Sell(h, inv[index])
	if err != nil {
		return 0, err
	}
	return price, g.heroes.Save(h)
}
func (g *GameFacade) Journal(name string) []world.ExpeditionReport { return g.journal.Entries(name) }
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
фасад не должен сам содержать игровую логику: он только связывает между собой компоненты, которые ее реализуют.
Применять фасад стоит, когда нужно дать простой вход в сложную подсистему или отделить от нее клиентский код.

## Консоль

Вывод в консоль научился показывать новые события боя, особенности классов и достижения:

**`labs/lab3/console/console_view.go`**

```go
package console

import (
	"fmt"
	"strings"

	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/hero"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/world"
)

// ConsoleView - весь вывод в консоль собран здесь
type ConsoleView struct{}

func (v ConsoleView) Describe(it item.Item) string {
	description := it.Name()

	enchantments := it.Enchantments()
	if len(enchantments) > 0 {
		names := make([]string, 0, len(enchantments))
		for _, enchantment := range enchantments {
			names = append(names, enchantment.Name())
		}
		description += " [" + strings.Join(names, ", ") + "]"
	}

	return description + " (" + it.Description() + ")"
}

func (v ConsoleView) PrintHero(h *hero.Hero) {
	fmt.Printf(
		"%s, %s %d ур. | здоровье %d | опыт %d/%d | золото %d\n",
		h.Name(), h.HeroClass().DisplayName(), h.Level(),
		h.MaxHealth(), h.Experience(), h.ExperienceToNextLevel(), h.Gold(),
	)
	fmt.Println("  Оружие: " + v.Describe(h.Weapon()))
	fmt.Println("  Броня:  " + v.Describe(h.Armor()))
	names := []string{}
	for _, a := range h.Achievements() {
		names = append(names, a.DisplayName())
	}
	if len(names) > 0 {
		fmt.Println("  Достижения: " + strings.Join(names, ", "))
	}
}

func (v ConsoleView) PrintReport(report world.ExpeditionReport, locationName string) {
	battle := report.Battle
	fmt.Printf("=== %s (сид %d) ===\n", locationName, report.Seed)
	fmt.Printf("%s встречает врага: %s (%d здоровья)\n", battle.Hero.Name, battle.Enemy.Name, battle.Enemy.MaxHealth)

	for _, event := range battle.Events {
		v.printEvent(event, battle)
	}

	v.printRewards(battle, report.Rewards)
}

func (v ConsoleView) printEvent(event combat.BattleEvent, battle combat.BattleReport) {
	source := participant(event.Source, battle)
	target := participant(event.Target, battle)

	switch event.Type {
	case combat.EventHit:
		fmt.Printf(
			"[раунд %d] %s -> %s: %d урона, осталось %d/%d\n",
			event.Round, source.Name, target.Name, event.Amount, event.TargetHealthAfter, target.MaxHealth,
		)
	case combat.EventCriticalHit:
		fmt.Printf(
			"[раунд %d] %s -> %s: КРИТ! %d урона, осталось %d/%d\n",
			event.Round, source.Name, target.Name, event.Amount, event.TargetHealthAfter, target.MaxHealth,
		)
	case combat.EventEvade:
		fmt.Printf("[раунд %d] %s -> %s: промах, %s уклоняется\n", event.Round, source.Name, target.Name, target.Name)
	case combat.EventAbsorb:
		fmt.Printf("[раунд %d] %s -> %s: барьер поглощает %d урона\n", event.Round, source.Name, target.Name, event.Amount)
	case combat.EventPowerStrike:
		fmt.Printf("[раунд %d] %s бьет в полную силу\n", event.Round, source.Name)
	case combat.EventDefend:
		fmt.Printf("[раунд %d] %s уходит в оборону\n", event.Round, source.Name)
	case combat.EventDeath:
		fmt.Printf("%s повержен!\n", target.Name)
	}
}

func (v ConsoleView) printRewards(battle combat.BattleReport, rewards world.Rewards) {
	if battle.Outcome == combat.Defeat {
		fmt.Printf("Поражение... %s едва уносит ноги и теряет %d золота\n", battle.Hero.Name, -rewards.Gold)
		return
	}

	fmt.Printf("Победа! +%d опыта, +%d золота\n", rewards.Experience, rewards.Gold)
	if rewards.Loot != nil {
		fmt.Println("Добыча: " + v.Describe(rewards.Loot))
	}
	if rewards.LevelsGained > 0 {
		fmt.Println("Новый уровень!")
	}
}

func participant(side combat.Side, battle combat.BattleReport) combat.Participant {
	if side == combat.SideHero {
		return battle.Hero
	}

	return battle.Enemy
}

func (ConsoleView) Summarize(report world.ExpeditionReport, locationName, tacticName string) string {
	result := "поражение"
	if report.Battle.Outcome == combat.Victory {
		result = "победа"
	}
	return fmt.Sprintf("%s, %s, тактика %q: %s", locationName, report.Battle.Enemy.Name, tacticName, result)
}
```

А игровой цикл в методе `main` теперь работает только с фасадом и хранит текущего героя:

Вместо прямой работы с каталогом, локациями и торговцем:

- создаём `GameFacade` с файлом сохранения и выбираем нового или сохранённого героя;
- перед экспедицией выбираем тактику;
- работам с экспедициями, экипировкой и торговлей через фасад;
- добавляем пункт меню для журнала экспедиций.

**`labs/lab3/main.go`**

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"strconv"
	"strings"

	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/console"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/game"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/hero"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
	"path/filepath"
)

var (
	input = bufio.NewScanner(os.Stdin)
	view  = console.ConsoleView{}
)

func main() {
	saveFile := filepath.Join("saves", "heroes.json")
	absolute, err := filepath.Abs(saveFile)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		return
	}
	fmt.Println("Файл сохранения: " + absolute)
	g := game.NewGameFacade(saveFile)
	h := chooseHero(g)
	fmt.Printf("%s разбивает лагерь на опушке мистического леса.\n", h.Name())

	for {
		fmt.Println()
		view.PrintHero(h)
		fmt.Println("1 - отправиться в экспедицию, 2 - снаряжение, 3 - торговец, 4 - журнал, 0 - выйти из игры")
		switch readLine() {
		case "1":
			goOnExpedition(h, g)
		case "2":
			manageEquipment(h, g)
		case "3":
			visitMerchant(h, g)
		case "4":
			for _, report := range g.Journal(h.Name()) {
				l, err := g.Location(report.LocationCode)
				if err != nil {
					fmt.Println(err)
					continue
				}
				t, err := g.Tactic(report.TacticCode)
				if err != nil {
					fmt.Println(err)
					continue
				}
				fmt.Println(view.Summarize(report, l.DisplayName(), t.DisplayName()))
			}
		case "0":
			return
		default:
			fmt.Println("Такого варианта нет")
		}
	}
}

func chooseHero(g *game.GameFacade) *hero.Hero {
	heroes, err := g.Heroes()
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	if len(heroes) == 0 {
		return createHero(g)
	}
	fmt.Println("Выберите сохраненного героя (0 - создать нового):")
	for i, h := range heroes {
		fmt.Printf("%d - %s, %s %d ур.\n", i+1, h.Name(), h.HeroClass().DisplayName(), h.Level())
	}
	index := readChoice(len(heroes))
	if index >= 0 {
		return heroes[index]
	}
	return createHero(g)
}

func createHero(g *game.GameFacade) *hero.Hero {
	fmt.Println("Создайте своего героя!")
	for {
		fmt.Println("Выберите класс:")
		heroClasses := hero.HeroClasses()
		for i, heroClass := range heroClasses {
			equipment, err := g.StarterKit(heroClass.Code())
			if err != nil {
				fmt.Println(err)
				continue
			}
			weapon, armor := equipment[0], equipment[1]
			fmt.Printf(
				"%d - %s (здоровье %d): %s, %s\n",
				i+1, heroClass.DisplayName(), heroClass.BaseHealth(), view.Describe(weapon), view.Describe(armor),
			)
		}
		classIndex := readChoice(len(heroClasses))
		if classIndex < 0 {
			continue
		}
		heroClass := heroClasses[classIndex]

		fmt.Println("Введите имя:")
		name := readLine()

		h, err := g.CreateHero(name, heroClass.Code())
		if err != nil {
			fmt.Println("Не получилось создать героя: " + err.Error())
			continue
		}

		return h
	}
}

func goOnExpedition(h *hero.Hero, g *game.GameFacade) {
	all := g.Locations()
	fmt.Println("Куда отправимся?")
	for i, location := range all {
		fmt.Printf("%d - %s (рекомендуемый уровень %d)\n", i+1, location.DisplayName(), location.RecommendedLevel())
	}
	index := readChoice(len(all))
	if index < 0 {
		return
	}

	location := all[index]
	fmt.Println("Выберите тактику:")
	tactics := g.Tactics()
	for i, t := range tactics {
		fmt.Printf("%d - %s: %s\n", i+1, t.DisplayName(), t.Description())
	}
	tacticIndex := readChoice(len(tactics))
	if tacticIndex < 0 {
		return
	}
	report, err := g.Explore(h.Name(), location.Code(), tactics[tacticIndex].Code())
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println()
	view.PrintReport(report, location.DisplayName())

	if report.Battle.Outcome == combat.Victory && report.Battle.Enemy.Code == "dragon" {
		fmt.Printf("\nДракон повержен! Барды еще долго будут петь о подвиге героя по имени %s.\n", h.Name())
	}
}

func manageEquipment(h *hero.Hero, g *game.GameFacade) {
	inventory := h.Inventory()
	if len(inventory) == 0 {
		fmt.Println("Инвентарь пуст")
		return
	}

	fmt.Println("Что экипировать? (0 - назад)")
	printItems(inventory)
	index := readChoice(len(inventory))
	if index < 0 {
		return
	}

	it := inventory[index]
	if err := g.Equip(h.Name(), index); err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println("Экипировано: " + view.Describe(it))
}

func visitMerchant(h *hero.Hero, g *game.GameFacade) {
	fmt.Printf("Торговец: \"Чего желаешь?\" (золото: %d)\n", h.Gold())
	fmt.Println("1 - купить, 2 - продать, 0 - назад")
	switch readLine() {
	case "1":
		buy(h, g)
	case "2":
		sell(h, g)
	}
}

func buy(h *hero.Hero, g *game.GameFacade) {
	showcase, err := g.Showcase()
	if err != nil {
		fmt.Println(err)
		return
	}
	for i, it := range showcase {
		fmt.Printf("%d - %s: %d золота\n", i+1, view.Describe(it), it.Price())
	}
	index := readChoice(len(showcase))
	if index < 0 {
		return
	}

	it, err := g.Buy(h.Name(), showcase[index].Code())
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println("Куплено: " + view.Describe(it))
}

func sell(h *hero.Hero, g *game.GameFacade) {
	inventory := h.Inventory()
	if len(inventory) == 0 {
		fmt.Println("Продавать нечего (экипированные предметы продать нельзя)")
		return
	}

	for i, it := range inventory {
		fmt.Printf("%d - %s: %d золота\n", i+1, view.Describe(it), g.SellPrice(it))
	}
	index := readChoice(len(inventory))
	if index < 0 {
		return
	}

	it := inventory[index]
	price, err := g.Sell(h.Name(), index)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Printf("Продано: %s за %d золота\n", it.Name(), price)
}

func printItems(items []item.Item) {
	for i, it := range items {
		fmt.Printf("%d - %s\n", i+1, view.Describe(it))
	}
}

// readLine читает строку без пробелов по краям; если ввод закончился, завершает программу
func readLine() string {
	if !input.Scan() {
		os.Exit(0)
	}

	return strings.TrimSpace(input.Text())
}

// readChoice читает номер варианта от 1 до count, возвращает индекс или -1, если ввели что-то другое
func readChoice(count int) int {
	choice, err := strconv.Atoi(readLine())
	if err != nil || choice < 1 || choice > count {
		return -1
	}

	return choice - 1
}
```

Супер! Попробуйте запустить игру, сохраниться и вернуться к своему герою :)

## Демонстрация паттернов

Чтобы показать все паттерны одним запуском, напишем демонстрационную программу:

Демонстрацию лабы 3 заменяем новой:

**`labs/lab3/cmd/demo/main.go`**

```go
package main

import (
	"fmt"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/console"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/defense"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/enemy"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/event"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/game"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/hero"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/storage"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/tactic"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/world"
	"os"
	"path/filepath"
	"strings"
)

func must[T any](value T, err error) T {
	if err != nil {
		panic(err)
	}
	return value
}
func check(err error) {
	if err != nil {
		panic(err)
	}
}
func newHero(name string, hc *hero.HeroClass, level int, catalog *item.ItemCatalog) *hero.Hero {
	return must(hero.NewBuilder().SetName(name).SetHeroClass(hc).SetStarterKit(hc.StarterKit(catalog)).SetLevel(level).Build())
}
func printHits(title string, chain defense.DamageHandler, count int) {
	results := []string{}
	for i := 0; i < count; i++ {
		d := defense.Damage{Amount: 30}
		chain.Handle(&d)
		if d.Evaded {
			results = append(results, "уклонение")
		} else if d.Absorbed > 0 {
			results = append(results, fmt.Sprintf("поглощено %d, прошло %d", d.Absorbed, d.Amount))
		} else {
			results = append(results, fmt.Sprintf("прошло %d", d.Amount))
		}
	}
	fmt.Println(title + ": " + strings.Join(results, "; "))
}
func main() {
	view := console.ConsoleView{}
	catalog := item.NewDefaultCatalog()
	dice := core.GetInstance()
	fmt.Println("===== Цепочка обязанностей =====")
	printHits("Броня 30%", defense.NewArmorHandler(0.3), 2)
	armor := defense.NewArmorHandler(0.3)
	armor.SetNext(defense.NewBarrierHandler(30))
	printHits("Броня 30% -> барьер 30", armor, 3)
	dice.Reseed(3)
	chain := defense.NewEvasionHandler(0.5)
	chain.SetNext(defense.NewArmorHandler(0.3)).SetNext(defense.NewBarrierHandler(30))
	printHits("Уклонение 50% -> броня 30% -> барьер 30", chain, 3)
	fmt.Println("\n===== Стратегия =====")
	for _, t := range []tactic.Tactic{tactic.BalancedTactic{}, tactic.AggressiveTactic{}, tactic.CautiousTactic{}} {
		h := newHero("Мерлин", hero.Mage, 2, catalog)
		report := must(world.ExploreWithTactic(world.NewCrypt(catalog), h, t, 9))
		result := "поражение"
		if report.Battle.Outcome == combat.Victory {
			result = "победа"
		}
		fmt.Printf("%s: %s, раундов %d, у героя осталось %d/%d здоровья\n", t.DisplayName(), result, report.Battle.Events[len(report.Battle.Events)-1].Round, h.Health(), h.MaxHealth())
	}
	fmt.Println("\n===== Декоратор =====")
	for _, e := range []enemy.Enemy{enemy.NewGoblin(), enemy.NewLegendaryEnemy(enemy.NewGoblin()), enemy.NewWindfuryEnemy(enemy.NewGoblin()), enemy.NewLegendaryEnemy(enemy.NewWindfuryEnemy(enemy.NewGoblin())), enemy.NewArmoredEnemy(enemy.NewGoblin()), enemy.NewArmoredEnemy(enemy.NewLegendaryEnemy(enemy.NewWindfuryEnemy(enemy.NewDragon())))} {
		dice.Reseed(5)
		hits := []string{}
		for i := 0; i < 3; i++ {
			hits = append(hits, fmt.Sprint(e.RollDamage().Damage))
		}
		d := e.ReceiveHit(20)
		fmt.Printf("%s: удары %s, награда %d опыта; входящий урон 20 -> %d, здоровье %d/%d\n", e.Name(), strings.Join(hits, ", "), e.ExperienceReward(), d.Amount, e.Health(), e.MaxHealth())
	}
	fmt.Println("\n===== Адаптер =====")
	bow := must(catalog.CreateWeapon(item.ElvenBow))
	haunted := enemy.NewHauntedWeapon(bow)
	fmt.Println("Предмет: " + view.Describe(bow))
	fmt.Printf("Враг:    %s (%d здоровья, награда %d опыта)\n", haunted.Name(), haunted.MaxHealth(), haunted.ExperienceReward())
	manor := world.NewHauntedManor(catalog)
	view.PrintReport(must(world.ExploreWithTactic(manor, newHero("Леголас", hero.Ranger, 2, catalog), tactic.CautiousTactic{}, 55)), manor.DisplayName())
	fmt.Println("\n===== Наблюдатель =====")
	events := &event.GameEventPublisher{}
	journal := &event.ExpeditionJournal{}
	events.Subscribe(event.Victory, &event.AchievementTracker{})
	events.Subscribe(event.Victory, journal)
	events.Subscribe(event.Defeat, journal)
	events.Subscribe(event.Victory, event.ListenerFunc(func(e event.GameEvent) { fmt.Printf("Событие %s: герой %s\n", e.Type, e.Hero.Name()) }))
	warrior := newHero("Артур", hero.Warrior, 1, catalog)
	report := must(world.ExploreWithSeed(world.NewForest(catalog), warrior, 7))
	result := event.Defeat
	if report.Battle.Outcome == combat.Victory {
		result = event.Victory
	}
	events.Publish(event.GameEvent{Type: result, Hero: warrior, Expedition: report})
	names := []string{}
	for _, a := range warrior.Achievements() {
		names = append(names, a.DisplayName())
	}
	fmt.Println("Достижения героя: " + strings.Join(names, ", "))
	fmt.Println("Записей в журнале:", len(journal.Entries(warrior.Name())))
	directory := must(os.MkdirTemp("", "lab-demo-"))
	defer os.RemoveAll(directory)
	fmt.Println("\n===== Заместитель =====")
	path := filepath.Join(directory, "heroes.json")
	file := storage.NewFileHeroRepository(path, catalog)
	check(file.Save(warrior))
	fmt.Println("Содержимое файла:")
	fmt.Print(string(must(os.ReadFile(path))))
	fmt.Println("Без заместителя два поиска дают один и тот же объект:", must(file.FindByName("Артур")) == must(file.FindByName("Артур")))
	cached := storage.NewCachingHeroRepository(file)
	h := must(cached.FindByName("Артур"))
	fmt.Println("С заместителем два поиска дают один и тот же объект:", h == must(cached.FindByName("Артур")))
	h.AddGold(100)
	check(cached.Save(h))
	fmt.Println("После сохранения через заместитель золото героя в файле:", must(file.FindByName("Артур")).Gold())
	fmt.Println("\n===== Фасад =====")
	g := game.NewGameFacade(filepath.Join(directory, "game.json"))
	must(g.CreateHero("Гэндальф", "mage"))
	dice.Reseed(11)
	expedition := must(g.Explore("Гэндальф", "forest", "cautious"))
	fmt.Println("Экспедиция через фасад: " + view.Summarize(expedition, must(g.Location(expedition.LocationCode)).DisplayName(), must(g.Tactic(expedition.TacticCode)).DisplayName()))
	view.PrintHero(must(g.Hero("Гэндальф")))
}
```

Вывод программы:

```text
===== Цепочка обязанностей =====
Броня 30%: прошло 21; прошло 21
Броня 30% -> барьер 30: поглощено 21, прошло 0; поглощено 9, прошло 12; прошло 21
Уклонение 50% -> броня 30% -> барьер 30: уклонение; уклонение; поглощено 21, прошло 0

===== Стратегия =====
Баланс: победа, раундов 9, у героя осталось 26/100 здоровья
Натиск: победа, раундов 9, у героя осталось 17/100 здоровья
Осторожность: победа, раундов 10, у героя осталось 21/100 здоровья

===== Декоратор =====
Гоблин: удары 12, 10, 8, награда 40 опыта; входящий урон 20 -> 20, здоровье 30/50
Легендарный Гоблин: удары 24, 20, 16, награда 80 опыта; входящий урон 20 -> 20, здоровье 30/50
Неистовый Гоблин: удары 22, 15, 15, награда 60 опыта; входящий урон 20 -> 20, здоровье 30/50
Легендарный Неистовый Гоблин: удары 44, 30, 30, награда 120 опыта; входящий урон 20 -> 20, здоровье 30/50
Бронированный Гоблин: удары 12, 10, 8, награда 40 опыта; входящий урон 20 -> 10, здоровье 40/50
Бронированный Легендарный Неистовый Дракон: удары 74, 98, 78, награда 1500 опыта; входящий урон 20 -> 8, здоровье 252/260

===== Адаптер =====
Предмет: Эльфийский лук (урон 14, шанс крита 40%)
Враг:    Одержимый Эльфийский лук (70 здоровья, награда 45 опыта)
=== Проклятый особняк (сид 55) ===
Леголас встречает врага: Легендарный Одержимый Эльфийский лук (70 здоровья)
[раунд 1] Леголас -> Легендарный Одержимый Эльфийский лук: 13 урона, осталось 57/70
[раунд 1] Легендарный Одержимый Эльфийский лук -> Леголас: 22 урона, осталось 93/115
[раунд 2] Леголас -> Легендарный Одержимый Эльфийский лук: КРИТ! 23 урона, осталось 34/70
[раунд 2] Легендарный Одержимый Эльфийский лук -> Леголас: 22 урона, осталось 71/115
[раунд 3] Леголас -> Легендарный Одержимый Эльфийский лук: 13 урона, осталось 21/70
[раунд 3] Легендарный Одержимый Эльфийский лук -> Леголас: 22 урона, осталось 49/115
[раунд 4] Леголас -> Легендарный Одержимый Эльфийский лук: 13 урона, осталось 8/70
[раунд 4] Легендарный Одержимый Эльфийский лук -> Леголас: 22 урона, осталось 27/115
[раунд 5] Леголас -> Легендарный Одержимый Эльфийский лук: 13 урона, осталось 0/70
Легендарный Одержимый Эльфийский лук повержен!
Победа! +90 опыта, +46 золота

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
    "level": 1,
    "experience": 80,
    "gold": 110,
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
        "code": "hunting-bow",
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
После сохранения через заместитель золото героя в файле: 210

===== Фасад =====
Экспедиция через фасад: Мистический лес, Гоблин, тактика "Осторожность": победа
Гэндальф, Маг 1 ур. | здоровье 85 | опыт 40/100 | золото 73
  Оружие: Посох ученика (урон 10-18)
  Броня:  Льняная роба (защита 10%)
  Достижения: Первая кровь, Счастливчик
```

Не забудьте закоммитить и запушить изменения!

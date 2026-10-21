# Лаба 3 — практика на Go

Это практическая часть [лабораторной работы 3](../laba3.md): требования к сдаче и вопросы к защите находятся в
основном тексте лабы, а здесь шаг за шагом разобрано эталонное решение.

Понадобится Go 1.22 или новее, сторонние библиотеки не нужны. Шаблон репозитория: [tech-prog-labs-2026-go](https://github.com/tidbid-kt/tech-prog-labs-2026-go). Код лабы пишите в папке `labs/lab3`.

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

```go
package core

import (
	"math/rand/v2"
	"sync"
)

// dice - одиночка: общий источник случайности на всю игру.
// Тип неэкспортируемый (с маленькой буквы), поэтому вне пакета core создать кость нельзя.
type dice struct {
	random *rand.Rand
}

var (
	// единственный экземпляр
	instance *dice
	once     sync.Once
)

// GetInstance возвращает единственный экземпляр кости
func GetInstance() *dice {
	// sync.Once выполнит функцию ровно один раз, даже если GetInstance одновременно вызовут несколько горутин
	once.Do(func() {
		instance = &dice{random: rand.New(rand.NewPCG(rand.Uint64(), rand.Uint64()))}
	})

	return instance
}

// Reseed задает сид: после этого все последующие броски повторяются в точности
func (d *dice) Reseed(seed uint64) {
	d.random = rand.New(rand.NewPCG(seed, seed))
}

// NextSeed возвращает случайный сид
func (d *dice) NextSeed() uint64 {
	return d.random.Uint64N(1_000_000)
}

// Roll возвращает случайное целое число от low до high включительно
func (d *dice) Roll(low, high int) int {
	return low + d.random.IntN(high-low+1)
}

// Chance возвращает true с вероятностью probability (от 0 до 1)
func (d *dice) Chance(probability float64) bool {
	return d.random.Float64() < probability
}

// Index возвращает случайный индекс от 0 до size-1, например чтобы выбрать элемент слайса
func (d *dice) Index(size int) int {
	return d.random.IntN(size)
}
```

В Go нет конструкторов и модификатора `private`, зато есть экспортируемые и неэкспортируемые имена: все, что начинается
с маленькой буквы, недоступно за пределами пакета. Тип кости `dice` неэкспортируемый, поэтому создать его вне пакета
`core` нельзя, а получить кость можно только через `GetInstance()`. Создает единственный экземпляр `sync.Once`.

Давайте разберем, что в этой реализации важно.

1. Тип кости неэкспортируемый. Вне пакета `core` создать кость нельзя, получить ее можно только через `GetInstance()`.
2. Экземпляр хранится в переменной пакета и создается лениво, при первом вызове `GetInstance()`. Пока кость никому не
   нужна, ее нет.
3. Со звездочкой заметим, что такая реализация потокобезопасна: `sync.Once` гарантирует, что функция создания выполнится ровно один раз, даже если
   `GetInstance()` одновременно вызовут несколько горутин. Тему многопоточности мы здесь затрагивать не будем, но для самостоятельного изучения она очень интересна.

Преимущество одиночки очевидно - у всех составляющих игры (оружия, врагов, локаций) одна и та же кость, поэтому один сид задает все броски. 
А еще кость не нужно передавать через конструкторы каждого предмета и каждого врага - к ней можно
обратиться откуда угодно.

Но тут есть и обратная сторона: одиночка - это глобальное состояние. Лук, посох и враги зависят от игральной кости, однако
по их конструкторам этого не видно, а подменить кость на другую (например, в тестах) не получится. Поэтому применять
одиночку стоит только там, где объект действительно должен быть единственным.

Также давайте сразу заведем ошибку, которую будем возвращать, если по коду ничего не нашлось (неизвестная локация,
неизвестный предмет и т.д.), в будущем она нам пригодится:

```go
package core

import "fmt"

// UnknownCodeError - ошибка, когда по коду ничего не нашлось (неизвестная локация, предмет и т.д.)
type UnknownCodeError struct {
	What string
	Code string
}

func (e *UnknownCodeError) Error() string {
	return fmt.Sprintf("%s с кодом '%s' не существует", e.What, e.Code)
}
```

В Go нет исключений: функции, которые могут не сработать, возвращают ошибку последним значением, а вызывающий код
проверяет ее (`if err != nil`). Ошибкой может быть любой тип с методом `Error() string`, поэтому для неизвестного кода
мы завели свой тип.

## Боевая механика

Давайте определим базовые абстракции нашей игры.

Интерфейс сущности, которая умеет атаковать и получать урон - так с точки зрения механики выглядят и герой, и враги:

```go
package combat

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
	ReceiveHit(damage int) int
}
```

Интерфейсы в Go реализуются неявно: писать `implements` не нужно, тип подходит под интерфейс, если у него есть все
перечисленные методы.

Удар представляет собой вот такую небольшую сущность:

```go
package combat

// Hit - удар: сколько урона и был ли он критическим
type Hit struct {
	Damage   int
	Critical bool
}
```

`Hit` - обычная структура с экспортируемыми (с большой буквы) полями.

Давайте теперь опишем механику боя. Зафиксируем стороны, типы событий и исход:

```go
package combat

type Side int

const (
	SideHero Side = iota
	SideEnemy
)
```

```go
package combat

type EventType int

const (
	EventHit EventType = iota
	EventCriticalHit
	EventDeath
)
```

```go
package combat

type Outcome int

const (
	Victory Outcome = iota
	Defeat
)
```

Бой делится на раунды. Каждое событие раунда (удар, критический удар, гибель) запишем вот так - тут и пригодятся наши константы:

```go
package combat

// BattleEvent - одно событие боя: в каком раунде, что произошло, кто кого,
// на сколько и сколько здоровья осталось у цели
type BattleEvent struct {
	Round             int
	Type              EventType
	Source            Side
	Target            Side
	Amount            int
	TargetHealthAfter int
}
```

Чтобы в отчете было видно, кто сражался, заведем класс участника боя: в нем все, что мы выводим о герое и его противнике:

```go
package combat

// Participant - участник боя в том виде, в котором он попадает в отчет
type Participant struct {
	Code      string
	Name      string
	MaxHealth int
}

func ParticipantOf(combatant Combatant) Participant {
	return Participant{Code: combatant.Code(), Name: combatant.Name(), MaxHealth: combatant.MaxHealth()}
}
```

Супер! Теперь можем составить общий отчет о бое, в нем будут два участника (герой и его противник), исход боя, а также информация о событиях, 
получится что-то вот такое:

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

А теперь наконец-то определим механику самого боя! В целом уже ясно, что идти она будет по раундам, 
в рамках которых герой и враг будут обмениваться ударами, давайте так и реализуем:

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
	dealt := defender.ReceiveHit(hit.Damage)
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

```go
package item

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
```

Дальше определим интерфейс предмета и общую часть всех предметов. В интерфейс заложим и контракт `Clone`:

```go
package item

import (
	"fmt"
	"slices"
)

// Item - прототип: любой предмет умеет создавать свою копию
type Item interface {
	// Clone возвращает копию предмета того же конкретного типа
	Clone() Item
	Code() string
	Name() string
	Price() int
	Enchantments() []*Enchantment
	// PossibleEnchantments - какие зачарования подходят этому предмету
	PossibleEnchantments() []*Enchantment
	Enchant(enchantment *Enchantment) error
	// Description - описание характеристик предмета, например "урон 12"
	Description() string
}

// baseItem - общая часть всех предметов, ее встраивают в себя конкретные типы предметов
type baseItem struct {
	// код шаблона, из которого получен предмет
	code         string
	name         string
	basePrice    int
	enchantments []*Enchantment
}

// cloneBase копирует общую часть предмета
func (b baseItem) cloneBase() baseItem {
	// присваивание структуры копирует все ее поля
	clone := b
	// но слайс - это ссылка на общий массив, поэтому для копии создаем свой слайс,
	// иначе копия и оригинал делили бы один и тот же объект списка зачарований
	clone.enchantments = slices.Clone(b.enchantments)
	return clone
}

func (b *baseItem) enchant(possible []*Enchantment, enchantment *Enchantment) error {
	if !slices.Contains(possible, enchantment) {
		return fmt.Errorf("зачарование '%s' не подходит для предмета '%s'", enchantment.Name(), b.name)
	}

	b.enchantments = append(b.enchantments, enchantment)
	return nil
}

func (b *baseItem) Code() string { return b.code }

func (b *baseItem) Name() string { return b.name }

func (b *baseItem) Price() int {
	price := b.basePrice
	for _, enchantment := range b.enchantments {
		price += enchantment.BonusPrice()
	}

	return price
}

func (b *baseItem) Enchantments() []*Enchantment {
	return slices.Clone(b.enchantments)
}
```

Обратите внимание на метод `cloneBase`. Присваивание структуры (`clone := b`) копирует все ее поля, но копия при этом
поверхностная: слайс - это ссылка на общий массив, и без `slices.Clone` копия и шаблон делили бы один и тот же список
зачарований. Зачарование, наложенное на копию, появилось бы и на шаблоне в каталоге, поэтому для копии мы создаем свой
слайс. А вот сами зачарования копировать не нужно - они неизменяемые, и копия может ссылаться на те же объекты: все их поля неэкспортируемые и задаются только
при создании.

Супер, общий контракт предметов понятен, давайте создадим контракт для оружия:

```go
package item

import "github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"

// Weapon - оружие: предмет, который умеет наносить урон
type Weapon interface {
	Item
	// RollDamage - как именно оружие наносит урон, решают конкретные типы
	RollDamage() combat.Hit
	Damage() int
}

// baseWeapon - общая часть всего оружия
type baseWeapon struct {
	baseItem
	damage int
}

func (w baseWeapon) cloneWeapon() baseWeapon {
	clone := w
	clone.baseItem = w.cloneBase()
	return clone
}

// Damage - урон с учетом зачарований
func (w *baseWeapon) Damage() int {
	total := w.damage
	for _, enchantment := range w.enchantments {
		total += enchantment.BonusDamage()
	}

	return total
}

func (w *baseWeapon) PossibleEnchantments() []*Enchantment {
	return ForWeapons
}

func (w *baseWeapon) Enchant(enchantment *Enchantment) error {
	return w.enchant(ForWeapons, enchantment)
}
```

Отлично, теперь наше первое оружие, меч. Его урон всегда одинаковый:

```go
package item

import (
	"fmt"

	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
)

// Sword - меч бьет стабильно
type Sword struct {
	baseWeapon
}

func NewSword(code, name string, basePrice, damage int) *Sword {
	return &Sword{baseWeapon{baseItem{code: code, name: name, basePrice: basePrice}, damage}}
}

func (s *Sword) RollDamage() combat.Hit {
	return combat.Hit{Damage: s.Damage(), Critical: false}
}

func (s *Sword) Clone() Item {
	return &Sword{baseWeapon: s.cloneWeapon()}
}

func (s *Sword) Description() string {
	return fmt.Sprintf("урон %d", s.Damage())
}
```

Лук же с некоторым шансом наносит критический, двойной урон:

```go
package item

import (
	"fmt"
	"math"

	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
)

const criticalMultiplier = 2

// Bow - лук с некоторым шансом наносит двойной урон
type Bow struct {
	baseWeapon
	criticalChance float64
}

func NewBow(code, name string, basePrice, damage int, criticalChance float64) *Bow {
	return &Bow{baseWeapon{baseItem{code: code, name: name, basePrice: basePrice}, damage}, criticalChance}
}

func (b *Bow) RollDamage() combat.Hit {
	if core.GetInstance().Chance(b.criticalChance) {
		return combat.Hit{Damage: b.Damage() * criticalMultiplier, Critical: true}
	}

	return combat.Hit{Damage: b.Damage(), Critical: false}
}

func (b *Bow) Clone() Item {
	// копируем всю структуру, а общую часть оружия - отдельно, вместе с ее слайсом зачарований
	clone := *b
	clone.baseWeapon = b.cloneWeapon()
	return &clone
}

func (b *Bow) Description() string {
	return fmt.Sprintf("урон %d, шанс крита %d%%", b.Damage(), int(math.Round(b.criticalChance*100)))
}
```

Урон магического посоха каждый раз немного разный:

```go
package item

import (
	"fmt"
	"math"

	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
)

// Staff - урон посоха непредсказуем: он может быть как больше, так и меньше базового
type Staff struct {
	baseWeapon
	scatter float64
}

func NewStaff(code, name string, basePrice, damage int, scatter float64) *Staff {
	return &Staff{baseWeapon{baseItem{code: code, name: name, basePrice: basePrice}, damage}, scatter}
}

func (s *Staff) RollDamage() combat.Hit {
	return combat.Hit{Damage: core.GetInstance().Roll(s.minDamage(), s.maxDamage()), Critical: false}
}

func (s *Staff) Clone() Item {
	clone := *s
	clone.baseWeapon = s.cloneWeapon()
	return &clone
}

func (s *Staff) Description() string {
	return fmt.Sprintf("урон %d-%d", s.minDamage(), s.maxDamage())
}

func (s *Staff) minDamage() int {
	return int(math.Round(float64(s.Damage()) * (1 - s.scatter)))
}

func (s *Staff) maxDamage() int {
	return int(math.Round(float64(s.Damage()) * (1 + s.scatter)))
}
```

Дальше давайте создадим класс для брони, он у нас будет общий для всех ее видов:

```go
package item

import (
	"fmt"
	"math"
)

// больше этой доли урона никакая броня не поглощает
const maxDefense = 0.75

// Armor - броня поглощает долю входящего урона
type Armor struct {
	baseItem
	defense float64
}

func NewArmor(code, name string, basePrice int, defense float64) *Armor {
	return &Armor{baseItem{code: code, name: name, basePrice: basePrice}, defense}
}

func (a *Armor) Clone() Item {
	clone := *a
	clone.baseItem = a.cloneBase()
	return &clone
}

// Defense - доля поглощаемого урона с учетом зачарований
func (a *Armor) Defense() float64 {
	total := a.defense
	for _, enchantment := range a.enchantments {
		total += enchantment.BonusDefense()
	}

	return math.Min(total, maxDefense)
}

func (a *Armor) PossibleEnchantments() []*Enchantment {
	return ForArmor
}

func (a *Armor) Enchant(enchantment *Enchantment) error {
	return a.enchant(ForArmor, enchantment)
}

func (a *Armor) Description() string {
	return fmt.Sprintf("защита %d%%", int(math.Round(a.Defense()*100)))
}
```

Каждый тип предмета реализует `Clone()` сам: копирует свою структуру целиком (`clone := *b`), а встроенную общую часть
копирует отдельно, вместе с ее слайсом зачарований. Возвращается интерфейс `Item`, поэтому тот, кто вызвал `Clone()`,
получает копию, не зная ее конкретного типа.

Теперь давайте напишем каталог: он хранит шаблоны всех предметов игры и выдает по ним конкретные экземпляры оружия и
брони:

```go
package item

import (
	"fmt"

	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
)

// ItemCatalog - каталог хранит шаблоны предметов, а наружу отдает только их копии
type ItemCatalog struct {
	// шаблоны хранятся в порядке добавления, предметов немного, поэтому ищем простым перебором
	prototypes []Item
}

func (c *ItemCatalog) Add(prototype Item) {
	c.prototypes = append(c.prototypes, prototype)
}

// Create - каталог не знает, какой конкретно тип у шаблона: копию создает сам шаблон
func (c *ItemCatalog) Create(code string) (Item, error) {
	for _, prototype := range c.prototypes {
		if prototype.Code() == code {
			return prototype.Clone(), nil
		}
	}

	return nil, &core.UnknownCodeError{What: "Предмет", Code: code}
}

func (c *ItemCatalog) CreateWeapon(code string) (Weapon, error) {
	item, err := c.Create(code)
	if err != nil {
		return nil, err
	}

	// проверка типа: является ли предмет оружием
	weapon, ok := item.(Weapon)
	if !ok {
		return nil, fmt.Errorf("предмет '%s' не является оружием", code)
	}

	return weapon, nil
}

func (c *ItemCatalog) CreateArmor(code string) (*Armor, error) {
	item, err := c.Create(code)
	if err != nil {
		return nil, err
	}

	armor, ok := item.(*Armor)
	if !ok {
		return nil, fmt.Errorf("предмет '%s' не является броней", code)
	}

	return armor, nil
}

func (c *ItemCatalog) Codes() []string {
	codes := make([]string, 0, len(c.prototypes))
	for _, prototype := range c.prototypes {
		codes = append(codes, prototype.Code())
	}

	return codes
}
```

В методе `Create` и работает прототип: каталог вызывает `Clone()` у шаблона и получает копию, не зная, меч это,
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

```go
package item

// коды шаблонов: по ним предметы ищутся в каталоге
// стартовое снаряжение
const TrainingSword = "training-sword"
const HuntingBow = "hunting-bow"
const ApprenticeStaff = "apprentice-staff"
const Chainmail = "chainmail"
const LeatherArmor = "leather-armor"
const LinenRobe = "linen-robe"

// товары торговца
const IronSword = "iron-sword"
const ElvenBow = "elven-bow"
const EmberStaff = "ember-staff"
const PlateArmor = "plate-armor"

// добыча
const GoblinCleaver = "goblin-cleaver"
const BoneBow = "bone-bow"
const NecromancerStaff = "necromancer-staff"
const KnightBlade = "knight-blade"
const CryptMail = "crypt-mail"
const DawnBlade = "dawn-blade"
const DragonscaleArmor = "dragonscale-armor"

// NewDefaultCatalog возвращает каталог со всеми предметами, которые есть в игре
func NewDefaultCatalog() *ItemCatalog {
	catalog := &ItemCatalog{}

	// стартовое снаряжение
	catalog.Add(NewSword(TrainingSword, "Тренировочный меч", 20, 12))
	catalog.Add(NewBow(HuntingBow, "Охотничий лук", 20, 10, 0.3))
	catalog.Add(NewStaff(ApprenticeStaff, "Посох ученика", 20, 14, 0.3))
	catalog.Add(NewArmor(Chainmail, "Кольчуга", 40, 0.3))
	catalog.Add(NewArmor(LeatherArmor, "Кожаная броня", 30, 0.2))
	catalog.Add(NewArmor(LinenRobe, "Льняная роба", 15, 0.1))

	// товары торговца
	catalog.Add(NewSword(IronSword, "Железный меч", 80, 17))
	catalog.Add(NewBow(ElvenBow, "Эльфийский лук", 90, 14, 0.4))
	catalog.Add(NewStaff(EmberStaff, "Тлеющий посох", 90, 20, 0.3))
	catalog.Add(NewArmor(PlateArmor, "Латы", 120, 0.4))

	// добыча
	catalog.Add(NewSword(GoblinCleaver, "Гоблинский тесак", 40, 15))
	catalog.Add(NewBow(BoneBow, "Костяной лук", 140, 17, 0.4))
	catalog.Add(NewStaff(NecromancerStaff, "Посох некроманта", 150, 25, 0.35))
	catalog.Add(NewSword(KnightBlade, "Клинок рыцаря", 150, 23))
	catalog.Add(NewArmor(CryptMail, "Кольчуга стража склепа", 160, 0.45))
	catalog.Add(NewSword(DawnBlade, "Клинок рассвета", 500, 35))
	catalog.Add(NewArmor(DragonscaleArmor, "Драконья чешуя", 500, 0.55))

	return catalog
}
```

## Стартовое снаряжение (Абстрактная фабрика / Abstract Factory)

По логике нашей игры мы хотим выдавать герою стартовое снаряжение в зависимости от его класса.
В целом это можно сделать простыми if'ами в коде основной программы, однако мы знаем, что в дальнейшем в игру будут
добавляться новые классы, поэтому здесь уместно будет применить шаблон *Абстрактная фабрика*.
Она позволяет создавать семейства связанных между собой объектов, не привязываясь к их конкретным классам. Семейство у
нас - это стартовое снаряжение класса.

В роли базовой фабрики выступит стартовый набор снаряжения:

```go
package kit

import "github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"

// StarterKitFactory - абстрактная фабрика: стартовый набор снаряжения
type StarterKitFactory interface {
	// CreateWeapon возвращает некое оружие
	CreateWeapon() (item.Weapon, error)
	// CreateArmor возвращает некую броню
	CreateArmor() (*item.Armor, error)
}
```

И для каждого из классов героя мы создадим свою реализацию этого набора: снаряжение каждого класса - это и есть
семейство объектов.

В наборе для воина будут лежать меч и кольчуга:

```go
package kit

import "github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"

// WarriorKitFactory - воин начинает с мечом и кольчугой
type WarriorKitFactory struct {
	catalog *item.ItemCatalog
}

func NewWarriorKitFactory(catalog *item.ItemCatalog) StarterKitFactory {
	return &WarriorKitFactory{catalog: catalog}
}

func (f *WarriorKitFactory) CreateWeapon() (item.Weapon, error) {
	return f.catalog.CreateWeapon(item.TrainingSword)
}

func (f *WarriorKitFactory) CreateArmor() (*item.Armor, error) {
	return f.catalog.CreateArmor(item.Chainmail)
}
```

В наборе для следопыта - лук и кожаная броня:

```go
package kit

import "github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"

// RangerKitFactory - следопыт начинает с луком и кожаной броней
type RangerKitFactory struct {
	catalog *item.ItemCatalog
}

func NewRangerKitFactory(catalog *item.ItemCatalog) StarterKitFactory {
	return &RangerKitFactory{catalog: catalog}
}

func (f *RangerKitFactory) CreateWeapon() (item.Weapon, error) {
	return f.catalog.CreateWeapon(item.HuntingBow)
}

func (f *RangerKitFactory) CreateArmor() (*item.Armor, error) {
	return f.catalog.CreateArmor(item.LeatherArmor)
}
```

Ну а для мага - посох и роба:

```go
package kit

import "github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"

// MageKitFactory - маг начинает с посохом и робой
type MageKitFactory struct {
	catalog *item.ItemCatalog
}

func NewMageKitFactory(catalog *item.ItemCatalog) StarterKitFactory {
	return &MageKitFactory{catalog: catalog}
}

func (f *MageKitFactory) CreateWeapon() (item.Weapon, error) {
	return f.catalog.CreateWeapon(item.ApprenticeStaff)
}

func (f *MageKitFactory) CreateArmor() (*item.Armor, error) {
	return f.catalog.CreateArmor(item.LinenRobe)
}
```

В Go нет перечислений, которые хранят данные, поэтому классы героя - это заранее созданные значения типа `HeroClass` в
переменных пакета. Каждый класс будет сам знать свою фабрику стартового снаряжения:

```go
package hero

import (
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

```go
package hero

import (
	"errors"
	"fmt"
	"math"
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
	name       string
	heroClass  *HeroClass
	level      int
	experience int
	gold       int
	health     int
	weapon     item.Weapon
	armor      *item.Armor
	inventory  []item.Item
}

// Builder - строитель, чтобы удобно и безопасно создавать героев
type Builder struct {
	name      string
	heroClass *HeroClass
	weapon    item.Weapon
	armor     *item.Armor
	level     int
	gold      int
	// первая ошибка, случившаяся при заполнении полей; ее вернет Build
	err error
}

// NewBuilder создает строителя: необязательные поля сразу получают значения по умолчанию
func NewBuilder() *Builder {
	return &Builder{level: 1, gold: startingGold}
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

	hero := &Hero{
		name:      b.name,
		heroClass: b.heroClass,
		level:     b.level,
		gold:      b.gold,
		weapon:    b.weapon,
		armor:     b.armor,
	}
	hero.health = hero.MaxHealth()
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

func (h *Hero) ReceiveHit(damage int) int {
	reducedDamage := int(math.Round(float64(damage) * (1 - h.armor.Defense())))
	// хотя бы единицу урона броня пропускает всегда
	dealt := max(1, reducedDamage)
	h.health = max(0, h.health-dealt)
	return dealt
}

func (h *Hero) IsAlive() bool {
	return h.health > 0
}

// поведение героя между экспедициями

// RestoreHealth - перед каждой экспедицией герой успевает отдохнуть
func (h *Hero) RestoreHealth() {
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
```

Обратите внимание еще на две вещи. В Go у полей нет значений по умолчанию, поэтому их задает функция
`NewBuilder`. Исключений тоже нет, поэтому строитель запоминает первую ошибку (например, если стартовый набор не смог
создать предмет), а `Build` возвращает ее вместе с героем: `h, err := ...Build()`.

Что в реализации строителя важно:

1. Поля героя неэкспортируемые, поэтому вне пакета `hero` заполнить героя можно только через строителя, а значит, в
   обход проверок героя не создать.
2. Каждый метод строителя возвращает сам строитель, поэтому вызовы можно выстраивать цепочкой.
3. Все проверки собраны в методе `Build`: если обязательное поле не заполнено или заполнено неправильно, герой
   просто не создается.
4. Необязательные поля получают значения по умолчанию в самом строителе.

Без строителя понадобился бы конструктор с шестью параметрами, в котором легко перепутать, например,
уровень и золото (оба - целые числа). А по цепочке вызовов строителя сразу видно, какое значение какому полю
присваивается.

Кроме того, для необязательных параметров пришлось бы заводить несколько функций-конструкторов на разные наборы
параметров: только все обязательные поля, обязательные + 1 необязательное, обязательные + другое необязательное, все поля.
В языках с перегрузкой конструкторов это называют телескопическим конструктором, и строитель избавляет от него.

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

```go
package enemy

import "github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"

// Enemy - враг: боец, за победу над которым дают награду
type Enemy interface {
	combat.Combatant
	ExperienceReward() int
	RollGoldReward() int
}
```

Общая логика врагов вынесена в структуру `baseEnemy`, которую конкретные враги встраивают в себя:

```go
package enemy

import (
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

func (e *baseEnemy) ReceiveHit(damage int) int {
	reducedDamage := int(math.Round(float64(damage) * (1 - e.resistance)))
	dealt := max(1, reducedDamage)
	e.health = max(0, e.health-dealt)
	return dealt
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

Определим злого гоблина, самого слабого врага, живущего в лесу:

```go
package enemy

// Goblin - самый простой враг, без особенностей
type Goblin struct {
	baseEnemy
}

func NewGoblin() *Goblin {
	return &Goblin{newBaseEnemy("goblin", "Гоблин", 50, 6, 14, 0, 40, 30)}
}
```

Теперь скелет, он уже чуть страшнее гоблина:

```go
package enemy

// Skeleton - кости плохо поддаются ударам, поэтому часть урона проходит мимо
type Skeleton struct {
	baseEnemy
}

func NewSkeleton() *Skeleton {
	return &Skeleton{newBaseEnemy("skeleton", "Скелет", 110, 10, 24, 0.3, 90, 70)}
}
```

И ужасный дракон: он наносит страшный урон огнем, а крепкая чешуя гасит часть урона:

```go
package enemy

import (
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
	return &Dragon{newBaseEnemy("dragon", "Дракон", 260, 16, 22, 0.25, 500, 400)}
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

Обратите внимание: дракон заменяет метод `RollDamage` встроенной структуры своим, и бой вызывает именно его, потому что
работает с драконом через интерфейс `Enemy`. А вот обратное не работает: метод встроенной структуры не может вызвать
"переопределенный" метод внешней, ведь в Go нет наследования. Поэтому сопротивление урону сделано полем с данными, а не
методом, который переопределяют наследники.

По итогам экспедиции герой получает награды: опыт, золото, добычу и новые уровни. Запишем их так:

```go
package world

import "github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"

// Rewards - итоги экспедиции для героя;
// Gold отрицательный, если герой проиграл и потерял золото, Loot равен nil, если добычи нет
type Rewards struct {
	Experience   int
	Gold         int
	Loot         item.Item
	LevelsGained int
}
```

Ну а общий итог экспедиции фиксируем вот так:

```go
package world

import "github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"

// ExpeditionReport - отчет об экспедиции: с каким сидом, куда, как прошел бой и что получил герой
type ExpeditionReport struct {
	Seed         uint64
	LocationCode string
	Battle       combat.BattleReport
	Rewards      Rewards
}
```

Интерфейс локации и общий сценарий экспедиции:

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

	h.RestoreHealth()
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

Положить общий сценарий в базовую структуру, как делают в языках с наследованием, в Go не получится: метод встроенной
структуры не может вызвать метод внешней. Поэтому сценарий - это функция, которая работает с любой локацией через
интерфейс `Location`, а фабричный метод объявлен в этом интерфейсе.

Также обратите внимание, что добыча здесь - копия шаблона из каталога, и зачарование накладывается именно на копию.

Отлично, теперь создаем конкретные локации:

Мистический лес, кишащий злыми, но слабыми гоблинами:

```go
package world

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/enemy"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
)

const ForestCode = "forest"

type Forest struct {
	baseLocation
}

func NewForest(catalog *item.ItemCatalog) *Forest {
	return &Forest{baseLocation{ForestCode, "Мистический лес", 1, catalog}}
}

func (f *Forest) CreateEnemy() enemy.Enemy {
	return enemy.NewGoblin()
}

func (f *Forest) LootTable() []string {
	return []string{item.GoblinCleaver, item.HuntingBow, item.LeatherArmor}
}
```

Заброшенный склеп, в котором бродят ожившие скелеты:

```go
package world

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/enemy"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
)

const CryptCode = "crypt"

type Crypt struct {
	baseLocation
}

func NewCrypt(catalog *item.ItemCatalog) *Crypt {
	return &Crypt{baseLocation{CryptCode, "Заброшенный склеп", 3, catalog}}
}

func (c *Crypt) CreateEnemy() enemy.Enemy {
	return enemy.NewSkeleton()
}

func (c *Crypt) LootTable() []string {
	return []string{item.KnightBlade, item.BoneBow, item.NecromancerStaff, item.CryptMail}
}
```

И логово дракона, прибежище жуткого чешуйчатого чудища:

```go
package world

import (
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/enemy"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
)

const DragonLairCode = "dragon-lair"

type DragonLair struct {
	baseLocation
}

func NewDragonLair(catalog *item.ItemCatalog) *DragonLair {
	return &DragonLair{baseLocation{DragonLairCode, "Логово дракона", 5, catalog}}
}

func (d *DragonLair) CreateEnemy() enemy.Enemy {
	return enemy.NewDragon()
}

func (d *DragonLair) LootTable() []string {
	return []string{item.DawnBlade, item.DragonscaleArmor}
}
```

Супер! Теперь если в игре появится еще одна локация, нам нужно будет лишь реализовать для нее фабричный метод в новом
классе, в остальном код экспедиций меняться не будет.

Сам фабричный метод объявлен в интерфейсе локации и возвращает абстракцию (`Enemy`), а не конкретного врага. Вызывает
его общий сценарий экспедиции `ExploreWithSeed`, поэтому конкретные локации не переписывают сценарий, а лишь определяют,
кого герой в нем встретит. В итоге сценарий написан один раз и не зависит ни от гоблинов, ни от скелетов, ни от
драконов.

Минус в том, что на каждый вид продукта нужен свой тип создателя: десять видов врагов - десять типов локаций, даже если
ничем другим локации не отличаются. Применять фабричный метод стоит, когда есть общий алгоритм, а объект, который в
нем создается, должен зависеть от конкретного типа.

Кстати, в языках с наследованием такой общий сценарий кладут в базовый класс - это еще один паттерн, *Шаблонный метод*:
базовый класс задает шаги алгоритма, а отдельные шаги переопределяются в наследниках. Фабричный метод часто встречается
именно как один из таких шагов. В Go роль шаблонного метода играет функция, которая работает с интерфейсом.

Давайте сравним фабричный метод с абстрактной фабрикой из прошлого раздела. Абстрактная фабрика - это отдельный объект,
который передается тому, кому он нужен (композиция - мы кратко касались ее на прошлой лекции). 
Ее методы вызывает внешний код (в нашем случае строитель героя), и создает она целое семейство объектов. 

Фабричный метод - это метод самой локации, каждая локация реализует его по-своему, и создает он один продукт. 
Но главное отличие в том, что фабричным методом пользуется общий сценарий самой локации (`ExploreWithSeed`). 
Снаружи этот фабричный метод никто не вызывает.

Чтобы всем этим пользоваться, осталось завести реестр локаций:

```go
package world

import (
	"slices"

	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
)

// LocationRegistry - все локации игры
type LocationRegistry struct {
	locations []Location
}

func NewLocationRegistry(locations ...Location) *LocationRegistry {
	return &LocationRegistry{locations: locations}
}

func (r *LocationRegistry) Get(code string) (Location, error) {
	for _, location := range r.locations {
		if location.Code() == code {
			return location, nil
		}
	}

	return nil, &core.UnknownCodeError{What: "Локация", Code: code}
}

func (r *LocationRegistry) All() []Location {
	return slices.Clone(r.locations)
}
```

## Лагерь

Между экспедициями герой отдыхает в лагере, где есть торговец (а куда же без него). 
Торговец продает предметы из каталога и скупает ненужное за половину цены:

```go
package camp

import (
	"slices"

	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/hero"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
)

const sellPricePercent = 50

// Merchant - торговец в лагере
type Merchant struct {
	catalog    *item.ItemCatalog
	assortment []string
}

func NewMerchant(catalog *item.ItemCatalog, assortment ...string) *Merchant {
	return &Merchant{catalog: catalog, assortment: assortment}
}

// Showcase возвращает образцы товаров, чтобы показать их покупателю
func (m *Merchant) Showcase() ([]item.Item, error) {
	showcase := make([]item.Item, 0, len(m.assortment))
	for _, code := range m.assortment {
		it, err := m.catalog.Create(code)
		if err != nil {
			return nil, err
		}
		showcase = append(showcase, it)
	}

	return showcase, nil
}

func (m *Merchant) Buy(h *hero.Hero, code string) (item.Item, error) {
	if !slices.Contains(m.assortment, code) {
		return nil, &core.UnknownCodeError{What: "Товар", Code: code}
	}

	// покупатель получает собственную копию предмета из каталога
	it, err := m.catalog.Create(code)
	if err != nil {
		return nil, err
	}
	if err := h.SpendGold(it.Price()); err != nil {
		return nil, err
	}

	h.AddItem(it)
	return it, nil
}

func (m *Merchant) Sell(h *hero.Hero, it item.Item) (int, error) {
	price := m.SellPrice(it)
	if err := h.RemoveItem(it); err != nil {
		return 0, err
	}

	h.AddGold(price)
	return price, nil
}

func (m *Merchant) SellPrice(it item.Item) int {
	return it.Price() * sellPricePercent / 100
}
```

## Консоль

Вся игровая логика наконец-то готова, осталось научить программу общаться с игроком. Весь вывод соберем в одном классе - он
превращает героя, предметы и отчеты об экспедиции в текст для консоли. 

Отдельный класс нужен потому, что дальше с игрой можно будет работать не только через консоль (спойлер-спойлер), и
привязывать к ней игровую логику не хочется.

Вот он:

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
```

Осталось написать игровой цикл - он будет в функции `main` (файл `main.go` в корне папки лабы):

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"strconv"
	"strings"

	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/camp"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/combat"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/console"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/hero"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/world"
)

var (
	input = bufio.NewScanner(os.Stdin)
	view  = console.ConsoleView{}
)

func main() {
	catalog := item.NewDefaultCatalog()
	locations := world.NewLocationRegistry(
		world.NewForest(catalog),
		world.NewCrypt(catalog),
		world.NewDragonLair(catalog),
	)
	merchant := camp.NewMerchant(catalog,
		item.IronSword, item.ElvenBow, item.EmberStaff, item.Chainmail, item.PlateArmor)

	h := createHero(catalog)
	fmt.Printf("%s разбивает лагерь на опушке мистического леса.\n", h.Name())

	for {
		fmt.Println()
		view.PrintHero(h)
		fmt.Println("1 - отправиться в экспедицию, 2 - снаряжение, 3 - торговец, 0 - выйти из игры")
		switch readLine() {
		case "1":
			goOnExpedition(h, locations)
		case "2":
			manageEquipment(h)
		case "3":
			visitMerchant(h, merchant)
		case "0":
			return
		default:
			fmt.Println("Такого варианта нет")
		}
	}
}

func createHero(catalog *item.ItemCatalog) *hero.Hero {
	fmt.Println("Создайте своего героя!")
	for {
		fmt.Println("Выберите класс:")
		heroClasses := hero.HeroClasses()
		for i, heroClass := range heroClasses {
			kit := heroClass.StarterKit(catalog)
			weapon, _ := kit.CreateWeapon()
			armor, _ := kit.CreateArmor()
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

		h, err := hero.NewBuilder().
			SetName(name).
			SetHeroClass(heroClass).
			SetStarterKit(heroClass.StarterKit(catalog)).
			Build()
		if err != nil {
			fmt.Println("Не получилось создать героя: " + err.Error())
			continue
		}

		return h
	}
}

func goOnExpedition(h *hero.Hero, locations *world.LocationRegistry) {
	all := locations.All()
	fmt.Println("Куда отправимся?")
	for i, location := range all {
		fmt.Printf("%d - %s (рекомендуемый уровень %d)\n", i+1, location.DisplayName(), location.RecommendedLevel())
	}
	index := readChoice(len(all))
	if index < 0 {
		return
	}

	location := all[index]
	report, err := world.Explore(location, h)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println()
	view.PrintReport(report, location.DisplayName())

	if report.Battle.Outcome == combat.Victory && location.Code() == world.DragonLairCode {
		fmt.Printf("\nДракон повержен! Барды еще долго будут петь о подвиге героя по имени %s.\n", h.Name())
	}
}

func manageEquipment(h *hero.Hero) {
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
	if err := h.Equip(it); err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println("Экипировано: " + view.Describe(it))
}

func visitMerchant(h *hero.Hero, merchant *camp.Merchant) {
	fmt.Printf("Торговец: \"Чего желаешь?\" (золото: %d)\n", h.Gold())
	fmt.Println("1 - купить, 2 - продать, 0 - назад")
	switch readLine() {
	case "1":
		buy(h, merchant)
	case "2":
		sell(h, merchant)
	}
}

func buy(h *hero.Hero, merchant *camp.Merchant) {
	showcase, err := merchant.Showcase()
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

	it, err := merchant.Buy(h, showcase[index].Code())
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println("Куплено: " + view.Describe(it))
}

func sell(h *hero.Hero, merchant *camp.Merchant) {
	inventory := h.Inventory()
	if len(inventory) == 0 {
		fmt.Println("Продавать нечего (экипированные предметы продать нельзя)")
		return
	}

	for i, it := range inventory {
		fmt.Printf("%d - %s: %d золота\n", i+1, view.Describe(it), merchant.SellPrice(it))
	}
	index := readChoice(len(inventory))
	if index < 0 {
		return
	}

	it := inventory[index]
	price, err := merchant.Sell(h, it)
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

Супер! Попробуйте запустить игру и победить дракона :)

В целом по коду программы видно, что мы почти не привязываемся к конкретным реализациям классов, а везде используем
абстракции (`Location`, а не конкретные локации для экспедиций, `StarterKitFactory`, а не конкретные наборы для
получения снаряжения, `Item`, а не конкретные предметы при копировании из каталога).

Таким образом в игру легко добавлять новые локации и врагов, классы героя со снаряжением и предметы.

> **_ДОП. ЗАДАНИЕ:_** Если хотите на практике в этом убедиться - попробуйте создать новую локацию и новый класс
> героя, выдав ему новое оружие и броню (не забудьте добавить их в каталог, а для новой локации - добычу!). Много ли мест в старом коде пришлось поменять?

## Демонстрация паттернов

Чтобы показать все паттерны одним запуском, напишем отдельную программу `cmd/demo`:

```go
package main

import (
	"fmt"

	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/console"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/core"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/hero"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/item"
	"github.com/tidbid-kt/tech-prog-labs-2026-go/labs/lab3/world"
)

func main() {
	catalog := item.NewDefaultCatalog()
	view := console.ConsoleView{}

	fmt.Println("===== Одиночка =====")
	first := core.GetInstance()
	second := core.GetInstance()
	fmt.Printf("core.GetInstance() возвращает один и тот же объект: %t\n", first == second)

	fmt.Println()
	fmt.Println("===== Прототип =====")
	plainSword := must(catalog.Create(item.IronSword))
	fierySword := must(catalog.Create(item.IronSword))
	if err := fierySword.Enchant(item.Fiery); err != nil {
		panic(err)
	}
	fmt.Println("Первая копия:        " + view.Describe(plainSword))
	fmt.Println("Вторая копия:        " + view.Describe(fierySword))
	fmt.Println("Новая копия шаблона: " + view.Describe(must(catalog.Create(item.IronSword))))
	fmt.Printf("Копии - разные объекты: %t\n", plainSword != fierySword)

	fmt.Println()
	fmt.Println("===== Абстрактная фабрика =====")
	for _, heroClass := range hero.HeroClasses() {
		kit := heroClass.StarterKit(catalog)
		weapon := must(kit.CreateWeapon())
		armor := must(kit.CreateArmor())
		fmt.Printf("%s: %s, %s\n", heroClass.DisplayName(), view.Describe(weapon), view.Describe(armor))
	}

	fmt.Println()
	fmt.Println("===== Строитель =====")
	h := must(hero.NewBuilder().
		SetName("Артур").
		SetHeroClass(hero.Warrior).
		SetStarterKit(hero.Warrior.StarterKit(catalog)).
		SetLevel(3).
		Build())
	view.PrintHero(h)
	_, err := hero.NewBuilder().
		SetHeroClass(hero.Mage).
		SetStarterKit(hero.Mage.StarterKit(catalog)).
		Build()
	fmt.Println("Герой без имени не создается: " + err.Error())
	_, err = hero.NewBuilder().SetName("Мерлин").Build()
	fmt.Println("Герой без класса не создается: " + err.Error())

	fmt.Println()
	fmt.Println("===== Фабричный метод =====")
	forest := world.NewForest(catalog)
	crypt := world.NewCrypt(catalog)
	view.PrintReport(must(world.ExploreWithSeed(forest, h, 7)), forest.DisplayName())
	fmt.Println()
	view.PrintReport(must(world.ExploreWithSeed(crypt, h, 7)), crypt.DisplayName())

}

// must возвращает значение, а если случилась ошибка, останавливает демонстрацию:
// в этой программе все данные заданы заранее, и ошибка означает ошибку в коде
func must[T any](value T, err error) T {
	if err != nil {
		panic(err)
	}

	return value
}
```

Вывод программы:

```
===== Одиночка =====
core.GetInstance() возвращает один и тот же объект: true

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
Герой без имени не создается: у героя должно быть имя
Герой без класса не создается: не выбран класс героя

===== Фабричный метод =====
=== Мистический лес (сид 7) ===
Артур встречает врага: Гоблин (50 здоровья)
[раунд 1] Артур -> Гоблин: 18 урона, осталось 32/50
[раунд 1] Гоблин -> Артур: 10 урона, осталось 140/150
[раунд 2] Артур -> Гоблин: 18 урона, осталось 14/50
[раунд 2] Гоблин -> Артур: 4 урона, осталось 136/150
[раунд 3] Артур -> Гоблин: 18 урона, осталось 0/50
Гоблин повержен!
Победа! +40 опыта, +26 золота

=== Заброшенный склеп (сид 7) ===
Артур встречает врага: Скелет (110 здоровья)
[раунд 1] Артур -> Скелет: 13 урона, осталось 97/110
[раунд 1] Скелет -> Артур: 17 урона, осталось 133/150
[раунд 2] Артур -> Скелет: 13 урона, осталось 84/110
[раунд 2] Скелет -> Артур: 7 урона, осталось 126/150
[раунд 3] Артур -> Скелет: 13 урона, осталось 71/110
[раунд 3] Скелет -> Артур: 15 урона, осталось 111/150
[раунд 4] Артур -> Скелет: 13 урона, осталось 58/110
[раунд 4] Скелет -> Артур: 8 урона, осталось 103/150
[раунд 5] Артур -> Скелет: 13 урона, осталось 45/110
[раунд 5] Скелет -> Артур: 9 урона, осталось 94/150
[раунд 6] Артур -> Скелет: 13 урона, осталось 32/110
[раунд 6] Скелет -> Артур: 10 урона, осталось 84/150
[раунд 7] Артур -> Скелет: 13 урона, осталось 19/110
[раунд 7] Скелет -> Артур: 15 урона, осталось 69/150
[раунд 8] Артур -> Скелет: 13 урона, осталось 6/110
[раунд 8] Скелет -> Артур: 8 урона, осталось 61/150
[раунд 9] Артур -> Скелет: 13 урона, осталось 0/110
Скелет повержен!
Победа! +90 опыта, +50 золота
Добыча: Костяной лук (урон 17, шанс крита 40%)
```

Не забудьте закоммитить и запушить изменения!

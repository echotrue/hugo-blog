---
title: "Turn-Based Game With Go【译】"
date: 2021-09-06T16:32:53+08:00
draft: false
---

>Today we will talk about how to write a simple turn-based game engine with Go. While writing our character’s abilities and fights with each other, we will use Interfaces,channels, and concurrency with Golang

今天我们将讨论下如何使用Golang编写一个简单的基于回合制的游戏引擎.在实现角色的能力和与其他角色战斗的功能时，我们会使用到Golang的`Interface`,`channels`,`concurrency`等特性。

>If you don’t have any idea about the turn-based game, it is about the different types of characters with different strengths, and different abilities fight and kill each other one by one. The winner is the survived last person.

如果你对回合制游戏一无所知，这里可以简单概括下。回合制游戏就是拥有不同力量，不同能力的各类角色之间互相战斗并逐个杀死对方. 获胜者是最后的幸存者。

### Turn Game Rules
>1. Fights mean every Warrier hit only the one Warrier in Turn. Every attacker loses stamina or mana after hit to an opponent.

战斗意味着每个战士每个回合只能攻击一个战士。每一个攻击者在击中对手后会失去体力值或者魔法值.
>2. Every Hit damage is changed by Warrier’s Level and his Weapon, Magic, or Animal kind.

每一次命中的伤害会因战士的等级，武器，法力，或者宠物类型而不同
>3. When a warrior is damaged by someone else, his or her blood will decreases. If the blood value is zero or less, he will die.

当一个战士被其他人伤害时，他的血量会减少，如果血量值降到0或者更少，他将会死亡.
>4. If a warrior’s stamina or mana less than zero, he can’t hit or make magic until the next Turn. Next Turn, his mana or stamina will be full.

如果一个战士的体力或者魔法值少于0，在下一回合之前他将不能施放技能。下一回合他的体力或者魔法值将会充满。


>Firstly let’s create base traits struct of these types of characters.

首先,创建不同角色的基础属性结构体

>heroCharacter/heroCharacter.go: All the characters have Name, Level (experience),Attacks(Damage List) and Blood(Energy)

所有的角色都有角色名称，等级（经验），攻击力（伤害列表）和血量（能量）

```Go
package heroCharacter

type Hero struct {
	Name string
	Level int
	Attacks map[string]int
	Blood int
}
```

>Let’s Create all Characters Interface. We will list all character’s common Actions in this interface.

创建角色`interface`,在这个接口中我们将定义所有角色通用的操作行为的方法。

>fightHero/fightHero.go: Every hero Hits the other characters, takes Damage after attacked by someone else. We can get info about him or her healthy, stamina or mana.And finally, we can check he or she died or not. These are the common functions of all Characters.

每个英雄角色击中其他角色，或者收到其他人的攻击。我们会获取他的健康，魔法或者体力信息.然后,检查他是否死亡.这是所有角色的通用方法。

```Go
package fightHero

type FightHero interface {

	Hit() int

	TakeDamage(damage int)

	GetInfo() (string, int)

	IsDeath() bool

}
```

### Wizard
> Let’s create first character Wizard.

创建第一个角色男巫

wizard/wizard.go:

#### properties of Wizard
```Go
type(
	Wizard struct{
		hero.Hero
		Manas map[string]int
		Mana int
		Magic string
	}
)
```

>Hero: Common character’s properties.

Hero:角色通用属性
>Manas: This is a collection of “string” magic names with their “int” mana costs.Example: “Manas: map[string]int{“FireBall”: 5, “Thunder”: 10, “Ghost Attack”:30}}”

Manas：魔法值消耗列表.魔法技能名称作为key,魔法技能消耗的魔法值作为value.示例中火球技能消耗5点魔法值,闪电技能消耗10点,幽灵攻击消耗30点
>Mana: This is integer Wizard’s mana value. Wizard needs mana, for making Magic.Without mana Wizard is nothing :)

Mana:魔法值,男巫的魔法值,男巫需要魔法值来施放魔法技能.没有魔法值,男巫啥也不是.
>Magic: This is Wizard’s weapon. He or she can hit an enemy with the Magic.
Example: “map[string]int{“FireBall”: 25, “Thunder”: 18, “Ghost Attack”: 30}”

Magic: 这是男巫的武器,他可以使用魔法技能攻击敌人.

#### Action of Wizard
##### FightHero interface Methods:

`Hit()`计算魔法技能伤害值,只有法力值足够的前提下才能施放魔法技能,魔法技能的伤害值和等级有关系.
```Go
func (w Wizard) Hit() int {  
	if w.Mana >= w.Manas[w.Magic] {  
		levelEffect := int(math.Ceil(float64(w.Level) * 0.1))  
		return w.Attacks[w.Magic] + levelEffect  
	}
	return 0  
}
```

`TakeDamage` 如果被其他角色攻击,血量将会减少,当血量为0时,男巫就会死亡.此处使用了指针类型`*Wizard`,因为当男巫受到伤害时必须及时减少男巫的血量
```Go
func (w *Wizard) TakeDamage(damage int) {  
	w.Blood = w.Blood - damage  
}
```

`GetInfo()` 获取男巫的名称和血量

`IsDeath():` 检测男巫是否死亡

##### Wizard Own Methods:
`GetMana():` 获取男巫当前的法力值

`SpendMana()` 消耗男巫的法力值

`CreateWizard()` 创建男巫对象

### Fighter (参考男巫)
### Druid (参考男巫)

### Fight (战斗)
我们已经定义了所有的英雄角色,现在开始让他们之间互相战斗,他们中只有一个能获胜幸存下来.
定义全局变量和随机函数:
```Go
var fighterList map[string]IHero.FightHero  
var fighterNumberList map[int]IHero.FightHero  
  
func GetRandomID(limit int) int {  
	rand.Seed(time.Now().UnixNano())  
	rndVictim := rand.Intn(limit) + 1  
	return rndVictim  
}  
func GetRandomBetweenID(minLimit int, maxlimit int) int {  
	rand.Seed(time.Now().UnixNano())  
	rndVictim := rand.Intn(maxlimit-minLimit) + minLimit + 1  
	return rndVictim  
}  
  
var isFighterDead bool = false  
var isWizardDead bool = false  
var isDruidDead bool = false  
  
var TotalLive int = 3

func MapRandomKeyGet(mapI interface{}) interface{} {  
	keys := reflect.ValueOf(mapI).MapKeys()  
	return keys[rand.Intn(len(keys))].Interface()  
}

```

- `fighterList`以英雄的名称保存英雄信息
- `fighterNumberList`根据英雄ID存储英雄名称
- `GetRandomID` 获取随机数
- `GetRandomBetweenID` 给定范围内获取随机数
- `isFighterDead, isWizardDead, isDruidDead` 每个英雄的生死状态
- `TotalLive` 几条生命
- `MapRandomKeyGet` 用来随机获取`Wizzard`的魔法技能,`Fighter`的武器,`Druid`的宠物
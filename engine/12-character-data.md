# 第十二章：角色数据系统

## RPG 角色的属性

```javascript
function CharacterData() {
    // 基础属性
    this.name = '勇者';
    this.level = 1;
    this.exp = 0;

    // 战斗属性
    this.hp = 100;
    this.maxHp = 100;
    this.mp = 30;
    this.maxMp = 30;

    this.attack = 10;    // 攻击力
    this.defense = 5;    // 防御力
    this.speed = 10;     // 速度

    // 装备栏
    this.weapon = null;
    this.armor = null;
    this.accessory = null;
}

// 获得经验
CharacterData.prototype.gainExp = function(amount) {
    this.exp += amount;

    // 检查升级
    var expNeeded = this.level * 100;
    while (this.exp >= expNeeded) {
        this.exp -= expNeeded;
        this.levelUp();
        expNeeded = this.level * 100;
    }
};

// 升级
CharacterData.prototype.levelUp = function() {
    this.level++;

    // 属性提升
    this.maxHp += 10;
    this.maxMp += 5;
    this.attack += 2;
    this.defense += 1;

    // 回满 HP/MP
    this.hp = this.maxHp;
    this.mp = this.maxMp;

    console.log(this.name + ' 升到了 ' + this.level + ' 级！');
};

// 计算总攻击力（基础 + 装备）
CharacterData.prototype.getTotalAttack = function() {
    var total = this.attack;
    if (this.weapon) {
        total += this.weapon.attack;
    }
    return total;
};

// 受到伤害
CharacterData.prototype.takeDamage = function(damage) {
    var actualDamage = Math.max(1, damage - this.defense);
    this.hp = Math.max(0, this.hp - actualDamage);
    return actualDamage;
};

// 是否死亡
CharacterData.prototype.isDead = function() {
    return this.hp <= 0;
};
```

---

## 经验值系统

```
等级    所需经验    累计经验
1       100         100
2       200         300
3       300         600
4       400         1000
...

公式：下一级所需经验 = 当前等级 × 100
```

---

## 本章小结

1. **属性系统**：HP、MP、攻击、防御
2. **升级系统**：经验值 → 升级 → 属性提升
3. **装备加成**：武器增加攻击力

## 下一章

[第十三章：物品与背包](./13-inventory.md) - 实现物品系统

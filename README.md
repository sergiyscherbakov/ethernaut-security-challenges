# Ethernaut Security Challenges - Домашнє завдання 11

## Розробник
**Сергій Щербаков**
- Email: sergiyscherbakov@ukr.net
- Telegram: @s_help_2010

## Опис проекту

Цей репозиторій містить рішення двох завдань з платформи [Ethernaut](https://ethernaut.openzeppelin.com/) - навчальної платформи для вивчення вразливостей смарт-контрактів. Проект демонструє типові атаки на смарт-контракти та методи їх експлуатації в освітніх цілях.

## Завдання та вразливості

### 1. Token Challenge - Integer Underflow
**Рівень складності:** Середній
**Вразливість:** Integer Underflow (Arithmetic Underflow)

#### Опис вразливості
Контракт Token використовує Solidity 0.6.0 без бібліотеки SafeMath, що робить його вразливим до integer underflow атаки. У функції `transfer()` перевірка `require(balances[msg.sender] - _value >= 0)` завжди повертає `true` для типу `uint`, оскільки беззнакові цілі числа не можуть бути від'ємними.

#### Контракти
- **Token.sol** - вразливий контракт токену
- **TokenAttacker.sol** - контракт-атакувач для експлуатації вразливості

#### Механізм атаки
1. Початковий баланс атакувача: 20 токенів
2. Виклик `transfer()` з величезним значенням (2^256 - 1)
3. Відбувається underflow: `20 - (2^256 - 1) = 21` (через переповнення)
4. Атакувач отримує майже необмежену кількість токенів

#### Захист
- Використовувати Solidity >= 0.8.0 (вбудована перевірка overflow/underflow)
- Використовувати бібліотеку SafeMath для версій < 0.8.0
- Додати правильну валідацію: `require(balances[msg.sender] >= _value)`

### 2. Delegation Challenge - Delegatecall Vulnerability
**Рівень складності:** Складний
**Вразливість:** Небезпечне використання delegatecall

#### Опис вразливості
Контракт Delegation використовує `delegatecall` у fallback функції без належного контролю. При виклику delegatecall код виконується в контексті викликаючого контракту, що дозволяє змінювати його змінні стану.

#### Контракти
- **Delegate.sol** - контракт з функцією pwn()
- **Delegation.sol** - вразливий контракт з delegatecall
- **DelegateAttacker.sol** - контракт-атакувач

#### Механізм атаки
1. Формування сигнатури функції: `bytes4(keccak256("pwn()"))`
2. Виклик fallback функції Delegation з підготовленими даними
3. Fallback робить delegatecall до Delegate.pwn()
4. Функція pwn() виконується в контексті Delegation
5. Змінна `owner` в Delegation змінюється на адресу атакувача

#### Метод через консоль браузера
```javascript
// Отримати поточного власника
await contract.owner()

// Виконати атаку
await contract.sendTransaction({
    from: player,
    data: web3.eth.abi.encodeFunctionSignature('pwn()')
})

// Перевірити нового власника
await contract.owner()
```

#### Захист
- Обмежити використання delegatecall
- Додати перевірки прав доступу
- Використовувати allowlist для викликів через delegatecall
- Ретельно контролювати msg.data у fallback функції

## Структура проекту

```
ethernaut-security-challenges/
├── Token.sol                 # Вразливий контракт токену (Solidity 0.6.0)
├── TokenAttacker.sol         # Атакувач для Token
├── Delegate.sol              # Базовий контракт з pwn()
├── Delegation.sol            # Вразливий контракт з delegatecall
├── DelegateAttacker.sol      # Атакувач для Delegation
├── 5-Token.jpg              # Скріншот успішної атаки на Token
├── 6 Delegation.jpg         # Скріншот успішної атаки на Delegation
└── README.md                # Цей файл
```

## Інструкція з компіляції та тестування

### Передумови
- [Remix IDE](https://remix.ethereum.org/)
- Браузер з Web3 (MetaMask)
- Тестова мережа Ethereum (Sepolia, Goerli)
- Акаунт на [Ethernaut](https://ethernaut.openzeppelin.com/)

### Тест 1: Token Challenge

#### Крок 1: Підготовка в Remix
1. Відкрити Remix IDE: https://remix.ethereum.org/
2. Створити файл `Token.sol` та вставити код контракту Token
3. Вибрати компілятор версії **0.6.12**
4. Скомпілювати контракт

#### Крок 2: Деплой Attacker контракту
1. В розділі "Deploy & Run Transactions" вибрати:
   - **Environment**: Injected Provider - MetaMask
   - **Contract**: Attacker
2. У конструктор вставити адресу інстансу Token з Ethernaut
3. Натиснути **Deploy**
4. Підтвердити транзакцію в MetaMask

#### Крок 3: Виконання атаки
1. Перевірити початковий баланс:
   ```javascript
   await contract.balanceOf(player)
   // Очікується: 20
   ```
2. У розгорнутому Attacker контракті натиснути **attack()**
3. Підтвердити транзакцію в MetaMask
4. Перевірити новий баланс:
   ```javascript
   await contract.balanceOf(player)
   // Очікується: дуже велике число (> 1e70)
   ```

#### Крок 4: Завершення
1. На сторінці Ethernaut натиснути **Submit Instance**
2. Підтвердити транзакцію
3. Дочекатися повідомлення про успішне проходження рівня

### Тест 2: Delegation Challenge

#### Метод 1: Через консоль браузера (швидкий)

1. Отримати інстанс на Ethernaut
2. Відкрити консоль браузера (F12)
3. Перевірити поточного власника:
   ```javascript
   await contract.owner()
   ```
4. Виконати атаку:
   ```javascript
   await contract.sendTransaction({
       from: player,
       data: web3.eth.abi.encodeFunctionSignature('pwn()')
   })
   ```
5. Перевірити, що власник змінився:
   ```javascript
   await contract.owner()
   // Має співпадати з адресою player
   ```
6. Натиснути **Submit Instance**

#### Метод 2: Через Attacker контракт (детальний)

##### Крок 1: Підготовка
1. Створити файли `Delegate.sol` та `Delegation.sol` в Remix
2. Створити файл `DelegateAttacker.sol`
3. Вибрати компілятор версії **0.8.0+**
4. Скомпілювати всі контракти

##### Крок 2: Деплой Attacker
1. В розділі "Deploy & Run Transactions":
   - **Environment**: Injected Provider - MetaMask
   - **Contract**: Attacker (з файлу DelegateAttacker.sol)
2. У конструктор вставити адресу інстансу Delegation з Ethernaut
3. Натиснути **Deploy** та підтвердити в MetaMask

##### Крок 3: Виконання атаки
1. Перевірити поточного власника:
   ```javascript
   await contract.owner()
   // Запам'ятати адресу
   ```
2. У розгорнутому Attacker контракті натиснути **attack()**
3. Підтвердити транзакцію в MetaMask
4. Дочекатися підтвердження
5. Перевірити нового власника:
   ```javascript
   await contract.owner()
   // Має бути ваша адреса (player)
   ```

##### Крок 4: Завершення
1. Натиснути **Submit Instance** на Ethernaut
2. Підтвердити транзакцію
3. Дочекатися успішного завершення

## Очікувані результати

### Token Challenge

| Крок | Дія | Очікуваний результат |
|------|-----|---------------------|
| 1 | Перевірка початкового балансу | 20 токенів |
| 2 | Виклик attack() | Транзакція успішна |
| 3 | Перевірка балансу після атаки | > 1.15e77 токенів |
| 4 | Submit Instance | "Level completed!" |

### Delegation Challenge

| Крок | Дія | Очікуваний результат |
|------|-----|---------------------|
| 1 | await contract.owner() | Початкова адреса власника |
| 2 | Виклик pwn() через delegatecall | Транзакція успішна |
| 3 | await contract.owner() | Адреса player |
| 4 | Submit Instance | "Level completed!" |

## Важливі адреси (з прикладів)

### Token Challenge
- **Instance Address**: `0x810331dF3C5054D79DD2EfC42bd7125287c3cB23`
- **Alternative Instance**: `0x6549da35ec715dF1669dEe8bF4C6dD806B5E611b`

### Delegation Challenge
- **Instance Address**: `0x4bF49aA16860abdF8818DD24048d90cEEdA6d659`
- **Attacker Contract**: `0x8e183854f0F047C20821002d44a6760372F77556`
- **Level Address**: `0x73379d8B82Fda494ee59555f333DF7D44483fD58`

## Можливі помилки та вирішення

### Token Challenge

**Помилка:** "Transaction reverted"
**Причина:** Використовується версія Solidity >= 0.8.0 з автоматичною перевіркою overflow
**Рішення:** Переконайтесь, що використовується компілятор версії 0.6.x

**Помилка:** Баланс не змінився
**Причина:** Неправильна адреса інстансу Token
**Рішення:** Перевірте адресу в конструкторі Attacker контракту

### Delegation Challenge

**Помилка:** "Attack failed"
**Причина:** Неправильно сформована сигнатура функції
**Рішення:** Використайте точну сигнатуру: `bytes4(keccak256("pwn()"))`

**Помилка:** Власник не змінився
**Причина:** Транзакція відправлена не з акаунту player
**Рішення:** Переконайтесь, що в MetaMask обрано правильний акаунт

## Навчальні матеріали

### Рекомендовані ресурси
- [Ethernaut](https://ethernaut.openzeppelin.com/) - платформа з завданнями
- [Solidity Documentation](https://docs.soliditylang.org/) - офіційна документація
- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/) - безпечні контракти
- [SWC Registry](https://swcregistry.io/) - база вразливостей смарт-контрактів

### Ключові концепції
- **Integer Overflow/Underflow** - переповнення цілих чисел
- **Delegatecall** - виклик коду в контексті іншого контракту
- **Fallback Function** - функція за замовчуванням
- **Storage Layout** - розташування змінних у пам'яті
- **Context Preservation** - збереження контексту виконання

## Best Practices для безпеки

1. **Використовуйте останні версії Solidity** (>= 0.8.0)
2. **Застосовуйте перевірки SafeMath** для старих версій
3. **Обмежуйте використання delegatecall**
4. **Проводьте аудит коду** перед деплоєм
5. **Тестуйте на тестових мережах**
6. **Використовуйте модифікатори для контролю доступу**
7. **Документуйте небезпечні функції**
8. **Уникайте складної логіки у fallback функціях**

## Disclaimer

⚠️ **ВАЖЛИВО:** Цей код призначений виключно для навчальних цілей у контексті платформи Ethernaut. Використання цих методів проти реальних смарт-контрактів без дозволу є незаконним та неетичним. Завжди проводьте тестування безпеки лише на власних контрактах або в авторизованих середовищах.

## Ліцензія

MIT

---

**Навчальна мета:** Розуміння типових вразливостей смарт-контрактів та методів їх запобігання для розробки безпечних децентралізованих додатків.

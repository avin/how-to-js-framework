# Урок 3: Представление элементов

## Проблема: как описать дерево элементов?

В предыдущем уроке мы писали виртуальные узлы руками:

```javascript
const vNode = {
  type: 'div',
  props: { class: 'container' },
  children: [
    { type: 'h1', props: null, children: ['Hello'] }
  ]
};
```

Это работает, но:
- Много повторяющегося кода
- Легко ошибиться в структуре
- Неудобно для сложных деревьев

**Решение**: создадим функцию-хелпер для создания виртуальных узлов.

## Функция h() - hyperscript

В React это `React.createElement()`, в Vue — `h()`. Мы тоже создадим функцию `h()`:

```javascript
// Вместо этого:
{ type: 'div', props: { class: 'container' }, children: ['Hello'] }

// Пишем так:
h('div', { class: 'container' }, 'Hello')
```

### Базовая реализация

```javascript
/**
 * Создаёт виртуальный узел
 * @param {string} type - Тип элемента ('div', 'span', и т.д.)
 * @param {object} props - Свойства и атрибуты
 * @param {...any} children - Дочерние элементы
 */
function h(type, props, ...children) {
  return {
    type,
    props: props || {},
    children
  };
}
```

### Использование

```javascript
// Простой элемент
const button = h('button', { class: 'btn' }, 'Нажми меня');
console.log(button);
// {
//   type: 'button',
//   props: { class: 'btn' },
//   children: ['Нажми меня']
// }

// Вложенная структура
const app = h('div', { id: 'app' },
  h('h1', null, 'Заголовок'),
  h('p', null, 'Параграф')
);
```

## Улучшаем функцию h()

### Проблема 1: Обработка массивов детей

Что если мы передаём массив:

```javascript
const items = ['Один', 'Два', 'Три'];
h('ul', null, items.map(text => h('li', null, text)));
```

Это создаст:
```javascript
{
  type: 'ul',
  children: [
    [ // Массив внутри массива!
      { type: 'li', children: ['Один'] },
      { type: 'li', children: ['Два'] },
      { type: 'li', children: ['Три'] }
    ]
  ]
}
```

**Решение**: Сплющиваем массивы

```javascript
function h(type, props, ...children) {
  return {
    type,
    props: props || {},
    children: flattenChildren(children)
  };
}

function flattenChildren(children) {
  return children.reduce((flat, child) => {
    // Если ребёнок - массив, рекурсивно сплющиваем
    if (Array.isArray(child)) {
      return flat.concat(flattenChildren(child));
    }
    // Иначе просто добавляем
    return flat.concat(child);
  }, []);
}
```

Теперь:
```javascript
h('ul', null,
  ['Один', 'Два', 'Три'].map(text => h('li', null, text))
);
// children: [{ type: 'li' }, { type: 'li' }, { type: 'li' }] ✓
```

### Проблема 2: Фильтрация null/undefined/boolean

В React можно писать:

```javascript
{isLoggedIn && <WelcomeMessage />}
```

Если `isLoggedIn = false`, то `false && <WelcomeMessage />` вернёт `false`. Нам нужно игнорировать такие значения:

```javascript
function flattenChildren(children) {
  return children.reduce((flat, child) => {
    if (Array.isArray(child)) {
      return flat.concat(flattenChildren(child));
    }
    // Игнорируем null, undefined, boolean
    if (child == null || typeof child === 'boolean') {
      return flat;
    }
    return flat.concat(child);
  }, []);
}
```

Теперь работает:
```javascript
h('div', null,
  true && h('p', null, 'Видимый'),
  false && h('p', null, 'Невидимый')
);
// children: [{ type: 'p', children: ['Видимый'] }] ✓
```

### Проблема 3: Числа и другие примитивы

Числа — валидные дети:

```javascript
h('div', null, 'Счёт: ', 42)
```

Но наш код сейчас создаст `children: ['Счёт: ', 42]`. При рендеринге в DOM нужно превратить числа в строки:

```javascript
function flattenChildren(children) {
  return children.reduce((flat, child) => {
    if (Array.isArray(child)) {
      return flat.concat(flattenChildren(child));
    }
    if (child == null || typeof child === 'boolean') {
      return flat;
    }
    // Превращаем числа в строки
    if (typeof child === 'number') {
      return flat.concat(String(child));
    }
    return flat.concat(child);
  }, []);
}
```

## Окончательная версия h()

```javascript
/**
 * Создаёт виртуальный DOM узел
 * @param {string} type - Тип элемента
 * @param {object|null} props - Свойства элемента
 * @param {...(VNode|string|number|boolean|null|undefined|Array)} children - Дочерние элементы
 * @returns {VNode}
 */
function h(type, props, ...children) {
  return {
    type,
    props: props || {},
    children: flattenChildren(children)
  };
}

/**
 * Рекурсивно сплющивает массивы детей и фильтрует невалидные значения
 */
function flattenChildren(children) {
  return children.reduce((flat, child) => {
    // Рекурсивно обрабатываем массивы
    if (Array.isArray(child)) {
      return flat.concat(flattenChildren(child));
    }
    // Игнорируем null, undefined, boolean
    if (child == null || typeof child === 'boolean') {
      return flat;
    }
    // Превращаем числа в строки
    if (typeof child === 'number') {
      return flat.concat(String(child));
    }
    // Добавляем строки и виртуальные узлы как есть
    return flat.concat(child);
  }, []);
}
```

## Типы виртуальных узлов

Теперь у нас есть два типа виртуальных узлов:

### 1. Элементы

```javascript
{
  type: 'div',        // Тип - строка
  props: { /* ... */ },
  children: [/* ... */]
}
```

### 2. Текстовые узлы

```javascript
"Простой текст"  // Просто строка
```

Позже мы добавим третий тип — **компоненты** (type будет функция, а не строка).

## Примеры использования

### Простой список

```javascript
const todos = ['Купить молоко', 'Написать код', 'Поспать'];

const todoList = h('ul', { class: 'todo-list' },
  todos.map(text => h('li', null, text))
);

console.log(todoList);
// {
//   type: 'ul',
//   props: { class: 'todo-list' },
//   children: [
//     { type: 'li', props: {}, children: ['Купить молоко'] },
//     { type: 'li', props: {}, children: ['Написать код'] },
//     { type: 'li', props: {}, children: ['Поспать'] }
//   ]
// }
```

### Условный рендеринг

```javascript
function UserGreeting({ name, isAdmin }) {
  return h('div', { class: 'greeting' },
    h('h1', null, `Привет, ${name}!`),
    isAdmin && h('p', { class: 'admin-badge' }, 'Администратор')
  );
}

// Для админа
UserGreeting({ name: 'Иван', isAdmin: true });
// children: [
//   { type: 'h1', ... },
//   { type: 'p', class: 'admin-badge', ... }
// ]

// Для обычного пользователя
UserGreeting({ name: 'Пётр', isAdmin: false });
// children: [
//   { type: 'h1', ... }
// ] - p отфильтрован!
```

### Счётчик

```javascript
function Counter(count) {
  return h('div', { class: 'counter' },
    h('p', null, 'Счёт: ', count),
    h('button', { onclick: 'increment()' }, '+'),
    h('button', { onclick: 'decrement()' }, '-')
  );
}

const vNode = Counter(42);
// {
//   type: 'div',
//   props: { class: 'counter' },
//   children: [
//     { type: 'p', children: ['Счёт: ', '42'] },
//     { type: 'button', children: ['+'] },
//     { type: 'button', children: ['-'] }
//   ]
// }
```

## Сравнение с React.createElement

Наша функция `h()` очень похожа на `React.createElement()`:

```javascript
// React
React.createElement('div', { className: 'app' },
  React.createElement('h1', null, 'Hello')
);

// Наш h()
h('div', { class: 'app' },
  h('h1', null, 'Hello')
);
```

Результат практически идентичен! React делает ещё некоторые оптимизации (добавляет `$$typeof`, `key`, `ref`), но базовая идея та же.

## JSX vs h()

JSX — это синтаксический сахар над `createElement` / `h`:

```jsx
// JSX (требует транспиляции)
<div className="app">
  <h1>Hello</h1>
</div>

// Компилируется в:
h('div', { className: 'app' },
  h('h1', null, 'Hello')
);
```

Мы используем `h()` напрямую, потому что:
- Не требуется сборка
- Код работает в браузере как есть
- Проще понять что происходит

Но для production приложений JSX удобнее.

## Создаём модуль фреймворка

Давайте организуем наш код:

**Файл: `nano-framework.js`**

```javascript
/**
 * Nano Framework - минимальный фреймворк с Virtual DOM
 */

// ============= Типы виртуальных узлов =============

/**
 * @typedef {Object} VNode
 * @property {string} type - Тип элемента ('div', 'span', и т.д.)
 * @property {Object} props - Свойства и атрибуты
 * @property {Array<VNode|string>} children - Дочерние элементы
 */

// ============= Создание виртуальных узлов =============

/**
 * Создаёт виртуальный DOM узел (hyperscript)
 */
export function h(type, props, ...children) {
  return {
    type,
    props: props || {},
    children: flattenChildren(children)
  };
}

/**
 * Рекурсивно сплющивает и нормализует дочерние элементы
 */
function flattenChildren(children) {
  return children.reduce((flat, child) => {
    if (Array.isArray(child)) {
      return flat.concat(flattenChildren(child));
    }
    if (child == null || typeof child === 'boolean') {
      return flat;
    }
    if (typeof child === 'number') {
      return flat.concat(String(child));
    }
    return flat.concat(child);
  }, []);
}
```

## Тестируем

**Файл: `test.html`**

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Тест h()</title>
</head>
<body>
  <script type="module">
    import { h } from './nano-framework.js';

    // Тест 1: Простой элемент
    const simple = h('div', { class: 'box' }, 'Hello');
    console.log('Простой элемент:', simple);

    // Тест 2: Вложенность
    const nested = h('div', null,
      h('h1', null, 'Заголовок'),
      h('p', null, 'Текст')
    );
    console.log('Вложенная структура:', nested);

    // Тест 3: Массивы
    const list = h('ul', null,
      [1, 2, 3].map(n => h('li', null, 'Элемент ', n))
    );
    console.log('Список:', list);

    // Тест 4: Условия
    const conditional = h('div', null,
      true && h('p', null, 'Показан'),
      false && h('p', null, 'Скрыт')
    );
    console.log('Условный рендеринг:', conditional);

    // Тест 5: Числа
    const withNumber = h('div', null, 'Счёт: ', 42);
    console.log('С числом:', withNumber);
  </script>
</body>
</html>
```

Откройте файл в браузере и посмотрите в консоль — все тесты должны работать!

## Что дальше?

Теперь у нас есть удобный способ **описывать** виртуальное дерево. В следующем уроке мы научимся **превращать** его в реальный DOM — реализуем функцию рендеринга.

## Задание

1. Добавьте поддержку фрагментов — специальный тип узла, который не создаёт обёртку:
```javascript
h(Fragment, null,
  h('li', null, '1'),
  h('li', null, '2')
)
// Должно рендериться без <fragment>
```

2. Создайте функцию-хелпер для создания классов:
```javascript
function classNames(...args) {
  // classNames('btn', isActive && 'active', 'large')
  // → 'btn active large' или 'btn large'
}
```

3. Представьте в виде вызовов `h()` следующую структуру:
```html
<div class="card">
  <img src="avatar.jpg" alt="Avatar">
  <h2>Заголовок карточки</h2>
  <p>Описание карточки</p>
  <button class="btn primary">Действие</button>
</div>
```

## Ключевые выводы

- Функция `h()` упрощает создание виртуальных узлов
- Она нормализует детей (сплющивает массивы, фильтрует false/null/undefined)
- Числа превращаются в строки
- Результат — обычный JavaScript объект
- Это аналог `React.createElement` или `Vue h`
- JSX — синтаксический сахар над такими функциями

---

[← Предыдущий урок: Virtual DOM - концепция](./02-virtual-dom-concept.md) | [Следующий урок: Первый рендеринг →](./04-first-rendering.md)

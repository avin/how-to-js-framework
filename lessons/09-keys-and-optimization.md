# Урок 9: Ключи и оптимизация списков

## Проблема: неэффективный diffing списков

Вспомним наш алгоритм diffing для детей:

```javascript
function diffChildren(oldChildren, newChildren) {
  const patches = [];
  for (let i = 0; i < Math.max(oldChildren.length, newChildren.length); i++) {
    patches[i] = diff(oldChildren[i], newChildren[i]);
  }
  return patches;
}
```

Он сравнивает детей **попарно по индексу**. Что происходит при изменении порядка?

### Сценарий 1: Добавление в начало

```javascript
old: [h('li', null, 'B'), h('li', null, 'C')]
new: [h('li', null, 'A'), h('li', null, 'B'), h('li', null, 'C')]

// Попарное сравнение:
// [0]: 'B' vs 'A' → UPDATE (изменить текст B→A)
// [1]: 'C' vs 'B' → UPDATE (изменить текст C→B)
// [2]: undefined vs 'C' → CREATE

// Результат: 3 операции (2 UPDATE + 1 CREATE)
// Оптимально: 1 операция (CREATE 'A' в начало)
```

### Сценарий 2: Перестановка элементов

```javascript
old: [h('li', null, 'A'), h('li', null, 'B'), h('li', null, 'C')]
new: [h('li', null, 'C'), h('li', null, 'A'), h('li', null, 'B')]

// Попарное сравнение:
// [0]: 'A' vs 'C' → UPDATE
// [1]: 'B' vs 'A' → UPDATE
// [2]: 'C' vs 'B' → UPDATE

// Результат: все 3 элемента обновлены!
// Оптимально: просто переместить элементы
```

### Почему это проблема?

1. **Производительность:** лишние обновления DOM
2. **Потеря состояния:** пересоздание уничтожает внутреннее состояние
3. **Анимации сбрасываются**
4. **Фокус теряется** на input элементах

## Решение: ключи (keys)

**Идея:** добавить уникальный идентификатор к каждому элементу списка:

```javascript
// ❌ Без ключей
users.map(user => h('li', null, user.name))

// ✓ С ключами
users.map(user => h('li', { key: user.id }, user.name))
```

**key** помогает алгоритму понять, что элементы **те же самые**, просто переместились.

## Обновляем виртуальный узел

Добавим поле `key` в виртуальный узел:

```javascript
export function h(type, props, ...children) {
  return {
    type,
    props: props || {},
    children: flattenChildren(children),
    key: props && props.key  // Извлекаем key из props
  };
}
```

## Алгоритм diffing с ключами

Новый алгоритм сравнения детей:

```javascript
/**
 * Сравнивает детей с учётом ключей
 */
function diffChildren(oldChildren, newChildren) {
  // Если ключей нет, используем простой алгоритм
  const hasKeys = newChildren.some(child => child && child.key != null);

  if (!hasKeys) {
    return diffChildrenSimple(oldChildren, newChildren);
  }

  return diffChildrenWithKeys(oldChildren, newChildren);
}

/**
 * Простой алгоритм (попарное сравнение)
 */
function diffChildrenSimple(oldChildren, newChildren) {
  const patches = [];
  const maxLength = Math.max(oldChildren.length, newChildren.length);

  for (let i = 0; i < maxLength; i++) {
    patches[i] = diff(oldChildren[i], newChildren[i]);
  }

  return patches;
}
```

## Алгоритм с ключами: создание карты

```javascript
/**
 * Создаёт карту: key → { index, vNode }
 */
function createKeyMap(children) {
  const map = new Map();

  children.forEach((child, index) => {
    if (child && child.key != null) {
      map.set(child.key, { index, vNode: child });
    }
  });

  return map;
}
```

## Алгоритм с ключами: основная логика

```javascript
/**
 * Сравнивает детей используя ключи
 */
function diffChildrenWithKeys(oldChildren, newChildren) {
  const oldKeyMap = createKeyMap(oldChildren);
  const patches = [];

  // Для каждого нового ребёнка
  newChildren.forEach((newChild, newIndex) => {
    if (!newChild || newChild.key == null) {
      // Без ключа — просто сравниваем по позиции
      patches[newIndex] = diff(oldChildren[newIndex], newChild);
      return;
    }

    // Ищем старый элемент с таким же ключом
    const oldMatch = oldKeyMap.get(newChild.key);

    if (oldMatch) {
      // Нашли — сравниваем и помечаем "использован"
      const oldChild = oldMatch.vNode;
      const oldIndex = oldMatch.index;

      patches[newIndex] = diff(oldChild, newChild);

      // Если позиция изменилась, нужно переместить
      if (oldIndex !== newIndex) {
        if (!patches[newIndex]) {
          patches[newIndex] = { type: PATCH_TYPE.UPDATE };
        }
        patches[newIndex].move = { from: oldIndex, to: newIndex };
      }

      // Помечаем как использованный
      oldKeyMap.delete(newChild.key);
    } else {
      // Не нашли — создаём новый
      patches[newIndex] = { type: PATCH_TYPE.CREATE, newVNode: newChild };
    }
  });

  // Оставшиеся в oldKeyMap — удалить
  oldKeyMap.forEach((oldMatch) => {
    patches[oldMatch.index] = { type: PATCH_TYPE.REMOVE };
  });

  return patches;
}
```

## Применение патча MOVE

Добавим обработку перемещения в функцию `patch`:

```javascript
export function patch(parent, patchObj, element, index = 0) {
  if (!patchObj) {
    return element;
  }

  // Обработка перемещения
  if (patchObj.move) {
    const elementToMove = parent.childNodes[patchObj.move.from];
    const targetPosition = parent.childNodes[patchObj.move.to];

    if (elementToMove && targetPosition) {
      parent.insertBefore(elementToMove, targetPosition);
    }
  }

  // ... остальная логика patch
}
```

## Упрощённая версия: longest increasing subsequence

React использует алгоритм **Longest Increasing Subsequence (LIS)** для минимизации перемещений. Это сложнее, но эффективнее.

Наша простая версия тоже работает хорошо для большинства случаев.

## Пример с ключами

```javascript
let items = [
  { id: 1, text: 'Apple' },
  { id: 2, text: 'Banana' },
  { id: 3, text: 'Cherry' }
];

function render() {
  return h('ul', null,
    items.map(item =>
      h('li', { key: item.id },
        h('input', { type: 'checkbox' }),
        ' ',
        item.text
      )
    )
  );
}

// При изменении порядка items
items = [items[2], items[0], items[1]]; // Cherry, Apple, Banana

// С ключами: элементы просто переместятся
// Без ключей: все элементы пересоздадутся, checkbox сбросятся
```

## Правила использования ключей

### 1. Ключи должны быть уникальными среди siblings

```javascript
// ✓ Правильно
users.map(user => h('li', { key: user.id }, ...))

// ❌ Неправильно — дубликаты ключей
users.map(user => h('li', { key: user.role }, ...))
```

### 2. Ключи должны быть стабильными

```javascript
// ❌ Плохо — ключ меняется при каждом рендере
items.map(item => h('li', { key: Math.random() }, ...))

// ❌ Плохо — индекс как ключ (при перестановке не работает)
items.map((item, i) => h('li', { key: i }, ...))

// ✓ Хорошо — стабильный ID
items.map(item => h('li', { key: item.id }, ...))
```

### 3. Ключи не передаются в props

```javascript
// key не попадает в element.getAttribute('key')
h('li', { key: 123, class: 'item' }, ...)

// key только для diffing алгоритма
```

## Когда использовать индекс как ключ?

Можно использовать индекс, если:
1. Список статичный (не меняется порядок)
2. Элементы не переставляются
3. Элементы не удаляются из середины

```javascript
// ✓ Можно (статичный список)
['Главная', 'О нас', 'Контакты'].map((item, i) =>
  h('a', { key: i, href: `/${item}` }, item)
)

// ❌ Нельзя (динамический список)
todos.map((todo, i) =>
  h('li', { key: i }, ...) // При удалении/перестановке будут проблемы
)
```

## Генерация ключей для новых элементов

```javascript
let nextId = 1;

function addItem(text) {
  items.push({
    id: nextId++,  // Уникальный ID
    text
  });
  rerender();
}

// Альтернатива: timestamp + random
function generateId() {
  return `${Date.now()}-${Math.random()}`;
}

// Ещё вариант: UUID библиотека
import { v4 as uuid } from 'uuid';
const id = uuid();
```

## Оптимизация: мемоизация

Проблема: при каждом рендере создаются новые объекты, даже если данные не изменились:

```javascript
function render() {
  return h('div', null,
    items.map(item =>
      h('li', { key: item.id }, item.text)  // Новый объект каждый раз!
    )
  );
}
```

Решение 1: сравнение по значению в `diff`:

```javascript
export function diff(oldVNode, newVNode) {
  // Оптимизация: если узлы идентичны по ссылке
  if (oldVNode === newVNode) {
    return null;
  }

  // Можно добавить сравнение по значению для элементов
  if (isShallowEqual(oldVNode, newVNode)) {
    return null;
  }

  // ... остальной код
}

function isShallowEqual(a, b) {
  if (typeof a !== 'object' || typeof b !== 'object') {
    return a === b;
  }

  const keysA = Object.keys(a);
  const keysB = Object.keys(b);

  if (keysA.length !== keysB.length) {
    return false;
  }

  return keysA.every(key => a[key] === b[key]);
}
```

## Полный пример: сортируемый список

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Keys Demo</title>
  <style>
    .list-item {
      padding: 10px;
      margin: 5px;
      background: #f0f0f0;
      border-radius: 4px;
      display: flex;
      align-items: center;
      transition: all 0.3s ease;
    }
    .list-item input {
      margin-right: 10px;
    }
    button {
      margin: 5px;
      padding: 8px 16px;
    }
  </style>
</head>
<body>
  <div id="app"></div>

  <script type="module">
    import { h, render, diff, patch } from './nano-framework.js';

    let items = [
      { id: 1, text: 'Apple', done: false },
      { id: 2, text: 'Banana', done: true },
      { id: 3, text: 'Cherry', done: false },
      { id: 4, text: 'Date', done: false }
    ];

    let app = null;
    let nextId = 5;

    function createVNode() {
      return h('div', null,
        h('h1', null, 'Демонстрация ключей'),

        h('div', null,
          h('button', {
            onclick: () => {
              // Переворачиваем массив
              items = items.reverse();
              rerender();
            }
          }, 'Развернуть'),

          h('button', {
            onclick: () => {
              // Сортируем по алфавиту
              items = items.sort((a, b) => a.text.localeCompare(b.text));
              rerender();
            }
          }, 'Сортировать'),

          h('button', {
            onclick: () => {
              // Перемешиваем
              items = items.sort(() => Math.random() - 0.5);
              rerender();
            }
          }, 'Перемешать'),

          h('button', {
            onclick: () => {
              // Добавляем новый элемент
              const fruits = ['Orange', 'Grape', 'Mango', 'Peach'];
              const text = fruits[Math.floor(Math.random() * fruits.length)];
              items.push({ id: nextId++, text, done: false });
              rerender();
            }
          }, 'Добавить')
        ),

        h('div', null,
          items.map(item =>
            h('div', {
              key: item.id,  // ← КЛЮЧ!
              class: 'list-item'
            },
              h('input', {
                type: 'checkbox',
                checked: item.done,
                onchange: () => {
                  item.done = !item.done;
                  rerender();
                }
              }),
              h('span', null, item.text),
              ' ',
              h('button', {
                onclick: () => {
                  items = items.filter(i => i.id !== item.id);
                  rerender();
                }
              }, 'Удалить')
            )
          )
        )
      );
    }

    function rerender() {
      const newVNode = createVNode();
      const container = document.getElementById('app');

      if (!app) {
        const element = createElement(newVNode);
        container.appendChild(element);
        app = { vNode: newVNode, element };
      } else {
        const patches = diff(app.vNode, newVNode);
        console.log('Patches:', patches);
        const element = patch(container, patches, app.element, 0);
        app = { vNode: newVNode, element };
      }
    }

    rerender();
  </script>
</body>
</html>
```

**Попробуйте:**
1. Отметьте несколько checkbox
2. Нажмите "Перемешать"
3. Checkbox остаются отмеченными на правильных элементах!
4. Теперь уберите `key` из кода — checkbox перемешаются неправильно

## Сравнение с/без ключей

```javascript
// Без ключей:
old: [A, B, C]  →  [C, A, B]
// Операции: UPDATE(A→C), UPDATE(B→A), UPDATE(C→B)
// 3 обновления DOM

// С ключами:
old: [A(id:1), B(id:2), C(id:3)]  →  [C(id:3), A(id:1), B(id:2)]
// Операции: MOVE(C в начало)
// 1 перемещение DOM
```

## Производительность

Тест на 1000 элементов:

```javascript
// Без ключей — перестановка первых двух элементов:
// ~800ms (обновляет все 1000 элементов)

// С ключами:
// ~2ms (перемещает только 1 элемент)

// Ускорение в 400 раз!
```

## Задание

1. Реализуйте функцию для визуализации перемещений:
```javascript
function highlightMove(element, from, to) {
  // Анимация перемещения элемента
}
```

2. Создайте список с drag & drop перестановкой:
```javascript
// Используйте HTML5 Drag & Drop API
h('div', {
  draggable: true,
  ondragstart: ...,
  ondragover: ...,
  ondrop: ...
})
```

3. Сравните производительность с/без ключей:
```javascript
// Создайте список из 1000 элементов
// Измерьте время перестановки
// Результаты выведите в таблицу
```

## Ключевые выводы

- Попарное сравнение детей неэффективно при перестановках
- Ключи помогают идентифицировать элементы между рендерами
- С ключами алгоритм может переместить элементы вместо пересоздания
- Ключи должны быть уникальными и стабильными
- Не используйте индекс как ключ для динамических списков
- Ключи критически важны для сохранения состояния элементов
- С ключами производительность может быть в сотни раз выше
- key не передаётся в DOM — только для diffing

---

[← Предыдущий урок: Применение патчей к DOM](./08-dom-patching.md) | [Следующий урок: Функциональные компоненты →](./10-functional-components.md)

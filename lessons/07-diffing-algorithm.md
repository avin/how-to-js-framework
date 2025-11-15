# Урок 7: Алгоритм diffing

## Проблема: неэффективное обновление

Сейчас при каждом изменении мы полностью пересоздаём DOM:

```javascript
function render() {
  const vNode = createApp(); // Новое виртуальное дерево
  mount(vNode, container);   // Удаляем всё и создаём заново
}
```

**Проблемы:**
- Медленно для больших приложений
- Теряется фокус на input элементах
- Сбрасываются анимации и состояние
- Бесполезная работа — пересоздаём то, что не изменилось

**Пример проблемы:**

```javascript
let count = 0;

function render() {
  mount(
    h('div', null,
      h('input', { type: 'text' }), // ← Этот input пересоздаётся
      h('button', { onclick: () => { count++; render(); } }, count)
    ),
    container
  );
}

// При клике на button:
// 1. Весь div удаляется
// 2. Создаётся новый div с новым input
// 3. Input теряет фокус и введённый текст!
```

## Решение: Diffing

**Идея:** вместо полного пересоздания сравниваем (diff) старое и новое виртуальные деревья, находим различия и применяем только необходимые изменения.

```javascript
oldVNode:  h('div', null, h('p', null, 'Hello'))
newVNode:  h('div', null, h('p', null, 'Hi'))
           ↓
Diff:      Текст <p> изменился: 'Hello' → 'Hi'
           ↓
Patch:     textNode.textContent = 'Hi'  // Только это!
```

## Типы изменений

При сравнении двух виртуальных узлов возможны следующие случаи:

### 1. Узлы идентичны
```javascript
old: h('div', { class: 'box' }, 'Text')
new: h('div', { class: 'box' }, 'Text')
// Ничего не делаем
```

### 2. Изменился текст
```javascript
old: 'Hello'
new: 'Hi'
// element.textContent = 'Hi'
```

### 3. Изменился тип элемента
```javascript
old: h('div', null, ...)
new: h('span', null, ...)
// Заменяем весь элемент
```

### 4. Изменились свойства
```javascript
old: h('div', { class: 'old' }, ...)
new: h('div', { class: 'new' }, ...)
// Обновляем только изменившиеся props
```

### 5. Изменились дети
```javascript
old: h('ul', null, h('li', null, 'A'), h('li', null, 'B'))
new: h('ul', null, h('li', null, 'A'), h('li', null, 'C'))
//                                                      ↑ изменился
// Рекурсивно сравниваем детей
```

## Типы патчей

Создадим константы для типов изменений:

```javascript
/**
 * Типы патчей (изменений)
 */
export const PATCH_TYPE = {
  CREATE: 'CREATE',       // Создать новый узел
  REMOVE: 'REMOVE',       // Удалить узел
  REPLACE: 'REPLACE',     // Заменить узел
  UPDATE: 'UPDATE',       // Обновить свойства
  TEXT: 'TEXT'           // Обновить текст
};
```

## Структура патча

Патч — это объект, описывающий изменение:

```javascript
{
  type: PATCH_TYPE.UPDATE,
  props: {
    add: { class: 'active' },              // Добавить/изменить
    remove: [{ name: 'disabled', value: true }] // Удалить (нужно знать старое значение)
  }
}

{
  type: PATCH_TYPE.REPLACE,
  newVNode: h('span', null, 'Text')
}

{
  type: PATCH_TYPE.TEXT,
  text: 'New text'
}
```

## Функция diff

Основная функция сравнения:

```javascript
/**
 * Сравнивает два виртуальных узла и возвращает патч
 * @param {VNode|string} oldVNode - Старый виртуальный узел
 * @param {VNode|string} newVNode - Новый виртуальный узел
 * @returns {Object|null} Патч или null если изменений нет
 */
export function diff(oldVNode, newVNode) {
  // 1. Новый узел не существует → удаляем
  if (newVNode == null) {
    return { type: PATCH_TYPE.REMOVE };
  }

  // 2. Старого узла не было → создаём
  if (oldVNode == null) {
    return { type: PATCH_TYPE.CREATE, newVNode };
  }

  // 3. Разные типы узлов
  if (typeof oldVNode !== typeof newVNode) {
    return { type: PATCH_TYPE.REPLACE, newVNode };
  }

  // 4. Оба — текстовые узлы
  if (typeof oldVNode === 'string') {
    if (oldVNode !== newVNode) {
      return { type: PATCH_TYPE.TEXT, text: newVNode };
    }
    return null; // Текст не изменился
  }

  // 5. Оба — элементы, но разные теги
  if (oldVNode.type !== newVNode.type) {
    return { type: PATCH_TYPE.REPLACE, newVNode };
  }

  // 6. Тот же элемент — проверяем props и children
  return diffElement(oldVNode, newVNode);
}
```

## Сравнение элементов

```javascript
/**
 * Сравнивает два элемента одинакового типа
 */
function diffElement(oldVNode, newVNode) {
  const propsPatch = diffProps(oldVNode.props, newVNode.props);
  const childrenPatches = diffChildren(
    oldVNode.children,
    newVNode.children
  );

  // Если нет изменений
  if (!propsPatch && childrenPatches.length === 0) {
    return null;
  }

  return {
    type: PATCH_TYPE.UPDATE,
    props: propsPatch,
    children: childrenPatches
  };
}
```

## Сравнение свойств

```javascript
/**
 * Сравнивает props двух элементов
 * @returns {Object|null} { add: {...}, remove: [{name,value}, ...] } или null
 */
function diffProps(oldProps, newProps) {
  const patches = {
    add: {},    // Новые/изменённые свойства
    remove: []  // Удалённые свойства + их прежние значения (например, обработчики событий)
  };

  // Проверяем изменённые и новые свойства
  for (const key in newProps) {
    if (oldProps[key] !== newProps[key]) {
      patches.add[key] = newProps[key];
    }
  }

  // Проверяем удалённые свойства
  for (const key in oldProps) {
    if (!(key in newProps)) {
      patches.remove.push({
        name: key,
        value: oldProps[key]
      });
    }
  }

  // Если нет изменений
  if (Object.keys(patches.add).length === 0 &&
      patches.remove.length === 0) {
    return null;
  }

  return patches;
}
```

**Пример:**

```javascript
const oldProps = { class: 'btn', disabled: true };
const newProps = { class: 'btn active', id: 'submit' };

diffProps(oldProps, newProps);
// {
//   add: { class: 'btn active', id: 'submit' },
//   remove: [{ name: 'disabled', value: true }]
// }
```

## Сравнение детей (простая версия)

Начнём с простого алгоритма — попарное сравнение:

```javascript
/**
 * Сравнивает массивы детей
 * @returns {Array} Массив патчей для каждого ребёнка
 */
function diffChildren(oldChildren, newChildren) {
  const patches = [];
  const maxLength = Math.max(oldChildren.length, newChildren.length);

  for (let i = 0; i < maxLength; i++) {
    patches[i] = diff(oldChildren[i], newChildren[i]);
  }

  return patches;
}
```

**Как это работает:**

```javascript
old: [h('li', null, 'A'), h('li', null, 'B'), h('li', null, 'C')]
new: [h('li', null, 'A'), h('li', null, 'X'), h('li', null, 'C'), h('li', null, 'D')]

patches: [
  null,                                  // A не изменилось
  { type: 'TEXT', text: 'X' },          // B → X
  null,                                  // C не изменилось
  { type: 'CREATE', newVNode: ... }     // D добавлено
]
```

## Проблема простого алгоритма

При изменении порядка элементов:

```javascript
old: [h('li', null, 'A'), h('li', null, 'B'), h('li', null, 'C')]
new: [h('li', null, 'C'), h('li', null, 'A'), h('li', null, 'B')]

// Простой алгоритм подумает:
// A → C  (заменить)
// B → A  (заменить)
// C → B  (заменить)
// Все три элемента будут заменены!

// Оптимальное решение:
// Просто переместить элементы
```

**Решение:** использовать ключи (keys) — разберём в уроке 9.

## Оптимизация: ранний выход

Если виртуальные узлы идентичны по ссылке, можно сразу вернуть `null`:

```javascript
export function diff(oldVNode, newVNode) {
  // Оптимизация: если узлы идентичны
  if (oldVNode === newVNode) {
    return null;
  }

  // ... остальной код
}
```

Это полезно при использовании мемоизации (React.memo).

## Полная реализация diff

```javascript
// nano-framework.js

export const PATCH_TYPE = {
  CREATE: 'CREATE',
  REMOVE: 'REMOVE',
  REPLACE: 'REPLACE',
  UPDATE: 'UPDATE',
  TEXT: 'TEXT'
};

export function diff(oldVNode, newVNode) {
  // Оптимизация: идентичные узлы
  if (oldVNode === newVNode) {
    return null;
  }

  // Удаление
  if (newVNode == null) {
    return { type: PATCH_TYPE.REMOVE };
  }

  // Создание
  if (oldVNode == null) {
    return { type: PATCH_TYPE.CREATE, newVNode };
  }

  // Разные типы (string vs object)
  if (typeof oldVNode !== typeof newVNode) {
    return { type: PATCH_TYPE.REPLACE, newVNode };
  }

  // Текстовые узлы
  if (typeof oldVNode === 'string') {
    return oldVNode !== newVNode
      ? { type: PATCH_TYPE.TEXT, text: newVNode }
      : null;
  }

  // Разные теги
  if (oldVNode.type !== newVNode.type) {
    return { type: PATCH_TYPE.REPLACE, newVNode };
  }

  // Один и тот же элемент
  return diffElement(oldVNode, newVNode);
}

function diffElement(oldVNode, newVNode) {
  const propsPatch = diffProps(oldVNode.props, newVNode.props);
  const childrenPatches = diffChildren(oldVNode.children, newVNode.children);

  if (!propsPatch && childrenPatches.every(p => p === null)) {
    return null;
  }

  return {
    type: PATCH_TYPE.UPDATE,
    props: propsPatch,
    children: childrenPatches
  };
}

function diffProps(oldProps, newProps) {
  const patches = { add: {}, remove: [] };

  // Изменённые и новые
  for (const key in newProps) {
    if (oldProps[key] !== newProps[key]) {
      patches.add[key] = newProps[key];
    }
  }

  // Удалённые
  for (const key in oldProps) {
    if (!(key in newProps)) {
      patches.remove.push({
        name: key,
        value: oldProps[key]
      });
    }
  }

  if (Object.keys(patches.add).length === 0 && patches.remove.length === 0) {
    return null;
  }

  return patches;
}

function diffChildren(oldChildren, newChildren) {
  const patches = [];
  const maxLength = Math.max(oldChildren.length, newChildren.length);

  for (let i = 0; i < maxLength; i++) {
    patches[i] = diff(oldChildren[i], newChildren[i]);
  }

  return patches;
}
```

## Пример использования

```javascript
const oldVNode = h('div', { class: 'container' },
  h('h1', null, 'Привет'),
  h('p', null, 'Старый текст')
);

const newVNode = h('div', { class: 'container active' },
  h('h1', null, 'Привет'),
  h('p', null, 'Новый текст'),
  h('button', null, 'Кнопка')
);

const patch = diff(oldVNode, newVNode);

console.log(patch);
// {
//   type: 'UPDATE',
//   props: {
//     add: { class: 'container active' },
//     remove: []
//   },
//   children: [
//     null,  // h1 не изменился
//     { type: 'TEXT', text: 'Новый текст' },  // p изменился
//     { type: 'CREATE', newVNode: {...} }     // button добавлен
//   ]
// }
```

## Сложность алгоритма

**Текущий алгоритм:** O(n), где n — количество узлов.

**Почему не O(n³)?**
- Мы НЕ ищем оптимальное решение для любых деревьев
- Предполагаем: элементы с разными типами создают разные деревья
- Предполагаем: дети сравниваются попарно по позиции
- Для списков используем keys (урок 9)

React использует похожий подход — эвристический O(n) вместо точного O(n³).

## Что дальше?

Мы научились **находить** различия (diff), но не применять их. В следующем уроке реализуем **patching** — применение патчей к реальному DOM.

## Задание

1. Добавьте логирование в функцию `diff` для визуализации работы:
```javascript
export function diff(oldVNode, newVNode) {
  console.log('Comparing:', oldVNode, newVNode);
  // ...
}
```

2. Реализуйте функцию для подсчёта изменений:
```javascript
function countPatches(patch) {
  // Вернуть количество изменений в патче
}
```

3. Протестируйте diff на разных сценариях:
```javascript
// Сценарий 1: Только изменение текста
// Сценарий 2: Добавление элемента
// Сценарий 3: Удаление элемента
// Сценарий 4: Замена элемента
// Сценарий 5: Изменение нескольких свойств
```

## Ключевые выводы

- Diffing — это сравнение двух виртуальных деревьев
- Патч описывает изменение: тип + данные
- Основные типы патчей: CREATE, REMOVE, REPLACE, UPDATE, TEXT
- Сравнение идёт рекурсивно: элемент → props → children
- Попарное сравнение детей простое, но неоптимальное
- Для списков нужны keys (следующие уроки)
- Сложность O(n) вместо O(n³) благодаря эвристикам
- Diff только находит изменения, не применяет их
- Следующий шаг — patching (применение патчей к DOM)

---

[← Предыдущий урок: Рендеринг списков и фрагментов](./06-lists-and-fragments.md) | [Следующий урок: Применение патчей к DOM →](./08-dom-patching.md)

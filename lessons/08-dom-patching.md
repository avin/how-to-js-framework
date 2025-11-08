# Урок 8: Применение патчей к DOM

## Проблема: патчи бесполезны без применения

В прошлом уроке мы научились находить различия между виртуальными деревьями:

```javascript
const patch = diff(oldVNode, newVNode);
// { type: 'UPDATE', props: {...}, children: [...] }
```

Но это просто **описание** изменений. Теперь нужно научиться **применять** эти изменения к реальному DOM.

## Задача patching

**Patching** — это процесс применения патча к DOM элементу:

```javascript
// Было:
<div class="old">Hello</div>

// Патч:
{ type: 'UPDATE', props: { add: { class: 'new' } }, children: [...] }

// Стало:
<div class="new">Hi</div>
```

## Функция patch

Основная функция применения патчей:

```javascript
/**
 * Применяет патч к DOM элементу
 * @param {HTMLElement} parent - Родительский элемент
 * @param {Object} patch - Патч для применения
 * @param {HTMLElement} element - Элемент для патча
 * @param {number} index - Индекс элемента в родителе
 * @returns {HTMLElement} Новый или обновлённый элемент
 */
export function patch(parent, patch, element, index = 0) {
  if (!patch) {
    return element;
  }

  switch (patch.type) {
    case PATCH_TYPE.CREATE:
      return patchCreate(parent, patch, index);

    case PATCH_TYPE.REMOVE:
      return patchRemove(parent, element);

    case PATCH_TYPE.REPLACE:
      return patchReplace(parent, patch, element);

    case PATCH_TYPE.TEXT:
      return patchText(element, patch);

    case PATCH_TYPE.UPDATE:
      return patchUpdate(element, patch);

    default:
      return element;
  }
}
```

## CREATE: создание нового элемента

```javascript
/**
 * Создаёт и добавляет новый элемент
 */
function patchCreate(parent, patch, index) {
  const newElement = createElement(patch.newVNode);

  // Вставляем на нужную позицию
  if (index < parent.childNodes.length) {
    parent.insertBefore(newElement, parent.childNodes[index]);
  } else {
    parent.appendChild(newElement);
  }

  return newElement;
}
```

**Пример:**

```javascript
// Было: <ul><li>A</li></ul>
// Добавляем: <li>B</li>

const parent = document.querySelector('ul');
const patch = { type: PATCH_TYPE.CREATE, newVNode: h('li', null, 'B') };

patchCreate(parent, patch, 1);
// Стало: <ul><li>A</li><li>B</li></ul>
```

## REMOVE: удаление элемента

```javascript
/**
 * Удаляет элемент из DOM
 */
function patchRemove(parent, element) {
  parent.removeChild(element);
  return null;
}
```

**Важно:** также нужно очистить обработчики событий, чтобы избежать утечек памяти. Но `removeChild` в большинстве случаев достаточно — браузер очищает события автоматически.

## REPLACE: замена элемента

```javascript
/**
 * Заменяет старый элемент новым
 */
function patchReplace(parent, patch, oldElement) {
  const newElement = createElement(patch.newVNode);
  parent.replaceChild(newElement, oldElement);
  return newElement;
}
```

**Когда используется:**
- Изменился тип тега: `<div>` → `<span>`
- Текстовый узел стал элементом или наоборот

```javascript
// Было: <div>Text</div>
// Стало: <span>Text</span>

const patch = {
  type: PATCH_TYPE.REPLACE,
  newVNode: h('span', null, 'Text')
};

patchReplace(parent, patch, oldDiv);
```

## TEXT: обновление текста

```javascript
/**
 * Обновляет текстовое содержимое
 */
function patchText(element, patch) {
  element.textContent = patch.text;
  return element;
}
```

**Пример:**

```javascript
// Было: <p>Hello</p>
// Стало: <p>Hi</p>

const p = document.querySelector('p');
const patch = { type: PATCH_TYPE.TEXT, text: 'Hi' };

patchText(p, patch);
```

## UPDATE: обновление элемента

Самый сложный случай — обновляем props и рекурсивно патчим детей:

```javascript
/**
 * Обновляет свойства и детей элемента
 */
function patchUpdate(element, patch) {
  // 1. Обновляем свойства
  if (patch.props) {
    patchProps(element, patch.props);
  }

  // 2. Рекурсивно обновляем детей
  if (patch.children) {
    patchChildren(element, patch.children);
  }

  return element;
}
```

### Обновление свойств

```javascript
/**
 * Обновляет свойства элемента
 * @param {HTMLElement} element
 * @param {Object} propsPatch - { add: {...}, remove: [...] }
 */
function patchProps(element, propsPatch) {
  // Удаляем старые свойства
  if (propsPatch.remove) {
    propsPatch.remove.forEach(propName => {
      removeProp(element, propName);
    });
  }

  // Добавляем/обновляем новые свойства
  if (propsPatch.add) {
    Object.keys(propsPatch.add).forEach(propName => {
      setProp(element, propName, propsPatch.add[propName]);
    });
  }
}
```

**Пример:**

```javascript
// Было: <button class="btn" disabled>Click</button>
// Стало: <button class="btn active" id="submit">Click</button>

const propsPatch = {
  add: { class: 'btn active', id: 'submit' },
  remove: ['disabled']
};

patchProps(button, propsPatch);
// <button class="btn active" id="submit">Click</button>
```

### Обновление детей

```javascript
/**
 * Обновляет детей элемента
 * @param {HTMLElement} parent
 * @param {Array} childrenPatches - Массив патчей для каждого ребёнка
 */
function patchChildren(parent, childrenPatches) {
  childrenPatches.forEach((childPatch, i) => {
    // Получаем текущий дочерний элемент
    const childElement = parent.childNodes[i];

    // Применяем патч
    patch(parent, childPatch, childElement, i);
  });
}
```

**Важный момент:** при удалении элементов индексы сдвигаются. Нужно патчить в обратном порядке:

```javascript
function patchChildren(parent, childrenPatches) {
  // Патчим с конца, чтобы удаления не сдвигали индексы
  for (let i = childrenPatches.length - 1; i >= 0; i--) {
    const childPatch = childrenPatches[i];
    const childElement = parent.childNodes[i];
    patch(parent, childPatch, childElement, i);
  }
}
```

## Полная реализация patch

```javascript
// nano-framework.js

import { PATCH_TYPE } from './diff.js';
import { createElement } from './create-element.js';
import { setProp, removeProp } from './props.js';

/**
 * Применяет патч к DOM
 */
export function patch(parent, patch, element, index = 0) {
  if (!patch) {
    return element;
  }

  switch (patch.type) {
    case PATCH_TYPE.CREATE: {
      const newElement = createElement(patch.newVNode);
      if (index < parent.childNodes.length) {
        parent.insertBefore(newElement, parent.childNodes[index]);
      } else {
        parent.appendChild(newElement);
      }
      return newElement;
    }

    case PATCH_TYPE.REMOVE: {
      parent.removeChild(element);
      return null;
    }

    case PATCH_TYPE.REPLACE: {
      const newElement = createElement(patch.newVNode);
      parent.replaceChild(newElement, element);
      return newElement;
    }

    case PATCH_TYPE.TEXT: {
      element.textContent = patch.text;
      return element;
    }

    case PATCH_TYPE.UPDATE: {
      // Обновляем свойства
      if (patch.props) {
        patchProps(element, patch.props);
      }

      // Обновляем детей (с конца!)
      if (patch.children) {
        for (let i = patch.children.length - 1; i >= 0; i--) {
          const childPatch = patch.children[i];
          const childElement = element.childNodes[i];
          patch(element, childPatch, childElement, i);
        }
      }

      return element;
    }

    default:
      return element;
  }
}

/**
 * Обновляет свойства элемента
 */
function patchProps(element, propsPatch) {
  if (propsPatch.remove) {
    propsPatch.remove.forEach(name => {
      removeProp(element, name);
    });
  }

  if (propsPatch.add) {
    Object.keys(propsPatch.add).forEach(name => {
      setProp(element, name, propsPatch.add[name]);
    });
  }
}
```

## Объединяем всё: функция render

Теперь объединим diff и patch в одну функцию:

```javascript
/**
 * Рендерит виртуальное дерево в контейнер
 * @param {VNode} newVNode - Новое виртуальное дерево
 * @param {HTMLElement} container - Контейнер
 * @param {VNode} oldVNode - Старое виртуальное дерево (для diff)
 * @param {HTMLElement} oldElement - Старый DOM элемент
 * @returns {Object} { vNode, element } для следующего рендера
 */
export function render(newVNode, container, oldVNode = null, oldElement = null) {
  // Первый рендер
  if (!oldVNode) {
    const element = createElement(newVNode);
    container.innerHTML = '';
    container.appendChild(element);
    return { vNode: newVNode, element };
  }

  // Повторный рендер — используем diff + patch
  const patches = diff(oldVNode, newVNode);
  const element = patch(container, patches, oldElement, 0);

  return { vNode: newVNode, element };
}
```

## Полный пример: работающий счётчик

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Nano Framework - Patching</title>
  <style>
    .counter {
      font-family: sans-serif;
      padding: 20px;
      max-width: 300px;
    }
    .count {
      font-size: 48px;
      font-weight: bold;
      color: #0066cc;
    }
    button {
      padding: 10px 20px;
      font-size: 16px;
      margin: 5px;
      cursor: pointer;
    }
    input {
      padding: 8px;
      font-size: 16px;
      margin: 10px 0;
      width: 100%;
    }
  </style>
</head>
<body>
  <div id="app"></div>

  <script type="module">
    import { h, render, diff, patch } from './nano-framework.js';

    let count = 0;
    let input = '';
    let app = null; // Хранилище для старого vNode и element

    function createVNode() {
      return h('div', { class: 'counter' },
        h('h1', null, 'Счётчик с patching'),

        // Input — теперь не теряет фокус!
        h('input', {
          type: 'text',
          value: input,
          placeholder: 'Введите текст...',
          oninput: (e) => {
            input = e.target.value;
            rerender();
          }
        }),

        h('p', null, 'Вы ввели: ', input || '(пусто)'),

        h('div', { class: 'count' }, count),

        h('div', null,
          h('button', {
            onclick: () => {
              count--;
              rerender();
            }
          }, '−'),
          h('button', {
            onclick: () => {
              count++;
              rerender();
            }
          }, '+'),
          h('button', {
            onclick: () => {
              count = 0;
              rerender();
            }
          }, 'Сброс')
        )
      );
    }

    function rerender() {
      const newVNode = createVNode();
      const container = document.getElementById('app');

      if (!app) {
        // Первый рендер
        const element = createElement(newVNode);
        container.appendChild(element);
        app = { vNode: newVNode, element };
      } else {
        // Патчинг
        const patches = diff(app.vNode, newVNode);

        // Логируем патчи для отладки
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
1. Введите текст в input
2. Кликните по кнопкам +/-
3. Input НЕ теряет фокус!
4. Откройте консоль — видите патчи

## Отладка патчинга

Добавим визуализацию изменений:

```javascript
function visualizePatch(patch, depth = 0) {
  if (!patch) return;

  const indent = '  '.repeat(depth);

  console.log(`${indent}${patch.type}`);

  if (patch.type === PATCH_TYPE.UPDATE) {
    if (patch.props) {
      console.log(`${indent}  Props:`, patch.props);
    }
    if (patch.children) {
      console.log(`${indent}  Children:`);
      patch.children.forEach((child, i) => {
        console.log(`${indent}    [${i}]:`);
        visualizePatch(child, depth + 2);
      });
    }
  }
}

// Использование
const patches = diff(oldVNode, newVNode);
visualizePatch(patches);
```

## Оптимизация: батчинг обновлений

Проблема: несколько изменений подряд вызывают несколько рендеров:

```javascript
count++;
input = 'new';
// Два рендера!
```

Решение — батчинг (группировка) обновлений:

```javascript
let isRenderScheduled = false;
let pendingUpdates = [];

function scheduleRender(updateFn) {
  pendingUpdates.push(updateFn);

  if (!isRenderScheduled) {
    isRenderScheduled = true;
    requestAnimationFrame(() => {
      // Применяем все обновления
      pendingUpdates.forEach(fn => fn());
      pendingUpdates = [];

      // Рендерим один раз
      rerender();

      isRenderScheduled = false;
    });
  }
}

// Использование
scheduleRender(() => { count++; });
scheduleRender(() => { input = 'new'; });
// Рендер произойдёт один раз в следующем кадре
```

## Производительность

Сравним производительность:

```javascript
// Без patching (полная перерисовка)
function renderWithoutPatch() {
  const start = performance.now();
  container.innerHTML = '';
  container.appendChild(createElement(vNode));
  const end = performance.now();
  console.log('Without patch:', end - start, 'ms');
}

// С patching
function renderWithPatch() {
  const start = performance.now();
  const patches = diff(oldVNode, newVNode);
  patch(container, patches, oldElement);
  const end = performance.now();
  console.log('With patch:', end - start, 'ms');
}

// Для большого списка (1000 элементов):
// Without patch: ~50ms
// With patch (изменение одного элемента): ~2ms
// Ускорение в 25 раз!
```

## Задание

1. Добавьте счётчик применённых патчей:
```javascript
let patchCount = 0;

function patch(parent, patch, element, index) {
  if (patch) patchCount++;
  // ... остальной код
}
```

2. Создайте визуализацию патчинга — подсвечивайте изменённые элементы:
```javascript
function highlightChange(element) {
  element.style.transition = 'background 0.3s';
  element.style.background = 'yellow';
  setTimeout(() => {
    element.style.background = '';
  }, 300);
}
```

3. Реализуйте функцию для измерения производительности:
```javascript
function benchmark(name, fn, iterations = 100) {
  const start = performance.now();
  for (let i = 0; i < iterations; i++) {
    fn();
  }
  const end = performance.now();
  console.log(`${name}: ${(end - start) / iterations}ms per iteration`);
}
```

## Ключевые выводы

- Patching — это применение описанных изменений к реальному DOM
- 5 типов патчей: CREATE, REMOVE, REPLACE, TEXT, UPDATE
- Каждый тип патча требует своего способа применения
- UPDATE рекурсивно обновляет props и children
- Важно патчить детей с конца, чтобы удаления не сдвигали индексы
- Diff + Patch = эффективное обновление DOM
- Input элементы теперь не теряют фокус!
- Батчинг позволяет группировать обновления
- Patching значительно быстрее полной перерисовки

---

[← Предыдущий урок: Алгоритм diffing](./07-diffing-algorithm.md) | [Следующий урок: Ключи и оптимизация списков →](./09-keys-and-optimization.md)

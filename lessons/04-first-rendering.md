# Урок 4: Первый рендеринг

## Задача

Теперь, когда у нас есть виртуальные узлы, нужно научиться превращать их в реальный DOM. Это называется **монтирование** (mounting).

```javascript
// Было: виртуальный узел
const vNode = h('div', { class: 'app' }, 'Hello');

// Нужно: реальный DOM элемент
const domElement = <div class="app">Hello</div>
```

## Типы узлов для рендеринга

У нас два типа виртуальных узлов:

1. **Текстовые узлы** — строки
2. **Элементы** — объекты с type, props, children

Начнём с простого.

## Рендеринг текстовых узлов

Текстовый узел — это просто строка:

```javascript
function createTextNode(text) {
  return document.createTextNode(text);
}

// Использование
const textNode = createTextNode('Привет');
document.body.appendChild(textNode);
```

Просто! DOM API уже предоставляет `createTextNode`.

## Рендеринг элементов

Для элемента нужно:
1. Создать DOM элемент с нужным тегом
2. Установить свойства и атрибуты
3. Рекурсивно отрендерить детей
4. Добавить детей в элемент

### Базовая версия

```javascript
/**
 * Создаёт DOM элемент из виртуального узла
 * @param {VNode|string} vNode - Виртуальный узел
 * @returns {Node} DOM узел
 */
function createElement(vNode) {
  // Если это текстовый узел
  if (typeof vNode === 'string') {
    return document.createTextNode(vNode);
  }

  // Создаём элемент
  const element = document.createElement(vNode.type);

  // Рекурсивно создаём детей
  vNode.children.forEach(child => {
    const childElement = createElement(child);
    element.appendChild(childElement);
  });

  return element;
}
```

Тестируем:

```javascript
const vNode = h('div', null,
  h('h1', null, 'Заголовок'),
  h('p', null, 'Параграф')
);

const dom = createElement(vNode);
document.body.appendChild(dom);

// Результат в HTML:
// <div>
//   <h1>Заголовок</h1>
//   <p>Параграф</p>
// </div>
```

Работает! Но мы пока не используем `props`.

## Добавляем свойства

Свойства элемента бывают разных типов:
- Обычные атрибуты: `class`, `id`, `title`
- Специальные свойства: `value`, `checked`
- События: `onclick`, `onchange`
- Стили: `style`

Начнём с простого — обычных атрибутов:

```javascript
function createElement(vNode) {
  if (typeof vNode === 'string') {
    return document.createTextNode(vNode);
  }

  const element = document.createElement(vNode.type);

  // Устанавливаем атрибуты
  setProps(element, vNode.props);

  // Создаём детей
  vNode.children.forEach(child => {
    element.appendChild(createElement(child));
  });

  return element;
}

/**
 * Устанавливает свойства на DOM элемент
 */
function setProps(element, props) {
  Object.keys(props).forEach(key => {
    setProp(element, key, props[key]);
  });
}

/**
 * Устанавливает одно свойство
 */
function setProp(element, name, value) {
  // Пока просто устанавливаем как атрибут
  element.setAttribute(name, value);
}
```

Проверяем:

```javascript
const vNode = h('div', { class: 'container', id: 'app' },
  h('button', { class: 'btn', title: 'Кликни' }, 'Кнопка')
);

const dom = createElement(vNode);
// <div class="container" id="app">
//   <button class="btn" title="Кликни">Кнопка</button>
// </div>
```

## Улучшаем setProp

### Проблема 1: className vs class

В JSX используют `className` вместо `class` (потому что `class` — зарезервированное слово в JS). Поддержим оба варианта:

```javascript
function setProp(element, name, value) {
  // className → class
  if (name === 'className') {
    element.setAttribute('class', value);
    return;
  }

  element.setAttribute(name, value);
}
```

### Проблема 2: Свойства vs атрибуты

Некоторые свойства нужно устанавливать через DOM API, а не через `setAttribute`:

```javascript
// НЕ работает:
input.setAttribute('value', 'текст');  // Установит начальное значение
input.value = 'новый текст';           // Установит текущее значение ✓

// НЕ работает:
checkbox.setAttribute('checked', true); // Строка "true", а не boolean
checkbox.checked = true;                // Boolean ✓
```

Исправляем:

```javascript
function setProp(element, name, value) {
  // className → class
  if (name === 'className') {
    name = 'class';
  }

  // Специальные свойства DOM
  if (name === 'value' || name === 'checked') {
    element[name] = value;
    return;
  }

  // Обычные атрибуты
  element.setAttribute(name, value);
}
```

### Проблема 3: События

События начинаются с `on`:

```javascript
h('button', { onclick: () => alert('Клик!') }, 'Кликни')
```

Нужно:
```javascript
function setProp(element, name, value) {
  if (name === 'className') {
    name = 'class';
  }

  // События
  if (name.startsWith('on')) {
    const eventName = name.substring(2).toLowerCase(); // onclick → click
    element.addEventListener(eventName, value);
    return;
  }

  // Специальные свойства
  if (name === 'value' || name === 'checked') {
    element[name] = value;
    return;
  }

  element.setAttribute(name, value);
}
```

Проверяем:

```javascript
let count = 0;

const vNode = h('button', {
  onclick: () => {
    count++;
    console.log('Счёт:', count);
  }
}, 'Кликни меня');

const button = createElement(vNode);
document.body.appendChild(button);
// Клик по кнопке → "Счёт: 1", "Счёт: 2", ...
```

### Проблема 4: style

Стили могут быть строкой или объектом:

```javascript
// Строка
h('div', { style: 'color: red; font-size: 16px' })

// Объект (удобнее)
h('div', { style: { color: 'red', fontSize: '16px' } })
```

Поддержим объекты:

```javascript
function setProp(element, name, value) {
  if (name === 'className') {
    name = 'class';
  }

  if (name.startsWith('on')) {
    const eventName = name.substring(2).toLowerCase();
    element.addEventListener(eventName, value);
    return;
  }

  // Стили
  if (name === 'style') {
    if (typeof value === 'string') {
      element.setAttribute('style', value);
    } else {
      // Объект стилей
      Object.keys(value).forEach(styleName => {
        element.style[styleName] = value[styleName];
      });
    }
    return;
  }

  if (name === 'value' || name === 'checked') {
    element[name] = value;
    return;
  }

  element.setAttribute(name, value);
}
```

Проверяем:

```javascript
const vNode = h('div', {
  style: {
    color: 'blue',
    fontSize: '20px',
    backgroundColor: 'yellow'
  }
}, 'Стилизованный текст');

const div = createElement(vNode);
// <div style="color: blue; font-size: 20px; background-color: yellow">
```

## Полная версия createElement

```javascript
/**
 * Создаёт реальный DOM элемент из виртуального узла
 * @param {VNode|string} vNode
 * @returns {Node}
 */
export function createElement(vNode) {
  // Текстовый узел
  if (typeof vNode === 'string') {
    return document.createTextNode(vNode);
  }

  // Создаём элемент
  const element = document.createElement(vNode.type);

  // Устанавливаем свойства
  setProps(element, vNode.props);

  // Рекурсивно создаём и добавляем детей
  vNode.children.forEach(child => {
    element.appendChild(createElement(child));
  });

  return element;
}

/**
 * Устанавливает все свойства на элемент
 */
function setProps(element, props) {
  Object.keys(props).forEach(key => {
    setProp(element, key, props[key]);
  });
}

/**
 * Устанавливает одно свойство
 */
function setProp(element, name, value) {
  // className → class
  if (name === 'className') {
    name = 'class';
  }

  // События: onclick, onchange и т.д.
  if (name.startsWith('on')) {
    const eventName = name.substring(2).toLowerCase();
    element.addEventListener(eventName, value);
    return;
  }

  // Стили
  if (name === 'style') {
    if (typeof value === 'string') {
      element.setAttribute('style', value);
    } else {
      Object.keys(value).forEach(styleName => {
        element.style[styleName] = value[styleName];
      });
    }
    return;
  }

  // Специальные DOM свойства
  if (name === 'value' || name === 'checked') {
    element[name] = value;
    return;
  }

  // Обычные атрибуты
  element.setAttribute(name, value);
}
```

## Функция mount — монтирование приложения

Добавим удобную функцию для монтирования:

```javascript
/**
 * Монтирует виртуальное дерево в реальный DOM
 * @param {VNode} vNode - Виртуальный узел
 * @param {HTMLElement} container - Контейнер для монтирования
 * @returns {Node} Созданный DOM элемент
 */
export function mount(vNode, container) {
  // Создаём DOM элемент
  const element = createElement(vNode);

  // Очищаем контейнер
  container.innerHTML = '';

  // Добавляем элемент
  container.appendChild(element);

  return element;
}
```

## Полный пример

**Файл: `nano-framework.js`**

```javascript
// ... (функция h из предыдущего урока)

export function createElement(vNode) {
  if (typeof vNode === 'string') {
    return document.createTextNode(vNode);
  }

  const element = document.createElement(vNode.type);
  setProps(element, vNode.props);

  vNode.children.forEach(child => {
    element.appendChild(createElement(child));
  });

  return element;
}

function setProps(element, props) {
  Object.keys(props).forEach(key => {
    setProp(element, key, props[key]);
  });
}

function setProp(element, name, value) {
  if (name === 'className') {
    name = 'class';
  }

  if (name.startsWith('on')) {
    const eventName = name.substring(2).toLowerCase();
    element.addEventListener(eventName, value);
    return;
  }

  if (name === 'style') {
    if (typeof value === 'string') {
      element.setAttribute('style', value);
    } else {
      Object.keys(value).forEach(styleName => {
        element.style[styleName] = value[styleName];
      });
    }
    return;
  }

  if (name === 'value' || name === 'checked') {
    element[name] = value;
    return;
  }

  element.setAttribute(name, value);
}

export function mount(vNode, container) {
  const element = createElement(vNode);
  container.innerHTML = '';
  container.appendChild(element);
  return element;
}
```

**Файл: `app.html`**

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Nano Framework - First Render</title>
  <style>
    .counter {
      font-family: sans-serif;
      padding: 20px;
      border: 2px solid #333;
      border-radius: 8px;
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
  </style>
</head>
<body>
  <div id="app"></div>

  <script type="module">
    import { h, mount } from './nano-framework.js';

    let count = 0;

    function render() {
      const vNode = h('div', { class: 'counter' },
        h('h1', null, 'Счётчик'),
        h('div', { class: 'count' }, count),
        h('div', null,
          h('button', {
            onclick: () => {
              count--;
              render();
            }
          }, '−'),
          h('button', {
            onclick: () => {
              count++;
              render();
            }
          }, '+'),
          h('button', {
            onclick: () => {
              count = 0;
              render();
            }
          }, 'Сброс')
        )
      );

      mount(vNode, document.getElementById('app'));
    }

    render();
  </script>
</body>
</html>
```

Откройте файл в браузере — счётчик работает! При каждом клике мы вызываем `render()`, который:
1. Создаёт новое виртуальное дерево
2. Превращает его в DOM
3. Заменяет старый DOM новым

## Проблема текущего подхода

Наш счётчик работает, но **неэффективно**:

```javascript
function render() {
  const vNode = h('div', { class: 'counter' },
    // ... всё дерево заново
  );

  mount(vNode, container);
  // container.innerHTML = '' ← удаляет всё!
  // appendChild(newElement) ← создаёт всё заново!
}
```

При каждом изменении `count` мы:
- Удаляем весь DOM
- Создаём весь DOM заново
- Теряем фокус на элементах
- Сбрасываем состояние (позиция скролла, анимации и т.д.)

Попробуйте добавить `<input>` в счётчик — при каждом изменении count фокус будет сбрасываться!

## Решение: diffing и patching

Вместо полной замены нужно:
1. Сравнить старое и новое виртуальное дерево (**diffing**)
2. Найти минимальный набор изменений
3. Применить только эти изменения (**patching**)

Это мы реализуем в следующих уроках!

## Что дальше?

Сейчас у нас есть:
- ✅ Создание виртуальных узлов (`h`)
- ✅ Рендеринг в реальный DOM (`createElement`, `mount`)
- ❌ Эффективное обновление

В следующих уроках:
- Урок 5: Расширенная работа с атрибутами (data-*, aria-*, и т.д.)
- Урок 7-8: Алгоритм diffing и patching

## Задание

1. Добавьте поддержку атрибута `disabled` для кнопок:
```javascript
h('button', { disabled: true }, 'Неактивная')
```

2. Создайте простую форму с input и выводом введённого текста:
```javascript
let text = '';
h('div', null,
  h('input', {
    value: text,
    oninput: (e) => { /* обновить text и перерисовать */ }
  }),
  h('p', null, 'Вы ввели: ', text)
)
```

3. Добавьте поддержку `data-*` атрибутов:
```javascript
h('div', { 'data-user-id': '123', 'data-role': 'admin' })
```

## Ключевые выводы

- `createElement` превращает виртуальные узлы в реальный DOM
- Текстовые узлы → `createTextNode`
- Элементы → `createElement` + установка props + рекурсия для детей
- Свойства бывают разных типов: атрибуты, DOM-свойства, события, стили
- `mount` — удобная функция для монтирования приложения
- Текущий подход неэффективен — пересоздаём весь DOM при каждом изменении
- Решение — diffing и patching (следующие уроки)

---

[← Предыдущий урок: Представление элементов](./03-element-representation.md) | [Следующий урок: Работа с атрибутами и свойствами →](./05-attributes-and-properties.md)

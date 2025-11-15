# Урок 5: Работа с атрибутами и свойствами

## Проблема: различия между атрибутами и свойствами

В предыдущем уроке мы создали базовую функцию `setProp`, но она не учитывает многие нюансы работы с DOM. Давайте разберёмся с различиями между **атрибутами** (attributes) и **свойствами** (properties).

### Атрибуты vs Свойства

```javascript
// HTML атрибут
<input value="initial">

// JavaScript свойство
input.value = "current"
```

**Атрибуты** — это то, что написано в HTML. Они устанавливают **начальное** значение.
**Свойства** — это то, что есть на DOM-объекте. Они отражают **текущее** состояние.

```javascript
const input = document.querySelector('input');

// Начальное значение (атрибут)
console.log(input.getAttribute('value')); // "initial"

// Пользователь ввёл "hello"
console.log(input.value); // "hello" (свойство изменилось)
console.log(input.getAttribute('value')); // всё ещё "initial" (атрибут не изменился)
```

## Категории свойств

### 1. Свойства, которые должны быть DOM properties

```javascript
// ❌ Неправильно
element.setAttribute('value', 'текст');      // Установит начальное значение
element.setAttribute('checked', true);       // Не работает! Установит строку "true"

// ✓ Правильно
element.value = 'текст';       // Установит текущее значение
element.checked = true;        // Установит boolean
```

Такие свойства: `value`, `checked`, `selected`, `disabled`, `readOnly`.

### 2. Boolean атрибуты

Boolean атрибуты работают особым образом:

```html
<!-- Наличие = true -->
<button disabled>Кнопка</button>
<input checked>

<!-- Отсутствие = false -->
<button>Кнопка</button>
<input>
```

Важно помнить, что имя HTML-атрибута не всегда совпадает с DOM-свойством.
Например, `<input readonly>` должен синхронизироваться с `element.readOnly`
с заглавной буквой `O`. Поэтому полезно держать карту соответствий между
атрибутом и свойством.

```javascript
// ❌ Неправильно
element.setAttribute('disabled', false); // Строка "false" - атрибут всё равно есть!

// ✓ Правильно
element.disabled = false;  // Свойство
// или
element.removeAttribute('disabled'); // Удаляем атрибут
```

### 3. События

События всегда должны устанавливаться через `addEventListener`:

```javascript
// ❌ Старый способ (не рекомендуется)
element.onclick = handler;

// ✓ Современный способ
element.addEventListener('click', handler);
```

### 4. className vs class

В JavaScript нельзя использовать `class` (зарезервированное слово):

```javascript
// ❌ Синтаксическая ошибка
element.class = 'btn';

// ✓ Правильно
element.className = 'btn';
// или
element.setAttribute('class', 'btn');
```

### 5. style

Стили можно устанавливать двумя способами:

```javascript
// Строка
element.setAttribute('style', 'color: red; font-size: 16px');

// Объект (удобнее)
element.style.color = 'red';
element.style.fontSize = '16px'; // camelCase!
```

### 6. data-* атрибуты

Пользовательские атрибуты:

```html
<div data-user-id="123" data-role="admin">
```

Через API dataset:
```javascript
element.dataset.userId = '123';   // data-user-id
element.dataset.role = 'admin';   // data-role
```

### 7. aria-* атрибуты (доступность)

```html
<button aria-label="Закрыть" aria-pressed="true">
```

Эти устанавливаются как обычные атрибуты.

## Улучшенная функция setProp

Создадим умную функцию, которая правильно обрабатывает все случаи:

```javascript
/**
 * Устанавливает свойство на DOM элемент
 * @param {HTMLElement} element
 * @param {string} name - Имя свойства
 * @param {any} value - Значение
 */
function setProp(element, name, value) {
  // 1. className → class
  if (name === 'className') {
    name = 'class';
  }

  // 2. События
  if (isEventProp(name)) {
    const eventName = extractEventName(name);
    element.addEventListener(eventName, value);
    return;
  }

  // 3. style (объект или строка)
  if (name === 'style') {
    setStyle(element, value);
    return;
  }

  // 4. Свойства, которые должны быть DOM properties
  if (isPropertyName(name)) {
    element[name] = value;
    return;
  }

  // 5. Boolean атрибуты
  if (isBooleanAttr(name)) {
    setBooleanAttr(element, name, value);
    return;
  }

  // 6. Обычные атрибуты
  // Если значение null/undefined - удаляем атрибут
  if (value == null) {
    element.removeAttribute(name);
  } else {
    element.setAttribute(name, value);
  }
}
```

### Вспомогательные функции

```javascript
/**
 * Проверяет, является ли имя событием
 */
function isEventProp(name) {
  return name.startsWith('on');
}

/**
 * Извлекает имя события из prop (onclick → click)
 */
function extractEventName(name) {
  return name.substring(2).toLowerCase();
}

/**
 * Свойства, которые должны устанавливаться через DOM API
 */
const DOM_PROPERTIES = new Set([
  'value',
  'checked',
  'selected',
  'innerHTML',
  'textContent',
  'innerText'
]);

function isPropertyName(name) {
  return DOM_PROPERTIES.has(name);
}

/**
 * Boolean атрибуты и соответствующие DOM-свойства.
 * Для readonly, например, HTML-атрибут пишется строчными,
 * а DOM-свойство — readOnly (с заглавной O).
 */
const BOOLEAN_ATTRS = new Map([
  ['disabled', 'disabled'],
  ['readonly', 'readOnly'],
  ['required', 'required'],
  ['autofocus', 'autofocus'],
  ['autoplay', 'autoplay'],
  ['controls', 'controls'],
  ['loop', 'loop'],
  ['muted', 'muted'],
  ['multiple', 'multiple'],
  ['hidden', 'hidden']
]);

function isBooleanAttr(name) {
  return BOOLEAN_ATTRS.has(name);
}

/**
 * Устанавливает boolean атрибут
 */
function setBooleanAttr(element, name, value) {
  const domProp = BOOLEAN_ATTRS.get(name) || name;

  if (value) {
    element.setAttribute(name, '');
    element[domProp] = true; // Также синхронизируем DOM-свойство
  } else {
    element.removeAttribute(name);
    element[domProp] = false;
  }
}

/**
 * Устанавливает стили
 */
function setStyle(element, style) {
  if (typeof style === 'string') {
    element.setAttribute('style', style);
  } else if (typeof style === 'object') {
    // Очищаем существующие стили
    element.style.cssText = '';

    // Устанавливаем новые
    Object.keys(style).forEach(key => {
      // Поддержка vendor prefixes
      if (key.startsWith('--')) {
        // CSS переменные
        element.style.setProperty(key, style[key]);
      } else {
        element.style[key] = style[key];
      }
    });
  }
}
```

## Удаление свойств

При обновлении нужно уметь **удалять** старые свойства:

```javascript
/**
 * Удаляет свойство с элемента
 */
function removeProp(element, name, value) {
  if (name === 'className') {
    name = 'class';
  }

  // События нужно удалить через removeEventListener
  if (isEventProp(name)) {
    const eventName = extractEventName(name);
    element.removeEventListener(eventName, value);
    return;
  }

  // style
  if (name === 'style') {
    element.style.cssText = '';
    return;
  }

  // DOM свойства
  if (isPropertyName(name)) {
    element[name] = getDefaultValue(element, name);
    return;
  }

  // Boolean атрибуты
  if (isBooleanAttr(name)) {
    const domProp = BOOLEAN_ATTRS.get(name) || name;
    element.removeAttribute(name);
    element[domProp] = false;
    return;
  }

  // Обычные атрибуты
  element.removeAttribute(name);
}

/**
 * Возвращает значение по умолчанию для свойства
 */
function getDefaultValue(element, name) {
  switch (name) {
    case 'value':
      return '';
    case 'checked':
    case 'selected':
      return false;
    default:
      return '';
  }
}
```

## Обновление свойств

Когда виртуальное дерево обновляется, нужно обновить изменившиеся свойства:

```javascript
/**
 * Обновляет свойства элемента
 * @param {HTMLElement} element
 * @param {Object} oldProps - Старые свойства
 * @param {Object} newProps - Новые свойства
 */
function updateProps(element, oldProps, newProps) {
  // Удаляем старые свойства, которых нет в новых
  Object.keys(oldProps).forEach(name => {
    if (!(name in newProps)) {
      removeProp(element, name, oldProps[name]);
    }
  });

  // Добавляем новые и обновляем изменившиеся
  Object.keys(newProps).forEach(name => {
    const oldValue = oldProps[name];
    const newValue = newProps[name];

    // Обновляем только если значение изменилось
    if (oldValue !== newValue) {
      // Для событий: сначала удаляем старый, потом добавляем новый
      if (isEventProp(name) && oldValue) {
        removeProp(element, name, oldValue);
      }
      setProp(element, name, newValue);
    }
  });
}
```

## Примеры использования

### Пример 1: Form элементы

```javascript
const form = h('form', { onsubmit: handleSubmit },
  h('input', {
    type: 'text',
    value: username,
    placeholder: 'Имя пользователя',
    required: true,
    oninput: (e) => updateUsername(e.target.value)
  }),
  h('input', {
    type: 'password',
    value: password,
    required: true
  }),
  h('input', {
    type: 'checkbox',
    checked: rememberMe,
    id: 'remember'
  }),
  h('label', { for: 'remember' }, 'Запомнить меня'),
  h('button', {
    type: 'submit',
    disabled: !isValid
  }, 'Войти')
);
```

### Пример 2: Стили

```javascript
const box = h('div', {
  style: {
    width: '100px',
    height: '100px',
    backgroundColor: isActive ? 'blue' : 'gray',
    transform: `rotate(${angle}deg)`,
    transition: 'all 0.3s ease',
    // CSS переменные
    '--custom-color': themeColor
  }
});
```

### Пример 3: Доступность (a11y)

```javascript
const dialog = h('div', {
  role: 'dialog',
  'aria-labelledby': 'dialog-title',
  'aria-describedby': 'dialog-desc',
  'aria-modal': 'true',
  hidden: !isOpen
},
  h('h2', { id: 'dialog-title' }, 'Заголовок'),
  h('p', { id: 'dialog-desc' }, 'Описание диалога'),
  h('button', {
    'aria-label': 'Закрыть диалог',
    onclick: handleClose
  }, '×')
);
```

### Пример 4: Data атрибуты

```javascript
const userCard = h('div', {
  class: 'user-card',
  'data-user-id': user.id,
  'data-role': user.role,
  'data-status': user.isOnline ? 'online' : 'offline'
},
  h('img', { src: user.avatar, alt: user.name }),
  h('h3', null, user.name)
);
```

## Оптимизация: мемоизация обработчиков событий

Проблема: при каждом рендере создаются новые функции:

```javascript
function Counter() {
  return h('button', {
    onclick: () => { count++ } // Новая функция каждый раз!
  }, count);
}
```

Это может вызвать проблемы при diffing (события всегда будут "разные"). Решение — позже, когда добавим хуки.

## Полная версия модуля

Обновим `nano-framework.js`:

```javascript
// ... (предыдущий код)

export function setProps(element, props) {
  Object.keys(props).forEach(key => {
    setProp(element, key, props[key]);
  });
}

function setProp(element, name, value) {
  if (name === 'className') name = 'class';

  if (isEventProp(name)) {
    element.addEventListener(extractEventName(name), value);
    return;
  }

  if (name === 'style') {
    setStyle(element, value);
    return;
  }

  if (isPropertyName(name)) {
    element[name] = value;
    return;
  }

  if (isBooleanAttr(name)) {
    setBooleanAttr(element, name, value);
    return;
  }

  if (value == null) {
    element.removeAttribute(name);
  } else {
    element.setAttribute(name, value);
  }
}

export function updateProps(element, oldProps, newProps) {
  Object.keys(oldProps).forEach(name => {
    if (!(name in newProps)) {
      removeProp(element, name, oldProps[name]);
    }
  });

  Object.keys(newProps).forEach(name => {
    if (oldProps[name] !== newProps[name]) {
      if (isEventProp(name) && oldProps[name]) {
        removeProp(element, name, oldProps[name]);
      }
      setProp(element, name, newProps[name]);
    }
  });
}

function removeProp(element, name, value) {
  if (name === 'className') name = 'class';

  if (isEventProp(name)) {
    element.removeEventListener(extractEventName(name), value);
    return;
  }

  if (name === 'style') {
    element.style.cssText = '';
    return;
  }

  if (isPropertyName(name)) {
    element[name] = '';
    return;
  }

  element.removeAttribute(name);
}

// Вспомогательные функции
function isEventProp(name) {
  return name.startsWith('on');
}

function extractEventName(name) {
  return name.substring(2).toLowerCase();
}

const DOM_PROPERTIES = new Set(['value', 'checked', 'selected']);

function isPropertyName(name) {
  return DOM_PROPERTIES.has(name);
}

const BOOLEAN_ATTRS = new Map([
  ['disabled', 'disabled'],
  ['readonly', 'readOnly'],
  ['required', 'required'],
  ['autofocus', 'autofocus'],
  ['hidden', 'hidden']
]);

function isBooleanAttr(name) {
  return BOOLEAN_ATTRS.has(name);
}

function setBooleanAttr(element, name, value) {
  const domProp = BOOLEAN_ATTRS.get(name) || name;

  if (value) {
    element.setAttribute(name, '');
    element[domProp] = true;
  } else {
    element.removeAttribute(name);
    element[domProp] = false;
  }
}

function setStyle(element, style) {
  if (typeof style === 'string') {
    element.setAttribute('style', style);
  } else if (typeof style === 'object') {
    element.style.cssText = '';
    Object.keys(style).forEach(key => {
      if (key.startsWith('--')) {
        element.style.setProperty(key, style[key]);
      } else {
        element.style[key] = style[key];
      }
    });
  }
}
```

## Задание

1. Добавьте поддержку `contentEditable`:
```javascript
h('div', { contentEditable: true }, 'Редактируемый текст')
```

2. Создайте компонент Toggle с правильной работой `checked`:
```javascript
function Toggle({ isOn, onToggle }) {
  return h('label', null,
    h('input', {
      type: 'checkbox',
      checked: isOn,
      onchange: onToggle
    }),
    ' ',
    isOn ? 'Включено' : 'Выключено'
  );
}
```

3. Реализуйте поддержку `classList` (массив классов):
```javascript
h('div', {
  classList: ['btn', isActive && 'active', 'large']
})
// → class="btn active large" или class="btn large"
```

## Ключевые выводы

- Атрибуты и свойства — это разные вещи
- Boolean атрибуты требуют специальной обработки
- События всегда через `addEventListener`
- `value`, `checked` и т.д. — это DOM свойства, а не атрибуты
- Стили можно задавать объектом (удобнее) или строкой
- При обновлении нужно удалять старые свойства
- Важно правильно обрабатывать `null`/`undefined` (удаление атрибутов)

---

[← Предыдущий урок: Первый рендеринг](./04-first-rendering.md) | [Следующий урок: Рендеринг списков и фрагментов →](./06-lists-and-fragments.md)

# Урок 14: Система событий

## Текущая реализация событий

Сейчас мы обрабатываем события просто:

```javascript
function setProp(element, name, value) {
  if (name.startsWith('on')) {
    const eventName = name.substring(2).toLowerCase();
    element.addEventListener(eventName, value);
    return;
  }
  // ...
}
```

**Проблемы:**
1. Событие не удаляется при обновлении компонента
2. Новая функция создаётся при каждом рендере
3. Нет делегирования событий
4. Нет нормализации событий между браузерами

## Проблема 1: Утечки памяти

```javascript
function Counter() {
  const [count, setCount] = useState(0);

  return h('button', {
    onclick: () => setCount(count + 1)  // Новая функция каждый раз!
  }, count);
}

// При каждом рендере:
// 1. Создаётся новая функция
// 2. Старая остаётся в памяти (утечка!)
```

### Решение: удаление старых событий

Обновим `updateProps`:

```javascript
function updateProps(element, oldProps, newProps) {
  // Удаляем старые события
  Object.keys(oldProps).forEach(name => {
    if (isEventProp(name) && oldProps[name] !== newProps[name]) {
      const eventName = extractEventName(name);
      element.removeEventListener(eventName, oldProps[name]);
    }
  });

  // Добавляем новые
  Object.keys(newProps).forEach(name => {
    if (isEventProp(name) && oldProps[name] !== newProps[name]) {
      const eventName = extractEventName(name);
      element.addEventListener(eventName, newProps[name]);
    }
  });

  // ... обновление остальных props
}
```

## Проблема 2: Создание новых функций

Каждый рендер создаёт новые функции обработчиков:

```javascript
// При каждом рендере count меняется → новая функция
h('button', { onclick: () => setCount(count + 1) }, count)
```

### Решение 1: Функциональное обновление

```javascript
// Функция не зависит от count → можно мемоизировать
const increment = () => setCount(c => c + 1);

h('button', { onclick: increment }, count)
```

### Решение 2: useCallback (опционально)

```javascript
function useCallback(callback, deps) {
  const [memoized, setMemoized] = useState({ callback, deps });

  if (!deps || deps.some((dep, i) => dep !== memoized.deps[i])) {
    setMemoized({ callback, deps });
  }

  return memoized.callback;
}

// Использование
function Counter() {
  const [count, setCount] = useState(0);

  const increment = useCallback(() => {
    setCount(c => c + 1);
  }, []); // Пустой массив — функция создаётся один раз

  return h('button', { onclick: increment }, count);
}
```

## Делегирование событий

**Идея:** вместо добавления обработчика на каждый элемент, добавить один обработчик на корень.

### Преимущества:
1. Меньше памяти
2. Быстрее добавление/удаление элементов
3. Работает с динамическими элементами

### Реализация

```javascript
/**
 * Глобальный обработчик событий
 */
const eventDelegation = new Map();

/**
 * Инициализация делегирования для корневого элемента
 */
export function initEventDelegation(root) {
  const eventTypes = [
    'click', 'input', 'change', 'submit',
    'keydown', 'keyup', 'focus', 'blur'
  ];

  eventTypes.forEach(eventType => {
    root.addEventListener(eventType, (e) => {
      handleDelegatedEvent(e);
    }, true); // true = capture phase
  });
}

/**
 * Обработка делегированного события
 */
function handleDelegatedEvent(nativeEvent) {
  let target = nativeEvent.target;

  // Поднимаемся по дереву до корня
  while (target) {
    const handlers = eventDelegation.get(target);

    if (handlers) {
      const handler = handlers[nativeEvent.type];
      if (handler) {
        handler(nativeEvent);

        // Если вызвали stopPropagation, прекращаем
        if (nativeEvent.cancelBubble) {
          break;
        }
      }
    }

    target = target.parentNode;
  }
}

/**
 * Регистрация обработчика события
 */
export function registerEvent(element, eventType, handler) {
  if (!eventDelegation.has(element)) {
    eventDelegation.set(element, {});
  }

  eventDelegation.get(element)[eventType] = handler;
}

/**
 * Удаление обработчика события
 */
export function unregisterEvent(element, eventType) {
  const handlers = eventDelegation.get(element);
  if (handlers) {
    delete handlers[eventType];

    if (Object.keys(handlers).length === 0) {
      eventDelegation.delete(element);
    }
  }
}
```

### Обновляем setProp для использования делегирования

```javascript
function setProp(element, name, value) {
  if (name === 'className') name = 'class';

  if (isEventProp(name)) {
    const eventType = extractEventName(name);
    registerEvent(element, eventType, value);
    return;
  }

  // ... остальные props
}

function removeProp(element, name, value) {
  if (isEventProp(name)) {
    const eventType = extractEventName(name);
    unregisterEvent(element, eventType);
    return;
  }

  // ...
}
```

## Нормализация событий (Synthetic Events)

React создаёт обёртку над нативными событиями для кроссбраузерности.

```javascript
/**
 * Синтетическое событие
 */
class SyntheticEvent {
  constructor(nativeEvent) {
    this.nativeEvent = nativeEvent;
    this.type = nativeEvent.type;
    this.target = nativeEvent.target;
    this.currentTarget = nativeEvent.currentTarget;

    // Копируем важные свойства
    this.key = nativeEvent.key;
    this.keyCode = nativeEvent.keyCode;
    this.which = nativeEvent.which;

    // Для форм
    this.value = nativeEvent.target?.value;
    this.checked = nativeEvent.target?.checked;

    this._prevented = false;
    this._stopped = false;
  }

  preventDefault() {
    this._prevented = true;
    this.nativeEvent.preventDefault();
  }

  stopPropagation() {
    this._stopped = true;
    this.nativeEvent.stopPropagation();
  }

  get defaultPrevented() {
    return this._prevented;
  }
}

/**
 * Создаёт синтетическое событие
 */
function createSyntheticEvent(nativeEvent) {
  return new SyntheticEvent(nativeEvent);
}
```

### Использование в делегировании

```javascript
function handleDelegatedEvent(nativeEvent) {
  // Создаём синтетическое событие
  const syntheticEvent = createSyntheticEvent(nativeEvent);

  let target = nativeEvent.target;

  while (target) {
    const handlers = eventDelegation.get(target);

    if (handlers) {
      const handler = handlers[nativeEvent.type];
      if (handler) {
        syntheticEvent.currentTarget = target;
        handler(syntheticEvent); // Передаём синтетическое событие

        if (syntheticEvent._stopped) {
          break;
        }
      }
    }

    target = target.parentNode;
  }
}
```

## Специальные события

### onChange для input

React нормализует `onChange` — вызывает при каждом изменении:

```javascript
function normalizeOnChange(element, handler) {
  const inputHandler = (e) => {
    handler(createSyntheticEvent(e));
  };

  if (element.type === 'checkbox' || element.type === 'radio') {
    element.addEventListener('change', inputHandler);
  } else {
    element.addEventListener('input', inputHandler);
  }

  return inputHandler;
}
```

### onSubmit с preventDefault

```javascript
function Form() {
  const handleSubmit = (e) => {
    e.preventDefault(); // Предотвращаем перезагрузку страницы
    console.log('Submit');
  };

  return h('form', { onsubmit: handleSubmit },
    h('input', { type: 'text' }),
    h('button', { type: 'submit' }, 'Отправить')
  );
}
```

## Полная реализация системы событий

```javascript
// event-system.js

/**
 * Хранилище обработчиков событий: element → { eventType: handler }
 */
const eventHandlers = new WeakMap();

/**
 * Типы событий для делегирования
 */
const delegatedEvents = new Set([
  'click', 'dblclick',
  'input', 'change',
  'submit', 'reset',
  'keydown', 'keypress', 'keyup',
  'mousedown', 'mouseup', 'mousemove',
  'focus', 'blur'
]);

/**
 * Корневой элемент с делегированием
 */
let delegationRoot = null;

/**
 * Инициализация системы событий
 */
export function initEventSystem(root) {
  if (delegationRoot) return;
  delegationRoot = root;

  delegatedEvents.forEach(eventType => {
    root.addEventListener(eventType, handleEvent, true);
  });
}

/**
 * Обработчик делегированного события
 */
function handleEvent(nativeEvent) {
  const syntheticEvent = new SyntheticEvent(nativeEvent);
  let target = nativeEvent.target;

  // Всплытие события
  while (target && target !== delegationRoot) {
    const handlers = eventHandlers.get(target);

    if (handlers && handlers[nativeEvent.type]) {
      syntheticEvent.currentTarget = target;
      handlers[nativeEvent.type](syntheticEvent);

      if (syntheticEvent._stopped) {
        break;
      }
    }

    target = target.parentNode;
  }
}

/**
 * Добавление обработчика события
 */
export function addEventListener(element, eventType, handler) {
  if (!eventHandlers.has(element)) {
    eventHandlers.set(element, {});
  }

  eventHandlers.get(element)[eventType] = handler;
}

/**
 * Удаление обработчика события
 */
export function removeEventListener(element, eventType) {
  const handlers = eventHandlers.get(element);
  if (handlers) {
    delete handlers[eventType];

    if (Object.keys(handlers).length === 0) {
      eventHandlers.delete(element);
    }
  }
}

/**
 * Синтетическое событие
 */
class SyntheticEvent {
  constructor(nativeEvent) {
    this.nativeEvent = nativeEvent;
    this.type = nativeEvent.type;
    this.target = nativeEvent.target;
    this.currentTarget = nativeEvent.currentTarget;

    // Копируем свойства
    Object.keys(nativeEvent).forEach(key => {
      if (typeof nativeEvent[key] !== 'function' && !(key in this)) {
        this[key] = nativeEvent[key];
      }
    });

    this._prevented = false;
    this._stopped = false;
  }

  preventDefault() {
    this._prevented = true;
    this.nativeEvent.preventDefault();
  }

  stopPropagation() {
    this._stopped = true;
    this.nativeEvent.stopPropagation();
  }

  get defaultPrevented() {
    return this._prevented;
  }

  persist() {
    // В React нужно для асинхронного доступа
    // У нас не реализуем pooling, поэтому просто заглушка
  }
}
```

## Пример использования

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Event System Demo</title>
  <style>
    .box {
      padding: 20px;
      margin: 10px;
      border: 2px solid #333;
    }
    .outer { background: #ffcccc; }
    .inner { background: #ccccff; }
  </style>
</head>
<body>
  <div id="app"></div>

  <script type="module">
    import { h, render, useState, initEventSystem } from './nano-framework.js';

    function EventDemo() {
      const [log, setLog] = useState([]);

      function addLog(message) {
        setLog([...log, `${new Date().toLocaleTimeString()}: ${message}`]);
      }

      function handleOuterClick(e) {
        addLog('Outer clicked');
        // e.stopPropagation(); // Раскомментируйте для остановки
      }

      function handleInnerClick(e) {
        addLog('Inner clicked');
        e.stopPropagation();
      }

      function handleSubmit(e) {
        e.preventDefault();
        addLog('Form submitted (prevented)');
      }

      function handleKeyDown(e) {
        if (e.key === 'Enter') {
          addLog(`Enter pressed: ${e.target.value}`);
        }
      }

      return h('div', null,
        h('h1', null, 'Демонстрация системы событий'),

        h('div', {
          class: 'box outer',
          onclick: handleOuterClick
        },
          'Outer (клик)',
          h('div', {
            class: 'box inner',
            onclick: handleInnerClick
          }, 'Inner (клик + stopPropagation)')
        ),

        h('form', { onsubmit: handleSubmit },
          h('input', {
            type: 'text',
            placeholder: 'Нажмите Enter',
            onkeydown: handleKeyDown
          }),
          h('button', { type: 'submit' }, 'Submit')
        ),

        h('button', {
          onclick: () => setLog([])
        }, 'Очистить лог'),

        h('div', null,
          h('h3', null, 'Лог событий:'),
          h('ul', null,
            log.map((msg, i) =>
              h('li', { key: i }, msg)
            )
          )
        )
      );
    }

    const root = document.getElementById('app');
    initEventSystem(root);
    render(h(EventDemo), root);
  </script>
</body>
</html>
```

## Оптимизации

### Event Pooling (React < 17)

React переиспользовал объекты событий для производительности:

```javascript
class EventPool {
  constructor(EventClass, maxSize = 10) {
    this.EventClass = EventClass;
    this.maxSize = maxSize;
    this.pool = [];
  }

  getEvent(nativeEvent) {
    if (this.pool.length > 0) {
      const event = this.pool.pop();
      event.reset(nativeEvent);
      return event;
    }
    return new this.EventClass(nativeEvent);
  }

  releaseEvent(event) {
    if (this.pool.length < this.maxSize) {
      event.reset(null);
      this.pool.push(event);
    }
  }
}
```

React 17+ убрал pooling, так как современные браузеры достаточно быстрые.

## Задание

1. Добавьте поддержку пользовательских событий:
```javascript
function CustomEvent() {
  return h('button', {
    onclick: () => {
      element.dispatchEvent(new CustomEvent('mycustom', { detail: 123 }));
    }
  });
}
```

2. Реализуйте debounce для событий input:
```javascript
function useDebounceEvent(handler, delay) {
  // Вернуть обработчик с debounce
}
```

3. Создайте компонент с drag & drop:
```javascript
function DraggableBox() {
  // ondragstart, ondrag, ondrop
}
```

## Ключевые выводы

- Система событий должна удалять старые обработчики
- Делегирование событий повышает производительность
- Синтетические события нормализуют различия браузеров
- preventDefault() и stopPropagation() работают как обычно
- useCallback мемоизирует функции-обработчики
- Современные фреймворки не используют event pooling
- Делегирование на корне вместо каждого элемента

---

[← Предыдущий урок: Контекст](./13-context.md) | [Следующий урок: Ссылки (refs) →](./15-refs.md)

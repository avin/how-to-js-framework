# Урок 15: Ссылки (Refs)

## Проблема: доступ к DOM элементам

Фреймворк абстрагирует работу с DOM, но иногда нужен прямой доступ:

```javascript
function AutoFocusInput() {
  // ❌ Как получить доступ к DOM элементу?
  // Нужно вызвать input.focus()

  return h('input', { type: 'text' });
}
```

**Случаи использования:**
- Управление фокусом
- Измерение размеров элемента
- Анимации
- Интеграция со сторонними библиотеками
- Прокрутка к элементу

## Решение: useRef

**Ref (reference)** — это объект-контейнер, который хранит изменяемое значение:

```javascript
function AutoFocusInput() {
  const inputRef = useRef(null);

  useEffect(() => {
    inputRef.current.focus();
  }, []);

  return h('input', {
    ref: inputRef,
    type: 'text'
  });
}
```

## Реализация useRef

```javascript
/**
 * Хук для создания мутабельной ссылки
 * @param {any} initialValue - Начальное значение
 * @returns {Object} { current: value }
 */
export function useRef(initialValue) {
  if (!currentInstance) {
    throw new Error('useRef can only be called inside a component');
  }

  if (!componentStates.has(currentInstance)) {
    componentStates.set(currentInstance, []);
  }

  const hooks = componentStates.get(currentInstance);
  const hookIndex = currentHookIndex;

  // Создаём ref только один раз
  if (!hooks[hookIndex]) {
    hooks[hookIndex] = {
      current: initialValue
    };
  }

  currentHookIndex++;

  return hooks[hookIndex];
}
```

**Важно:** изменение `ref.current` НЕ вызывает перерисовку!

## Обработка атрибута ref

Обновим `setProp` для работы с ref:

```javascript
function setProp(element, name, value) {
  // ... другие проверки

  // Обработка ref
  if (name === 'ref') {
    if (value && typeof value === 'object' && 'current' in value) {
      value.current = element;
    } else if (typeof value === 'function') {
      value(element); // Callback ref
    }
    return;
  }

  // ... остальной код
}
```

При удалении элемента обнуляем ref:

```javascript
function removeProp(element, name, value) {
  if (name === 'ref') {
    if (value && typeof value === 'object' && 'current' in value) {
      value.current = null;
    } else if (typeof value === 'function') {
      value(null);
    }
    return;
  }

  // ...
}
```

## Примеры использования

### Пример 1: Фокус на input

```javascript
function SearchBar() {
  const inputRef = useRef(null);

  useEffect(() => {
    // Фокусируемся при монтировании
    inputRef.current.focus();
  }, []);

  return h('input', {
    ref: inputRef,
    type: 'text',
    placeholder: 'Поиск...'
  });
}
```

### Пример 2: Измерение размеров

```javascript
function ResizableBox() {
  const boxRef = useRef(null);
  const [size, setSize] = useState({ width: 0, height: 0 });

  useEffect(() => {
    const updateSize = () => {
      if (boxRef.current) {
        const rect = boxRef.current.getBoundingClientRect();
        setSize({ width: rect.width, height: rect.height });
      }
    };

    updateSize();
    window.addEventListener('resize', updateSize);

    return () => window.removeEventListener('resize', updateSize);
  }, []);

  return h('div', null,
    h('div', {
      ref: boxRef,
      style: { padding: '20px', background: '#f0f0f0' }
    }, 'Измеряемый блок'),
    h('p', null, `Размер: ${size.width}x${size.height}`)
  );
}
```

### Пример 3: Прокрутка к элементу

```javascript
function ScrollToSection() {
  const sectionRef = useRef(null);

  function scrollToSection() {
    sectionRef.current.scrollIntoView({
      behavior: 'smooth',
      block: 'start'
    });
  }

  return h('div', null,
    h('button', { onclick: scrollToSection }, 'Прокрутить вниз'),

    // Много контента...
    Array(50).fill(null).map((_, i) =>
      h('p', { key: i }, `Параграф ${i + 1}`)
    ),

    h('div', {
      ref: sectionRef,
      style: { padding: '20px', background: 'yellow' }
    }, 'Целевая секция')
  );
}
```

### Пример 4: Canvas рисование

```javascript
function Canvas() {
  const canvasRef = useRef(null);

  useEffect(() => {
    const canvas = canvasRef.current;
    const ctx = canvas.getContext('2d');

    // Рисуем
    ctx.fillStyle = 'blue';
    ctx.fillRect(10, 10, 150, 100);

    ctx.fillStyle = 'red';
    ctx.beginPath();
    ctx.arc(200, 75, 50, 0, Math.PI * 2);
    ctx.fill();
  }, []);

  return h('canvas', {
    ref: canvasRef,
    width: 400,
    height: 200,
    style: { border: '1px solid black' }
  });
}
```

### Пример 5: Видео плеер

```javascript
function VideoPlayer({ src }) {
  const videoRef = useRef(null);
  const [playing, setPlaying] = useState(false);

  function togglePlay() {
    if (playing) {
      videoRef.current.pause();
    } else {
      videoRef.current.play();
    }
    setPlaying(!playing);
  }

  return h('div', null,
    h('video', {
      ref: videoRef,
      src,
      width: 640,
      height: 360
    }),
    h('button', { onclick: togglePlay },
      playing ? 'Пауза' : 'Воспроизвести'
    )
  );
}
```

## Callback Refs

Альтернатива useRef — функция обратного вызова:

```javascript
function CallbackRefExample() {
  const [height, setHeight] = useState(0);

  // Функция вызывается с элементом при монтировании
  function measureRef(element) {
    if (element) {
      setHeight(element.getBoundingClientRect().height);
    }
  }

  return h('div', null,
    h('div', {
      ref: measureRef,
      style: { padding: '20px' }
    }, 'Измеряемый блок'),
    h('p', null, `Высота: ${height}px`)
  );
}
```

**Когда использовать:**
- Нужно выполнить код сразу при получении элемента
- Динамические refs (в циклах)
- Несколько операций с элементом

## Refs в циклах

```javascript
function ItemList({ items }) {
  const itemRefs = useRef([]);

  function scrollToItem(index) {
    itemRefs.current[index]?.scrollIntoView({
      behavior: 'smooth',
      block: 'nearest'
    });
  }

  return h('div', null,
    h('div', null,
      items.map((item, i) =>
        h('button', {
          key: item.id,
          onclick: () => scrollToItem(i)
        }, `К пункту ${i + 1}`)
      )
    ),

    h('div', { style: { height: '200px', overflow: 'auto' } },
      items.map((item, i) =>
        h('div', {
          key: item.id,
          ref: (el) => { itemRefs.current[i] = el; },
          style: { padding: '20px', margin: '10px', background: '#f0f0f0' }
        }, `Пункт ${i + 1}: ${item.text}`)
      )
    )
  );
}
```

## useRef для хранения значений

useRef можно использовать для хранения любых значений, которые не должны вызывать перерисовку:

### Пример: Предыдущее значение

```javascript
function usePrevious(value) {
  const ref = useRef();

  useEffect(() => {
    ref.current = value;
  });

  return ref.current;
}

function Counter() {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);

  return h('div', null,
    h('p', null, `Текущее: ${count}`),
    h('p', null, `Предыдущее: ${prevCount ?? 'нет'}`),
    h('button', { onclick: () => setCount(count + 1) }, '+')
  );
}
```

### Пример: Таймер

```javascript
function Timer() {
  const [seconds, setSeconds] = useState(0);
  const [running, setRunning] = useState(false);
  const intervalRef = useRef(null);

  useEffect(() => {
    if (running) {
      intervalRef.current = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
        intervalRef.current = null;
      }
    }

    return () => {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    };
  }, [running]);

  return h('div', null,
    h('h2', null, `${seconds}с`),
    h('button', {
      onclick: () => setRunning(!running)
    }, running ? 'Стоп' : 'Старт'),
    h('button', {
      onclick: () => {
        setSeconds(0);
        setRunning(false);
      }
    }, 'Сброс')
  );
}
```

### Пример: Счётчик рендеров

```javascript
function useRenderCount() {
  const renderCount = useRef(0);

  useEffect(() => {
    renderCount.current += 1;
  });

  return renderCount.current;
}

function Component() {
  const [count, setCount] = useState(0);
  const renderCount = useRenderCount();

  return h('div', null,
    h('p', null, `Count: ${count}`),
    h('p', null, `Renders: ${renderCount}`),
    h('button', { onclick: () => setCount(count + 1) }, '+')
  );
}
```

## forwardRef (передача ref в компонент)

Иногда нужно передать ref через компонент:

```javascript
/**
 * Создаёт компонент, который может принимать ref
 */
export function forwardRef(component) {
  return function ForwardRefWrapper(props) {
    return component(props, props.ref);
  };
}

// Использование
const FancyInput = forwardRef((props, ref) => {
  return h('input', {
    ...props,
    ref,
    class: 'fancy-input'
  });
});

function Parent() {
  const inputRef = useRef(null);

  return h('div', null,
    h(FancyInput, { ref: inputRef, type: 'text' }),
    h('button', {
      onclick: () => inputRef.current.focus()
    }, 'Фокус')
  );
}
```

## useImperativeHandle

Кастомизация значения ref:

```javascript
export function useImperativeHandle(ref, createHandle, deps) {
  useEffect(() => {
    if (ref) {
      const handle = createHandle();

      if (typeof ref === 'function') {
        ref(handle);
      } else if (ref && 'current' in ref) {
        ref.current = handle;
      }

      return () => {
        if (typeof ref === 'function') {
          ref(null);
        } else if (ref && 'current' in ref) {
          ref.current = null;
        }
      };
    }
  }, deps);
}

// Использование
const FancyInput = forwardRef((props, ref) => {
  const inputRef = useRef(null);

  useImperativeHandle(ref, () => ({
    focus: () => {
      inputRef.current.focus();
    },
    clear: () => {
      inputRef.current.value = '';
    }
  }), []);

  return h('input', { ...props, ref: inputRef });
});

function Parent() {
  const inputRef = useRef(null);

  return h('div', null,
    h(FancyInput, { ref: inputRef }),
    h('button', { onclick: () => inputRef.current.focus() }, 'Фокус'),
    h('button', { onclick: () => inputRef.current.clear() }, 'Очистить')
  );
}
```

## Полный пример

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Refs Demo</title>
  <style>
    body {
      font-family: sans-serif;
      max-width: 600px;
      margin: 40px auto;
    }
    .panel {
      padding: 20px;
      margin: 10px 0;
      border: 2px solid #333;
      border-radius: 8px;
    }
    .highlight {
      background: yellow;
      transition: background 0.5s;
    }
  </style>
</head>
<body>
  <div id="app"></div>

  <script type="module">
    import { h, render, useState, useRef, useEffect } from './nano-framework.js';

    function RefDemo() {
      const inputRef = useRef(null);
      const panel1Ref = useRef(null);
      const panel2Ref = useRef(null);
      const [highlighted, setHighlighted] = useState(null);

      function focusInput() {
        inputRef.current.focus();
      }

      function scrollToPanel(ref, name) {
        ref.current.scrollIntoView({ behavior: 'smooth' });
        setHighlighted(name);
        setTimeout(() => setHighlighted(null), 500);
      }

      return h('div', null,
        h('h1', null, 'Демонстрация Refs'),

        h('div', { class: 'panel' },
          h('h3', null, 'Управление фокусом'),
          h('input', {
            ref: inputRef,
            type: 'text',
            placeholder: 'Кликните кнопку для фокуса'
          }),
          h('button', { onclick: focusInput }, 'Фокус на input')
        ),

        h('div', { class: 'panel' },
          h('h3', null, 'Прокрутка к элементам'),
          h('button', {
            onclick: () => scrollToPanel(panel1Ref, 'panel1')
          }, 'К панели 1'),
          ' ',
          h('button', {
            onclick: () => scrollToPanel(panel2Ref, 'panel2')
          }, 'К панели 2')
        ),

        // Много контента для прокрутки
        Array(10).fill(null).map((_, i) =>
          h('p', { key: i }, `Параграф ${i + 1}`)
        ),

        h('div', {
          ref: panel1Ref,
          class: `panel ${highlighted === 'panel1' ? 'highlight' : ''}`
        },
          h('h3', null, 'Панель 1'),
          h('p', null, 'Эта панель подсвечивается при прокрутке')
        ),

        Array(10).fill(null).map((_, i) =>
          h('p', { key: `mid-${i}` }, `Параграф ${i + 11}`)
        ),

        h('div', {
          ref: panel2Ref,
          class: `panel ${highlighted === 'panel2' ? 'highlight' : ''}`
        },
          h('h3', null, 'Панель 2'),
          h('p', null, 'И эта тоже')
        )
      );
    }

    render(h(RefDemo), document.getElementById('app'));
  </script>
</body>
</html>
```

## Правила использования refs

### 1. Не злоупотребляйте

```javascript
// ❌ Плохо — для этого есть state
function Counter() {
  const countRef = useRef(0);
  return h('button', {
    onclick: () => {
      countRef.current++;
      // Не перерисуется!
    }
  }, countRef.current);
}

// ✓ Хорошо
function Counter() {
  const [count, setCount] = useState(0);
  return h('button', {
    onclick: () => setCount(count + 1)
  }, count);
}
```

### 2. Не читайте/пишите refs во время рендера

```javascript
// ❌ Плохо
function Component() {
  const ref = useRef(0);
  ref.current += 1; // Не делайте так!

  return h('div', null, ref.current);
}

// ✓ Хорошо — в useEffect
function Component() {
  const ref = useRef(0);

  useEffect(() => {
    ref.current += 1;
  });

  return h('div', null, 'Rendered');
}
```

## Задание

1. Создайте компонент с кастомным контекстным меню:
```javascript
// Показывайте меню по правому клику, используйте refs для позиционирования
```

2. Реализуйте infinite scroll:
```javascript
// Используйте ref для отслеживания позиции прокрутки
```

3. Создайте компонент для измерения производительности:
```javascript
function usePerformance() {
  // Измеряйте время между рендерами используя refs
}
```

## Ключевые выводы

- useRef создаёт мутабельный контейнер
- ref.current можно изменять без перерисовки
- Используйте refs для доступа к DOM элементам
- Callback refs для динамических случаев
- forwardRef передаёт ref через компонент
- useImperativeHandle кастомизирует значение ref
- Не злоупотребляйте refs — предпочитайте декларативный подход
- refs полезны для фокуса, анимаций, измерений, интеграций

---

[← Предыдущий урок: Система событий](./14-event-system.md) | [Следующий урок: Reconciliation →](./16-reconciliation.md)

# Урок 11: Состояние (useState)

## Проблема: управление состоянием компонента

Сейчас компоненты **не имеют собственного состояния**. Все данные глобальные:

```javascript
// Глобальные переменные - плохо!
let count = 0;

function Counter() {
  return h('div', null,
    h('p', null, count),
    h('button', {
      onclick: () => {
        count++;
        rerender(); // Перерисовываем всё приложение!
      }
    }, '+')
  );
}
```

**Проблемы:**
1. Глобальные переменные загрязняют пространство имён
2. Невозможно использовать несколько экземпляров компонента
3. Нужно вручную вызывать `rerender()`
4. Перерисовывается всё приложение, а не только компонент

## Решение: useState

**useState** — это функция (хук), которая:
1. Создаёт переменную состояния
2. Привязывает её к компоненту
3. Автоматически перерисовывает компонент при изменении

```javascript
function Counter() {
  const [count, setCount] = useState(0);

  return h('div', null,
    h('p', null, count),
    h('button', { onclick: () => setCount(count + 1) }, '+')
  );
}
```

## Концепция хуков

**Хук (hook)** — это специальная функция, которая "подключается" к механизму фреймворка.

Правила хуков:
1. Вызываются только внутри компонентов
2. Вызываются только на верхнем уровне (не в циклах/условиях)
3. Вызываются в одном и том же порядке при каждом рендере

## Реализация useState

### Хранилище состояния

Нам нужно хранить состояние каждого компонента:

```javascript
/**
 * Глобальное хранилище состояния компонентов
 */
const componentStates = new Map();

/**
 * Текущий рендерящийся компонент
 */
let currentComponent = null;

/**
 * Индекс текущего хука
 */
let currentHookIndex = 0;
```

### Функция useState

```javascript
/**
 * Хук для управления состоянием
 * @param {any} initialValue - Начальное значение
 * @returns {[any, Function]} Кортеж [значение, setter]
 */
export function useState(initialValue) {
  if (!currentComponent) {
    throw new Error('useState can only be called inside a component');
  }

  // Получаем или создаём массив хуков для компонента
  if (!componentStates.has(currentComponent)) {
    componentStates.set(currentComponent, []);
  }

  const hooks = componentStates.get(currentComponent);
  const hookIndex = currentHookIndex;

  // Инициализация хука при первом вызове
  if (hooks[hookIndex] === undefined) {
    hooks[hookIndex] = {
      value: typeof initialValue === 'function'
        ? initialValue()  // Ленивая инициализация
        : initialValue
    };
  }

  const hook = hooks[hookIndex];

  // Функция обновления состояния
  const setState = (newValue) => {
    // Поддержка функциональных обновлений
    const nextValue = typeof newValue === 'function'
      ? newValue(hook.value)
      : newValue;

    // Обновляем только если значение изменилось
    if (nextValue !== hook.value) {
      hook.value = nextValue;
      // Планируем перерисовку компонента
      scheduleRerender(currentComponent);
    }
  };

  // Увеличиваем индекс для следующего хука
  currentHookIndex++;

  return [hook.value, setState];
}
```

### Вызов компонента с контекстом

Обновим функцию `h()` для установки контекста:

```javascript
export function h(type, props, ...children) {
  // Если type — функция (компонент)
  if (typeof type === 'function') {
    // Сохраняем текущий контекст
    const prevComponent = currentComponent;
    const prevHookIndex = currentHookIndex;

    // Устанавливаем новый контекст
    currentComponent = type; // Используем саму функцию как ID
    currentHookIndex = 0;

    try {
      // Вызываем компонент
      const componentProps = { ...props, children };
      const vNode = type(componentProps);

      return vNode;
    } finally {
      // Восстанавливаем предыдущий контекст
      currentComponent = prevComponent;
      currentHookIndex = prevHookIndex;
    }
  }

  // Обычный элемент
  return {
    type,
    props: props || {},
    children: flattenChildren(children),
    key: props && props.key
  };
}
```

### Планирование перерисовки

```javascript
/**
 * Множество компонентов для перерисовки
 */
const componentsToRerender = new Set();
let isRerenderScheduled = false;

/**
 * Планирует перерисовку компонента
 */
function scheduleRerender(component) {
  componentsToRerender.add(component);

  if (!isRerenderScheduled) {
    isRerenderScheduled = true;
    // Используем микрозадачу для батчинга
    Promise.resolve().then(() => {
      isRerenderScheduled = false;
      rerenderComponents();
    });
  }
}

/**
 * Перерисовывает все запланированные компоненты
 */
function rerenderComponents() {
  const components = Array.from(componentsToRerender);
  componentsToRerender.clear();

  // Перерисовываем корневой компонент
  // (в реальности нужно хранить дерево компонентов)
  if (rootRenderFunction) {
    rootRenderFunction();
  }
}
```

## Примеры использования useState

### Простой счётчик

```javascript
function Counter() {
  const [count, setCount] = useState(0);

  return h('div', null,
    h('h2', null, 'Счётчик'),
    h('p', null, `Значение: ${count}`),
    h('button', { onclick: () => setCount(count + 1) }, '+'),
    h('button', { onclick: () => setCount(count - 1) }, '-'),
    h('button', { onclick: () => setCount(0) }, 'Сброс')
  );
}
```

### Несколько состояний

```javascript
function Form() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [subscribed, setSubscribed] = useState(false);

  return h('form', null,
    h('input', {
      value: name,
      placeholder: 'Имя',
      oninput: (e) => setName(e.target.value)
    }),
    h('input', {
      value: email,
      placeholder: 'Email',
      oninput: (e) => setEmail(e.target.value)
    }),
    h('label', null,
      h('input', {
        type: 'checkbox',
        checked: subscribed,
        onchange: (e) => setSubscribed(e.target.checked)
      }),
      ' Подписаться на рассылку'
    ),
    h('p', null, `Привет, ${name}! Email: ${email}`)
  );
}
```

### Объект как состояние

```javascript
function UserProfile() {
  const [user, setUser] = useState({
    name: 'Иван',
    age: 25,
    city: 'Москва'
  });

  function updateName(name) {
    // Важно: создаём новый объект!
    setUser({ ...user, name });
  }

  function updateAge(age) {
    setUser({ ...user, age: parseInt(age) });
  }

  return h('div', null,
    h('input', {
      value: user.name,
      oninput: (e) => updateName(e.target.value)
    }),
    h('input', {
      type: 'number',
      value: user.age,
      oninput: (e) => updateAge(e.target.value)
    }),
    h('p', null, `${user.name}, ${user.age} лет, ${user.city}`)
  );
}
```

### Массив как состояние

```javascript
function TodoList() {
  const [todos, setTodos] = useState([]);
  const [input, setInput] = useState('');

  function addTodo() {
    if (input.trim()) {
      setTodos([...todos, {
        id: Date.now(),
        text: input,
        done: false
      }]);
      setInput('');
    }
  }

  function toggleTodo(id) {
    setTodos(todos.map(todo =>
      todo.id === id ? { ...todo, done: !todo.done } : todo
    ));
  }

  function deleteTodo(id) {
    setTodos(todos.filter(todo => todo.id !== id));
  }

  return h('div', null,
    h('input', {
      value: input,
      oninput: (e) => setInput(e.target.value)
    }),
    h('button', { onclick: addTodo }, 'Добавить'),
    h('ul', null,
      todos.map(todo =>
        h('li', { key: todo.id },
          h('input', {
            type: 'checkbox',
            checked: todo.done,
            onchange: () => toggleTodo(todo.id)
          }),
          ' ',
          todo.text,
          ' ',
          h('button', { onclick: () => deleteTodo(todo.id) }, 'Удалить')
        )
      )
    )
  );
}
```

## Функциональные обновления

Когда новое состояние зависит от предыдущего:

```javascript
function Counter() {
  const [count, setCount] = useState(0);

  function increment() {
    // ❌ Проблема: если вызвать дважды быстро, работает неправильно
    setCount(count + 1);
    setCount(count + 1); // count всё ещё старое значение!
  }

  function incrementCorrect() {
    // ✓ Правильно: функциональное обновление
    setCount(c => c + 1);
    setCount(c => c + 1); // Работает правильно!
  }

  return h('div', null,
    h('p', null, count),
    h('button', { onclick: incrementCorrect }, '+2')
  );
}
```

## Ленивая инициализация

Для дорогих вычислений начального значения:

```javascript
function ExpensiveComponent() {
  // ❌ Плохо: вычисляется при каждом рендере
  const [data, setData] = useState(expensiveCalculation());

  // ✓ Хорошо: вычисляется только один раз
  const [data, setData] = useState(() => expensiveCalculation());

  return h('div', null, data);
}

function expensiveCalculation() {
  console.log('Дорогое вычисление...');
  let result = 0;
  for (let i = 0; i < 1000000; i++) {
    result += i;
  }
  return result;
}
```

## Батчинг обновлений

```javascript
function MultiUpdate() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  function handleClick() {
    // Оба обновления батчатся — один рендер!
    setCount(c => c + 1);
    setFlag(f => !f);

    console.log('Обновления запланированы');
    // Рендер произойдёт асинхронно
  }

  console.log('Рендер');

  return h('div', null,
    h('p', null, `Count: ${count}, Flag: ${flag}`),
    h('button', { onclick: handleClick }, 'Обновить')
  );
}
```

## Полный пример приложения

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>useState Demo</title>
  <style>
    body {
      font-family: sans-serif;
      max-width: 600px;
      margin: 40px auto;
    }
    .counter {
      padding: 20px;
      border: 2px solid #333;
      border-radius: 8px;
      margin: 20px 0;
    }
    .count {
      font-size: 48px;
      font-weight: bold;
      text-align: center;
      color: #0066cc;
    }
    button {
      padding: 10px 20px;
      margin: 5px;
      font-size: 16px;
      cursor: pointer;
    }
  </style>
</head>
<body>
  <div id="app"></div>

  <script type="module">
    import { h, render, useState } from './nano-framework.js';

    function Counter({ title }) {
      const [count, setCount] = useState(0);
      const [step, setStep] = useState(1);

      console.log(`Рендер Counter "${title}": count=${count}, step=${step}`);

      return h('div', { class: 'counter' },
        h('h2', null, title),
        h('div', { class: 'count' }, count),
        h('div', null,
          h('button', {
            onclick: () => setCount(c => c - step)
          }, `-${step}`),
          h('button', {
            onclick: () => setCount(c => c + step)
          }, `+${step}`),
          h('button', {
            onclick: () => setCount(0)
          }, 'Сброс')
        ),
        h('div', null,
          h('label', null, 'Шаг: '),
          h('input', {
            type: 'number',
            value: step,
            oninput: (e) => setStep(parseInt(e.target.value) || 1)
          })
        )
      );
    }

    function App() {
      const [showSecond, setShowSecond] = useState(false);

      return h('div', null,
        h('h1', null, 'Демонстрация useState'),
        h(Counter, { title: 'Счётчик 1' }),
        h('button', {
          onclick: () => setShowSecond(!showSecond)
        }, showSecond ? 'Скрыть второй счётчик' : 'Показать второй счётчик'),
        showSecond && h(Counter, { title: 'Счётчик 2' })
      );
    }

    // Инициализация
    render(h(App), document.getElementById('app'));
  </script>
</body>
</html>
```

## Правила работы с состоянием

### 1. Состояние иммутабельно

```javascript
// ❌ Неправильно — мутация
const [items, setItems] = useState([1, 2, 3]);
items.push(4);  // Не вызовет перерисовку!

// ✓ Правильно — новый массив
setItems([...items, 4]);
```

### 2. Не изменяйте объекты напрямую

```javascript
// ❌ Неправильно
const [user, setUser] = useState({ name: 'Иван' });
user.name = 'Пётр';  // Не сработает!

// ✓ Правильно
setUser({ ...user, name: 'Пётр' });
```

### 3. Группируйте связанное состояние

```javascript
// ❌ Плохо — много отдельных состояний
const [firstName, setFirstName] = useState('');
const [lastName, setLastName] = useState('');
const [age, setAge] = useState(0);
const [city, setCity] = useState('');

// ✓ Лучше — один объект
const [user, setUser] = useState({
  firstName: '',
  lastName: '',
  age: 0,
  city: ''
});
```

## Задание

1. Создайте компонент калькулятора:
```javascript
function Calculator() {
  // Состояние: текущее число, операция, предыдущее число
  // Кнопки: 0-9, +, -, *, /, =, C
}
```

2. Реализуйте компонент формы регистрации с валидацией:
```javascript
function RegistrationForm() {
  // Поля: email, password, confirmPassword
  // Показывать ошибки валидации
}
```

3. Создайте компонент таймера:
```javascript
function Timer() {
  // Кнопки: Старт, Пауза, Сброс
  // Отображение: MM:SS
  // Используйте setInterval
}
```

## Ключевые выводы

- useState позволяет компонентам иметь собственное состояние
- Хуки вызываются в одном порядке при каждом рендере
- setState автоматически планирует перерисовку
- Обновления батчатся для производительности
- Состояние должно быть иммутабельным
- Функциональные обновления для зависимых изменений
- Ленивая инициализация для дорогих вычислений
- Следующий шаг — useEffect для побочных эффектов

---

[← Предыдущий урок: Функциональные компоненты](./10-functional-components.md) | [Следующий урок: Lifecycle (useEffect) →](./12-lifecycle.md)

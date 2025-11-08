# Урок 6: Рендеринг списков и фрагментов

## Проблема: как рендерить массивы?

В реальных приложениях часто нужно отображать списки данных:

```javascript
const users = [
  { id: 1, name: 'Иван' },
  { id: 2, name: 'Мария' },
  { id: 3, name: 'Пётр' }
];

// Как превратить массив в виртуальные узлы?
```

Попробуем интуитивный подход:

```javascript
const vNode = h('ul', null,
  users.map(user => h('li', null, user.name))
);
```

Проблема: `users.map()` возвращает **массив**, а `h()` ожидает отдельные элементы в children:

```javascript
// Что мы получаем:
{
  type: 'ul',
  props: {},
  children: [
    [  // ← массив внутри массива!
      { type: 'li', props: {}, children: ['Иван'] },
      { type: 'li', props: {}, children: ['Мария'] },
      { type: 'li', props: {}, children: ['Пётр'] }
    ]
  ]
}

// Что нужно:
{
  type: 'ul',
  props: {},
  children: [
    { type: 'li', props: {}, children: ['Иван'] },
    { type: 'li', props: {}, children: ['Мария'] },
    { type: 'li', props: {}, children: ['Пётр'] }
  ]
}
```

## Решение 1: flatten children

Модифицируем функцию `h()` для автоматического выравнивания массивов:

```javascript
/**
 * Выравнивает массивы детей (flatten)
 * @param {Array} children
 * @returns {Array}
 */
function flattenChildren(children) {
  const result = [];

  for (const child of children) {
    if (Array.isArray(child)) {
      // Рекурсивно выравниваем вложенные массивы
      result.push(...flattenChildren(child));
    } else if (child != null && child !== false && child !== true) {
      // Пропускаем null, undefined, boolean
      result.push(child);
    }
  }

  return result;
}

/**
 * Создаёт виртуальный узел
 */
export function h(type, props, ...children) {
  return {
    type,
    props: props || {},
    children: flattenChildren(children)
  };
}
```

Теперь это работает:

```javascript
const vNode = h('ul', null,
  users.map(user => h('li', null, user.name))
);

// Результат:
// {
//   type: 'ul',
//   children: [
//     { type: 'li', children: ['Иван'] },
//     { type: 'li', children: ['Мария'] },
//     { type: 'li', children: ['Пётр'] }
//   ]
// }
```

### Почему пропускаем boolean?

Это позволяет использовать условный рендеринг:

```javascript
h('div', null,
  isLoggedIn && h('p', null, 'Добро пожаловать!'),
  // Если isLoggedIn = false, то false не рендерится
  showWarning && h('p', null, 'Внимание!')
)
```

## Пример: список задач

```javascript
const todos = [
  { id: 1, text: 'Изучить Virtual DOM', done: true },
  { id: 2, text: 'Написать diffing', done: false },
  { id: 3, text: 'Добавить hooks', done: false }
];

function TodoList({ todos }) {
  return h('div', { class: 'todo-list' },
    h('h2', null, 'Список задач'),
    h('ul', null,
      todos.map(todo =>
        h('li', {
          class: todo.done ? 'done' : '',
          style: {
            textDecoration: todo.done ? 'line-through' : 'none',
            color: todo.done ? '#999' : '#000'
          }
        },
          h('input', {
            type: 'checkbox',
            checked: todo.done
          }),
          ' ',
          todo.text
        )
      )
    )
  );
}
```

## Фрагменты (Fragments)

### Проблема: лишний wrapper

Иногда нужно вернуть несколько элементов без обёртки:

```javascript
// ❌ Лишний div
function UserInfo() {
  return h('div', null,  // ← не нужен!
    h('h3', null, user.name),
    h('p', null, user.email)
  );
}

// Результат:
// <div>     ← лишний!
//   <h3>Иван</h3>
//   <p>ivan@example.com</p>
// </div>
```

### Решение: Fragment

Fragment — это специальный тип элемента, который не создаёт DOM узел:

```javascript
/**
 * Символ для обозначения фрагмента
 */
export const Fragment = Symbol('Fragment');

/**
 * Создаёт фрагмент - группу элементов без wrapper
 */
export function createFragment(...children) {
  return h(Fragment, null, ...children);
}
```

Обновляем `createElement`:

```javascript
export function createElement(vNode) {
  if (typeof vNode === 'string') {
    return document.createTextNode(vNode);
  }

  // Фрагмент
  if (vNode.type === Fragment) {
    // Создаём document fragment
    const fragment = document.createDocumentFragment();
    vNode.children.forEach(child => {
      fragment.appendChild(createElement(child));
    });
    return fragment;
  }

  // Обычный элемент
  const element = document.createElement(vNode.type);
  setProps(element, vNode.props);

  vNode.children.forEach(child => {
    element.appendChild(createElement(child));
  });

  return element;
}
```

Использование:

```javascript
function UserInfo() {
  return h(Fragment, null,
    h('h3', null, user.name),
    h('p', null, user.email)
  );
}

// Результат (без лишнего div):
// <h3>Иван</h3>
// <p>ivan@example.com</p>
```

### Краткая запись Fragment

Для удобства можно использовать массив:

```javascript
function UserInfo() {
  return [
    h('h3', null, user.name),
    h('p', null, user.email)
  ];
}
```

Но нужно обновить `mount` для работы с массивами:

```javascript
export function mount(vNode, container) {
  // Если vNode - массив, оборачиваем во Fragment
  if (Array.isArray(vNode)) {
    vNode = h(Fragment, null, ...vNode);
  }

  const element = createElement(vNode);
  container.innerHTML = '';
  container.appendChild(element);
  return element;
}
```

## Условный рендеринг

### Тернарный оператор

```javascript
function Greeting({ isLoggedIn, username }) {
  return h('div', null,
    isLoggedIn
      ? h('h1', null, `Привет, ${username}!`)
      : h('button', null, 'Войти')
  );
}
```

### Логическое И (&&)

```javascript
function Notifications({ count }) {
  return h('div', null,
    h('h2', null, 'Уведомления'),
    count > 0 && h('span', { class: 'badge' }, count)
    // Если count = 0, то false не рендерится
  );
}
```

### Switch-подобная логика

```javascript
function StatusIcon({ status }) {
  const icons = {
    success: '✓',
    error: '✗',
    warning: '⚠',
    info: 'ℹ'
  };

  return h('span', {
    class: `status-icon ${status}`,
    style: {
      color: status === 'success' ? 'green' :
             status === 'error' ? 'red' :
             status === 'warning' ? 'orange' : 'blue'
    }
  }, icons[status] || '?');
}
```

## Работа с вложенными списками

```javascript
const categories = [
  {
    id: 1,
    name: 'Фрукты',
    items: ['Яблоко', 'Банан', 'Апельсин']
  },
  {
    id: 2,
    name: 'Овощи',
    items: ['Морковь', 'Огурец', 'Помидор']
  }
];

function CategoriesList({ categories }) {
  return h('div', null,
    categories.map(category =>
      h('div', { class: 'category' },
        h('h3', null, category.name),
        h('ul', null,
          category.items.map(item =>
            h('li', null, item)
          )
        )
      )
    )
  );
}
```

## Фильтрация и сортировка списков

```javascript
function FilteredList({ items, filter, sortBy }) {
  // Фильтруем
  let filtered = items.filter(item =>
    item.name.toLowerCase().includes(filter.toLowerCase())
  );

  // Сортируем
  if (sortBy === 'name') {
    filtered = filtered.sort((a, b) => a.name.localeCompare(b.name));
  } else if (sortBy === 'date') {
    filtered = filtered.sort((a, b) => b.date - a.date);
  }

  return h('div', null,
    h('p', null, `Найдено: ${filtered.length}`),
    filtered.length === 0
      ? h('p', { class: 'empty' }, 'Ничего не найдено')
      : h('ul', null,
          filtered.map(item =>
            h('li', null, item.name)
          )
        )
  );
}
```

## Полный пример: динамический список

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Nano Framework - Lists</title>
  <style>
    .app {
      font-family: sans-serif;
      max-width: 600px;
      margin: 20px auto;
    }
    .controls {
      margin-bottom: 20px;
    }
    input[type="text"] {
      padding: 8px;
      width: 300px;
      margin-right: 10px;
    }
    button {
      padding: 8px 16px;
      cursor: pointer;
    }
    ul {
      list-style: none;
      padding: 0;
    }
    li {
      padding: 10px;
      margin: 5px 0;
      background: #f0f0f0;
      border-radius: 4px;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }
    .completed {
      text-decoration: line-through;
      opacity: 0.6;
    }
    .remove-btn {
      background: #ff4444;
      color: white;
      border: none;
      padding: 4px 8px;
      border-radius: 4px;
      cursor: pointer;
    }
  </style>
</head>
<body>
  <div id="app"></div>

  <script type="module">
    import { h, mount, Fragment } from './nano-framework.js';

    let todos = [
      { id: 1, text: 'Изучить Virtual DOM', completed: true },
      { id: 2, text: 'Реализовать diffing', completed: false }
    ];
    let input = '';
    let nextId = 3;

    function addTodo() {
      if (input.trim()) {
        todos = [...todos, {
          id: nextId++,
          text: input,
          completed: false
        }];
        input = '';
        render();
      }
    }

    function toggleTodo(id) {
      todos = todos.map(todo =>
        todo.id === id
          ? { ...todo, completed: !todo.completed }
          : todo
      );
      render();
    }

    function removeTodo(id) {
      todos = todos.filter(todo => todo.id !== id);
      render();
    }

    function render() {
      const vNode = h('div', { class: 'app' },
        h('h1', null, 'Список задач'),

        // Форма добавления
        h('div', { class: 'controls' },
          h('input', {
            type: 'text',
            value: input,
            placeholder: 'Новая задача...',
            oninput: (e) => {
              input = e.target.value;
              render();
            },
            onkeypress: (e) => {
              if (e.key === 'Enter') {
                addTodo();
              }
            }
          }),
          h('button', { onclick: addTodo }, 'Добавить')
        ),

        // Статистика
        h('p', null,
          `Всего: ${todos.length}, `,
          `Выполнено: ${todos.filter(t => t.completed).length}`
        ),

        // Список задач
        todos.length === 0
          ? h('p', { style: { color: '#999' } }, 'Список пуст')
          : h('ul', null,
              todos.map(todo =>
                h('li', {
                  class: todo.completed ? 'completed' : ''
                },
                  h('label', {
                    style: { cursor: 'pointer', flex: 1 }
                  },
                    h('input', {
                      type: 'checkbox',
                      checked: todo.completed,
                      onchange: () => toggleTodo(todo.id)
                    }),
                    ' ',
                    todo.text
                  ),
                  h('button', {
                    class: 'remove-btn',
                    onclick: () => removeTodo(todo.id)
                  }, '×')
                )
              )
            )
      );

      mount(vNode, document.getElementById('app'));
    }

    render();
  </script>
</body>
</html>
```

## Проблема производительности

При каждом изменении мы пересоздаём весь список:

```javascript
todos.map(todo => h('li', ...))  // Новые виртуальные узлы каждый раз
mount(vNode, container)          // Пересоздаём весь DOM
```

**Что плохо:**
- При изменении одной задачи пересоздаём все `<li>`
- Теряется фокус на элементах
- Сбрасываются анимации
- Неэффективно для больших списков

**Решение:** diffing алгоритм с ключами (keys) — в следующих уроках!

## Задание

1. Создайте компонент `Select` для выбора из списка:
```javascript
const options = ['JavaScript', 'Python', 'Rust', 'Go'];
h('select', { onchange: handleChange },
  options.map(opt => h('option', { value: opt }, opt))
)
```

2. Реализуйте фильтруемый список пользователей:
```javascript
// Фильтр по имени + сортировка по возрасту
const users = [
  { id: 1, name: 'Иван', age: 25 },
  { id: 2, name: 'Мария', age: 30 },
  { id: 3, name: 'Пётр', age: 20 }
];
```

3. Добавьте поддержку вложенных комментариев:
```javascript
const comments = [
  {
    id: 1,
    text: 'Отличная статья!',
    replies: [
      { id: 2, text: 'Согласен!' },
      { id: 3, text: 'Тоже понравилось' }
    ]
  }
];
// Рекурсивно рендерите комментарии и ответы
```

## Ключевые выводы

- Массивы автоматически выравниваются (flatten) в `children`
- `null`, `undefined`, `boolean` не рендерятся — удобно для условий
- Fragment позволяет группировать элементы без wrapper
- `map()` — основной способ рендеринга списков
- Условный рендеринг: тернарный оператор и `&&`
- Вложенные `map()` для вложенных списков
- Текущий подход неэффективен — пересоздаём весь DOM
- Для оптимизации нужны keys и diffing алгоритм (следующие уроки)

---

[← Предыдущий урок: Работа с атрибутами и свойствами](./05-attributes-and-properties.md) | [Следующий урок: Алгоритм diffing →](./07-diffing-algorithm.md)

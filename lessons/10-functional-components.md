# Урок 10: Функциональные компоненты

## Проблема: дублирование кода

Сейчас мы пишем всё приложение как одну большую функцию:

```javascript
function render() {
  return h('div', null,
    h('header', null,
      h('nav', null,
        h('a', { href: '/' }, 'Главная'),
        h('a', { href: '/about' }, 'О нас')
      )
    ),
    h('main', null,
      h('h1', null, 'Список пользователей'),
      users.map(user =>
        h('div', { class: 'user-card' },
          h('img', { src: user.avatar }),
          h('h2', null, user.name),
          h('p', null, user.email)
        )
      )
    ),
    h('footer', null, '© 2024')
  );
}
```

**Проблемы:**
1. Код тяжело читать
2. Нельзя переиспользовать части (например, UserCard)
3. Всё в одном месте — сложно поддерживать
4. Нет изоляции логики

## Решение: компоненты

**Компонент** — это функция, которая принимает props и возвращает виртуальное дерево:

```javascript
function UserCard({ user }) {
  return h('div', { class: 'user-card' },
    h('img', { src: user.avatar }),
    h('h2', null, user.name),
    h('p', null, user.email)
  );
}

// Использование
h('div', null,
  users.map(user => UserCard({ user }))
)
```

Но это неудобно — нужно вручную вызывать функцию. Давайте интегрируем компоненты в `h()`.

## Компоненты как элементы

Идея: использовать функцию как `type`:

```javascript
// Вместо строки используем функцию
h(UserCard, { user: userData })

// Внутри h() проверяем тип:
if (typeof type === 'function') {
  // Это компонент!
}
```

## Обновляем функцию h()

```javascript
export function h(type, props, ...children) {
  // Если type — функция, вызываем её
  if (typeof type === 'function') {
    const componentProps = { ...props, children };
    return type(componentProps);
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

**Как это работает:**

```javascript
// Было:
function UserCard({ user }) {
  return h('div', { class: 'user-card' },
    h('h2', null, user.name)
  );
}
UserCard({ user });

// Стало:
h(UserCard, { user })

// h() вызовет UserCard({ user, children: [] })
// и вернёт результат (виртуальное дерево)
```

## Пример: Header компонент

```javascript
function Header({ title }) {
  return h('header', null,
    h('h1', null, title),
    h('nav', null,
      h('a', { href: '/' }, 'Главная'),
      h('a', { href: '/about' }, 'О нас'),
      h('a', { href: '/contact' }, 'Контакты')
    )
  );
}

// Использование
function App() {
  return h('div', null,
    h(Header, { title: 'Мой сайт' }),
    h('main', null, 'Контент')
  );
}
```

## Передача children

```javascript
function Card({ title, children }) {
  return h('div', { class: 'card' },
    h('h3', null, title),
    h('div', { class: 'card-content' }, ...children)
  );
}

// Использование
h(Card, { title: 'Заголовок' },
  h('p', null, 'Первый параграф'),
  h('p', null, 'Второй параграф')
)

// children автоматически передаются в props
```

## Композиция компонентов

```javascript
function Button({ text, onClick, variant = 'primary' }) {
  return h('button', {
    class: `btn btn-${variant}`,
    onclick: onClick
  }, text);
}

function Form({ onSubmit }) {
  return h('form', { onsubmit: onSubmit },
    h('input', { type: 'text', placeholder: 'Имя' }),
    h('input', { type: 'email', placeholder: 'Email' }),
    h(Button, { text: 'Отправить', variant: 'primary' }),
    h(Button, { text: 'Отмена', variant: 'secondary' })
  );
}

function App() {
  return h('div', null,
    h('h1', null, 'Регистрация'),
    h(Form, { onSubmit: handleSubmit })
  );
}
```

## Props по умолчанию

```javascript
function Avatar({ src, size = 48, alt = 'Avatar' }) {
  return h('img', {
    src,
    alt,
    width: size,
    height: size,
    style: { borderRadius: '50%' }
  });
}

// Использование
h(Avatar, { src: '/user.jpg' })
// size и alt будут 48 и 'Avatar'
```

## Деструктуризация props

```javascript
// ✓ Хорошо — явно видно, что используется
function UserCard({ name, email, avatar }) {
  return h('div', null,
    h('img', { src: avatar }),
    h('h2', null, name),
    h('p', null, email)
  );
}

// Тоже можно — остальные props
function Button({ variant, ...rest }) {
  return h('button', {
    ...rest,
    class: `btn btn-${variant}`
  });
}
```

## Условный рендеринг в компонентах

```javascript
function Greeting({ isLoggedIn, username }) {
  if (isLoggedIn) {
    return h('div', null,
      h('h1', null, `Привет, ${username}!`),
      h('button', null, 'Выйти')
    );
  }

  return h('div', null,
    h('h1', null, 'Добро пожаловать!'),
    h('button', null, 'Войти')
  );
}

// Или через тернарный оператор
function Greeting({ isLoggedIn, username }) {
  return isLoggedIn
    ? h('div', null, h('h1', null, `Привет, ${username}!`))
    : h('div', null, h('h1', null, 'Добро пожаловать!'));
}
```

## Списки в компонентах

```javascript
function TodoItem({ todo, onToggle, onDelete }) {
  return h('li', { key: todo.id },
    h('input', {
      type: 'checkbox',
      checked: todo.done,
      onchange: () => onToggle(todo.id)
    }),
    h('span', {
      style: { textDecoration: todo.done ? 'line-through' : 'none' }
    }, todo.text),
    h('button', { onclick: () => onDelete(todo.id) }, 'Удалить')
  );
}

function TodoList({ todos, onToggle, onDelete }) {
  return h('ul', null,
    todos.map(todo =>
      h(TodoItem, { todo, onToggle, onDelete })
    )
  );
}
```

## Вложенные компоненты

```javascript
function Comment({ comment }) {
  return h('div', { class: 'comment' },
    h('div', { class: 'comment-header' },
      h('strong', null, comment.author),
      ' · ',
      h('span', { class: 'date' }, comment.date)
    ),
    h('p', null, comment.text),
    // Рекурсивно рендерим ответы
    comment.replies && comment.replies.length > 0 &&
      h('div', { class: 'replies' },
        comment.replies.map(reply =>
          h(Comment, { comment: reply, key: reply.id })
        )
      )
  );
}

function CommentList({ comments }) {
  return h('div', { class: 'comment-list' },
    comments.map(comment =>
      h(Comment, { comment, key: comment.id })
    )
  );
}
```

## Передача функций через props

```javascript
function SearchBar({ onSearch }) {
  let query = '';

  return h('div', { class: 'search-bar' },
    h('input', {
      type: 'text',
      placeholder: 'Поиск...',
      oninput: (e) => {
        query = e.target.value;
      }
    }),
    h('button', {
      onclick: () => onSearch(query)
    }, 'Найти')
  );
}

function App() {
  function handleSearch(query) {
    console.log('Поиск:', query);
    // Выполняем поиск
  }

  return h('div', null,
    h(SearchBar, { onSearch: handleSearch })
  );
}
```

## Переиспользование компонентов

```javascript
function Badge({ text, color = 'blue' }) {
  return h('span', {
    class: 'badge',
    style: {
      backgroundColor: color,
      color: 'white',
      padding: '4px 8px',
      borderRadius: '4px'
    }
  }, text);
}

function UserCard({ user }) {
  return h('div', { class: 'user-card' },
    h('h3', null, user.name),
    h('div', null,
      h(Badge, { text: user.role, color: 'green' }),
      user.isOnline && h(Badge, { text: 'Online', color: 'blue' })
    )
  );
}

function Notification({ notification }) {
  return h('div', { class: 'notification' },
    h(Badge, { text: notification.type, color: 'red' }),
    h('p', null, notification.message)
  );
}
```

## Компоненты высшего порядка (HOC)

```javascript
// HOC — функция, которая принимает компонент и возвращает новый компонент
function withLoading(Component) {
  return function LoadingWrapper({ isLoading, ...props }) {
    if (isLoading) {
      return h('div', { class: 'loading' }, 'Загрузка...');
    }
    return h(Component, props);
  };
}

// Использование
const UserListWithLoading = withLoading(UserList);

h(UserListWithLoading, {
  isLoading: isLoading,
  users: users
})
```

## Полный пример: приложение с компонентами

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Functional Components</title>
  <style>
    * { box-sizing: border-box; }
    body {
      font-family: sans-serif;
      max-width: 600px;
      margin: 40px auto;
      padding: 0 20px;
    }
    .todo-item {
      padding: 10px;
      border-bottom: 1px solid #eee;
      display: flex;
      align-items: center;
      gap: 10px;
    }
    .todo-item input[type="checkbox"] {
      width: 20px;
      height: 20px;
    }
    .todo-text {
      flex: 1;
    }
    .btn {
      padding: 8px 16px;
      border: none;
      border-radius: 4px;
      cursor: pointer;
    }
    .btn-primary { background: #0066cc; color: white; }
    .btn-danger { background: #cc0000; color: white; }
    .input-group {
      display: flex;
      gap: 10px;
      margin-bottom: 20px;
    }
    .input-group input {
      flex: 1;
      padding: 8px;
      border: 1px solid #ccc;
      border-radius: 4px;
    }
  </style>
</head>
<body>
  <div id="app"></div>

  <script type="module">
    import { h, mount, diff, patch, createElement } from './nano-framework.js';

    // Компоненты
    function Header({ title, subtitle }) {
      return h('header', null,
        h('h1', null, title),
        subtitle && h('p', null, subtitle)
      );
    }

    function TodoItem({ todo, onToggle, onDelete }) {
      return h('div', { class: 'todo-item', key: todo.id },
        h('input', {
          type: 'checkbox',
          checked: todo.done,
          onchange: () => onToggle(todo.id)
        }),
        h('span', {
          class: 'todo-text',
          style: {
            textDecoration: todo.done ? 'line-through' : 'none',
            color: todo.done ? '#999' : '#000'
          }
        }, todo.text),
        h('button', {
          class: 'btn btn-danger',
          onclick: () => onDelete(todo.id)
        }, 'Удалить')
      );
    }

    function TodoList({ todos, onToggle, onDelete }) {
      if (todos.length === 0) {
        return h('p', { style: { color: '#999', textAlign: 'center' } },
          'Список пуст. Добавьте первую задачу!'
        );
      }

      return h('div', null,
        todos.map(todo =>
          h(TodoItem, { todo, onToggle, onDelete, key: todo.id })
        )
      );
    }

    function AddTodoForm({ onAdd }) {
      let input = '';

      function handleSubmit(e) {
        e.preventDefault();
        if (input.trim()) {
          onAdd(input.trim());
          input = '';
          rerender();
        }
      }

      return h('form', { class: 'input-group', onsubmit: handleSubmit },
        h('input', {
          type: 'text',
          value: input,
          placeholder: 'Новая задача...',
          oninput: (e) => {
            input = e.target.value;
            rerender();
          }
        }),
        h('button', { class: 'btn btn-primary', type: 'submit' }, 'Добавить')
      );
    }

    function Stats({ total, completed }) {
      return h('div', {
        style: {
          padding: '10px',
          background: '#f5f5f5',
          borderRadius: '4px',
          marginTop: '20px'
        }
      },
        h('strong', null, 'Статистика: '),
        `Всего: ${total}, `,
        `Выполнено: ${completed}, `,
        `Осталось: ${total - completed}`
      );
    }

    function App({ todos, onAdd, onToggle, onDelete }) {
      const completed = todos.filter(t => t.done).length;

      return h('div', null,
        h(Header, {
          title: 'Todo App',
          subtitle: 'Построено с Nano Framework'
        }),
        h(AddTodoForm, { onAdd }),
        h(TodoList, { todos, onToggle, onDelete }),
        h(Stats, { total: todos.length, completed })
      );
    }

    // Состояние приложения
    let todos = [
      { id: 1, text: 'Изучить компоненты', done: true },
      { id: 2, text: 'Добавить state', done: false }
    ];
    let nextId = 3;
    let app = null;

    // Обработчики
    function addTodo(text) {
      todos = [...todos, { id: nextId++, text, done: false }];
      rerender();
    }

    function toggleTodo(id) {
      todos = todos.map(todo =>
        todo.id === id ? { ...todo, done: !todo.done } : todo
      );
      rerender();
    }

    function deleteTodo(id) {
      todos = todos.filter(todo => todo.id !== id);
      rerender();
    }

    // Рендеринг
    function rerender() {
      const newVNode = h(App, {
        todos,
        onAdd: addTodo,
        onToggle: toggleTodo,
        onDelete: deleteTodo
      });

      const container = document.getElementById('app');

      if (!app) {
        const element = createElement(newVNode);
        container.appendChild(element);
        app = { vNode: newVNode, element };
      } else {
        const patches = diff(app.vNode, newVNode);
        const element = patch(container, patches, app.element, 0);
        app = { vNode: newVNode, element };
      }
    }

    rerender();
  </script>
</body>
</html>
```

## Преимущества компонентов

1. **Переиспользование:** один раз написали — используем везде
2. **Изоляция:** каждый компонент отвечает за свою часть
3. **Читаемость:** понятная структура приложения
4. **Тестирование:** легко тестировать отдельные компоненты
5. **Композиция:** собираем сложное из простого

## Паттерны компонентов

### Презентационные компоненты

Только отображение, без логики:

```javascript
function UserAvatar({ src, size, name }) {
  return h('img', {
    src,
    alt: name,
    width: size,
    height: size,
    style: { borderRadius: '50%' }
  });
}
```

### Контейнер-компоненты

Содержат логику и данные:

```javascript
function UserListContainer() {
  const users = fetchUsers(); // Логика получения данных

  return h(UserList, { users }); // Передача в презентационный
}
```

## Задание

1. Создайте компонент `Tabs` с переключением вкладок:
```javascript
function Tabs({ tabs, activeTab, onTabChange }) {
  // tabs = [{ id, title, content }, ...]
}
```

2. Реализуйте компонент `Modal` (модальное окно):
```javascript
function Modal({ isOpen, title, children, onClose }) {
  if (!isOpen) return null;
  // ...
}
```

3. Создайте компонент `Pagination`:
```javascript
function Pagination({ currentPage, totalPages, onPageChange }) {
  // Рендерит кнопки: << 1 2 3 ... 10 >>
}
```

## Ключевые выводы

- Компонент — функция, принимающая props и возвращающая vNode
- Компоненты позволяют разбить приложение на части
- Props передаются как объект, дети — в `children`
- Компоненты можно композировать (вкладывать друг в друга)
- Разделение на презентационные и контейнер-компоненты
- Компоненты упрощают переиспользование и тестирование
- Следующий шаг — добавить state (внутреннее состояние)

---

[← Предыдущий урок: Ключи и оптимизация списков](./09-keys-and-optimization.md) | [Следующий урок: Состояние (useState) →](./11-state.md)

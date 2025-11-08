# Урок 17: Построение Todo приложения

## Финальный проект

Применим всё, что изучили, для создания полнофункционального Todo приложения с:
- Добавление/удаление/редактирование задач
- Фильтрация (все/активные/завершённые)
- Подсчёт статистики
- Сохранение в localStorage
- Анимации
- Оптимизация производительности

## Архитектура приложения

```
App
├── Header (заголовок + статистика)
├── TodoInput (форма добавления)
├── FilterButtons (фильтры)
├── TodoList
│   └── TodoItem (отдельная задача)
└── Footer (дополнительные действия)
```

## Структура данных

```javascript
// Задача
{
  id: number,
  text: string,
  completed: boolean,
  createdAt: timestamp,
  priority: 'low' | 'medium' | 'high'
}

// Состояние приложения
{
  todos: Todo[],
  filter: 'all' | 'active' | 'completed',
  editingId: number | null
}
```

## Контекст приложения

```javascript
// TodoContext.js
import { createContext, useState, useEffect } from './nano-framework.js';

const TodoContext = createContext();

function TodoProvider({ children }) {
  // Состояние
  const [todos, setTodos] = useState(() => {
    const saved = localStorage.getItem('todos');
    return saved ? JSON.parse(saved) : [];
  });

  const [filter, setFilter] = useState('all');
  const [editingId, setEditingId] = useState(null);

  // Сохранение в localStorage
  useEffect(() => {
    localStorage.setItem('todos', JSON.stringify(todos));
  }, [todos]);

  // Методы
  const addTodo = (text, priority = 'medium') => {
    const todo = {
      id: Date.now(),
      text,
      completed: false,
      createdAt: Date.now(),
      priority
    };
    setTodos([...todos, todo]);
  };

  const toggleTodo = (id) => {
    setTodos(todos.map(todo =>
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    ));
  };

  const deleteTodo = (id) => {
    setTodos(todos.filter(todo => todo.id !== id));
  };

  const updateTodo = (id, text) => {
    setTodos(todos.map(todo =>
      todo.id === id ? { ...todo, text } : todo
    ));
    setEditingId(null);
  };

  const clearCompleted = () => {
    setTodos(todos.filter(todo => !todo.completed));
  };

  const toggleAll = () => {
    const allCompleted = todos.every(todo => todo.completed);
    setTodos(todos.map(todo => ({
      ...todo,
      completed: !allCompleted
    })));
  };

  // Вычисляемые значения
  const filteredTodos = todos.filter(todo => {
    if (filter === 'active') return !todo.completed;
    if (filter === 'completed') return todo.completed;
    return true;
  });

  const stats = {
    total: todos.length,
    active: todos.filter(t => !t.completed).length,
    completed: todos.filter(t => t.completed).length
  };

  const value = {
    todos,
    filteredTodos,
    filter,
    setFilter,
    editingId,
    setEditingId,
    stats,
    addTodo,
    toggleTodo,
    deleteTodo,
    updateTodo,
    clearCompleted,
    toggleAll
  };

  return h(TodoContext.Provider, { value }, ...children);
}

export { TodoContext, TodoProvider };
```

## Компоненты

### Header

```javascript
function Header() {
  const { stats } = useContext(TodoContext);

  return h('header', { class: 'header' },
    h('h1', null, 'Список задач'),
    h('div', { class: 'stats' },
      h('span', null, `Всего: ${stats.total}`),
      ' · ',
      h('span', null, `Активных: ${stats.active}`),
      ' · ',
      h('span', null, `Завершено: ${stats.completed}`)
    )
  );
}
```

### TodoInput

```javascript
function TodoInput() {
  const [text, setText] = useState('');
  const [priority, setPriority] = useState('medium');
  const { addTodo } = useContext(TodoContext);
  const inputRef = useRef(null);

  function handleSubmit(e) {
    e.preventDefault();

    if (text.trim()) {
      addTodo(text.trim(), priority);
      setText('');
      setPriority('medium');
      inputRef.current.focus();
    }
  }

  return h('form', { class: 'todo-input', onsubmit: handleSubmit },
    h('input', {
      ref: inputRef,
      type: 'text',
      value: text,
      placeholder: 'Что нужно сделать?',
      oninput: (e) => setText(e.target.value),
      class: 'input-text'
    }),

    h('select', {
      value: priority,
      onchange: (e) => setPriority(e.target.value),
      class: 'input-priority'
    },
      h('option', { value: 'low' }, 'Низкий'),
      h('option', { value: 'medium' }, 'Средний'),
      h('option', { value: 'high' }, 'Высокий')
    ),

    h('button', { type: 'submit', class: 'btn btn-primary' }, 'Добавить')
  );
}
```

### FilterButtons

```javascript
function FilterButtons() {
  const { filter, setFilter } = useContext(TodoContext);

  const filters = [
    { value: 'all', label: 'Все' },
    { value: 'active', label: 'Активные' },
    { value: 'completed', label: 'Завершённые' }
  ];

  return h('div', { class: 'filters' },
    filters.map(f =>
      h('button', {
        key: f.value,
        class: `btn ${filter === f.value ? 'active' : ''}`,
        onclick: () => setFilter(f.value)
      }, f.label)
    )
  );
}
```

### TodoItem

```javascript
const TodoItem = memo(({ todo }) => {
  const { toggleTodo, deleteTodo, updateTodo, editingId, setEditingId } = useContext(TodoContext);
  const [editText, setEditText] = useState(todo.text);
  const editInputRef = useRef(null);

  const isEditing = editingId === todo.id;

  useEffect(() => {
    if (isEditing && editInputRef.current) {
      editInputRef.current.focus();
    }
  }, [isEditing]);

  function handleEdit() {
    setEditingId(todo.id);
    setEditText(todo.text);
  }

  function handleSave() {
    if (editText.trim()) {
      updateTodo(todo.id, editText.trim());
    } else {
      deleteTodo(todo.id);
    }
  }

  function handleKeyDown(e) {
    if (e.key === 'Enter') {
      handleSave();
    } else if (e.key === 'Escape') {
      setEditingId(null);
      setEditText(todo.text);
    }
  }

  const priorityColors = {
    low: '#4CAF50',
    medium: '#FF9800',
    high: '#F44336'
  };

  if (isEditing) {
    return h('li', { class: 'todo-item editing', key: todo.id },
      h('input', {
        ref: editInputRef,
        type: 'text',
        value: editText,
        oninput: (e) => setEditText(e.target.value),
        onblur: handleSave,
        onkeydown: handleKeyDown,
        class: 'edit-input'
      })
    );
  }

  return h('li', {
    class: `todo-item ${todo.completed ? 'completed' : ''} priority-${todo.priority}`,
    key: todo.id
  },
    h('div', { class: 'todo-content' },
      h('input', {
        type: 'checkbox',
        checked: todo.completed,
        onchange: () => toggleTodo(todo.id),
        class: 'todo-checkbox'
      }),

      h('span', {
        class: 'todo-text',
        ondblclick: handleEdit
      }, todo.text),

      h('span', {
        class: 'priority-badge',
        style: { background: priorityColors[todo.priority] }
      }, todo.priority)
    ),

    h('div', { class: 'todo-actions' },
      h('button', {
        onclick: handleEdit,
        class: 'btn btn-small',
        title: 'Редактировать'
      }, '✏️'),

      h('button', {
        onclick: () => deleteTodo(todo.id),
        class: 'btn btn-small btn-danger',
        title: 'Удалить'
      }, '🗑️')
    )
  );
});
```

### TodoList

```javascript
function TodoList() {
  const { filteredTodos } = useContext(TodoContext);

  if (filteredTodos.length === 0) {
    return h('div', { class: 'empty-state' },
      h('p', null, '📝 Список пуст'),
      h('p', { class: 'hint' }, 'Добавьте первую задачу!')
    );
  }

  return h('ul', { class: 'todo-list' },
    filteredTodos.map(todo =>
      h(TodoItem, { todo, key: todo.id })
    )
  );
}
```

### Footer

```javascript
function Footer() {
  const { todos, stats, clearCompleted, toggleAll } = useContext(TodoContext);

  if (todos.length === 0) return null;

  return h('footer', { class: 'footer' },
    h('button', {
      onclick: toggleAll,
      class: 'btn btn-secondary'
    }, stats.active > 0 ? 'Отметить все' : 'Снять все'),

    stats.completed > 0 &&
      h('button', {
        onclick: clearCompleted,
        class: 'btn btn-danger'
      }, `Удалить завершённые (${stats.completed})`)
  );
}
```

### App

```javascript
function App() {
  return h(TodoProvider, null,
    h('div', { class: 'app' },
      h(Header),
      h(TodoInput),
      h(FilterButtons),
      h(TodoList),
      h(Footer)
    )
  );
}
```

## Полный HTML файл

```html
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Todo App - Nano Framework</title>
  <style>
    * {
      box-sizing: border-box;
      margin: 0;
      padding: 0;
    }

    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      min-height: 100vh;
      padding: 20px;
    }

    .app {
      max-width: 600px;
      margin: 0 auto;
      background: white;
      border-radius: 12px;
      box-shadow: 0 20px 60px rgba(0,0,0,0.3);
      overflow: hidden;
    }

    .header {
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      color: white;
      padding: 30px;
      text-align: center;
    }

    .header h1 {
      font-size: 32px;
      margin-bottom: 10px;
    }

    .stats {
      font-size: 14px;
      opacity: 0.9;
    }

    .todo-input {
      padding: 20px;
      display: flex;
      gap: 10px;
      border-bottom: 1px solid #eee;
    }

    .input-text {
      flex: 1;
      padding: 12px;
      border: 2px solid #ddd;
      border-radius: 6px;
      font-size: 16px;
      transition: border-color 0.3s;
    }

    .input-text:focus {
      outline: none;
      border-color: #667eea;
    }

    .input-priority {
      padding: 12px;
      border: 2px solid #ddd;
      border-radius: 6px;
      font-size: 14px;
    }

    .btn {
      padding: 12px 24px;
      border: none;
      border-radius: 6px;
      font-size: 14px;
      cursor: pointer;
      transition: all 0.3s;
      font-weight: 500;
    }

    .btn:hover {
      transform: translateY(-2px);
      box-shadow: 0 4px 12px rgba(0,0,0,0.15);
    }

    .btn-primary {
      background: #667eea;
      color: white;
    }

    .btn-secondary {
      background: #f0f0f0;
      color: #333;
    }

    .btn-danger {
      background: #F44336;
      color: white;
    }

    .btn-small {
      padding: 6px 12px;
      font-size: 12px;
    }

    .btn.active {
      background: #667eea;
      color: white;
    }

    .filters {
      padding: 20px;
      display: flex;
      gap: 10px;
      justify-content: center;
      border-bottom: 1px solid #eee;
    }

    .todo-list {
      list-style: none;
      max-height: 400px;
      overflow-y: auto;
    }

    .todo-item {
      padding: 16px 20px;
      border-bottom: 1px solid #eee;
      display: flex;
      align-items: center;
      justify-content: space-between;
      transition: background 0.3s;
      animation: slideIn 0.3s ease-out;
    }

    @keyframes slideIn {
      from {
        opacity: 0;
        transform: translateX(-20px);
      }
      to {
        opacity: 1;
        transform: translateX(0);
      }
    }

    .todo-item:hover {
      background: #f8f8f8;
    }

    .todo-item.completed .todo-text {
      text-decoration: line-through;
      opacity: 0.5;
    }

    .todo-content {
      display: flex;
      align-items: center;
      gap: 12px;
      flex: 1;
    }

    .todo-checkbox {
      width: 20px;
      height: 20px;
      cursor: pointer;
    }

    .todo-text {
      flex: 1;
      font-size: 16px;
      cursor: pointer;
    }

    .priority-badge {
      padding: 4px 8px;
      border-radius: 4px;
      color: white;
      font-size: 11px;
      text-transform: uppercase;
      font-weight: bold;
    }

    .todo-actions {
      display: flex;
      gap: 8px;
      opacity: 0;
      transition: opacity 0.3s;
    }

    .todo-item:hover .todo-actions {
      opacity: 1;
    }

    .edit-input {
      width: 100%;
      padding: 8px;
      font-size: 16px;
      border: 2px solid #667eea;
      border-radius: 4px;
    }

    .empty-state {
      padding: 60px 20px;
      text-align: center;
      color: #999;
    }

    .empty-state p {
      font-size: 24px;
      margin-bottom: 10px;
    }

    .hint {
      font-size: 14px !important;
    }

    .footer {
      padding: 20px;
      display: flex;
      justify-content: space-between;
      border-top: 1px solid #eee;
      background: #f8f8f8;
    }
  </style>
</head>
<body>
  <div id="app"></div>

  <script type="module">
    import {
      h, render, useState, useEffect, useContext, useRef,
      createContext, memo
    } from './nano-framework.js';

    // Весь код компонентов здесь...
    // (TodoContext, Header, TodoInput, FilterButtons, TodoItem, TodoList, Footer, App)

    render(h(App), document.getElementById('app'));
  </script>
</body>
</html>
```

## Дополнительные возможности

### 1. Анимации удаления

```javascript
function TodoItem({ todo }) {
  const [isRemoving, setIsRemoving] = useState(false);
  const { deleteTodo } = useContext(TodoContext);

  function handleDelete() {
    setIsRemoving(true);
    setTimeout(() => {
      deleteTodo(todo.id);
    }, 300); // Длительность анимации
  }

  return h('li', {
    class: `todo-item ${isRemoving ? 'removing' : ''}`,
    // ...
  });
}

// CSS
.todo-item.removing {
  animation: slideOut 0.3s ease-out;
  opacity: 0;
  transform: translateX(100%);
}
```

### 2. Drag & Drop сортировка

```javascript
function TodoItem({ todo, index }) {
  const [isDragging, setIsDragging] = useState(false);
  const { reorderTodos } = useContext(TodoContext);

  return h('li', {
    draggable: true,
    ondragstart: (e) => {
      e.dataTransfer.setData('index', index);
      setIsDragging(true);
    },
    ondragend: () => setIsDragging(false),
    ondragover: (e) => e.preventDefault(),
    ondrop: (e) => {
      e.preventDefault();
      const fromIndex = parseInt(e.dataTransfer.getData('index'));
      reorderTodos(fromIndex, index);
    },
    class: `todo-item ${isDragging ? 'dragging' : ''}`,
    // ...
  });
}
```

### 3. Поиск по задачам

```javascript
function SearchBar() {
  const [query, setQuery] = useState('');
  const { setSearchQuery } = useContext(TodoContext);

  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    setSearchQuery(debouncedQuery);
  }, [debouncedQuery]);

  return h('input', {
    type: 'search',
    value: query,
    placeholder: 'Поиск задач...',
    oninput: (e) => setQuery(e.target.value)
  });
}

function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}
```

### 4. Экспорт/импорт данных

```javascript
function ExportImport() {
  const { todos, setTodos } = useContext(TodoContext);

  function exportData() {
    const dataStr = JSON.stringify(todos, null, 2);
    const dataBlob = new Blob([dataStr], { type: 'application/json' });
    const url = URL.createObjectURL(dataBlob);

    const link = document.createElement('a');
    link.href = url;
    link.download = 'todos.json';
    link.click();

    URL.revokeObjectURL(url);
  }

  function importData(e) {
    const file = e.target.files[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = (event) => {
      try {
        const imported = JSON.parse(event.target.result);
        setTodos(imported);
      } catch (err) {
        alert('Ошибка импорта: ' + err.message);
      }
    };
    reader.readAsText(file);
  }

  return h('div', { class: 'export-import' },
    h('button', { onclick: exportData }, 'Экспорт'),
    h('input', {
      type: 'file',
      accept: '.json',
      onchange: importData,
      style: { display: 'none' },
      id: 'import-input'
    }),
    h('label', {
      for: 'import-input',
      class: 'btn'
    }, 'Импорт')
  );
}
```

## Задания для самостоятельной работы

1. **Категории задач:**
   - Добавьте категории (работа, личное, учёба)
   - Фильтрация по категориям
   - Цветовое кодирование

2. **Дедлайны:**
   - Добавьте поле даты выполнения
   - Подсветка просроченных задач
   - Сортировка по дате

3. **Подзадачи:**
   - Возможность добавлять подзадачи
   - Вложенный список
   - Прогресс выполнения

4. **Синхронизация:**
   - Сохранение на сервер
   - Загрузка с сервера
   - Конфликты при обновлении

5. **PWA:**
   - Service Worker для offline режима
   - Web App Manifest
   - Push уведомления

## Ключевые выводы

- Разделение на компоненты упрощает разработку
- Context избавляет от props drilling
- localStorage для персистентности данных
- Мемоизация оптимизирует производительность
- Refs для фокуса и измерений
- useEffect для сайд-эффектов
- Анимации улучшают UX
- Тестирование важно для качества

## Поздравляем!

Вы построили свой JavaScript фреймворк и создали полноценное приложение!

Теперь вы понимаете:
- Как работает Virtual DOM
- Что такое reconciliation
- Как устроены хуки
- Почему React/Vue/других фреймворков устроены именно так

**Куда двигаться дальше:**
1. Изучите исходный код React/Preact
2. Добавьте TypeScript
3. Реализуйте роутинг
4. Создайте систему плагинов
5. Оптимизируйте производительность
6. Напишите тесты

Удачи в разработке!

---

[← Предыдущий урок: Reconciliation](./16-reconciliation.md) | [Начало курса →](./01-introduction.md)

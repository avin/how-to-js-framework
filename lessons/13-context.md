# Урок 13: Контекст

## Проблема: props drilling

Когда данные нужно передать через несколько уровней компонентов:

```javascript
function App() {
  const user = { name: 'Иван', role: 'admin' };

  return h(Dashboard, { user });
}

function Dashboard({ user }) {
  return h('div', null,
    h(Header, { user }),  // Передаём дальше
    h(Sidebar, { user })  // И ещё раз
  );
}

function Header({ user }) {
  return h(UserMenu, { user }); // И ещё...
}

function UserMenu({ user }) {
  // Наконец-то используем!
  return h('div', null, `Привет, ${user.name}!`);
}
```

**Проблема:** передаём `user` через компоненты, которые его не используют (Dashboard, Header).

Это называется **props drilling** — пробрасывание props через промежуточные компоненты.

**Недостатки:**
1. Громоздкий код
2. Сложно рефакторить
3. Промежуточные компоненты зависят от чужих данных
4. Трудно добавить новые данные

## Решение: Context

**Context** позволяет передавать данные через дерево компонентов без явного пробрасывания props.

```javascript
// Создаём контекст
const UserContext = createContext();

function App() {
  const user = { name: 'Иван', role: 'admin' };

  // Оборачиваем в Provider
  return h(UserContext.Provider, { value: user },
    h(Dashboard)
  );
}

function Dashboard() {
  // Не передаём user!
  return h('div', null,
    h(Header),
    h(Sidebar)
  );
}

function UserMenu() {
  // Читаем из контекста
  const user = useContext(UserContext);
  return h('div', null, `Привет, ${user.name}!`);
}
```

## Создание контекста

```javascript
/**
 * Создаёт новый контекст
 * @param {any} defaultValue - Значение по умолчанию
 */
export function createContext(defaultValue = undefined) {
  const context = {
    _currentValue: defaultValue,
    Provider: null,
    _id: Symbol('context')
  };

  const valueStack = [];

  // Provider — компонент для предоставления значения
  context.Provider = function Provider({ value, children }) {
    // Сохраняем предыдущее значение в стек,
    // чтобы корректно работать с вложенными провайдерами
    valueStack.push(context._currentValue);
    context._currentValue = value;

    // Рендерим детей
    const result = h(Fragment, null, ...children);

    // После рендера восстанавливаем прошлое значение
    context._currentValue = valueStack.pop();

    return result;
  };

  return context;
}
```

Теперь один Provider не «ломает» значение для соседних веток:
значение кладётся в стек перед рендером и восстанавливается после,
поэтому вложенные провайдеры работают независимо.

## Хук useContext

```javascript
/**
 * Читает значение из контекста
 * @param {Object} context - Контекст
 * @returns {any} Текущее значение контекста
 */
export function useContext(context) {
  if (!currentInstance) {
    throw new Error('useContext can only be called inside a component');
  }

  if (!context || !context._id) {
    throw new Error('Invalid context object');
  }

  // Возвращаем текущее значение
  return context._currentValue;
}
```

## Примеры использования

### Пример 1: Тема приложения

```javascript
const ThemeContext = createContext('light');

function App() {
  const [theme, setTheme] = useState('light');

  return h(ThemeContext.Provider, { value: theme },
    h('div', null,
      h('button', {
        onclick: () => setTheme(theme === 'light' ? 'dark' : 'light')
      }, 'Переключить тему'),
      h(Content)
    )
  );
}

function Content() {
  return h('div', null,
    h(Header),
    h(Article),
    h(Footer)
  );
}

function Article() {
  const theme = useContext(ThemeContext);

  return h('article', {
    style: {
      background: theme === 'light' ? '#fff' : '#333',
      color: theme === 'light' ? '#000' : '#fff',
      padding: '20px'
    }
  }, 'Текст статьи');
}
```

### Пример 2: Аутентификация

```javascript
const AuthContext = createContext(null);

function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  const login = (username, password) => {
    // Имитация API запроса
    setUser({ name: username, role: 'user' });
  };

  const logout = () => {
    setUser(null);
  };

  const value = { user, login, logout };

  return h(AuthContext.Provider, { value }, ...children);
}

function LoginButton() {
  const { user, login, logout } = useContext(AuthContext);

  if (user) {
    return h('button', { onclick: logout }, `Выйти (${user.name})`);
  }

  return h('button', {
    onclick: () => login('Иван', '123')
  }, 'Войти');
}

function UserProfile() {
  const { user } = useContext(AuthContext);

  if (!user) {
    return h('div', null, 'Вы не авторизованы');
  }

  return h('div', null, `Привет, ${user.name}!`);
}

function App() {
  return h(AuthProvider, null,
    h('div', null,
      h(LoginButton),
      h(UserProfile)
    )
  );
}
```

### Пример 3: Язык интерфейса (i18n)

```javascript
const translations = {
  ru: {
    welcome: 'Добро пожаловать',
    login: 'Войти',
    logout: 'Выйти'
  },
  en: {
    welcome: 'Welcome',
    login: 'Login',
    logout: 'Logout'
  }
};

const LanguageContext = createContext('ru');

function LanguageProvider({ children }) {
  const [lang, setLang] = useState('ru');

  const t = (key) => translations[lang][key] || key;

  const value = { lang, setLang, t };

  return h(LanguageContext.Provider, { value }, ...children);
}

function LanguageSwitcher() {
  const { lang, setLang } = useContext(LanguageContext);

  return h('select', {
    value: lang,
    onchange: (e) => setLang(e.target.value)
  },
    h('option', { value: 'ru' }, 'Русский'),
    h('option', { value: 'en' }, 'English')
  );
}

function Greeting() {
  const { t } = useContext(LanguageContext);

  return h('h1', null, t('welcome'));
}

function App() {
  return h(LanguageProvider, null,
    h('div', null,
      h(LanguageSwitcher),
      h(Greeting)
    )
  );
}
```

### Пример 4: Настройки приложения

```javascript
const SettingsContext = createContext({
  fontSize: 16,
  darkMode: false
});

function SettingsProvider({ children }) {
  const [settings, setSettings] = useState({
    fontSize: 16,
    darkMode: false
  });

  const updateSetting = (key, value) => {
    setSettings({ ...settings, [key]: value });
  };

  const value = { settings, updateSetting };

  return h(SettingsContext.Provider, { value }, ...children);
}

function SettingsPanel() {
  const { settings, updateSetting } = useContext(SettingsContext);

  return h('div', null,
    h('label', null,
      'Размер шрифта: ',
      h('input', {
        type: 'range',
        min: 12,
        max: 24,
        value: settings.fontSize,
        oninput: (e) => updateSetting('fontSize', e.target.value)
      }),
      ` ${settings.fontSize}px`
    ),
    h('label', null,
      h('input', {
        type: 'checkbox',
        checked: settings.darkMode,
        onchange: (e) => updateSetting('darkMode', e.target.checked)
      }),
      ' Тёмная тема'
    )
  );
}

function Content() {
  const { settings } = useContext(SettingsContext);

  return h('div', {
    style: {
      fontSize: `${settings.fontSize}px`,
      background: settings.darkMode ? '#222' : '#fff',
      color: settings.darkMode ? '#fff' : '#000',
      padding: '20px'
    }
  }, 'Текст с настраиваемыми параметрами');
}
```

## Множественные контексты

Можно использовать несколько контекстов одновременно:

```javascript
const ThemeContext = createContext('light');
const UserContext = createContext(null);
const LanguageContext = createContext('ru');

function App() {
  const [theme, setTheme] = useState('light');
  const [user, setUser] = useState(null);
  const [lang, setLang] = useState('ru');

  return h(ThemeContext.Provider, { value: theme },
    h(UserContext.Provider, { value: user },
      h(LanguageContext.Provider, { value: lang },
        h(Dashboard)
      )
    )
  );
}

function Dashboard() {
  const theme = useContext(ThemeContext);
  const user = useContext(UserContext);
  const lang = useContext(LanguageContext);

  return h('div', null,
    `Theme: ${theme}, User: ${user?.name || 'Guest'}, Lang: ${lang}`
  );
}
```

## Оптимизация: разделение контекстов

Разделяйте данные и функции в разные контексты для оптимизации:

```javascript
// ❌ Плохо — всё в одном контексте
const AppContext = createContext();

function Provider({ children }) {
  const [state, setState] = useState({});
  // При изменении state все потребители перерисовываются!
  return h(AppContext.Provider, { value: { state, setState } }, ...children);
}

// ✓ Лучше — разделить
const StateContext = createContext();
const DispatchContext = createContext();

function Provider({ children }) {
  const [state, setState] = useState({});

  // Функции не меняются → компоненты, использующие только dispatch, не перерисовываются
  return h(StateContext.Provider, { value: state },
    h(DispatchContext.Provider, { value: setState },
      ...children
    )
  );
}
```

## Паттерн: Custom Provider

Создайте переиспользуемый Provider с логикой:

```javascript
function createStore(initialState, reducer) {
  const StateContext = createContext();
  const DispatchContext = createContext();

  function Provider({ children }) {
    const [state, setState] = useState(initialState);

    const dispatch = (action) => {
      setState(prevState => reducer(prevState, action));
    };

    return h(StateContext.Provider, { value: state },
      h(DispatchContext.Provider, { value: dispatch },
        ...children
      )
    );
  }

  function useState() {
    return useContext(StateContext);
  }

  function useDispatch() {
    return useContext(DispatchContext);
  }

  return { Provider, useState, useDispatch };
}

// Использование
const TodoStore = createStore(
  { todos: [] },
  (state, action) => {
    switch (action.type) {
      case 'ADD':
        return { todos: [...state.todos, action.payload] };
      case 'REMOVE':
        return { todos: state.todos.filter(t => t.id !== action.payload) };
      default:
        return state;
    }
  }
);

function App() {
  return h(TodoStore.Provider, null,
    h(TodoList)
  );
}

function TodoList() {
  const state = TodoStore.useState();
  const dispatch = TodoStore.useDispatch();

  return h('ul', null,
    state.todos.map(todo =>
      h('li', { key: todo.id },
        todo.text,
        h('button', {
          onclick: () => dispatch({ type: 'REMOVE', payload: todo.id })
        }, 'Удалить')
      )
    )
  );
}
```

## Полный пример

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Context Demo</title>
  <style>
    * { box-sizing: border-box; }
    body { font-family: sans-serif; margin: 0; }
    .app {
      min-height: 100vh;
      transition: all 0.3s;
    }
    .controls {
      padding: 20px;
      border-bottom: 2px solid;
    }
    .content {
      padding: 20px;
    }
  </style>
</head>
<body>
  <div id="app"></div>

  <script type="module">
    import { h, render, useState, useContext, createContext, Fragment } from './nano-framework.js';

    // Создаём контекст темы
    const ThemeContext = createContext();

    function ThemeProvider({ children }) {
      const [theme, setTheme] = useState('light');

      const themes = {
        light: {
          bg: '#ffffff',
          text: '#000000',
          border: '#cccccc'
        },
        dark: {
          bg: '#1a1a1a',
          text: '#ffffff',
          border: '#444444'
        }
      };

      const value = {
        theme,
        setTheme,
        colors: themes[theme]
      };

      return h(ThemeContext.Provider, { value }, ...children);
    }

    function ThemeToggle() {
      const { theme, setTheme, colors } = useContext(ThemeContext);

      return h('button', {
        onclick: () => setTheme(theme === 'light' ? 'dark' : 'light'),
        style: {
          padding: '10px 20px',
          background: colors.bg,
          color: colors.text,
          border: `2px solid ${colors.border}`,
          cursor: 'pointer'
        }
      }, theme === 'light' ? 'Тёмная тема' : 'Светлая тема');
    }

    function Header() {
      const { colors } = useContext(ThemeContext);

      return h('div', {
        class: 'controls',
        style: {
          borderColor: colors.border
        }
      },
        h('h1', null, 'Демонстрация Context'),
        h(ThemeToggle)
      );
    }

    function Article() {
      const { colors } = useContext(ThemeContext);

      return h('article', { class: 'content' },
        h('h2', null, 'Статья'),
        h('p', null, 'Тема применяется ко всем компонентам через Context.'),
        h('p', null, 'Промежуточным компонентам не нужно пробрасывать props!')
      );
    }

    function App() {
      const { colors } = useContext(ThemeContext);

      return h('div', {
        class: 'app',
        style: {
          background: colors.bg,
          color: colors.text
        }
      },
        h(Header),
        h(Article)
      );
    }

    function Root() {
      return h(ThemeProvider, null,
        h(App)
      );
    }

    render(h(Root), document.getElementById('app'));
  </script>
</body>
</html>
```

## Когда использовать Context

**Используйте Context для:**
- Данных, нужных многим компонентам (тема, язык, пользователь)
- Избежания props drilling
- Глобального состояния (с осторожностью)

**Не используйте Context для:**
- Часто меняющихся данных (производительность)
- Простых случаев (1-2 уровня вложенности)
- Замены всего управления состоянием

## Задание

1. Создайте NotificationContext для показа уведомлений:
```javascript
// Любой компонент может показать уведомление
const { showNotification } = useContext(NotificationContext);
showNotification('Сохранено!', 'success');
```

2. Реализуйте ModalContext для модальных окон:
```javascript
// Открытие модального окна из любого места
const { openModal, closeModal } = useContext(ModalContext);
openModal(h(MyModal));
```

3. Создайте CartContext для корзины покупок:
```javascript
// Компоненты могут добавлять товары в корзину
const { items, addToCart, removeFromCart } = useContext(CartContext);
```

## Ключевые выводы

- Context решает проблему props drilling
- createContext создаёт контекст с Provider
- useContext читает значение из ближайшего Provider
- Provider передаёт значение всему поддереву
- Можно использовать несколько контекстов
- Разделяйте данные и функции для оптимизации
- Context не заменяет полноценное управление состоянием
- Используйте для глобальных данных (тема, язык, пользователь)

---

[← Предыдущий урок: Lifecycle (useEffect)](./12-lifecycle.md) | [Следующий урок: Система событий →](./14-event-system.md)

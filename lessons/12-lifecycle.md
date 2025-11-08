# Урок 12: Жизненный цикл (useEffect)

## Проблема: побочные эффекты

Компоненты часто нужно взаимодействовать с внешним миром:

```javascript
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  // ❌ Проблема: где делать fetch?
  fetch(`/api/users/${userId}`)
    .then(res => res.json())
    .then(setUser);
  // Вызывается при каждом рендере → бесконечный цикл!

  return h('div', null, user ? user.name : 'Загрузка...');
}
```

**Побочные эффекты (side effects):**
- HTTP запросы
- Подписки на события
- Таймеры (setTimeout, setInterval)
- Работа с localStorage
- Прямая работа с DOM
- Логирование

**Проблема:** когда выполнять эффекты?

## Решение: useEffect

**useEffect** выполняет код **после** рендера компонента:

```javascript
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    // Выполнится ПОСЛЕ рендера
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(setUser);
  }, [userId]); // Зависимости

  return h('div', null, user ? user.name : 'Загрузка...');
}
```

## Синтаксис useEffect

```javascript
useEffect(
  effectFunction,  // Функция эффекта
  dependencies     // Массив зависимостей (опционально)
);
```

**Когда выполняется:**
1. После первого рендера (монтирования)
2. После каждого рендера, если зависимости изменились
3. Возвращаемая функция — cleanup (очистка)

## Реализация useEffect

```javascript
/**
 * Хук для побочных эффектов
 * @param {Function} effect - Функция эффекта
 * @param {Array} deps - Массив зависимостей
 */
export function useEffect(effect, deps) {
  if (!currentComponent) {
    throw new Error('useEffect can only be called inside a component');
  }

  if (!componentStates.has(currentComponent)) {
    componentStates.set(currentComponent, []);
  }

  const hooks = componentStates.get(currentComponent);
  const hookIndex = currentHookIndex;

  // Предыдущий эффект
  const prevEffect = hooks[hookIndex];

  // Проверяем, изменились ли зависимости
  const hasChanged = !prevEffect ||
    !deps ||
    !prevEffect.deps ||
    deps.some((dep, i) => dep !== prevEffect.deps[i]);

  if (hasChanged) {
    // Сохраняем эффект для выполнения после рендера
    hooks[hookIndex] = {
      effect,
      deps,
      cleanup: prevEffect?.cleanup
    };

    // Планируем выполнение эффекта
    scheduleEffect(currentComponent, hookIndex);
  } else {
    // Зависимости не изменились — сохраняем старый эффект
    hooks[hookIndex] = prevEffect;
  }

  currentHookIndex++;
}
```

### Планирование эффектов

```javascript
/**
 * Очередь эффектов для выполнения
 */
const effectQueue = [];

/**
 * Планирует выполнение эффекта
 */
function scheduleEffect(component, hookIndex) {
  effectQueue.push({ component, hookIndex });

  // Выполняем эффекты после рендера
  if (!isEffectScheduled) {
    isEffectScheduled = true;
    Promise.resolve().then(flushEffects);
  }
}

let isEffectScheduled = false;

/**
 * Выполняет все запланированные эффекты
 */
function flushEffects() {
  isEffectScheduled = false;

  effectQueue.forEach(({ component, hookIndex }) => {
    const hooks = componentStates.get(component);
    const hook = hooks[hookIndex];

    if (hook) {
      // Вызываем cleanup предыдущего эффекта
      if (hook.cleanup) {
        hook.cleanup();
      }

      // Выполняем новый эффект
      const cleanup = hook.effect();

      // Сохраняем cleanup функцию
      if (typeof cleanup === 'function') {
        hook.cleanup = cleanup;
      }
    }
  });

  effectQueue.length = 0;
}
```

## Варианты использования useEffect

### 1. Без зависимостей — при каждом рендере

```javascript
useEffect(() => {
  console.log('Компонент отрендерился');
});
// Выполняется после каждого рендера
```

### 2. Пустой массив зависимостей — только при монтировании

```javascript
useEffect(() => {
  console.log('Компонент смонтирован');
}, []);
// Выполняется только один раз при монтировании
```

### 3. С зависимостями — при изменении зависимостей

```javascript
useEffect(() => {
  console.log(`userId изменился: ${userId}`);
}, [userId]);
// Выполняется при изменении userId
```

## Cleanup функция

Для очистки ресурсов возвращаем функцию:

```javascript
useEffect(() => {
  // Настройка
  const timer = setInterval(() => {
    console.log('Tick');
  }, 1000);

  // Cleanup — вызывается при размонтировании или перед новым эффектом
  return () => {
    clearInterval(timer);
  };
}, []);
```

## Примеры использования

### Пример 1: Загрузка данных

```javascript
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    setLoading(true);
    setError(null);

    fetch(`/api/users/${userId}`)
      .then(res => {
        if (!res.ok) throw new Error('Failed to fetch');
        return res.json();
      })
      .then(data => {
        setUser(data);
        setLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setLoading(false);
      });
  }, [userId]); // Перезагружаем при изменении userId

  if (loading) return h('div', null, 'Загрузка...');
  if (error) return h('div', null, `Ошибка: ${error}`);

  return h('div', null,
    h('h2', null, user.name),
    h('p', null, user.email)
  );
}
```

### Пример 2: Подписка на события

```javascript
function WindowSize() {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight
  });

  useEffect(() => {
    function handleResize() {
      setSize({
        width: window.innerWidth,
        height: window.innerHeight
      });
    }

    // Подписываемся
    window.addEventListener('resize', handleResize);

    // Cleanup — отписываемся
    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []); // Пустой массив — подписка один раз

  return h('div', null,
    `Размер окна: ${size.width} x ${size.height}`
  );
}
```

### Пример 3: Таймер

```javascript
function Timer() {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    if (!isRunning) return;

    const interval = setInterval(() => {
      setSeconds(s => s + 1);
    }, 1000);

    return () => clearInterval(interval);
  }, [isRunning]); // Перезапускаем при изменении isRunning

  return h('div', null,
    h('h2', null, `Время: ${seconds}с`),
    h('button', {
      onclick: () => setIsRunning(!isRunning)
    }, isRunning ? 'Пауза' : 'Старт'),
    h('button', {
      onclick: () => {
        setSeconds(0);
        setIsRunning(false);
      }
    }, 'Сброс')
  );
}
```

### Пример 4: localStorage

```javascript
function useLocalStorage(key, initialValue) {
  // Читаем из localStorage при монтировании
  const [value, setValue] = useState(() => {
    const saved = localStorage.getItem(key);
    return saved ? JSON.parse(saved) : initialValue;
  });

  // Сохраняем в localStorage при изменении
  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}

function Counter() {
  // Состояние автоматически сохраняется в localStorage
  const [count, setCount] = useLocalStorage('count', 0);

  return h('div', null,
    h('p', null, count),
    h('button', { onclick: () => setCount(count + 1) }, '+')
  );
}
```

### Пример 5: Заголовок документа

```javascript
function DocumentTitle({ title }) {
  useEffect(() => {
    const prevTitle = document.title;
    document.title = title;

    // Восстанавливаем при размонтировании
    return () => {
      document.title = prevTitle;
    };
  }, [title]);

  return null; // Ничего не рендерим
}

function App() {
  const [page, setPage] = useState('home');

  return h('div', null,
    h(DocumentTitle, { title: `Страница: ${page}` }),
    h('button', { onclick: () => setPage('about') }, 'О нас'),
    h('h1', null, page)
  );
}
```

### Пример 6: Фокус на input

```javascript
function AutoFocusInput() {
  const [inputRef, setInputRef] = useState(null);

  useEffect(() => {
    if (inputRef) {
      inputRef.focus();
    }
  }, [inputRef]);

  return h('input', {
    ref: (el) => setInputRef(el),
    type: 'text',
    placeholder: 'Автоматический фокус'
  });
}
```

## Множественные эффекты

Можно использовать несколько useEffect:

```javascript
function UserDashboard({ userId }) {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);

  // Эффект 1: загрузка пользователя
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(setUser);
  }, [userId]);

  // Эффект 2: загрузка постов
  useEffect(() => {
    fetch(`/api/users/${userId}/posts`)
      .then(res => res.json())
      .then(setPosts);
  }, [userId]);

  // Эффект 3: обновление заголовка
  useEffect(() => {
    if (user) {
      document.title = `Профиль: ${user.name}`;
    }
  }, [user]);

  return h('div', null,
    user && h('h1', null, user.name),
    h('ul', null, posts.map(post =>
      h('li', { key: post.id }, post.title)
    ))
  );
}
```

## Правила useEffect

### 1. Всегда указывайте зависимости

```javascript
// ❌ Плохо — забыли зависимость
useEffect(() => {
  console.log(userId);
}, []); // userId может измениться!

// ✓ Хорошо
useEffect(() => {
  console.log(userId);
}, [userId]);
```

### 2. Не забывайте cleanup

```javascript
// ❌ Утечка памяти
useEffect(() => {
  const interval = setInterval(() => console.log('tick'), 1000);
}, []);

// ✓ Правильно
useEffect(() => {
  const interval = setInterval(() => console.log('tick'), 1000);
  return () => clearInterval(interval);
}, []);
```

### 3. Избегайте бесконечных циклов

```javascript
// ❌ Бесконечный цикл
const [count, setCount] = useState(0);
useEffect(() => {
  setCount(count + 1); // Меняет count → эффект снова выполняется
});

// ✓ Правильно — с условием
useEffect(() => {
  if (count < 10) {
    setCount(count + 1);
  }
}, [count]);
```

## Полный пример: Live поиск

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>useEffect Demo</title>
  <style>
    body {
      font-family: sans-serif;
      max-width: 600px;
      margin: 40px auto;
    }
    .search-box {
      margin: 20px 0;
    }
    .search-box input {
      width: 100%;
      padding: 10px;
      font-size: 16px;
      border: 2px solid #ccc;
      border-radius: 4px;
    }
    .result {
      padding: 10px;
      border-bottom: 1px solid #eee;
    }
    .loading {
      color: #666;
      font-style: italic;
    }
  </style>
</head>
<body>
  <div id="app"></div>

  <script type="module">
    import { h, render, useState, useEffect } from './nano-framework.js';

    // Имитация API
    function searchUsers(query) {
      return new Promise(resolve => {
        setTimeout(() => {
          const users = [
            { id: 1, name: 'Иван Иванов' },
            { id: 2, name: 'Мария Петрова' },
            { id: 3, name: 'Пётр Сидоров' },
            { id: 4, name: 'Анна Смирнова' }
          ];

          const filtered = users.filter(user =>
            user.name.toLowerCase().includes(query.toLowerCase())
          );

          resolve(filtered);
        }, 500);
      });
    }

    function SearchBar() {
      const [query, setQuery] = useState('');
      const [results, setResults] = useState([]);
      const [loading, setLoading] = useState(false);

      useEffect(() => {
        // Не ищем пустую строку
        if (!query.trim()) {
          setResults([]);
          return;
        }

        setLoading(true);

        // Debounce — отменяем предыдущий запрос
        const timer = setTimeout(() => {
          searchUsers(query).then(users => {
            setResults(users);
            setLoading(false);
          });
        }, 300);

        // Cleanup — отменяем таймер при новом вводе
        return () => clearTimeout(timer);
      }, [query]);

      return h('div', null,
        h('div', { class: 'search-box' },
          h('input', {
            type: 'text',
            value: query,
            placeholder: 'Поиск пользователей...',
            oninput: (e) => setQuery(e.target.value)
          })
        ),

        loading && h('div', { class: 'loading' }, 'Поиск...'),

        h('div', null,
          results.map(user =>
            h('div', { class: 'result', key: user.id },
              h('strong', null, user.name)
            )
          )
        ),

        !loading && query && results.length === 0 &&
          h('div', null, 'Ничего не найдено')
      );
    }

    function App() {
      return h('div', null,
        h('h1', null, 'Live поиск'),
        h(SearchBar)
      );
    }

    render(h(App), document.getElementById('app'));
  </script>
</body>
</html>
```

## Паттерны useEffect

### Debouncing

```javascript
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

### Интервал

```javascript
function useInterval(callback, delay) {
  useEffect(() => {
    if (delay === null) return;

    const interval = setInterval(callback, delay);
    return () => clearInterval(interval);
  }, [callback, delay]);
}
```

## Задание

1. Создайте компонент с живыми часами:
```javascript
function Clock() {
  // Обновляйте время каждую секунду
}
```

2. Реализуйте компонент с автосохранением:
```javascript
function AutoSaveForm() {
  // Сохраняйте в localStorage через 2 секунды после изменения
}
```

3. Создайте компонент для отслеживания позиции мыши:
```javascript
function MouseTracker() {
  // Показывайте координаты мыши
}
```

## Ключевые выводы

- useEffect для побочных эффектов после рендера
- Второй аргумент — массив зависимостей
- Пустой массив [] — эффект один раз при монтировании
- Без массива — эффект при каждом рендере
- Cleanup функция для очистки ресурсов
- Всегда указывайте все зависимости
- Cleanup вызывается перед новым эффектом и при размонтировании
- Используйте несколько useEffect для разделения логики

---

[← Предыдущий урок: Состояние (useState)](./11-state.md) | [Следующий урок: Контекст →](./13-context.md)

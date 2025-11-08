# Приложение B: Дальнейшее развитие

## Поздравляем!

Вы создали полноценный JavaScript-фреймворк с Virtual DOM! Это серьёзное достижение. Но путь разработки фреймворка бесконечен — всегда есть что улучшить.

В этом приложении мы рассмотрим идеи для дальнейшего развития NanoFramework.

---

## 1. JSX Support

### Проблема
Писать `h()` вручную утомительно:
```javascript
h('div', { class: 'card' },
  h('h2', null, title),
  h('p', null, description)
)
```

### Решение: JSX
```jsx
<div className="card">
  <h2>{title}</h2>
  <p>{description}</p>
</div>
```

### Как добавить

1. **Настроить Babel**

```json
// .babelrc
{
  "plugins": [
    ["@babel/plugin-transform-react-jsx", {
      "pragma": "h",
      "pragmaFrag": "Fragment"
    }]
  ]
}
```

2. **Или использовать esbuild**

```javascript
// build.js
import esbuild from 'esbuild';

esbuild.build({
  entryPoints: ['src/app.jsx'],
  bundle: true,
  jsxFactory: 'h',
  jsxFragment: 'Fragment',
  outfile: 'dist/app.js'
});
```

3. **Реализовать Fragment**

```javascript
export const Fragment = Symbol('Fragment');

function h(type, props, ...children) {
  if (type === Fragment) {
    return {
      type: Fragment,
      props: {},
      children: flattenChildren(children)
    };
  }
  // ... остальное
}
```

### Преимущества
- Более читаемый код
- Синтаксическая подсветка в IDE
- Привычно для React-разработчиков

---

## 2. TypeScript

### Проблема
Нет типизации — легко допустить ошибку:
```javascript
h('button', { onclik: handler }) // Опечатка!
```

### Решение: Определения типов

```typescript
// nano-framework.d.ts
export interface VNode {
  type: string | Function | symbol;
  props: Props;
  children: Array<VNode | string>;
}

export interface Props {
  [key: string]: any;
  className?: string;
  style?: React.CSSProperties;
  onClick?: (e: MouseEvent) => void;
  // ... остальные события
}

export function h(
  type: string | Function,
  props: Props | null,
  ...children: Array<VNode | string | number | boolean | null | undefined>
): VNode;

export function useState<T>(
  initialValue: T | (() => T)
): [T, (value: T | ((prev: T) => T)) => void];

export function useEffect(
  effect: () => void | (() => void),
  deps?: any[]
): void;

// ... остальные хуки
```

### JSX типизация

```typescript
// jsx.d.ts
declare namespace JSX {
  interface IntrinsicElements {
    div: Props;
    span: Props;
    button: Props & { type?: 'button' | 'submit' | 'reset' };
    input: Props & {
      type?: string;
      value?: string;
      checked?: boolean;
    };
    // ... все HTML элементы
  }

  interface Element extends VNode {}
}
```

### Преимущества
- Автодополнение в IDE
- Проверка типов на этапе компиляции
- Меньше ошибок

---

## 3. Дополнительные хуки

### useMemo

Мемоизация вычислений:

```javascript
export function useMemo(factory, deps) {
  const hook = getCurrentHook();

  if (!hook.value || !areDepsEqual(hook.deps, deps)) {
    hook.value = factory();
    hook.deps = deps;
  }

  return hook.value;
}
```

Использование:
```javascript
function ExpensiveComponent({ items }) {
  const total = useMemo(() => {
    console.log('Пересчёт...');
    return items.reduce((sum, item) => sum + item.price, 0);
  }, [items]);

  return h('div', null, `Итого: ${total}`);
}
```

### useCallback

Мемоизация функций:

```javascript
export function useCallback(callback, deps) {
  return useMemo(() => callback, deps);
}
```

Использование:
```javascript
function Parent() {
  const [count, setCount] = useState(0);

  const handleClick = useCallback(() => {
    console.log('Клик!');
  }, []); // Функция создаётся один раз

  return h(Child, { onClick: handleClick });
}
```

### useReducer

Альтернатива useState для сложного состояния:

```javascript
export function useReducer(reducer, initialState) {
  const [state, setState] = useState(initialState);

  const dispatch = useCallback((action) => {
    setState(prev => reducer(prev, action));
  }, [reducer]);

  return [state, dispatch];
}
```

Использование:
```javascript
function todoReducer(state, action) {
  switch (action.type) {
    case 'add':
      return [...state, action.todo];
    case 'remove':
      return state.filter(t => t.id !== action.id);
    default:
      return state;
  }
}

function TodoList() {
  const [todos, dispatch] = useReducer(todoReducer, []);

  const addTodo = (text) => {
    dispatch({ type: 'add', todo: { id: Date.now(), text } });
  };

  return h('div', null, /* ... */);
}
```

### useLayoutEffect

Синхронный эффект (выполняется до отрисовки):

```javascript
export function useLayoutEffect(effect, deps) {
  const hook = getCurrentHook();

  const hasChanged = !hook.deps || !areDepsEqual(hook.deps, deps);

  if (hasChanged) {
    // Cleanup
    if (hook.cleanup) {
      hook.cleanup();
    }

    // Выполняем синхронно (до paint)
    hook.cleanup = effect();
    hook.deps = deps;
  }
}
```

---

## 4. Мемоизация компонентов (memo)

### Проблема
Компонент перерисовывается даже если props не изменились:

```javascript
function Child({ name }) {
  console.log('Child render');
  return h('div', null, name);
}

function Parent() {
  const [count, setCount] = useState(0);
  return h('div', null,
    h(Child, { name: 'John' }), // Рендерится при каждом изменении count!
    h('button', { onclick: () => setCount(count + 1) }, count)
  );
}
```

### Решение: memo()

```javascript
export function memo(Component, arePropsEqual) {
  const MemoizedComponent = (props) => {
    const prevPropsRef = useRef(null);
    const prevVNodeRef = useRef(null);

    const shouldUpdate = !prevPropsRef.current ||
      !(arePropsEqual || shallowEqual)(prevPropsRef.current, props);

    if (shouldUpdate) {
      prevVNodeRef.current = Component(props);
      prevPropsRef.current = props;
    }

    return prevVNodeRef.current;
  };

  return MemoizedComponent;
}

function shallowEqual(a, b) {
  const keysA = Object.keys(a);
  const keysB = Object.keys(b);

  if (keysA.length !== keysB.length) return false;

  return keysA.every(key => a[key] === b[key]);
}
```

Использование:
```javascript
const MemoizedChild = memo(Child);

function Parent() {
  const [count, setCount] = useState(0);
  return h('div', null,
    h(MemoizedChild, { name: 'John' }), // Рендерится только при изменении name
    h('button', { onclick: () => setCount(count + 1) }, count)
  );
}
```

---

## 5. Portal

### Проблема
Нужно рендерить компонент вне родительского дерева (модалки, тултипы):

```html
<div id="app">
  <!-- Обычный контент -->
</div>
<div id="modal-root">
  <!-- Модалка должна быть здесь -->
</div>
```

### Решение: createPortal

```javascript
export function createPortal(children, container) {
  return {
    type: PORTAL,
    props: { container },
    children: Array.isArray(children) ? children : [children]
  };
}

const PORTAL = Symbol('Portal');

// В createElement:
function createElement(vNode) {
  // ...
  if (vNode.type === PORTAL) {
    const fragment = document.createDocumentFragment();
    vNode.children.forEach(child => {
      fragment.appendChild(createElement(child));
    });
    vNode.props.container.appendChild(fragment);
    return fragment;
  }
  // ...
}
```

Использование:
```javascript
function Modal({ children, isOpen }) {
  if (!isOpen) return null;

  return createPortal(
    h('div', { class: 'modal-overlay' },
      h('div', { class: 'modal' }, children)
    ),
    document.getElementById('modal-root')
  );
}
```

---

## 6. Error Boundaries

### Проблема
Ошибка в компоненте ломает всё приложение.

### Решение: ErrorBoundary компонент

```javascript
class ErrorBoundary {
  constructor(props) {
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    console.error('Error caught:', error, errorInfo);
    this.props.onError?.(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback(this.state.error);
    }
    return this.props.children;
  }
}
```

Но у нас функциональные компоненты! Нужна адаптация...

**Альтернатива**: useErrorHandler хук

```javascript
export function useErrorHandler() {
  const [error, setError] = useState(null);

  if (error) throw error;

  return setError;
}

// Использование
function Component() {
  const handleError = useErrorHandler();

  useEffect(() => {
    fetchData().catch(handleError);
  }, []);

  return h('div', null, '...');
}
```

---

## 7. Suspense и Lazy Loading

### Проблема
Нужно показывать загрузку пока компонент грузится.

### Решение: Suspense (упрощённая версия)

```javascript
const LOADING = Symbol('Loading');

export function lazy(loader) {
  let Component = null;
  let promise = null;

  return (props) => {
    if (!Component) {
      if (!promise) {
        promise = loader().then(module => {
          Component = module.default || module;
        });
      }
      throw promise; // Бросаем Promise!
    }

    return h(Component, props);
  };
}

export function Suspense({ fallback, children }) {
  const [isLoading, setIsLoading] = useState(false);

  try {
    return children;
  } catch (promise) {
    if (promise instanceof Promise) {
      setIsLoading(true);
      promise.finally(() => setIsLoading(false));
      return fallback;
    }
    throw promise;
  }
}
```

Использование:
```javascript
const LazyComponent = lazy(() => import('./HeavyComponent.js'));

function App() {
  return h(Suspense, { fallback: h('div', null, 'Загрузка...') },
    h(LazyComponent)
  );
}
```

---

## 8. Server-Side Rendering (SSR)

### Цель
Рендерить компоненты в HTML на сервере для:
- SEO
- Быстрой первой отрисовки
- Доступности

### Реализация: renderToString

```javascript
export function renderToString(vNode) {
  // Текстовый узел
  if (typeof vNode === 'string') {
    return escapeHtml(vNode);
  }

  // Компонент
  if (typeof vNode.type === 'function') {
    return renderToString(vNode.type(vNode.props));
  }

  // Элемент
  const { type, props, children } = vNode;

  // Открывающий тег
  let html = `<${type}`;

  // Атрибуты
  Object.keys(props).forEach(key => {
    if (key.startsWith('on')) return; // Пропускаем события
    const value = props[key];
    if (value != null) {
      html += ` ${key}="${escapeHtml(String(value))}"`;
    }
  });

  html += '>';

  // Дети
  children.forEach(child => {
    html += renderToString(child);
  });

  // Закрывающий тег
  html += `</${type}>`;

  return html;
}

function escapeHtml(str) {
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;');
}
```

Использование:
```javascript
// server.js (Node.js)
import { renderToString } from './nano-framework.js';
import { App } from './App.js';

const html = `
<!DOCTYPE html>
<html>
  <head><title>App</title></head>
  <body>
    <div id="app">${renderToString(h(App))}</div>
    <script src="/client.js"></script>
  </body>
</html>
`;
```

### Hydration
На клиенте нужно "оживить" статический HTML:

```javascript
export function hydrate(vNode, container) {
  // Прикрепляем обработчики событий к существующим элементам
  // вместо пересоздания DOM
}
```

---

## 9. Роутинг

### Простой роутер

```javascript
export function createRouter(routes) {
  let currentRoute = null;
  const listeners = [];

  const router = {
    push(path) {
      window.history.pushState(null, '', path);
      notifyListeners();
    },

    subscribe(listener) {
      listeners.push(listener);
      return () => {
        const index = listeners.indexOf(listener);
        listeners.splice(index, 1);
      };
    },

    getRoute() {
      const path = window.location.pathname;
      for (const route of routes) {
        const match = matchPath(path, route.path);
        if (match) {
          return { ...route, params: match.params };
        }
      }
      return null;
    }
  };

  function notifyListeners() {
    currentRoute = router.getRoute();
    listeners.forEach(fn => fn(currentRoute));
  }

  window.addEventListener('popstate', notifyListeners);

  return router;
}

function matchPath(pathname, pattern) {
  // Простая реализация
  const regex = new RegExp('^' + pattern.replace(/:(\w+)/g, '(?<$1>[^/]+)') + '$');
  const match = pathname.match(regex);
  return match ? { params: match.groups || {} } : null;
}
```

### Компоненты роутера

```javascript
const RouterContext = createContext(null);

export function Router({ router, children }) {
  const [route, setRoute] = useState(router.getRoute());

  useEffect(() => {
    return router.subscribe(setRoute);
  }, [router]);

  return h(RouterContext.Provider, { value: { router, route } }, children);
}

export function Route({ path, component: Component }) {
  const { route } = useContext(RouterContext);

  if (!route || route.path !== path) {
    return null;
  }

  return h(Component, route.params);
}

export function Link({ to, children }) {
  const { router } = useContext(RouterContext);

  const handleClick = (e) => {
    e.preventDefault();
    router.push(to);
  };

  return h('a', { href: to, onclick: handleClick }, children);
}
```

Использование:
```javascript
const routes = [
  { path: '/', component: Home },
  { path: '/about', component: About },
  { path: '/users/:id', component: UserProfile }
];

const router = createRouter(routes);

function App() {
  return h(Router, { router },
    h('nav', null,
      h(Link, { to: '/' }, 'Главная'),
      h(Link, { to: '/about' }, 'О нас')
    ),
    h(Route, { path: '/', component: Home }),
    h(Route, { path: '/about', component: About }),
    h(Route, { path: '/users/:id', component: UserProfile })
  );
}
```

---

## 10. DevTools

### Идеи для инструментов разработчика

1. **Component Tree Inspector**
   - Показывать дерево компонентов
   - Props каждого компонента
   - State

2. **Time Travel Debugging**
   - История изменений state
   - Возможность "откатиться"

3. **Performance Profiler**
   - Измерение времени рендера
   - Выявление медленных компонентов

### Простая реализация логирования

```javascript
export let __DEBUG__ = false;

export function enableDebug() {
  __DEBUG__ = true;
}

// В коде фреймворка:
function render(vNode) {
  if (__DEBUG__) {
    console.time('Render');
  }

  // ... рендеринг

  if (__DEBUG__) {
    console.timeEnd('Render');
    console.log('VNode tree:', vNode);
  }
}
```

---

## 11. Concurrent Mode (продвинуто)

### Идея
Разбивать тяжёлую работу на части, чтобы не блокировать UI.

```javascript
// Упрощённая концепция
function scheduleWork(callback) {
  if ('requestIdleCallback' in window) {
    requestIdleCallback(callback);
  } else {
    setTimeout(callback, 0);
  }
}

export function startTransition(callback) {
  scheduleWork(() => {
    callback();
  });
}
```

Это сложная тема, требующая переписывания reconciler!

---

## 12. CSS-in-JS

### Styled components

```javascript
export function styled(tag) {
  return (styles) => {
    return (props) => {
      const className = generateClassName(styles, props);
      return h(tag, { ...props, class: className }, props.children);
    };
  };
}

const Button = styled('button')`
  background: ${props => props.primary ? 'blue' : 'gray'};
  color: white;
  padding: 10px;
`;

// Использование
h(Button, { primary: true }, 'Click me')
```

---

## 13. Тестирование

### Утилиты для тестов

```javascript
export function renderToTest(vNode) {
  const container = document.createElement('div');
  mount(vNode, container);
  return {
    container,
    getByText(text) {
      return container.querySelector(`*:contains('${text}')`);
    },
    rerender(newVNode) {
      mount(newVNode, container);
    },
    unmount() {
      container.innerHTML = '';
    }
  };
}
```

Использование с Jest:
```javascript
test('Counter increments', () => {
  const { container, getByText } = renderToTest(h(Counter));

  const button = getByText('+');
  button.click();

  expect(getByText('Count: 1')).toBeInTheDocument();
});
```

---

## Roadmap развития

### Фаза 1: Стабильность
- [ ] Полное покрытие тестами
- [ ] Исправление багов
- [ ] Документация

### Фаза 2: Developer Experience
- [ ] JSX support
- [ ] TypeScript типы
- [ ] DevTools
- [ ] Better error messages

### Фаза 3: Производительность
- [ ] useMemo / useCallback
- [ ] memo()
- [ ] Оптимизация diffing
- [ ] Code splitting

### Фаза 4: Экосистема
- [ ] Роутер
- [ ] State management
- [ ] Form библиотека
- [ ] Animation библиотека

### Фаза 5: Enterprise
- [ ] SSR
- [ ] Suspense
- [ ] Concurrent Mode
- [ ] TypeScript переписывание

---

## Заключение

Вы прошли путь от нуля до создания собственного фреймворка! Теперь вы понимаете:

✅ Как работает Virtual DOM
✅ Зачем нужны hooks
✅ Как работает diffing и reconciliation
✅ Как устроены современные фреймворки
✅ Какие проблемы они решают

### Что дальше?

1. **Используйте знания**: Вы теперь понимаете React/Vue глубже
2. **Экспериментируйте**: Добавляйте фичи в NanoFramework
3. **Делитесь**: Опубликуйте на GitHub, напишите статью
4. **Учитесь дальше**: Изучите исходники React, Vue, Preact

### Полезные ресурсы

- [React Internals](https://github.com/facebook/react)
- [Vue Source Code](https://github.com/vuejs/core)
- [Preact](https://github.com/preactjs/preact) — маленький, читаемый
- [Million.js](https://million.dev/) — оптимизированный VDOM
- [Build Your Own React](https://pomb.us/build-your-own-react/)

---

**Спасибо за прохождение курса!**

Удачи в разработке! 🚀

---

[← Предыдущее приложение: Сравнение с React/Vue](./a-comparison.md)

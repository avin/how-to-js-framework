# Урок 16: Reconciliation (согласование)

## Что такое reconciliation?

**Reconciliation** — это процесс обновления DOM путём сравнения нового виртуального дерева со старым и применения минимального набора изменений.

Мы уже реализовали основы:
- Diff — сравнение виртуальных деревьев
- Patch — применение изменений к DOM
- Keys — оптимизация списков

Теперь разберём полный процесс и оптимизации.

## Жизненный цикл обновления

```
1. Событие (клик, ввод, таймер)
        ↓
2. Обновление состояния (setState)
        ↓
3. Планирование перерисовки (batching)
        ↓
4. Вызов функции компонента → новый vNode
        ↓
5. Reconciliation (diff + patch)
        ↓
6. Коммит изменений в DOM
        ↓
7. Выполнение эффектов (useEffect)
```

## Fiber Architecture (React 16+)

React 16 ввёл **Fiber** — новую архитектуру reconciliation.

### Проблема старого подхода

```javascript
// Старый алгоритм — синхронный
function reconcile(oldTree, newTree) {
  diff(oldTree, newTree);        // Блокирует основной поток
  patch(container, patches);      // Не прерывается
  // Если дерево большое — браузер "зависает"
}
```

### Решение: Fiber

**Fiber** разбивает работу на маленькие единицы (fibers), которые можно прерывать:

```javascript
// Концептуальная схема Fiber
class Fiber {
  constructor(vNode) {
    this.vNode = vNode;
    this.child = null;      // Первый ребёнок
    this.sibling = null;    // Следующий сосед
    this.parent = null;     // Родитель
    this.alternate = null;  // Старый fiber (для сравнения)
    this.effectTag = null;  // Тип изменения
  }
}
```

### Фазы Fiber

**1. Render фаза** (прерываемая):
- Создание fiber дерева
- Diff и вычисление изменений
- Можно прервать для обработки приоритетных задач

**2. Commit фаза** (неприрываемая):
- Применение изменений к DOM
- Выполняется атомарно
- Нельзя прервать

## Упрощённая реализация Fiber

```javascript
/**
 * Рабочая единица (fiber)
 */
class WorkUnit {
  constructor(type, props) {
    this.type = type;
    this.props = props;
    this.dom = null;
    this.parent = null;
    this.child = null;
    this.sibling = null;
    this.alternate = null;
    this.effectTag = null;
  }
}

/**
 * Создаёт fiber из vNode
 */
function createFiber(vNode, parent, alternate) {
  const fiber = new WorkUnit(vNode.type, vNode.props);
  fiber.parent = parent;
  fiber.alternate = alternate;

  return fiber;
}

/**
 * Текущая рабочая единица
 */
let workInProgress = null;

/**
 * Корень fiber дерева
 */
let workInProgressRoot = null;

/**
 * Текущее (старое) дерево
 */
let currentRoot = null;

/**
 * Планирует работу
 */
export function scheduleWork(vNode, container) {
  workInProgressRoot = {
    dom: container,
    props: { children: [vNode] },
    alternate: currentRoot
  };

  workInProgress = workInProgressRoot;

  // Запускаем работу в следующем idle
  requestIdleCallback(performWork);
}

/**
 * Выполняет работу порциями
 */
function performWork(deadline) {
  while (workInProgress && deadline.timeRemaining() > 1) {
    workInProgress = performUnitOfWork(workInProgress);
  }

  // Если работа закончена — коммитим
  if (!workInProgress && workInProgressRoot) {
    commitRoot();
  }

  // Если остались задачи — продолжаем
  if (workInProgress) {
    requestIdleCallback(performWork);
  }
}

/**
 * Выполняет одну единицу работы
 */
function performUnitOfWork(fiber) {
  // 1. Обрабатываем текущий fiber
  if (!fiber.dom && fiber.type) {
    fiber.dom = createElement(fiber);
  }

  // 2. Создаём fibers для детей
  reconcileChildren(fiber, fiber.props.children);

  // 3. Возвращаем следующую единицу работы
  if (fiber.child) {
    return fiber.child;
  }

  let nextFiber = fiber;
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling;
    }
    nextFiber = nextFiber.parent;
  }

  return null;
}

/**
 * Reconciliation детей
 */
function reconcileChildren(fiber, children) {
  let oldFiber = fiber.alternate?.child;
  let prevSibling = null;

  children.forEach((child, index) => {
    const newFiber = createFiber(child, fiber, oldFiber);

    // Определяем effectTag (тип изменения)
    if (oldFiber) {
      if (oldFiber.type === newFiber.type) {
        newFiber.effectTag = 'UPDATE';
      } else {
        newFiber.effectTag = 'REPLACE';
      }
    } else {
      newFiber.effectTag = 'PLACEMENT';
    }

    // Связываем siblings
    if (index === 0) {
      fiber.child = newFiber;
    } else {
      prevSibling.sibling = newFiber;
    }

    prevSibling = newFiber;
    oldFiber = oldFiber?.sibling;
  });
}

/**
 * Коммит изменений в DOM
 */
function commitRoot() {
  commitWork(workInProgressRoot.child);
  currentRoot = workInProgressRoot;
  workInProgressRoot = null;
}

/**
 * Коммит одного fiber
 */
function commitWork(fiber) {
  if (!fiber) return;

  const parentDOM = fiber.parent.dom;

  if (fiber.effectTag === 'PLACEMENT' && fiber.dom) {
    parentDOM.appendChild(fiber.dom);
  } else if (fiber.effectTag === 'UPDATE' && fiber.dom) {
    updateDom(fiber.dom, fiber.alternate.props, fiber.props);
  } else if (fiber.effectTag === 'DELETION') {
    parentDOM.removeChild(fiber.dom);
  }

  commitWork(fiber.child);
  commitWork(fiber.sibling);
}
```

## Приоритеты обновлений

Разные обновления имеют разный приоритет:

```javascript
const Priority = {
  IMMEDIATE: 1,      // Клики, ввод текста
  USER_BLOCKING: 2,  // Ховер, скролл
  NORMAL: 3,         // Сетевые запросы
  LOW: 4,            // Аналитика
  IDLE: 5            // Неважные обновления
};

/**
 * Планировщик с приоритетами
 */
class Scheduler {
  constructor() {
    this.taskQueue = [];
  }

  schedule(callback, priority = Priority.NORMAL) {
    const task = { callback, priority, time: Date.now() };

    // Вставляем по приоритету
    const index = this.taskQueue.findIndex(t => t.priority > priority);
    if (index === -1) {
      this.taskQueue.push(task);
    } else {
      this.taskQueue.splice(index, 0, task);
    }

    this.flush();
  }

  flush() {
    if (this.flushing) return;
    this.flushing = true;

    requestIdleCallback((deadline) => {
      while (this.taskQueue.length > 0 && deadline.timeRemaining() > 1) {
        const task = this.taskQueue.shift();
        task.callback();
      }

      this.flushing = false;

      if (this.taskQueue.length > 0) {
        this.flush();
      }
    });
  }
}

const scheduler = new Scheduler();

// Использование
function handleClick() {
  scheduler.schedule(() => {
    setState(newValue);
  }, Priority.IMMEDIATE);
}

function loadData() {
  scheduler.schedule(() => {
    fetchData();
  }, Priority.NORMAL);
}
```

## Concurrent Mode (React 18+)

**Concurrent Mode** позволяет React работать над несколькими обновлениями одновременно:

```javascript
// Автоматические transitions
function SearchResults() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  // Низкоприоритетное обновление
  function handleChange(e) {
    setQuery(e.target.value); // Высокий приоритет (ввод)

    startTransition(() => {
      searchAPI(e.target.value).then(setResults); // Низкий приоритет
    });
  }

  return h('div', null,
    h('input', { value: query, oninput: handleChange }),
    h('ul', null, results.map(r => h('li', { key: r.id }, r.text)))
  );
}
```

## Тайминг обновлений

### requestIdleCallback

Выполняет работу когда браузер простаивает:

```javascript
function performWork(deadline) {
  let shouldYield = false;

  while (workInProgress && !shouldYield) {
    workInProgress = performUnitOfWork(workInProgress);
    shouldYield = deadline.timeRemaining() < 1;
  }

  if (workInProgress) {
    requestIdleCallback(performWork);
  } else {
    commitRoot();
  }
}

requestIdleCallback(performWork);
```

### requestAnimationFrame

Для анимаций — синхронизация с частотой обновления экрана (60 FPS):

```javascript
function animationUpdate() {
  // Обновление анимации
  updateAnimationState();

  if (isAnimating) {
    requestAnimationFrame(animationUpdate);
  }
}

requestAnimationFrame(animationUpdate);
```

## Оптимизации reconciliation

### 1. Bail out (ранний выход)

Если компонент не изменился — пропускаем:

```javascript
function reconcileComponent(fiber, vNode) {
  // Если props не изменились
  if (shallowEqual(fiber.props, vNode.props)) {
    return fiber.child; // Пропускаем детей
  }

  // Обычная reconciliation
  return performWork(fiber);
}
```

### 2. Memo (мемоизация)

Кэширование результата рендера:

```javascript
export function memo(Component, arePropsEqual = shallowEqual) {
  let prevProps = null;
  let prevResult = null;

  return function MemoizedComponent(props) {
    if (prevProps && arePropsEqual(prevProps, props)) {
      return prevResult; // Возвращаем закэшированный результат
    }

    prevProps = props;
    prevResult = Component(props);
    return prevResult;
  };
}

// Использование
const ExpensiveList = memo(({ items }) => {
  console.log('Рендер ExpensiveList');
  return h('ul', null,
    items.map(item => h('li', { key: item.id }, item.text))
  );
});
```

### 3. useMemo — мемоизация вычислений

Важно: `useMemo` не должен вызывать `setState` внутри себя — это приведёт к
бесконечным рендерам. Гораздо надёжнее хранить кэш в `useRef`.

```javascript
export function useMemo(factory, deps) {
  const memoRef = useRef({ value: null, deps: null });

  if (!memoRef.current.deps || !arraysEqual(deps, memoRef.current.deps)) {
    memoRef.current = {
      value: factory(),
      deps
    };
  }

  return memoRef.current.value;
}

// Использование
function FilteredList({ items, filter }) {
  const filtered = useMemo(() => {
    console.log('Фильтрация...');
    return items.filter(item => item.includes(filter));
  }, [items, filter]);

  return h('ul', null,
    filtered.map(item => h('li', { key: item }, item))
  );
}
```

## Измерение производительности

### React DevTools Profiler

```javascript
function Profiler({ id, onRender, children }) {
  const startTime = useRef(null);

  useEffect(() => {
    startTime.current = performance.now();
  });

  useEffect(() => {
    const endTime = performance.now();
    const duration = endTime - startTime.current;

    onRender(id, 'update', duration);
  });

  return children;
}

// Использование
h(Profiler, {
  id: 'TodoList',
  onRender: (id, phase, duration) => {
    console.log(`${id} ${phase}: ${duration}ms`);
  }
}, h(TodoList, { todos }))
```

### Performance API

```javascript
function measureComponent(name, fn) {
  performance.mark(`${name}-start`);
  const result = fn();
  performance.mark(`${name}-end`);
  performance.measure(name, `${name}-start`, `${name}-end`);

  const measure = performance.getEntriesByName(name)[0];
  console.log(`${name}: ${measure.duration}ms`);

  return result;
}

// Использование
const vNode = measureComponent('render', () => {
  return App({ todos });
});
```

## Стратегии оптимизации

### 1. Разделяйте часто и редко меняющиеся данные

```javascript
// ❌ Плохо — всё в одном
function App() {
  const [user, setUser] = useState({});
  const [theme, setTheme] = useState('light');
  const [todos, setTodos] = useState([]);
  // При изменении любого — перерисовка всего
}

// ✓ Хорошо — разделено
function App() {
  return h('div', null,
    h(ThemeProvider, null,    // Меняется редко
      h(UserProvider, null,   // Меняется редко
        h(TodoList)           // Меняется часто
      )
    )
  );
}
```

### 2. Поднимайте состояние вверх осторожно

```javascript
// ❌ Плохо — состояние слишком высоко
function App() {
  const [inputValue, setInputValue] = useState('');
  // Каждое нажатие клавиши → перерисовка всего App!
  return h('div', null,
    h(Header),
    h(Input, { value: inputValue, onChange: setInputValue }),
    h(Footer)
  );
}

// ✓ Хорошо — состояние локальное
function App() {
  return h('div', null,
    h(Header),
    h(InputWrapper),  // Состояние внутри
    h(Footer)
  );
}
```

### 3. Виртуализация длинных списков

```javascript
function VirtualList({ items, itemHeight, containerHeight }) {
  const [scrollTop, setScrollTop] = useState(0);

  const visibleStart = Math.floor(scrollTop / itemHeight);
  const visibleEnd = Math.ceil((scrollTop + containerHeight) / itemHeight);

  const visibleItems = items.slice(visibleStart, visibleEnd);

  return h('div', {
    style: { height: `${containerHeight}px`, overflow: 'auto' },
    onscroll: (e) => setScrollTop(e.target.scrollTop)
  },
    h('div', { style: { height: `${items.length * itemHeight}px` } },
      visibleItems.map((item, i) =>
        h('div', {
          key: item.id,
          style: {
            height: `${itemHeight}px`,
            position: 'absolute',
            top: `${(visibleStart + i) * itemHeight}px`
          }
        }, item.text)
      )
    )
  );
}
```

## Задание

1. Реализуйте простой Profiler:
```javascript
function useProfiler(componentName) {
  // Измеряйте время рендера
}
```

2. Создайте HOC для отслеживания перерисовок:
```javascript
function withRenderCount(Component) {
  // Показывайте количество рендеров
}
```

3. Оптимизируйте список с 10000 элементов:
```javascript
// Используйте виртуализацию, memo, keys
```

## Ключевые выводы

- Reconciliation — процесс обновления DOM через diff + patch
- Fiber разбивает работу на прерываемые единицы
- Render фаза прерываема, commit фаза — нет
- Приоритеты позволяют важным обновлениям идти первыми
- Concurrent Mode работает над несколькими обновлениями
- memo мемоизирует результат компонента
- useMemo мемоизирует вычисления
- Разделяйте часто и редко меняющиеся данные
- Используйте виртуализацию для длинных списков
- Измеряйте производительность для оптимизации

---

[← Предыдущий урок: Refs](./15-refs.md) | [Следующий урок: Построение Todo приложения →](./17-todo-app.md)

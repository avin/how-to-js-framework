# Урок 2: Virtual DOM - концепция

## Проблема: почему работа с DOM медленная?

DOM (Document Object Model) — это API для работы с HTML-документом. Каждая операция с DOM (создание элемента, изменение атрибута, добавление в дерево) — это относительно дорогая операция, потому что:

1. **Браузер делает много работы**: пересчитывает стили, layout, перерисовывает страницу
2. **DOM API медленнее обычных JavaScript объектов**
3. **Множественные изменения вызывают множественные reflow/repaint**

Пример проблемного кода:

```javascript
// Каждая операция вызывает reflow/repaint
const list = document.getElementById('list');

for (let i = 0; i < 1000; i++) {
  const item = document.createElement('li');
  item.textContent = `Item ${i}`;
  list.appendChild(item); // Reflow после каждого добавления!
}
```

Браузер вынужден 1000 раз пересчитывать layout. Можно оптимизировать:

```javascript
// Лучше: собираем всё в DocumentFragment
const fragment = document.createDocumentFragment();

for (let i = 0; i < 1000; i++) {
  const item = document.createElement('li');
  item.textContent = `Item ${i}`;
  fragment.appendChild(item); // Нет reflow
}

list.appendChild(fragment); // Один reflow
```

Но это всё равно требует от разработчика думать об оптимизации. Что если автоматизировать этот процесс?

## Концепция Virtual DOM

**Virtual DOM (VDOM)** — это JavaScript-представление реального DOM-дерева. Это обычные объекты, которые описывают структуру интерфейса.

### Сравнение

```javascript
// Реальный DOM (медленно)
const button = document.createElement('button');
button.className = 'btn';
button.textContent = 'Нажми меня';

// Virtual DOM (быстро - это просто объект!)
const vButton = {
  type: 'button',
  props: {
    class: 'btn'
  },
  children: ['Нажми меня']
};
```

## Как работает Virtual DOM?

### Шаг 1: Создаём виртуальное дерево

Вместо прямого создания DOM, мы создаём JavaScript-объекты:

```javascript
// Представим такой HTML:
// <div class="container">
//   <h1>Заголовок</h1>
//   <p>Параграф</p>
// </div>

const vNode = {
  type: 'div',
  props: { class: 'container' },
  children: [
    {
      type: 'h1',
      props: null,
      children: ['Заголовок']
    },
    {
      type: 'p',
      props: null,
      children: ['Параграф']
    }
  ]
};
```

### Шаг 2: Превращаем виртуальное дерево в реальный DOM

```javascript
function createDOMElement(vNode) {
  // Если это текст
  if (typeof vNode === 'string') {
    return document.createTextNode(vNode);
  }

  // Создаём элемент
  const element = document.createElement(vNode.type);

  // Устанавливаем атрибуты
  if (vNode.props) {
    Object.keys(vNode.props).forEach(key => {
      element.setAttribute(key, vNode.props[key]);
    });
  }

  // Рекурсивно создаём детей
  vNode.children.forEach(child => {
    element.appendChild(createDOMElement(child));
  });

  return element;
}

// Использование
const realDOM = createDOMElement(vNode);
document.body.appendChild(realDOM);
```

### Шаг 3: При изменениях сравниваем старое и новое дерево

Это ключевая часть! Вместо пересоздания всего DOM:

```javascript
const oldVNode = {
  type: 'div',
  props: { class: 'container' },
  children: [
    { type: 'h1', props: null, children: ['Старый заголовок'] }
  ]
};

const newVNode = {
  type: 'div',
  props: { class: 'container' },
  children: [
    { type: 'h1', props: null, children: ['Новый заголовок'] } // Изменился!
  ]
};

// Сравниваем и находим разницу:
// - div не изменился
// - h1 не изменился
// - Текст внутри h1 изменился - нужно обновить только его!
```

### Шаг 4: Применяем только необходимые изменения

```javascript
// Вместо полного пересоздания:
// container.innerHTML = ''; // Удалить всё
// container.appendChild(newElement); // Создать заново

// Делаем точечное обновление:
const h1 = container.querySelector('h1');
h1.firstChild.textContent = 'Новый заголовок'; // Минимальное изменение!
```

## Преимущества Virtual DOM

### 1. Производительность

Операции с JavaScript-объектами в тысячи раз быстрее, чем с DOM:

```javascript
// Быстро: миллион операций за миллисекунды
const vNodes = [];
for (let i = 0; i < 1000000; i++) {
  vNodes.push({ type: 'div', props: null, children: [] });
}

// Медленно: тысяча операций может занять секунды
const domNodes = [];
for (let i = 0; i < 1000; i++) {
  domNodes.push(document.createElement('div'));
}
```

### 2. Батчинг (группировка изменений)

Фреймворк может накопить изменения и применить их одним разом:

```javascript
// Без VDOM: 3 reflow
element.style.width = '100px';  // reflow
element.style.height = '100px'; // reflow
element.style.color = 'red';    // reflow

// С VDOM: изменения накапливаются в объекте,
// потом применяются разом = 1 reflow
updateVNode(element, {
  width: '100px',
  height: '100px',
  color: 'red'
});
```

### 3. Декларативность

Вы описываете ФИНАЛЬНОЕ состояние, а не шаги по его достижению:

```javascript
// Императивно (сложно)
if (isLoggedIn && !wasLoggedIn) {
  showWelcomeMessage();
  hideLoginButton();
  showLogoutButton();
} else if (!isLoggedIn && wasLoggedIn) {
  hideWelcomeMessage();
  hideLogoutButton();
  showLoginButton();
}

// Декларативно (просто)
function render(isLoggedIn) {
  return isLoggedIn
    ? { type: 'div', children: ['Добро пожаловать', LogoutButton] }
    : { type: 'div', children: [LoginButton] };
}
```

### 4. Кроссплатформенность

Virtual DOM — это абстракция. Можно рендерить не только в браузерный DOM:

```javascript
// Браузер
render(vNode, domElement);

// Canvas
renderToCanvas(vNode, canvasContext);

// React Native - нативные компоненты
renderToNative(vNode, nativeView);

// Строка (серверный рендеринг)
const html = renderToString(vNode);
```

## Недостатки Virtual DOM

Важно понимать, что VDOM — это **компромисс**, а не серебряная пуля:

### 1. Оверхед памяти

Нужно хранить виртуальное дерево в памяти:

```javascript
// Двойные затраты памяти
const virtualTree = { /* вся структура */ };  // Память
const realDOM = document.getElementById('app'); // Память
```

### 2. Оверхед вычислений при diffing

Сравнение деревьев — это O(n) операция:

```javascript
// При каждом обновлении нужно пройти по дереву
function diff(oldTree, newTree) {
  // Много сравнений и вычислений
}
```

### 3. Не всегда быстрее прямых операций

Для простых случаев прямая работа с DOM может быть быстрее:

```javascript
// Для одного изменения VDOM избыточен
element.textContent = 'Новый текст'; // Быстро

// VDOM добавляет накладные расходы
const newVNode = { type: 'div', children: ['Новый текст'] };
diff(oldVNode, newVNode);  // Медленнее для единичного изменения
patch(element, changes);
```

**Но**: VDOM даёт предсказуемость и простоту разработки, что важнее raw-скорости для большинства приложений.

## Альтернативы Virtual DOM

Некоторые фреймворки не используют VDOM:

### Svelte
Компилирует компоненты в императивный код во время сборки:

```javascript
// Svelte знает ЧТО изменилось на этапе компиляции
let count = 0;
$: doubled = count * 2; // Компилятор создаст код обновления только для doubled
```

### Solid.js
Использует fine-grained реактивность:

```javascript
// Создаёт DOM один раз, потом обновляет только реактивные части
const [count, setCount] = createSignal(0);
return <div>Count: {count()}</div>; // Обновится только текстовый узел
```

## Визуализация процесса

Вот как выглядит полный цикл работы с VDOM:

```
1. Состояние изменилось
   state = { count: 1 } → state = { count: 2 }

2. Создаём новое виртуальное дерево
   render(state) → newVTree

3. Сравниваем с предыдущим
   diff(oldVTree, newVTree) → patches

4. Применяем изменения к реальному DOM
   patch(domElement, patches)

5. Сохраняем новое дерево для следующего сравнения
   oldVTree = newVTree
```

## Пример: полный цикл

```javascript
// Начальное состояние
let state = { count: 0 };

// 1. Функция создания виртуального дерева
function render(state) {
  return {
    type: 'div',
    props: null,
    children: [
      {
        type: 'p',
        props: null,
        children: [`Счёт: ${state.count}`]
      },
      {
        type: 'button',
        props: { onclick: increment },
        children: ['Увеличить']
      }
    ]
  };
}

// 2. Первый рендер
let vTree = render(state);
let domElement = createDOMElement(vTree);
document.body.appendChild(domElement);

// 3. При изменении состояния
function increment() {
  state.count++;

  // Создаём новое дерево
  const newVTree = render(state);

  // Сравниваем
  const patches = diff(vTree, newVTree);

  // Применяем изменения
  patch(domElement, patches);

  // Сохраняем
  vTree = newVTree;
}
```

## Что дальше?

Теперь мы понимаем **концепцию** Virtual DOM. В следующих уроках мы:
1. Создадим структуру данных для виртуальных узлов
2. Реализуем функцию рендеринга
3. Напишем алгоритм diffing
4. Научимся применять изменения эффективно

## Задание

1. Создайте виртуальное представление следующего HTML в виде объектов:
```html
<ul class="list">
  <li>Первый</li>
  <li>Второй</li>
  <li>Третий</li>
</ul>
```

2. Напишите простую функцию, которая превращает виртуальный узел в реальный DOM-элемент (без поддержки событий пока)

3. Подумайте: какие изменения потребуются в DOM при переходе от:
```javascript
{ type: 'div', children: ['Hello'] }
// к
{ type: 'div', children: ['Hello', ' ', 'World'] }
```

## Ключевые выводы

- Virtual DOM — это JavaScript-представление реального DOM
- Он позволяет оптимизировать обновления через diffing
- Это компромисс между производительностью и удобством разработки
- Не всегда самый быстрый подход, но даёт предсказуемость
- Основа для декларативного программирования UI

---

[← Предыдущий урок: Введение](./01-introduction.md) | [Следующий урок: Представление элементов →](./03-element-representation.md)

# Что нового в React 18? 

## Install

```javascript
npm install react@alpha react-dom@alpha
```

## Root API

React 18 включает два корневых API, которые называются **Legacy Root API** и **New Root API**.

* Legacy Root API (Устаревший корневой API): это существующий API, вызываемый с помощью ReactDOM.render. Этот метод создает корень приложения, работающий в «устаревшем» режиме, который работает точно так же, как в React 17. Перед релизом будет добавлено предупреждение, указывающее, что метод устарел, и, что необходимо переключиться на New Root API.


* New Root API (Новый корневой API): новый корневой API вызывается с помощью ReactDOM.createRoot. Это создает корень приложения, работающий в React 18, который добавляет все улучшения React 18 и позволяет использовать параллельные функции. Это будет основной корневой API в дальнейшем.

**Что такое корень приложения?**

В React «корень» - это указатель на структуру данных верхнего уровня, которую React использует для отслеживания дерева при рендеринге. В устаревшем API корень был непрозрачен для пользователя, потому что он был прикреплён к элементу DOM и доступ к нему был через узел DOM. В то время, как в New Root API сначала создаётся корень приложения, а затем вызывается его рендеринг:

```javascript
import ReactDOM from "react-dom";
import App from "App";

const container = document.getElementById("app");

// Old
ReactDOM.render(<App />, container);

// New
const root = ReactDOM.createRoot(container);

root.render(<App />);
```

API был изменён по нескольким причинам: это исправляет некоторую эргономику API при запуске обновлений. Как показано выше, в устаревшем API нам нужно продолжать передавать контейнер в рендер, даже если он никогда не изменится. Это также означает, что нам не нужно хранить корень на узле DOM, хотя мы все еще делаем это сегодня.

___

## Пакетная обработка (Автоматическая группировка изменений)

**Пакетная обработка** — это когда React **группирует несколько обновлений состояния в один рендеринг** для повышения производительности.

Например, если у нас есть два обновления состояния внутри одного события клика по кнопке, то React всегда объединяет их в один рендеринг. Если мы запустим следующий [код](https://929i6.csb.app/ "ссылка на проект") , мы увидим, что каждый раз, когда мы нажимаем, то React выполнит только один рендеринг, хотя мы устанавливаем состояние дважды:

```javascript
function App() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  function handleClick() {
    setCount(c => c + 1); // Еще не перерисовывает
    setFlag(f => !f); // Еще не перерисовывает
    // React выполнит рендеринг только один раз в конце (это и есть пакетная обработка!)
  }

  return (
    <div>
      <button onClick={handleClick}>Next</button>
      <h1 style={{ color: flag ? "blue" : "black" }}>{count}</h1>
      <LogEvents />
    </div>
  );
}

function LogEvents(props) {
  useLayoutEffect(() => {
    console.log("Commit");
  });
  console.log("Render");
  return null;
}
```
Это отлично подходит для повышения производительности, поскольку позволяет избежать ненужных повторных рендеров. Это также предотвращает отрисовку нашего компонента с не полностью обновленным состоянием, что могло бы вызвать ошибки.

Это напоминает работу официанта: Он дожидается, когда вы сделаете полностью весь заказ, а не бежит каждый раз на кухню, когда вы озвучиваете блюдо.

Однако, если нам нужно получить данные, а затем обновить состояние в handleClick, то React **НЕ будет** пакетировать обновления и выполнит два независимых обновления.

Это связано с тем, что React раньше группировал обновления только во время события браузера (например, клика), но [здесь](https://cn5ct.csb.app/ "ссылка на проект") мы обновляем состояние после того, как событие уже было обработано (в обратном вызове fetch):

```javascript
function App() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  function handleClick() {
    fetchSomething().then(() => {
      // React 17 и более ранние версии НЕ объединяют их в пакеты, потому что
      // они запускаются после события в callback, а не во время его
      setCount(c => c + 1); // Вызывает повторный рендер
      setFlag(f => !f); // Вызывает повторный рендер
    });
  }

  return (
    <div>
      <button onClick={handleClick}>Next</button>
      <h1 style={{ color: flag ? "blue" : "black" }}>{count}</h1>
      <LogEvents />
    </div>
  );
}

function LogEvents(props) {
  useLayoutEffect(() => {
    console.log("Commit");
  });
  console.log("Render");
  return null;
}

function fetchSomething() {
  return new Promise((resolve) => setTimeout(resolve, 100));
}
```
До React 18 обновления группировались только внутри обработчиков событий React. Обновления внутри промисов, setTimeout, нативных обработчиков событий или любого другого события по умолчанию не группировались в React.

**Что такое автоматическая группировка изменений?**

Но, начиная с React 18 все обновления будут автоматически группироваться независимо от того, откуда они исходят.

Это означает, что обновления внутри setTimeouts, промисов, нативных обработчиков событий или любого другого события будут пакетироваться так же, как обновления внутри событий React. Ожидается, что это приведет к меньшему количеству обновлений компонента и, следовательно, к повышению производительности в [наших](https://lgz88.csb.app/ "ссылка на проект") приложениях:

```javascript
function App() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  function handleClick() {
    fetchSomething().then(() => {
      // React 18 и более поздние версии ДЕЛАЮТ пакетную обработку:
      setCount(c => c + 1);
      setFlag(f => !f);
      // React выполнит рендеринг только один раз в конце (это и есть пакетная обработка!)
    });
  }

  return (
    <div>
      <button onClick={handleClick}>Next</button>
      <h1 style={{ color: flag ? "blue" : "black" }}>{count}</h1>
    </div>
  );
} 	
```

React будет автоматически пакетировать обновления независимо от того, где они происходят, поэтому это:

```javascript
function handleClick() {
  setCount(c => c + 1);
  setFlag(f => !f);
  // React выполнит рендеринг только один раз в конце (это и есть пакетная обработка!)
```

ведёт себя так же как это:

```javascript
setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);
  // React выполнит рендеринг только один раз в конце (это и есть пакетная обработка!)
}, 1000);
```

и так же как это:

```javascript
elm.addEventListener('click', () => {
  setCount(c => c + 1);
  setFlag(f => !f);
  // React выполнит рендеринг только один раз в конце (это и есть пакетная обработка!)
});
```

> React запускает пакетные обновления только тогда, когда это вообще безопасно. Например, React гарантирует, что для каждого инициированного пользователем события, такого как клик или нажатие клавиши, DOM полностью обновится перед следующим событием. Это гарантирует, например, что форма, которая заблокирована при отправке, не может быть отправлена дважды.

**Что делать, если мы не хотим выполнять пакетную обработку?**

Обычно пакетирование безопасно, но некоторый код может зависеть от чтения чего-либо из DOM сразу после изменения состояния. В этих случаях вы можете использовать **ReactDOM.flushSync()**, чтобы отказаться от пакетной обработки:

```javascript
import { flushSync } from 'react-dom'; // запомни: react-dom, не react

function handleClick() {
  flushSync(() => {
    setCounter(c => c + 1);
  });
  // React уже обновил DOM
  flushSync(() => {
    setFlag(f => !f);
  });
  // React уже обновил DOM
}
```

Ожидается, что это будет использоваться в исключительных случаях.
___ 

## startTransition 

Иногда из-за небольших действий, таких как нажатие кнопки или ввод текста, на экране может происходить многое. Это может привести к зависанию страницы или зависанию во время выполнения всей работы.

Например, рассмотрим фильтрацию списка данных на основе текста, введённого в поле ввода. Нам необходимо сохранить значение поля в состоянии, чтобы мы могли фильтровать данные и контролировать значение этого поля ввода. Наш код может выглядеть примерно так:

```javascript
import React from 'react'

const [inputValue, setInputValue] = React.useState('')

const handleChangeInput = (e) => {
  setInputValue(e.target.value)
}
```

Здесь, когда пользователь вводит символ, мы обновляем входное значение и используем новое значение для поиска в списке и отображения результатов. Для обновлений на большом экране это может вызвать задержку на странице при отображении всего содержимого, из-за чего набор текста или другие действия будут казаться медленными и невосприимчивыми. Даже если список не слишком длинный, сами элементы списка могут быть сложными и разными при каждом нажатии клавиши, и может не быть четкого способа оптимизировать их отображение.

Концептуально проблема в том, что необходимо выполнить два разных обновления. Первое обновление — это срочное обновление, чтобы изменить значение поля ввода и, возможно, некоторый пользовательский интерфейс вокруг него. Второе — менее срочное обновление, показывающее результаты поиска.

```javascript
import React from 'react'

const [inputValue, setInputValue] = React.useState('')
const [searchQuery, setSearchQuery] = React.useState('')

const handleChangeInput = (e) => {
  // срочное: показывает что было написано
  setInputValue(e.target.value)

  // не срочное: показывает результат
  setSearchQuery(e.target.value)
}
```

Пользователи ожидают, что первое обновление будет немедленным, потому что встроенная в браузере обработка этих взаимодействий выполняется быстро. Но второе обновление может немного задержаться. Пользователи не ожидают, что оно завершится немедленно, и это хорошо, потому что может потребоваться много работы. (На самом деле разработчики часто искусственно задерживают такие обновления с помощью таких приемов, как **debouncing**.)


<p align="center">
  <img src="https://miro.medium.com/proxy/1*cMutiTstVGIcEl-dtmxlJg.gif" />
  <p align="center">Google карты с реализацией debouncing</p>
</p>

До React 18 все обновления рендерились сразу. Это означает, что два указанных выше состояния по-прежнему будут отображаться одновременно и по-прежнему будет блокировать взаимодействие пользователя с интерфейсом до тех пор, пока все не будет отрисовано. Чего нам не хватает, так это способа сообщить React, какие обновления являются срочными, а какие нет.

Новый API **startTransition** решает эту проблему, давая нам возможность отмечать обновления как «переходы»:

```javascript
import React from 'react'

const [inputValue, setInputValue] = React.useState('')
const [searchQuery, setSearchQuery] = React.useState('')
const [isPending, startTransition] = React.useTransition()

const handleChangeInput = () => {
  // срочное: показывает что было написано
  setInputValue(e.target.value)

  startTransition(() => {
    // Transition: показывает результат
    setSearchQuery(e.target.value)
  })
}
```

Обновления, заключенные в **startTransition**, обрабатываются как несрочные и будут прерваны, если поступят более приоритетные обновления, такие как щелчки или нажатия клавиш. Если переход прерывается пользователем (например, путем ввода нескольких символов в строке), React дает выполнить устаревшую работу по рендерингу, которая не была завершена, и рендерить только последнее обновление.

Переходы позволяют поддерживать оперативность большинства взаимодействий, даже если они приводят к значительным изменениям пользовательского интерфейса. Они также позволяют нам не тратить время на рендеринг контента, который больше не актуален.

До 18-ой версии React не предоставлял инструмента для приоритизации обновлений пользовательского интерфейса. Для решения этой задачи, как правило, использовались такие инструменты, как setTimeout, throttling и debouncing. Минус этих решений в том, что код становится ассинхронным, усложняя управление состоянием приложения и остаётся блокировка браузера, после завершения таймаута.

Начиная с React 18 появился хук useTransition(), который помогает нам отмечать обновления пользовательского интерфейса как низкоприоритетные, что особенно полезно для тяжелых несрочных обновлений.
___

## useDeferredValue

```javascript
const deferredValue = useDeferredValue(value, { timeoutMs: 2000 });
```

Хук возвращает отложенную версию для значения, получение которого может «отставать» от рендера всего компонента более чем на timeoutMs.

Обычно этот хук используется для создания отзывчивого интерфейса, когда часть компонента нужно немедленно отрендерить на основе пользовательского ввода и при этом другая часть ожидает загрузки данных.

Хорошим примером является ввод текста.

```javascript
function App() {
  const [text, setText] = useState("hello");
  const deferredText = useDeferredValue(text, { timeoutMs: 2000 });

  return (
    <div className="App">
      {/* Вводим текст в поле ... */}
      <input value={text} onChange={handleChange} />
      ...
      {/* ... при этом список может отображать отложенное значение */}
      <MySlowList text={deferredText} />
    </div>
  );
 }
```
Такой подход позволяет немедленно показывать новый текст для input, чтобы улучшить отзывчивость UI. В тоже время, если рендер MySlowList «отстает» на 2 секунды, установленные в timeoutMs, то в фоновом режиме отображается отложенное значение.

___

## Suspense и Потоковый рендер на сервере (SSR - server-side rendering)

Server-side rendering  - простыми словами, он генерирует HTML из компонент React на сервере и возвращает этот HTML пользователю.

Если мы не используем рендеринг на стороне сервера (известный как SSR), то наш сайт будет пустым при первом запуске.

<p align="center">
  <img src="https://github.com/IFabrichnov/React-18/blob/main/README-IMG/1.jpg" />
  <p align="center">Первый запуск без SSR</p>
</p>

Это связано с тем, что браузеру необходимо получить и прочитать наш код JS, для чего требуется время, прежде чем наши компоненты загрузятся и страница станет интерактивной.

Однако с SSR пользователь видит, как наше приложение выглядит напрямую, но без каких-либо взаимодействий во время загрузки JS.

SSR работает следующим образом: сначала отрисовывает все компоненты на сервере, а затем отправляет результат в виде HTML в браузер. После этого загружается JS, и HTML становится интерактивным благодаря так называемой гидратации. Это превращает наши статические HTML-элементы в динамические компоненты React.

<p align="center">
  <img src="https://github.com/IFabrichnov/React-18/blob/main/README-IMG/2.gif" />
  <p align="center">Как работает SSR</p>
</p>


Основная проблема с этим подходом заключается в том, что до тех пор, пока не будет извлечен, загружен JS и не будет обработан HTML - наша страница не будет интерактивной.

Чтобы решить эту проблему, **React 18** теперь предлагает потоковую передачу HTML (Streaming HTML) для SSR:

Проще говоря, потоковая передача HTML означает, что сервер может отправлять части наших компонентов по мере их рендеринга.

<p align="center">
  <img src="https://github.com/IFabrichnov/React-18/blob/main/README-IMG/3.jpg" />
</p>

```javascript
<Suspense fallback={<h1>Загрузка...</h1>}>
  <ProfilePhoto />
  <ProfileDetails />
</Suspense>
```

Это работает с использованием **Suspense** (Suspense позволяет нашим компонентам «ждать», отображая запасной интерфейс, прежде чем они будут отрендерены), когда мы указываем, какие части нашего приложения будут загружаться дольше, а какие должны быть отрисованы напрямую.


___

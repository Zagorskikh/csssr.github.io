---
layout:  post
title: Осторожно! Возможны побочные эффекты
summary: Разбираемся с тем, что такое сайд-эффекты и что они несут нашему коду
author: bracketsarrows
authorLink: https://twitter.com/bracketsarrows
---

> Начинать нужно с того, что сеет сомнение.
>
> -- <cite>Братья Стругацкие</cite>

## Типичный фронтенд

Давайте посмотрим на простой пример фронтенд-кода:

```javascript
var URL = 'https://api.giphy.com/v1/gifs/random?api_key=dc6zaTOxFJmzC&tag=cats';

function app() {
  document.addEventListener(
    'click',
    () => 
      if (event.target.tagName === 'BUTTON') {
        fetch(URL)
          .then(response => response.json().data.image_url)
          .then(gifSrc => document.querySelector('img').setAttribute('src', gifSrc))
      }
  );
}
```

Что является результатом работы этой функции?

 - функция делает HTTP запрос при помощи `fetch`;
 - функция устанавливает атрибут `src` тегу `img` при помощи `.setAttribute`.

От чего зависит эта функция?

 - функция зависит от событий, полученных из `document.addEventListener`;
 - функция зависит от результата HTTP запроса, полученного из `fetch`.

Теперь давайте скроем реализацию функции и ещё раз посмотрим на неё:

```javascript
function app() {
  ...
  return;
}
```

Хм, теперь кажется, что функция ни от чего не зависит и вообще ничего не делает. В целом всё логично — не видя реализации функции, мы не можем определить её действие. Давайте посмотрим на другую функцию:

```javascript
function sum(list) {
  var result = list.reduce((acc, item) => acc + item, 0);
  return result;
}
```

И скроем её реализацию:

```javascript
function sum(list) {
  ...
  return result;
}
```

В этом случае мы видим и от чего зависит функция, и что возвращает: она явно что-то делает и возвращает результат, вычисленный от переданного ей `list`.

Почему так? В чем разница между `app` и `sum`? Что это за два разных вида функций? 

## Два вида функций

`sum` берёт зависимые значения _только_ из списка аргументов и возвращает результат _только_ при помощи оператора `return` — то есть взаимодействует с окружающим кодом _только_ через стандартные механизмы вызова функции.

Такие функции называются чистыми (pure). 

`app` же, наоборот, в качестве аргументов использует данные, которые попадают в неё неявно (не через список аргументов), и возвращает результат своей работы неявно (не через `return`).

Про такие функции говорят, что они обладают _сайд-эффектами_, и в противоположность чистым функциям их называют грязными.

То есть:

> Cайд-эффектами называют неявные зависимости функции или неявные результаты её работы.

Звучит довольно просто, однако на практике в языках без явных ограничений на чистоту функции [данное определение предстаёт не таким однозначным](http://staltz.com/is-your-javascript-function-actually-pure.html).

Чтобы избежать софистики, давайте вместо попытки дать определение сайд-эффектам попробуем выделить их ключевые свойства. То, что будет удовлетворять этим свойствам, мы и будем считать сайд-эффектом.

## Сайд-эффект — что ты такое?

Первое и самое главное:

#### Сайд-эффект — это не первоклассный обьект

Что это значит? [Первоклассный объект (first-class object)](https://ru.wikipedia.org/wiki/%D0%9E%D0%B1%D1%8A%D0%B5%D0%BA%D1%82_%D0%BF%D0%B5%D1%80%D0%B2%D0%BE%D0%B3%D0%BE_%D0%BA%D0%BB%D0%B0%D1%81%D1%81%D0%B0) — это сущность в языке программирования, которую:

 - можно сохранить в переменной или структурах данных;
 - можно передать в функцию как аргумент;
 - можно вернуть из функции как результат.

Проще говоря, первоклассный объект можно легко представить в виде некоторого _значения_.
Очевидно, что для сайд-эффектов это не так. Мы не можем просто взять и переписать функцию `app` в виде:

```javascript
function app(httpResponse, domEvent) {
  ... our logic
  return { httpRequest, domMutation };
}
```
![No first class](/images/side-effects/no-first-class.jpg)

#### Сайд-эффекты зависят от/влияют на внешнее окружение

Это довольно очевидное свойство. Для того, чтобы корректно выполнить некоторый сайд-эффект, нам необходимо корректное окружение: для HTTP запроса нужна рабочая сеть, для запроса к DOM необходим сформированный DOM и так далее.

#### Сайд-эффекты лишают функции ссылочной прозрачности

Ссылочная прозрачность (referential transparency) — свойство функции, благодаря которому можно всегда и везде вместо результата работы функции подставить её вызов.

То есть вместо `var result = sum(list); return [result, result];` можно спокойно написать `return [sum(list), sum(list)];`.

Функции с сайд-эффектами не обладают этим свойством — вызвав `app` два раза, мы получим совершенно другие результаты (будет два listener вместо одного, два HTTP запроса при клике вместо одного и так далее).

#### Сайд-эффекты изменяют свойства кода, в котором используются, до самой вершины стека вызовов

Функция с сайд-эффектом внутри сама становится в некотором смысле сайд-эффектом. Что такое `fetch`? Это функция, которая внутри себя содержит сайд-эффекты, или это целиком сайд-эффект? А если завернуть её в дополнительную обёртку? Для нас важно то, что использование сайд-эффектов внутри других функций приводит к приобретению этими функциями всех свойств сайд-эффектов.

На самом деле, скорее всего, последние два свойства вытекают из первого — но это только догадка, и поэтому я решил вынести их в отдельные пункты.

## Что всё это значит для нас?

Ну, окей, сайд-эффекты имеют какие-то свои специфичные свойства. Но что эти свойства означают для нас на практике?

#### Код с сайд-эффектами сложен для анализа (как человеком, так и машиной)

Для начала давайте посмотрим на код без сайд-эффектов — все функции в нём чистые.

```javascript
function calcAnything(value) {
  var a = calcA(value);
  var b = calcB(value);
  return сalcC(value, b);
}
```
Мы можем легко увидеть, какие значения от каких зависят, а какие не играют роли. К примеру, мы видим, что от `a` ничего не зависит и его вычисление можно смело удалить.

```javascript
function calcAnything(value) {
  return сalcC(value, calcB(value));
}
```

Более того, мы можем построить граф вычислений:

![Scheme of control flow](/images/side-effects/analyze.svg)

А теперь попробуем проделать то же самое с кодом, в котором, _возможно_, содержатся сайд-эффекты:

```javascript
function doSomething() {
  var a = doSomethingA(value);
  var b = doSomethingB(value);
  return doSomethingC(value, b);
}
```

Мы всё ещё можем сказать, что, к примеру, `b` зависит от `value`, но мы не можем со всей уверенностью утверждать, что `b` не зависит от вычисления `a`. Представьте, что `doSomethingA` записывает что-то в файл, из которого затем читает `doSomethingB`. Соответственно, в коде с сайд-эффектами любое вычисление потенциально может зависеть от любого, так как все они влияют на один и тот же внешний мир.

Это оказывает влияние на:

 - машинный анализ (IDE не сможет подсказать нам, правильно ли мы используем ту или иную функцию);
 - анализ человеком (зачастую код-ревью просто не работает, потому что правки кода в одном месте влияют на другую часть системы).

Также в случае кода с сайд-эффектами значительно усложняется рефакторинг. К примеру, удаление неиспользующегося кода:

![Delete unused code](/images/side-effects/complicate-refactoring.gif)

#### Код с сайд-эффектами сложно переиспользовать

Это менее очевидное следствие из свойств сайд-эффектов. Давайте вновь взглянем на код, состоящий из чистых функций.

Мы написали функцию, которая считает длину и сумму массива:

```javascript
function calcLengthAndSum() { ... }
```
Используя её, мы можем легко написать функцию, вычисляющую только длину списка:

```javascript
function calcLength(list) {
  return calcLengthAndSum(list).length;
}
```

Или его среднее:

```javascript
function calcMean(list) {
  var res = calcLengthAndSum(list);
  return res.sum / res.length;
}
```

Чистые функции крайне легко переиспользуются: даже если они делают что-то лишнее или что-то немного не так, мы всегда можем исправить это, просто добавив обёртку из ещё одной чистой функции.

Как говорится: 

> Любую проблему можно решить путём введения дополнительного уровня абстракции, кроме проблемы слишком большого количества уровней абстракции.
>

Давайте посмотрим на функцию с сайд-эффектами, которая делает HTTP запрос и записывает результат в некий файл:

```javascript
function sendRequestAndWriteFile() { ... }
```

Теперь где-то в коде нам понадобилось отправлять тот же запрос, но не записывать его в файл.

Cкорее всего, мы добавим специальную опцию в `sendRequestAndWriteFile`:

```javascript
function sendRequest() {
  return sendRequestAndWriteFile({
    onlyRequest: true
  });
}
```

То же самое для ситуации, когда нам захотелось отправлять запросы на другой URL:

```javascript
function sendRequestAndWriteFileOnUrl(url) {
  return sendRequestAndWriteFile({ url: url })
}
```

Так как сайд-эффект — это _неявный_ результат, мы не можем повлиять на него за пределами места его создания — отбросить его часть или как-то преобразовать, как в случае с результатом `calcLengthAndSum`.

![Compose pure vs dirty](/images/side-effects/compose-pure-vs-dirty.svg)

Это заставляет нас вместо простого переиспользования добавлять множество опций в функцию, что очень сильно увеличивает [цикломатическую сложность](https://ru.wikipedia.org/wiki/Цикломатическая_сложность) кода a.k.a [complexity](http://eslint.org/docs/rules/complexity).

Количество вариантов выполнения растёт со скоростью 2^n: для функции всего с двумя булевыми опциями мы получаем уже 4 варианта исполнения, для трёх опций — уже 8 и так далее.

#### Cайд-эффекты сложно тестировать

С этим наверняка знакомы все. Сравните:

```javascript
it('2 + 2 = 4', () => {
  var result = add(2, 2);
  expect(result).toBe(4);
})
```

с:

```javascript
it('remoteAdd send args to endpoint', () => {
  var fakeRes = {};
  mock(HttpClient, {post: (...args) => {
    return fakeRes;
  }})
  var result = remoteAdd(2, 2);
  expect(result).toBe(fakeRes);
});
```

Выглядит немного сложнее, да? Но постойте, мы не проверили тот факт, что наша функция делает ровно один `HTTP` запрос. Исправим:

```diff
it('remoteAdd send args to endpoint', () => {
+ var postCalls = [];
  var fakeRes = {};
  mock(HttpClient, {post: (...args) => {
+   postCalls.push(args);
    return fakeRes;
  }})
  var result = remoteAdd(2, 2);
+ expect(postCalls.length).toBe(1);
+ expect(postCalls[0]).toEqual({url: URL, params: {a: 2, b: 2}});
  expect(result).toBe(fakeRes);
});
```

Зачем мы вообще пишем автоматизированные тесты? Для того, чтобы контролировать _изменения_ кодовой базы. В случае изменения поведения кода мы сразу увидим, что старые тесты не пройдут. Дело в том, что при достаточно большой кодовой базе различные изменения могут конфликтовать между собой и «ломать» друг друга. Автоматические тесты защищают наши изменения от случайной поломки при каких-либо правках (это может быть банальный мерж конфликтов).

Давайте изменим тестируемые функции и посмотрим, что произойдёт в обоих случаях:

```diff
function add(a, b) {
- return a + b;
+ return {result: a + b};
}
```

Тест, конечно же, упадёт — `4` не равно `{result: 4}`.

Теперь давайте изменим `remoteAdd`:

```diff
function remoteAdd(a, b) {
+ writeFile('log', [a, b]);
  return HttpClient.post(a, b);
}
```
Результат работы функции изменился — теперь она ещё и пишет в файл. Однако наш тест совершенно этого не заметил и продолжает проходить как ни в чём не бывало.

![My tests is passed](/images/side-effects/bad-tests.png)

Так как функции с сайд-эффектами зависят от внешнего мира и влияют на него неявно, то, соотвественно, чтобы корректно протестировать такую функцию, необходимо:

 - полностью смоделировать _весь_ окружающий мир (сетевые запросы, состояние `DOM`, состояние файловой системы) в виде некоторого _значения_ до вызова функции;
 - вызвать функцию;
 - проверить состояние значения, моделирующего _весь_ внешний мир.

Очевидно, что сделать это полностью корректно, скорее всего, _невозможно_, так как количество типов сайд-эффектов в `JS` не ограничено.

Ситуацию может немного исправить соглашение, что все зависимости, которые могут исполнять сайд-эффекты, должны явно передаваться в функцию, то есть:

```diff
-function remoteAdd(a, b) {
+function remoteAdd(a, b, HttpClient) {
  return HttpClient.post(a, b);
}
```

Добавим `writeFile`:

```diff
-function remoteAdd(a, b, HttpClient) {
+function remoteAdd(a, b, HttpClient, writeFile) {
+ writeFile('log', [a, b]);
  return HttpClient.post(a, b);
}
```

Наш тест упадёт с `undefined is not function`, так как мы не передали `writeFile`. Отлично!

Какие проблемы у подобного подхода:

 - мы не можем точно определить, какие зависимости делают сайд-эффект, а какие нет. Поэтому мы будем передавать абсолютно _все_ зависимости таким образом. Очевидно, что делать это самостоятельно невозможно, поэтому нам понадобится [специальная система модулей](https://docs.angularjs.org/guide/module#!). Данный подход получил название Dependency Injection.
 - даже с такой системой мы всё ещё можем написать плохой мок в тестах — к примеру, полениться проверить вызов метода `.put` у `HttpClient`. Да, мы сделали описание зависимости от внешнего мира более явным. Но у нас всё ещё нет стандартного способа сравнить состояние мира до и после, как мы можем сделать это с данными при помощи стандартной операции глубокого сравнения. Эту операцию мы будем вынуждены писать заново каждый раз для каждой зависимости и, скорее всего, для каждого теста. И рано или поздно мы, вероятней всего, где-нибудь допустим ошибку.

Если бы мы хотели протестировать код с сайд-эффектами «честно», мы должны были написать что-то подобное:

```javascript
// некий объект, отслеживающий состояние внешнего мира
var world = new World()
var result = remoteAdd(world, a, b)
expect(result).toEqual(output)
// все изменения внешнего мира с момента создания объекта
expect(world.changes()).toEqual(sideEffects)
```

Тогда бы мы получили возможность сверять выполненные сайд-эффекты в функции с ожидаемыми при помощи стандартного `.toEqual` (обычного глубокого сравнения объектов). Простая передача зависимостей через аргументы (и DI как её следствие) не даёт нам такой возможности — это половинчатое решение. С одной стороны, оно не решает проблему тестирования до конца, с другой — вносит определённое усложнение в наш код: к статической системе модулей добавляется ещё одна, динамическая. Так случилось потому, что вместо устранения причины проблемы (наличия сайд-эффектов) мы пытались исправить лишь её последствия (сложности тестирования).

[Более подробная статья, почему моки — это сложно](http://rea.tech/to-kill-a-mockingtest/)

Альтернативным способом решения проблемы является написание полностью интеграционных тестов с полноценным браузерным или серверным окружением. Помимо всё тех же проблем со сложностью создания и сравнения состояния окружения (к примеру, как сравнить весь `DOM` до некоторой операции и после?) добавляются ещё и следующие проблемы:

 - подобные тесты медленно запускаются, долго работают, тратят кучу электричества, работают ненадёжно;
 - нет возможности хоть как-то локализовать проблему: можно понять, что что-то не работает, но нельзя понять, почему.

#### Cайд-эффекты непредсказуемы и не воспроизводимы

> Всё течёт, всё меняется, никто не может дважды войти в один и тот же поток, и к смертной сущности никто не прикоснется дважды!
>
> -- <cite>Гераклит</cite>

Станете ли вы заворачивать вызов `add(2, 2)` в блок `try/catch`? Думаю, нет — в этом нет смысла.
А `divide(a, b)`? Да, конечно — при `b === 0` произойдет ошибка.

Может ли произойти ошибка при вызове `remoteAdd(2, 2)`, и если да, то при каких входных параметрах? Да, может, при любых параметрах. А может и не произойти. Мы не знаем и никак не можем повлиять на это. Внешний мир непредсказуем, он может сломаться в любой момент: браузер может упасть, сеть может погаснуть, а сервер — сгореть.

Из-за того, что внешний мир непредсказуем и постоянно изменяется, мы также не можем воспроизвести результаты вычислений, содержащих сайд-эффекты.

`add(a, b) === add(a, b)` будет всегда истинно в любых условиях и окружении. Мы можем легко воспроизвести результаты некоторой проблемы с продакшена, просто взяв, к примеру, входные данные с мониторинга и запустив вычисления с этими параметрами. Сайд-эффекты приводят к невоспроизводимым багам: чтобы понять, в чём была проблема, нам надо проанализировать не только наш код, но и состояние всего окружающего мира в тот момент. Это намного более трудоёмко, а порой и вообще невозможно.

#### Cайд-эффекты не типизируются

![Undefined is not a function](/images/side-effects/undefined-is-not-function.jpg)

Одним из способов контроля кодовой базы и доказательства отсутствия нежелательного поведения программы является статическая типизация.

Очень много копий сломано вокруг того, [нужна ли она вообще](https://habrahabr.ru/post/192108/) или [нет](https://medium.com/javascript-scene/the-shocking-secret-about-static-types-514d39bf30a3#.kzfrsyeim). 

Моё мнение простое: статическая типизация — это инструмент, незаменимый для написания некоторого типа ПО — такого, где очень много кода, много программистов, много связанных подсистем.

Однако вопрос не в этом, а в том, как на использование типизации влияют сайд-эффекты. Рассмотрим пример: предположим, мы имеем функцию, которая по описанию изменения как-то модифицирует `DOM`: 

```javascript
function patchDOM(patch: DOMPatch): void { ... }
``` 

Неявно эта функция зависит от существования `DOM`. И её результатом является его изменение. Однако мы никак не можем описать эту информацию в типах — ни о неявной зависимости от `DOM`, ни о том, что её результатом будет его изменение. В результате, если мы случайно применим эту функцию в окружении без DOM, то получим ошибку при исполнении:

```javascript
function serverProgram() {
  ...
  patchDOM(patch); // run-time error
}
```

Сайд-эффекты за счет своей неявности не поддаются описанию типами, и тайп-чекер не может помочь найти проблемы с ними.

Возможным решением является использование [имитации Higher-Kinded types](https://medium.com/@gcanti/higher-kinded-types-in-flow-275b657992b7#.430oe011t) для реализации [типа `Eff`](https://medium.com/@gcanti/the-eff-monad-implemented-in-flow-40803670c3eb#.521yqwoyu) при помощи `Flow`:

```javascript
type DOM = { type: 'DOM' };
function patchDOM(patch: DOMPatch): Eff<{ write: DOM }, void> { ... }
```

И, соотвественно, `serverProgram` просто не скомпилируется, если у неё в типе не будет указан `Eff` типа `{ write: DOM }`, а внутри неё будет использоваться `patchDOM`.

Однако:

 - этот способ полагается на не самые очевидные механизмы `Flow` и довольно сложен для понимания;
 - он не поможет в случае наличия сайд-эффектов в коллбэках или других местах, из которых нельзя _вернуть_ результат;
 - такие типы не будут выведены для javascript API (`console`, `XMLHttpRequest` и так далее).


В общем, решение не общее и не самое простое.

#### Усложнение интерактивной разработки кода с сайд-эффектами

Интерактивная разработка начинает набирать популярность. Практически все более-менее популярные языки имеют в стандартной поставке [REPL](https://ru.wikipedia.org/wiki/REPL) (отдельно или в составе дебаггера). Современные браузеры вообще позволяют [писать код прямо в них](http://www.hongkiat.com/blog/google-chrome-workspace/).

Появляются и отдельные IDE, нацеленные именно на интерактивную разработку. К примеру, [Light table](http://lighttable.com), позволяющая в реальном времени следить за результатами вычислений:

![Light Table](/images/side-effects/watches.png)

[Отличная статья о том, почему интерактивная разработка — это прекрасно!](http://tonsky.me/blog/interactive-development/)

Давайте посмотрим, какие коррективы вносят сайд-эффекты в подобную практику. Допустим, мы разрабатываем функцию, осуществляющую запрос на удаление некоторого юзера — `deleteUser`. Очевидно, что мы не сможем запустить эту функцию несколько раз для одного и того же юзера, чтобы проверить её работу в REPL. Более того, для проверки результатов её работы нам понадобится каждый раз смотреть текущее состояние сервера.

Главное преимущество интерактивной разработки — быстрая ответная реакция от только что написанного кода — в таком случае сводится на нет тем, что нам необходимо постоянно наблюдать состояние окружающего мира и периодически исправлять его. Например, восстанавливать удалённого юзера.

Возможным решением здесь будет REPL, интегрированный в тестовый фреймворк: `jest-repl`, `mocha debug repl` со встроенной возможностью устанавливать моки для определённых сайд-эффектов.

#### Для кода с сайд-эффектами сложно применить тестирование, основанное на проверке свойств

Тестирование, основанное на проверке свойств, или генеративное тестирование, или property-based тестирование — техника, позволяющая описывать свойства какой-то программной сущности (функции, к примеру) и проверять её при помощи генерации входных параметров.

Это очень мощная техника, которая позволяет доказать (с некоторой долей вероятности, конечно же) некоторые утверждения о программном коде. Она становится особенно важной в языках без сильной статической системы типов. 

Но, как я отметил в [предыдущей своей статье](http://blog.csssr.ru/2017/04/25/property-testing/), крайне сложно определить какое-либо свойство для кода с сайд-эффектами — за счет их непредсказуемости.

## Мелкие вредители, вредящие по-крупному

![Gremlin](/images/side-effects/gremlins.png)

Сайд-эффекты — как гремлины, ломают все доступные программисту инструменты, до которых доберутся: типизация, тесты, интерактивные среды, статические анализаторы в IDE, код-ревью.

Но почему это всё действительно важно? Ну да, что-то стало сложнее сделать, но программисты привыкли к борьбе со сложностями. И в отдельных пунктах я приводил результаты такой борьбы — инструменты, которые призваны хоть как-то уменьшить негативное влияние сайд-эффектов.

Проблема в том, что сайд-эффекты не только усложняют написание и работу с самим кодом, но и «ломают» два _базовых_ способа разработки ПО. И это уже реальная проблема.

## Два ~~сломанных~~ способа разработки ПО 

На самом _высоком_ уровне, по большому счету, существует только [два способа разработки](https://en.wikipedia.org/wiki/Top-down_and_bottom-up_design). Всё остальное — либо их комбинации, либо производные.

![Top Down or Bottom up](/images/side-effects/vityaz.png)

Давайте рассмотрим на простом примере оба способа. Предположим, что нам надо разработать систему `printSum`, которая будет выводить на экран сумму какого-то списка.

#### «Сверху-вниз» a.k.a Нисходящий стиль a.k.a Top-Down
 
Определяем спецификацию самого верхнего уровня API — описываем входное и выходное воздействие: 

```javascript
printSum(list: Array): PrintedSumToScreenEffect
```
Затем определяем спецификации API уровнем ниже, которые необходимы, чтобы из `Array` получить `PrintedSumToScreenEffect`. Очевидно, что нам требуются две функции:

```javascript
sum(list: Array): number
printToScreen(value: number): PrintedSumToScreenEffect
```

Теперь, просто глядя на спецификации, мы понимаем, что сначала необходимо вызывать от `list` функцию `sum`, а затем от её результата —  `printToScreen`.

Осталось придумать, как записывать спецификацию для описания входных и выходных данных: 

 - В идеале она должна описывать все возможные входные воздействия и результаты довольно общим образом — тесты подходят не очень хорошо, так как они описывают отдельные кейсы вместо общего поведения системы.
 - Мы должны иметь возможность проверить, что разработанная нами программа соответствует изначальной спецификации, иначе при имплементации есть большой шанс получить значительное расхождение со спецификацией. Диаграммы и прочие способы, связанные с [ИЗО](https://ru.wikipedia.org/wiki/UML), не дадут нам такой возможности.

Вы, наверно, уже догадались, что лучше всего здесь [подойдёт хорошая система типов](http://blog.ploeh.dk/2015/08/10/type-driven-development/).

> Люди иногда спрашивают: «Что служит аналогом UML для Haskell?». Когда меня впервые спросили об этом 10 лет назад, я подумал: «Ума не приложу. Может быть, нам стоит придумать свой UML». Сейчас я думаю: «Это просто типы!». Люди рисуют UML-диаграммы, чтобы понять общую схему программы. Именно этим занимаются программисты на функциональных языках и на Haskell, когда придумывают сигнатуры типов для модулей и функций в этих модулях.
>
> -- <cite>[Саймон Пейтон Джонс](http://fprog.ru/2010/issue6/interview-simon-peyton-jones/)</cite>

Однако, как мы выяснили, без определённых уловок большая часть систем типов не способны работать с сайд-эффектами и уж точно не могут вывести типы таких эффектов из контекста. Можно было бы, конечно, заменить типы тестами (что, например, сделано во всем известном Test Driven Development), но, как мы уже увидели, тесты не очень хорошо подходят для этого из-за своей дискретной природы, к тому же они тоже страдают от сайд-эффектов.
 
Таким образом сайд-эффекты ломают первый базовый способ разработки ПО. Но, может, со вторым нам повезёт больше?

#### «Снизу-вверх» a.k.a Восходящий стиль a.k.a Bottom-Up

Мы можем пойти с другой стороны.

Уже по описанию задачи видно, что нам надо будет уметь выводить что-то на экран и надо уметь складывать. Мы не будем пытаться определить точные спецификации. Вместо этого просто напишем общие и минимально необходимые функции для этого. Больше всего эти функции будут похожи на отдельные небольшие библиотеки (очень малоспецифичные и очень переиспользуемые единицы), так как мы ещё не знаем, что за API нам придётся с их помощью строить.

```javascript
sum(1, 2, 3, 4)
// -> 10
print('Hello', screen)
// -> prints 'Hello' to screen
```

Затем строим из этих функций API более высокого уровня:

```javascript
function printSum(list)
  print(sum(...list), screen)
}
```

Для такого итеративного процесса нам необходим инструмент, позволяющий легко экспериментировать с небольшими кусочками кода и иметь возможность быстро запустить отдельные функции на разных входных данных. При этом в процессе разработки нам не так важно зафиксировать некоторый результат и уметь его воспроизводить — вероятней всего, мы редко будем менять имплементацию уже написанного API. Тесты тут будут скорее мешать низкой скоростью ответной реакции и своей хрупкостью.

Лучше всего для такого стиля разработки подойдёт `REPL`. Собственно, такой вид разработки и получил распространие в языках с богатой практикой использования `REPL`: `Lisp`, `SmallTalk`.

Однако, как мы помним, `REPL` теряет свое главное преимущество (быстрый отклик) при разработке кода с сайд-эффектами. Второй фундаментальный способ тоже не выдержал этой битвы.

![Dead knigth](/images/side-effects/vityaz-no-way.png)

Но ведь не весь наш код содержит сайд-эффекты? И, может, все эти неприятности касаются только тех участков, в которых мы их используем? Может, просто стараться использовать поменьше «грязных» функций и побольше «чистых»?

## Лед-9 для программного кода

![Cats cradle](/images/side-effects/cats-cradle.jpg)


> \- Ты читал «Колыбель для кошки»?
>
> \- Нет.
>
> \- Итак, в этом романе мир погибает потому, что во льду обнаружена молекула, которая при соприкосновении с водой превращает её в лед. А поскольку все воды мира связаны — пруд с ручьем, ручей с рекой, река с озером, озеро с океаном — таким образом, весь мир замерзает и погибает. И эта молекула называется «Лед-9».
>

Программисты, заставшие `JS` без `Promise` и `async-await`, могут почувствовать что-то знакомое в описании недостатков сайд-эффектов. Эти же проблемы зачастую упоминались в обсуждении асинхронных функций, основанных на коллбэках.

> Асинхронность и сайд-эффекты выглядят довольно связанными проблемами — решив только одну из них, вы не избавитесь от всех их недостатков, они лишь переместятся. С другой стороны, хорошее решение одной из этих проблем может помочь решить другую. В дальнейшем мы увидим, что это не случайно и на самом деле оба этих явления представляют собой лишь частные случаи более фундаментальной проблемы отсутствия общего способа абстракции control-flow программы и выражения его first-class значениями.

Но самое ужасное в таких функциях было то, что они заражали весь код, в котором использовались. Обычная функция при использовании в ней функции с коллбэком переставала возвращать результат через `return` и начинала прокидывать его в коллбэк — и, в свою очередь, сама становилась «ядом» для использующего её кода.

Появилась даже метафора двухцветного языка:

 - [Статья Bob Nystrom](http://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)
 - [Доклад Андрея Саломатина](https://www.youtube.com/watch?v=OGSppLmGchY)

```javascript
blue•function doSomethingAzure() {
  // This is a blue function...
}
red•function doSomethingCarnelian() {
  // This is a red function...
}

blue•function doSomethingAzure() {
  doSomethingCarnelian()•red;  // Error — you can't call red inside blue
}
```

Все функции в языке делятся на «красные» и «синие». И чтобы вызвать «красную» функцию в «синей», нам необходимо перекрасить «синюю» функцию в красный цвет.

Происходило это из-за того, что подобные асинхронные функции для взаимодействия с кодом не использовали стандартные способы взаимодействия — возврат значения через `return`. Как мы помним, функции с сайд-эффектами ведут себя так же и возвращают свои результаты неявно.

Поэтому всё то же самое применимо и к ним:

> Функция с сайд-эффектом внутри сама становится сайд-эффектом. Следовательно, используя такую функцию внутри другой (чистой), мы автоматически превращаем её в грязную — она начинает возвращать часть результата неявно. И так далее до самой вершины стека вызовов.

Рассмотрим пример. Допустим, у нас есть такая иерархия вызовов:

![Pure tree](/images/side-effects/pure_tree.svg)

Все вызовы чистые и предсказуемые. С ними нет никаких проблем.

Но неожиданно нам понадобилось кэшировать результаты вычисления функции `calcForItem` в `localStorage`:

```javascript
function calcForItem(...) {
  ...
  var cachedResult = localStorage
    .getItem(argsHash);
  if (cachedResult) {
    return cachedResult;
  } else {
    ...
    localStorage.setItem(argsHash, res)
    return res;
  }
}
```

И наша иерархия стала выглядеть так (красным отмечены функции с сайд-эффектами):

![Not pure tree](/images/side-effects/not_pure_tree.svg)

Изменив код всего _одной_ функции, мы изменили свойства (в плане тестируемости, надёжности, композируемости) для _всего_ стека вызовов.

В некотором смысле мы теряем _контроль_ над своим кодом. Его поведение может измениться, хотя он сам останется прежним — просто API, на котором основан наш код, _внезапно_ станет «грязным» и «заразит» его.

## In Soviet Russia side-effects control you!

Если мы не контролируем ПО, то оно начинает контролировать нас (разработчиков). В результате всего вышесказанного у нас получается код, который:

 - _нельзя_ полноценно протестировать, да и, чтобы протестировать хоть как-то, нужно приложить много усилий — из-за этого мы пишем недостаточно тестов. Проверьте свое покрытие кода с сайд-эффектами и кода, работающего только с данными — скорее всего, во втором случае цифра будет намного выше;

 - _нельзя_ полноценно типизировать, потому что значительную часть фронтенд-кода занимают функции типа `handleClick(): void`, `dispatch(): void`, `setState(): void`;

 - _нельзя_ верифицировать или попробовать доказать его свойства при помощи property-based тестов — у большей его части просто нет каких-либо предсказуемых свойств;

 - неудобен для работы в интерактивной среде (`REPL`), потому что там не получится работать с DOM-элементами или безопасно послать HTTP-запрос;

 - практически не поддается рефакторингу, так как сайд-эффекты создают неявные зависимости;

 - непредсказуем — невозможно создать надёжные инструменты воспроизведения поведения приложения. Обычно не является проблемой по результатам мониторинга понять, _что_ произошо. Но вот _почему_ так произошло — может быть совершенно неясно, так как части приложения влияют друг на друга неявно;

 - очень тяжело переиспользуется. Иногда без внесения правок в исходники нельзя переиспользовать то или иное решение. Поэтому наученные горьким опытом разработчики зачастую стремятся не переиспользовать крупные части своего приложения (не говоря уже о больших и сложных сторонних компонентах), так как не ясно, возможно ли будет избавиться от некоторых нежелательных действий, если они вдруг станут не нужны.

При этом разработчики сами по себе не хотят писать такой код. Почти любой разработчик скажет, что код должен быть хорошо тестируемым, переиспользуемым и так далее. Нас учат этому с самого начала работы в индустрии.

Однако сайд-эффекты в нашем коде вынуждают нас частично отказаться от всех хороших практик и практически всех доступных программисту инструментов.

Софт, который контролирует людей? Я знаю, кому это точно понравится:

![Skynet](/images/side-effects/skynet.png)

При этом мы не пытаемся избавиться от этого контроля и воспринимаем его как что-то само собой разумеющееся. Все эти проблемы давно известны, но мы не пытаемся воздействовать на их _причину_, а только исправляем отдельные симптомы:

 - `React` стал популярен во многом благодаря тому, что абстрагировал часть сайд-эффектов, возникающих при работе с `DOM`. Но только часть — [работа с событиями](https://facebook.github.io/react/docs/handling-events.html) и [работа с отдельными элементами](https://facebook.github.io/react/docs/refs-and-the-dom.html) всё так же происходят при помощи сайд-эффектов.
 - `Redux` позволил описывать преобразования глобального мутабельного стейта при помощи [чистых функций](http://redux.js.org/docs/basics/Reducers.html). Однако сами изменения вызываются при помощи грязной функции [`dispatch`](http://redux.js.org/docs/api/Store.html#dispatch). Да и, к примеру, то же самое сетевое взаимодействие всё так же продолжает порождать сайд-эффекты (хотя это можно исправить при помощи [`redux-loop`](https://github.com/redux-loop/redux-loop), [`redux-saga`](https://github.com/yelouafi/redux-saga)).
 - `Angular 2`, напротив, позволяет обрабатывать [события без использования коллбэков (то есть без сайд-эффектов)](http://learnangular2.com/outputs/). Но он не имеет какого-либо решения для абстракции остальных сайд-эффектов.

## I have a dream...

Попытки решать фундаментальную проблему, борясь только с её отдельными проявлениями, вряд ли могут увенчаться успехом.

> Нам необходимо фундаментальное решение проблемы сайд-эффектов в нашем коде. Причём оно должно было максимально общим — не привязанным к какому-то фреймворку или инфраструктуре и не навязывающим какую-либо конкретную архитектуру. Оно должно решать ровно одну проблему и делать это хорошо.
>

Но как мы можем создать такое решение?

Мы не можем избавиться от сайд-эффектов совсем или даже уменьшить их число — нельзя вдруг начать делать меньше `HTTP` запросов или меньше работать с `DOM`. Сайд-эффекты — взаимодействие с внешним миром — это вообще самая главная часть нашего приложения. Если приложение не делает их, значит, скорее всего, оно вообще ничего не делает.

![No side effects](/images/side-effects/haskell.png)

Но мы можем _полностью_ отделить логику нашего приложения от сайд-эффектов.

Всё наше приложение станет полностью чистой функцией — будет явно принимать все входящие воздействия и явно возвращать выходные. А сайд-эффекты будут исполняться отдельно от основного приложения.

Существует минимум 3 способа сделать это, и все они основаны на теоретических основах Computer Science, разработанных около 40 лет назад.

Все эти способы объединяет то, что они созданы для абстракции `control-flow` программы. `Control-flow` — это скелет нашей программы, её базис, поэтому эти способы возникают при решении большей части проблем в Computer Science — не только при решении проблемы сайд-эффектов. Однако это тема для следующей статьи.

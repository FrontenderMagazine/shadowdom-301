# Теневая модель документа: продвинутые концепции и DOM API

В этой статье продолжается описание удивительных возможностей теневой модели 
документа (Shadow DOM). Оно построено на концепциях, описанных в [первой][1] и 
[второй][2] статье о теневой модели документа.

## Использование нескольких корневых элементов теневого дерева

Когда вы организовываете дома вечеринку с большим количеством приглашённых, она 
может превратиться в столпотворение, если все набьются в одну комнату. Вам нужна 
возможность распределить группы людей по разным комнатам. То же самое возможно и 
в теневой модели документа, тоесть ведущие элементы теневого дерева могут 
содержать больше одного корневого элемента. 

Давайте посмотрим что случится если прикрепить несколько корневых элементов к 
одному ведущему:

    <div id="example1">Ведущий элемент</div>
    <script>
    var container = document.querySelector('#example1');
    var root1 = container.webkitCreateShadowRoot();
    var root2 = container.webkitCreateShadowRoot();
    root1.innerHTML = '<div>Первый корневой элемент</div>';
    root2.innerHTML = '<div>Второй корневой элемент</div>';
    </script>

![Прикрепление нескольких теневых деревьев][деревья]

<div class="demo">
  <div id="example1">Ведущий элемент</div>
</div>

<script src="js/example-1.js"></script>

**Подсказка:** Активируйте «Отображать теневую модель документа (Show Shadow 
DOM)» в инструментах разработчика чтобы иметь возможность просматривать корневые 
элементы теневого дерева.

Отображается «Второй корневой элемент», несмотря на тот факт, что мы уже 
подключили теневое дерево. Это происходит потому, что побеждает последнее дерево, 
добавленное в ведущий элемент. Когда дело касается отображения, здесь 
применяется принцип LIFO<a href="#note-1" class="reference">1</a>. Инструменты 
разработчика подтверждают такое поведение.

Теневые деревья, добавленные в ведущий элемент, образуют стек в порядке 
добавления, последнее добавленное оказывается в начале. Последнее добавленное 
дерево и отображается.

> Последнее добавленное дерево называют «младшим деревом», то, что было 
добавлено ранее, называют «старшим деревом». В этом примере `root2` — это 
младшее дерево, а `root1` — старшее.

Так зачем прописывать несколько теневых деревьев если отображается только 
последнее? Читайте дальше в разделе о теневых точках вставки. 

### Теневые точки вставки

«Теневые точки вставки» (через `<shadow>`) похожи на обычные точки вставки 
(через `<content>`) тем, что тоже являются заполнителями. Однако вместо того, 
чтобы выступать заполнителями *содержимого* ведущего элемента, они служат 
ведущими элементами для других *теневых деревьев*. 

Как вы уже должно быть догадались, чем дальше в лес, тем всё стаёт ещё более 
запутанным. Потому в спецификации очень чётко прописано что происходит когда 
используется несколько элементов `<shadow>`:

> Если в теневом дереве прописаны несколько точек вставки через `<shadow>`, 
учитывается только первая из них. Все остальные игнорируются.

Оглядываясь на наш первый пример, мы видим что первый корневой элемент `root1` 
был проигнорирован. Добавление точки вставки через `<shadow>`, возвращает его в 
игру:

    <div id="example2">Ведущий элемент</div>
    <script>
    var container = document.querySelector('#example2');
    var root1 = container.webkitCreateShadowRoot();
    var root2 = container.webkitCreateShadowRoot();
    root1.innerHTML = '<div>Первый корневой элемент</div><content></content>';
    root2.innerHTML = '<div>Второй корневой элемент</div><shadow></shadow>';
    </script>

![Теневые точки вставки][точки вставки]

<div class="demo">
  <div id="example2">Ведущий элемент</div>
</div>

<script src="js/example-2.js"></script>

В этом примере есть несколько интересных особенностей:

1. «Второй корневой элемент» отображается над «Первый корневой элемент». Это 
происходит из-за расположения точки вставки через `<shadow>`. Если вам нужно 
обратное, переместите точку вставки: `root2.innerHTML = '<shadow></shadow><div>
Второй корневой элемент</div>';`.
2. Обратите внимание, что в `root1` теперь добавлена точка вставки через 
`<content>`. Благодаря этому отображается текстовый узел «Ведущий элемент».

**Что отображается на месте `<shadow>`?**

Иногда нужно знать какое теневое дерево генерируется вместо `<shadow>`. К этому 
дереву можно обратиться через `.olderShadowRoot`:

    root2.querySelector('shadow').olderShadowRoot === root1 //true

Для `.olderShadowRoot` не используется вендорный префикс, потому что 
`HTMLShadowElement` целесообразно использовать только в контексте теневой модели 
документа… для которой уже добавлен префикс :).

## Обращение к корневому элементу ведущего элемента

Если в элемент помещено теневое дерево, можно обратиться к его [младшему 
корневому элементу][4] с помощью `.webkitShadowRoot`:

    var root = host.webkitCreateShadowRoot();
    console.log(host.webkitShadowRoot === root); // true
    console.log(document.body.webkitShadowRoot); // null

> Я не уверен зачем в спецификации прописан `.shadowRoot`. Он сводит на нет 
принципы инкапсуляции теневой модели документа и даёт посторонним отдушину для 
просмотра дерева, которое должно было быть скрытым.

Если вы не хотите чтобы кто-то копался в ваших теневых деревьях, переопределите 
значение `.shadowRoot` на `null`:

    Object.defineProperty(host, 'webkitShadowRoot', {
      get: function() { return null; },
      set: function(value) { }
    });

Это своего рода хак, но зато он работает. Сильные мира сего также ищут способ 
сделать теневое дерево скрытым.

И наконец, важно помнить, что несмотря на свою чудесно-фантастическую сущность, 
**Теневая модель документа не задумывалась как средство безопасности**. Не 
рассчитывайте на неё когда нужно полностью изолировать контент. 

## Создание теневой модели документа на JavaScript

Если вы предпочитаете создавать теневой DOM на JavaScript, это возможно с 
помощью `HTMLContentElement` и `HTMLShadowElement`:

    <div id="example3">
      <span>Ведущий элемент</span>
    </div>
    <script>
    var container = document.querySelector('#example3');
    var root1 = container.webkitCreateShadowRoot();
    var root2 = container.webkitCreateShadowRoot();

    var div = document.createElement('div');
    div.textContent = 'Root 1 FTW';
    root1.appendChild(div);

     // HTMLContentElement
    var content = document.createElement('content');
    content.select = 'span'; // выбор всех элементов span в ведущем элементе
    root1.appendChild(content);

    var div = document.createElement('div');
    div.textContent = 'Root 2 FTW';
    root2.appendChild(div);

    // HTMLShadowElement
    var shadow = document.createElement('shadow');
    root2.appendChild(shadow);
    </script>

Этот пример почти идентичен представленному в предыдущей части. Единственная 
разница в том, что сейчас я использую `select` чтобы выдернуть новый `<span>`.

## Извлечение распределённых узлов

Узлы, выбранные в ведущем элементе и «распределённые» в теневое дерево 
называются… барабанная дробь… распределёнными узлами! Они могут пересекать 
границу теневого дерева после вызова точкой вставки.

Немного странным является то, что точки вставки не перемещают элементы DOM 
физически. Ведущие узлы остаются нетронутыми. Точки вставки просто 
перепроецируют узлы из ведущего элемента в теневое дерево. Это дело 
представления/генерации: <span style="text-decoration:line-through">«Переместить 
эти узлы вот сюда»</span> «Сгенерировать эти узлы вот здесь». 

Нельзя переместить DOM в `<content>`.

Например:

    <div><h2>Ведущий узел</h2></div>
    <script>
    var shadowRoot = document.querySelector('div').webkitCreateShadowRoot();
    shadowRoot.innerHTML = '<content select="h2"></content>';

    var h2 = document.querySelector('h2');
    console.log(shadowRoot.querySelector('content[select="h2"] h2')); // null;
    console.log(shadowRoot.querySelector('content').contains(h2)); // false
    </script>

Вуаля! `h2` не является дочерним элементом теневого дерева. Это ведёт к ещё 
одному интересному факту:

> Точки вставки необычайно могущественны. Рассматривайте их как способ создать 
«декларативный API» для теневого дерева. Ведущий элемент может содержать всю 
разметку мира, но до того как я привяжу её к теневому дереву посредством точки 
вставки, она бессмысленна.

## `Element.getDistributedNodes()`

Нельзя переместить код в `<content>`, однако `.getDistributedNodes()` API 
позволяет запрашивать распределённые узлы в точке вставки:

    <div id="example4">
      <h2>Эрик</h2>
      <h2>Бидельман</h2>
      <div>Цифровой джедай</div>
      <h4>текст подвала</h4>
    </div>

    <template id="sdom">
      <header>
        <content select="h2"></content>
      </header>
      <section>
        <content select="div"></content>
      </section>
      <footer>
        <content select="h4:first-of-type"></content>
      </footer>
    </template>

    <script>
    var container = document.querySelector('#example4');

    var root = container.webkitCreateShadowRoot();
    root.appendChild(document.querySelector('#sdom').content.cloneNode(true));

    var html = [];
    [].forEach.call(root.querySelectorAll('content'), function(el) {
      html.push(el.outerHTML + ': ');
      var nodes = el.getDistributedNodes();
      [].forEach.call(nodes, function(node) {
        html.push(node.outerHTML);
      });
      html.push('\n');
    });
    </script>

<div id="example4" style="display:none">
  <h2>Эрик</h2>
  <h2>Бидельман</h2>
  <div>Цифровой джедай</div>
  <h4>текст подвала</h4>
</div>

<p><template id="sdom">
  <header>
    <content select="h2"></content>
  </header>
  <section>
    <content select="div"></content>
  </section>
  <footer>
    <content select="h4:first-of-type"></content>
  </footer>
</template></p>
<div id="example4-log" class="demo">
 <textarea readonly></textarea>
</div>

<script src="js/example-4.js"></script>

## Инструмент: Визуализатор теневой модели документа

Понять чёрную магию, коей является теневая модель документа, непросто. Помню как 
я сам в первый раз ломал голову над тем, как она работает.

Я создал инструмент на основе [d3.js][6] для наглядной демонстрации работы 
теневой модели документа. Оба поля для разметки с левой стороны можно 
редактировать. Вы можете вставить туда свою разметку и поэкспериментировать с 
ней, чтобы проверить как всё работает и как точки вставки втискивают ведущие 
узлы в теневое дерево.

![Визуализатор теневой модели документа][Визуализатор] 

[Запустить визуализатор теневой модели документа][7]

<iframe src="http://www.youtube.com/embed/qnJ_s58ubxg" allowfullscreen="" frameborder="0" height="315" width="420"></iframe>

Попробуйте сами и дайте мне знать что вы о нём думаете!

## Модель обработки событий

Некоторые события могут пересечь границу теневого дерева, а другие — нет. В тех 
случаях, когда события пересекают границу, приемник событий подстраивается для 
сохранения инкапсуляции, обеспечиваемой верхней границей корневого элемента 
теневого дерева. Тоесть, **события перенастраиваются так, чтобы казалось будто 
они запущены ведущим элементом, а не внутренними элементами теневого дерева**.

Если ваш браузер поддерживает теневую модель документа<a href="#note-1" class="reference">2</a>, 
вы должны увидеть внизу площадку для визуализации событий. Элементы жёлтого 
цвета являются частью разметки теневого дерева. Элементы синего цвета — частью 
ведущего элемента. Желтая рамка вокруг «Я узел в ведущем элементе» обозначает 
что это распределённый узел, вставленный через точку вставки `<content>`.

Кнопки «Воспроизведение события» показывают какие действия можно попробовать 
применить.  


<link rel="stylesheet" href="css/example5.css">

<div id="example5" class="demo">
  <div data-host>
    <div class="blue">Я узел в ведущем элементе</div>
  </div>

  <template style="display:none;"><!-- display:none used for older browsers -->
    <style>
    .scopestyleforolderbrowsers * {
      border: 4px solid #FC0;
    }
    .scopestyleforolderbrowsers input {
      padding: 5px;
    }
    .scopestyleforolderbrowsers div {
      background: #FC0;
      padding: 5px;
      border-radius: 3px;
      margin: 5px 0;
    }
    content::-webkit-distributed(*) {
      border: 4px solid #FC0;
    }
    </style>
    <section class="scopestyleforolderbrowsers">
      <div>Я узел в теневом дереве</div>
      <div>Я узел в теневом дереве</div>
      <content></content>
      <input type="text" placeholder="Я в теневом дереве">
      <div>Я узел в теневом дереве</div>
      <div>Я узел в теневом дереве</div>
    </section>
  </template>

  <aside class="cursor"></aside>

  <div class="buttons">
    <button data-action="playAnimation" data-action-idx="1">Воспроизвести событие 1</button><br>
    <button data-action="playAnimation" data-action-idx="2">Воспроизвести событие 2</button><br>
    <button data-action="playAnimation" data-action-idx="3">Воспроизвести событие 3</button><br>
    <button data-action="clearLog">Очистить лог</button>
  </div>

  <output></output>
</div>

<script src="js/example-5.js"></script>

**Воспроизведение события 1**

* Это интересно. Вы должны увидеть выполнение события `mouseout` при переходе от 
ведущего элемента (`<div data-host>`) к узлу синего цвета. Хоть это и 
распределённый узел, он все-таки принадлежит ведущему узлу, а не теневому дереву. 
Перемещение курсора далее вниз на жёлтое поле опять запускает событие `mouseout` 
для узла синего цвета. 

**Воспроизведение события 2**

* Запускается одно событие `mouseout` для ведущего элемента (в самом конце). По 
идее, вы должны увидеть запуск события `mouseout` для всех жёлтых блоков. Однако, 
в этом случае эти элементы являются внутренними по отношению к теневому дереву и 
событие не проходит через его верхнюю границу. 

**Воспроизведение события 3**

* Обратите внимание что после клика по полю ввода, событие `focusin` выполняется 
не для поля вода `input`, а для ведущего элемента. Оно перенастраивается!

### События, которые всегда останавливаются

Следующие события никогда не пересекают границу теневого дерева:

* `abort`
* `error`
* `select`
* `change`
* `load`
* `reset`
* `resize`
* `scroll`
* `selectstart`

## Заключение

Надеюсь вы согласитесь, что **Теневая модель документа необычайно могущественна**. 
Впервые нам доступна надлежащая инкапсуляция без `<iframe>` или других приёмов в 
нагрузку.

Теневая модель документа — несомненно зверь непростой. Но она стоит того, чтобы 
быть добавлена в веб-платформу. Уделите ей время. Изучите её. Задавайте вопросы.

Если вы хотели бы узнать больше, прочитайте [вступительную статью Доминика о 
Теневой модели документа][8] и мою статью [«Теневая модель документа: CSS и 
стили»][9].

---

### Примечания

<a href="#note-1" id="note-1" class="reference">1</a> LIFO (*от англ. Last In, 
First Out, «последним пришёл — первым ушёл»*) — способ организации и 
манипулирования данными относительно времени и приоритетов. В структурированном 
линейном списке, организованном по принципу LIFO, элементы могут добавляться и 
выбираться только с одного конца, называемого «вершиной списка».

<a href="#note-2" id="note-2" class="reference">2</a> Для этого вам нужно 
использовать Google Chrome и активировать «Отображать теневую модель документа 
(Show Shadow DOM)» в инструментах разработчика.

[1]: /shadowdom/
[2]: /shadowdom-201/
[3]: http://www.html5rocks.com/en/tutorials/webcomponents/shadowdom/#toc-separation-separate
[4]: http://www.html5rocks.com/en/tutorials/webcomponents/shadowdom-301/#youngest-tree
[5]: http://www.html5rocks.com/en/tutorials/webcomponents/shadowdom-301/#toc-shadow-insertion
[6]: http://d3js.org/
[7]: http://html5-demos.appspot.com/static/shadowdom-visualizer/index.html
[8]: http://www.html5rocks.com/en/tutorials/webcomponents/shadowdom/
[9]: http://www.html5rocks.com/en/tutorials/webcomponents/shadowdom-201/

[деревья]: img/stacking-ru.png
[точки вставки]: img/shadow-insertion-point-ru.png
[Визуализатор]: img/visualizer.png
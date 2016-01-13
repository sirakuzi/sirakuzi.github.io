---
title: "Руководство Эмиллера по разработке модулей для Nginx"
description: "Еще одна попытка перевести это произведение."
categories: [posts]
layout: post
time: 14:54:55
---
<style type="text/css">
div.figure
{
    margin: 2em;
    text-align: center;
}
div.figure img { display: block; margin: 0 auto 1em; }
</style>

<section>
<small><p>Оригинал: <a href="http://www.evanmiller.org/nginx-modules-guide.html">Emiller's Guide To Nginx Module Development</a>.
<br>Автор: <a href="http://www.evanmiller.org/">Эван Миллер</a>
<br>Перевод: <a href="http://sirakuzi.github.io/">sirakuzi.github.io</a> на основе перевода <a href="http://kung-fu-tzu.ru/">Петра Леонова</a> черновика от 14 июля 2008 г.
<br>Первая публикация: 28 апреля 2007 г. (Последнее изменение от 16 января 2013 г. &ndash; <a href="#changes">изменения</a>)</p></small>
</section>

<section>
<ul>
<li><span style="font-variant: small-caps;">Брюс Уэйн</span>: <em>Что это?</em></li>
<li><span style="font-variant: small-caps">Люциус Фокс</span>: <em>Бэтмобиль? Ох… лучше вам не знать.</em></li>
</ul>
</section>

<section>
<p>Чтобы полностью оценить вебсервер Nginx стоит понять Бэтмена, персонажа комиксов.</p>

<p>Бэтмен быстр. Nginx быстр. Batman борется с преступностью. Nginx борется с лишней нагрузкой на процессор и утечками памяти. Бэтмен хорошо держится под натиском врага. Nginx, в свою очередь, держится превосходно при очень большой нагузке на сервер.</p>

<p>Но кем бы был Бэтмен без своего <strong>Пояса</strong>.</p>
<div class="figure">
    <img src="/i/utility-belt.jpg" alt="Пояс Бэтмена"/> 
    <strong>Рисунок 1</strong>: Пояс Бэтмена, в обнимку с пузом Кристиана Бэйла.
</div>

<p>В любой момент в Поясе Бэтмена может найтись набор отмычек, пара бумерангов, бэт-наручники, бэт-маячок, бэт-дротик, прибор ночного видения, термитные гранаты, дымовая завеса, фонарик, обруч из криптонита, паяльная лампа или Apple iPhone. Если Бэтмену надо кого-то усыпить, ослепить, оглушить, выследить, пристукнуть, притормозить, довести до слез или заэсэмэсить насмерть, то он тянется к своему Поясу. Для Бэтмена он так много значит, что он скорей забыл бы одеть штаны, чем пояс. А у Бэтмена штанов и нет, вместо них ему приходится носить резиновые бронеритузы (рис. 1).</p>

<p>Вместо Пояса Бэтмена у Nginx имеется свой <strong>набор модулей</strong>. Когда нужно сжать запрос или передать его по частям, Nginx запускает соответсвующий модуль. Когда Nginx блокирует доступ с какого-либо IP адреса или проверяет данные HTTP-авторизации, на самом деле это делает один из модулей. Если Nginx подсоединяется к Memcache или FastCGI-серверу, то именно модуль связывает их.</p>

<p>Пояс Бэтмена напичкан всякими полезностями, но иногда Бэтмену нужно что-то новенькое. Например, обнаружился новый противник, которого не удержать бэт-наручниками и не сломить бумерангом. Или Бэтмену скоро пригодится новая способность, такая как способность дышать под водой. Вот тогда-то Бэтмен и зовет <strong>Люциуса Фокса</strong>, чтобы тот придумал новую бэт-штуковину.</p>

<div class="figure">
    <img src="/i/lucius.jpg" alt="Люциус Фокс"/> 
    <strong>Рисунок 2</strong>: Брюс Уэйн (он же Бэтмен) со своим инженером Люциусом Фоксом.
</div>


<p>Целью этого руководства является подробно рассказать вам о модулях Nginx, чтобы вы смогли стать как Люциус Фокс. После прочтения руководства вы сможете проектировать и реализовывать отличные модули, которые помогут Nginx делать то, чего он раньше не умел. Система модулей Nginx содержит много ньюансов и особенностей, так что вы, наверно, будете часто возвращаться к этому руководству. Я постарался изложить концепции настолько доступно, насколько это возможно. но, без сомнений, создание модулей для Nginx все равно остается трудной задачей.</p>

<p>Но кто говорил, что создавать бэт-штуковины будет легко?</p>
</section>


<section>
<h>Содержание</h>
<ol start="0">
    <li><a href="#prerequisites">Предварительные требования</a></li>
    <li><a href="#overview">Устройство Модулей в первом приближении</a></li>
    <li><a href="#components">Компоненты Модуля Nginx</a>
    <ol>
        <li><a href="#configuration-structs">Структуры конфигурации Модуля</a></li>
        <li><a href="#directives">Директивы Модуля</a></li>
        <li><a href="#context">Контекст Модуля</a>
        <ol>
            <li><a href="#create_loc_conf">create_loc_conf</a></li>
            <li><a href="#merge_loc_conf">merge_loc_conf</a></li>
        </ol></li>
        <li><a href="#definition">Описание Модуля</a></li>
        <li><a href="#installation">Установка Модуля</a></li>
    </ol></li>
    <li><a href="#handlers">Обработчики</a>
    <ol>
        <li><a href="#non-proxying">Устройство Обработчиков (не проксирующих)</a>
        <ol>
            <li><a href="#non-proxying-config">Получение конфигурации локейшна</a></li>
            <li><a href="#non-proxying-response">Создание ответа</a></li>
            <li><a href="#non-proxying-header">Отправка заголовка</a></li>
            <li><a href="#non-proxying-body">Отправка тела</a></li>
        </ol></li>
        <li><a href="#proxying">Устроство Обработчиков Upstream (a.k.a. Прокси)</a>
        <ol>
            <li><a href="#proxying-summary">Основные свойства upstream callback</a></li>
            <li><a href="#create_request">The create_request callback</a></li>
            <li><a href="#process_header">The process_header callback</a></li>
            <li><a href="#keeping-state">Keeping state</a></li>
        </ol></li>
        <li><a href="#handler-installation">Handler Installation</a></li>
    </ol></li>
    <li><a href="#filters">Filters</a>
    <ol>
        <li><a href="#filters-header">Anatomy of a Header Filter</a></li>
        <li><a href="#filters-body">Anatomy of a Body Filter</a></li>
        <li><a href="#filters-installation">Filter Installation</a></li>
    </ol></li>
    <li><a href="#load_balancers">Load-Balancers</a>
    <ol>
        <li><a href="#lb-directive">The enabling directive</a></li>
        <li><a href="#lb-registration">The registration function</a></li>
        <li><a href="#lb-upstream">The upstream initialization function</a></li>
        <li><a href="#lb-peer">The peer initialization function</a></li>
        <li><a href="#lb-function">The load-balancing function</a></li>
        <li><a href="#lb-release">The peer release function</a></li>
    </ol></li>
    <li><a href="#compiling">Writing and Compiling a New Nginx Module</a></li>
    <li><a href="#advanced">Advanced Topics</a></li>
    <li><a href="#code">Code References</a></li>
</ol>

<section>
<a name="prerequisites"></a>
<h2>0. Предварительные требования</h2>
<p>Вы должны неплохо знать Си. Не просто его синтаксис, а то, как работать со структурами и не бояться указателей и ссылок на функции. А также иметь представление о препроцессоре и макросах. Если вам надо немного освежить знания, то ничто не сможет сравниться с <a href="https://ru.wikipedia.org/wiki/Язык_программирования_Си_(книга)">K&amp;R</a>.</p>

<p>Полезно понимать основы HTTP. Мы же, вообще-то, собираемся работать с web-сервером.</p>

<p>Пригодятся знания структуры конфигурационного файла Nginx'а. Вот основные моменты: существуют четыре <em>контекста</em> (называеются они <em>main</em> — главный, <em>server</em> — сервер, <em>upstream</em> — апстрим, и <em>location</em> — локейшн) в которых могут быть директивы с одним и более параметрами. Директивы в главном контексте применяются ко всему-всему; директивы из котекста сервера применяются к конкретному хосту/порту; директивы в апстриме описывают набор бэкендов; а директивы в контексте локейшна применяются к разным путям запроса (например, "/", "/images" и т.д.) Локешн наследует конфигурацию содержащему его серверному контексту, а сервер наследует главному контексту. Контекст апстрима не наследует никому, у него собственные директивы, которых больше нигде не используются. Я буду иногда упоминать эти четыре контекста, так что… не забывайте про них.</p>

<p>Ну что же, начнем!</p>

<section>
<a name="overview"></a>
<h2>1. Устройство Модулей в первом приближении</h2>

<p>У модулей Nginx'a могут быть три роли, которые мы рассмотрим:</p>
<ul>
    <li><em>обработчики</em> обрабатывают запрос и генерируют данные ответа</li>
    <li><em>фильтры</em> обрабатывают данные, полученные от обработчика</li>
    <li><em>балансировщики</em> выбирают бэкенд, которому передать запрос, если определено несколько бэкендов</li>
</ul>

<p>Модули делают реальную работу, которую обычно делают web-серверы: когда Nginx отправляет файл или проксирует запрос к другому серверу, то это делает модуль-обработчик. Когда Nginx гзипит данные или обрабатывает SSI-директивы, он делает это с помощью модуля-фильтра. Ядро Nginx'а берет на себя работу с сетью и реализацию протоколов, а также запускает модули, которые необходимы для обработки запроса. Децентрализованная архитектура позволяет нам создавать отдельные компоненты, которые делают что-то, что нам нужно.</p>

<p>Замечание: в отличие от модулей Apache, модули Nginx'а <em>не</em> подгружаются динамически (другими словами, модули вкомпилированы прямо в бинарник Nginx'а).</p>

<p>Как же тогда модули задействуются? Обычно, на стадии загрузки сервера каждый обработчик получает шанс прикрепиться к каким-либо локейшнам из конфигурационного файла. Если несколько обработчиков попробуют занять один локейшн, то победит только один (в хорошем конфиге такого не произойдет). Обработчик может завершиться с тремя результатами: все хорошо, произошла ошибка, или он может отказаться от обработки локейшна в пользу обработка по умолчанию (обычно, это выдача статических файлов).</p>

<p>Если обработчик является реверс-прокси для несколькоих бэкэнд серверов, то ему понадобится помощь модуля другого типа: балансировщика нагрузки. Балансировщик получает запрос вместе с набором бэкенд серверов и принимает решение, какому серверу передать запрос. Nginx поставлется с двумя модулями балансировки: round-robin, который выбирает серверы по очереди (как при раздаче карт в покере), и модуль с методом хеширования IP адреса, который гарантирует, что запрос конкретного клиента каждый раз будет передаваться одному и тому же бэкенду.</p>

<p>Если обработчик не вернул ошибку, управление перейдет к фильтрам. Один локейшн могут обрабатывать несколько модулей-фильтров, так, например, ответ может быть сжат, а потом выдаваться chunk'ами. Порядок запуска фильтров определяется на этапе компиляции. Фильры используют классический паттерн «<a href="http://ru.wikipedia.org/wiki/Chain_of_Responsibility">цепочка обязанностей</a>»: запускается один фильтр, делает свою работу, потом запускается второй, и так далее, пока не выполнится последний фильтр, и Nginx завершит обработку запроса.</p>

<p>Самая вкусная особенность цепочки фильтров заключается в том, что один фильтр не должен ждать, пока другой завершит свою работу целиком. Можно начать обрабатывать результат работы предыдущего фильтра по мере поступления, по типу потоков (пайпов) в Юниксе. Фильры оперируют <em>буферами</em>, размер которых, обычно, равен размеру страницы (4 Кб), но размер всегда можно задать в nginx.conf. Это означает, например, то, что что модуль может начать сжимать ответ и отправлять его клиенту еще до того, как модуль получит весь ответ от бэкенда целиком. Чудесно!</p>

<p>Чтобы увидеть картину в целом, рассмотрим типичный цикл обработки запроса:</p>
<ol>
    <li>Клиент посылает HTTP-запрос;</li>
    <li>Nginx выбирает подходящий обработчик на основе конфигурации локейшна;</li>
    <li>Балансировщик (если необходимо) выбирет бэкенд сервер;</li>
    <li>Обработчик делает свое дело и передает каждый буфер с данными результата первому фильтру;</li>
    <li>Первый фильтр передает результаты второму фильтру;</li>
    <li>Второй — третьему, третий — четвертому, и так далее;</li>
    <li>Получившийся ответ отправляется клиенту.</li>
</ol>

<p>Я сказал «типичный» цикл потому, что обработку в Nginx'е можно настраивать <em>как угодно</em>. Определить когда и как должен запускать модуль может оказаться непростой задачей для разработчика (я бы сказал очень даже не простой задачей). Настройка модуля проходит в следствии вызова ряда колбеков, и их не так уж мало. Конкретно, можно определить функцию, которая будет запущена:</p>
<ul>
    <li>Прямо перед чтением конфигурационного файла</li>
    <li>Для каждой директивы конфигурации локейшна или сервера в которой она появляется</li>
    <li>Когда Nginx инициализирует главную конфигурацию</li>
    <li>Когда Nginx инициализирует конфигурацию сервера (хост/порт)</li>
    <li>Когда Nginx мерджит конфигурацию сервера с главной конфигурацией</li>
    <li>Когда Nginx инициализирует конфигурацию локейшна</li>
    <li>Когда Nginx мерджит конфигурацией сервера с вложенной конфигурацией локейшна</li>
    <li>Когда запускается главный процесс Nginx'а</li>
    <li>Когда запускается новый рабочий процесс</li>
    <li>Когда рабочий процесс завершается</li>
    <li>Когда главный процесс завершается</li>
    <li>При обработке запроса</li>
    <li>При Фильтрации заголовка ответа</li>
    <li>При Фильтрации тела ответа</li>
    <li>При выборе бэкенд сервера</li>
    <li>В момент инициализации запроса к бэкенд серверу</li>
    <li>В момент <em>пере</em>инициализации запроса к бэкенд серверу</li>
    <li>В момент обработки ответа от бэкенд сервера</li>
    <li>В момент завершения работы с бэкенд сервером</li>
</ul>

<p>Подумать только! Это может смутить. В вашем распоряжении большая мощь, но можно начать делать что-то полезное, используя всего несколько хуков и соответсвующих функций. Время погрузиться в модули Nginx'а.</p>

<section>
<a name="components"></a>
<h2>2. Компоненты Модуля Nginx</h2>

<p>Как я уже сказал, в вашем распоряжении <em>огромный</em> запас гибкости для разработки модуля Nginx'а. В этом разделе описываются те части, которые есть практически в любом модуле. Это моможет нам лучше понять устройство модуля. А так же вы сможете оценить, когда можно будет перейти к написанию собственного модуля.</p>

<a name="configuration-structs"></a>
<h3>2.1. Структуры конфигурации Модуля</h3>

<p>Модули могут определить до трех конфигурационных структур, по одной для главного контекста, контекста сервера и контекста локейшна. Большинству модулей достаточно конфигурации локейшна. Для этих структур принято такое соглашение о наименовании: <code>ngx_http_&lt;название_модуля&gt;_(main|srv|loc)_conf_t</code>. Вот пример, взятый из модуля dav:</p>
<code>
<pre><code class="cpp">
typedef struct {
    ngx_uint_t  methods;
    ngx_flag_t  create_full_put_path;
    ngx_uint_t  access;
} ngx_http_dav_loc_conf_t;
</code></pre>
</code>

<p>Заметьте, что в Nginx'е используются специальные типы данных (<code>ngx_uint_t</code> и <code>ngx_flag_t</code>). Это просто алиасы для простых типов данных, которые мы все знаем и любим (см. <a class="source" href="http://lxr.evanmiller.org/http/source/core/ngx_config.h#L79">core/ngx_config.h</a>, если есть сомнения).</p>

<p>Элементы конфигурационной структуры заполняются директивами модуля.</p>

<a name="directives"></a>
<h3>2.2. Директивы Модуля</h3>

<p>Директивы модуля описыватся в статическом массиве элементов типа <code>ngx_command_t</code>. Вот пример того, как их определять взятый из маленького модуля, который я <em>(прим. перев. тут и далее речь об Эване Миллере)</em> написал:</p>

<code>
<pre><code class="cpp">
static ngx_command_t  ngx_http_circle_gif_commands[] = {
    { ngx_string("circle_gif"),
      NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS,
      ngx_http_circle_gif,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },

    { ngx_string("circle_gif_min_radius"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_num_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_circle_gif_loc_conf_t, min_radius),
      NULL },
      ...
      ngx_null_command
};
</code></pre>
</code>


<p>А вот определение структуры <code>ngx_command_t</code> (той, что мы сейчас заполняем), оно взято из файла <a class="source" href="http://lxr.evanmiller.org/http/source/core/ngx_conf_file.h#L77">core/ngx_conf_file.h</a>:</p>

<code>
<pre><code class="cpp">
struct ngx_command_t {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};
</code></pre>
</code>


<p>На первый взгляд кажется что это больше, чем нужно, но у каждого поля свое назначение.</p>

<p>В поле <code>name</code> хранится имя директивы, задается без пробелов. Используется тип <code>ngx_str_t</code>, значение которого чаще всего создаются с помощью макроса <code>ngx_str("proxy_pass")</code>. Замечание: структура <code>ngx_str_t</code> состоит из поля <code>data</code>, которое содержит саму строку, и поля <code>len</code>, в котором хранится длина строки. В большинстве случаев Nginx использует эту структуру взамен обычных строк.</p>

<p>Значение поля <code>type</code> задается с помощью набора флагов, которые определяют, где можно использовать эту директиву, и сколько она принимает параметров. Значение получается с помощью бинарного «ИЛИ»:</p>

<ul>
    <li><code>NGX_HTTP_MAIN_CONF</code>: разрешает использовать директиву в главном контексте</li>
    <li><code>NGX_HTTP_SRV_CONF</code>: в контексте сервера (хоста)</li>
    <li><code>NGX_HTTP_LOC_CONF</code>: в контексте локешна</li>
    <li><code>NGX_HTTP_UPS_CONF</code>: в контексте апстрима</li>
</ul>

<ul>
    <li><code>NGX_CONF_NOARGS</code>: сообщает, что директива не принимает аргументы</li>
    <li><code>NGX_CONF_TAKE1</code>: принимает ровно 1 аргументы</li>
    <li><code>NGX_CONF_TAKE2</code>: принимает ровно 2 аргумента</li>
    <li>…</li>
    <li><code>NGX_CONF_TAKE7</code>: принимает ровно 7 аргументов</li>
</ul>

<ul>
    <li><code>NGX_CONF_FLAG</code>: директива принимает булево значение ("on" или "off")</li>
    <li><code>NGX_CONF_1MORE</code>: директиве принимает 1 или более аргументов</li>
    <li><code>NGX_CONF_2MORE</code>: директиве принимает 2 или более аргументов</li>
</ul>

<p>Существуют также еще несколько других опций, смотрите <a class="source" href="http://lxr.evanmiller.org/http/source/core/ngx_conf_file.h#L1">core/ngx_conf_file.h</a>.</p>

<p>В поле структуры <code>set</code> хранится указатель на функцию, вызываемую для настройки части конфигурации модуля; обычно, эта функция преводит данные из аргументов в удобный формат и сохраняет их в соответстующей структуре конфигурации модуля. Функция принимает три аргумента:</p>

<ol>
    <li>указатель на структуру <code>ngx_conf_t</code>, которая содержит переданные директиве аргументы</li>
    <li>указатель на текущую структуру <code>ngx_command_t</code></li>
    <li>указатель на собственную структуру конфигурации модуля</li>
</ol>

<p>Эта функция настройки будет вызвана тогда, когда соответствующая директива будет встречена в конфигурационном файле. Nginx предосталяет набор функций для перевода разных типов параметров в структуру для последующей обработки. Среди этих функций хочу виделить следующие:</p>

<ul>
    <li><code>ngx_conf_set_flag_slot</code>: переводит "on" или "off" в 1 или 0</li>
    <li><code>ngx_conf_set_str_slot</code>: певодит строку параметра в <code>ngx_str_t</code></li>
    <li><code>ngx_conf_set_num_slot</code>: парсит число и возвращает его как <code>int</code></li>
    <li><code>ngx_conf_set_size_slot</code>: парсит размер ("8k", "1m" и т.д.) и возвращает его как <code>size_t</code></li>
</ul>

<p>Существуют и другие подобные функции, и они тоже очень удобны (посмотрите <a class="source" href="http://lxr.evanmiller.org/http/source/core/ngx_conf_file.h#L329">core/ngx_conf_file.h</a>). Модули также могут здесь ссылаться и на свои функции, если по каким-то причинам не достаточно встроенных.</p>

<p>Откуда эти встроенные функции знают куда сохранять данные? В этом им помогают следующие два поля структуры <code>ngx_command_t</code>: <code>conf</code> и <code>offset</code>. Поле <code>conf</code> указывае Nginx'у куда сохранить данные: в главную, серверную или конфигурационную структуру локешна (задается с помощью <code>NGX_HTTP_MAIN_CONF_OFFSET</code>, <code>NGX_HTTP_SRV_CONF_OFFSET</code>, или <code>NGX_HTTP_LOC_CONF_OFFSET</code>). Поле <code>offset</code> указывает в какую часть структуры записать значение.</p>

<p>И, <em>наконец</em>, поле <code>post</code> это указатель на еще одну штуку, которая может потребоваться модулю на этапе чтения конфигурации. Чаще всего равна <code>NULL</code>.</p>

<p>Набор директив должен заканчиваться полем <code>ngx_null_command</code>.</p>

<a name="context"></a>
<h3>2.3. Контекст Модуля</h3>

<p>Это статическая структура типа <code>ngx_http_module_t</code>, в которой определяются несколько указателей на функции для создания трех конфигураций и слияния их вместе. Называют ее <code>ngx_http_&lt;имя_модуля&gt;_module_ctx</code>. Вот назначение этих функций по порядку:</p>

<ul>
    <li>преконфигурация</li>
    <li>постконфигурация</li>
    <li>создание главной конфигурации (выделение памяти и задание значений по умолчанию)</li>
    <li>инициализация главной конфигурации (переопределение данных на взятые из nginx.conf)</li>
    <li>создание конфигурации сервера</li>
    <li>слияние ее с главной конфигурацией</li>
    <li>создание конфигурации локейшна</li>
    <li>слияние ее с конфигурацией сервера</li>
</ul>

<p>Функции принимают разные аргументы в зависимости от своего назначения. Вот описание этой структуры, взятое из <a class="source" href="http://lxr.evanmiller.org/http/source/http/ngx_http_config.h#L22">http/ngx_http_config.h</a>, в котором видны разные сигнатуры функций-колбеков:</p>

<code>
<pre><code class="cpp">
typedef struct {
    ngx_int_t   (*preconfiguration)(ngx_conf_t *cf);
    ngx_int_t   (*postconfiguration)(ngx_conf_t *cf);

    void       *(*create_main_conf)(ngx_conf_t *cf);
    char       *(*init_main_conf)(ngx_conf_t *cf, void *conf);

    void       *(*create_srv_conf)(ngx_conf_t *cf);
    char       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf);

    void       *(*create_loc_conf)(ngx_conf_t *cf);
    char       *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf);
} ngx_http_module_t;
</code></pre>
</code>

<p>Указатели на те функции, которые вам не пригодятся, можете заполнить <code>NULL</code>, Nginx поймет такую запись.</p>

<p>Большинство обработчиков используют только две последние: функцию для выделения памяти под структуру конфигурации локейшна (именуемая <code>ngx_http_&lt;имя_модуля&gt;_create_loc_conf</code>), и функцию устанавливающую значения по умолчанию и слияние этой конфигурации с любой унаследованной конфигурацией (именуемая <code>ngx_http_&lt;имя_модуля&gt;_merge_loc_conf</code>). Функция, объединяющая конфиги, также может вернуть ошибку невалидной конфигурации, что остановит загрузку сервера.</p>

<p>Вот пример структуры контекста модуля:</p>

<pre><code class="cpp">
static ngx_http_module_t  ngx_http_circle_gif_module_ctx = {
    NULL,                          /* преконфигурация */
    NULL,                          /* постконфигурация */

    NULL,                          /* создание главной конфигурации */
    NULL,                          /* инициализация главной конфигурации */

    NULL,                          /* создание конфигурации сервера */
    NULL,                          /* слияние конфигурации сервера */

    ngx_http_circle_gif_create_loc_conf,  /* создание конфигурации локейшна */
    ngx_http_circle_gif_merge_loc_conf /* слияние конфигурации локейшна */
};
</code></pre>

<p>Пришло время разобраться со всем этим подробнее. Эти конфигурационные колбеки очень похожи во всех модулях и используют одни и те же части Nginx API, так что их надо хорошо знать.</p>

<a name="create_loc_conf"></a>
<h4>2.3.1. create_loc_conf</h4>

<p>Вот так выглядит минимальная реализация функции <code>create_loc_conf</code>, взятая из моего модуля circle_gif (за подробностями в <a href="/nginx/ngx_http_circle_gif_module.c.txt">исходник</a>). Она получает управляющую структуру (<code>ngx_conf_t</code>) и возвращает вновь созданную структуру конфигурации модуля (в этом примере <code>ngx_http_circle_gif_loc_conf_t</code>).</p>

<pre><code class="cpp">
static void *
ngx_http_circle_gif_create_loc_conf(ngx_conf_t *cf)
{
    ngx_http_circle_gif_loc_conf_t  *conf;

    conf = ngx_pcalloc(cf-&gt;pool, sizeof(ngx_http_circle_gif_loc_conf_t));
    if (conf == NULL) {
        return NGX_CONF_ERROR;
    }
    conf-&gt;min_radius = NGX_CONF_UNSET_UINT;
    conf-&gt;max_radius = NGX_CONF_UNSET_UINT;
    return conf;
}
</code></pre>

<p>Прежде всего обратите внимание на особенность управления памятью в Nginx'е: он сам позаботится об освобождении памяти - <code>free</code> только тогда, когда для выделения памяти вы используете <code>ngx_palloc</code> (умную обертку для <code>malloc</code>) или <code>ngx_pcalloc</code> (умную обертку для <code>calloc</code>).</p>

<p>Возможными вариантами задания UNSET являются: <code>NGX_CONF_UNSET_UINT</code>, <code>NGX_CONF_UNSET_PTR</code>, <code>NGX_CONF_UNSET_SIZE</code>, <code>NGX_CONF_UNSET_MSEC</code>, и для всех случаев <code>NGX_CONF_UNSET</code>. UNSET показывает функции слияния конфигов, что это значение надо переопределить.</p>

<a name="merge_loc_conf"></a>
<h4>2.3.2. merge_loc_conf</h4>

<p>Вот так выглядит функция слияния конфигов в модуле circle_gif:</p>

<pre><code class="cpp">
static char *
ngx_http_circle_gif_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
{
    ngx_http_circle_gif_loc_conf_t *prev = parent;
    ngx_http_circle_gif_loc_conf_t *conf = child;

    ngx_conf_merge_uint_value(conf-&gt;min_radius, prev-&gt;min_radius, 10);
    ngx_conf_merge_uint_value(conf-&gt;max_radius, prev-&gt;max_radius, 20);

    if (conf-&gt;min_radius &lt; 1) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, 
            "min_radius должен быть больше или равен 1");
        return NGX_CONF_ERROR;
    }
    if (conf-&gt;max_radius &lt; conf-&gt;min_radius) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, 
            "max_radius должен быть больше или равен min_radius");
        return NGX_CONF_ERROR;
    }

    return NGX_CONF_OK;
}
</code></pre>

<p>Приятной особенностью Nginx'а является набор функций для слияния разных типов данных (<code>ngx_conf_merge_&lt;тип_данных&gt;_value</code>); их аргументами являются</p>
<ol>
    <li>значение <em>текущего</em> локешна</li>
    <li>значение, задаваемое, если #1 не установлено</li>
    <li>задаваемое по умолчанию, если не установлено ни #1, ни #2</li>
</ol>

<p>Результат будет записан в первый аргумент. Примерами таких функций являются <code>ngx_conf_merge_size_value</code>, <code>ngx_conf_merge_msec_value</code> и другие. Обратитесь к <a class="source" href="http://lxr.evanmiller.org/http/source/core/ngx_conf_file.h#L254">core/ngx_conf_file.h</a> за полным списком.</p>

<div class="aside">
<p>Вопрос: как же этим функциям удается записать данные в первый аргумент, если он передается по значению?</p>
<p>Ответ: это макросы (они разворачиваются в несколько конструкций <code>if</code>).</p>
</div>

<p>Интересно так же то, как эти функции обрабатывают ошибки. В случае неудачи в лог выводится сообщение, и функция возвращает <code>NGX_CONF_ERROR</code>. В этом случает запуск сервера прекращается. Так как сообщение выводится с уровнем <code>NGX_LOG_EMERG</code>, оно будет продублировано в стандартный поток ошибок. Кстати, <a class="source" href="http://lxr.evanmiller.org/http/source/core/ngx_log.h#L1">core/ngx_log.h</a> содержит полный список уровней вывода.)</p>

<a name="definition"></a>
<h3>2.4. Описание Модуля</h3>

<p>Теперь добавим еще один уровень абстракции, структуру <code>ngx_module_t</code>. Переменную назовем <code>ngx_http_&lt;имя_модуля&gt;_module</code>. В ней описываются указатели на контекст и директивы модуля вместе с остальными колбеками(завершение треда, завершение процесса и т.д.). Определение модуля иногда используется, чтобы найти какие-либо данные, связанные с этим модулем. Определение модуля обычно выглядит так:</p>

<pre><code class="cpp">
ngx_module_t  ngx_http_&lt;имя_модуля&gt;_module = {
    NGX_MODULE_V1,
    &amp;ngx_http_&lt;имя_модуля&gt;_module_ctx, /* контекст модуля */
    ngx_http_&lt;module name&gt;_commands,   /* директивы модуля */
    NGX_HTTP_MODULE,               /* тип модуля */
    NULL,                          /* инициализация мастера */
    NULL,                          /* инициализация модуля */
    NULL,                          /* инициализация процесса */
    NULL,                          /* инициализация треда */
    NULL,                          /* завершение треда */
    NULL,                          /* завершение процесса */
    NULL,                          /* завершение мастера */
    NGX_MODULE_V1_PADDING
};
</code></pre>

<p>…замените &lt;имя_модуля&gt; соответственно. Модули погут определять колбеки для моментов создания и уничтожения процессов и тредов, но большинство модулей стараются не усложнять себе жизнь. (Чтобы посмотреть список аргументов, обратитесь к <a class="source" href="http://lxr.evanmiller.org/http/source/core/ngx_conf_file.h#L110">core/ngx_conf_file.h</a>.)</p>

<a name="installation"></a>
<h3>2.5. Установка Модуля</h3>

<p>The proper way to install a module depends on whether the module is a handler, filter, or load-balancer; so the details are reserved for those respective sections.

<a name="handlers"></a>
<h2>3. Handlers</h2>

<p>Now we'll put some trivial modules under the microscope to see how they work.</p>

<a name="non-proxying"></a>
<h3>3.1. Anatomy of a Handler (Non-proxying)</h3>

<p>Handlers typically do four things: get the location configuration, generate an appropriate response, send the header, and send the body. A handler has one argument, the request struct. A request struct has a lot of useful information about the client request, such as the request method, URI, and headers. We'll go over these steps one by one.</p>

<a name="non-proxying-config"></a>
<h4>3.1.1. Getting the location configuration</h4>

<p>This part's easy. All you need to do is call <code>ngx_http_get_module_loc_conf</code> and pass in the current request struct and the module definition. Here's the relevant part of my circle gif handler:</p>

<code><pre>
static ngx_int_t
ngx_http_circle_gif_handler(ngx_http_request_t *r)
{
    ngx_http_circle_gif_loc_conf_t  *circle_gif_config;
    circle_gif_config = ngx_http_get_module_loc_conf(r, ngx_http_circle_gif_module);
    ...
</pre></code>

<p>Now I've got access to all the variables that I set up in my merge function.</p>

<a name="non-proxying-response"></a>
<h4>3.1.2. Generating a response</h4>

<p>This is the interesting part where modules actually do work.</p>

<p>The request struct will be helpful here, particularly these elements:</p>

<code><pre>
typedef struct {
...
/* the memory pool, used in the ngx_palloc functions */
    ngx_pool_t                       *pool; 
    ngx_str_t                         uri;
    ngx_str_t                         args;
    ngx_http_headers_in_t             headers_in;

...
} ngx_http_request_t;
</pre></code>

<p><code>uri</code> is the path of the request, e.g. "/query.cgi".</p>

<p><code>args</code> is the part of the request after the question mark (e.g. "name=john").</p>

<p><code>headers_in</code> has a lot of useful stuff, such as cookies and browser information, but many modules don't need anything from it. See <a href="http://lxr.evanmiller.org/http/source/http/ngx_http_request.h#L158" class="source">http/ngx_http_request.h</a> if you're interested.</p>

<p>This should be enough information to produce some useful output. The full <code>ngx_http_request_t</code> struct can be found in <a href="http://lxr.evanmiller.org/http/source/http/ngx_http_request.h#L316" class="source">http/ngx_http_request.h</a>.</p>

<a name="non-proxying-header"></a>
<h4>3.1.3. Sending the header</h4>

<p>The response headers live in a struct called <code>headers_out</code> referenced by the request struct. The handler sets the ones it wants and then calls <code>ngx_http_send_header(r)</code>. Some useful parts of <code>headers_out</code> include:</p>

<code><pre>
typedef stuct {
...
    ngx_uint_t                        status;
    size_t                            content_type_len;
    ngx_str_t                         content_type;
    ngx_table_elt_t                  *content_encoding;
    off_t                             content_length_n;
    time_t                            date_time;
    time_t                            last_modified_time;
..
} ngx_http_headers_out_t;
</pre></code>

<p>(The rest can be found in <a href="http://lxr.evanmiller.org/http/source/http/ngx_http_request.h#L220" class="source">http/ngx_http_request.h</a>.)</p>

<p>So for example, if a module were to set the Content-Type to "image/gif", Content-Length to 100, and return a 200 OK response, this code would do the trick:</p>

<code><pre>
    r-&gt;headers_out.status = NGX_HTTP_OK;
    r-&gt;headers_out.content_length_n = 100;
    r-&gt;headers_out.content_type.len = sizeof("image/gif") - 1;
    r-&gt;headers_out.content_type.data = (u_char *) "image/gif";
    ngx_http_send_header(r);
</pre></code>

<p>Most legal HTTP headers are available (somewhere) for your setting pleasure. However, some headers are a bit trickier to set than the ones you see above; for example, <code>content_encoding</code> has type <code>(ngx_table_elt_t*)</code>, so the module must allocate memory for it. This is done with a function called <code>ngx_list_push</code>, which takes in an <code>ngx_list_t</code> (similar to an array) and returns a reference to a newly created member of the list (of type <code>ngx_table_elt_t</code>). The following code sets the Content-Encoding to "deflate" and sends the header:</p>

<code><pre>
    r-&gt;headers_out.content_encoding = ngx_list_push(&amp;r-&gt;headers_out.headers);
    if (r-&gt;headers_out.content_encoding == NULL) {
        return NGX_ERROR;
    }
    r-&gt;headers_out.content_encoding-&gt;hash = 1;
    r-&gt;headers_out.content_encoding-&gt;key.len = sizeof("Content-Encoding") - 1;
    r-&gt;headers_out.content_encoding-&gt;key.data = (u_char *) "Content-Encoding";
    r-&gt;headers_out.content_encoding-&gt;value.len = sizeof("deflate") - 1;
    r-&gt;headers_out.content_encoding-&gt;value.data = (u_char *) "deflate";
    ngx_http_send_header(r);
</pre></code>

<p>This mechanism is usually used when a header can have multiple values simultaneously; it (theoretically) makes it easier for filter modules to add and delete certain values while preserving others, because they don't have to resort to string manipulation.</p>

<a name="non-proxying-body"></a>
<h4>3.1.4. Sending the body</h4>

<p>Now that the module has generated a response and put it in memory, it needs to assign the response to a special buffer, and then assign the buffer to a <em>chain link</em>, and <em>then</em> call the "send body" function on the chain link.</p>

<p>What are the chain links for? Nginx lets handler modules generate (and filter modules process) responses one buffer at a time; each chain link keeps a pointer to the next link in the chain, or <code>NULL</code> if it's the last one. We'll keep it simple and assume there is just one buffer.</p>

<p>First, a module will declare the buffer and the chain link:</p>

<code><pre>
    ngx_buf_t    *b;
    ngx_chain_t   out;
</pre></code>

<p>The next step is to allocate the buffer and point our response data to it:</p>

<code><pre>
    b = ngx_pcalloc(r-&gt;pool, sizeof(ngx_buf_t));
    if (b == NULL) {
        ngx_log_error(NGX_LOG_ERR, r-&gt;connection-&gt;log, 0, 
            "Failed to allocate response buffer.");
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    b-&gt;pos = some_bytes; /* first position in memory of the data */
    b-&gt;last = some_bytes + some_bytes_length; /* last position */

    b-&gt;memory = 1; /* content is in read-only memory */
    /* (i.e., filters should copy it rather than rewrite in place) */

    b-&gt;last_buf = 1; /* there will be no more buffers in the request */
</pre></code>

<p>Now the module attaches it to the chain link:</p>

<code><pre>
    out.buf = b;
    out.next = NULL;
</pre></code>

<p>FINALLY, we send the body, and return the status code of the output filter chain all in one go:</p>

<code><pre>
    return ngx_http_output_filter(r, &amp;out);
</pre></code>

<p>Buffer chains are a critical part of Nginx's IO model, so you should be comfortable with how they work.</p>

<div class="aside">
    <p>Trivia question: Why does the buffer have the <code>last_buf</code> variable, when we can tell we're at the end of a chain by checking "next" for <code>NULL</code>?</p>
    <p>Answer: A chain might be incomplete, i.e., have multiple buffers, but not all the buffers in this request or response. So some buffers are at the end of the chain but not the end of a request. This brings us to&hellip;</p>
</div>

<a name="proxying"></a>
<h3>3.2. Anatomy of an Upstream (a.k.a Proxy) Handler</h3>

<p>I waved my hands a bit about having your handler generate a response. Sometimes you'll be able to get that response just with a chunk of C code, but often you'll want to talk to another server (for example, if you're writing a module to implement another network protocol). You <em>could</em> do all of the network programming yourself, but what happens if you receive a partial response? You don't want to block the primary event loop with your own event loop while you're waiting for the rest of the response. You'd kill the Nginx's performance. Fortunately, Nginx lets you hook right into its own mechanisms for dealing with back-end servers (called "upstreams"), so your module can talk to another server without getting in the way of other requests. This section describes how a module talks to an upstream, such as Memcached, FastCGI, or another HTTP server.</p>

<a name="proxying-summary"></a>
<h4>3.2.1. Summary of upstream callbacks</h4>

<p>Unlike the handler function for other modules, the handler function of an upstream module does little "real work". It does <em>not</em> call <code>ngx_http_output_filter</code>. It merely sets callbacks that will be invoked when the upstream server is ready to be written to and read from. There are actually 6 available hooks:</p>

<p><code>create_request</code> crafts a request buffer (or chain of them) to be sent to the upstream</p>

<p><code>reinit_request</code> is called if the connection to the back-end is reset (just before <code>create_request</code> is called for the second time)</p>

<p><code>process_header</code> processes the first bit of the upstream's response, and usually saves a pointer to the upstream's "payload"</p>

<p><code>abort_request</code> is called if the client aborts the request</p>

<p><code>finalize_request</code> is called when Nginx is finished reading from the upstream</p>

<p><code>input_filter</code> is a body filter that can be called on the response body (e.g., to remove a trailer)</p>

<p>How do these get attached? An example is in order. Here's a simplified version of the proxy module's handler:</p>

<code><pre>
static ngx_int_t
ngx_http_proxy_handler(ngx_http_request_t *r)
{
    ngx_int_t                   rc;
    ngx_http_upstream_t        *u;
    ngx_http_proxy_loc_conf_t  *plcf;

    plcf = ngx_http_get_module_loc_conf(r, ngx_http_proxy_module);

/* set up our upstream struct */
    u = ngx_pcalloc(r-&gt;pool, sizeof(ngx_http_upstream_t));
    if (u == NULL) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    u-&gt;peer.log = r-&gt;connection-&gt;log;
    u-&gt;peer.log_error = NGX_ERROR_ERR;

    u-&gt;output.tag = (ngx_buf_tag_t) &amp;ngx_http_proxy_module;

    u-&gt;conf = &amp;plcf-&gt;upstream;

/* attach the callback functions */
    u-&gt;create_request = ngx_http_proxy_create_request;
    u-&gt;reinit_request = ngx_http_proxy_reinit_request;
    u-&gt;process_header = ngx_http_proxy_process_status_line;
    u-&gt;abort_request = ngx_http_proxy_abort_request;
    u-&gt;finalize_request = ngx_http_proxy_finalize_request;

    r-&gt;upstream = u;

    rc = ngx_http_read_client_request_body(r, ngx_http_upstream_init);

    if (rc &gt;= NGX_HTTP_SPECIAL_RESPONSE) {
        return rc;
    }

    return NGX_DONE;
}
</pre></code>

<p>It does a bit of housekeeping, but the important parts are the callbacks. Also notice the bit about <code>ngx_http_read_client_request_body</code>. That's setting another callback for when Nginx has finished reading from the client.</p>

<p>What will each of these callbacks do? Usually, <code>reinit_request</code>, <code>abort_request</code>, and <code>finalize_request</code> will set or reset some sort of internal state and are only a few lines long. The real workhorses are <code>create_request</code> and <code>process_header</code>.</p>

<a name="create_request"></a>
<h4>3.2.2. The create_request callback</h4>

<p>For the sake of simplicity, let's suppose I have an upstream server that reads in one character and prints out two characters. What would my functions look like?</p>

<p>The <code>create_request</code> needs to allocate a buffer for the single-character request, allocate a chain link for that buffer, and then point the upstream struct to that chain link. It would look like this:</p>

<code><pre>
static ngx_int_t
ngx_http_character_server_create_request(ngx_http_request_t *r)
{
/* make a buffer and chain */
    ngx_buf_t *b;
    ngx_chain_t *cl;

    b = ngx_create_temp_buf(r-&gt;pool, sizeof("a") - 1);
    if (b == NULL)
        return NGX_ERROR;

    cl = ngx_alloc_chain_link(r-&gt;pool);
    if (cl == NULL)
        return NGX_ERROR;

/* hook the buffer to the chain */
    cl-&gt;buf = b;
/* chain to the upstream */
    r-&gt;upstream-&gt;request_bufs = cl;

/* now write to the buffer */
    b-&gt;pos = "a";
    b-&gt;last = b-&gt;pos + sizeof("a") - 1;

    return NGX_OK;
}
</pre></code>

<p>That wasn't so bad, was it? Of course, in reality you'll probably want to use the request URI in some meaningful way. It's available as an <code>ngx_str_t</code> in <code>r-&gt;uri</code>, and the GET paramaters are in <code>r-&gt;args</code>, and don't forget you also have access to the request headers and cookies.</p>

<a name="process_header"></a>
<h4>3.2.3. The process_header callback</h4>

<p>Now it's time for the <code>process_header</code>. Just as <code>create_request</code> added a pointer to the request body, <code>process_header</code> <em>shifts the response pointer to the part that the client will receive</em>. It also reads in the header from the upstream and sets the client response headers accordingly.</p>

<p>Here's a bare-minimum example, reading in that two-character response. Let's suppose the first character is the "status" character. If it's a question mark, we want to return a 404 File Not Found to the client and disregard the other character. If it's a space, then we want to return the other character to the client along with a 200 OK response. All right, it's not the most useful protocol, but it's a good demonstration. How would we write this <code>process_header</code> function?</p>

<code><pre>
static ngx_int_t
ngx_http_character_server_process_header(ngx_http_request_t *r)
{
    ngx_http_upstream_t       *u;
    u = r-&gt;upstream;

    /* read the first character */
    switch(u-&gt;buffer.pos[0]) {
        case '?':
            r-&gt;header_only; /* suppress this buffer from the client */
            u-&gt;headers_in.status_n = 404;
            break;
        case ' ':
            u-&gt;buffer.pos++; /* move the buffer to point to the next character */
            u-&gt;headers_in.status_n = 200;
            break;
    }

    return NGX_OK;
}
</pre></code>

<p>That's it. Manipulate the header, change the pointer, it's done. Notice that <code>headers_in</code> is actually a response header struct like we've seen before (cf. <a href="http://lxr.evanmiller.org/http/source/http/ngx_http_request.h#L158" class="source">http/ngx_http_request.h</a>), but it can be populated with the headers from the upstream. A real proxying module will do a lot more header processing, not to mention error handling, but you get the main idea.</p>

<p>But&hellip; what if we don't have the whole header from the upstream in one buffer?</p>

<a name="keeping-state"></a>
<h4>3.2.4. Keeping state</h4>

<p>Well, remember how I said that <code>abort_request</code>, <code>reinit_request</code>, and <code>finalize_request</code> could be used for resetting internal state? That's because many upstream modules <em>have</em> internal state. The module will need to define a <em>custom context struct</em> to keep track of what it has read so far from an upstream. This is NOT the same as the "Module Context" referred to above. That's of a pre-defined type, whereas the custom context can have whatever elements and data you need (it's your struct). This context struct should be instantiated inside the <code>create_request</code> function, perhaps like this:</p>

<code><pre>
    ngx_http_character_server_ctx_t   *p;   /* my custom context struct */

    p = ngx_pcalloc(r-&gt;pool, sizeof(ngx_http_character_server_ctx_t));
    if (p == NULL) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    ngx_http_set_ctx(r, p, ngx_http_character_server_module);
</pre></code>

<p>That last line essentially registers the custom context struct with a particular request and module name for easy retrieval later. Whenever you need this context struct (probably in all the other callbacks), just do:</p>

<code><pre>
    ngx_http_proxy_ctx_t  *p;
    p = ngx_http_get_module_ctx(r, ngx_http_proxy_module);
</pre></code>

<p>And <code>p</code> will have the current state. Set it, reset it, increment, decrement, shove arbitrary data in there, whatever you want. This is a great way to use a persistent state machine when reading from an upstream that returns data in chunks, again without blocking the primary event loop. Nice!</p>

<a name="handler-installation"></a>
<h3>3.3. Handler Installation</h3>

<p>Handlers are installed by adding code to the callback of the directive that enables the module. For example, my circle gif <code>ngx_command_t</code> looks like this:</p>

<code><pre>
    { ngx_string("circle_gif"),
      NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS,
      ngx_http_circle_gif,
      0,
      0,
      NULL }
</pre></code>

<p>The callback is the third element, in this case <code>ngx_http_circle_gif</code>. Recall that the arguments to this callback are the directive struct (<code>ngx_conf_t</code>, which holds the user's arguments), the relevant <code>ngx_command_t</code> struct, and a pointer to the module's custom configuration struct. For my circle gif module, the function looks like:</p>

<code><pre>
static char *
ngx_http_circle_gif(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_core_loc_conf_t  *clcf;

    clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
    clcf-&gt;handler = ngx_http_circle_gif_handler;

    return NGX_CONF_OK;
}
</pre></code>

<p>There are two steps here: first, get the "core" struct for this location, then assign a handler to it. Pretty simple, eh?</p>

<p>I've said all I know about handler modules. It's time to move onto filter modules, the components in the output filter chain.</p>

<a name="filters"></a>
<h2>4. Filters</h2>

<p>Filters manipulate responses generated by handlers. Header filters manipulate the HTTP headers, and body filters manipulate the response content.</p>

<a name="filters-header"></a>
<h3>4.1. Anatomy of a Header Filter</h3>

<p>A header filter consists of three basic steps:</p>

<ol>
    <li>Decide whether to operate on this response</li>
    <li>Operate on the response</li>
    <li>Call the next filter</li>
</ol>

<p>To take an example, here's a simplified version of the "not modified" header filter, which sets the status to 304 Not Modified if the client's If-Modified-Since header matches the response's Last-Modified header. Note that header filters take in the <code>ngx_http_request_t</code> struct as the only argument, which gets us access to both the client headers and soon-to-be-sent response headers.</p>

<code><pre>
static
ngx_int_t ngx_http_not_modified_header_filter(ngx_http_request_t *r)
{
    time_t  if_modified_since;

    if_modified_since = ngx_http_parse_time(r-&gt;headers_in.if_modified_since-&gt;value.data,
                              r-&gt;headers_in.if_modified_since-&gt;value.len);

/* step 1: decide whether to operate */
    if (if_modified_since != NGX_ERROR &amp;&amp; 
        if_modified_since == r-&gt;headers_out.last_modified_time) {

/* step 2: operate on the header */
        r-&gt;headers_out.status = NGX_HTTP_NOT_MODIFIED;
        r-&gt;headers_out.content_type.len = 0;
        ngx_http_clear_content_length(r);
        ngx_http_clear_accept_ranges(r);
    }

/* step 3: call the next filter */
    return ngx_http_next_header_filter(r);
}
</pre></code>

<p>The <code>headers_out</code> structure is just the same as we saw in the section about handlers (cf. <a href="http://lxr.evanmiller.org/http/source/http/ngx_http_request.h#L220" class="source">http/ngx_http_request.h</a>), and can be manipulated to no end.</p>

<a name="filters-body"></a>
<h3>4.2. Anatomy of a Body Filter</h3>

<p>The buffer chain makes it a little tricky to write a body filter, because the body filter can only operate on one buffer (chain link) at a time. The module must decide whether to <em>overwrite</em> the input buffer, <em>replace</em> the buffer with a newly allocated buffer, or <em>insert</em> a new buffer before or after the buffer in question. To complicate things, sometimes a module will receive several buffers so that it has an <em>incomplete buffer chain</em> that it must operate on. Unfortunately, Nginx does not provide a high-level API for manipulating the buffer chain, so body filters can be difficult to understand (and to write). But, here are some operations you might see in action.</p>

<p>A body filter's prototype might look like this (example taken from the "chunked" filter in the Nginx source):</p>

<code><pre>
static ngx_int_t ngx_http_chunked_body_filter(ngx_http_request_t *r, ngx_chain_t *in);
</pre></code>

<p>The first argument is our old friend the request struct. The second argument is a pointer to the head of the current partial chain (which could contain 0, 1, or more buffers).</p>

<p>Let's take a simple example. Suppose we want to insert the text "&lt;l!-- Served by Nginx --&gt;" to the end of every request. First, we need to figure out if the response's final buffer is included in the buffer chain we were given. Like I said, there's not a fancy API, so we'll be rolling our own for loop:</p>

<code><pre>
    ngx_chain_t *chain_link;
    int chain_contains_last_buffer = 0;

    chain_link = in;
    for ( ; ; ) {
        if (chain_link-&gt;buf-&gt;last_buf)
            chain_contains_last_buffer = 1;
        if (chain_link-&gt;next == NULL)
            break;
        chain_link = chain_link-&gt;next;
    }
</pre></code>

<p>Now let's bail out if we don't have that last buffer:</p>

<code><pre>
    if (!chain_contains_last_buffer)
        return ngx_http_next_body_filter(r, in);
</pre></code>

<p>Super, now the last buffer is stored in chain_link. Now we allocate a new buffer:</p>

<code><pre>
    ngx_buf_t    *b;
    b = ngx_calloc_buf(r-&gt;pool);
    if (b == NULL) {
        return NGX_ERROR;
    }
</pre></code>

<p>And put some data in it:</p>

<code><pre>
    b-&gt;pos = (u_char *) "&lt;!-- Served by Nginx --&gt;";
    b-&gt;last = b-&gt;pos + sizeof("&lt;!-- Served by Nginx --&gt;") - 1;
</pre></code>

<p>And hook the buffer into a new chain link:</p>

<code><pre>
    ngx_chain_t   *added_link;

    added_link = ngx_alloc_chain_link(r-&gt;pool);
    if (added_link == NULL)
        return NGX_ERROR;

    added_link-&gt;buf = b;
    added_link-&gt;next = NULL;
</pre></code>

<p>Finally, hook the new chain link to the final chain link we found before:</p>

<code><pre>
    chain_link-&gt;next = added_link;
</pre></code>

<p>And reset the "last_buf" variables to reflect reality:</p>

<code><pre>
    chain_link-&gt;buf-&gt;last_buf = 0;
    added_link-&gt;buf-&gt;last_buf = 1;
</pre></code>

<p>And pass along the modified chain to the next output filter:</p>

<code><pre>
    return ngx_http_next_body_filter(r, in);
</pre></code>

<p>The resulting function takes much more effort than what you'd do with, say, mod_perl (<code>$response-&gt;body =~ s/$/&lt;!-- Served by mod_perl --&gt;/</code>), but the buffer chain is a very powerful construct, allowing programmers to process data incrementally so that the client gets something as soon as possible. However, in my opinion, the buffer chain desperately needs a cleaner interface so that programmers can't leave the chain in an inconsistent state. For now, manipulate it at your own risk.</p>

<a name="filters-installation"></a>
<h3>4.3. Filter Installation</h3>

<p>Filters are installed in the post-configuration step. We install both header filters and body filters in the same place.</p>

<p>Let's take a look at the chunked filter module for a simple example. Its module context looks like this:</p>

<code><pre>
static ngx_http_module_t  ngx_http_chunked_filter_module_ctx = {
    NULL,                                  /* preconfiguration */
    ngx_http_chunked_filter_init,          /* postconfiguration */
  ...
};
</pre></code>

<p>Here's what happens in <code>ngx_http_chunked_filter_init</code>:

<code><pre>
static ngx_int_t
ngx_http_chunked_filter_init(ngx_conf_t *cf)
{
    ngx_http_next_header_filter = ngx_http_top_header_filter;
    ngx_http_top_header_filter = ngx_http_chunked_header_filter;

    ngx_http_next_body_filter = ngx_http_top_body_filter;
    ngx_http_top_body_filter = ngx_http_chunked_body_filter;

    return NGX_OK;
}
</pre></code>

<p>What's going on here? Well, if you remember, filters are set up with a CHAIN OF RESPONSIBILITY. When a handler generates a response, it calls two functions: <code>ngx_http_output_filter</code>, which calls the global function reference <code>ngx_http_top_body_filter</code>; and <code>ngx_http_send_header</code>, which calls the global function reference <code>ngx_http_top_header_filter</code>.</p>

<p><code>ngx_http_top_body_filter</code> and <code>ngx_http_top_header_filter</code> are the respective "heads" of the body and header filter chains. Each "link" on the chain keeps a function reference to the next link in the chain (the references are called <code>ngx_http_next_body_filter</code> and <code>ngx_http_next_header_filter</code>). When a filter is finished executing, it just calls the next filter, until a specially defined "write" filter is called, which wraps up the HTTP response. What you see in this filter_init function is the module adding itself to the filter chains; it keeps a reference to the old "top" filters in its own "next" variables and declares <em>its</em> functions to be the new "top" filters. (Thus, the last filter to be installed is the first to be executed.)</p>

<div class="aside">
    <p>Side note: how does this work exactly?</p>
    <p>Each filter either returns an error code or uses this as the return statement:</p>

    <code>return ngx_http_next_body_filter();</code>

    <p>Thus, if the filter chain reaches the (specially-defined) end of the chain, an "OK" response is returned, but if there's an error along the way, the chain is cut short and Nginx serves up the appropriate error message. It's a singly-linked list with fast failures implemented solely with function references. Brilliant.</p>
</div>

<a name="load_balancers"></a>
<h2>5. Load-Balancers</h2>

<p>A load-balancer is just a way to decide which backend server will receive a particular request; implementations exist for distributing requests in round-robin fashion or hashing some information about the request. This section will describe both a load-balancer's installation and its invocation, using the upstream_hash module (<a class="source" href="/nginx/ngx_http_upstream_hash_module.c.txt">full source</a>) as an example. upstream_hash chooses a backend by hashing a variable specified in nginx.conf.</p>

<p>A load-balancing module has six pieces:</p>

<ol>
<li>The enabling configuration directive (e.g, <code>hash;</code>) will call a <em>registration function</em></li>
<li>The registration function will define the legal <code>server</code> options (e.g., <code>weight=</code>) and register an <em>upstream initialization function</em></li>
<li>The upstream initialization function is called just after the configuration is validated, and it:
<ul>
    <li>resolves the <code>server</code> names to particular IP addresses</li>
    <li>allocates space for sockets</li>
    <li>sets a callback to the <em>peer initialization function</em></li>
</ul>
<li>the peer initialization function, called once per request, sets up data structures that the <em>load-balancing function</em> will access and manipulate;</li>
<li>the load-balancing function decides where to route requests; it is called at least once per client request (more, if a backend request fails). This is where the interesting stuff happens.</li>
<li>and finally, the <em>peer release function</em> can update statistics after communication with a particular backend server has finished (whether successfully or not)</li>
</ol>

<p>It's a lot, but I'll break it down into pieces.</p>

<a name="lb-directive"></a>
<h3>5.1. The enabling directive</h3>

<p>Directive declarations, recall, specify both where they're valid and a function to call when they're encountered. A directive for a load-balancer should have the <code>NGX_HTTP_UPS_CONF</code> flag set, so that Nginx knows this directive is only valid inside an <code>upstream</code> block. It should provide a pointer to a <em>registration function</em>. Here's the directive declaration from the upstream_hash module:</p>

<code><pre>
    { ngx_string("hash"),
      NGX_HTTP_UPS_CONF|NGX_CONF_NOARGS,
      ngx_http_upstream_hash,
      0,
      0,
      NULL },
</pre></code>

<p>Nothing new there.</p>

<a name="lb-registration"></a>
<h3>5.2. The registration function</h3>

<p>The callback <code>ngx_http_upstream_hash</code> above is the registration function, so named (by me) because it registers an <em>upstream initialization function</em> with the surrounding <code>upstream</code> configuration. In addition, the registration function defines which options to the <code>server</code> directive are legal inside this particular <code>upstream</code> block (e.g., <code>weight=</code>, <code>fail_timeout=</code>). Here's the registration function of the upstream_hash module:</p>

<code><pre>
ngx_http_upstream_hash(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
 {
    ngx_http_upstream_srv_conf_t  *uscf;
    ngx_http_script_compile_t      sc;
    ngx_str_t                     *value;
    ngx_array_t                   *vars_lengths, *vars_values;

    value = cf-&gt;args-&gt;elts;

    /* the following is necessary to evaluate the argument to "hash" as a $variable */
    ngx_memzero(&amp;sc, sizeof(ngx_http_script_compile_t));

    vars_lengths = NULL;
    vars_values = NULL;

    sc.cf = cf;
    sc.source = &amp;value[1];
    sc.lengths = &amp;vars_lengths;
    sc.values = &amp;vars_values;
    sc.complete_lengths = 1;
    sc.complete_values = 1;

    if (ngx_http_script_compile(&amp;sc) != NGX_OK) {
        return NGX_CONF_ERROR;
    }
    /* end of $variable stuff */

    uscf = ngx_http_conf_get_module_srv_conf(cf, ngx_http_upstream_module);

    /* the upstream initialization function */
    uscf-&gt;peer.init_upstream = ngx_http_upstream_init_hash;

    uscf-&gt;flags = NGX_HTTP_UPSTREAM_CREATE;

    /* OK, more $variable stuff */
    uscf-&gt;values = vars_values-&gt;elts;
    uscf-&gt;lengths = vars_lengths-&gt;elts;

    /* set a default value for "hash_method" */
    if (uscf-&gt;hash_function == NULL) {
        uscf-&gt;hash_function = ngx_hash_key;
    }

    return NGX_CONF_OK;
 }
</pre></code>
 
<p>Aside from jumping through hoops so we can evaluation <code>$variable</code> later, it's pretty straightforward; assign a callback, set some flags. What flags are available?</p>

<ul>
<li><code>NGX_HTTP_UPSTREAM_CREATE</code>: let there be <code>server</code> directives in this upstream block. I can't think of a situation where you wouldn't use this.</li>
<li><code>NGX_HTTP_UPSTREAM_WEIGHT</code>: let the <code>server</code> directives take a <code>weight=</code> option</li>
<li><code>NGX_HTTP_UPSTREAM_MAX_FAILS</code>: allow the <code>max_fails=</code> option</li>
<li><code>NGX_HTTP_UPSTREAM_FAIL_TIMEOUT</code>: allow the <code>fail_timeout=</code> option</li>
<li><code>NGX_HTTP_UPSTREAM_DOWN</code>: allow the <code>down</code> option</li>
<li><code>NGX_HTTP_UPSTREAM_BACKUP</code>: allow the <code>backup</code> option</li>
</ul>

<p>Each module will have access to these configuration values. <em>It's up to the module to decide what to do with them.</em> That is, <code>max_fails</code> will not be automatically enforced; all the failure logic is up to the module author. More on that later. For now, we still haven't finished followed the trail of callbacks. Next up, we have the upstream initialization function (the <code>init_upstream</code> callback in the previous function).</p>

<a name="lb-upstream"></a>
<h3>5.3. The upstream initialization function</h3>

<p>The purpose of the upstream initialization function is to resolve the host names, allocate space for sockets, and assign (yet another) callback.  Here's how upstream_hash does it:</p>

<code><pre>
ngx_int_t
ngx_http_upstream_init_hash(ngx_conf_t *cf, ngx_http_upstream_srv_conf_t *us)
{
    ngx_uint_t                       i, j, n;
    ngx_http_upstream_server_t      *server;
    ngx_http_upstream_hash_peers_t  *peers;

    /* set the callback */
    us-&gt;peer.init = ngx_http_upstream_init_upstream_hash_peer;

    if (!us-&gt;servers) {
        return NGX_ERROR;
    }

    server = us-&gt;servers-&gt;elts;

    /* figure out how many IP addresses are in this upstream block. */
    /* remember a domain name can resolve to multiple IP addresses. */
    for (n = 0, i = 0; i &lt; us-&gt;servers-&gt;nelts; i++) {
        n += server[i].naddrs;
    }

    /* allocate space for sockets, etc */
    peers = ngx_pcalloc(cf-&gt;pool, sizeof(ngx_http_upstream_hash_peers_t)
            + sizeof(ngx_peer_addr_t) * (n - 1));

    if (peers == NULL) {
        return NGX_ERROR;
    }

    peers-&gt;number = n;

    /* one port/IP address per peer */
    for (n = 0, i = 0; i &lt; us-&gt;servers-&gt;nelts; i++) {
        for (j = 0; j &lt; server[i].naddrs; j++, n++) {
            peers-&gt;peer[n].sockaddr = server[i].addrs[j].sockaddr;
            peers-&gt;peer[n].socklen = server[i].addrs[j].socklen;
            peers-&gt;peer[n].name = server[i].addrs[j].name;
        }
    }

    /* save a pointer to our peers for later */
    us-&gt;peer.data = peers;

    return NGX_OK;
}
</pre></code>

<p>This function is a bit more involved than one might hope. Most of the work seems like it should be abstracted, but it's not, so that's what we live with. One strategy for simplifying things is to call the upstream initialization function of another module, have it do all the dirty work (peer allocation, etc), and then override the <code>us-&gt;peer.init</code> callback afterwards. For an example, see <a class="source" href="http://lxr.evanmiller.org/http/source/http/modules/ngx_http_upstream_ip_hash_module.c#L80">http/modules/ngx_http_upstream_ip_hash_module.c</a>.</p>

<p>The important bit from our point of view is setting a pointer to the <em>peer initialization function</em>, in this case <code>ngx_http_upstream_init_upstream_hash_peer</code>.

<a name="lb-peer"></a>
<h3>5.4. The peer initialization function</h3>

<p>The peer initialization function is called once per request. It sets up a data structure that the module will use as it tries to find an appropriate backend server to service that request; this structure is persistent across backend re-tries, so it's a convenient place to keep track of the number of connection failures, or a computed hash value. By convention, this struct is called <code>ngx_http_upstream_&lt;module name&gt;_peer_data_t</code>.

<p>In addition, the peer initalization function sets up two callbacks:</p>

<ul>
    <li><code>get</code>: the load-balancing function</li>
    <li><code>free</code>: the peer release function (usually just updates some statistics when a connection finishes)</li>
</ul>

<p>As if that weren't enough, it also initalizes a variable called <code>tries</code>. As long as <code>tries</code> is positive, nginx will keep retrying this load-balancer. When <code>tries</code> is zero, nginx will give up. It's up to the <code>get</code> and <code>free</code> functions to set <code>tries</code> appropriately.</p>

<p>Here's a peer initialization function from the upstream_hash module:</p>

<code><pre>
static ngx_int_t
ngx_http_upstream_init_hash_peer(ngx_http_request_t *r,
    ngx_http_upstream_srv_conf_t *us)
{
    ngx_http_upstream_hash_peer_data_t     *uhpd;
    
    ngx_str_t val;

    /* evaluate the argument to "hash" */
    if (ngx_http_script_run(r, &amp;val, us-&gt;lengths, 0, us-&gt;values) == NULL) {
        return NGX_ERROR;
    }

    /* data persistent through the request */
    uhpd = ngx_pcalloc(r-&gt;pool, sizeof(ngx_http_upstream_hash_peer_data_t)
	    + sizeof(uintptr_t) 
	      * ((ngx_http_upstream_hash_peers_t *)us-&gt;peer.data)-&gt;number 
                  / (8 * sizeof(uintptr_t)));
    if (uhpd == NULL) {
        return NGX_ERROR;
    }

    /* save our struct for later */
    r-&gt;upstream-&gt;peer.data = uhpd;

    uhpd-&gt;peers = us-&gt;peer.data;

    /* set the callbacks and initialize "tries" to "hash_again" + 1*/
    r-&gt;upstream-&gt;peer.free = ngx_http_upstream_free_hash_peer;
    r-&gt;upstream-&gt;peer.get = ngx_http_upstream_get_hash_peer;
    r-&gt;upstream-&gt;peer.tries = us-&gt;retries + 1;

    /* do the hash and save the result */
    uhpd-&gt;hash = us-&gt;hash_function(val.data, val.len);

    return NGX_OK;
}
</pre></code>

<p>That wasn't so bad. Now we're ready to pick an upstream server.</p>

<a name="lb-function"></a>
<h3>5.5. The load-balancing function</h3>

<p>It's time for the main course. The real meat and potatoes. This is where the module picks an upstream. The load-balancing function's prototype looks like:</p>

<code><pre>
static ngx_int_t 
ngx_http_upstream_get_&lt;module_name&gt;_peer(ngx_peer_connection_t *pc, void *data);
</pre></code>

<p><code>data</code> is our struct of useful information concerning this client connection. <code>pc</code> will have information about the server we're going to connect to. The job of the load-balancing function is to fill in values for <code>pc-&gt;sockaddr</code>, <code>pc-&gt;socklen</code>, and <code>pc-&gt;name</code>. If you know some network programming, then those variable names might be familiar; but they're actually not very important to the task at hand. We don't care what they stand for; we just want to know where to find appropriate values to fill them.</p>

<p>This function must find a list of available servers, choose one, and assign its values to <code>pc</code>. Let's look at how upstream_hash does it.</p>

<p>upstream_hash previously stashed the server list into the <code>ngx_http_upstream_hash_peer_data_t</code> struct back in the call to <code>ngx_http_upstream_init_hash</code> (above). This struct is now available as <code>data</code>:</p>

<code><pre>
    ngx_http_upstream_hash_peer_data_t *uhpd = data;
</pre></code>

<p>The list of peers is now stored in <code>uhpd-&gt;peers-&gt;peer</code>. Let's pick a peer from this array by dividing the computed hash value by the number of servers:</p>

<code><pre>
    ngx_peer_addr_t *peer = &amp;uhpd-&gt;peers-&gt;peer[uhpd-&gt;hash % uhpd-&gt;peers-&gt;number];
</pre></code> 

<p>Now for the grand finale:</p>

<code><pre>
    pc-&gt;sockaddr = peer-&gt;sockaddr;
    pc-&gt;socklen  = peer-&gt;socklen;
    pc-&gt;name     = &amp;peer-&gt;name;

    return NGX_OK;
</pre></code>

<p>That's all! If the load-balancer returns <code>NGX_OK</code>, it means, "go ahead and try this server". If it returns <code>NGX_BUSY</code>, it means all the backend hosts are unavailable, and Nginx should try again.</p>

<p>But&hellip; how do we keep track of what's unavailable? And what if we don't want it to try again?</p>

<a name="lb-release"></a>
<h3>5.6. The peer release function</h3>

<p>The peer release function operates after an upstream connection takes place; its purpose is to track failures. Here is its function prototype:</p>

<code><pre>
void 
ngx_http_upstream_free_&lt;module name&gt;_peer(ngx_peer_connection_t *pc, void *data, 
    ngx_uint_t state);
</pre></code>

<p>The first two parameters are just the same as we saw in the load-balancer function. The third parameter is a <code>state</code> variable, which indicates whether the connection was successful. It may contain two values bitwise OR'd together: <code>NGX_PEER_FAILED</code> (the connection failed) and <code>NGX_PEER_NEXT</code> (either the connection failed, or it succeeded but the application returned an error). Zero means the connection succeeded.</p>

<p>It's up to the module author to decide what to do about these failure events. If they are to be used at all, the results should be stored in <code>data</code>, a pointer to the custom per-request data struct.</p>

<p>But the crucial purpose of the peer release function is to set <code>pc-&gt;tries</code> to zero if you don't want Nginx to keep trying this load-balancer during this request. The simplest peer release function would look like this:</p>

<code><pre>
    pc-&gt;tries = 0;
</pre></code>

<p>That would ensure that if there's ever an error reaching a backend server, a 502 Bad Proxy error will be returned to the client.</p>

<p>Here's a more complicated example, taken from the upstream_hash module. If a backend connection fails, it marks it as failed in a bit-vector (called <code>tried</code>, an array of type <code>uintptr_t</code>), then keeps choosing a new backend until it finds one that has not failed.</p>

<code><pre>
#define ngx_bitvector_index(index) index / (8 * sizeof(uintptr_t))
#define ngx_bitvector_bit(index) (uintptr_t) 1 &lt;&lt; index % (8 * sizeof(uintptr_t))

static void
ngx_http_upstream_free_hash_peer(ngx_peer_connection_t *pc, void *data,
    ngx_uint_t state)
{
    ngx_http_upstream_hash_peer_data_t  *uhpd = data;
    ngx_uint_t                           current;

    if (state &amp; NGX_PEER_FAILED
            &amp;&amp; --pc-&gt;tries)
    {
        /* the backend that failed */
        current = uhpd-&gt;hash % uhpd-&gt;peers-&gt;number;

       /* mark it in the bit-vector */
        uhpd-&gt;tried[ngx_bitvector_index(current)] |= ngx_bitvector_bit(current);

        do { /* rehash until we're out of retries or we find one that hasn't been tried */
            uhpd-&gt;hash = ngx_hash_key((u_char *)&amp;uhpd-&gt;hash, sizeof(ngx_uint_t));
            current = uhpd-&gt;hash % uhpd-&gt;peers-&gt;number;
        } while ((uhpd-&gt;tried[ngx_bitvector_index(current)] &amp; ngx_bitvector_bit(current)) &amp;&amp; --pc-&gt;tries);
    }
}
</pre></code>

<p>This works because the load-balancing function will just look at the new value of <code>uhpd-&gt;hash</code>.</p>

<p>Many applications won't need retry or high-availability logic, but it's possible to provide it with just a few lines of code like you see here.</p>

<a name="compiling"></a>
<h2>6. Writing and Compiling a New Nginx Module</h2>

<p>So by now, you should be prepared to look at an Nginx module and try to understand what's going on (and you'll know where to look for help). Take a look in <a href="http://lxr.evanmiller.org/http/source/http/modules/" class="source">src/http/modules/</a> to see the available modules. Pick a module that's similar to what you're trying to accomplish and look through it. Stuff look familiar? It should. Refer between this guide and the module source to get an understanding about what's going on.</p>

<p>But Emiller didn't write a <em>Balls-In Guide to Reading Nginx Modules</em>. Hell no. This is a <em>Balls-Out Guide</em>. We're not reading. We're writing. Creating. Sharing with the world.</em>

<p>First thing, you're going to need a place to work on your module. Make a folder for your module anywhere on your hard drive, but separate from the Nginx source (and make sure you have the latest copy from <a href="http://nginx.net">nginx.net</a>). Your new folder should contain two files to start with:</p>
<ul>
    <li>"config"</li>
    <li>"ngx_http_&lt;your module&gt;_module.c"</li>
</ul>

<p>The "config" file will be included by <code>./configure</code>, and its contents will depend on the type of module.</p>

<p><strong>"config" for filter modules:</strong></p>

<code><pre>
ngx_addon_name=ngx_http_&lt;your module&gt;_module
HTTP_AUX_FILTER_MODULES="$HTTP_AUX_FILTER_MODULES ngx_http_&lt;your module&gt;_module"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_&lt;your module&gt;_module.c"
</pre></code>

<p><strong>"config" for other modules:</strong></p>

<code><pre>
ngx_addon_name=ngx_http_&lt;your module&gt;_module
HTTP_MODULES="$HTTP_MODULES ngx_http_&lt;your module&gt;_module"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_&lt;your module&gt;_module.c"
</pre></code>

<p>Now for your C file. I recommend copying an existing module that does something similar to what you want, but rename it "ngx_http_&lt;your module&gt;_module.c". Let this be your model as you change the behavior to suit your needs, and refer to this guide as you understand and refashion the different pieces.</p>

<p>When you're ready to compile, just go into the Nginx directory and type</p>

<code><pre>
./configure --add-module=path/to/your/new/module/directory
</pre></code>

<p>and then <code>make</code> and <code>make install</code> like you normally would. If all goes well, your module will be compiled right in. Nice, huh? No need to muck with the Nginx source, and adding your module to new versions of Nginx is a snap, just use that same <code>./configure</code> command. By the way, if your module needs any dynamically linked libraries, you can add this to your "config" file:</p>

<code><pre>
CORE_LIBS="$CORE_LIBS -lfoo"
</pre></code>

<p>Where <code>foo</code> is the library you need. If you make a cool or useful module, be sure to send a note to the <a href="http://wiki.codemongers.com/MailinglistSubscribe">Nginx mailing list</a> and share your work.</p>

<a name="advanced"></a>
<h2>7. Advanced Topics</h2>

<p>This guide covers the basics of Nginx module development. For tips on writing more sophisticated modules, be sure to check out <em><a href="nginx-modules-guide-advanced.html">Emiller's Advanced Topics In Nginx Module Development</a></em>.</p>

<a name="code"></a>
<h2>Appendix A: Code References</h2>

<p><a href="http://lxr.evanmiller.org/http/source/">Nginx source tree (cross-referenced)</a></p> 
<p><a href="http://lxr.evanmiller.org/http/source/http/modules/">Nginx module directory (cross-referenced)</a></p> 
<p><a href="/nginx/ngx_http_circle_gif_module.c.txt">Example addon: circle_gif</a></p> 
<p><a href="/nginx/ngx_http_upstream_hash_module.c.txt">Example addon: upstream_hash</a></p> 
<p><a href="http://github.com/gnosek/nginx-upstream-fair/tree/master">Example addon: upstream_fair</a></p>

<a name="changes"></a>
<h2>Appendix B: Changelog</h2>

<ul>
    <li>January 16, 2013: Corrected code sample in 5.5.</li>
    <li>December 20, 2011: Corrected code sample in 4.2 (one more time).</li>
    <li>March 14, 2011: Corrected code sample in 4.2 (again).</li>
    <li>November 11, 2009: Corrected code sample in 4.2.</li>
    <li>August 13, 2009: Reorganized, and moved <em><a href="nginx-modules-guide-advanced.html">Advanced Topics</a></em> to a separate article.</li>
    <li>July 23, 2009: Corrected code sample in 3.5.3.</li>
    <li>December 24, 2008: Corrected code sample in 3.4.</li>
    <li>July 14, 2008: Added information about subrequests; slight reorganization</li>
    <li>July 12, 2008: Added <a href="http://localdomain.pl/">Grzegorz Nosek</a>'s guide to shared memory</li>
    <li>July 2, 2008: Corrected "config" file for filter modules; rewrote introduction; added TODO section</li>
    <li>May 28, 2007: Changed the load-balancing example to the simpler upstream_hash module</li>
    <li>May 19, 2007: Corrected bug in body filter example</li>
    <li>May 4, 2007: Added information about load-balancers</li>
    <li>April 28, 2007: Initial draft</li>
</ul>

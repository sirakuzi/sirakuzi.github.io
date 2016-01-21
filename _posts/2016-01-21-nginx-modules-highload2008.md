---
title: "Валерий Холодков. Конференция HighLoad++ 2008. Разработка модулей для nginx"
description: "Еще одна попытка перевести это произведение."
categories: [posts]
layout: post
time: 14:54:55
---

<h3 align="center">Разработка модулей для nginx</h3>
<h3 align="center">Конференция HighLoad++ 2008</h3>
<h3 align="center">Валерий Холодков</h3>
<h2>Введение</h2>
<p>Материал, который Вы читаете в данный момент, основан на моих собственных
исследованиях исходного кода и материалах списка рассылки nginx. Поскольку
я не являюсь автором nginx и участвовал в обсуждении далеко не всех деталей
реализации, приведенная информация может быть не верна на 100%.</p>
<p>Вы предупреждены!</p>
<p>Материал организован следующим образом: в первой главе описываются общие принципы работы
асинхронных серверов и объясняется почему разработка модулей для nginx требует особого подхода,
во второй главе описывается API, который предоставляет nginx модулям, в третьей главе описываются
особенности взаимодействия nginx и модулей, в четвертой и последней главе описываются подходы к
реализации модулей.</p>
<h2>1. Асинхронные серверы</h2>
<p>Главной особенностью реализации nginx, является то, что все сетевые операции
ввода-вывода выполняются асинхронно относительно выполнения рабочих процессов. 
Это дает следующие преимущества:
<ul>
<li>Минимизирует количество рабочих процессов, и таким образом:</li>
<li>Минимизирует использование памяти, а так же:</li>
<li>Минимизирует синхронизацию между рабочими процессами, и кроме того:</li>
<li>Минимизирует количество переключений контекста</li>
</ul>
Однако обладает следующими недостатками:
<ul>
<li>Требует специального подхода к программированию, поэтому:</li>
<li>Невозможно использовать блокирующие операции ввода-вывода, и таким образом большинство библиотек и сред</li>
</ul>
Рабочие процессы nginx выполняют цикл в начале которого с помощью специального системного вызова
все сокеты опрашиваются на предмет появления событий. Событием может быть:
<ul>
<li>Запрос на установку нового соединения (для слушающих сокетов);</li>
<li>Факт появления данных во входном буфере сокета;</li>
<li>Факт появления свободного пространства в выходном буфере сокета.</li>
</ul>
В завершении цикла выполняется обработка событий. Для всех запросов на установку соединения
выполняется прием нового соединения. Для всех сокетов, имеющих данные во входном буфере
выполняется прием и обработка этих данных. Для всех сокетов, имеющих свободное пространство
в выходном буфере выполняется отправка новых данных. В каждом из этих случаев используются
неблокирующие операции.
</p>
<p>Когда обрабатывается один сокет одним из рабочих процессов, все остальные сокеты этого рабочего
процесса продолжают ожидать обработки. Если обработка одного сокета затягивается, то остальные
сокеты начинают испытывать "голод": приходящие данные скапливаются во входных буферах сокетов,
а готовые к записи сокеты не получают новых данных. На клиентской стороне подобная ситуация
выглядит как "зависание". Для предотвращения голодания сокетов сервер и компоненты сервера
должны быть реализованы с использованием следующих принципов:
<ul>
<li>Избегать длительных вычислительных процессов;</li>
<li>Минимизировать число синхронизаций с другими процессами;</li>
<li>Избегать блокирующих системных вызовов.</li>
</ul>
В UNIX дисковые операции ввода-вывода всегда блокирующие (за исключением POSIX realtime API). Казалось
бы, они должны противоречить последнему принципу. Однако, за счет кэширования и упреждающего чтения
блокирование удается свести к минимуму.</p>
<p>Из-за описанных выше ограничений полномасштабные веб-приложения сложно реализовать
исключительно в модулях nginx.</p>
<h2>2. API nginx</h2>
<h2>2.1. Управление памятью</h2>
<p>Управление памятью в nginx осуществляется с помощью пулов. Пул --
это последовательность предварительно выделенных блоков динамической памяти.
Пул привязан к объекту, который определяет время жизни всех объектов,
выделенных в пуле. Таким объектом может быть, например, запрос или цикл
обработки событий. Пулы используются исходя из следующих соображений:</p>
<ul>
<li>для минимизации накладных расходов на выделение мелких объектов;</li>
<li>для минимизации накладных расходов на удаление объектов;</li>
<li>минимизаровать вероятность утечек памяти;</li>
<li>для уменьшения числа системных вызовов sbrk.</li>
</ul>
<p>Для выделения памяти используются следующие функции:</p>
<a name="ngx_palloc"/>
<a name="ngx_pcalloc"/>
<pre>
void *ngx_palloc(ngx_pool_t *pool, size_t size);
void *ngx_pcalloc(ngx_pool_t *pool, size_t size);
</pre>
<p>
<b>pool</b> -- пул, из которого будет выделена память;<br>
<b>size</b> -- размер выделяемой памяти в байтах;<br>
<b>результат</b> -- указатель на выделенную память, либо NULL, если не
удалось выделить.<br></p>
<p>Функция <i>ngx_pcalloc</i> в дополнение заполняет выделенную память нулями.</p>
<p>Для маловероятного случая освобождения памяти используется функция <i>ngx_pfree</i>:</p>
<a name="ngx_pfree"/>
<pre>
ngx_int_t ngx_pfree(ngx_pool_t *pool, void *p);
</pre>
<b>pool</b> -- пул, в который будет возвращена память;<br>
<b>size</b> -- размер выделяемой структуры;<br>
<b>результат</b> -- NGX_OK, если удалось освободить, NGX_DECLINED, если
невозможно освободить.
</p>
<p>
Для регистрации деструктора (например для закрытия файловых дескрипторов или
удаления файлов) используется следующие структура и функции:</p>
<a name="ngx_pool_cleanup_add"/>
<a name="ngx_pool_cleanup_t"/>
<pre>
typedef struct ngx_pool_cleanup_s  ngx_pool_cleanup_t;

struct ngx_pool_cleanup_s {
    ngx_pool_cleanup_pt   handler;
    void                 *data;
    ngx_pool_cleanup_t   *next;
};

ngx_pool_cleanup_t *ngx_pool_cleanup_add(ngx_pool_t *p, size_t size);
</pre>
<p>
<b>p</b> -- пул, в котором будет зарегистрирован деструктор;<br>
<b>size</b> -- размер выделяемой структуры-контекста, которая будет передана деструктору;<br>
<b>результат</b> -- указатель на деструктор, если удалось выделить, NULL,
если не удалось выделить.<br></p>
<p>После выполнения <i>ngx_pool_cleanup_add</i> поле <i>data</i> указывает на контекст длиной <i>size</i>,
который переходит в распоряжение пользователя.</p>
<p>Поля в структуре <a href="#ngx_pool_cleanup_t">ngx_pool_cleanup_t</a> имеют следующее значение:<br>
<b>handler</b> -- хэндлер, который будет вызван при удалении пула;<br>
<b>data</b> -- указатель на структуру-контекст, которая будет передана деструктору;<br>
<b>next</b> -- указатель на следующий деструктор в пуле.<br>
</p>
<p>Пример использования:</p>
<pre>
static void ngx_sample_cleanup_handler(void *data);

static ngx_int_t ngx_http_sample_module_handler(ngx_http_request_t *r)
{
    ngx_pool_cleanup_t    *cln;
    ngx_sample_cleanup_t  *scln;

    cln = ngx_pool_cleanup_add(r-&gt;pool, sizeof(ngx_sample_cleanup_t));

    if(cln == NULL)
        return NGX_ERROR;

    cln-&gt;handler = ngx_sample_cleanup_handler;

    scln = cln-&gt;data;

    [... инициализация scln ...]
}

static void ngx_sample_cleanup_handler(void *data)
{
    ngx_sample_cleanup_t        *scln = data;

    [... использование scln ...]
}
</pre>
<h2>2.2. Векторы</h2>
<p>Векторы в nginx описываются следующей структурой:</p>
<pre>
struct ngx_array_s {
    void        *elts;
    ngx_uint_t   nelts;
    size_t       size;
    ngx_uint_t   nalloc;
    ngx_pool_t  *pool;
};

typedef struct ngx_array_s ngx_array_t;
</pre>
<p>
<b>pool</b> -- пул, в котором будет распределена память под элементы;<br>
<b>elts</b> -- указатель на элементы;<br>
<b>nelts</b> -- число элементов в векторе в данный момент;<br>
<b>size</b> -- размер элемента вектора в байтах;<br>
<b>nalloc</b> -- число элементов, для которых распределена память в данный момент.<br>
</p>
<p>Для создания вектора используется функция <i>ngx_array_create</i>:</p>
<a name="ngx_array_create"/>
<pre>
ngx_array_t *ngx_array_create(ngx_pool_t *p, ngx_uint_t n, size_t size);
</pre>
<p>
<b>p</b> -- пул, в котором будет распределена память под вектор и его элементы;<br>
<b>n</b> -- число элементов, под которые будет зарезервирована память;<br>
<b>size</b> -- размер элемента вектора в байтах;<br>
<b>результат</b> -- указатель на вектор, если удалось выделить, NULL,
если не удалось выделить.</p>
<p>Пример создания вектора:</p>
<pre>
typedef struct {
    [...]
} ngx_sample_struct_t;

{
    ngx_array_t *v;

    v = ngx_array_create(pool, 10, sizeof(ngx_sample_struct_t));

    if (v == NULL) {
        return NGX_ERROR;
    }
}
</pre>
<p>Если память под структуру <i>ngx_array_t</i> предварительно выделена,
то для её инициализации используется функция <i>ngx_array_init</i>:</p>
<a name="ngx_array_init"/>
<pre>
ngx_int_t ngx_array_init(ngx_array_t *array, ngx_pool_t *pool, ngx_uint_t n, size_t size);
</pre>
<p>
<b>array</b> -- указатель на структуру <i>ngx_array_t</i>;<br>
<b>p</b> -- пул, в котором будет распределена память под элементы вектора;<br>
<b>n</b> -- число элементов, под которые будет зарезервирована память;<br>
<b>size</b> -- размер элемента вектора в байтах;<br>
<b>результат</b> -- NGX_OK, если удалось выделить, NGX_ERROR, если не удалось выделить.<br>
</p>
<p>Пример инициализации вектора:</p>
<pre>
typedef struct {
    ngx_array_t v;
    [...]
} ngx_sample_struct_t;

{
    ngx_sample_struct_t t;

    if(ngx_array_init(&amp;t.v, pool, 10, sizeof(ngx_sample_struct_t)) != NGX_OK) {
        return NGX_ERROR;
    }
}
</pre>
<p>Для добавления элементов в конец вектора используются функции <i>ngx_array_push</i> и
<i>ngx_array_push_n</i>:</p>
<a name="ngx_array_push"/>
<a name="ngx_array_push_n"/>
<pre>
void *ngx_array_push(ngx_array_t *a);
void *ngx_array_push_n(ngx_array_t *a, ngx_uint_t n);
</pre>
<p>
<b>a</b> -- вектор, к которому добавляется элемент(ы);<br>
<b>n</b> -- число элементов, которые будут выделены;<br>
<b>результат</b> -- указатель на элемент, если удалось выделить, NULL,
если не удалось выделить.</p>
<p>Функция <i>ngx_array_push_n</i> добавляет n элементов к вектору.</p>
<p>При добавлении новых элементов к вектору необходимо учитывать, что отрезок памяти, выделенный
под элементы, расширяется за счет свободной памяти пула, расположенной непосредственно
после него. Если свободной памяти не обнаруживается, то выделяется новый непрерывный
отрезок памяти, и его длина каждый раз растет по двоично-экспонециальному принципу.</p>
<p>Пример:</p>
<pre>
typedef struct {
    [...]
} ngx_sample_struct_t;

{
    ngx_array_t *v;
    
    [...]

    h = ngx_array_push(v);
    if (h == NULL) {
        return NGX_ERROR;
    }
    
    [... использование h...]
}
</pre>
<p>Для удаления вектора используется функция <i>ngx_array_destroy</i>. Удаление
вектора выполняется только в том случае, если память, распределенная под его элементы,
располагалась в конце пула.</p>
<a name="ngx_array_destroy"/>
<pre>
void ngx_array_destroy(ngx_array_t *a);
</pre>
<p>
<b>a</b> -- вектор, который будет удален;<br>
</p>
<h2>2.3. Списки</h2>
<a name="2.3">
<p>
<i><a href="http://catap.ru/blog/2008/03/22/nginx-programming-guide-list/">См. оригинал главы</a></i>
</p>
<h3>Общие&nbsp;сведения</h3>
<p>В nginx (для 0.6.29 вполне актуально) есть связанные списки. Каждый список представляет собой&nbsp;структуру:
</p>
<pre>
<code>typedef struct {
    ngx_list_part_t  *last;
    ngx_list_part_t   part;
    size_t            size;
    ngx_uint_t        nalloc;
    ngx_pool_t       *pool;
} ngx_list_t;
</code>
</pre>
<p>Здесь:</p>
<ul>
  <li><b>last</b> — указатель на предыдущий элемент&nbsp;списка</li>
  <li><b>part</b> — указатель на текущий элемент&nbsp;списка</li>
  <li><b>size</b> — размер элементов списка. Т.е. если надо хранить строки, то размер будет равен&nbsp;sizeof(ngx_str_t)</li>
  <li><b>nalloc</b> — количество выделенных элементов памяти для одной&nbsp;части.</li>
  <li><b>pool</b> — указатель на пул&nbsp;памяти</li>
</ul>
<p>Список состоит из частей (часть — выделенный блок&nbsp;памяти):</p>
<pre>
<code>typedef struct ngx_list_part_s  ngx_list_part_t;

struct ngx_list_part_s {
    void             *elts;
    ngx_uint_t        nelts;
    ngx_list_part_t  *next;
};
</code>
</pre>
<p>Здесь:</p>
<ul>
  <li><b>elts</b> — указатель на память, где располагаются элементы&nbsp;списка </li>
  <li><b>nelts</b> — количество использованных элементов в&nbsp;части </li>
  <li><b>next</b> — указатель на следующую&nbsp;часть </li>
</ul>
<p>Т.е. список — это несколько частей (минимум 1), в которых располагаются элементы.
Каждая часть имеет свою длину и указатель на следующую часть или
<span class="caps">NULL</span>, если в ней еще есть место. Размер всех частей <em>равен</em>.
</p>
<h3>Работа со&nbsp;списком</h3>
<h4>Создание</h4>
<p>Для создания списка используется функция:</p>
<pre><code>ngx_list_t *ngx_list_create(ngx_pool_t *pool, ngx_uint_t n, size_t size);</code></pre>
<p>
<b>size</b> — размер одного элемента,
<b>n</b> — количество элементов в части,
<b>pool</b> — пул памяти, в котором выделяется память.</p>
<p>Функция создает новый список и инициализирует его
(создает первую часть и определяет размер части).
Возвращаемое значение — указатель на список или
<span class="caps">NULL</span> в случае&nbsp;ошибки.  
</p>
<h4>Инициализация</h4>
<p>Список можно не создавать, а просто инициализировать
(если у нас он определен на стеке), используя&nbsp;функцию:</p>
<pre><code>static ngx_inline ngx_int_t ngx_list_init(ngx_list_t *list, ngx_pool_t *pool, ngx_uint_t n, size_t size)</code></pre>
<p>Параметры ее аналогичны ngx_list_create, за исключением первого — это указатель на список,
который будет инициализироваться (или переинициализироваться).
Функция возвращает либо NGX_OK в случае успеха, либо NGX_ERROR в случае проблем.
Функция небезопасна, т.е. можно переинициализировать любой существующий список и все старые данные потеряются&nbsp;☺</p>
<h4>Добавление&nbsp;элемента</h4>
<p>Для добавления элемента в список используется&nbsp;функция:</p>
<pre><code>void *ngx_list_push(ngx_list_t *list);</code></pre>
<p>Она принимает указатель на список и возвращает указатель на место в части,
в которое можно записывать значения. Если часть закончилась, создается новая
и возвращается указатель на ее первый&nbsp;элемент.</p>
<h4>Навигация по элементам&nbsp;списка</h4>
<p>Для навигации по списку можно использовать такой&nbsp;код:</p>
<pre><code>part = &amp;list.part;
data = part-&gt;elts;

for (i = 0 ;;i++) {
  if (i &gt;= part-&gt;nelts) {
    if (part-&gt;next == NULL) {
      break;
    }

    part = part-&gt;next;
    data = part-&gt;elts;
    i = 0;
  }

  /* use element of list as data[i] */

}</code></pre>
<p>Вместо комментария вставляется ваш код, в котором будет происходить обращение к элементам&nbsp;списка.</p>
<h3>Пример&nbsp;использования списка</h3>
<p>Пример функции, в которой создается список из двух частей и осуществляется навигация по&nbsp;нему: </p>
<pre><code>void ngx_list_sample_using(ngx_http_request_t *r)
{
  ngx_list_t       list;
  ngx_uint_t       i;
  ngx_list_part_t *part;
  ngx_uint_t      *list_element;
  ngx_uint_t      *sum = 0;

  if (ngx_list_init(&amp;list, r-&gt;pool, 5, sizeof(ngx_uint_t)) == NGX_ERROR) {
    return;
  }

  for (i = 0; i &lt; 10; i++) {
    list_element = ngx_list_push(&amp;list);
    if (list_element == NULL) {
      return;
    }
    *list_element = i; 
  }

  part = &amp;list.part;
  data = part-&gt;elts;

  for (i = 0 ;;i++) {
    if (i &gt;= part-&gt;nelts) {
      if (part-&gt;next == NULL) {
        break;
      }

      part = part-&gt;next;
      data = part-&gt;elts;
      i = 0;
    }

  sum += data[i];

  }
/* here sum is 45 */
}

</code></pre>
<h2>2.4. Управление буферами и очередями</h2>
<h2>2.4.1 Управление буферами</h2>
<p>Буферы используются для отслеживания прогресса приема, отправления и
обработки данных. Заголовок буфера описывается структурой <i>ngx_buf_t</i>.
Данные буфера могут располагаться в изменяемой или неизменяемой
памяти или в файле. Для изменяемой памяти актуальны следующие поля:</p>
<a name="ngx_buf_t">
<pre>
typedef struct ngx_buf_s ngx_buf_t;

struct ngx_buf_s {
    [...]

    u_char          *pos; /* начало окна */
    u_char          *last; /* конец окна */
    u_char          *start; /* начало буфера */
    u_char          *end; /* конец буфера */

    unsigned         temporary:1; /* = 1 */

    [...]
};
</pre>
<p>Для неизменяемой памяти:</p>
<pre>
struct ngx_buf_s {
    [...]

    u_char          *pos; /* начало окна */
    u_char          *last; /* конец окна */
    u_char          *start; /* начало буфера */
    u_char          *end; /* конец буфера */

    unsigned         memory:1; /* = 1 */

    [...]
};
</pre>
<p>Для файла:</p>
<pre>
struct ngx_buf_s {
    [...]

    off_t            file_pos; /* начало окна */
    off_t            file_last /* конец окна */
    ngx_file_t      *file; /* указатель на файл */

    unsigned         in_file:1; /* = 1 */

    [...]
};
</pre>
<p></p>
<p>Окно определяет часть буфера, которую осталось отправить, осталось обработать, либо
заполнена полученными данными. 
В процессе заполнения указатель <i>last</i> перемещается в направлении <i>end</i>, в процессе
обработки или отправления указатель <i>pos</i> перемещается в направлении <i>last</i> (или <i>file_pos</i> в направлении
<i>file_last</i>).
В случае, если все данные буфера отправлены или обработаны,
то <i>pos</i> == <i>last</i> или <i>file_pos</i> == <i>file_last</i>. В случае, если буфер не заполнен, то 
<i>pos</i> == <i>last</i> == <i>start</i>.</p>
<p>Кроме того, буфер содержит флаги, описывающие как необходимо обрабатывать
данные, содержащиеся в буфере:
</p>
<pre>
struct ngx_buf_s {
    [...]

    unsigned         recycled:1; /* буфер повторно использован после освобождения */
    unsigned         flush:1; /* все буферизированные данные должны быть обработаны и переданы на следующий уровень после обработки этого буфера */
    unsigned         last_buf:1; /* указывает на то, что буфер является последним в потоке данных */
    unsigned         last_in_chain:1; /* указывает на то, что буфер является последним в данной цепи (очереди) */
    unsigned         temp_file:1; /* указывает на то, что буфер располагается во временном файле */

    [...]
};
</pre>
<p>В случае, если буфер константный, в оперативной памяти может находиться произвольное число структур <a href="#ngx_buf_t">ngx_buf_t</a>,
указывающих на идентичные данные, располагающиеся, например, в сегменте константных данных или в конфигурации модуля (см. <a href="#2.9">2.9</a>). Окно
определяет прогресс отправления этих данных.</p>
<p>Для выделения памяти под структуру <a href="#ngx_buf_t">ngx_buf_t</a> используются макросы:</p>
<a name="ngx_alloc_buf"/>
<a name="ngx_calloc_buf"/>
<pre>
ngx_alloc_buf(pool);
ngx_calloc_buf(pool);
</pre>
<p>
<b>pool</b> -- пул, из которого будет выделен буфер;<br>
<b>rvalue</b> -- указатель на структуру <a href="#ngx_buf_t">ngx_buf_t</a>, если удалось выделить,
NULL, если не удалось выделить. После выделения все поля структуры необходимо
инициализировать. Макрос <i>ngx_calloc_buf</i> преобразуется в функцию, которая в дополнение
зполняет выделенную память нулями.
</p>
<p>Для выделения памяти под временный буфер используется следующая функция:</p>
<a name="ngx_create_temp_buf"/>
<pre>
ngx_buf_t *ngx_create_temp_buf(ngx_pool_t *pool, size_t size);
</pre>
<p>
<b>pool</b> -- пул, из которого будет выделен буфер;<br>
<b>size</b> -- размер буфера (расстояние между start и end);<br>
<b>результат</b> -- указатель на структуру <a href="#ngx_buf_t">ngx_buf_t</a>, если удалось выделить,
NULL, если не удалось выделить. После выделения pos и last будут равны
start и флаг temporary будет установлен в 1.
</p>
<h2>2.4.2 Управление очередями</h2>
<p>Очереди (или цепи) связывают несколько буферов в последовательность,
которая определяет порядок приема, отправления или обработки данных.</p>
<pre>
struct ngx_chain_s {
    ngx_buf_t    *buf; /* буфер, связанный с текущим звеном */
    ngx_chain_t  *next; /* следующее звено */
};

typedef struct ngx_chain_s ngx_chain_t;
</pre>
<p>Для выделения памяти под цепи используются следующие функции:</p>
<a name="ngx_alloc_chain_link"/>
<a name="ngx_create_chain_of_bufs"/>
<a name="ngx_chain_get_free_buf"/>
<pre>
typedef struct {
    ngx_int_t    num;
    size_t       size;
} ngx_bufs_t;

ngx_chain_t *ngx_alloc_chain_link(ngx_pool_t *pool);
ngx_chain_t *ngx_create_chain_of_bufs(ngx_pool_t *pool, ngx_bufs_t *bufs);
ngx_chain_t *ngx_chain_get_free_buf(ngx_pool_t *p, ngx_chain_t **free);
</pre>
<p>функция <i>ngx_alloc_chain_link</i> выделяет память под одно звено из пула;<br>
<br>
функция <i>ngx_create_chain_of_bufs</i> выделяет память под цепь звеньев и буферы;<br>
<br>
функция <i>ngx_chain_get_free_buf</i> выделяет звено из цепи cвободных
буферов или из пула, если цепь пуста.<br>
<br>
<b>pool</b> -- пул, из которого будет(будут) выделен(ы) звенья/буферы при необходимости;<br>
<b>bufs</b> -- структура, описывающая размер и число буферов;<br>
<b>free</b> -- указатель на цепочку свободных буферов;<br>
<b>результат</b> -- указатель на структуру ngx_chain_t, если удалось выделить,
NULL, если не удалось выделить.</p>
<p>Для освобождения звеньев используется следующий макрос:</p>
<a name="ngx_free_chain"/>
<pre>
ngx_free_chain(pool, cl)
</pre>
<p>
<b>pool</b> -- пул, в который возвращается звено,
<b>cl</b> -- возвращаемое звено.<br>
</p>
<p>Управление очередями:</p>
<a name="ngx_chain_add_copy"/>
<a name="ngx_chain_update_chains"/>
<pre>
ngx_int_t ngx_chain_add_copy(ngx_pool_t *pool, ngx_chain_t **chain,
    ngx_chain_t *in);
void ngx_chain_update_chains(ngx_chain_t **free, ngx_chain_t **busy,
    ngx_chain_t **out, ngx_buf_tag_t tag);
</pre>
функция <i>ngx_chain_add_copy</i> добавляет буферы из цепи <i>in</i> к
цепи <i>chain</i>, выделяя новые звенья. Возвращает NGX_OK, если удалось успешно добавить, 
NGX_ERROR, если не удалось выделить память;<br><br>
функция <i>ngx_chain_update_chains</i> добавляет все обработанные или отправленные буферы с тегом
<i>tag</i> из цепи <i>out</i> и <i>busy</i> к цепи <i>free</i>, оставшиеся добавляет к цепи
<i>busy</i>.<br>
<br>
<b>pool</b> -- пул, из которого будут выделены звенья при необходимости,
</p>
<h2>2.5. Строки</h2>
<p>
В nginx строки хранятся в Pascal-like форме с целью избежать накладных
расходов при вычислении длины, а так же копирования в некоторых
ситуациях. 
<pre>
typedef struct {
    size_t      len;
    u_char     *data;
} ngx_str_t;
</pre>
<b>len</b> -- длина строки в байтах,
<b>data</b> -- указатель на память, содержащую строку.<br>
<br>
</p>
<h2>2.6. Переменные</h2>
<p>Переменные -- это именованные контейнеры данных, которые можно
преобразовывать в строки или из строк. Значения переменных могут хранится
в любой форме. Для преобразования из строк и в строки используются эксессоры --
функции установки и получения значения переменной:</p>
<pre>
typedef void (*ngx_http_set_variable_pt) (ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data);
typedef ngx_int_t (*ngx_http_get_variable_pt) (ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data);
</pre>
<i>ngx_http_set_variable_pt</i> -- тип функций, вызываемых для установки
значения переменной; <i>ngx_http_get_variable_pt</i> -- тип функций, вызываемых для
получения значения переменной.<br>
<br>
Аргументы функций:<br>
<b>r</b> -- запрос, в контексте которого устанавливает или запрашивается
значение переменной,<br>
<b>v</b> -- устанавливаемое или получаемое значение переменной,<br>
<b>data</b> -- контекст переменной.<br>
<br>
Сама переменная описывается следующей структурой:
<pre>
struct ngx_http_variable_s {
    ngx_str_t                     name;
    ngx_http_set_variable_pt      set_handler;
    ngx_http_get_variable_pt      get_handler;
    uintptr_t                     data;
    ngx_uint_t                    flags;

    [...]
};
</pre>
<b>name</b> -- имя переменной,<br>
<b>set_handler</b> -- функция установки значения,<br>
<b>get_handler</b> -- функция получения значения,<br>
<b>data</b> -- контекст переменной,<br>
<b>flags</b> -- флаги:<br>
<ul>
<li><i>NGX_HTTP_VAR_CHANGEABLE</i> -- переменная изменяема (например командой set модуля rewrite);</li>
<li><i>NGX_HTTP_VAR_NOCACHEABLE</i> -- значение не кэшируемо;</li>
<li>[...]</li>
</ul>
<h3>Добавление переменной</h3>
<p>Для добавления новой переменной используется следующая функция:</p>
<a name="ngx_http_add_variable"/>
<pre>
ngx_http_variable_t *ngx_http_add_variable(ngx_conf_t *cf, ngx_str_t *name,
    ngx_uint_t flags);
</pre>
<b>cf</b> -- конфигурация, в которой создается переменная,<br>
<b>name</b> -- имя переменной,<br>
<b>flags</b> -- флаги,<br>
<b>результат</b> -- указатель на структуру <i>ngx_http_variable_t</i>.<br>
<br>
Пример вызова:<br>
<pre>
static ngx_str_t  ngx_http_var_name = ngx_string("var");

[...]

{
    ngx_http_variable_t  *var;

    var = ngx_http_add_variable(cf, &amp;ngx_http_var_name, NGX_HTTP_VAR_NOCACHEABLE);
}
</pre>
<h3>Получения индекса переменной</h3>
<a name="ngx_http_get_variable_index">
<pre>
ngx_int_t ngx_http_get_variable_index(ngx_conf_t *cf, ngx_str_t *name);
</pre>
<b>cf</b> -- конфигурация, в которой определена переменная,<br>
<b>name</b> -- имя переменной,<br>
<b>результат</b> -- индекс переменной.<br>
<br>
<h3>Получения строкового значения переменной</h3>
<a name="ngx_http_get_indexed_variable">
<a name="ngx_variable_value_t">
<a name="ngx_http_variable_value_t">
<pre>
typedef struct {
    unsigned    len:28;

    unsigned    valid:1;
    unsigned    no_cacheable:1;
    unsigned    not_found:1;

    [...]

    u_char     *data;
} ngx_variable_value_t;

typedef ngx_variable_value_t  ngx_http_variable_value_t;

ngx_http_variable_value_t *ngx_http_get_indexed_variable(ngx_http_request_t *r,
    ngx_uint_t index);
</pre>
<b>r</b> -- запрос, в контексте которого запрашивается переменная,<br>
<b>index</b> -- индекс переменной,<br>
<b>результат</b> -- значение переменной в виде указателя на структуру
<i>ngx_http_variable_value_t</i> (синоним <i>ngx_variable_value_t</i>).<br>
Поля в структуре <i>ngx_variable_value_t</i> означают следующее:
<ul>
<li><b>len</b> -- длина строкового значения перменной;</li>
<li><b>data</b> -- указатель на память, содержащую строковое значение перменной;</li>
<li><b>valid</b> -- 1, если строковое значение переменной в кэше актуально;</li>
<li><b>no_cacheable</b> -- 1, если значение переменной не кэшируемо;</li>
<li><b>not_found</b> -- 1, если значение переменной не найдено;</li>
</ul>
</p>
<h2>2.7. Скрипты</h2>
<p>
Скрипты в nginx -- байт-код для генерации строк. Скрипт можно создать или
скомпилировать из шаблона, затем выполнить произвольное число раз. Шаблон --
это строка со ссылками на переменные, которые имеют вид $varible_name или
${variable_name}. Переменные могут быть символьными, либо позиционными,
которые имеют индексы от 0 до 9: $0, ... $9. Позиционные переменные заполняются
модулем rewrite. При выполнении скрипта используются значения переменных на этот
момент выполнения, либо на момент попадания значения переменной в кэш, если
значение кэшируемо. Для компиляции шаблона необходимо заполнить следующую структуру:
<pre>
typedef struct {
    ngx_conf_t                 *cf; /* указатель на конфигурацию ngx_conf_t */
    ngx_str_t                  *source /* компилируемый шаблон */;

    ngx_array_t               **lengths; /* код для определения длины результата */
    ngx_array_t               **values; /* код для генерации результата */
    
    ngx_uint_t                  variables; /* предполагаемое число переменных */

    unsigned                    complete_lengths:1; /* генерировать код для определения длины */
    unsigned                    complete_values:1; /* генерировать код для генерации значения */
} ngx_http_script_compile_t;
</pre>
<p>
Заполненную структуру необходимо передать в функцию <i>ngx_http_script_compile</i>. Пример:
<a name="ngx_http_script_compile">
<pre>
static ngx_str_t ngx_http_script_source = ngx_string("Your IP-address is $remote_addr");

{
    ngx_http_script_compile_t   sc;
    ngx_array_t                 *lengths = NULL;
    ngx_array_t                 *values = NULL;

    ngx_memzero(&amp;sc, sizeof(ngx_http_script_compile_t));

    sc.cf = cf;
    sc.source = &amp;ngx_http_script_source;
    sc.lengths = &amp;lengths;
    sc.values = &amp;values;
    sc.variables = 1;
    sc.complete_lengths = 1;
    sc.complete_values = 1;

    if (ngx_http_script_compile(&amp;sc) != NGX_OK) {
        return NGX_CONF_ERROR;
    }

    return NGX_CONF_OK;
}
</pre>
<p>Для выполнения скрипта используется функция <i>ngx_http_script_run</i>:</p>
<a name="ngx_http_script_run">
<pre>
u_char *ngx_http_script_run(ngx_http_request_t *r, ngx_str_t *value,
    void *code_lengths, size_t reserved, void *code_values);
</pre>
<b>r</b> -- запрос, в контексте которого выполняется скрипт,<br>
<b>value</b> -- указатель на строку, в которую будет помещен результат,<br>
<b>code_lengths</b> -- указатель на код для получения длины результата,<br>
<b>reserved</b> -- зарезервированный аргумент,<br>
<b>code_values</b> -- указатель на код для получения результата,<br>
<b>результат</b> -- указатель на байт памяти, следующий за последним 
байтом результата, либо NULL, если при выполнении скрипта произошла ошибка.<br>
<br>
Пример:<br>
<pre>
[...]
{
    ngx_str_t value;

    if (ngx_http_script_run(r, &amp;value, lengths-&gt;elts, 0,
        values-&gt;elts) == NULL)
    {
        return NGX_ERROR;
    }

    [...]
}
</pre>
<p>Если число переменных в шаблоне неизвестно, то можно использовать функцию
<i>ngx_http_script_variables_count</i> для их подсчета:</p>
<a name="ngx_http_script_variables_count">
<pre>
ngx_uint_t ngx_http_script_variables_count(ngx_str_t *value);
</pre>
<p>
<b>value</b> -- указатель на строку,<br>
<b>результат</b> -- число переменных в строке.<br>
<br>
Пример:<br>
</p>
<pre>
static ngx_str_t ngx_http_script_source = ngx_string("Your IP-address is $remote_addr");

{
    ngx_int_t                   n;
    ngx_http_script_compile_t   sc;
    ngx_array_t                 *lengths = NULL;
    ngx_array_t                 *values = NULL;

    n = ngx_http_script_variables_count(&amp;ngx_http_script_source);

    if(n &gt; 0) {
        ngx_memzero(&amp;sc, sizeof(ngx_http_script_compile_t));

        sc.cf = cf;
        sc.source = &amp;ngx_http_script_source;
        sc.lengths = &amp;lengths;
        sc.values = &amp;values;
        sc.variables = n;
        sc.complete_lengths = 1;
        sc.complete_values = 1;

        if (ngx_http_script_compile(&amp;sc) != NGX_OK) {
            return NGX_CONF_ERROR;
        }
    }

    return NGX_CONF_OK;
}
</pre>
<p>Если шаблон не содержит переменных, то можно соптимизировать вызов <i>ngx_http_script_run</i>,
проверив, что любой из векторов, сожержищих байт-код, не инициализирован:</p>
<pre>
[...]
{
    ngx_str_t value;

    if (lengths == NULL) {
        value.data = ngx_http_script_source.data;
        value.len = ngx_http_script_source.len;
    }else{
        if (ngx_http_script_run(r, &amp;value, lengths-&gt;elts, 0,
            values-&gt;elts) == NULL)
        {
            return NGX_ERROR;
        }
    }

    [...]
}
</pre>
<h2>2.8. Регулярные выражения</h2>
<p>Регулярные выражения в nginx реализованы на основе библиотеки PCRE и доступны
только при её наличии в системе. В исходном коде доступность регулярных выражений
индицируется макросом <i>NGX_PCRE</i>. Регулярные выражения используюутся по принципу
аналогичному скриптам: компилируются один раз, затем многократно выполняются. Для компиляции
регулярного выражения используется функция <i>ngx_regex_compile</i>:</p>
<a name="ngx_regex_compile">
<pre>
#if (NGX_PCRE)
typedef pcre  ngx_regex_t;

ngx_regex_t *ngx_regex_compile(ngx_str_t *pattern, ngx_int_t options,
    ngx_pool_t *pool, ngx_str_t *err);
#endif
</pre>
<p>
<b>pattern</b> -- указатель на строку, содержащую регулярное выражение;<br>
<b>options</b> -- флаги, задающие некоторые параметры;<br>
<b>pool</b> -- пул, в котором будет выделена память под регулярное выражение;<br>
<b>err</b> -- строка, содержащая текстовое описание ошибки, которая произошла при компиляции регулярного выражения;<br>
<b>результат</b> -- указатель на структуру, содержащую скомпилированное регулярное выражение.<br>
<br>
Параметр <i>options</i> может содержать флаг <i>NGX_REGEX_CASELESS</i>, означающий, что
регулярное выражение не чувствительно к регистру символов.</p>
<p>Пример компиляции регулярного выражения:</p>
<pre>
</pre>
<p>Для подсчета числа ссылок в регулярном выражении используется
функция <i>ngx_regex_capture_count</i></p>
<a name="ngx_regex_capture_count">
<pre>
#if (NGX_PCRE)
ngx_int_t ngx_regex_capture_count(ngx_regex_t *re);
#endif
</pre>
<p>
<b>re</b> -- указатель на скомпилированное регулярное выражение;<br>
<b>результат</b> -- число ссылок.</p>
<p>Для выполнения регулярного выражения используется
функция <i>ngx_regex_exec</i></p>
<a name="ngx_regex_exec">
<pre>
#if (NGX_PCRE)
ngx_int_t ngx_regex_exec(ngx_regex_t *re, ngx_str_t *s, int *captures,
    ngx_int_t size);
#endif
</pre>
<p>
<b>re</b> -- указатель на скомпилированное регулярное выражение;<br>
<b>s</b> -- указатель строку, для которой будет выполнено регулярное выражение;<br>
<b>captures</b> -- указатель вектор, в который будут помещены позиции подстрок,
соответствующие ссылкам;<br>
<b>size</b> -- число элементов вектора captures;<br>
<b>результат</b> -- 0, если регулярное выражение совпало, <i>NGX_REGEX_NO_MATCHED</i>,
если регулярное выражение не совпало, значение меньше <i>NGX_REGEX_NO_MATCHED</i> если
произошла ошибка.</p>
<p>Число элементов вектора <i>captures</i> должно быть ровно в три раза больше
числа ссылок в регулярном выражении. В первые две трети вектора помещаются
позиции подстрок, соответствующие ссылкам в регулярном выражении, оставшаяся
часть используется библиотекой PCRE для внутренних нужд. Каждый первый
элемент первых двух третей вектора содержит позицию первого символа подстроки,
каждый второй -- позицию, следющую за позицией последнего символа подстроки.</p>
<h2>2.9. Конфигурация модуля</h2>
<a name="2.9"/>
<p>Конфигурация модуля во время выполнения храниться в бинарном виде
в структурах определяемых разработчиком. HTTP-запрос связывается с
конфигурациями трех уровней: основной, виртуального сервера
и location'а. На каждом из уровней имеет смысл хранить только те
параметры конфигурации, которые нельзя разделить между
экземплярами конфигураций последующих уровней. Например,
имена виртуального сервера и адрес слушающего сокета
нельзя разделить между location'ами, поэтому имеет смысл
хранить эти параметры в конфигурации виртуального сервера.
Для доступа к конфигурациям всех уровней
во время разбора файла конфигурации используются следующие <i>макросы</i>:<br>
<a name="ngx_http_conf_get_module_main_conf">
<a name="ngx_http_conf_get_module_srv_conf">
<a name="ngx_http_conf_get_module_loc_conf">
<pre>
ngx_http_conf_get_module_main_conf(cf, module)
ngx_http_conf_get_module_srv_conf(cf, module)
ngx_http_conf_get_module_loc_conf(cf, module)
</pre>
<b>cf</b> -- указатель на структуру ngx_conf_t (конфигурация),<br>
<b>module</b> -- структура <a href="#ngx_module_t">ngx_module_t</a> (описание модуля),<br> 
<b>rvalue</b> -- указатель на конфигурацию модуля соответствующего
уровня.<br>
</p>
<p>Для доступа к конфигурациям всех уровней
во время обработки запроса используются следующие <i>макросы</i>:<br>
<a name="ngx_http_get_module_main_conf">
<a name="ngx_http_get_module_srv_conf">
<a name="ngx_http_get_module_loc_conf">
<pre>
ngx_http_get_module_main_conf(r, module)
ngx_http_get_module_srv_conf(r, module)
ngx_http_get_module_loc_conf(r, module)
</pre>
<b>r</b> -- указатель на структуру ngx_http_request_t (запрос),<br>
<b>module</b> -- структура <a href="#ngx_module_t">ngx_module_t</a> (описание модуля),<br> 
<b>rvalue</b> -- указатель на конфигурацию модуля соответствующего
уровня.<br>
<h2>2.10. Контекст модуля</h2>
<p>Контекст модуля во время обработки запроса храниться в бинарном виде
в структурах определяемых разработчиком. Для установки контекста модуля
используется следующий макрос:</p>
<a name="ngx_http_set_ctx">
<pre>
ngx_http_set_ctx(r, c, module)
</pre>
<p>
<b>r</b> -- указатель на структуру ngx_http_request_t (запрос),
<b>c</b> -- указатель на контекст модуля (структура определяемая разработчиком),
<b>module</b> -- структура <a href="#ngx_module_t">ngx_module_t</a> (описание модуля). 
</p>
<p>Для доступа к контексту модуля используется следующий макрос:</p>
<a name="ngx_http_get_module_ctx">
<pre>
ngx_http_get_module_ctx(r, module)
</pre>
<p>
<b>r</b> -- указатель на структуру ngx_http_request_t (запрос),
<b>module</b> -- структура <a href="#ngx_module_t">ngx_module_t</a> (описание модуля). 
<b>rvalue</b> -- указатель на контекст модуля.<br>
</p>
<h2>3. Nginx и модули</h2>
<h2>3.1. Фазы обработки запросов в nginx</h2>
<p>Nginx обрабатывает запросы с использованием нескольких фаз. На каждой фазе
вызываются 0 или более хэндлеров. 
<ol>
<li>NGX_HTTP_SERVER_REWRITE_PHASE -- фаза преобразования URI запроса на уровне виртуального сервера;</li>
<li>NGX_HTTP_FIND_CONFIG_PHASE -- фаза поиска контекста запроса (location'a);</li>
<li>NGX_HTTP_REWRITE_PHASE -- фаза преобразования URI запроса на уровне location'a;</li>
<li>NGX_HTTP_POST_REWRITE_PHASE -- фаза обработки результатов преобразования URI запроса;</li>
<li>NGX_HTTP_PREACCESS_PHASE -- фаза подготовки данных для проверки ограничений доступа к ресурсу;</li>
<li>NGX_HTTP_ACCESS_PHASE -- фаза проверки ограничений доступа к ресурсу;</li>
<li>NGX_HTTP_POST_ACCESS_PHASE -- фаза обработки результатов проверки ограничений доступа к ресурсу;</li>
<li>NGX_HTTP_CONTENT_PHASE -- фаза генерации ответа;</li>
<li>NGX_HTTP_LOG_PHASE -- фаза записи в лог.</li>
</ol>
На всех фазах могут быть зарегистрированы пользовательские хэндлеры, за
исключением следующих фаз:<br>
<ul>
<li>NGX_HTTP_FIND_CONFIG_PHASE. На этой фазе вызов хэндлеров не происходит,
вместо этого выполняется поиск контекста запроса (location'a) и заполнение
заголовка ответа "Location".</li>
<li>NGX_HTTP_POST_ACCESS_PHASE. На этой фазе вызов хэндлеров не происходит,
а лишь применяется результат проверки доступа к ресурсу. Фаза необходима для
реализации директивы <i>satisfy all/any</i>.</li>
<li>NGX_HTTP_POST_REWRITE_PHASE. На этой фазе вызов хэндлеров не происходит,
вместо этого выполняется </li>
</ul>
Хэндлером могут быть возвращены следующие константы:
<ul>
<li>NGX_OK -- выполнение хэндлера завершено успешно, необходимо перейти к
выполнению следующей фазы запроса;</li>
<li>NGX_DECLINED -- запрос не предназначен данному хэндлеру, необходимо перейти к следующему хэндлеру;</li>
<li>NGX_AGAIN (для фаз server rewrite, rewrite, preaccess, access), NGX_DONE (для фазы content) -- выполнение хэндлера завершено успешно, необходимо подождать появления некоторого события (например, возможности записи в сокет) и повторить вызов хэндлера;</li>
<li>NGX_ERROR, NGX_HTTP_... -- при выполнении хэндлера произошла ошибка.</li>
</ul>
</p>
<p>Для регистрации хэндлера необходимо обратиться к основной конфигурации
модуля <i>ngx_http_core_module</i> и добавить хэндлер к одному из элементов
вектора <i>phases</i>. Пример регистрации хэндлера на фазе NGX_HTTP_CONTENT_PHASE:</p>
<pre>
static ngx_int_t
ngx_http_sample_module_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&amp;cmcf-&gt;phases[NGX_HTTP_CONTENT_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_sample_module_handler;

    return NGX_OK;
}
</pre>
<p>Хэндлеры фаз вызываются вне зависимости от конфигурации. В связи с этим, хэндлер должен
уметь определять когда он не применим и возвращать NGX_DECLINED, и делать это как можно быстрее,
чтобы избежать потери производительности.</p>
<p>Фаза NGX_HTTP_ACCESS_PHASE используется для вызова хэндлеров, ограничивающих доступ к
ресурсам. На этой фазе порядок перехода к следующим хэндлерам
или фазам определяеться директивой <i>satisfy</i>. Значения, возвращаемые
хэндлером, преобретают дополнительный смысл:
<ul>
<li>NGX_OK -- хэндлер позволяет обратиться к ресурсу;</li>
<li>NGX_HTTP_FORBIDDEN, NGX_HTTP_UNAUTHORIZED -- хэндлер не позволяет обратиться к ресурсу.</li>
</ul>
В случае <i>satisfy all</i> для перехода к следующей фазе необходимо, чтобы все хэндлеры вернули NGX_OK.
В случае <i>satisfy any</i> нужно чтобы хотя бы один хэндлер вернул NGX_OK.</p>
<p>Фаза NGX_HTTP_CONTENT_PHASE используется для генерации ответа. Если в конфигурации уровня location'а
модуля <i>ngx_http_core_module</i> переопределен параметр <i>handler</i>, то все запросы направляются этому хэндлеру,
в противном случае используются хэндлеры фазы NGX_HTTP_CONTENT_PHASE из основной конфигурации. Хэндлер location'а
не может быть возобновлен: возврат NGX_DONE не приводит к повторному вызову хэндлера. Пример
переопределния хэндлера location'а:</p>
<pre>
static char *
ngx_http_sample_module_command(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_core_loc_conf_t  *clcf;

    clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
    clcf-&gt;handler = ngx_http_sample_handler;

    return NGX_CONF_OK;
}
</pre>
<h2>3.2. Встраивание модуля в nginx</h2>
<p>Для встраивания в nginx модуль должен содержать метаинформацию, которая
описывает как инициализировать и конфигурировать модуль. Метаинформация
представляется структурой <i>ngx_module_t</i>:
<a name="ngx_module_t">
<pre>
struct ngx_module_s {

    [...]

    ngx_uint_t            version;

    void                 *ctx;
    ngx_command_t        *commands;
    ngx_uint_t            type;

    [...]

    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);

    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);

    [...]

    void                (*exit_process)(ngx_cycle_t *cycle);

    void                (*exit_master)(ngx_cycle_t *cycle);

    [...]

};

typedef struct ngx_module_s      ngx_module_t;
</pre>
Назначение полей:<br>
<b>version</b> -- содержит версию модуля (на данный момент 1),<br>
<b>ctx</b> -- указатель на глобальный контекст модуля,<br> 
<b>commands</b> -- указатель на вектор описателей директив модуля,<br>
<b>type</b> -- тип модуля: NGX_HTTP_MODULE, NGX_EVENT_MODULE, NGX_MAIL_MODULE и другие,<br>
<b>init_module</b> -- хэндлер, вызываемый при инициализации модуля в основном процессе,<br>
<b>init_process</b> -- хэндлер, вызываемый при инициализации рабочего процесса,<br>
<b>exit_process</b> -- хэндлер, вызываемый при завершении рабочего процесса,<br>
<b>exit_master</b> -- хэндлер, вызываемый при завершении основного процесса.<br>
<br>
Пример определения:</p>
<pre>
#include &lt;ngx_config.h&gt;
#include &lt;ngx_core.h&gt;

[...]

ngx_module_t  ngx_http_some_module = {
    NGX_MODULE_V1,
    &amp;ngx_http_some_module_ctx,             /* module context */
    ngx_http_some_commands,                /* module directives */
    NGX_HTTP_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
</pre>
<p>
Примечание: экземпляр типа <i>ngx_module_t</i> должен быть объявлен с
квалификатором <i>extern</i>. Поскольку все определения имеют
квалификатор <i>extern</i>, то в примере он опущен.
</p>
<h3>Описание HTTP-модуля</h3>
<p>HTTP-модули в поле <b>ctx</b> структуры <i>ngx_module_t</i> содержат указатель на структуру <i>ngx_http_module_t</i>.
</p>
<a name="ngx_http_module_t">
<pre>
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
</pre>
<p>
Назначение полей структуры:<br>
<b>preconfiguration</b> -- хэндлер, вызываемый перед обработкой файла конфигурации,<br>
<b>postconfiguration</b> -- хэндлер, вызываемый после обработки файла конфигурации,<br> 
<b>create_main_conf</b> -- хэндлер, вызываемый для создания основной конфигурации,<br>
<b>init_main_conf</b> -- хэндлер, вызываемый для инициализации основной конфигурации,<br>
<b>create_srv_conf</b> -- хэндлер, вызываемый для создания конфигурации виртуального сервера,<br>
<b>merge_srv_conf</b> -- хэндлер, вызываемый для слияния конфигураций виртуального сервера,<br>
<b>create_loc_conf</b> -- хэндлер, вызываемый создания конфигурации location'а,<br>
<b>merge_loc_conf</b> -- хэндлер, вызываемый для слияния конфигураций location'а.<br>
<br>
Любое из полей может содержать значение NULL, означающее, что вызывать хэндлер не нужно.
</p>
<p>Пример определения:</p>
<pre>
ngx_http_module_t  ngx_http_some_module_ctx = {
    ngx_http_some_module_add_variables,    /* preconfiguration */
    NULL,                                  /* postconfiguration */

    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */

    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */

    ngx_http_some_module_create_loc_conf,  /* create location configuration */
    NULL                                   /* merge location configuration */
};
</pre>
<h2>3.2.1. Описание и обработка директив конфигурации модуля</h2>
<p>Директивы конфигурации описываются структурой <i>ngx_command_t</i>:</p>
<a name="ngx_command_t">
<a name="ngx_null_command">
<pre>
struct ngx_command_s {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};

#define ngx_null_command  { ngx_null_string, 0, NULL, 0, 0, NULL }

typedef struct ngx_command_s     ngx_command_t;
</pre>
<p>
Назначение полей структуры:<br>
<b>name</b> -- название директивы,<br>
<b>type</b> -- тип директивы, число аргументов и блоки, в которых может присутствовать директива:<br> 
<table border="1">
    <tr>
        <td><b>Флаг</b></td><td><b>Значение</b></td>
    </tr>
    <tr>
        <td>NGX_CONF_NOARGS</td><td>Директива не принимает аргументов</td>
    </tr>
    <tr>
        <td>NGX_CONF_TAKE1 ... NGX_CONF_TAKE7</td><td>Директива принимает указанное количество аргументов</td>
    </tr>
    <tr>
        <td>NGX_CONF_TAKE12</td><td>Директива принимает 1 или 2 аргумента</td>
    </tr>
    <tr>
        <td>NGX_CONF_TAKE13</td><td>Директива принимает 1 или 3 аргумента</td>
    </tr>
    <tr>
        <td>NGX_CONF_TAKE123</td><td>Директива принимает 1, 2 или 3 аргумента</td>
    </tr>
    <tr>
        <td>NGX_CONF_TAKE1234</td><td>Директива принимает 1, 2, 3 или 4 аргумента</td>
    </tr>
    <tr>
        <td>NGX_CONF_BLOCK</td><td>Дополнительный аргумент директивы является блоком</td>
    </tr>
    <tr>
        <td>NGX_CONF_FLAG</td><td>Директива является флагом</td>
    </tr>
    <tr>
        <td>NGX_CONF_ANY</td><td>Директива принимает 0 или более аргументов</td>
    </tr>
    <tr>
        <td>NGX_CONF_1MORE</td><td>Директива принимает 1 или более аргументов</td>
    </tr>
    <tr>
        <td>NGX_CONF_2MORE</td><td>Директива принимает 2 или более аргументов</td>
    </tr>
    <tr>
        <td>NGX_DIRECT_CONF</td><td>Директива может присутствовать в основном файле конфигурации</td>
    </tr>
    <tr>
        <td>NGX_MAIN_CONF</td><td>Директива может присутствовать на корневом уровне конфигурации</td>
    </tr>
    <tr>
        <td>NGX_ANY_CONF</td><td>Директива может присутствовать на любом уровне конфигурации</td>
    </tr>
    <tr>
        <td>NGX_HTTP_MAIN_CONF</td><td>Директива может присутствовать на уровне физического HTTP-сервера</td>
    </tr>
    <tr>
        <td>NGX_HTTP_SRV_CONF</td><td>Директива может присутствовать на уровне виртуального HTTP-сервера</td>
    </tr>
    <tr>
        <td>NGX_HTTP_LOC_CONF</td><td>Директива может присутствовать на уровне location'а</td>
    </tr>
    <tr>
        <td>NGX_HTTP_LMT_CONF</td><td>Директива может присутствовать в блоке limit_except</td>
    </tr>
    <tr>
        <td>NGX_HTTP_LIF_CONF</td><td>Директива может присутствовать в блоке if() на уровне локейшена</td>
    </tr>
</table>
<b>set</b> -- хэндлер, вызываемый в момент обнаружения директивы. Для удобства реализовано множество
стандартных хэндлеров:<br>
<table border="1">
        <td><b>Название хэндлера</b></td><td><b>Тип данных</b></td><td><b>Тип поля в конфигурации модуля, на который указывает offset</b></td>
    </tr>
    <tr>
        <td><a name="ngx_conf_set_flag_slot"/>ngx_conf_set_flag_slot</td><td>Флаг</td><td>ngx_flag_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_set_str_slot"/>ngx_conf_set_str_slot</td><td>Строка</td><td>ngx_str_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_set_str_array_slot"/>ngx_conf_set_str_array_slot</td><td>Вектор строк</td><td>Указатель на ngx_array_t -&gt; ngx_str_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_set_keyval_slot"/>ngx_conf_set_keyval_slot</td><td>Вектор пар ключ-значение</td><td>Указатель на ngx_array_t -&gt; ngx_keyval_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_set_num_slot"/>ngx_conf_set_num_slot</td><td>Целое число со знаком</td><td>ngx_int_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_set_size_slot"/>ngx_conf_set_size_slot</td><td>Длина</td><td>size_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_set_off_slot"/>ngx_conf_set_off_slot</td><td>Смещение</td><td>off_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_set_msec_slot"/>ngx_conf_set_msec_slot</td><td>Миллисекунды</td><td>ngx_msec_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_set_sec_slot"/>ngx_conf_set_sec_slot</td><td>Секунды</td><td>time_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_set_bufs_slot"/>ngx_conf_set_bufs_slot</td><td>Число и размер буферов</td><td>ngx_bufs_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_set_enum_slot"/>ngx_conf_set_enum_slot</td><td>Перечисляемое значение</td><td>ngx_uint_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_set_bitmask_slot"/>ngx_conf_set_bitmask_slot</td><td>Битовая карта</td><td>ngx_uint_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_set_path_slot"/>ngx_conf_set_path_slot</td><td>Путь в файловой системе и число символов в хэшированных каталогах</td><td>ngx_path_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_set_access_slot"/>ngx_conf_set_access_slot</td><td>Права доступа</td><td>ngx_uint_t</td>
    </tr>
</table>
<b>conf</b> -- уровень конфигурации модуля, на которую ссылается директива, либо 0, если
директива имеет специализированный обработчик,<br>
<b>offset</b> -- смещение поля в конфигурации модуля, которое задается этой директивой,<br>
<b>post</b> -- указатель на дескриптор постпроцессинга,<br>
</p>
<p>Список директив модуля описываться вектором, который заканчивается значением <a href="#ngx_null_command">ngx_null_command</a>. Пример:</p>
<pre>
typedef struct {
    ngx_str_t   foobar;
} ngx_http_some_module_loc_conf_t;

static ngx_command_t  ngx_http_some_module_commands[] = {

    { ngx_string("foobar"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_str_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_some_module_loc_conf_t, foobar),
      NULL },

    ngx_null_command
};
</pre>
<p>Описана директива <b>foobar</b>, принимающая 1 аргумент. Директива может
присутствовать на основном уровне конфигурации, конфигурации виртуального HTTP-сервера
и локейшена. Аргумент директивы конвертируется в строку и записывается
в поле foobar конфигурации модуля .</p>
<h3>Создание конфигураций</h3>
<p>Перед обработкой файла конфигурации структуры, содержащие конфигурацию, должны быть
выделены и инициализированы. Для этого используются хэндлеры <i>create_main_conf</i>,
<i>create_srv_conf</i> и <i>create_loc_conf</i>. </p>
<h3>Слияние конфигураций</h3>
<p>Для упрощения конфигурирования, на каждом из уровней конфигурации создается шаблон
конфигурации каждого из последующих уровней. Рассмотрим пример:</p>
<pre>
http {
    server {

        zip_buffers 10 4k;

        location /foobar {
            # gzip_buffers 10 4k наследована с предыдущего уровня
            gzip on;
        }

        location /foobaz {
            # gzip_buffers 10 4k наследована с предыдущего уровня
        }
    }
}
</pre>
<p>При обработке блока <i>server</i> будет создана шаблонная конфигурация для модуля
ngx_http_gzip_filter_module и к ней будет применена директива <i>gzip_buffers</i>. При
переходе к обработке блока location /foobar {} будет создана ещё одна конфигурация
для модуля ngx_http_gzip_filter_module и к ней будет применена директива <i>gzip</i>.
После обработки блока http {} шаблонную конфигурацию необходимо слить с 
конфигурацией блоков /foobar и /foobaz. В процессе слияния все неустановленные параметры
конфигурации заполняются значениями из соответствующих параметров шаблонной конфигурации,
либо значением по-умолчанию. Для упрощения слияния используются следующие <i>макросы</i>:</p>
<table border="1">
    <tr>
        <td><b>Название</b></td><td><b>Тип данных</b></td><td><b>Тип поля в конфигурации модуля</b></td>
    </tr>
    <tr>
        <td><a name="ngx_conf_merge_ptr_value"/>ngx_conf_merge_ptr_value</td><td>Указатель</td><td>pointer</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_merge_uint_value"/>ngx_conf_merge_uint_value</td><td>Целое число</td><td>ngx_uint_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_merge_msec_value"/>ngx_conf_merge_msec_value</td><td>Время в миллисекундах</td><td>ngx_msec_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_merge_sec_value"/>ngx_conf_merge_sec_value</td><td>Время в секундах</td><td>time_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_merge_size_value"/>ngx_conf_merge_size_value</td><td>Длина</td><td>size_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_merge_bufs_value"/>ngx_conf_merge_bufs_value</td><td>Число и размер буферов</td><td>ngx_bufs_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_merge_bitmask_value"/>ngx_conf_merge_bitmask_value</td><td>Битовая карта</td><td>ngx_uint_t</td>
    </tr>
    <tr>
        <td><a name="ngx_conf_merge_path_value"/>ngx_conf_merge_path_value</td><td>Путь в файловой системе и число символов в хэшированных каталогах</td><td>ngx_path_t</td>
    </tr>
</table>
<p></p>
<p>Для слияния конфигурации виртуального сервера используется хэндлер <i>merge_srv_conf</i>,
для конфигурации location'a используется хэндлер <i>merge_loc_conf</i>. Пример:</p>
<pre>
typedef struct {
    ngx_str_t str_param;
    ngx_uint_t int_param;
} ngx_http_sample_module_loc_conf_t;

static char *
ngx_http_sample_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
{
    ngx_http_sample_module_loc_conf_t  *prev = parent;
    ngx_http_sample_module_loc_conf_t  *conf = child;

    ngx_conf_merge_str_value(conf-&gt;str_param, prev-&gt;str_param, "default value");

    ngx_conf_merge_uint_value(conf-&gt;int_param,
                              prev-&gt;int_param, 1);

    return NGX_CONF_OK;
}

ngx_http_module_t  ngx_http_some_module_ctx = {
    NULL,                                  /* preconfiguration */
    NULL,                                  /* postconfiguration */

    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */

    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */

    ngx_http_some_module_create_loc_conf,  /* create location configuration */
    ngx_http_some_module_merge_loc_conf    /* merge location configuration */
};
</pre>
<h2>3.2.2. Описание и вычисление переменных модуля</h2>
<p>Поддерживаемые модулем переменные должны быть созданы перед разбором
блоков файла конфигурации, в которых эти переменные могут встретиться.
Чтобы создать переменные до разбора файлов конфигурации нужно использовать
хэндлер <i>preconfiguration</i> в структуре
<a href="#ngx_http_module_t">ngx_http_module_t</a>. Пример:</p>
<pre>
static ngx_int_t
ngx_http_some_module_add_variables(ngx_conf_t *cf);

ngx_http_module_t  ngx_http_some_module_ctx = {
    ngx_http_some_module_add_variables,    /* preconfiguration */

    [...]
};

static ngx_http_variable_t ngx_http_some_module_variables[] = {

    { ngx_string("var"), NULL, ngx_http_some_module_variable,
      0,
      NGX_HTTP_VAR_NOCACHEABLE, 0 },

    { ngx_null_string, NULL, NULL, 0, 0, 0 }
};

static ngx_int_t
ngx_http_some_module_add_variables(ngx_conf_t *cf)
{
    ngx_http_variable_t  *var, *v;

    for (v = ngx_http_some_module_variables; v-&gt;name.len; v++) {
        var = ngx_http_add_variable(cf, &amp;v-&gt;name, v-&gt;flags);
        if (var == NULL) {
            return NGX_ERROR;
        }

        var-&gt;get_handler = v-&gt;get_handler;
        var-&gt;data = v-&gt;data;
    }

    return NGX_OK;
}
</pre>
Для генерации значения переменной необходимо реализовать функцию, которая заполняет
структуру <a href="#ngx_http_variable_value_t">ngx_http_variable_value_t</a>, используя
данные из запроса, из контекстов, конфигураций модулей или из других источников. Пример:
<pre>
static ngx_int_t
ngx_http_some_module_variable(ngx_http_request_t *r,
    ngx_http_variable_value_t *v,  uintptr_t data)
{
    v-&gt;valid = 1;
    v-&gt;no_cacheable = 1;
    v-&gt;not_found = 0;

    v-&gt;data = (u_char*)"42";
    v-&gt;len = 2;

    return NGX_OK;
}
</pre>
</p>
<h2>3.3. Компиляция модуля и сборка с nginx</h2>
<p>Для сборки модуля с nginx необходимо указать путь к каталогу с исходным кодом модуля скрипту
./configure в параметре --add-module= командной строки. В указанном каталоге должен
находиться файл config. Файл config -- это скрипт, который система сборки
nginx включает и выполняет на стадии конфигурации. Задача скрипта -- установить набор
переменных, управляющих сборкой. Список наиболее важных переменных:</p>
<table border="1">
    <tr>
        <td><b>Имя перменной</b></td><td><b>Назначение</b></td>
    </tr>
    <tr>
        <td>ngx_addon_name</td><td>Имя текущего дополнительного модуля</td>
    </tr>
    <tr>
        <td>NGX_ADDON_SRCS</td><td>Список всех исходных файлов всех дополнительных модулей, которые нужно скомпилировать</td>
    </tr>
    <tr>
        <td>NGX_ADDON_DEPS</td><td>Список всех зависимых файлов всех дополнительных модулей (как правило заголовочные файлы).</td>
    </tr>
    <tr>
        <td>HTTP_MODULES</td><td>Список всех HTTP модулей</td>
    </tr>
    <tr>
        <td>HTTP_AUX_FILTER_MODULES</td><td>Список всех вспомогательных фильтров</td>
    </tr>
    <tr>
        <td>USE_MD5</td><td>Использовать ли поддержку MD5 (YES/NO)</td>
    </tr>
    <tr>
        <td>USE_SHA1</td><td>Использовать ли поддержку SHA-1 (YES/NO)</td>
    </tr>
    <tr>
        <td>USE_ZLIB</td><td>Использовать ли библиотеку zlib (YES/NO)</td>
    </tr>
</table>
<p>Для ссылок на каталог, в котором расположены файлы модуля, используется
переменная <i>ngx_addon_dir</i>.  Пример файла config:</p>
<pre>
ngx_addon_name=ngx_http_sample_module
HTTP_MODULES="$HTTP_MODULES ngx_http_sample_module"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_sample_module.c"
</pre>
<p>Предположим, файлы модуля расположены в каталоге /home/valery/work/sample_module. Для
включения модуля nginx нужно конфигурировать следующим образом:</p>
<pre>
path/to/nginx$ ./configure --add-module=/home/valery/work/sample_module
</pre>
<p>Далее:</p>
<pre>
path/to/nginx$ make
path/to/nginx$ make install
</pre>
<p>Инсталлированному экземпляру nginx станут доступны директивы и переменные подключенного модуля.</p>
<h2>4. Модули</h2>
<p>Это -- недописанная глава. Она должна содержать важный и интересный материал, но у автора пока нет хорошей идеи, относительно того,
как его преподнести.</p>
<h2>Связаться с автором</h2>
<p>Valery Kholodkov <a href="mailto:valery+nginx@grid.net.ru">valery+nginx@grid.net.ru</a><br>Пожалуйста, используйте расширение адреса при составлении письма мне.</p>
<h2>Ссылки</h2>
<p>Nginx: <a href="http://www.sysoev.ru/nginx/">www.sysoev.ru/nginx/</a> -- это веб-сервер разработанный Игорем Сысоевым.</p>
<h2>Благодарности</h2>
<ul>
<li>Кирилл Коринский. Глава <a href="#2.3">2.3</a>. См. оригинал: <a href="http://catap.ru/blog/2008/03/22/nginx-programming-guide-list/">nginx programming guide: list</a></li>
</ul>
<hr>
Copyright (C) 2008 Valery Kholodkov
</body>
</html>

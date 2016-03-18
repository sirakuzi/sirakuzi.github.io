---
title: "Расширенные возможности в разработке модулей для Nginx от Эмиллера"
description: "Неполный перевод"
categories: [posts]
layout: post
time: 15:48:25
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
<small><p>Оригинал: <a href="http://www.evanmiller.org/nginx-modules-guide-advanced.html">Emiller's Advanced Topics In Nginx Module Development</a>.
<br>Автор: <a href="http://www.evanmiller.org/">Эван Миллер</a> (в соавторстве с <a href="http://nginx.localdomain.pl/">Гжегожем Носеком (Grzegorz Nosek)</a>)
<br>Неполный перевод: <a href="http://sirakuzi.github.io/">sirakuzi</a>.
<br>Перевод в процессе, последнее обновление 18 марта 2016 г.
<br>Черновик от 13 августа 2009 г.</p></small>
</section>

<section>
<p>В свою очередь <em><a href="/posts/2016/01/12/emillers-nginx-modules-guide-ru.html">Руководство Эмиллера по разработке модулей для Nginx</a></em> описывает насущные проблемы при написании простого обработчика, фильтра или балансировщика нагрузки для Nginx, в то время как этот документ рассматривает три расширенных возможности для амбициозных разработчиков Nginx: общая память, подзапросы и парсинг (обработку). Поскольку эти темы находятся на границе вселенной Nginx, код представленный здесь довольно скуден. Примеры так же могут быть неактуальными. Но будем надеятся, что вам удастся не только остаться на плаву, но и приобрести несколько новых приспособлений в свой пояс.</p>
</section>

<section>
<h2>Содержание</h2>
    <ol>
        <li><a href="#shm">Общая память</a></li>
        <ol>
            <li><a href="#shm-foreword">A (fore)word of caution</a></li>
            <li><a href="#shm-creating">Creating and using a shared memory segment</a></li>
            <li><a href="#shm-slab">Using the slab allocator</a></li>
            <li><a href="#shm-spinlocks">Spinlocks, atomic memory access</a></li>
            <li><a href="#shm-rbtrees">Using rbtrees</a></li>
        </ol>
        <li><a href="#subrequests">Подзапросы</a></li>
        <ol>
            <li><a href="#subrequests-redirect">Внутреннее перенаправление</a></li>
            <li><a href="#subrequests-single">Одиночный подзапрос</a></li>
            <li><a href="#subrequests-sequential">Sequential subrequests</a></li>
            <li><a href="#subrequests-parallel">Parallel subrequests</a></li>
        </ol>
        <li><a href="#parsing">Parsing With Ragel</a> *NEW*</li>
        <ol>
            <li><a href="#parsing-install">Installing ragel</a></li>
            <li><a href="#parsing-call">Calling ragel from nginx</a></li>
            <li><a href="#parsing-grammar">Writing a grammar</a></li>
            <li><a href="#parsing-actions">Writing some actions</a></li>
            <li><a href="#parsing-finale">Putting it all together</a></li>
        </ol>
        <li><a href="#todo">TODO</a></li>
    </ol>
</section>

<section>
<a name="shm"></a>
<h2>1. Общая память</h2>

<p><em>Глава написана <a href="http://localdomain.pl/">Гжегожем Носеком</a></em></p>

<p>Nginx, будучи без поддержки потоков, позволяет рабочим процессам иметь общую память. Однако, это сильно отличается от распределителя пула памяти к которому мы привыкли, т.к. у разделяемого сегмента размер фиксирован и не может быть изменен без перезагрузки Nginx'а или же без уничтожения всех содержащихся там данных.</p>
</section>

<section>
<a name="shm-foreword"></a>
<h3>1.1. Введение - предупреждение</h3>
<p>Во-первых, пользуйтесь на свой страх и риск. Этот гайд был написан через несколько месяцев после опыта работы с общей памятью в Nginx и хоть я и пытаюсь быть предельно точным (и я потратил какое-то время чтобы освежить свою память), но ни в коей мере это не гарантируется. Вы предупреждены.</p>

<p>Кроме того, 100% этих знаний были почерпаны из чтения исходного кода и обратного инжиниринга основных процессов, так что вероятно есть лучшие способы сделать описанные тут вещи.</p>

<p>А, и это руководство основано на версии 0.6.31, тем не менее версия насколько я знаю 0.5.x совместима на 100%, и 0.7.x так же не должен вносить каких-либо изменений влияющих на совместимость.</p>

<p>В качестве реального примера использования общей памяти в Nginx см. <a href="http://github.com/gnosek/nginx-upstream-fair/tree/master">модуль upstream_fair</a>.</p>

<p>Всё это скорее всего совсем не будет работать на Windows. Дампы памяти в зеркале заднего вида ближе чем они кажутся.</p>
</section>

<section>
<a name="shm-creating"></a>
<h3>1.2. Создание и использование сегмента общей памяти</h3>
<p>
Чтобы создать сегмент общей памяти в Nginx вам потребуется:
<ul>
<li>предоставить функцию-конструктор для инициализации сегмента</li>

<li>вызвать <code>ngx_shared_memory_add</code></li>
</ul>
</p>


<p>
Эти два пункта содержат основные подводные камни (на которые я наткнулся), а именно:
<section><ol>
<li><p>Your constructor will be called multiple times and it's up to you to find out whether you're called the first time (and should set something up), or not (and should probably leave everything alone). The prototype for the shared memory constructor looks like:
<pre>
static ngx_int_t init(ngx_shm_zone_t *shm_zone, void *data);
</pre>
</p>
<p>The data variable will contain the contents of <code>oshm_zone-&gt;data</code>, where <code>oshm_zone</code> is the "old" shm zone descriptor (more about it later). This variable is the only value that can survive a reload, so you must use it if you don't want to lose the contents of your shared memory.</p>

<p>Your constructor function will probably look roughly similar to the one
in upstream_fair, i.e.:

<code><pre>
static ngx_int_t
init(ngx_shm_zone_t *shm_zone, void *data)
{
        if (data) { /* we're being reloaded, propagate the data "cookie" */
                shm_zone-&gt;data = data;
                return NGX_OK;
        }

        /* set up whatever structures you wish to keep in the shm */

        /* initialise shm_zone-&gt;data so that we know we have
        been called; if nothing interesting comes to your mind, try
        shm_zone-&gt;shm.addr or, if you're desperate, (void*) 1, just set
        the value to something non-NULL for future invocations
        */
        shm_zone-&gt;data = something_interesting;

        return NGX_OK;
}
</pre></code>
</p>
</li>
<li>
<p>You must be careful when to access the shm segment.</p>

<p>The interface for adding a shared memory segment looks like:
<code><pre>
ngx_shm_zone_t *
ngx_shared_memory_add(ngx_conf_t *cf, ngx_str_t *name, size_t size,
        void *tag);
</pre></code>
</p>

<p><code>cf</code> is the reference to the config file (you'll probably create the segment in response to a config option), name is the name of the segment (as a <code>ngx_str_t</code>, i.e. a counted string), size is the size in bytes (which will usually get rounded up to the nearest multiple of the page size, e.g. 4KB on many popular architectures) and tag is a, well, tag for detecting naming conflicts. If you call <code>ngx_shared_memory_add</code> multiple times with the same name, tag and size, you'll get only a single segment. If you specify different names, you'll get several distinct segments and if you specify the same name but different size or tag, you'll get an error. A good choice for the tag value could be e.g. the pointer to your module descriptor.</p> 

<p>After you call <code>ngx_shared_memory_add</code> and receive the new <code>shm_zone</code> descriptor, you must set up the constructor in <code>shm_zone-&gt;init</code>. Wait...  after you add the segment? Yes, and that's a major gotcha. This implies that the segment is not created while calling <code>ngx_shared_memory_add</code> (because you specify the constructor only later). What really happens looks like this (grossly simplified):

<ol>
<li><p>parse the whole config file, noting requested shm segments</p></li>
<li><p>afterwards, create/destroy all the segments in one go</p>

<p>The constructors are called here. Note that every time your ctor is called, it is with another value of <code>shm_zone</code>. The reason is that the descriptor lives as long as the cycle (generation in Apache terms) while the segment lives as long as the master and all the workers. To let <i>some</i> data survive a reload, you have access to the old descriptor's <code>-&gt;data</code> field (mentioned above).</p>
</li>
<li><p>(re)start workers which begin handling requests</p></li>
<li><p>upon receipt of SIGHUP, goto 1</p></li>

</ol>
</p>

<p>Also, you really must set the constructor, otherwise nginx will consider
your segment unused and won't create it at all.</p>

<p>Now that you know it, it's pretty clear that you cannot rely on having access to the shared memory while parsing the config. You can access the whole segment as <code>shm_zone-&gt;shm.addr</code> (which will be NULL before the segment gets really created). Any access after the first parsing run (e.g. inside request handlers or on subsequent reloads) should be fine.</p>

</ol></section></p>
</section>

<section>
<a name="shm-slab"></a>
<h3>1.3. Using the slab allocator</h3>
<p>Now that you have your new and shiny shm segment, how do you use it? The simplest way is to use another memory tool that nginx has at your disposal, namely the slab allocator. Nginx is nice enough to initialise the slab for you in every new shm segment, so you can either use it, or ignore the slab structures and overwrite them with your own data.</p>

<p>The interface consists of two functions:

<ul>
<li><code>void *ngx_slab_alloc(ngx_slab_pool_t *pool, size_t size);</code></li>
<li><code>void ngx_slab_free(ngx_slab_pool_t *pool, void *p);</code></li>
</ul>

The first argument is simply <code>(ngx_slab_pool_t *)shm_zone-&gt;shm.addr</code> and the other one is either the size of the block to allocate, or the pointer to the block to free. (trivia: not once is <code>ngx_slab_free</code> called in vanilla nginx code)</p>
</section>

<section>
<a name="shm-spinlocks"></a>
<h3>1.4. Spinlocks, atomic memory access</h3>

<p>Remember that shared memory is inherently dangerous because you can have multiple processes accessing it at the same time. The slab allocator has a per-segment lock (<code>shpool-&gt;mutex</code>) which is used to protect the segment against concurrent modifications.</p>

<p>You can also acquire and release the lock yourself, which is useful if you want to implement some more complicated operations on the segment, like searching or walking a tree. The two snippets below are essentially equivalent:

<code><pre>
/*
void *new_block;
ngx_slab_pool_t *shpool = (ngx_slab_pool_t *)shm_zone-&gt;shm.addr;
*/

new_block = ngx_slab_alloc(shpool, ngx_pagesize);
</pre></code>

<code><pre>
ngx_shmtx_lock(&amp;shpool-&gt;mutex);
new_block = ngx_slab_alloc_locked(shpool, ngx_pagesize);
ngx_shmtx_unlock(&amp;shpool-&gt;mutex);
</pre></code>

In fact, ngx_slab_alloc looks almost exactly like above.
</p>

<p>If you perform any operations which depend on no new allocations (or, more to the point, frees), protect them with the slab mutex. However, remember that nginx mutexes are implemented as spinlocks (non-sleeping), so while they are very fast in the uncontended case, they can easily eat 100% CPU when waiting. So don't do any long-running operations while holding the mutex (especially I/O, but you should avoid any system calls at all).</p>

<p> You can also use your own mutexes for more fine-grained locking, via the <code>ngx_mutex_init()</code>, <code>ngx_mutex_lock()</code> and <code>ngx_mutex_unlock()</code> functions.</p>

<p>As an alternative for locks, you can use atomic variables which are guaranteed to be read or written in an uninterruptible way (no worker process may see the value halfway as it's being written by another one).</p>

<p>Atomic variables are defined with the type <code>ngx_atomic_t</code> or <code>ngx_atomic_uint_t</code> (depending on signedness). They should have at least 32 bits. To simply read or unconditionally set an atomic variable, you don't need any special constructs:

<code><pre>
ngx_atomic_t i = an_atomic_var;
an_atomic_var = i + 5;
</pre></code>
</p>

<p>Note that anything can happen between the two lines; context switches, execution of code on other other CPUs, etc.</p>

<p>To atomically read and modify a variable, you have two functions (very platform-specific) with their interface declared in <code>src/os/unix/ngx_atomic.h</code>:

<ul>
<li><p><code>ngx_atomic_cmp_set(lock, old, new)</code></p>
<p>Atomically retrieves old value of <code>*lock</code> and stores <code>new</code> under the same
address. Returns 1 if <code>*lock</code> was equal to <code>old</code> before overwriting.</p>
</li>
<li><p><code>ngx_atomic_fetch_add(value, add)</code></p>

<p>Atomically adds <code>add</code> to <code>*value</code> and returns the old <code>*value</code>.
</p>
</li>
</ul>
</p>
</section>

<a name="shm-rbtrees"></a>
<h3>1.5. Using rbtrees</h3>

<p>OK, you have your data neatly allocated, protected with a suitable lock but you'd also like to organise it somehow. Again, nginx has a very nice structure just for this purpose - a red-black tree.</p>

<p>Highlights (API-wise):
<ul>
<li>requires an insertion callback, which inserts the element in the
tree (probably according to some predefined order) and then calls
<code>ngx_rbt_red(the_newly_added_node)</code> to rebalance the tree</li>
<li>requires all leaves to be set to a predefined sentinel object (not NULL)</li>
</ul></p>

<p>This chapter is about shared memory, not rbtrees so shoo! Go read the source for <a href="http://github.com/gnosek/nginx-upstream-fair/tree/master">upstream_fair</a> to see creating and walking an rbtree in action.</p>

<a name="subrequests"></a>
<h2>2. Подзапросы</h2>

<p>Подзапросы - одни из наиболее интересных составляющих Nginx. С подзапросами вы можете возвращать данные <em>URL отличного</em> от того который изначально запросил клиент. Некоторые среды веб-разработки называют это "внутренним перенаправлением". Но Nginx идет дальше: не только модули могут выполнять <em>множественные подзапросы</em> и объединять их выводы в один ответ, но и подзапросы могут выполнять их собственные под-подзапросы, и под-подзапросы могут инициировать под-под-подзапросы, и... суть вы поняли. Подзапросы могут обращаться к файлам на жестком диске, других обработчикам или к upstream-серверам; это не важно в концепции Nginx. Насколько я знаю ,только фильтры могут инициировать подзапросы.</p>

<a name="subrequests-redirect"></a>
<h3>2.1. Внутреннее перенаправление</h3>

<p>Если все что вы хотите это вернуть URL отличную от той которую вызвал пользователь, то рекомендую вам использовать функцию <code>ngx_http_internal_redirect</code>. Ее прообраз:

<code><pre>
ngx_int_t
ngx_http_internal_redirect(ngx_http_request_t *r, ngx_str_t *uri, ngx_str_t *args)
</pre></code>

<p>Где <code>r</code> это структура запроса, <code>uri</code> и <code>args</code> это новая URI. Примите к сведению что в качестве URI <em>должны</em> указываться локейшены уже определенные в nginx.conf; к примеру вы не сможете произвести перенаправление на произвольный домен. Обработчики должны возвращать возвращаемое значение фунцкии <code>ngx_http_internal_redirect</code>, т.е. перенапрявляющие обработчики обычно завершаются такой командой:</p>

<code><pre>
return ngx_http_internal_redirect(r, &amp;uri, &amp;args);
</pre></code>

<p>Внутренние перенаправления используютя в модуле "index" (который перенаправляет URL оканчивающиеся на / на index.html), а так же в сервисе Nginx X-Accel-Redirect.</p>

<a name="subrequests-single"></a>
<h3>2.2. Одиночный подзапрос</h3>

<p>Подзапросы наиболее полезны для вставки дополнительного контента <em>основанного на данных из первоначального ответа</em>. Subrequests are most useful for inserting additional content <em>based on data from the original response</em>. For example, the SSI (server-side include) module uses a filter to scan the contents of the returned document, and then replaces "include" directives with the contents of the specified URLs.</p>

<p>We'll start with a simpler example. We'll make a filter that treats the entire contents of a document as a URL to be retrieved, and then appends the new document to the URL itself. Remember that the URL must be a location in nginx.conf.</p>

<code><pre>
static ngx_int_t
ngx_http_append_uri_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    int                 rc; 
    ngx_str_t           uri;
    ngx_http_request_t *sr;

    /* First copy the document buffer into the URI string */
    uri.len = in-&gt;buf-&gt;last - in-&gt;buf-&gt;pos;
    uri.data = ngx_palloc(r-&gt;pool, uri.len);
    if (uri.data == NULL)
        return NGX_ERROR;
    ngx_memcpy(uri.data, in-&gt;-buf-&gt;pos, uri.len);

    /* Now return the original document (i.e. the URI) to the client */
    rc = ngx_http_next_body_filter(r, in);

    if (rc == NGX_ERROR)
        return rc;

    /* Finally issue the subrequest */
    return ngx_http_subrequest(r, &amp;uri, NULL /* args */, 
        NULL /* callback */, 0 /* flags */);
}
</pre></code>

<p>The prototype of <code>ngx_http_subrequest</code> is:</p>

<code><pre>
ngx_int_t ngx_http_subrequest(ngx_http_request_t *r,
    ngx_str_t *uri, ngx_str_t *args, ngx_http_request_t **psr, 
        ngx_http_post_subrequest_t *ps, ngx_uint_t flags)
</pre></code>

<p>Where: 
<ul>
    <li><code>*r</code> is the original request</li>
    <li><code>*uri</code> and <code>*args</code> refer to the sub-request</li>
    <li><code>**psr</code> is a <em>reference to a NULL pointer</em> that will point to the new (sub-)request structure</li>
    <li><code>*ps</code> is a callback for when the subrequest is finished. I've never used this, but see <a href="http://lxr.evanmiller.org/http/source/http/ngx_http_request.h#L303" class="source">http/ngx_http_request.h</a> for details.</li>
    <li><code>flags</code> can be a bitwise-OR'ed combination of:</li>
    <ul>
        <li><code>NGX_HTTP_ZERO_IN_URI</code>: the URI contains a character with ASCII code 0 (also known as '\0'), <em>or</em> contains "%00"</li>
        <li><code>NGX_HTTP_SUBREQUEST_IN_MEMORY</code>: store the result of the subrequest in a contiguous chunk of memory (usually not necessary)</li>
    </ul>
</ul>

<p>The results of the subrequest will be inserted where you expect. If you want to modify the results of the subrequest, you can use another filter (or the same one!). You can tell whether a filter is operating on the primary request or a subrequest with this test:</p>

<code><pre>
if (r == r-&gt;main) { 
    /* primary request */
} else {
    /* subrequest */
}
</pre></code>

<p>The simplest example of a module that issues a single subrequest is the <a href="http://lxr.evanmiller.org/http/source/http/modules/ngx_http_addition_filter_module.c" class="source">"addition" module</a>.</p>

<a name="subrequests-sequential"></a>
<h3>2.3. Sequential subrequests</h3>

<div class="aside">
    <em>Note, 8/13/2009: This section may be out of date due to changes in Nginx's subrequest processing introduced in Nginx 0.7.25. Follow at your own risk. -EM</em>
</div>

<p>You might think issuing multiple subrequests is as simple as:</p>

<code><pre>
int rc1, rc2, rc3;
rc1 = ngx_http_subrequest(r, uri1, ...);
rc2 = ngx_http_subrequest(r, uri2, ...);
rc3 = ngx_http_subrequest(r, uri3, ...);
</pre></code>

<p>You'd be wrong! Remember that Nginx is single-threaded. Subrequests might need to access the network, and if so, Nginx needs to return to its other work while it waits for a response. So we need to check the return value of <code>ngx_http_subrequest</code>, which can be one of:</p>
<ul>
    <li><code>NGX_OK</code>: the subrequest finished without touching the network</li>
    <li><code>NGX_DONE</code>: the client reset the network connection</li>
    <li><code>NGX_ERROR</code>: there was a server error of some sort</li>
    <li><code>NGX_AGAIN</code>: the subrequest requires network activity</li> 
</ul>

<p>If your subrequest returns <code>NGX_AGAIN</code>, your filter should also immediately return <code>NGX_AGAIN</code>. When that subrequest finishes, and the results have been sent to the client, Nginx is nice enough to call your filter again, from which you can issue the next subrequest (or do some work in between subrequests). It helps, of course, to keep track of your planned subrequests in a context struct. You should also take care to return errors immediately, too.</p>

<p>Let's make a simple example. Suppose our context struct contains an array of URIs, and the index of the next subrequest:</p>

<code><pre>
typedef struct {
    ngx_array_t  uris;
    int          i;
} my_ctx_t;
</pre></code>

<p>Then a filter that simply concatenates the contents of these URIs together might something look like:</p>

<code><pre>
static ngx_int_t
ngx_http_multiple_uris_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    my_ctx_t  *ctx;
    int rc = NGX_OK;
    ngx_http_request_t *sr;

    if (r != r-&gt;main) { /* subrequest */
        return ngx_http_next_body_filter(r, in);
    }

    ctx = ngx_http_get_module_ctx(r, my_module);
    if (ctx == NULL) {
        /* populate ctx and ctx-&gt;uris here */
    }
    while (rc == NGX_OK &amp;&amp; ctx-&gt;i &lt; ctx-&gt;uris.nelts) {
        rc = ngx_http_subrequest(r, &amp;((ngx_str_t *)ctx-&gt;uris.elts)[ctx-&gt;i++],
            NULL /* args */, &amp;sr, NULL /* cb */, 0 /* flags */);
    }

    return rc; /* NGX_OK/NGX_ERROR/NGX_DONE/NGX_AGAIN */
}
</pre></code>

<p>Let's think this code through. There might be more going on than you expect.</p>

<p>First, the filter is called on the original response. Based on this response we populate <code>ctx</code> and <code>ctx-&gt;uris</code>. Then we enter the while loop and call <code>ngx_http_subrequest</code> for the first time.</p>

<p>If <code>ngx_http_subrequest</code> returns NGX_OK then we move onto the next subrequest immediately. If it returns with NGX_AGAIN, we break out of the while loop and return NGX_AGAIN.</p>

<p>Suppose we've returned an NGX_AGAIN. The subrequest is pending some network activity, and Nginx has moved on to other things. But when that subrequest is finished, Nginx will call our filter at least two more times:</p>
<ol>
    <li>once with <code>r</code> set to the subrequest, and <code>in</code> set to buffers from the subrequest's response</li>
    <li>once with <code>r</code> set to the original request, and <code>in</code> set to NULL</li>
</ol>

<p>To distinguish these two cases, we must test whether <code>r == r-&gt;main</code>. In this example we call the next filter if we're filtering the subrequest. But if we're in the main request, we'll just pick up the while loop where we last left off. <code>in</code> will be set to NULL because there aren't actually any new buffers to process.</p>

<p>When the last subrequest finishes and all is well, we return NGX_OK.</p>

<p>This example is of course greatly simplified. You'll have to figure out how to populate <code>ctx-&gt;uris</code> on your own. But the example shows how simple it is to re-enter the subrequesting loop, and break out as soon as we get an error or <code>NGX_AGAIN</code>.</p>

<a name="subrequests-parallel"></a>
<h3>2.4. Parallel subrequests</h3>

<p>It's also possible to issue several subrequests at once without waiting for previous subrequests to finish. This technique is, in fact, too advanced even for <em>Emiller's Advanced Topics in Nginx Module Development</em>. See the <a href="http://lxr.evanmiller.org/http/source/http/modules/ngx_http_ssi_filter_module.c" class="source">SSI module</a> for an example.</p>

<a name="parsing"></a>
<h2>3. Parsing with Ragel</h2>

<p>If your module is dealing with any kind of input, be it an incoming HTTP header or a full-blown template language, you will need to write a parser. Parsing is one of those things that seems easy&mdash;how hard can it be to convert a string into a struct?&mdash;but there is definitely a right way to parse and a wrong way to parse. Unfortunately, Nginx sets a bad example by choosing (what I feel is) the wrong way.</p>

<p>What's wrong with Nginx's parsing code?</p>

<code><pre>
    /* Random snippet from Nginx parsing code */

   for (p = ctx-&gt;pos; p &lt; last; p++) {
        ch = *p;

        switch (state) {
            case ssi_tag_state:
                switch (ch) {
                    case '!':
                        /* do stuff */
                ...
</pre></code>

<p>Nginx does all of its parsing, whether of SSI includes, HTTP headers, or Nginx configuration files, using state machines. A <strong>state machine</strong>, you might recall from your college Theory of Computation class, reads a tape of characters, moves from state to state based on what it reads, and might perform some action based on what character it reads and what state it is in. So for example, if I wanted to parse positive decimal point numbers with a state machine, I might have a "reading stuff left of the period" state, a "just read a period" state, and a "reading stuff right of the period" state, and move among them as I read in each digit.</p>

<p>Unfortunately, state machine parsers are usually verbose, complex, hard to understand, and hard to modify. From a software development point of view, a better approach is to use a parser generator. A <strong>parser generator</strong> translates high-level, highly readable parsing rules into a low-level state machine. The <em>compiled</em> code from a parser generator is virtually the same as that of handwritten state machine, but <em>your</em> code is much easier to work with.</p>

<p>There are a number of parser generators available, each with their own special syntaxes, but I am going to focus on one parser generator in particular: Ragel. Ragel is a good choice because it was designed to work with buffered inputs. Given Nginx's buffer-chain architecture, there is a very good chance that you will parsing a buffered input, whether you really want to or not.</p>

<a name="parsing-install"></a>
<h3>3.1. Installing Ragel</h3>

<p>Use your system's package manager or else <a href="http://www.complang.org/ragel/">download Ragel from here</a>.</p>

<a name="parsing-call"></a>
<h3>3.2. Calling Ragel from Nginx</h3>

<p>It's a good idea to put your parser functions in a separate file from the rest of the module. You will then need to:</p>
<ul>
    <li>Create a header (<code>.h</code>) file for the parser</li>
    <li>Include the header from your module</li>
    <li>Create a Ragel (<code>.rl</code>) file</li>
    <li>Generate a C (<code>.c</code>) file from the Ragel file</li>
    <li>Include the C file in your module config</li>
</ul>

<p>The header file should just have prototype for parser functions, which you can include in your module via the usual <code>#include "my_module_parser.h"</code> directive. The real work is writing the Ragel file. We will work through a simple example. The official Ragel User Guide (<a href="http://www.complang.org/ragel/ragel-guide-6.5.pdf">PDF available here</a>) is fully 56 pages long and gives the programmer tremendous power, but we will just go through the parts of Ragel you really need for a simple parser.</p>

<p>Ragel files are C files interspersed with special Ragel commands and functions. Ragel commands are in blocks of code surrounded by <code>%%{</code> and <code>}%%</code>. The first two Ragel commands you will want in your parser are:</p>

<code><pre>
%%{
    machine my_parser;
    write data;
}%%
</pre></code>

<p>These two commands should appear after any pre-processor directives but before your parser function. The <code>machine</code> command gives a name to the state machine Ragel is about to build for you. The <code>write</code> command will create the state definitions that the state machine will use. Don't worry about these commands too much.</p>

<p>Next you can start writing your parser function as regular C. It can take any arguments you want and should return an <code>ngx_int_t</code> with <code>NGX_OK</code> upon success and <code>NGX_ERROR</code> upon failure. You should pass in, if not a pointer to the input you want to parse, then at least some kind of context struct that contains the input data.</p>

<p>Ragel will create a number of variables implicitly for you. Other variables you need to define yourself in order to use Ragel. At the top of your function, you need to declare:</p>
<ul>
    <li><code>u_char *p</code> - pointer to the beginning of the input</li>
    <li><code>u_char *pe</code> - pointer to the end of the input</li>
    <li><code>int cs</code> - an integer which stores the state machine's state</li>
</ul>

<p>Ragel will start its parsing wherever <code>p</code> points, and finish up as soon as it reaches <code>pe</code>. Therefore <code>p</code> and <code>pe</code> should both be pointers on a contiguous chunk of memory. Note that when Ragel is finished running on a particular input, you can save the value of <code>cs</code> (the machine state) and resume parsing on additional input buffers exactly where you left off. In this way Ragel works across multiple input buffers and fits beautifully into Nginx's event-driven architecture.</p>

<a name="parsing-grammar"></a>
<h3>3.3. Writing a grammar</h3>

<p>Next we want to write the Ragel grammar for our parser. A grammar is just a set of rules that specifies which kinds of input are allowed; a Ragel grammar is special because it allows us to perform actions as we scan each character. To take advantage of Ragel, you must learn the Ragel grammar syntax; it is not difficult, but it is not trivial, either.</p>

<p>Ragel grammars are defined by sets of rules. A rule has an arbitrary name on the left side of an equals sign and a specification on the right side, followed by a semicolon. The rule specification is a mixture of regular expressions and actions. We will get to actions in a minute.</p>

<p>The most important rule is called "main." All grammars must have a rule for main. The rule for main is special in that 1) the name is not arbitrary and 2) it uses <code>:=</code> instead of <code>=</code> to separate the name from the specification.</p>

<p>Let's start with a simple example: a parser for processing Range requests. This code is adapted from my <a href="http://wiki.nginx.org/NginxNgxZip">mod_zip</a> module, which also includes a more complicated parser for processing lists of files, if you are interested.</p>

<p>The "main" rule for our byte range parser is quite simple:</p>

<code><pre>
    main := "bytes=" byte_range_set;
</pre></code>

<p>That rule just says "the input should consist of the string <code>bytes=</code> followed by input which follows the rule called <code>byte_range_set</code>." So we need to define the rule <code>byte_range_set</code>:</p>

<code><pre>
    byte_range_set = byte_range_specs ( "," byte_range_specs )*;
</pre></code>

<p>That rule just says "<code>byte_range_set</code> consists of a <code>byte_range_specs</code> followed by zero or more commas each followed by a <code>byte_range_specs</code>." In other words, a <code>byte_range_set</code> is a comma-separated list of <code>byte_range_specs</code>'s. You might recognize the <code>*</code> as a Kleene star or from regular expressions.</p>

<p>Next we need to define the <code>byte_range_specs</code> rule:</p>

<code><pre>
    byte_range_specs = byte_range_spec &gt;new_range;
</pre></code>

<p>The <code>&gt;</code> character is special. It says that <code>new_range</code> is not the name of another rule, but the name of an <em>action</em>, and the action should be taken at the <em>beginning</em> of <em>this</em> rule, i.e. the beginning of <code>byte_range_specs</code>. The most important special characters are:</p>

<ul>
    <li><code>&gt;</code> - action should be taken at the beginning of this rule</li>
    <li><code>$</code> - action should be taken as each character is processed</li>
    <li><code>%</code> - action should be taken at the end of this rule</li>
</ul>

<p>There are others as well, which you can read about in the Ragel User Guide. These are enough to get you started without being too confusing.</p>

<p>Before we get into actions, let's finish defining our rules. In the rule for <code>byte_range_specs</code> (plural), we referred to a rule called <code>byte_range_spec</code> (singular). It is defined as:</p>

<code><pre>
    byte_range_spec = [0-9]+ $start_incr
                      "-"
                      [0-9]+ $end_incr;
</pre></code>

<p>This rule states "read one or more digits, executing the action <code>start_incr</code> for each, then read a dash, then read one or more digits, executing the action <code>end_incr</code> for each." Notice that no actions are taken at the beginning or end of <code>byte_range_spec</code>.</p>

<p>When you are actually writing a grammar, you should write the rules in reverse order of what I have here. Rules should refer only to other rules that have been previously defined. So "main" should always be the <em>last</em> rule in your grammar, not the first.</p>

<p>Our byte-range grammar is now finished; it's time to specify the actions.</p>

<a name="parsing-actions"></a>
<h3>3.4. Writing some actions</h3>

<p>Actions are chunks of C code which have access to a few special variables. The most important special variables are:</p>

<ul>
    <li><code>fc</code> - the current character being read</li>
    <li><code>fpc</code> - a pointer to the current character being read</li>
</ul>

<p><code>fc</code> is most useful for <code>$</code> actions, i.e. actions performed on each character of a string or regular expression. <code>fpc</code> is more useful for <code>&gt;</code> and <code>%</code> actions, that is, actions taken at the start or end of a rule.</p>

<p>To return to our byte-range example, here is the <code>new_range</code> action. It does not use any special variables.</p>

<code><pre>
    action new_range {
        if ((range = ngx_array_push(&amp;ctx-&gt;ranges)) == NULL) {
            return NGX_ERROR;
        }
        range-&gt;start = 0; range-&gt;end = 0;
    }
</pre></code>

<p><code>new_range</code> is surprisingly dull. It just allocated a new "range" struct on the "ranges" array stored in our context struct. Notice that as long as we include the right header files, Ragel actions have full access to the Nginx API.</p>

<p>Next we define the two remaining actions, <code>start_incr</code> and <code>end_incr</code>. These actions parse positive integers into the appropriate variables. As we read each digit of a number, we want to multiply the stored number by 10 and add the digit.  Here we take advantage of the special variable <code>fc</code> described above:</p>

<code><pre>
    action start_incr { range-&gt;start = range-&gt;start * 10 + (fc - '0'); }

    action end_incr { range-&gt;end = range-&gt;end * 10 + (fc - '0'); }
</pre></code>

<p>Note the old parsing trick of subtracting '0' to convert a character to an integer.</p>

<p>That's it for actions. We are almost finished with our parser.</p>

<a name="parsing-finale"></a>
<h3>3.5. Putting it all together</h3>

<p>Actions and the grammar should go inside a Ragel block inside your parser function, but after the declarations of <code>p</code>, <code>pe</code>, and <code>cs</code>. I.e., something like:</p>

<code><pre>
    ngx_int_t my_parser(/* some context, the request struct, etc. */) 
    {
        int cs;
        u_char *p = ctx-&gt;beginning_of_data;
        u_char *pe = ctx-&gt;end_of_data;

        %%{
            /* Actions */
            action some_action { /* C code goes here */ }
            action other_action { /* etc. */ }

            /* Grammar */
            my_rule = [0-9]+ "-" &gt;some_action;
            main := ( my_rule )* &gt;other_action;

            write init;
            write exec;
        }%%

        if (cs &lt; my_parser_first_final) {
            return NGX_ERROR;
        }

        return NGX_OK;
    }
</pre></code>

<p>We've added a few extra pieces here. The first are <code>write init</code> and <code>write exec</code>. These are commands to Ragel to insert the generated parser (written in C) right there.</p>

<p>The other extra bit is the comparison of <code>cs</code> to <code>my_parser_first_final</code>. Recall that <code>cs</code> stores the parser's state. This check ensures that the parser is in a valid state after it has finished processing input. If we are parsing across multiple input buffers, then instead of this check we will store <code>cs</code> somewhere and retrieve it when we want to continue parsing.</p>

<p>Finally, we are ready to generate the actual parser. The code we've written so far should be in a Ragel (<code>.rl</code>) file; when we're ready to compile, we just run the command:</p>

<code><pre>
    ragel my_parser.rl
</pre></code>

<p>This command will produce a file called "my_parser.c". To ensure that it is compiled by Nginx, you then need to add a line to your module's "config" file, like this:</p>

<code><pre>
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/my_parser.c"
</pre></code>

<p>Once you get the hang of parsing with Ragel, you will wonder how you ever did without it. You will actually <em>want</em> to write parsers in your Nginx modules. Ragel opens up a whole new set of possible modules to the imaginative developer.</p>

<a name="todo"></a>
<h2>4. TODO: Advanced Topics Not Yet Covered Here</h2>

<p>Topics not yet covered in this guide:</p>
<ul>
    <li>Parallel subrequests</li>
    <li>Built-in data structures (red-black trees, arrays, hash tables...)</li>
    <li>Access control modules</li>
</ul>

<hr>

<p><a href="/posts/2016/01/12/emillers-nginx-modules-guide-ru.html">Back to Emiller's Guide to Nginx Module Development</a></p>

<hr>

<p><a href="/">Back to home page</a> &ndash; <a href="http://twitter.com/EvMill">Follow on Twitter</a></p>


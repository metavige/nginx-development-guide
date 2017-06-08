NGINX開發指南
===========

* [譯者序](#譯者序)
* [簡介](#簡介)
    * [代碼結構](#代碼結構)
    * [頭文件](#頭文件)
    * [整數](#整數)
    * [常用返回值](#常用返回值)
    * [錯誤處理](#錯誤處理)
* [字串](#字串)
    * [概述](#概述)
    * [格式化](#格式化)
    * [數值轉換](#數值轉換)
    * [正規表達式](#正規表達式)
* [時間](#時間)
* [容器](#容器)
    * [陣列](#陣列)
    * [列表](#列表)
    * [隊列](#隊列)
    * [紅黑樹](#紅黑樹)
    * [哈希](#哈希)
* [內存管理](#內存管理)
    * [堆](#堆)
    * [內存池](#內存池)
    * [共享內存](#共享內存)
* [日誌](#日誌)
* [週期](#週期)
* [緩衝](#緩衝)
* [網絡](#網絡)
    * [連接](#連接)
* [事件](#事件)
    * [事件](#事件)
    * [I/O事件](#I/O事件)
    * [定時器事件](#定時器事件)
    * [延遲事件](#延遲事件)
    * [遍歷事件](#遍歷事件)
* [進程](#進程)
* [線程](#線程)
* [模塊](#模塊)
    * [添加新模塊](#添加新模塊)
    * [核心模塊](#核心模塊)
    * [配置指令](#配置指令)
* [HTTP](#HTTP)
    * [連接](#連接)
    * [請求](#請求)
    * [配置](#配置)
    * [階段](#階段)
    * [變量](#變量)
    * [複雜值](#複雜值)
    * [請求重定向](#請求重定向)
    * [子請求](#子請求)
    * [請求結束](#請求結束)
    * [請求體](#請求體)
    * [響應](#響應)
    * [響應體](#響應體)
    * [響應體過濾](#響應體過濾)
    * [構建過濾模塊](#構建過濾模塊)
    * [緩衝復用](#緩衝復用)
    * [負載均衡](#負載均衡)
    
譯者序
=====

本文檔是nginx官方文檔「Developer Guide」（[https://nginx.org/en/docs/dev/development_guide.html](https://nginx.org/en/docs/dev/development_guide.html)）的中文版本，由白山雲（[http://www.baishancloud.com](http://www.baishancloud.com/zh/)）NGINX開發團隊負責翻譯。官方文檔是HTML頁面發佈的，我們翻譯的時候轉成了Markdown，以方便編輯。同時也一並保留了英文的Markdown版本：[https://github.com/baishancloud/nginx-development-guide/blob/master/en.md](https://github.com/baishancloud/nginx-development-guide/blob/master/en.md)。希望此中文版文檔能為廣大的nginx以及開源愛好者提供入門指導，開發出優秀的nginx模塊，回饋社區。本文的官方版本並沒有全部完成，依然處於活躍更新的狀態，中文版本會持續保持跟蹤並持續更新。

簡介
===

代碼結構
-------
* auto — 編譯腳本
* src
    * core — 基礎數據結構和函數 — 字串，陣列，日誌，內存池等
    * event — 事件機制核心模塊
        * modules — 具體事件機制模塊：epoll，kqueue，select等
    * http — HTTP核心模塊和公共代碼
        * modules — 其他HTTP模塊
        * v2 — HTTP/2模塊
    * mail — 郵件協議模塊
    * os — 平台相關代碼
        * unix
        * win32
    * stream — 流模塊

頭文件
-----
每個nginx文件都應該在開頭包含如下兩個頭文件：

```
#include <ngx_config.h>
#include <ngx_core.h>
```

除此之外，HTTP相關的代碼還要包含：

```
#include <ngx_http.h>
```

郵件模塊的代碼應該包含：

```
#include <ngx_mail.h>
```

Stream模塊的代碼應該包含：

```
#include <ngx_stream.h>
```

整數
----
一般情況下，nginx代碼使用如下兩個整數類型：ngx_int_t和ngx_uint_t，分別用typedef定義成了intptr_t和uintptr_t。

常用返回值
--------
nginx中的大多數函數使用如下類型的返回值：

* NGX_OK — 處理成功
* NGX_ERROR — 處理失敗
* NGX_AGAIN — 處理未完成，函數需要被再次調用
* NGX_DECLINED — 處理被拒絕，例如相關功能在配置文件中被關閉。不要將此當成錯誤。
* NGX_BUSY — 資源不可用
* NGX_DONE — 處理完成或者在他處繼續處理。也可以作為處理成功使用。
* NGX_ABORT — 函數終止。也可以作為處理出錯的返回值。

錯誤處理
-------
為了獲取最近一次系統錯誤碼，nginx提供了ngx_errno宏。該宏被映射到了POSIX平台的errno變量上，而在Windows平台中，則變為對GetLastError()的函數調用。為了獲取最近一次socket錯誤碼，nginx提供了ngx_socket_errno宏。同樣，在POSIX平台上該宏被映射為errno變量，而在Windows環境中則是對WSAGetLastError()進行調用。考慮到對性能的影響，ngx_errno和ngx_socket_errno不應該被連續訪問。如果有連續、頻繁訪問的需要，則應該將錯誤碼的值存儲到類型為ngx_err_t的本地變量中，然後使用本地變量進行訪問。如果需要設置錯誤碼，可以使用ngx_set_errno(errno)和ngx_set_socket_errno(errno)這兩個宏。

ngx_errno和ngx_socket_errno變量可以在調用日誌相關函數ngx_log_error()和ngx_log_debugX()的時候使用，這樣具體的錯誤文本就會被添加到日誌輸出中。

一個使用ngx_errno的例子：

```
void
ngx_my_kill(ngx_pid_t pid, ngx_log_t *log, int signo)
{
    ngx_err_t  err;

    if (kill(pid, signo) == -1) {
        err = ngx_errno;

        ngx_log_error(NGX_LOG_ALERT, log, err, "kill(%P, %d) failed", pid, signo);

        if (err == NGX_ESRCH) {
            return 2;
        }

        return 1;
    }

    return 0;
}
```

字串
=====

概述
----
nginx使用無符號的char類型指針來表示C字串：u_char *。

nginx字串類型ngx_str_t的定義如下所示：

```
typedef struct {
    size_t      len;
    u_char     *data;
} ngx_str_t;
```

結構體成員len存放字串的長度，成員data指向字串本身數據。在ngx_str_t中存放的字串，對於超出len長度的部分可以是NULL結尾（'\0'——譯者注），也可以不是。在大多數情況是不以NULL結尾的。然而，在nginx的某些代碼中（例如解析配置的時候），ngx_str_t中的字串是以NULL結尾的，這種情況會使得字串比較變得更加簡單，也使得使用系統調用的時候更加容易。

nginx提供了一系列關於字串處理的函數。它們在src/core/ngx_string.h文件中定義。其中的一部分就是對C庫中字串函數的封裝：

* ngx_strcmp()
* ngx_strncmp()
* ngx_strstr()
* ngx_strlen()
* ngx_strchr()
* ngx_memcmp()
* ngx_memset()
* ngx_memcpy()
* ngx_memmove()

還有一些nginx特有的字串函數：

* ngx_memzero() 內存清0
* ngx_cpymem() 和ngx_memcpy()行為類似，不同的是該函數返回的是copy後的最終目的地址，這在需要連續拼接多個字串的場景下很方便。
* ngx_movemem() 和ngx_memmove()的行為類似，不同的是該函數返回的是move後的最終目的地址。
* ngx_strlchr() 在字串中查找一個特定字符，字串由兩個指針界定。

最後是一些大小寫轉換和字串比較的函數：

* ngx_tolower()
* ngx_toupper()
* ngx_strlow()
* ngx_strcasecmp()
* ngx_strncasecmp()

格式化
-----
nginx提供了一些格式化字串的函數。以下這些函數支持nginx特有的類型：

* ngx_sprintf(buf, fmt, ...)
* ngx_snprintf(buf, max, fmt, ...)
* ngx_slpintf(buf, last, fmt, ...)
* ngx_vslprint(buf, last, fmt, args)
* ngx_vsnprint(buf, max, fmt, args)

這些函數支持的全部格式化選項定義在src/core/ngx_string.c文件中，以下是其中的一部分：

```
%O — off_t
%T — time_t
%z — size_t
%i — ngx_int_t
%p — void *
%V — ngx_str_t *
%s — u_char * (null-terminated)
%*s — size_t + u_char *
```

'u'修飾符將類型指明為無符號，'X'和'x'則將輸出轉換為16進制。

例如：

```
u_char     buf[NGX_INT_T_LEN];
size_t     len;
ngx_int_t  n;

/* set n here */

len = ngx_sprintf(buf, "%ui", n) — buf;
```


數值轉換
-------
nginx實現了若干用於數值轉換的函數：

* ngx_atoi(line, n) — 將一個指定長度的字串轉換為一個正整數，類型為ngx_int_t。出錯返回NGX_ERROR。
* ngx_atosz(line, n) — 同上，轉換類型為ssize_t
* ngx_atoof(line, n) — 同上，轉換類型為off_t
* ngx_atotm(line, n) — 同上，轉換類型為time_t
* ngx_atofp(line, n, point) — 將一個固定長度的定點小數字串轉換為ngx_int_t類型的正整數。轉換結果會左移point指定的10進制位數。字串中的定點小數不能含有多過point參數指定的小數位。出錯返回NGX_ERROR。舉例：ngx_atofp("10.5", 4, 2) 返回1050
* ngx_hextoi(line, n) — 將表示16進制正整數的字串轉換為ngx_int_t類型的整數。出錯返回NGX_ERROR。

正規表達式
--------
nginx中的正規表達式接口是對PCRE庫的封裝。相關的頭文件是src/core/ngx_regex.h。

要使用正規表達式進行字串匹配，首先需要對正規表達式進行編譯，這通常是在配置解析階段處理的。需要注意的是，因為PCRE的支持是可選的，因此所有使用正則相關接口的代碼都需要用NGX_PCRE括起來：

```
#if (NGX_PCRE)
ngx_regex_t          *re;
ngx_regex_compile_t   rc;

u_char                errstr[NGX_MAX_CONF_ERRSTR];

ngx_str_t  value = ngx_string("message (\\d\\d\\d).*Codeword is '(?<cw>\\w+)'");

ngx_memzero(&rc, sizeof(ngx_regex_compile_t));

rc.pattern = value;
rc.pool = cf->pool;
rc.err.len = NGX_MAX_CONF_ERRSTR;
rc.err.data = errstr;
/* rc.options are passed as is to pcre_compile() */

if (ngx_regex_compile(&rc) != NGX_OK) {
    ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "%V", &rc.err);
    return NGX_CONF_ERROR;
}

re = rc.regex;
#endif
```

編譯成功之後，結構體ngx_regex_compile_t的captures和named_captures成員分別會被填上正規表達式中全部以及命名捕獲的數量。

然後，編譯過的正規表達式就可以用來進行字串匹配：

```
ngx_int_t  n;
int        captures[(1 + rc.captures) * 3];

ngx_str_t input = ngx_string("This is message 123. Codeword is 'foobar'.");

n = ngx_regex_exec(re, &input, captures, (1 + rc.captures) * 3);
if (n >= 0) {
    /* string matches expression */

} else if (n == NGX_REGEX_NO_MATCHED) {
    /* no match was found */

} else {
    /* some error */
    ngx_log_error(NGX_LOG_ALERT, log, 0, ngx_regex_exec_n " failed: %i", n);
}
```

ngx_regex_exec()的參數有：編譯了的正規表達式re，待匹配的字串s，可選的用於存放發現的捕獲和其大小的整數陣列。捕獲陣列的大小必須是3的倍數，這是PCRE庫的API要求的。在上面例子中，該陣列的大小是通過總捕獲數加上字串自身來計算得出的。

現在，如果成功匹配，則可以對捕獲進行訪問：

```
u_char     *p;
size_t      size;
ngx_str_t   name, value;

/* all captures */
for (i = 0; i < n * 2; i += 2) {
    value.data = input.data + captures[i];
    value.len = captures[i + 1] — captures[i];
}

/* accessing named captures */

size = rc.name_size;
p = rc.names;

for (i = 0; i < rc.named_captures; i++, p += size) {

    /* capture name */
    name.data = &p[2];
    name.len = ngx_strlen(name.data);

    n = 2 * ((p[0] << 8) + p[1]);

    /* captured value */
    value.data = &input.data[captures[n]];
    value.len = captures[n + 1] — captures[n];
}
```

ngx_regex_exec_array()函數接受ngx_regex_elt_t元素的陣列（其實就是多個編譯好的正規表達式以及對應的名字），一個待匹配字串以及一個log。該函數會對待匹配字串逐一應用陣列中的正規表達式，直到匹配成功或者無一匹配。存在成功的匹配則返回NGX_OK，否則返回NGX_DECLINED，出錯返回NGX_ERROR。

時間
====

結構體 ngx_time_t 將GMT格式的時間表示分割成秒和毫秒：

```
typedef struct {
    time_t      sec;
    ngx_uint_t  msec;
    ngx_int_t   gmtoff;
} ngx_time_t;
```

ngx_tm_t 是 struct tm 的一個別名，用在 UNIX 平台和Windows上的SYSTEMTIME。

為了獲取當前時間，通常只需要訪問一個可用的全局變量，表示所需格式的緩存時間值。ngx_current_msec 變量保存著自Epoch以來的毫秒數，並截成ngx_msec_t。

以下是可用的字串表示：

* ngx_cached_err_log_time — 用在 error log: "1970/09/28 12:00:00"
* ngx_cached_http_log_time — 用在 HTTP access log: "28/Sep/1970:12:00:00 +0600"
* ngx_cached_syslog_time — 用在 syslog: "Sep 28 12:00:00"
* ngx_cached_http_time — 用在 HTTP headers: "Mon, 28 Sep 1970 06:00:00 GMT"
* ngx_cached_http_log_iso8601 — ISO 8601 標準格式: "1970-09-28T12:00:00+06:00"

宏 ngx_time() 和 ngx_timeofday() 返回當前時間的秒，是訪問緩存時間值的首選方式。

為了明確獲取時間，可以使用ngx_gettimeofday()，它會更新參數（指向struct timeval）。當nginx從系統調用回到事件循環體時，時間總是會更新。如果想立即更新時間，調用 ngx_time_update() 或 ngx_time_sigsafe_up date() （如果在信號處理上下文需要用到）。

以下函數將 time_t 轉換成可分解的時間表示形式，對於libc前綴的那些，可以使用 ngx_tm_t 或者 struct tm。

* ngx_gmtime(), ngx_libc_gmtime() — 結果時間是 UTC
* ngx_localtime(), ngx_libc_localtime() — 結果時間是相對時區

ngx_http_time(buf, time) 返回用於適合 HTTP headers（比如 "Mon, 28 Sep 1970 06:00:00 GMT"）的字串表示。另一種可能轉變通過 ngx_http_cookie_time(buf, time) 提供，用於生成適合HTTP cookies ("Thu, 3 1-Dec-37 23:55:55 GMT") 的格式。

容器
====

陣列
----
表示nginx陣列（array）的結構體ngx_array_t定義如下：

```
typedef struct {
    void        *elts;
    ngx_uint_t   nelts;
    size_t       size;
    ngx_uint_t   nalloc;
    ngx_pool_t  *pool;
} ngx_array_t;
```

陣列的元素可以通過elts成員獲取。元素的個數存放在nelts成員里。size成員記錄單個元素的大小，size成員是在陣列初始化的時候設置的。

陣列可以使用調用ngx_array_create(pool, n, size)來創建，其所需內存在提供的pool中。一個已經分配過內存的陣列對象，可以調用ngx_array_init(array, pool, n, size)進行初始化。


```
ngx_array_t  *a, b;

/* create an array of strings with preallocated memory for 10 elements */
a = ngx_array_create(pool, 10, sizeof(ngx_str_t));

/* initialize string array for 10 elements */
ngx_array_init(&b, pool, 10, sizeof(ngx_str_t));
```

使用下面的函數向陣列添加元素：

* ngx_array_push(a) 向陣列末尾添加一個元素並返回其指針
* ngx_array_push_n(a, n) 向陣列末尾添加n個元素並返回指向其中第一個元素的指針

如果現有內存無法滿足新元素的需要，陣列會分配新的內存並將現有元素複製過去。新分配的內存一般是原有內存的2倍大。

```
s = ngx_array_push(a);
ss = ngx_array_push_n(&b, 3);
```

列表
----
nginx中的列表（List）由一系列的陣列組成，並為可能插入大量item進行了優化。列表類型定義如下：

```
typedef struct {
    ngx_list_part_t  *last;
    ngx_list_part_t   part;
    size_t            size;
    ngx_uint_t        nalloc;
    ngx_pool_t       *pool;
} ngx_list_t;
```

實際的item存放在列表部件結構中，定義如下：

```
typedef struct ngx_list_part_s  ngx_list_part_t;

struct ngx_list_part_s {
    void             *elts;
    ngx_uint_t        nelts;
    ngx_list_part_t  *next;
};
```

使用之前，列表必須通過ngx_list_init(list, pool, n, size)初始化，或者通過ngx_list_create(pool, n, size)創建。兩個方式都需要指定單一條目的大小以及每個列表部件中item的數量。ngx_list_push(list)函數用來向列表添加一個item。遍歷item是通過直接訪問列表成員實現的，參考以下示例：

```
ngx_str_t        *v;
ngx_uint_t        i;
ngx_list_t       *list;
ngx_list_part_t  *part;

list = ngx_list_create(pool, 100, sizeof(ngx_str_t));
if (list == NULL) { /* error */ }

/* add items to the list */

v = ngx_list_push(list);
if (v == NULL) { /* error */ }
ngx_str_set(v, "foo");

v = ngx_list_push(list);
if (v == NULL) { /* error */ }
ngx_str_set(v, "bar");

/* iterate over the list */

part = &list->part;
v = part->elts;

for (i = 0; /* void */; i++) {

    if (i >= part->nelts) {
        if (part->next == NULL) {
            break;
        }

        part = part->next;
        v = part->elts;
        i = 0;
    }

    ngx_do_smth(&v[i]);
}
```

nginx中列表的主要用途是處理HTTP中輸入和輸出的頭部。

列表不支持刪除item。然而，如果需要的話，可以將item標識成missing而不是真正的刪除他們。例如，HTTP的輸出頭部——以ngx_table_elt_t對象存儲——可以通過將ngx_table_elt_t結構的hash成員設置成0來將其標識為missing。這樣一來，該HTTP頭部就不會被遍歷到。

隊列
----

nginx里的隊列是一個雙向鏈表，每個節點定義如下：

```
typedef struct ngx_queue_s  ngx_queue_t;

struct ngx_queue_s {
    ngx_queue_t  *prev;
    ngx_queue_t  *next;
};
```

頭部隊列節點沒有連接任何數據。使用之前，列表頭部要先調用 ngx_queue_init(q) 以初始化。隊列支持如下操作：

* ngx_queue_insert_head(h, x), ngx_queue_insert_tail(h, x) — 插入新節點
* ngx_queue_remove(x) — 刪除隊列節點
* ngx_queue_split(h, q, n) — 從a節點切割，隊列尾部起將變成新的獨立的隊列
* ngx_queue_add(h, n) — 將隊列n加到隊列h
* ngx_queue_head(h), ngx_queue_last(h) — 返回首或尾隊列節點
* ngx_queue_sentinel(h) - 返回隊列哨兵用來結束迭代
* ngx_queue_data(q, type, link) — 返回指向queue的data字段的起始地址，根據它的queue字段的偏移量

例子：

```
typedef struct {
    ngx_str_t    value;
    ngx_queue_t  queue;
} ngx_foo_t;

ngx_foo_t    *f;
ngx_queue_t   values;

ngx_queue_init(&values);

f = ngx_palloc(pool, sizeof(ngx_foo_t));
if (f == NULL) { /* error */ }
ngx_str_set(&f->value, "foo");

ngx_queue_insert_tail(&values, f);

/* insert more nodes here */

for (q = ngx_queue_head(&values);
     q != ngx_queue_sentinel(&values);
     q = ngx_queue_next(q))
{
    f = ngx_queue_data(q, ngx_foo_t, queue);

    ngx_do_smth(&f->value);
}
```

紅黑樹
------

頭文件 src/core/ngx_rbtree.h 提供了訪問紅黑樹的定義。

```
typedef struct {
    ngx_rbtree_t       rbtree;
    ngx_rbtree_node_t  sentinel;

    /* custom per-tree data here */
} my_tree_t;

typedef struct {
    ngx_rbtree_node_t  rbnode;

    /* custom per-node data */
    foo_t              val;
} my_node_t;
```

為了處理整個樹，需要兩個節點：root 和 sentinel。通常他們被添加到某些自定義的結構中，這樣就能將數據組織到樹中，其中包含指向數據的邊接。

初始化樹：

```
my_tree_t  root;

ngx_rbtree_init(&root.rbtree, &root.sentinel, insert_value_function);
```

The insert_value_function is a function that is responsible for traversing the tree and inserting new values into correct place. For example, the ngx_str_rbtree_insert_value functions is designed to deal with ngx_str_t type.

```
void ngx_str_rbtree_insert_value(ngx_rbtree_node_t *temp,
                                 ngx_rbtree_node_t *node,
                                 ngx_rbtree_node_t *sentinel)
```

第一個參數是樹中插入的節點，第二個是新創建的用來添加的節點，最後一個是樹的sentinel。

遍歷非常簡單明瞭，用下面的輪詢函數模式作為演示。

```
my_node_t *
my_rbtree_lookup(ngx_rbtree_t *rbtree, foo_t *val, uint32_t hash)
{
    ngx_int_t           rc;
    my_node_t          *n;
    ngx_rbtree_node_t  *node, *sentinel;

    node = rbtree->root;
    sentinel = rbtree->sentinel;

    while (node != sentinel) {

        n = (my_node_t *) node;

        if (hash != node->key) {
            node = (hash < node->key) ? node->left : node->right;
            continue;
        }

        rc = compare(val, node->val);

        if (rc < 0) {
            node = node->left;
            continue;
        }

        if (rc > 0) {
            node = node->right;
            continue;
        }

        return n;
    }

    return NULL;
}
```

compare() 是一個返回較小，相等或較大的經典函數。為了更快的查找，並且避免比較太大的對象，整型的hash字段就派上用場了。

為了添加節點到樹，需要分配新節點，初始化它，然後調用 ngx_rbtree_insert()：

```
    my_node_t          *my_node;
    ngx_rbtree_node_t  *node;

    my_node = ngx_palloc(...);
    init_custom_data(&my_node->val);

    node = &my_node->rbnode;
    node->key = create_key(my_node->val);

    ngx_rbtree_insert(&root->rbtree, node);
```

to remove a node:

```
ngx_rbtree_delete(&root->rbtree, node);
```

哈希
----

唏希表定義在 src/core/ngx_hash.h，支持精確和通配符匹配。後者需要額外的處理，放在下面的章節專門描述。

初始化哈希時，我們需要提前知道元素的個數，以便nginx能更好的優化哈希表。max_size 和 bucket_size 這兩參數需要配置。細節詳見官方提供的文檔。通常這兩參數會做成用戶可配置的。哈希初始化的設置放在ngx_hash_init_t類型的存儲中。而哈希表本身的類型是 ngx_hash_t。

```
ngx_hash_t       foo_hash;
ngx_hash_init_t  hash;

hash.hash = &foo_hash;
hash.key = ngx_hash_key;
hash.max_size = 512;
hash.bucket_size = ngx_align(64, ngx_cacheline_size);
hash.name = "foo_hash";
hash.pool = cf->pool;
hash.temp_pool = cf->temp_pool;
```

key是一個指向能根據字串創建整型的函數的指針。nginx提供了兩個通用的函數：ngx_hash_key(data, len) 和 ngx_hash_key_lc(data, len)。後者將字串轉為小寫，這需要這個字串是可寫的。如果不想這樣，NGX_HASH_READONLY_KEY 標記可以傳給這個函數，然後初始化陣列鍵（見下文）。

哈希keys保存在ngx_hash_keys_arrays_t里，然後通過 ngx_hash_keys_array_init(arr, type) 初始化。

```
ngx_hash_keys_arrays_t  foo_keys;

foo_keys.pool = cf->pool;
foo_keys.temp_pool = cf->temp_pool;

ngx_hash_keys_array_init(&foo_keys, NGX_HASH_SMALL);
```

第二個參數可以是NGX_HASH_SMALL或者NGX_HASH_LARGE，用於控制哈希表的預分配。如果你想hash包含更多的無素，請用NGX_HASH_LARGE。

ngx_hash_add_key(keys_array, key, value, flags) 函數用於將key添加到hash keys array：

```
ngx_str_t k1 = ngx_string("key1");
ngx_str_t k2 = ngx_string("key2");

ngx_hash_add_key(&foo_keys, &k1, &my_data_ptr_1, NGX_HASH_READONLY_KEY);
ngx_hash_add_key(&foo_keys, &k2, &my_data_ptr_2, NGX_HASH_READONLY_KEY);
```

現在就可能通過調用 ngx_hash_init(hinit, key_names, nelts) 來完成hash表的創建：

```
ngx_hash_init(&hash, foo_keys.keys.elts, foo_keys.keys.nelts);
```

這樣是有可能錯誤的，如果max_size或者bucket_size不足夠大的話。當hash創建了之後， ngx_hash_find(hash, key, name, len) 函數可用來查找無素：

```
my_data_t   *data;
ngx_uint_t   key;

key = ngx_hash_key(k1.data, k1.len);

data = ngx_hash_find(&foo_hash, key, k1.data, k1.len);
if (data == NULL) {
    /* key not found */
}
```

通配符匹配
----------

為了創建能運行通配符的hash，需要用 ngx_hash_combined_t 類型。它包含了上面提到的hash類型，還有兩個額外的keys arrays：dns_wc_head 和 dns_wc_tail。它的基本的初始化類似於普通hash。

```
ngx_hash_init_t      hash
ngx_hash_combined_t  foo_hash;

hash.hash = &foo_hash.hash;
hash.key = ...;
```

可以使用 NGX_HASH_WILDCARD_KEY 標記來添加通配符的key。

```
/* k1 = ".example.org"; */
/* k2 = "foo.*";        */
ngx_hash_add_key(&foo_keys, &k1, &data1, NGX_HASH_WILDCARD_KEY);
ngx_hash_add_key(&foo_keys, &k2, &data2, NGX_HASH_WILDCARD_KEY);
```

這個函數重新組織通配符和添加keys到對應的陣列。詳細用法和匹配算法參考map模塊。

根據添加keys的內容，你可能需要初始化三個keys arrays：一個用於前面提到的精確陣列，另外兩個用於從頭或尾的模糊匹配：

```
if (foo_keys.dns_wc_head.nelts) {

    ngx_qsort(foo_keys.dns_wc_head.elts,
              (size_t) foo_keys.dns_wc_head.nelts,
              sizeof(ngx_hash_key_t),
              cmp_dns_wildcards);

    hash.hash = NULL;
    hash.temp_pool = pool;

    if (ngx_hash_wildcard_init(&hash, foo_keys.dns_wc_head.elts,
                               foo_keys.dns_wc_head.nelts)
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    foo_hash.wc_head = (ngx_hash_wildcard_t *) hash.hash;
}
```

keys 陣列需要先排序，然後初始化後的結果必須添加到合併hash。dns_wc_tail 也是類似的操作。

查找合併hash通過 ngx_hash_find_combined(chash, key, name, len)：

```
/* key = "bar.example.org"; — will match ".example.org" */
/* key = "foo.example.com"; — will match "foo.*"        */

hkey = ngx_hash_key(key.data, key.len);
res = ngx_hash_find_combined(&foo_hash, hkey, key.data, key.len);
```

內存管理
========

堆
--

nginx提供以下的函數用於從系統堆分配內存：

* ngx_alloc(size, log) — 從系統堆分配內存。這個封裝了malloc()，並且帶有log。分配錯誤和調試信息都會記錄到log。
* ngx_calloc(size, log) — 和 ngx_alloc() 一樣，但是將分配後的內存填充為0。
* ngx_memalign(alignment, size, log) — 從系統堆分配可對齊的內存。如果平台提供了posix_memalign()，就用它做為封裝。否則返回調用傳遞最大對齊值參數的ngx_alloc()。
* ngx_free(p) — 釋放內存。這是free()的封裝。

內存池
------

大部份nginx分配使用內存池完成。在內存池分配的內存會在內存池銷毀時自動釋放。這樣就提供了更好的分配性能，並且控制內存變的更簡單。

內存池是通過在內部連續的內存塊分配對象的。當一個塊滿時，新的塊會被分配並且加入到該池的內存塊列表。當塊裝不了一個大的分配時，分配會交給系統，然後返回指向存到該內存池，以後以後釋放。

nginx 內存池類型為 ngx_pool_t。支持以下操作：

* ngx_create_pool(size, log) — 根據塊大小創建內存池。返回pool對象也是在裡面內存池里創建的。
* ngx_destroy_pool(pool) — 銷毀整個內存池，包括pool對象自己。
* ngx_palloc(pool, size) — 從內存池分配對齊的內存。
* ngx_pcalloc(pool, size) — 從內存池分配對齊的內存並且置為0。
* ngx_pnalloc(pool, size) — 從內存池分配沒對齊的內存。大部份用於分配字串。
* ngx_pfree(pool, p) — 釋放前面內存池中分配的內存。只對那些由系統分配的內存才會釋放。

```
u_char      *p;
ngx_str_t   *s;
ngx_pool_t  *pool;

pool = ngx_create_pool(1024, log);
if (pool == NULL) { /* error */ }

s = ngx_palloc(pool, sizeof(ngx_str_t));
if (s == NULL) { /* error */ }
ngx_str_set(s, "foo");

p = ngx_pnalloc(pool, 3);
if (p == NULL) { /* error */ }
ngx_memcpy(p, "foo", 3);
```

因為鏈 ngx_chain_t 在nginx經常使用，所以nginx內存池提供了一種方式來復用它們。ngx_pool_t 的 chain 字段保留了原先已經分配的列表用來復用。 為了有效分配內存池中的chain，應當使用 ngx_alloc_chain_link(pool) 函數。該函數查找內存池中空閒的chain，只有當為空時才分配一個新的。使用ngx_free_chain(pool, cl) 可以回收chain。 

cleanup handler可以註冊在pool里。cleanup handler 是一個帶有參數的回調，在內存池銷毀時調用。內存池通常在特定的nginx對象（比如HTTP請求），並且在對象的生命週期結束時銷毀，以釋放對象自己。註冊內存池cleanup可以方便地釋放資源，關閉文件描述符，或者做最後的關聯在對象上的數據的調整。

通過調用ngx_pool_cleanup_add(pool, size)註冊pool cleanup，它將返回 ngx_pool_cleanup_t 類型的指針，調用者會設置它。size 參數用分配cleanup上下文的大小。

```
ngx_pool_cleanup_t  *cln;

cln = ngx_pool_cleanup_add(pool, 0);
if (cln == NULL) { /* error */ }

cln->handler = ngx_my_cleanup;
cln->data = "foo";

...

static void
ngx_my_cleanup(void *data)
{
    u_char  *msg = data;

    ngx_do_smth(msg);
}
```

共享內存
--------

nginx用共享內存在進程之間共享公共的數據。函數 ngx_shared_memory_add(cf, name, size, tag) 添加新的共享內存實體到cycle。該函數接收 name 和 zone的大小。每個共享內存必須有唯一的名稱。如果提供的名稱存在，並且tag值也匹配，則會復用舊的zone實體。tag不匹配會被認為錯誤。通常模塊地址會被當作tag的值，這樣在模塊里就能通過name來復用共享內存。

以下是 ngx_shm_zone_t 的字段：

* init — 初始化回調函數，在實際的共享內存映射後調用。
* data — data 上下文，傳遞給初始化回調函數。
* noreuse — 村記。禁止復用從舊的cycle里的共享內存。
* tag — 共享內存tag。
* shm — 類型為 ngx_shm_t 的特定平台對象，有以下幾個字段：
    * addr — 映射的共享內存地址，初始為NULL
    * size — 共享內存大小
    * name — 共享內存名稱
    * log — 共享內存log
    * exists — 標記。表示共享內存繼承自主進程 (Windows特定)

共享內存zone實體會在ngx_init_cycle()解析配置後在映射到實際的內存。對POSIX系統，mmap() 系統調用用來創建匿名共享映射。對Windows，使用CreateFileMapping()/MapViewOfFileEx()對。

nginx提供了 ngx_slab_pool_t 來分配共享內存。對每個zone，slab pool會自動創建用來分配內存。這個池在共享zone的開頭，並且通過表達式 (ngx_slab_pool_t *) shm_zone->shm.addr 訪問。共享內存的分配通過調用 ngx_slab_alloc(pool, size)/ngx_slab_c alloc(pool, size) 函數完成，內存通過調用 ngx_slab_free(pool, p) 釋放。

slab pool 將共享zone分成多個頁。每個頁被用於分配同樣大小的對象。大小推薦為2的次方，並且不小於8。其它值被四捨五入。對每個頁，bitmask被用來表示哪些塊是已經使用的和哪些是空閒的。對大小超過半頁（通常是2048字節），將按完整的頁大小分配。

為了保護數據不會併發訪問，需要有 ngx_slab_pool_t 的 mutex 字段。mutex 在分配和釋放內存里被使用。然後它也可以用來保護其它分配自共享內存的數據。調用 ngx_shmtx_lock(&shpool->mutex) 鎖住，調用 ngx_shmtx_unlock(&shpool->mutex) 解鎖。

```
ngx_str_t        name;
ngx_foo_ctx_t   *ctx;
ngx_shm_zone_t  *shm_zone;

ngx_str_set(&name, "foo");

/* allocate shared zone context */
ctx = ngx_pcalloc(cf->pool, sizeof(ngx_foo_ctx_t));
if (ctx == NULL) {
    /* error */
}

/* add an entry for 65k shared zone */
shm_zone = ngx_shared_memory_add(cf, &name, 65536, &ngx_foo_module);
if (shm_zone == NULL) {
    /* error */
}

/* register init callback and context */
shm_zone->init = ngx_foo_init_zone;
shm_zone->data = ctx;


...

static ngx_int_t
ngx_foo_init_zone(ngx_shm_zone_t *shm_zone, void *data)
{
    ngx_foo_ctx_t  *octx = data;

    size_t            len;
    ngx_foo_ctx_t    *ctx;
    ngx_slab_pool_t  *shpool;

    value = shm_zone->data;

    if (octx) {
        /* reusing a shared zone from old cycle */
        ctx->value = octx->value;
        return NGX_OK;
    }

    shpool = (ngx_slab_pool_t *) shm_zone->shm.addr;

    if (shm_zone->shm.exists) {
        /* initialize shared zone context in Windows nginx worker */
        ctx->value = shpool->data;
        return NGX_OK;
    }

    /* initialize shared zone */

    ctx->value = ngx_slab_alloc(shpool, sizeof(ngx_uint_t));
    if (ctx->value == NULL) {
        return NGX_ERROR;
    }

    shpool->data = ctx->value;

    return NGX_OK;
}
```

日誌
=======

nginx用ngx_log_t對象記錄日誌。nginx的日誌提供以下幾種方式：

* stderr — 記錄到標準錯誤輸出
* file — 記錄到文件
* syslog — 記錄到syslog
* memory — 記錄到內部內存用於開發的目的。這塊內存可以在debugger時訪問。

一個日誌實例可以是一個日誌對象鏈接，每個通過next連接起來。每個消息都被寫到所有的日誌對象。

每個日誌對象有錯誤級別，用於限制消息寫到它自己。以下是nginx提供的幾種錯誤級別：

* NGX_LOG_EMERG
* NGX_LOG_ALERT
* NGX_LOG_CRIT
* NGX_LOG_ERR
* NGX_LOG_WARN
* NGX_LOG_NOTICE
* NGX_LOG_INFO
* NGX_LOG_DEBUG

對於調試日誌，有以下幾種選項：

* NGX_LOG_DEBUG_CORE
* NGX_LOG_DEBUG_ALLOC
* NGX_LOG_DEBUG_MUTEX
* NGX_LOG_DEBUG_EVENT
* NGX_LOG_DEBUG_HTTP
* NGX_LOG_DEBUG_MAIL
* NGX_LOG_DEBUG_STREAM

通常而言，日誌是通過error_log指令創建的，並且在各個階段都有效，cycle, 配置解析, 客戶端連接和其它。

nginx提供以下的日誌宏：

* ngx_log_error(level, log, err, fmt, ...) — 記錄錯誤
* ngx_log_debug0(level, log, err, fmt), ngx_log_debug1(level, log, err, fmt, arg1) etc — 調試日誌，提供最多8個可格式化的參數。

一條日誌被存放於棧上大小為NGX_MAX_ERROR_STR（當前為2048字節）的緩衝區里。日誌消息的前綴由錯誤等級，進程PID，連接id（存儲於log->connection）以及系統錯誤文本組成。對於非調式日誌（non-debug），log->handler也會被調用以向日誌消息增加更多的具體信息。HTTP模塊將ngx_http_log_error()函數設置為log handler來記錄客戶端和服務器的IP地址，當前動作（存儲於log->action），客戶端的請求行以及server name等等。

例如：

```
/* specify what is currently done */
log->action = "sending mp4 to client」;

/* error and debug log */
ngx_log_error(NGX_LOG_INFO, c->log, 0, "client prematurely
              closed connection」);

ngx_log_debug2(NGX_LOG_DEBUG_HTTP, mp4->file.log, 0,
               "mp4 start:%ui, length:%ui」, mp4->start, mp4->length);
```

將輸出日誌：

```
2016/09/16 22:08:52 [info] 17445#0: *1 client prematurely closed connection while
sending mp4 to client, client: 127.0.0.1, server: , request: "GET /file.mp4 HTTP/1.1」
2016/09/16 23:28:33 [debug] 22140#0: *1 mp4 start:0, length:10000
```

週期
=====

cycle 對象保持了nginx的運行時上文，由指定的配置創建。cycle的類型是 ngx_cycle_t。在配置重新加載後，新的cycle將從新版的配置創建，而舊的cycle通常在新的成功創建之後刪除。目前活動的cycle保存在 ngx_cycle 這個全局變量並且繼承自新啓動的nginx進程。

cycle 是通過ngx_init_cycle()這個函數創建的。這個函數接收老的cycle作為參數。它用於定位配置並且盡可能多的繼承舊的cycle以達到平滑過度。當nginx啓動時，模擬的cycle被創建，然後被根據配置的正常cycle替換。

以下是cycle的一些字段：

* pool — cycle內存池。每個新的cycle都會創建一個內存池。
* log — cycle日誌。初始時這個log繼承自舊的cycle。當讀完配置後，它會指向new_log。
* new_log - cycle日誌。由配置創建。會根據最外層範圍的error_log指令的設置而變化。
* connections, connections_n — 每個工作進程有一個類型為 ngx_connection_t 的陣列 connections，由 event 模塊在進程初始化時創建。connections的數目由worker_connections指令指定。
* files, files_n — 將文件描述符映射到nginx連接的陣列。這個映射由event模塊使用，具有NGX_USE_FD_EVENT標記（當前是poll和devpoll）。
* conf_ctx — 模塊配置陣列。這些配置在讀nginx配置文件時創建和填充。
* modules, modules_n — 類型為 ngx_module_t 的模塊陣列，包括靜態和通過當前配置加載的動態模塊。
* listening — 類型為 ngx_listening_t 的監聽socket陣列。監聽對象通常由不同模塊的監聽指令通過調用ngx_create_listening()函數添加。nginx基於這些監聽對象創建監聽socket。
* paths - 類型為 ngx_path_t 的陣列。paths由那些想操作指定目錄的模塊通過調用ngx_add_path()添加。nginx讀完配置之後，如果這些目錄不存在，nginx會創建它們。些外，兩個handler會被加到每個path：
    * path loader — 只會在nginx啓動或配置加載60秒後執行一次。通常讀取目錄並將數據保存在共享內存里，這個handler由名為「nginx cache loader」的進程調用。
    * path manager — 定期執行。通常移走目錄中舊的文件並將變化重新反映到共享內存里。這個handler由名為「nginx cache manager」的進程調用。
* open_files — 類型為ngx_open_file_t的列表。一個open file對象通過調用ngx_conf_open_file()創建。nginx讀完配置後會根據open_files列表打開文件，並且將文件描述符保存在各自的open file對象的fd字段。這些文件會以追加模塊打開，並且如果不存在時創建。nginx的工作進程在收到reopen信號（通常是USR1）後會重新打開被打開。這種情況下fd會變成新的文件描述符。目前這些打開的文件被用於日誌。
* shared_memory — 共享內存zone的列表，通過調用ngx_shared_memory_add()函數添加。在所有nginx進程里，共享內存zone會映射到同樣的地址，以共享所有的數據，比如HTTP緩存的in-memory樹。

緩衝
======

nginx對 input/output 操作提供了類型為 ngx_buf_t 的buffer。它通常用於保存寫入到目的的或從源讀的數據。buffer可以將數據指向內存或文件。
 技術上來講同時指向這兩種也是可能的。緩衝區的內存是單獨創建的，並且不會關聯到 ngx_buf_t 這個結構體。

ngx_buf_t 結構體有以下字段：

* start, end — 內存塊的邊界，分配給這個緩衝區。
* pos, last — 內存緩衝區邊界，一般在 start .. end 以內。
* file_pos, file_last — 文件緩衝區邊界。相對文件開頭的偏移量。
* tag — 唯一值。用於區分buffer，由不同的模塊創建，通常是為了buffer復用。
* file — file對象。
* temporary — 臨時標記。意味著這個buffer指向可寫的內存。
* memory — 內存標記。表示這個buffer指向只讀的內存。
* in_file — 文件村記。表示該當前buffer指向文件的數據。
* flush — 清空標記。表示應該清空這個buffer之前的所有數據。
* recycled — 可回收標記。表示該buffer可以被回收，而且應該盡快的使用。
* sync — 同步標記。表示這個buffer不帶任何數據或特殊的像flush, last_buf這樣的。一般這樣的buffer在nginx里會被認為是錯誤的，這個標記允許略過錯誤檢查。
* last_buf — 標記。表示這個buffer是輸出的最後一個。
* last_in_chain — 標記。表示在(子)請求沒有更多有數據的buffer了。
* shadow — 指向另一個buffer，與當前的buffer有關。通常當前buffer使用這個shadow buffer的數據。一量當前buffer使用完了，這個shadow buffer也應該被標記為已使用。
* last_shadow — 標記。表示當前buffer是最後一個buffer，並且指向特殊的shadow buffer。
* temp_file — 標記。表示這個buffer是臨時文件。

輸入輸出 buffer 連接在一個鏈里。鏈是定義為下的一系列 ngx_chain_t 。

```
typedef struct ngx_chain_s  ngx_chain_t;

struct ngx_chain_s {
    ngx_buf_t    *buf;
    ngx_chain_t  *next;
};
```

每個鏈保存著它的buffer，並且指向下一個鏈。

使用buffer和chain例子：

```
ngx_chain_t *
ngx_get_my_chain(ngx_pool_t *pool)
{
    ngx_buf_t    *b;
    ngx_chain_t  *out, *cl, **ll;

    /* first buf */
    cl = ngx_alloc_chain_link(pool);
    if (cl == NULL) { /* error */ }

    b = ngx_calloc_buf(pool);
    if (b == NULL) { /* error */ }

    b->start = (u_char *) "foo";
    b->pos = b->start;
    b->end = b->start + 3;
    b->last = b->end;
    b->memory = 1; /* read-only memory */

    cl->buf = b;
    out = cl;
    ll = &cl->next;

    /* second buf */
    cl = ngx_alloc_chain_link(pool);
    if (cl == NULL) { /* error */ }

    b = ngx_create_temp_buf(pool, 3);
    if (b == NULL) { /* error */ }

    b->last = ngx_cpymem(b->last, "foo", 3);

    cl->buf = b;
    cl->next = NULL;
    *ll = cl;

    return out;
}
```

網絡
==========

連接
----------

連接結構體 ngx_connection_t 是socket描述符的封裝。有如下字段：

* fd — socket描述符
* data — 任意連接上下文。通常指向更高層次的對象，構建在連接的上面，比如HTTP請求或Stream會話。
* read, write — 連接的讀寫事件。
* recv, send, recv_chain, send_chain — 連接的I/O操作。
* pool — 連接池
* log — connection日誌
* sockaddr, socklen, addr_text — 客戶端的二進制和文格形式的地址。
* local_sockaddr, local_socklen — 本地二進制形式地址。初始時這些為空，通過函數 ngx_connection_local_sockaddr() 得到本地socket地址。
* proxy_protocol_addr, proxy_protocol_port - PROXY protocol 客戶端地址和端口，如果為連接開啓了 PROXY protocol。
* ssl — nginx 連接 SSL 上下文
* reusable — 可復用村記。
* close — 關閉標記。表示連接是可復用的，並且應該被關閉。

nginx connection可以透傳SSL層。這種情況下connection ssl字段指向一個ngx_ssl_connection_t結構體，保留著這個連接的SSL相關的數據，包括 SSL_CTX 和 SSL。處理函數 recv, send, recv_chain, send_chain 被設置成對應的SSL函數。

每個進程的connection數量被限制為 worker_connections 的值。所有的connection結構體會提前創建並且保存在cycle的connections這個字段裡。通過 ngx_get_connection(s, log) 獲得一個connection結構體。該函數接收socket描述符並且會在connection結構體里作封裝。

國為每個進程有connection數的限制，nginx提供了一個搶佔connection的方式。通過 ngx_reusable_connection(c, reusable) 允許或禁止connection的復用。調用 ngx_reusable_connection(c, 1) 設置reuse標記並且將connection加入 cycle 的 reusable_connections_queue。每當 ngx_get_connection() 發現 cycle 的 free_connections 無可用的 connection 時，它會調用 ngx_drain_connections() 以釋放一定數量的可復用connection。對每個這樣的 connection，關閉標記被設置並且讀handler被調用以便通過調用ngx_close_connection(c)釋放connection，然後將它設置為可復用。連接處於可復用狀態下，調用ngx_reusable_connection(c, 0)可以取消復用。舉個nginx里connection可復用的例子，在接收客戶端的數據之前，HTTP客戶端的connection會被標記為可復用。


事件
======

事件
----

事件對象 ngx_event_t 在nginx里提供了一種特定事件發生時能被通知的方式。

以下是 ngx_event_t 的一些字段：

* data — 任意的事件上下文，用於事件處理。通常指向 connection，使其綁定到事件。
* handler — 回調函數。當事件發生時調用。
* write — 寫標記。表示這是一個寫事件。用於區分讀寫事件。
* active — 活躍標記。表示該事件收到I/O通知後已經註冊，一般來自像 epoll, kqueue, poll 這樣的通知機制。
* ready — 就緒標記。表示這個事件接收到I/O通知。
* delayed — 延遲標記。意味著I/O由於限速而延遲。
* timer — 紅黑樹節點。用於添加進超時紅黑樹。
* timer_set — 定時器設置標記。意味著這個事件定時器被設置，但還未過期。
* timedout — 超時標記。意味著這個事件已經過期。
* eof — 讀結束標記。表示讀結束。
* pending_eof — 結束掛起標記。表示結束是在socket上掛起的，雖然可能還有一些數據可用。這個標記通過 epoll EPOLLRDHUP 事件 or kqueue EV_EOF 標記傳遞。
* error — 錯誤標記。意思當讀或寫時發生了錯誤。
* cancelable — 可取消標記。表示當nginx工作進程退出時，即使該事件沒過期也能被立即調用。它提供了一種方式用來完成特定動作，比如清空日誌文件。
* posted — 隊列加入標記。意味這個事件已經加入了隊列。
* queue — 隊列節點。用於加到隊列。

I/O事件
-------

每個通過調用ngx_get_connection()的 connection 有兩個事件：c->read 和 c->write。這兩事件用於接受可讀寫socket的通知。所有的這些事件都是邊緣觸發模式，意味著只有socket的狀態變化時它們才會觸發。舉個例子，假設只讀了部份數據，當有更多的數據到達時，nginx不會重新發讀通知。即使底層的I/O通知機制本質上是水平觸發的（poll, select等等），nginx將會把它們轉成邊緣觸發。為了將不同平台的事件通知機制統一起來，當處理I/O socket通知或任何I/O操作後，必須調用ngx_handle_read_event(rev, flags) and ngx_handle_write_event(wev, lowat) 這兩函數。通常這兩函數在讀或寫事件處理結束後調用一次。

定時器事件
----------

事件可以被設置以通知超時過期。ngx_add_timer(ev, timer) 函數設置事件的超時時間，ngx_del_timer(ev) 刪除前面設置的超時。當前為所有事件設置的超時都存放在一個全局的超時紅黑樹 ngx_event_timer_rbtree。這個樹key的類型是 ngx_msec_t，值是從1970年1月1日算起的過期時間。這個樹結構提供了快速的插入和刪除，以及訪問那些最小的超時。後者被nginx用於查找等待I/O事件的時間以及之後的過期事件。

已加事件
-------------

添加事件意味著它的handler會在某個時間點的事件遍歷時調用。加入事件對簡化代碼和防止棧溢出是一個好的實踐。加入的事件放在一個隊列里。宏 ngx_post_event(ev, q) 加入事件到 post queue，ngx_delete_posted_event(ev) 從它所加入的隊列中刪除事件。通常事件加到 ngx_posted_events 這個隊 列。 這個事件在稍後事件遍歷中被事件(在所有的I/O和定時器事件已經處理後)。 ngx_event_process_posted() 函數用來處理事件隊列。這個函數一直處理到列隊為空，這意味著在當前的事件遍歷過程中可以加更多的事件。

例子：

```
void
ngx_my_connection_read(ngx_connection_t *c)
{
    ngx_event_t  *rev;

    rev = c->read;

    ngx_add_timer(rev, 1000);

    rev->handler = ngx_my_read_handler;

    ngx_my_read(rev);
}


void
ngx_my_read_handler(ngx_event_t *rev)
{
    ssize_t            n;
    ngx_connection_t  *c;
    u_char             buf[256];

    if (rev->timedout) { /* timeout expired */ }

    c = rev->data;

    while (rev->ready) {
        n = c->recv(c, buf, sizeof(buf));

        if (n == NGX_AGAIN) {
            break;
        }

        if (n == NGX_ERROR) { /* error */ }

        /* process buf */
    }

    if (ngx_handle_read_event(rev, 0) != NGX_OK) { /* error */ }
}
```

遍歷事件
----------

所有做I/O處理的nginx進程都有一個事件遍歷。唯一沒有I/O的進程是master進程，因為它花大部份時間在sigsuspend()上面，以等待信號的到達。事件遍歷由 ngx_process_events_and_timers 函數實現。只要進程存在，這個函數就會一直重復的調用。它有以下幾個階段：

* 找出通過調用 ngx_event_find_timer() 的最小超時時間。該函數找到最左邊的定時器樹節點，並且返回該節點的到期毫秒數。
* 處理I/O事件。通過nginx配置選出對應的事件通知機制，然後處理。這個handler會一直待待至有I/O事件發生，或者最小的超時時間。對每個發生的讀寫事件，它的ready標記會被設置，它的handler會被調用。對Linux而言，通常會使用 ngx_epoll_process_events() 來調用 epoll_wait() 以等待I/O發生。
* 通過調用 ngx_event_expire_timers() 處理過期事件。這個定時器樹會從最左側的節點向右歷遍，直到找到沒有過期到期的超時。對每個超時的節點，timedout 標記會被設置，timer_set 會被重量置，並且事件handler會被調用。
* 通過調用 ngx_event_process_posted() 處理已加事件。這個函數一直重復刪除和處理隊列里的第一個無素，直到隊列為空。

所有這些nginx進程也處理信號。信號handler只是設置了在 ngx_process_events_and_timers() 調用之後的全局變量。

進程
=========

nginx有好幾種進程類型。當前進程的類型保存在ngx_process這個全局變量。

* NGX_PROCESS_MASTER — 主進程運行ngx_master_process_cycle()這個函數。主進程不能有任何的I/O，並且只對信號響應。它讀取配置，創建cycle，啓動和控制子進程。

* NGX_PROCESS_WORKER — 工作進程運行ngx_worker_process_cycle()函數。工作進程由子進程創建，處理客戶端連接。他們同樣也響應來自主進程的信號。

* NGX_PROCESS_SINGLE — 單進程只存在於master_process模式模式的情況下。生命週期函數是ngx_single_process_cycle()。這個進程創建生命週期並且處理客戶端連接。

* NGX_PROCESS_HELPER — 目前只有兩種help進程：cache manager 和 cache loader. 它們共用同樣的生命週期函數ngx_cache_manager_process_cycle()。

所有的nginx處理如下信號：

* NGX_SHUTDOWN_SIGNAL (SIGQUIT) — 優雅結束。收到此信號後主進程發送 shutdown 信號給所有的子進程。當沒有任何子進程時，主進程釋放生命週期內存池然後結束。工作進程收到此信號後，關閉所有的監聽端口然後一直等到超時樹為空，最後釋放生命週期內存池並且結束。cache 管理進程收到這個信號後立馬退出。收到信號後 ngx_quit 設置為0，然後在處理完成後立馬重置。ngx_exiting 在工作進程處理退出狀態時設置為1。

* NGX_TERMINATE_SIGNAL (SIGTERM) - 終止。. 收到此信號後主進程發送 terminate 信號給所有的子進程。如果子進程1秒內沒結束，它們會通過SIGKILL 信號被殺掉。當沒有任何子進程時，主進程釋放生命週期內存池然後結束。工作進程或cache管理進程釋放生命週期內存池並且結束。ngx_terminate 在收到結信號後設置為1.

* NGX_NOACCEPT_SIGNAL (SIGWINCH) - 優雅結束工作進程。

* NGX_RECONFIGURE_SIGNAL (SIGHUP) - 配置熱加載。 收到此信號後主進程根據配置文件創建新的cycle。如果這個新的cycle被成功的創建了，舊的cycle會被刪除並且啓動新的子進程。同時舊進程會被到 shutdown 信號。在單進程模式下，nginx 同樣創建新的cycle，但是舊的會一直保留到所有跟它關聯的連接都結束了。工作進程和helper進程忽略這種信號。

* NGX_REOPEN_SIGNAL (SIGUSR1) — 重新打開文件。主進程發送這個信號給工作進程。工作進程重新打開來自cycle的open_files。

* NGX_CHANGEBIN_SIGNAL (SIGUSR2) — 更新可執行程序。主進程啓動新的可執行程序，將所有的監聽文件描述符傳給它。這些列表是通過環境變量「NGINX」 傳遞的，描述符值以分號分隔。新的nginx實例讀這個變量然後將socket描述符添加到自己的初始cycle。其它進程忽略這種信號。

雖然nginx工作進程可以接受和處理POSIX信號，但是主進程卻不通過調用標準kill()給工作進程和help進程發送信號。nginx通過內部進程間通道發送消息。即使這樣，目前nginx也只是從主進程給工作進程發送消息。這些消息攜帶同樣的信號。這些通過是socketpairs，其對端在不同的進程。

當運行可執行程序，可以通過-s參數指定幾種值。分別是 stop, quit, reopen, reload。它們被轉化成信號 NGX_TERMINATE_SIGNAL, NGX_SHUTDOWN_SIGNAL, NGX_REOPEN_SIGNAL 和 NGX_RECONFIGURE_SIGNAL 並且被發送給nginx主進程，通過從nginx pid文件獲取進程id。

線程
====

可以將可能阻塞nginx工作進程的任務移到一個獨立的線程。舉例，nginx可以配置成使用線程來執行文件I/O操作。另一個例子是使用不具有異步接口的庫，不能按通常方式用於nginx。請記住，線程接口是現有異步處理客戶端連接的一種補充，而不是一種替代。

為了處理異步，可以使用以下原生pthread的封裝：

```
typedef pthread_mutex_t  ngx_thread_mutex_t;

ngx_int_t ngx_thread_mutex_create(ngx_thread_mutex_t *mtx, ngx_log_t *log);
ngx_int_t ngx_thread_mutex_destroy(ngx_thread_mutex_t *mtx, ngx_log_t *log);
ngx_int_t ngx_thread_mutex_lock(ngx_thread_mutex_t *mtx, ngx_log_t *log);
ngx_int_t ngx_thread_mutex_unlock(ngx_thread_mutex_t *mtx, ngx_log_t *log);

typedef pthread_cond_t  ngx_thread_cond_t;

ngx_int_t ngx_thread_cond_create(ngx_thread_cond_t *cond, ngx_log_t *log);
ngx_int_t ngx_thread_cond_destroy(ngx_thread_cond_t *cond, ngx_log_t *log);
ngx_int_t ngx_thread_cond_signal(ngx_thread_cond_t *cond, ngx_log_t *log);
ngx_int_t ngx_thread_cond_wait(ngx_thread_cond_t *cond, ngx_thread_mutex_t *mtx,
    ngx_log_t *log);
```

nginx實現了線程池策略，而不是為每個任務創建一個線程。可以配置多個線程池用於不同的目的（舉例，在不同的碰盤組上執行I/O）。每個線程池在啓動時創建，並且包含一定數目的線程用來處理一個任務隊列。當任務完成時，預定的handler就會被調用。

頭文件 src/core/ngx_thread_pool.h 包含了對應的定義：

```
struct ngx_thread_task_s {
    ngx_thread_task_t   *next;
    ngx_uint_t           id;
    void                *ctx;
    void               (*handler)(void *data, ngx_log_t *log);
    ngx_event_t          event;
};

typedef struct ngx_thread_pool_s  ngx_thread_pool_t;

ngx_thread_pool_t *ngx_thread_pool_add(ngx_conf_t *cf, ngx_str_t *name);
ngx_thread_pool_t *ngx_thread_pool_get(ngx_cycle_t *cycle, ngx_str_t *name);

ngx_thread_task_t *ngx_thread_task_alloc(ngx_pool_t *pool, size_t size);
ngx_int_t ngx_thread_task_post(ngx_thread_pool_t *tp, ngx_thread_task_t *task);
```

在配置階段，一個模塊通過調用ngx_thread_pool_add(cf, name)獲取線程池引用，以便使用線程。這個函數要麼創建新的線程池，要麼返回name對應存在的創建池引用。

在運行階段，用ngx_thread_task_post(tp, task)函數將任務添加進tp線程池的隊列。結構體ngx_thread_task_t包含了所有信息，用來執行線程里的用戶函數，傳遞參數和建立完成時的處理handler。

```
typedef struct {
    int    foo;
} my_thread_ctx_t;


static void
my_thread_func(void *data, ngx_log_t *log)
{
    my_thread_ctx_t *ctx = data;

    /* this function is executed in a separate thread */
}


static void
my_thread_completion(ngx_event_t *ev)
{
    my_thread_ctx_t *ctx = ev->data;

    /* executed in nginx event loop */
}


ngx_int_t
my_task_offload(my_conf_t *conf)
{
    my_thread_ctx_t    *ctx;
    ngx_thread_task_t  *task;

    task = ngx_thread_task_alloc(conf->pool, sizeof(my_thread_ctx_t));
    if (task == NULL) {
        return NGX_ERROR;
    }

    ctx = task->ctx;

    ctx->foo = 42;

    task->handler = my_thread_func;
    task->event.handler = my_thread_completion;
    task->event.data = ctx;

    if (ngx_thread_task_post(conf->thread_pool, task) != NGX_OK) {
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

模塊
=======

添加新模塊
--------
標準nginx模塊位於獨立的目錄，至少包含兩個文件：config和包含模塊源碼的文件。config包含需要跟nginx整合的信息，比如：
```
ngx_module_type=CORE
ngx_module_name=ngx_foo_module
ngx_module_srcs="$ngx_addon_dir/ngx_foo_module.c"

. auto/module

ngx_addon_name=$ngx_module_name
```

這是個POSIX shell腳本，它能設置（或訪問）以下變量：

* ngx_module_type — 模塊類型。可選值包括 CORE, HTTP, HTTP_FILTER, HTTP_INIT_FILTER, HTTP_AUX_FILTER, MAIL, STREAM, or MISC
* ngx_module_name — 模塊名稱。可以用空格分隔並且單個源文件可以構造多個模塊。如果是動態模塊，第一個名稱將作為二制進文件的名稱。這些名稱必須跟模塊裡面的能匹配。
* ngx_addon_name — 該模塊在控制台的輸出文本。
* ngx_module_srcs — 編譯該模塊時用到的源文件列表，用空格分隔。$ngx_addon_dir 變量可用作替代符，表示模塊的當前路徑。
* ngx_module_incs — 用於構建該模塊的包含路徑。
* ngx_module_deps — 模塊依賴頭文件列表。
* ngx_module_libs — 模塊用到的鏈接庫列表。 舉個例子，libpthread 可以這樣被鏈接 ngx_module_libs=-lpthread。這些宏可以直接在nginx里使用： LIBXSLT, LIBGD, GEOIP, PCRE, OPENSSL, MD5, SHA1, ZLIB, and PERL
* ngx_module_link — 模塊鏈接形式，DYNAMIC表示動態模塊，ADDON表示靜態模塊，其它根據不同的值會執行不同的操作。
* ngx_module_order — 模塊順序，設置模塊的加載順序在 HTTP_FILTER 和 HTTP_AUX_FILTER 類型的模塊中是很有用的。模塊按反序加載。
  在列表底部附近的 ngx_http_copy_filter_module 是最先被執行的。它讀數據給其它的filter使用。在列表頭部附近的ngx_http_write_filter_module 輸出數據，並且是最後執行的。

  選項格式是這樣的：當前模塊名稱緊接著用空格分隔的模塊列表，這些列表位置靠前，但執行是靠後。這個模塊將被插入在這個列表最後一個模塊的前面。
  
  對filter模塊默認是「ngx_http_copy_filter」，這樣該模塊被插入在copy filter之前，執行也就是copy filter的後面。對其它類型模塊默認值為空。

模塊通過使用 --add-module=/path/to/module 表示靜態編譯，--add-dynamic-module=/path/to/module 表示動態編譯。

核心模塊
-------

模塊是nginx的構建方式，nginx的大部份功能也被實現成模塊。模塊源文件必須包含類型為 ngx_module_t 的全局變量，定義為：

```
struct ngx_module_s {

    /* private part is omitted */

    void                 *ctx;
    ngx_command_t        *commands;
    ngx_uint_t            type;

    ngx_int_t           (*init_master)(ngx_log_t *log);

    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);

    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
    void                (*exit_thread)(ngx_cycle_t *cycle);
    void                (*exit_process)(ngx_cycle_t *cycle);

    void                (*exit_master)(ngx_cycle_t *cycle);

    /* stubs for future extensions are omitted */
};
```

省略私有部分包含模塊版本，簽名和預定義的宏 NGX_MODULE_V1。

每個模塊包含有私有數據的上下文，特定的配置指令，命令陣列，還有可能在特寫被調用的函數鈎子。模塊的生命週期由下面這些組成：

* 配置指令處理函數在master進程解析配置文件時被調用。
* init_module 在master進程成功解析配置後調用。
* master進程創建了worker進程，然後調用這些worker進程各自的 init_process。
* 當一個工作進程收到來自master的shutdown命令後 exit_process 被調用。
* master進程在退出前調用 exit_master。

init_module 可能會被調用多次，如果master進程做了配置的reload。

init_master, init_thread and exit_thread 目前是沒有實現的；線程在nginx里用於補充處理IO功能，而init_master看起來不是必須的。

type定義了模塊類型，有以下幾種：

* NGX_CORE_MODULE
* NGX_EVENT_MODULE
* NGX_HTTP_MODULE
* NGX_MAIL_MODULE
* NGX_STREAM_MODULE

NGX_CORE_MODULE 是最基礎和通用的，處於最低層次的類型。其它類型都依賴在它上面，並且提供更方便的方式去處理各自領域的問題，比如事件和http請求。

核心模塊有 ngx_core_module, ngx_errlog_module, ngx_regex_module, ngx_thread_pool_module, ngx_openssl_module，當然 http, stream, mail and event 也是。核心模塊的上下文定義如下：

```
typedef struct {
    ngx_str_t             name;
    void               *(*create_conf)(ngx_cycle_t *cycle);
    char               *(*init_conf)(ngx_cycle_t *cycle, void *conf);
} ngx_core_module_t;
```

name只是用於方便識別的模塊字串名稱，create_conf 和 init_conf 指向創建和初始模塊對應的配置結構體。對核心模塊，create_conf在解析配置之前被調用， init_conf 在配置成功解析後調用。典型的 create_conf 函數分配空間用於配置，並且設置默認值。init_conf 處理已知配置，然後執行合理的校驗和完成配置初始化。

舉個例子，很簡單的模塊 ngx_foo_module 是這樣的：

```
/*
 * Copyright (C) Author.
 */


#include <ngx_config.h>
#include <ngx_core.h>


typedef struct {
    ngx_flag_t  enable;
} ngx_foo_conf_t;


static void *ngx_foo_create_conf(ngx_cycle_t *cycle);
static char *ngx_foo_init_conf(ngx_cycle_t *cycle, void *conf);

static char *ngx_foo_enable(ngx_conf_t *cf, void *post, void *data);
static ngx_conf_post_t  ngx_foo_enable_post = { ngx_foo_enable };


static ngx_command_t  ngx_foo_commands[] = {

    { ngx_string("foo_enabled"),
      NGX_MAIN_CONF|NGX_DIRECT_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      0,
      offsetof(ngx_foo_conf_t, enable),
      &ngx_foo_enable_post },

      ngx_null_command
};


static ngx_core_module_t  ngx_foo_module_ctx = {
    ngx_string("foo"),
    ngx_foo_create_conf,
    ngx_foo_init_conf
};


ngx_module_t  ngx_foo_module = {
    NGX_MODULE_V1,
    &ngx_foo_module_ctx,                   /* module context */
    ngx_foo_commands,                      /* module directives */
    NGX_CORE_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};


static void *
ngx_foo_create_conf(ngx_cycle_t *cycle)
{
    ngx_foo_conf_t  *fcf;

    fcf = ngx_pcalloc(cycle->pool, sizeof(ngx_foo_conf_t));
    if (fcf == NULL) {
        return NULL;
    }

    fcf->enable = NGX_CONF_UNSET;

    return fcf;
}


static char *
ngx_foo_init_conf(ngx_cycle_t *cycle, void *conf)
{
    ngx_foo_conf_t *fcf = conf;

    ngx_conf_init_value(fcf->enable, 0);

    return NGX_CONF_OK;
}


static char *
ngx_foo_enable(ngx_conf_t *cf, void *post, void *data)
{
    ngx_flag_t  *fp = data;

    if (*fp == 0) {
        return NGX_CONF_OK;
    }

    ngx_log_error(NGX_LOG_NOTICE, cf->log, 0, "Foo Module is enabled");

    return NGX_CONF_OK;
}
```

配置指令
------------------------

ngx_command_t 表示一個配置指令。每個模塊包含一組指令，每個指令的格式表示了如何處理參數和解析時調用的函數。

```
struct ngx_command_s {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};
```

指令陣列以 「ngx_null_command」 結束。name 是指令名稱，體現在配置文件中，比如 「worker_processes」 or 「listen」。type 是bit組合，表示參數個數，指令類型和其它對應的屬性。參數的標記為：

* NGX_CONF_NOARGS — 沒有參數
* NGX_CONF_1MORE — 至少一個參數
* NGX_CONF_2MORE — 至少兩個參數
* NGX_CONF_TAKE1..7 — 明確的1..7個參數
* NGX_CONF_TAKE12, 13, 23, 123, 1234 — 一個或兩個參數，一個或參數，依此類推。

指令類型：

* NGX_CONF_BLOCK — 表示是一個塊，比如它可能用 { } 包含其它指令，或自己實現的解析以處理包含的內容，比如 map 指領。
* NGX_CONF_FLAG — 表示是個boolean的標記，「on」 或者 「off」。

指令的上下文定義了配置的位置，並且關聯到對應的存儲配置的地方。

* NGX_MAIN_CONF — 上層配置
* NGX_HTTP_MAIN_CONF — http 塊
* NGX_HTTP_SRV_CONF — http server 塊
* NGX_HTTP_LOC_CONF — http location 塊
* NGX_HTTP_UPS_CONF — http upstream 塊
* NGX_HTTP_SIF_CONF — http server 「if」 塊
* NGX_HTTP_LIF_CONF — http location 「if」 塊
* NGX_HTTP_LMT_CONF — http 「limit_except」 塊
* NGX_STREAM_MAIN_CONF — stream 塊
* NGX_STREAM_SRV_CONF — stream server 塊
* NGX_STREAM_UPS_CONF — stream upstream 塊
* NGX_MAIL_MAIN_CONF — mail 塊
* NGX_MAIL_SRV_CONF — mail server 塊
* NGX_EVENT_CONF — event 塊
* NGX_DIRECT_CONF — 沒有層級的上下文，直接存儲在模塊的ctx

配置解析時根據這些標記，要麼對放錯位置的指令拋出錯誤，要麼調用指令handler，這樣即使相同的配置在不同的location也能存儲到能區分的位置。

set字段定義瞭解析配置時調用的handler，並且將解析的值存放到對應的配置結構體。Nginx提供了一些方便的公共函數集：

* ngx_conf_set_flag_slot — 將 「on」 or 「off」 轉化成 ngx_flag_t 類型的值 1 or 0
* ngx_conf_set_str_slot — 存儲類型為 ngx_str_t 的值
* ngx_conf_set_str_array_slot — 追加元素為ngx_str_t的ngx_array_t一個新的值。array會自動創建，如果不存在的話。
* ngx_conf_set_keyval_slot — 追加元素為ngx_keyval_t的ngx_array_t一個新的值。第一個作為鍵，第二個作為值，如果不存在的話。
* ngx_conf_set_num_slot — 轉化參數為 ngx_int_t 類型的值
* ngx_conf_set_size_slot — 轉化參數為 size_t 類型的值
* ngx_conf_set_off_slot — 轉化參數為 off_t 類型的值
* ngx_conf_set_msec_slot — 轉化參數為 ngx_msec_t 類型的值
* ngx_conf_set_sec_slot — 轉化參數為 time_t 類型的值
* ngx_conf_set_bufs_slot — 轉化兩個參數為 ngx_bufs_t，包含了 ngx_int_t 類型的 number 和 buffers的size
* ngx_conf_set_enum_slot — 轉化參數為 ngx_uint_t 類型的值。這是個類似枚舉的功能，可以傳以 null-terminated 結尾的 ngx_conf_enum_t 陣列給post字段，以設置對應的值。
* ngx_conf_set_bitmask_slot — 轉化參數為 ngx_uint_t 類型的值。這是個類似枚舉的功能，可以傳以 null-terminated ngx_conf_bitmask_t 陣列給post字段，以設置對應的值。 
* set_path_slot — 轉化參數為 ngx_path_t 類型並且做必須的初始化。詳情請看 proxy_temp_path 指令
* set_access_slot — 轉化參數為文件權限mask。詳情請看 proxy_store_access 指令。

conf字段定義了哪個上下文用來存儲指令的值，或者用NULL表示不使用上下文。簡單的核心模塊不用配置上下文並且設置 NGX_DIRECT_CONF 標識。 在真實場景里，像http或stream的模塊往往更複雜，配置可以在pre-server或者pre-location里，還有甚至是在 "if" 里的。這樣的模塊里，配置結構會更複雜，請到一些模塊里看他們是如何管理各自的配置的。

* NGX_HTTP_MAIN_CONF_OFFSET — http 塊配置
* NGX_HTTP_SRV_CONF_OFFSET — http 塊配置
* NGX_HTTP_LOC_CONF_OFFSET — http 塊配置
* NGX_STREAM_MAIN_CONF_OFFSET — stream 塊配置
* NGX_STREAM_SRV_CONF_OFFSET — stream server 塊配置
* NGX_MAIL_MAIN_CONF_OFFSET — mail 塊配置
* NGX_MAIL_SRV_CONF_OFFSET — mail server 塊配置

offset字段定義了存儲該指令值的位置在配置結構體的偏移大小。典型的使用是調用 offsetof() 宏。

post字段包含雙重意思：它可能在主handler完成後調用，或者傳額外的數據給主handler。第一種情況 ngx_conf_post_t 需要初始化handler，舉個例子：

```
static char *ngx_do_foo(ngx_conf_t *cf, void *post, void *data);
static ngx_conf_post_t  ngx_foo_post = { ngx_do_foo };
```

post函數參數是：ngx_conf_post_t它自己, data 來自主handler的參數。


HTTP
====

連接
----

每個HTTP客戶端連接經歷以下幾個階段：

* ngx_event_accept() 接受一個客戶端TCP連接。這個函數在監聽socket發生讀通知時被調用。在這階段創建新的 ngx_connecton_t 對象。這個對象封裝了新接受的客戶端socket。每個nginx監聽會提供並傳遞給這個新的connection對象一個handler。比如 HTTP connection 是ngx_http_init_connection(c)。
* ngx_http_init_connection() 執行了HTTP connection的早期初始化。這個階段為connection創建了一個 ngx_http_connection_t 對象，並且引用存放在 connection 的 data 字段。稍後會被替換成 HTTP request 對象。PROXY 協議的解析和 SSL 握手也發生在這個階段。
* ngx_http_wait_request_handler() 是讀事件handler，當客戶端socket有數據可讀時被調用。在這個階段會創建 HTTP request 對象 ngx_http_request_t 並且設置到 connection 的 data 字段。
* ngx_http_process_request_line() 是讀事件handler，用來讀請求行。這個 handler 在 ngx_http_wait_request_handler() 里設置。數據被讀進 connection 的 buffer。 buffer的大小初始值是指令 client_header_buffer_size。整個 client header 應該是適合這個buffer的。如果這個初始值不夠時，會分配一個更大的buffer，它的大小為指令large_client_header_buffers的值。
* ngx_http_process_request_headers() 是讀事件handler，在 ngx_http_process_request_line() 之後設置，被用來讀請求頭。
* ngx_http_core_run_phases() 當整個http請求頭讀完和解析後調用。這個函數運行從 NGX_HTTP_POST_READ_PHASE 到 NGX_HTTP_CONTENT_PHASE 的請求階段。最後階段產生響應內容並傳給整個filter鏈。響應不一定要在這階段發給客戶端。它可能緩衝起來然後在最後階段發送。
* ngx_http_finalize_request() 通常在請求已經產生了所有的輸出或發生錯誤時調用。後者會查找合適的錯誤頁面作為響應。如果響應沒有完全的發送給客戶端，HTTP寫處理 ngx_http_writer() 會被激活以完成數據的發送。
* ngx_http_finalize_connection() 在響應完全發送給客戶端後調用，然後銷毀請求。如果客戶端連接的keepalive功能啓用了，ngx_http_set_keepalive() 會被調用，用來銷毀當前請求並等待這個連接的下一個請求。否則，調用 ngx_http_close_request() 同時銷毀請求和連接。

請求
----

對每個客戶端HTTP請求創建一個ngx_http_request_t對象。以下是這個對象的一些字段：

* connection — 指向類型為 ngx_connection_t 的 connection 對象。多個請求可能同時指向同個連接 - 一個主請求和它的多個子請求。一個請求被刪除後，新的請求可能會在同樣的連接上被創建。

    注意：HTTP連接 ngx_connection_t 的 data 字段會指向這個請求。這種請求被認為是激活的，相反的其它該連接上的請求則不是。激活的請求被用來處理客戶端事件，並且允許發送它的響應給客戶端。通常每個請求會在某個時間點激活以發送它的數據。

* ctx — 一組HTTP模塊的上下文。每個類型為 NGX_HTTP_MODULE 的模塊在這個請求里可以存任意的東西（通常指向一個結構體）。值存放在模塊ctx_index位置上對應ctx陣列的地方。以下宏提供了獲取和設置請求上下文的方便方式。
    * ngx_http_get_module_ctx(r, module) — 返回模塊的上下文。
    * ngx_http_set_ctx(r, c, module) — 設置c為模塊的上下文。
* main_conf, srv_conf, loc_conf — 當前請求的配置陣列。配置存放在模塊的ctx_index對應的位置。
* read_event_handler, write_event_handler - 請求的讀寫事件handler。通常，HTTP連接用 ngx_http_request_handler() 作為讀寫事件 handler。這個函數會調用當前激活請求的 read_event_handler 和 write_event_handler。
* cache — 用於緩存上游響應的緩存對象。
* upstream — 用於代理的上游對象。
* pool — 請求內存池。這個內存池在請求被刪除後被銷毀。這個請求對象本身也是從該內存池分配的。對需要活動在整個客戶端連接生命週期的分配，應該使用 ngx_connection_t 的 內存池。
* header_in — 從請求頭讀的buffer。
* headers_in, headers_out — 輸入和輸出的 HTTP 頭部對象。兩個對象都包含類型為 ngx_list_t 的 headers 頭部域，用來保存原始的頭部列表。此外還有比較特別的單獨字段，用來直接獲取和設置，比如 content_length_n, status 等等。
* request_body — 客戶端請求體對象。
* start_sec, start_msec — 請求創建時間點。用於跟蹤請求時間。
* method, method_name — 客戶端HTTP請求方法的數字和文本表示方式。方法的數字值定義在 src/http/ngx_http_request.h，有 NGX_HTTP_GET, NGX_HTTP_HEAD, NGX_HTTP_POST 等宏。
* http_protocol, http_version, http_major, http_minor - 客戶端HTTP協議和版本的文本形式 (「HTTP/1.0」, 「HTTP/1.1」 等)，數字形式 (NGX_HTTP_VERSION_10, NGX_HTTP_VERSION_11 等) 和主次版本號
* request_line, unparsed_uri — 客戶端原始的請求行和URI。
* uri, args, exten — 當前請求的請求URI, 參數和文件擴展名。URI值可能由於規範跟客戶端發送過來的原始URI不同。經過請求處理，這些值可能在內部重定向時發生改變。
* main — 指向主請求對象。創建這個創建用來處理HTTP請求，而那些子請求被創建用來執行主請求里的特定子任務。
* parent — 子請求指向的父請求。
* postponed — 依次要發送和創建的buffer和子請求列表。這個列表被用在 postpone filter 以提供連續的請求輸出，它的各部份由子請求創建。
* post_subrequest — 不用於主請求。指向子請求完成會調用的具有上下文的handler。不用於主請求。
* posted_requests — 開始要執行或恢復的請求列表。通過調用請求的write_event_handler完成啓動或恢復。通常這個handler會保留請求主函數，第一個運行請求階段並且產生輸出的。

    一個請求經常通過調用 ngx_http_post_request(r, NULL)加到posted_requests。這樣會加到主請求的 posted_requests 列表裡。函數會 ngx_http_run_posted_requests(c) 會運行所有的請求，這些添加在通過連接激活請求對應的主請求。這個函數應該在所有的事件處理中調用，這樣能產生新的添加請求。通常在執行了請求的讀寫處理後調用。

* phase_handler — 當前請求階段的索引。
* ncaptures, captures, captures_data — 請求最後一次正則匹配產生的正則capture。當處理一個請求時，有很多地方可以發生正則匹配：map 查找， server 通過 SNI 或 HTTP Host 查找，rewrite, proxy_redirect 等等。capture 在查找時產生並且保存這些字段裡。字段 ncaptures 保存caputure的個數, captures 保存 capture 邊界，captures_data 保存字串，針對這些匹配到的正則和被用於精確的capture。每次正則匹配後，請求capture會重置並且保存新的值。
* count — 請求引用計數。這個字段只發生在主請求上。通過簡單的 r->main->count++ 就可以遞增。要通過 ngx_http_finalize_request(r, rc) 遞減。創建子請求和運行讀請求體處理都會增加這個計數。
* subrequests — 當前子請求的嵌套級別。每個子請求會讓它的父請求的嵌套級別數減1。一旦這個值到達0就會發生錯誤，主請求的這個值定義為 NGX_HTTP_MAX_SUBREQUESTS 這個常量。
* uri_changes — 請求的URI剩餘可改變數。一個請求可以改變它的URI的總次數限制為 NGX_HTTP_MAX_URI_CHANGES 這個常量。每次變化都會遞減直到0。後者會導致錯誤發生。這些被認為是改變URI的操作是重寫和內部重定向到普通或有命名的location。
* blocked — 請求上的阻塞次數。只要此值為非0,請求不會被終止。目前這個值會由於待處理AIO（POSIX AIO和線程操作）操作和緩存鎖增加。
* buffered — 位，表示一些模塊緩衝了請求產生的輸出。一些filter都可以緩衝輸出，比如 sub_filter 可以緩衝數據用來作部分字串匹配，copy filter 因為缺少空閒的output_buffers緩衝數據，等等。只要這個值為非0，請求就不會終止，期望繼續刷新。
* header_only — 標記。用於表示不需要輸出請求體。舉例，這個標記用於 HTTP HEAD 請求。
* keepalive — 標記。用於表示否支持客戶端的持久連接。這個值根據 HTTP 版本和 頭部 "Connection" 的值推算出。

* header_sent — 標記。表示請求的頭部信息已經發送（不一定發到客戶端）。
* internal — 標記。表示當前請求是內部的。要進入這種內部的狀態，請求必須通過內部重定向或者是一個子請求。內部請求進入內部的location。
* allow_ranges — 標記。用於表示如果是HTTP Range的請求，可以發送部份響應給客戶端。
* subrequest_ranges — 標記。用於表示處理子請求時，允許發送部分響應給客戶端。
* single_range — 標記。表示只有一個連續的range能被發送給客戶端。這個標記通常在發送數據流時設置，比如來自代理服務器，並且整個響應不是一次完成的。
* main_filter_need_in_memory, filter_need_in_memory — 標記。用於表示輸出應該產生自內存，而非文件。這個被copy filter用來從文件buffer讀數據，即使開了sendfile。兩者的匹別在設置它們的filter模塊的location。這些在postpone filter調用之前的filters，設置了filter_need_in_memory 表明當前請求的輸出應該來自memory buffer。在之後調用的filter設置 main_filter_need_in_memory 表明主請求和子請求在發送輸出時都要從讀文件到內存里。
* filter_need_temporary — 表示請求輸出應該產生自 temporary buffer，而且不能是只讀的memory buffer或file buffer。這個用於那些可能直接改變要發送buffer輸出的filter。

配置
-------------

每個HTTP模塊都可以有三種類型的配置：

* Main配置。 此配置作用於整個http{}塊，屬於全局配置。此配置中存儲了模塊的全局配置。
* Server配置. 此配置作用於一個server{}塊，用於存儲模塊server特有的配置。
* Location配置. 此配置作用於一個location{}塊，if{}塊或者limit_except()塊，用於存儲location相關的配置。

上述配置的結構體是在nginx的配置階段，通過調用一系列函數來創建的。這些函數會為配置結構體分配內存，並進行初始化和合併操作。下面的例子演示了如何創建一個簡單的location配置。該配置中只有一個無符號整形的配置項foo。

```
typedef struct {
    ngx_uint_t  foo;
} ngx_http_foo_loc_conf_t;


static ngx_http_module_t  ngx_http_foo_module_ctx = {
    NULL,                                  /* preconfiguration */
    NULL,                                  /* postconfiguration */

    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */

    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */

    ngx_http_foo_create_loc_conf,          /* create location configuration */
    ngx_http_foo_merge_loc_conf            /* merge location configuration */
};


static void *
ngx_http_foo_create_loc_conf(ngx_conf_t *cf)
{
    ngx_http_foo_loc_conf_t  *conf;

    conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_foo_loc_conf_t));
    if (conf == NULL) {
        return NULL;
    }

    conf->foo = NGX_CONF_UNSET_UINT;

    return conf;
}


static char *
ngx_http_foo_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
{
    ngx_http_foo_loc_conf_t *prev = parent;
    ngx_http_foo_loc_conf_t *conf = child;

    ngx_conf_merge_uint_value(conf->foo, prev->foo, 1);
}
```

在例子中可見，ngx_http_foo_create_loc_conf()函數創建了一個新的配置結構，ngx_http_foo_merge_loc_conf()函數則將配置和更高層次的配置進行合併。實際上，server和location的配置並不僅僅存在於server和location這兩個配置層次中，而是為相應更高的配置層次全部進行創建。具體來說，server配置也會在main層次進行創建，而location配置同時會在main, server和location三個層次創建。這些配置使得server和location的配置出現在任何層次的nginx配置中成為了可能。最終各級配置會進行合併。為了在合併的時候識別出缺失的配置並進行忽略，nginx提供了一系列類似於NGX_CONF_UNSET和NGX_CONF_UNSET_UINT這樣的宏。標準的nginx合併宏，比如ngx_conf_merge_value()和ngx_conf_merge_uint_value()，提供了一種更加方便的方法來對配置選項進行合併，此外如果在配置文件中沒有顯式的進行配置，上述合併宏還可以設置默認值。完整的合併宏請參考src/core/ngx_conf_file.h文件。

可以使用如下這些宏來再配置階段訪問HTTP模塊的配置。它們的第一個參數都是ngx_conf_t類型的指針。

* ngx_http_conf_get_module_main_conf(cf, module)
* ngx_http_conf_get_module_srv_conf(cf, module)
* ngx_http_conf_get_module_loc_conf(cf, module)

下面的例子展示了nginx核心模塊ngx_http_core_module的location配置的指針，並修改其content handler內容的過程。

```
static ngx_int_t ngx_http_foo_handler(ngx_http_request_t *r);


static ngx_command_t  ngx_http_foo_commands[] = {

    { ngx_string("foo"),
      NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS,
      ngx_http_foo,
      0,
      0,
      NULL },

      ngx_null_command
};


static char *
ngx_http_foo(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_core_loc_conf_t  *clcf;

    clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
    clcf->handler = ngx_http_bar_handler;

    return NGX_CONF_OK;
}
```

在運行階段，可以使用下面的這些宏來獲取HTTP模塊的配置。

* ngx_http_get_module_main_conf(r, module)
* ngx_http_get_module_srv_conf(r, module)
* ngx_http_get_module_loc_conf(r, module)

需要將指向表示HTTP請求的ngx_http_request_t結構體的指針傳遞給這些宏。對於一個請求，main配置從不會發生變化，server配置會在切換虛擬服務器配置後發生改變。請求的location配置會隨著rewrite或者內部重定向而被多次改變。下面的例子展示了如何在運行階段獲取HTTP配置。

```
static ngx_int_t
ngx_http_foo_handler(ngx_http_request_t *r)
{
    ngx_http_foo_loc_conf_t  *flcf;

    flcf = ngx_http_get_module_loc_conf(r, ngx_http_foo_module);

    ...
}
```


階段
----
每個HTTP請求都會經過一系列HTTP階段（phase），其中每個階段都會負責處理不同的功能。大部分階段允許註冊handler，這些階段的handler會在請求到達這個階段的時候被調用。很多標準nginx模塊通過註冊階段handler的方式來實現在某個請求處理階段被調用模塊邏輯。下面是nginx HTTP階段列表：

* NGX_HTTP_POST_READ_PHASE是最開始的一個階段。ngx_http_realip_module模塊在此註冊了handler，這樣一來就可以在其他模塊被觸發之前就替換掉客戶端的IP地址。
* NGX_HTTP_SERVER_REWRITE_PHASE是用來執行server層面rewrite腳本的階段。ngx_http_rewrite_module模塊在這裡註冊handler。
* NGX_HTTP_FIND_CONFIG_PHASE — 基於請求URI來查找location的特殊階段。這個階段裡不允許註冊任何handler。該階段只執行匹配location的動作。在進入到這個階段之前，請求中的location被設置成了對應server中的默認location，任何模塊獲取請求的location配置，只會得到默認location的配置。這個階段之後，請求將會得到新的location配置。
* NGX_HTTP_REWRITE_PHASE — 和NGX_HTTP_SERVER_REWRITE_PHASE階段類似，不過是執行上個階段新選擇的location中的rewrite動作。
* NGX_HTTP_POST_REWRITE_PHASE — 用於將請求重定向到新location的特殊階段，這種重定向會在URI被rewrite過的情況下發生。重定向是通過重新跳轉回NGX_HTTP_FIND_CONFIG_PHASE階段來實現的。該階段不允許註冊handler。
* NGX_HTTP_PREACCESS_PHASE — 這是一個可以註冊不同類型handler的通用階段，此時沒有進行過訪問控制檢查。標準nginx模塊ngx_http_limit_conn_module和ngx_http_limit_req_module在此階段註冊了handler。
* NGX_HTTP_ACCESS_PHASE — 對請求進行訪問控制權限檢查的階段。ngx_http_access_module和ngx_http_auth_basic_module這些標準nginx模塊在此階段註冊handler。如果使用satisfy指令進行相應的配置，則可以實現只要任意一個handler進行了放行，請求就可以繼續後續的處理。
* NGX_HTTP_POST_ACCESS_PHASE — 對於satisfy設置為any時候的特殊階段。如果某些access階段的handler阻斷了了訪問且沒有其他handler放行，則請求會被阻斷。此階段不允許註冊任何handler。
* NGX_HTTP_TRY_FILES_PHASE — 實現try_file功能的特殊階段。此階段不允許註冊任何handler。
* NGX_HTTP_CONTENT_PHASE — 用於生成HTTP應答的階段。多個標準nginx模塊在此階段註冊handler，例如ngx_http_index_module和ngx_http_static_module模塊。所有註冊的這些模塊handler會被按照順序調用直到其中的一個生成輸出。也可以基於每個location單獨設置content handler。如果ngx_http_core_module模塊的location配置中的handler成員被設置，則在NGX_HTTP_CONTENT_PHASE階段此handler會被調用，註冊到此階段的其他handler會被忽略。
* NGX_HTTP_LOG_PHASE用來對請求記錄日誌。當前，只有ngx_http_log_module模塊在此階段註冊handler以便記錄訪問日誌。Log階段handler在每個請求結束，但還沒被釋放的時候被調用。

以下是使用preaccess階段handler的例子：

```
static ngx_http_module_t  ngx_http_foo_module_ctx = {
    NULL,                                  /* preconfiguration */
    ngx_http_foo_init,                     /* postconfiguration */

    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */

    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */

    NULL,                                  /* create location configuration */
    NULL                                   /* merge location configuration */
};


static ngx_int_t
ngx_http_foo_handler(ngx_http_request_t *r)
{
    ngx_str_t  *ua;

    ua = r->headers_in->user_agent;

    if (ua == NULL) {
        return NGX_DECLINED;
    }

    /* reject requests with "User-Agent: foo" */
    if (ua->value.len == 3 && ngx_strncmp(ua->value.data, "foo", 3) == 0) {
        return NGX_HTTP_FORBIDDEN;
    }

    return NGX_DECLINED;
}


static ngx_int_t
ngx_http_foo_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_PREACCESS_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_foo_handler;

    return NGX_OK;
}
```

階段的handler可以返回如下返回值：

* NGX_OK — 繼續執行下個階段
* NGX_DECLINED — 繼續執行當前階段的下一個handler。如果當前handler是本階段的最後一個handler，則執行下個階段。
* NGX_AGAIN, NGX_DONE — 掛起階段處理直到事件發生。此場景可以用來處理異步I/O操作或者進行延遲處理。階段處理應該通過對ngx_http_core_run_phases()函數的調用來恢復。
* 任何其他的返回值都被視為請求結束，尤其是HTTP應答碼，這種情況下會以返回的HTTP應答碼結束當前請求。

一些階段對返回值的處理稍有不同。在content階段，除了NGX_DECLINED之外的任何返回值都會被當成結束請求處理。對於location提供的content handler，任何返回值都會別當成結束狀態碼進行處理。在access階段，如果使用了satisfy any模式，返回除了NGX_OK，NGX_DECLINED，NGX_AGAIN和NGX_DONE之外的值會被作為阻斷處理。如果沒有其他的access handler對請求放行或者通過一個返回碼阻斷，則前述導致阻斷的返回值會被當成結束狀態碼。

變量
----

訪問已有變量
----------

變量可以通過索引（即index，這是最常用的方式）或者名字（參考下文關於創建變量的章節）。索引是在配置階段，當一個變量添加到配置中的時候創建。變量索引可以通過ngx_http_get_variable_index()函數獲取：

```
ngx_str_t  name;  /* ngx_string("foo") */
ngx_int_t  index;

index = ngx_http_get_variable_index(cf, &name);
```

這裡，cf變量是一個指向nginx配置的指針，name則指向變量名稱字串。該函數在執行出錯時候返回NGX_ERROR，其他情況下典型的做法是將返回的索引存儲在模塊配置中以便後續使用。

所有的HTTP變量都是基於HTTP請求的上下文而計算的，其結果也是與HTTP請求相關並存儲於其中。所有用於計算變量的函數的返回值都是ngx_http_variable_value_t類型，該類型代表了一個變量的值。

```
typedef ngx_variable_value_t  ngx_http_variable_value_t;

typedef struct {
    unsigned    len:28;

    unsigned    valid:1;
    unsigned    no_cacheable:1;
    unsigned    not_found:1;
    unsigned    escape:1;

    u_char     *data;
} ngx_variable_value_t;
```

說明：

* len — 值的長度
* data — 值本身
* valid — 值是有效的
* not_found — 變量沒有找到，因此data和len成員無意義；例如，像嘗試獲取$arg_foo這種類型的變量的值，而請求中卻沒有名為foo的參數時，就可能發生這種情況。
* no_cacheable — 禁止緩存結果值
* escape — 由日誌模塊內部使用，用來標記在輸出時需要進行轉移的變量值

ngx_http_get_flushed_variable()和ngx_http_get_indexed_variable()函數用來獲取變量值。它們擁有相同的接口 —— 一個HTTP請求r作為計算變量值的上下文以及一個index參數，用於指示哪個變量。以下是一個典型的用法：

```
ngx_http_variable_value_t  *v;

v = ngx_http_get_flushed_variable(r, index);

if (v == NULL || v->not_found) {
    /* we failed to get value or there is no such variable, handle it */
    return NGX_ERROR;
}

/* some meaningful value is found */
```

這兩個函數的區別是，ngx_http_get_indexed_variable()返回緩存的變量值而ngx_http_get_flushed_variable()函數對於不可緩存的變量進行刷新處理。

有一些場景中需要處理那些在配置階段還不知道名字的變量，這些變量無法通過使用索引來訪問，例如SSI和Perl模塊。對於這類場景，可以使用ngx_http_get_variable(r, name, key)函數。該函數通過變量名字和它的哈希key來查找變量。

創建變量
-------

ngx_http_add_variable()函數用來創建一個變量。其參數有：配置（註冊變量的配置），變量名和用來控制變量行為的標記位：

* NGX_HTTP_VAR_CHANGEABLE  — 允許變量被重新定義；如果另外一個模塊使用同樣的名字定義變量，不會產生衝突。例如，這個特點允許用戶使用set指令覆蓋變量。
* NGX_HTTP_VAR_NOCACHEABLE  — 禁止緩存，在類似於$time_local這樣的變量上使用。
* NGX_HTTP_VAR_NOHASH  — 標識這個變量只能通過索引訪問，不允許通過變量名訪問。這是一個小的優化，可以在類似於SSI或者Perl這樣的模塊中不需要此變量的時候使用。
* NGX_HTTP_VAR_PREFIX  — 該變量的名字是一個前綴。相關的handler必須實現額外的邏輯來獲取指定的變量值。例如，所有"arg_"變量都被同一個handler處理，該handler在請求的參數中查找並返回對應的參數值。

此函數在失敗時返回NULL，否則返回一個指向ngx_http_variable_t類型的指針：

```
struct ngx_http_variable_s {
    ngx_str_t                     name;
    ngx_http_set_variable_pt      set_handler;
    ngx_http_get_variable_pt      get_handler;
    uintptr_t                     data;
    ngx_uint_t                    flags;
    ngx_uint_t                    index;
};
```

get和set handler被用來獲取以及設置變量的值，data成員會被傳遞給變量handler，index成員中存儲的是分配的變量索引，用來引用變量。

通常，一個以null結尾的上述結構體陣列會在模塊中創建，並在preconfiguration階段將陣列中的變量添加到配置中：

```
static ngx_http_variable_t  ngx_http_foo_vars[] = {

    { ngx_string("foo_v1"), NULL, ngx_http_foo_v1_variable, NULL, 0, 0 },

    { ngx_null_string, NULL, NULL, 0, 0, 0 }
};

static ngx_int_t
ngx_http_foo_add_variables(ngx_conf_t *cf)
{
    ngx_http_variable_t  *var, *v;

    for (v = ngx_http_foo_vars; v->name.len; v++) {
        var = ngx_http_add_variable(cf, &v->name, v->flags);
        if (var == NULL) {
            return NGX_ERROR;
        }

        var->get_handler = v->get_handler;
        var->data = v->data;
    }

    return NGX_OK;
}
```

HTTP模塊上下文中的preconfiguration成員會被賦值為這個函數，並在解析HTTP配置之前被調用，所以它可以處理這些變量。

get handler負責為某個請求計算變量的值，例如：

```
static ngx_int_t
ngx_http_variable_connection(ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data)
{
    u_char  *p;

    p = ngx_pnalloc(r->pool, NGX_ATOMIC_T_LEN);
    if (p == NULL) {
        return NGX_ERROR;
    }

    v->len = ngx_sprintf(p, "%uA", r->connection->number) - p;
    v->valid = 1;
    v->no_cacheable = 0;
    v->not_found = 0;
    v->data = p;

    return NGX_OK;
}
```

如果內部出現錯誤（比如分配內存失敗）則返回NGX_ERROR，否則返回NGX_OK。變量計算結果的狀態可以通過ngx_http_variable_value_t的flags成員的值來瞭解（參考前文相關描述）。

set handler允許設置變量所指向的屬性。例如，$limit_rate變量的set handler修改了請求的limit_rate成員的值：

```
...
{ ngx_string("limit_rate"), ngx_http_variable_request_set_size,
  ngx_http_variable_request_get_size,
  offsetof(ngx_http_request_t, limit_rate),
  NGX_HTTP_VAR_CHANGEABLE|NGX_HTTP_VAR_NOCACHEABLE, 0 },
...

static void
ngx_http_variable_request_set_size(ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data)
{
    ssize_t    s, *sp;
    ngx_str_t  val;

    val.len = v->len;
    val.data = v->data;

    s = ngx_parse_size(&val);

    if (s == NGX_ERROR) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "invalid size \"%V\"", &val);
        return;
    }

    sp = (ssize_t *) ((char *) r + data);

    *sp = s;

    return;
}
```

複雜值
-------

複雜值提供了一種簡單的方法來計算一個包含有文本、變量以及文本變量組合等情況的表達式的值。

由ngx_http_compile_complex_value所表示的複雜值在配置階段被編譯到ngx_http_complex_value_t類型中，該編譯的結果在運行階段可以被用來計算表達式的值。

```
ngx_str_t                         *value;
ngx_http_complex_value_t           cv;
ngx_http_compile_complex_value_t   ccv;

value = cf->args->elts; /* directive arguments */

ngx_memzero(&ccv, sizeof(ngx_http_compile_complex_value_t));

ccv.cf = cf;
ccv.value = &value[1];
ccv.complex_value = &cv;
ccv.zero = 1;
ccv.conf_prefix = 1;

if (ngx_http_compile_complex_value(&ccv) != NGX_OK) {
    return NGX_CONF_ERROR;
}
```

這裡，ccv里包含了全部初始化複雜值cv所需的參數：

* cf — 配置指針
* value — 待解析的字串 (輸入)
* complex_value — 編譯後的值 (輸出)
* zero — 是否對結果進行0結尾處理
* conf_prefix — 是否將結果帶上配置前綴（nginx當前查找配置的目錄）
* root_prefix — 是否將結果帶上根前綴（通常是nginx的安裝目錄）

zero標記位在需要把結果傳遞給要求0結尾字串的庫時，非常有用，而前綴相關的標記位在處理文件名時很方便。

對於正確的編譯，可以從cv.lengths成員獲取到表達式中是否存在變量的情況。如果為NULL，則表示表達式中只是純文本，所以沒有必要將其保存成一個複雜值，使用簡單的字串就可以了。

ngx_http_set_complex_value_slot()可以在聲明指令的時候對複雜值進行初始化。

在運行階段，複雜值可以使用ngx_http_complex_value()函數來計算：

```
ngx_str_t  res;

if (ngx_http_complex_value(r, &cv, &res) != NGX_OK) {
    return NGX_ERROR;
}
```

給定請求r和之前編譯的cv，該函數會對表達式的值進行急計算並將結果存放在res變量中。

請求重定向
---------

HTTP請求總是通過ngx_http_request_t結構體的loc_conf成員來綁定到某個location上。這意味著在任意時刻，任何模塊都可以通過調用ngx_http_get_module_loc_conf(r, module)來獲取到location的配置。在HTTP請求的生命週期內，其location可能會改變多次。初始時，default server的default location會被分配給HTTP請求。一旦這個請求切換到了另外一個不同的server（比如通過HTTP的"Host"頭，或者通過SSL的SNI擴展），該server的default location也同樣會分配給這個請求。接下來在NGX_HTTP_FIND_CONFIG_PHASE階段中會重新為請求選擇location。在這個階段裡，location的選擇是基於請求的URI，在此server中全部的非命名location中查找得來的。ngx_http_rewrite_module模塊也可能在NGX_HTTP_REWRITE_PHASE階段對請求的URI進行修改，這樣的話請求會重新發送回NGX_HTTP_FIND_CONFIG_PHASE階段使用新的URI進行location匹配。

也可以在任意時候通過對ngx_http_internal_redirect(r, uri, args)和ngx_http_named_location(r, name)函數進行調用來實現將請求重定向到一個新的location。

ngx_http_internal_redirect(r, uri, args)函數修改請求的URI並且將請求發送回NGX_HTTP_SERVER_REWRITE_PHASE階段。之後請求被分配到server默認的location上，然後在NGX_HTTP_FIND_CONFIG_PHASE階段根據請求新的URI來選擇location。

下面是一個同時帶有新的請求參數的內部重定向的例子。

```
ngx_int_t
ngx_http_foo_redirect(ngx_http_request_t *r)
{
    ngx_str_t  uri, args;

    ngx_str_set(&uri, "/foo");
    ngx_str_set(&args, "bar=1");

    return ngx_http_internal_redirect(r, &uri, &args);
}
```

The function ngx_http_named_location(r, name) redirects a request to a named location. The name of the location is passed as the argument. The location is looked up among all named locations of the current server, after which the requests switches to the NGX_HTTP_REWRITE_PHASE phase.

ngx_http_named_location(r, name)函數將請求重定向到一個命名location。目標location的名稱通過參數傳遞，並在當前server中的全部命名location中查找，接著請求會被發送到NGX_HTTP_REWRITE_PHASE階段。

下面是一個將請求重定向到命名location @foo的例子：

```
ngx_int_t
ngx_http_foo_named_redirect(ngx_http_request_t *r)
{
    ngx_str_t  name;

    ngx_str_set(&name, "foo");

    return ngx_http_named_location(r, &name);
}
```

當ngx_http_internal_redirect(r, uri, args)和ngx_http_named_location(r, name)這兩個函數被調用時，nginx模塊可能已經向HTTP請求的ctx成員中存儲了一些上下文。這些上下文在請求發生location切換之後可能會變得不一致。為了避免這種不一致性，所有的請求上下文會被這兩個函數清除。

被重定向以及被重寫的請求成為了內部請求進而可以訪問內部location。內部請求的internal標記位被設置為真。

子請求
-----

子請求主要用來將一個請求的輸出合併到另外一個請求中，很可能和其他數據混合。一個子請求看起來就像是一個普通的請求，但是和其父請求共享某些數據。具體來說，所有和客戶端輸入相關的數據都是共享的，因為子請求不從客戶端接收任何額外的數據。子請求的請求結構中的parent成員保存了指向其父請求的指針，如果是main request則此成員為空。成員main存儲了指向一組請求中main請求的指針。

子請求從NGX_HTTP_SERVER_REWRITE_PHASE階段開始。它經歷的其他階段和普通請求相同，並基於其URI來分配location。

子請求的輸出頭總是被忽略。子請求的輸出體通過ngx_http_postpone_filter插入到父請求產生的數據中的合適位置。

子請求和活動請求的概念相關。一個請求r被認為是活動的，如果c->data == r，c是表示nginx和客戶端連接的對象。在任意時候，只有一組請求中的活動請求才允許將其輸出緩衝發送給客戶端。一個非活動請求仍然可以將其數據發送到過濾鏈中，但是這些數據不會通過ngx_http_postpone_filter過濾並且數據會一直保留在這個過濾器中，直到請求變成活動狀態。下面是一些關於請求活動性的規則：

* 開始時，main請求是活動的
* 一個活動請求的第一個子請求在被創建之後立刻變為活動的
* 如果活動請求的子請求隊列上的下一個請求之前的數據都已經發送完，則ngx_http_postpone_filter會將此請求激活
* 當一個請求結束了，它的父請求變為活動請求

一個子請求是用過調用ngx_http_subrequest(r, uri, args, psr, ps, flags)函數來創建的，其中r是父請求，uri和args分別是子請求的URI和請求參數，psr是一個輸出參數，含有新創建的子請求的引用，ps是一個回調函數，用來在子請求結束的時候通知父請求，flags是子請求的創建標記位。有如下標記位可以使用：

* NGX_HTTP_SUBREQUEST_IN_MEMORY - 子請求的輸出不需要發送給客戶端，而是在內存中保留。此標記位只對代理子請求有效。在子請求結束後，它的輸出會以ngx_buf_t類型存放在r->upstream->buffer中。
* NGX_HTTP_SUBREQUEST_WAITED - 子請求的done標記位會被設置，即使當其結束時處於非活動狀態。這個標記位被SSI過濾器使用。
* NGX_HTTP_SUBREQUEST_CLONE - 子請求作為父請求的克隆來創建。如此創建的子請求將繼承父請求的location並從父請求所在的階段繼續執行。

下面的例子中創建了一個URI為"/foo"的子請求。

```
ngx_int_t            rc;
ngx_str_t            uri;
ngx_http_request_t  *sr;

...

ngx_str_set(&uri, "/foo");

rc = ngx_http_subrequest(r, &uri, NULL, &sr, NULL, 0);
if (rc == NGX_ERROR) {
    /* error */
}
```

這個例子是將當前請求進行克隆並為子請求設置了一個結束回調函數。

```
ngx_int_t
ngx_http_foo_clone(ngx_http_request_t *r)
{
    ngx_http_request_t          *sr;
    ngx_http_post_subrequest_t  *ps;

    ps = ngx_palloc(r->pool, sizeof(ngx_http_post_subrequest_t));
    if (ps == NULL) {
        return NGX_ERROR;
    }

    ps->handler = ngx_http_foo_subrequest_done;
    ps->data = "foo";

    return ngx_http_subrequest(r, &r->uri, &r->args, &sr, ps,
                               NGX_HTTP_SUBREQUEST_CLONE);
}


ngx_int_t
ngx_http_foo_subrequest_done(ngx_http_request_t *r, void *data, ngx_int_t rc)
{
    char  *msg = (char *) data;

    ngx_log_error(NGX_LOG_INFO, r->connection->log, 0,
                  "done subrequest r:%p msg:%s rc:%i", r, msg, rc);

    return rc;
}
```

子請求通常在body過濾器中創建。在這種情況下，子請求的輸出可以被當成任意的顯式請求輸出處理。這意味著子請求的輸出會在其他全部先於子請求創建的顯式緩衝之後，以及在除此之外的任何緩衝之前，發送給客戶端。這個順序對於大型的子請求層次結構也同樣有效。下面演示了將一個子請求插入到所有請求數據緩衝之後，但是在擁有last_buf的最後一個緩衝之前的例子。

```
ngx_int_t
ngx_http_foo_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    ngx_int_t                   rc;
    ngx_buf_t                  *b;
    ngx_uint_t                  last;
    ngx_chain_t                *cl, out;
    ngx_http_request_t         *sr;
    ngx_http_foo_filter_ctx_t  *ctx;

    ctx = ngx_http_get_module_ctx(r, ngx_http_foo_filter_module);
    if (ctx == NULL) {
        return ngx_http_next_body_filter(r, in);
    }

    last = 0;

    for (cl = in; cl; cl = cl->next) {
        if (cl->buf->last_buf) {
            cl->buf->last_buf = 0;
            cl->buf->last_in_chain = 1;
            cl->buf->sync = 1;
            last = 1;
        }
    }

    /* Output explicit output buffers */

    rc = ngx_http_next_body_filter(r, in);

    if (rc == NGX_ERROR || !last) {
        return rc;
    }

    /*
     * Create the subrequest.  The output of the subrequest
     * will automatically be sent after all preceding buffers,
     * but before the last_buf buffer passed later in this function.
     */

    if (ngx_http_subrequest(r, ctx->uri, NULL, &sr, NULL, 0) != NGX_OK) {
        return NGX_ERROR;
    }

    ngx_http_set_ctx(r, NULL, ngx_http_foo_filter_module);


    /* Output the final buffer with the last_buf flag */

    b = ngx_calloc_buf(r->pool);
    if (b == NULL) {
        return NGX_ERROR;
    }

    b->last_buf = 1;

    out.buf = b;
    out.next = NULL;

    return ngx_http_output_filter(r, &out);
}
```

一個子請求也可以為了輸出數據之外的目的而創建。例如，ngx_http_auth_request_module在NGX_HTTP_ACCESS_PHASE階段創建了一個子請求。為了在這個階段禁止任何輸出，子請求的header_only標誌被設置。這可以避免子請求的body被發送到客戶端。子請求的header無論如何都是被忽略的。子請求的結果可以通過回調handler來分析處理。

請求結束
-------

一個HTTP請求通過調用ngx_http_finalize_request(r, rc)來完成其生命週期。這通常是content handler在向過濾鏈發送完全部輸出數據後執行的。在這個時候，數據有可能還沒有全部發送到客戶端，而是其中一部分依然緩存在過濾鏈的某處。如果是這樣，ngx_http_finalize_request(r, rc)函數會自動註冊一個特殊的handlerngx_http_writer(r)來完成數據的發送。一個請求也可能是因為產生了某種錯誤或者因為標準的HTTP響應碼需要被返回給客戶端，而被終結。

ngx_http_finalize_request(r, rc)函數接受如下的rc參數值：

* NGX_DONE - 快速結束。減少請求引用計數並且如果為0的話就銷毀請求。和客戶端的連接可能會被繼續復用。
* NGX_ERROR, NGX_HTTP_REQUEST_TIME_OUT (408), NGX_HTTP_CLIENT_CLOSED_REQUEST (499) - 錯誤結束。盡可能快結束請求並關閉客戶端連接。
* NGX_HTTP_CREATED (201), NGX_HTTP_NO_CONTENT (204), 大於或等於 NGX_HTTP_SPECIAL_RESPONSE (300) - 特殊響應結束。對這些值nginx要麼發送默認代號響應頁面給客戶端，要麼根據error_page location執行內部重定向（如果配置了的話）。
* 其它值被認為是成功結束，並且可能激活請求writer完成發送響應體。一旦body發送完畢，請求計數就會遞減。如果到達0,則該請求會被銷毀，但是客戶端可能因為其它請求繼續被使用著。如果計數大於0, 則該請求內還有未完成的活動，它們將在後面被繼續完成。

請求體
------

為處理客戶端請求體，nginx提供了兩個函數：ngx_http_read_client_request_body(r, post_handler) 和 ngx_http_discard_request_body(r)。每一個函數讀請求體並且設到 request_body 字段。第二個函數指示nginx丟棄（讀和忽略）請求體。每個請求必須調用它們其中的一個。通常，這個在content階段完成。

讀或丟棄客戶端請求體不能在子請求里。這個需要在主請求里完成。當一個子請求創建時，如果父請求已經在前面讀了請求體，則子請求會繼承父的request_body以便使用。

函數 ngx_http_read_client_request_body(r, post_handler) 開始讀請求體的處理。當請求體完全讀取後，post_handler 回調函數會被調用以繼續處理請求。如果沒有請求體或已讀，則回調函數會立即被調用。函數 ngx_http_read_client_request_body(r, post_handler) 分配類型為ngx_http_request_body_t的request_body字段。該對象的bufs字段將結果保留為buffer chain。請求體可以保存在內存buffer，如果client_body_buffer_size不足於容納整個在內存的body時，則保存在文件buffer。

以下例子讀客戶端請求體並返回它的大小。

```
ngx_int_t
ngx_http_foo_content_handler(ngx_http_request_t *r)
{
    ngx_int_t  rc;

    rc = ngx_http_read_client_request_body(r, ngx_http_foo_init);

    if (rc >= NGX_HTTP_SPECIAL_RESPONSE) {
        /* error */
        return rc;
    }

    return NGX_DONE;
}


void
ngx_http_foo_init(ngx_http_request_t *r)
{
    off_t         len;
    ngx_buf_t    *b;
    ngx_int_t     rc;
    ngx_chain_t  *in, out;

    if (r->request_body == NULL) {
        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    len = 0;

    for (in = r->request_body->bufs; in; in = in->next) {
        len += ngx_buf_size(in->buf);
    }

    b = ngx_create_temp_buf(r->pool, NGX_OFF_T_LEN);
    if (b == NULL) {
        ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }

    b->last = ngx_sprintf(b->pos, "%O", len);
    b->last_buf = (r == r->main) ? 1: 0;
    b->last_in_chain = 1;

    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = b->last - b->pos;

    rc = ngx_http_send_header(r);

    if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
        ngx_http_finalize_request(r, rc);
        return;
    }

    out.buf = b;
    out.next = NULL;

    rc = ngx_http_output_filter(r, &out);

    ngx_http_finalize_request(r, rc);
}
```

以下請求的字段會影響請求體的讀取方式。

* request_body_in_single_buf - 將請求體讀到單一內存buffer。
* request_body_in_file_only - 始終將請求體讀到文件，即使適合內存緩衝區。
* request_body_in_persistent_file - 創建後不刪除該文件。這樣的文件可以被移到其它目錄。
* request_body_in_clean_file - 當請求結束時刪除該文件。當文件希望被移到其它目錄，但由於某種原因沒移走，這時該字段就派上用場了。
* request_body_file_group_access - 啓用文件組權限。默認情況文件以0600權限被創建。當該標記設置時，0660權限就被用上了。
* request_body_file_log_level - 記錄文件錯誤的日誌級別。
* request_body_no_buffering - 不緩衝的讀請求體。

當設置request_body_no_buffering這個標記，讀請求體的非緩衝模式就開啓了。這種模式下，調用完 ngx_http_read_client_request_body()之後，bufs鏈可能只保留請求體的一部份。要繼續讀下個部分，應該調用ngx_http_read_unbuffered_request_body(r) 函數。返回值為 NGX_AGAIN 並且設置了標記reading_body表明還有更多的數據可讀。如果調用該函數後 bufs 是 NULL，則說明此該沒有數據可讀。當請求體下個部份可用時，請求回調用函數 read_event_handler 回被調用。

響應
----

nginx里的HTTP響應是通過發送響應頭和接著可選的響應體產生的。兩者被傳進filter鏈里並且最終寫到客戶端socket。一個nginx模塊可以安裝它的handler到header或body filter里，並且處理來自上一個handler的輸出。

響應頭
------

通過函數 ngx_http_send_header(r) 發送輸出頭。在調用這個函數之前，r->headers_out 必須包含所有被用來發送HTTP響應頭的數據。r->headers_out的status字段通常是需要設置的。如果該響應狀態碼指示響應體應該接著頭部，content_length_n 也可以設置。該值默認是-1，表示響應體大小是未知的。這種情況下，就會用到chunked傳輸。想輸出任意的頭部，需要加到頭部列表裡。

```
static ngx_int_t
ngx_http_foo_content_handler(ngx_http_request_t *r)
{
    ngx_int_t         rc;
    ngx_table_elt_t  *h;

    /* send header */

    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = 3;

    /* X-Foo: foo */

    h = ngx_list_push(&r->headers_out.headers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    h->hash = 1;
    ngx_str_set(&h->key, "X-Foo");
    ngx_str_set(&h->value, "foo");

    rc = ngx_http_send_header(r);

    if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
        return rc;
    }

    /* send body */

    ...
}
```

頭部過濾
--------------

函數 ngx_http_send_header(r) 通過調用首個頭部filter handler ngx_http_top_header_filter 執行頭部filter鏈。它假設所有的header heandle會調用鏈里的下一個hanndler直到最後一個handler ngx_http_header_filter(r)。 這個最後的handler構造了基於 r->headers_out 的 HTTP 響應並且將它傳給 ngx_http_writer_filter 以作輸出。

要將一個handler添加到 header filter 鏈, 需要在配置階段將它的地址保存在 ngx_http_top_header_filter 這個全局變量。前一個handler的地址通常保存在模塊里的一個靜態變量，並且在退出前由新加入的handler調用。

以下是個header filter模塊的例子，對每個狀態是200的輸出都加個 "X-Foo: foo" 頭部信息。

```
#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>


static ngx_int_t ngx_http_foo_header_filter(ngx_http_request_t *r);
static ngx_int_t ngx_http_foo_header_filter_init(ngx_conf_t *cf);


static ngx_http_module_t  ngx_http_foo_header_filter_module_ctx = {
    NULL,                                   /* preconfiguration */
    ngx_http_foo_header_filter_init,        /* postconfiguration */

    NULL,                                   /* create main configuration */
    NULL,                                   /* init main configuration */

    NULL,                                   /* create server configuration */
    NULL,                                   /* merge server configuration */

    NULL,                                   /* create location configuration */
    NULL                                    /* merge location configuration */
};


ngx_module_t  ngx_http_foo_header_filter_module = {
    NGX_MODULE_V1,
    &ngx_http_foo_header_filter_module_ctx, /* module context */
    NULL,                                   /* module directives */
    NGX_HTTP_MODULE,                        /* module type */
    NULL,                                   /* init master */
    NULL,                                   /* init module */
    NULL,                                   /* init process */
    NULL,                                   /* init thread */
    NULL,                                   /* exit thread */
    NULL,                                   /* exit process */
    NULL,                                   /* exit master */
    NGX_MODULE_V1_PADDING
};


static ngx_http_output_header_filter_pt  ngx_http_next_header_filter;


static ngx_int_t
ngx_http_foo_header_filter(ngx_http_request_t *r)
{
    ngx_table_elt_t  *h;

    /*
     * The filter handler adds "X-Foo: foo" header
     * to every HTTP 200 response
     */

    if (r->headers_out.status != NGX_HTTP_OK) {
        return ngx_http_next_header_filter(r);
    }

    h = ngx_list_push(&r->headers_out.headers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    h->hash = 1;
    ngx_str_set(&h->key, "X-Foo");
    ngx_str_set(&h->value, "foo");

    return ngx_http_next_header_filter(r);
}


static ngx_int_t
ngx_http_foo_header_filter_init(ngx_conf_t *cf)
{
    ngx_http_next_header_filter = ngx_http_top_header_filter;
    ngx_http_top_header_filter = ngx_http_foo_header_filter;

    return NGX_OK;
}
```

響應體
------

通過函數 ngx_http_output_filter(r, cl) 發響應體。該函數能被調用多次。每次它會發送作為buffer鏈的響應體的一部份。最後的body buffer應該有設置last_buf標記。

以下例子產生一個完整的HTTP輸出 "foo" 作為響應體。為了讓這個例子不止能在主請求運行，也在子請求能運行。輸出的最後buffer會設置 last_in_chain 標記。標記 last_buf 只會對主請求設置，因為子請求的最後buffer不會作為整個輸出的結束。

```
static ngx_int_t
ngx_http_bar_content_handler(ngx_http_request_t *r)
{
    ngx_int_t     rc;
    ngx_buf_t    *b;
    ngx_chain_t   out;

    /* send header */

    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = 3;

    rc = ngx_http_send_header(r);

    if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
        return rc;
    }

    /* send body */

    b = ngx_calloc_buf(r->pool);
    if (b == NULL) {
        return NGX_ERROR;
    }

    b->last_buf = (r == r->main) ? 1: 0;
    b->last_in_chain = 1;

    b->memory = 1;

    b->pos = (u_char *) "foo";
    b->last = b->pos + 3;

    out.buf = b;
    out.next = NULL;

    return ngx_http_output_filter(r, &out);
}
```

響應體過濾
------------

函數 ngx_http_output_filter(r, cl) 通過調用首個body filter handler ngx_http_top_body_filter 執行響應體過濾鏈。它假定每個body handler會調用鏈里的下一個handler直到最後的handler ngx_http_write_filter(r, cl) 被調用。

body filter handler會接收一個buffer鏈。這個handler會處理 buffers 並且傳可能新的chain給下個handler。值得注意的是，傳入的ngx_chain_t鏈接屬於調用者。它們不用被復用或者改變。當handler完成後，調用者可以用它的輸出鏈來跟蹤其發送的buffer。如果想保存buffer chain或替換一些繼續要發送的buffer，該handler應該分配它自己的鏈。

以下是一個簡單的計算響應體大小的body模塊。結果作為 $counter 變量可以被用在 access 日誌。

```
#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>


typedef struct {
    off_t  count;
} ngx_http_counter_filter_ctx_t;


static ngx_int_t ngx_http_counter_body_filter(ngx_http_request_t *r,
    ngx_chain_t *in);
static ngx_int_t ngx_http_counter_variable(ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data);
static ngx_int_t ngx_http_counter_add_variables(ngx_conf_t *cf);
static ngx_int_t ngx_http_counter_filter_init(ngx_conf_t *cf);


static ngx_http_module_t  ngx_http_counter_filter_module_ctx = {
    ngx_http_counter_add_variables,        /* preconfiguration */
    ngx_http_counter_filter_init,          /* postconfiguration */

    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */

    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */

    NULL,                                  /* create location configuration */
    NULL                                   /* merge location configuration */
};


ngx_module_t  ngx_http_counter_filter_module = {
    NGX_MODULE_V1,
    &ngx_http_counter_filter_module_ctx,   /* module context */
    NULL,                                  /* module directives */
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


static ngx_http_output_body_filter_pt  ngx_http_next_body_filter;

static ngx_str_t  ngx_http_counter_name = ngx_string("counter");


static ngx_int_t
ngx_http_counter_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    ngx_chain_t                    *cl;
    ngx_http_counter_filter_ctx_t  *ctx;

    ctx = ngx_http_get_module_ctx(r, ngx_http_counter_filter_module);
    if (ctx == NULL) {
        ctx = ngx_pcalloc(r->pool, sizeof(ngx_http_counter_filter_ctx_t));
        if (ctx == NULL) {
            return NGX_ERROR;
        }

        ngx_http_set_ctx(r, ctx, ngx_http_counter_filter_module);
    }

    for (cl = in; cl; cl = cl->next) {
        ctx->count += ngx_buf_size(cl->buf);
    }

    return ngx_http_next_body_filter(r, in);
}


static ngx_int_t
ngx_http_counter_variable(ngx_http_request_t *r, ngx_http_variable_value_t *v,
    uintptr_t data)
{
    u_char                         *p;
    ngx_http_counter_filter_ctx_t  *ctx;

    ctx = ngx_http_get_module_ctx(r, ngx_http_counter_filter_module);
    if (ctx == NULL) {
        v->not_found = 1;
        return NGX_OK;
    }

    p = ngx_pnalloc(r->pool, NGX_OFF_T_LEN);
    if (p == NULL) {
        return NGX_ERROR;
    }

    v->data = p;
    v->len = ngx_sprintf(p, "%O", ctx->count) - p;
    v->valid = 1;
    v->no_cacheable = 0;
    v->not_found = 0;

    return NGX_OK;
}


static ngx_int_t
ngx_http_counter_add_variables(ngx_conf_t *cf)
{
    ngx_http_variable_t  *var;

    var = ngx_http_add_variable(cf, &ngx_http_counter_name, 0);
    if (var == NULL) {
        return NGX_ERROR;
    }

    var->get_handler = ngx_http_counter_variable;

    return NGX_OK;
}


static ngx_int_t
ngx_http_counter_filter_init(ngx_conf_t *cf)
{
    ngx_http_next_body_filter = ngx_http_top_body_filter;
    ngx_http_top_body_filter = ngx_http_counter_body_filter;

    return NGX_OK;
}
```

構建過濾模塊
------------

當寫一個body或header過濾模塊是，要特別注意filter的順序。已經有一些已經註冊的標準nginx模塊。註冊一個filter模塊到相對其它模塊的正確位置是很重要的。通常filter會在模塊自己的postconfiguration handler里註冊。filter的調用順序跟它們的註冊時的順序剛好相反。

nginx給第三方模塊提供了個特殊的槽口 HTTP_AUX_FILTER_MODULES。想在這個插槽註冊一個filter模塊，模塊的配置里應該將 ngx_module_type 變量設置值為 HTTP_AUX_FILTER。

以下例子顯示一個filter模塊的配置文件，並且假設只有一個源文件 ngx_http_foo_filter_module.c。

```
ngx_module_type=HTTP_AUX_FILTER
ngx_module_name=ngx_http_foo_filter_module
ngx_module_srcs="$ngx_addon_dir/ngx_http_foo_filter_module.c"

. auto/module
```

緩衝復用
--------

當處理或更改緩衝區流時，經常需要復用已分配的buffer。nginx代碼里比較標準通用的處理方式是保留兩個buffer鏈：free and busy。 free 鏈保留所有空閒的 buffer。這些buffer可以拿來復用。busy 鏈保存所有當前模塊發送的buffer，但仍然被其它的filter handler使用。如果它的大小大於0,則認為該buffer還在使用。通常一個buffer被一個filter消費時，它的pos（或file_pos對文件buffer而言）會移向last （或file_pos對文件buffer而言）。一旦整個buffer被完全消費完，它就可以復用了。為將新空閒的buffer更新到空閒chain，需要完整的遍歷busy鏈，並將大小為0的buffer移到free的首部。這種操作很常見，所以有個特殊的函數 ngx_chain_update_chains(free, busy, out, tag) 專門處理這個。這個函數追加output chain到busy，並且將空閒的buffer從busy移到free。只有匹配tag的buffer才能復用。這樣就讓一個模塊只能復用它自己分配的buffer。

以下例子為每個新進的buffer加入字串 "foo"。該模塊盡可能的復用這些新分配的buffer。注意：為了讓該例子運行的沒問題，需要安裝header filter，並且將 content_length_n 設置為-1 （這節上面有提到）。

```
typedef struct {
    ngx_chain_t  *free;
    ngx_chain_t  *busy;
}  ngx_http_foo_filter_ctx_t;


ngx_int_t
ngx_http_foo_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    ngx_int_t                   rc;
    ngx_buf_t                  *b;
    ngx_chain_t                *cl, *tl, *out, **ll;
    ngx_http_foo_filter_ctx_t  *ctx;

    ctx = ngx_http_get_module_ctx(r, ngx_http_foo_filter_module);
    if (ctx == NULL) {
        ctx = ngx_pcalloc(r->pool, sizeof(ngx_http_foo_filter_ctx_t));
        if (ctx == NULL) {
            return NGX_ERROR;
        }

        ngx_http_set_ctx(r, ctx, ngx_http_foo_filter_module);
    }

    /* create a new chain "out" from "in" with all the changes */

    ll = &out;

    for (cl = in; cl; cl = cl->next) {

        /* append "foo" in a reused buffer if possible */

        tl = ngx_chain_get_free_buf(r->pool, &ctx->free);
        if (tl == NULL) {
            return NGX_ERROR;
        }

        b = tl->buf;
        b->tag = (ngx_buf_tag_t) &ngx_http_foo_filter_module;
        b->memory = 1;
        b->pos = (u_char *) "foo";
        b->last = b->pos + 3;

        *ll = tl;
        ll = &tl->next;

        /* append the next incoming buffer */

        tl = ngx_alloc_chain_link(r->pool);
        if (tl == NULL) {
            return NGX_ERROR;
        }

        tl->buf = cl->buf;
        *ll = tl;
        ll = &tl->next;
    }

    *ll = NULL;

    /* send the new chain */

    rc = ngx_http_next_body_filter(r, out);

    /* update "busy" and "free" chains for reuse */

    ngx_chain_update_chains(r->pool, &ctx->free, &ctx->busy, &out,
                            (ngx_buf_tag_t) &ngx_http_foo_filter_module);

    return rc;
}
```

負載均衡
-------
ngx_http_upstream_module提供了向遠程服務器發送HTTP請求的基本功能。其他具體的協議模塊，例如HTTP或FastCDI，都會使用這個功能。該模塊同時還提供了可以定制負載均衡算法的接口並默認實現了round-robin（輪詢）算法

例如，提供其他的負載均衡算法的模塊有least_conn和hash這些。需要注意的是，這些模塊實際上是作為upstream模塊的擴展而實現的，他們之間共享了大量的代碼，比如對於服務器組的表示。keepalive模塊是另外一個例子，這是一個獨立的模塊，擴展了upstream的功能。

ngx_http_upstream_module可以通過在配置文件中配置upstream塊來顯式配置，或者通過使用可以接受URL作為參數的指令來隱式開啓，比如proxy_pass這種指令。只有顯示的配置才能選擇負載均衡算法。upstream模塊有自己的指令上下文NGX_HTTP_UPS_CONF。相關結構體定義如下：

```
struct ngx_http_upstream_srv_conf_s {
    ngx_http_upstream_peer_t         peer;
    void                           **srv_conf;

    ngx_array_t                     *servers;  /* ngx_http_upstream_server_t */

    ngx_uint_t                       flags;
    ngx_str_t                        host;
    u_char                          *file_name;
    ngx_uint_t                       line;
    in_port_t                        port;
    ngx_uint_t                       no_port;  /* unsigned no_port:1 */

#if (NGX_HTTP_UPSTREAM_ZONE)
    ngx_shm_zone_t                  *shm_zone;
#endif
};
```

* srv_conf — upstream模塊的配置上下文
* servers — ngx_http_upstream_server_t的陣列，存放的是對upstream塊中一組server指令解析的配置
* flags — 指定特定負載均衡算法支持哪些特性（通過server指令的參數配置）的標記位。
   * NGX_HTTP_UPSTREAM_CREATE — 用來區分顯式定義的upstream和通過proxy_pass類型指令(FastCGI, SCGI等)隱式創建的upstream
   * NGX_HTTP_UPSTREAM_WEIGHT — 支持「weight」
   * NGX_HTTP_UPSTREAM_MAX_FAILS — 支持「max_fails」
   * NGX_HTTP_UPSTREAM_FAIL_TIMEOUT — 支持「fail_timeout」
   * NGX_HTTP_UPSTREAM_DOWN — 支持「down」
   * NGX_HTTP_UPSTREAM_BACKUP — 支持「backup」
   * NGX_HTTP_UPSTREAM_MAX_CONNS — 支持「max_conns」
* host — upstream的名字
* file_name, line — 配置文件名字以及upstream塊所在行
* port and no_port — 顯式upstream未使用
* shm_zone — 此upstream使用的共享內存
* peer — 存放用來初始化upstream配置通用方法的對象：

```
typedef struct {
    ngx_http_upstream_init_pt        init_upstream;
    ngx_http_upstream_init_peer_pt   init;
    void                            *data;
} ngx_http_upstream_peer_t;
```

實現負載均衡算法的模塊必須設置這些方法並初始化私有數據。
如果init_upstream在配置階段沒有初始化，ngx_http_upstream_module會將其默認設置成ngx_http_upstream_init_round_robin。

   * init_upstream(cf, us) — 配置階段方法，用於初始化一組服務器並初始化init()方法。一個典型的負載均衡模塊使用upstream塊中的一組服務器來創建某種有效的數據結構並在data成員中存放自身的配置。
   * init(r, us) — 初始化用於每個請求的ngx_http_upstream_t.peer (不要和之前用於每個upstream的ngx_http_upstream_srv_conf_t.peer搞混了)結構，該結構用於進行負載均衡。該結構會作為所有處理服務器選擇的回調函數的data參數傳遞。

當nginx需要將請求轉給其他服務器進行處理時，它會調用配置好的負載均衡算法來選擇一個地址，併發起連接。選擇算法是從ngx_http_upstream_t.peer對象中獲取的，該對象的類型是ngx_peer_connection_t：

```
struct ngx_peer_connection_s {
    [...]

    struct sockaddr                 *sockaddr;
    socklen_t                        socklen;
    ngx_str_t                       *name;

    ngx_uint_t                       tries;

    ngx_event_get_peer_pt            get;
    ngx_event_free_peer_pt           free;
    ngx_event_notify_peer_pt         notify;
    void                            *data;

#if (NGX_SSL || NGX_COMPAT)
    ngx_event_set_peer_session_pt    set_session;
    ngx_event_save_peer_session_pt   save_session;
#endif

    [..]
};
```

這個結構體有如下成員：

* sockaddr, socklen, name — 待連接的upstream服務器的地址；此為負載均衡算法的輸出參數
* data — 每請求的負載均衡算法所需數據；記錄選擇算法的狀態並且通常會含有指向upstream配置的指針。此data會被作為參數傳遞給所有處理服務器選擇的函數（見下文）
* tries — 連接upstream服務器的重試次數
* get, free, notify, set_session, and save_session - 負載均衡算法模塊的方法，詳細見下文

所有的方法至少接受兩個參數：peer連接對象pc以及由ngx_http_upstream_srv_conf_t.peer.init()創建的data參數。注意，一般來說，由於負載均衡算法的」chaining」，這個data和pc.data是不同的，

* get(pc, data) — 當upstream模塊需要將請求發送給一個服務器而需要知道服務器地址的時候，該方法會被調用。該方法負責填寫ngx_peer_connection_t結構的sockaddr，socklen和name成員。返回值有如下幾種：
   * NGX_OK — 服務器已選擇
   * NGX_ERROR — 發生了內部錯誤
   * NGX_BUSY — 當前沒有可用服務器。有多種原因會導致這個情況的發生，例如：動態服務器組為空，全部服務器均為失敗狀態，全部服務器已經達到最大連接數或者其他類似情況。
   * NGX_DONE — keepalive模塊用這個返回值來說明底層連接進行了復用，因此不需要和upstream服務器間創建一條新連接。
* free(pc, data, state) — 當upstream模塊同某個upstream服務器通信結束後，調用此方法。state參數指示了upstream連接的完成狀態，是一個bitmask，可以被設置成這些值：NGX_PEER_FAILED - 失敗，NGX_PEER_NEXT - 403和404的特殊情況，不作為失敗對待，NGX_PEER_KEEPALIVE。此外，嘗試次數也在這個方法遞減。
* notify(pc, data, type) — 開源版本中未使用。
* set_session(pc, data)和save_session(pc, data) — SSL相關方法，用於緩存同upstream服務器間的SSL會話，由round-robin負載均衡算法實現。


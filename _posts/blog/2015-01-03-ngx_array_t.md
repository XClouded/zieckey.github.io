---
layout: post
title: Nginx源码研究（3）——Nginx数组ngx_array_t和示例
description: 本文主要介绍Nginx数组ngx_array_t这一重要的数据结构的使用方法和具体实现。
category: blog
tags : [Nginx, 网络编程]
---

本文主要介绍Nginx数组`ngx_array_t`这一重要的数据结构的使用方法和具体实现。

`ngx_array_t`是nginx内部使用的数组结构。nginx的数组结构在存储上与大家认知的C语言内置的数组有相似性，比如实际上存储数据的区域也是一大块连续的内存。但是数组除了存储数据的内存以外还包含一些元信息来描述相关的一些信息。ngx_array_t的定义位于`src/core/ngx_array.{c,h}`里面。

ngx_array.h实现和注释如下：

```C
#include <ngx_config.h>
#include <ngx_core.h>

// 动态数组
struct ngx_array_s {
    // elts指向数组的首地址
    void        *elts; 
    // nelts是数组中已经使用的元素个数
    ngx_uint_t   nelts; 
    // 每个数组元素占用的内存大小
    size_t       size;  
    // 当前数组中能够容纳元素个数的总大小
    ngx_uint_t   nalloc; 
    // 内存池对象
    ngx_pool_t  *pool;  
};

/*
从内存池中创建n个元素的数组，元素大小为size
创建一个新的数组对象，并返回这个对象。

p:	数组分配内存使用的内存池；
n:	数组的初始容量大小，即在不扩容的情况下最多可以容纳的元素个数。
size:	单个元素的大小，单位是字节。

注意事项: 由于使用ngx_palloc分配内存，数组在扩容时，旧的内存不会被释放，会造成内存的浪费。
因此，最好能提前规划好数组的容量，在创建或者初始化的时候一次搞定，避免多次扩容，造成内存浪费。
 */
ngx_array_t *ngx_array_create(ngx_pool_t *p, ngx_uint_t n, size_t size);

// 销毁该数组对象，并释放其分配的内存回内存池。
void ngx_array_destroy(ngx_array_t *a);

// 在数组a上新追加一个元素，并返回指向新元素的指针。
// 需要把返回的指针使用类型转换，转换为具体的类型，然后再给新元素本身或者是各字段（如果数组的元素是复杂类型）赋值。
// 如果数组已满，则重新分配两倍（nalloc*size)的内存空间，且nalloc更新为2*nalloc
void *ngx_array_push(ngx_array_t *a);

// 返回将要添加n个元素到数组中其首个元素的地址
void *ngx_array_push_n(ngx_array_t *a, ngx_uint_t n);


// 如果一个数组对象是被分配在堆上的，那么当调用ngx_array_destroy销毁以后，如果想再次使用，就可以调用此函数。
// 如果一个数组对象是被分配在栈上的，那么就需要调用此函数，进行初始化的工作以后，才可以使用。
static ngx_inline ngx_int_t
ngx_array_init(ngx_array_t *array, ngx_pool_t *pool, ngx_uint_t n, size_t size)
{
    /*
     * set "array->nelts" before "array->elts", otherwise MSVC thinks
     * that "array->nelts" may be used without having been initialized
     */

    array->nelts = 0;
    array->size = size;
    array->nalloc = n;
    array->pool = pool;

    array->elts = ngx_palloc(pool, n * size);
    if (array->elts == NULL) {
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

测试代码，完整的工程编译请参考 [https://github.com/zieckey/nginx-research项目](https://github.com/zieckey/nginx-research)

```CPP
#include "allinc.h"

struct User {
    ngx_str_t name;
    ngx_str_t url;
    int id;
};

/*
* 
*/
TEST_UNIT(ngx_array_t) {
    ngx_array_t* a = ngx_array_create(g_pool, 10, sizeof(User));

    H_TEST_ASSERT(a->nalloc == 10);
    H_TEST_ASSERT(a->size == sizeof(User));

    // Add one element to the array
    User* u = (User*)ngx_array_push(a);
    u->name.data = (u_char*)ngx_pcalloc(g_pool, 32);
    strcpy((char*)u->name.data, "CodeG");
    u->name.len = strlen("CodeG");
    ngx_str_set(&u->url, "http://blog.codeg.cn/2015/01/03/ngx_array_t");
    u->id = 0;
    H_TEST_ASSERT(a->nelts == 1);
    H_TEST_ASSERT(a->elts == u); // elts指向数组的首地址，因此与第一个数组元素地址相同

    // Add another one element to the array
    u = (User*)ngx_array_push(a);
    u->name.data = (u_char*)ngx_pcalloc(g_pool, 32);
    strcpy((char*)u->name.data, "zieckey");
    u->name.len = strlen("zieckey");
    ngx_str_set(&u->url, "http://blog.codeg.cn/2014/12/13/Hello-CodeG");
    u->id = 1;
    H_TEST_ASSERT(a->nelts == 2);

    // Add 3rd element to the array
    u = (User*)ngx_array_push(a);
    u->name.data = (u_char*)ngx_pcalloc(g_pool, 32);
    strcpy((char*)u->name.data, "zieckey");
    u->name.len = strlen("zieckey");
    ngx_str_set(&u->url, "https://github.com/zieckey");
    u->id = 2;
    H_TEST_ASSERT(a->nelts == 3);

    // Traversal the array
    for (ngx_uint_t i = 0; i < a->nelts; ++i) {
        u = (User*)((char*)a->elts + sizeof(User)*i);
        H_TEST_ASSERT(u->id == i);
        printf("id=%d name=[%s] url=[%s]\n", u->id, (char*)u->name.data, (char*)u->url.data);
    }
}

```

上述代码输出如下：

	id=0 name=[CodeG] url=[http://blog.codeg.cn/2015/01/03/ngx_array_t]
	id=1 name=[zieckey] url=[http://blog.codeg.cn/2014/12/13/Hello-CodeG]
	id=2 name=[zieckey] url=[https://github.com/zieckey]

### 参考：

- [淘宝：Nginx开发从入门到精通 http://tengine.taobao.org/book/chapter_02.html#ngx-array-t-100](http://tengine.taobao.org/book/chapter_02.html#ngx-array-t-100)
- [nginx源码注释 https://github.com/zieckey/nginx-1.0.14_comment/blob/master/src/core/ngx_array.h](https://github.com/zieckey/nginx-1.0.14_comment/blob/master/src/core/ngx_array.h)
- [nginx源码注释 https://github.com/zieckey/nginx-1.0.14_comment/blob/master/src/core/ngx_array.c](https://github.com/zieckey/nginx-1.0.14_comment/blob/master/src/core/ngx_array.c)
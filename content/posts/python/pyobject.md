---
title: "浅谈 Python 的对象实现（PyObject）"
date: 2024-03-30T12:00:00+08:00
draft: false
categories:
- Python
tags:
- Python
- PyObject
---


# PyObject

Everything is object, except keywords

> 所有的都是对象，除了关键字

{{<collapse summary="Include/object.h" openByDefault=true >}}
```c
/* PyObject_HEAD 是所有 OBJECT 的初始化段 */
#define PyObject_HEAD PyObject ob_base;
typedef struct _object PyObject;

struct _object {
    Py_ssize_t ob_refcnt;
    PyTypeObject *ob_type;
};

/* Py_GIL_DISABLED （无 GIL 锁的版本），添加了额外的 mutex */
struct _object {
    uintptr_t ob_tid;
    uint16_t _padding;
    struct _PyMutex ob_mutex;   // per-object lock
    uint8_t ob_gc_bits;         // gc-related state
    uint32_t ob_ref_local;      // local reference count
    Py_ssize_t ob_ref_shared;   // shared (atomic) reference count
    PyTypeObject *ob_type;
};
```
{{</collapse>}}

**ob_refcnt**：引用计数器
**ob_type**：类型对象的指针，指向描述该如何操作此类型的类型对象

> this->data 使用 this->ob_type 的**处理函数**, 处理函数被存放在 this->ob_type->data 中，因此可以在运行时改变类型的属性（处理函数）

我们先来看下 float 类型的数据结构

{{<collapse summary="Include/cpython/floatobject.h" openByDefault=true >}}
```c
/* FILE:  */
typedef struct {
    PyObject_HEAD // ob_type = &PyFloat_Type
    double ob_fval; // 数据段
} PyFloatObject;
```
{{</collapse>}}

如下是阐述了该如何操作 float 类型的类型对象

{{<collapse summary="Objects/floatobject.c">}}
```c
static PyNumberMethods float_as_number = {
    float_add,          /* nb_add */
    float_sub,          /* nb_subtract */
    float_mul,          /* nb_multiply */
    float_rem,          /* nb_remainder */
    float_divmod,       /* nb_divmod */
    float_pow,          /* nb_power */
    (unaryfunc)float_neg, /* nb_negative */
    float_float,        /* nb_positive */
    (unaryfunc)float_abs, /* nb_absolute */
    (inquiry)float_bool, /* nb_bool */
    0,                  /* nb_invert */
    0,                  /* nb_lshift */
    0,                  /* nb_rshift */
    0,                  /* nb_and */
    0,                  /* nb_xor */
    0,                  /* nb_or */
    float___trunc___impl, /* nb_int */
    0,                  /* nb_reserved */
    float_float,        /* nb_float */
    0,                  /* nb_inplace_add */
    0,                  /* nb_inplace_subtract */
    0,                  /* nb_inplace_multiply */
    0,                  /* nb_inplace_remainder */
    0,                  /* nb_inplace_power */
    0,                  /* nb_inplace_lshift */
    0,                  /* nb_inplace_rshift */
    0,                  /* nb_inplace_and */
    0,                  /* nb_inplace_xor */
    0,                  /* nb_inplace_or */
    float_floor_div,    /* nb_floor_divide */
    float_div,          /* nb_true_divide */
    0,                  /* nb_inplace_floor_divide */
    0,                  /* nb_inplace_true_divide */
};

PyTypeObject PyFloat_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "float",
    sizeof(PyFloatObject),
    0,
    (destructor)float_dealloc,                  /* tp_dealloc */
    0,                                          /* tp_vectorcall_offset */
    0,                                          /* tp_getattr */
    0,                                          /* tp_setattr */
    0,                                          /* tp_as_async */
    (reprfunc)float_repr,                       /* tp_repr */
    &float_as_number,                           /* tp_as_number */
    0,                                          /* tp_as_sequence */
    0,                                          /* tp_as_mapping */
    (hashfunc)float_hash,                       /* tp_hash */
    0,                                          /* tp_call */
    0,                                          /* tp_str */
    PyObject_GenericGetAttr,                    /* tp_getattro */
    0,                                          /* tp_setattro */
    0,                                          /* tp_as_buffer */
    Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE |
        _Py_TPFLAGS_MATCH_SELF,               /* tp_flags */
    float_new__doc__,                           /* tp_doc */
    0,                                          /* tp_traverse */
    0,                                          /* tp_clear */
    float_richcompare,                          /* tp_richcompare */
    0,                                          /* tp_weaklistoffset */
    0,                                          /* tp_iter */
    0,                                          /* tp_iternext */
    float_methods,                              /* tp_methods */
    0,                                          /* tp_members */
    float_getset,                               /* tp_getset */
    0,                                          /* tp_base */
    0,                                          /* tp_dict */
    0,                                          /* tp_descr_get */
    0,                                          /* tp_descr_set */
    0,                                          /* tp_dictoffset */
    0,                                          /* tp_init */
    0,                                          /* tp_alloc */
    float_new,                                  /* tp_new */
    .tp_vectorcall = (vectorcallfunc)float_vectorcall,
};
```
{{</collapse>}}

进入 float_new 函数，表示该如何 new 一个 float 类型

{{<collapse summary="float_new_impl" openByDefault=true >}}
```c
// with clinic，这是一个生成 C-Api 的预处理器，拓展成 float_new
static PyObject *float_new_impl(PyTypeObject *type, PyObject *x) {
  if (type != &PyFloat_Type) {
    if (x == NULL) {
      x = _PyLong_GetZero();
    }
    return float_subtype_new(type, x); /* Wimp out */
  }

  if (x == NULL) {
    return PyFloat_FromDouble(0.0);
  }
  /* If it's a string, but not a string subclass, use
     PyFloat_FromString. */
  if (PyUnicode_CheckExact(x))
    return PyFloat_FromString(x);
  return PyNumber_Float(x);
}
```
{{</collapse>}}

对于 tp_as_number 数值化 tp，则是包含了所有对数值操作的处理函数

例如数值相加 float_add

{{<collapse summary="Objects/floatobject.c" openByDefault=true >}}
```c
static PyObject *float_add(PyObject *v, PyObject *w) {
  double a, b;
  CONVERT_TO_DOUBLE(v, a);
  CONVERT_TO_DOUBLE(w, b);
  a = a + b;
  return PyFloat_FromDouble(a);
}

/* ForExample */

PyFloatObject num;

num->ob_fval // 代表具体的数据
```
{{</collapse>}}

同样，像数组这种多个相同类型的不定长**ListType**，python定义了PyObject_VAR_HEAD

{{<collapse summary="Include/object.h" openByDefault=true >}}
```c
#define PyObject_VAR_HEAD PyVarObject ob_base;
typedef struct {
    PyObject ob_base;
    Py_ssize_t ob_size; /* 可变变量数目 */
} PyVarObject;

/* FILE: cpython/Include/cpython/listobject.h */
typedef struct {
    PyObject_VAR_HEAD
    /* Vector of pointers to list elements.  list[0] is ob_item[0], etc. */
    PyObject **ob_item; /* 指向的连续指针数组的二级指针，队首对象为*(ob_item[0])，类推 */
    Py_ssize_t allocated; /* ob_item指向的指针数组的容量 */
} PyListObject;
```
{{</collapse>}}

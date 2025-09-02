---
title: "浅谈Python的对象实现（PyObject）"
date: 2024-03-30T03:31:04+08:00
draft: false
---


# PyObject

Everything is object, except keywords

> 所有的都是对象，除了关键字

```c
/* FILE: cpython/Include/object.h */
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

**ob_refcnt**即引用计数器，**ob_type**是类型对象的指针，这里的ob_type指向了一个描述了该如何操作此类型的类型对象

> [this->data]使用[this->ob_type]的**处理函数**, 而**处理函数**则是this->ob_type->data里的数据，因而可以随意在运行时改变类型的属性（处理函数），相当于是改变上一层对象的数据

先来看看一个float类型的数据结构

```c
/* FILE: cpython/Include/cpython/floatobject.h */
typedef struct {
    PyObject_HEAD // ob_type = &PyFloat_Type
    double ob_fval; // 数据段
} PyFloatObject;
```

如下是阐述了操作float类型的类型对象

```c
/* FILE: cpython/Objects/floatobject.c */
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

随便挑个**float_new**看看，这个函数表示该如何new一个float类型

```c
// with clinic，这是一个生成 C-Api 的预处理器，拓展成float_new
static PyObject *
float_new_impl(PyTypeObject *type, PyObject *x)
{
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

类似的，对于**tp_as_number**数值化tp，里面包含了一套对于数值操作的所有**处理函数**，对于数值相加，则如**float_add**

```c
/* FILE: cpython/Objects/floatobject.c */
static PyObject *
float_add(PyObject *v, PyObject *w)
{
    double a,b;
    CONVERT_TO_DOUBLE(v, a);
    CONVERT_TO_DOUBLE(w, b);
    a = a + b;
    return PyFloat_FromDouble(a);
}

/* ForExample */

PyFloatObject num;

num->ob_fval // 代表具体的数据
```

同样，像数组这种多个相同类型的不定长**ListType**，python定义了PyObject_VAR_HEAD

```c
/* FILE: cpython/Include/object.h */
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

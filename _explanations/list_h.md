---
layout: page
title:  An in-depth Explanation - list.h
---
For many kernel operations, it is neccessary to have a linked list data structure to keep track of everything.  This is an implementation of a linked list that reimplements the list for each new type you want a linked list for.

## list.h source

Here is the source for list.h:
``` c
fndef LIST_H
#define LIST_H

#define DEFINE_LIST(nodeType) \
typedef struct nodeType##list { \
    struct nodeType * head; \
    struct nodeType * tail; \
    uint32_t size;\
} nodeType##_list_t;

#define DEFINE_LINK(nodeType) \
struct nodeType * next##nodeType;\
struct nodeType * prev##nodeType;

#define INITIALIZE_LIST(list) \
    list.head = list.tail = (void *)0;\
    list.size = 0;

#define IMPLEMENT_LIST(nodeType) \
void append_##nodeType##_list(nodeType##_list_t * list, struct nodeType * node) {  \
    list->tail->next##nodeType = node;                                       \
    node->prev##nodeType = list->tail;                                       \
    list->tail = node;                                                       \
    node->next##nodeType = NULL;                                             \
    list->size += 1;                                                         \
}                                                                            \
                                                                             \
void push_##nodeType##_list(nodeType##_list_t * list, struct nodeType * node) {    \
    node->next##nodeType = list->head;                                       \
    node->prev##nodeType = NULL;                                             \
    list->head = node;                                                       \
    list->size += 1;                                                         \
}                                                                            \
                                                                             \
struct nodeType * peek_##nodeType##_list(nodeType##_list_t * list) {         \
    return list->head;                                                       \
}                                                                            \
                                                                             \
struct nodeType * pop_##nodeType##_list(nodeType##_list_t * list) {          \
    struct nodeType * res = list->head;                                      \
    list->head = list->head->next##nodeType;                                 \
    list->head->prev##nodeType = NULL;                                                 \
    list->size -= 1;                                                         \
    return res;                                                              \
}                                                                            \
                                                                             \
uint32_t size_##nodeType##_list(nodeType##_list_t * list) {                  \
    return list->size;                                                       \
}                                                                            \
                                                                             \
struct nodeType * next_##nodeType##_list(struct nodeType * node) {           \
    return node->next##nodeType;                                             \
}                                                                            \

#endif
```

## Usage
If you want a linked list of a certain type, inside the type you shouldput `DEFINE_LINK(typename)`.  Then after the type is defined, you should put `DEFINE_LIST(typename);IMPLEMENT_LIST(typename)`.  This will generate a linked list type and functions for this type.

The type will be called `typename_list_t`.  They should be created as global variables and initialized before use using `INITIALIZE_LIST(instance)`

Once this is done, you can use the functions.  They all are named `action_typename_list`, so if you want to use the list as a queue, you can use `append_typename_list` and `pop_typename_list`

## An Example
Say you had the following type, and you want to store it in a linked list:
``` c
typedef struct point {
    int x;
    int y;
} point_t;
```

If you were to use this list, then you would do the following in the header file:
``` c
typedef struct point {
    int x;
    int y;
    DEFINE_LINK(point);
} point_t;

DEFINE_LIST(point);
IMPLEMENT_LIST(point);
```

This defines a linked list that can only contain `struct point`, and implements the linked list funcitons.  The linked list can be used like this:
``` c
point_list_t points;

void do_something(point_t * point) {
    INITIALIZE_LIST(points);
    append_point_list(&points, point);
}
```

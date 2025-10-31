---
title: C语言实现继承，封装，多态
date: 2025-05-12 09:05:24
tags: [MCU]
index_img: /img/avatar.png

---

## C语言实现继承，封装，多态

不透明指针实现封装

```c
// person.h （头文件）
typedef struct Person Person; // 前向声明

Person* person_create(const char* name, int age);
void person_destroy(Person* p);

void person_set_name(Person* p, const char* name);
const char* person_get_name(const Person* p);
void person_set_age(Person* p, int age);
int person_get_age(const Person* p);
void person_print_info(const Person* p);

// person.c （实现文件）
struct Person {
    char* name;
    int age;
};

Person* person_create(const char* name, int age) {
    Person* p = malloc(sizeof(Person));
    p->name = strdup(name);
    p->age = age;
    return p;
}

// 其他函数实现...

```

结构体嵌套实现继承

```c
// base.h
typedef struct {
    int id;
    char* type;
} Base;

void base_init(Base* b, int id, const char* type);
void base_cleanup(Base* b);
void base_print(const Base* b);

// derived.h
typedef struct {
    Base base; // 必须作为第一个成员
    double value;
} Derived;

void derived_init(Derived* d, int id, const char* type, double value);
void derived_print(const Derived* d);

// derived.c
void derived_print(const Derived* d) {
    base_print(&d->base); // 调用基类方法
    printf("Value: %f\n", d->value);
}
```

函数指针实现多态

```c
// shape.h
typedef struct Shape Shape;

struct Shape {
    void (*draw)(Shape*); // 虚函数
    void (*move)(Shape*, int, int);
    void (*destroy)(Shape*);
};

void shape_draw(Shape* s);
void shape_move(Shape* s, int x, int y);
void shape_destroy(Shape* s);

// circle.h
typedef struct {
    Shape base; // 必须作为第一个成员
    int x, y;
    int radius;
} Circle;

Circle* circle_create(int x, int y, int radius);

// circle.c
static void circle_draw(Shape* s) {
    Circle* c = (Circle*)s;
    printf("Drawing circle at (%d,%d) radius %d\n", c->x, c->y, c->radius);
}

Circle* circle_create(int x, int y, int radius) {
    Circle* c = malloc(sizeof(Circle));
    c->base.draw = circle_draw;
    c->base.move = circle_move;
    c->base.destroy = circle_destroy;
    c->x = x;
    c->y = y;
    c->radius = radius;
    return c;
}
```





#### 完整示例：动物类层次结构

基类定义

```c
typedef struct Animal Animal;

struct Animal {
    void (*speak)(Animal*);
    void (*destroy)(Animal*);
    const char* name;
};

void animal_speak(Animal* a);
void animal_destroy(Animal* a);
```

派生类dog

```c
static void dog_speak(Animal* a) {
    Dog* d = (Dog*)a;
    printf("%s the dog says: Woof! (Breed: %d)\n", d->base.name, d->breed);
}

static void dog_destroy(Animal* a) {
    Dog* d = (Dog*)a;
    free((char*)d->base.name);
    free(d);
}

Dog* dog_create(const char* name, int breed) {
    Dog* d = malloc(sizeof(Dog));
    d->base.speak = dog_speak;
    d->base.destroy = dog_destroy;
    d->base.name = strdup(name);
    d->breed = breed;
    return d;
}
```

派生类cat

```c
typedef struct {
    Animal base;
    int color;
} Cat;

Cat* cat_create(const char* name, int color);
```

使用多态

```c
int main() {
    Animal* animals[3];
    
    animals[0] = (Animal*)dog_create("Buddy", 1);
    animals[1] = (Animal*)cat_create("Whiskers", 2);
    animals[2] = (Animal*)dog_create("Max", 3);
    
    for (int i = 0; i < 3; i++) {
        animal_speak(animals[i]);
    }
    
    for (int i = 0; i < 3; i++) {
        animal_destroy(animals[i]);
    }
    
    return 0;
}
```





#### 虚表实现多态

```c
// vtable.h
typedef struct {
    void (*speak)(void*);
    void (*destroy)(void*);
} AnimalVTable;

// animal.c
struct Animal {
    const AnimalVTable* vtable;
    const char* name;
};

void animal_speak(Animal* a) {
    a->vtable->speak(a);
}

// dog.c
static AnimalVTable dog_vtable = {
    .speak = dog_speak,
    .destroy = dog_destroy
};

Dog* dog_create(const char* name, int breed) {
    Dog* d = malloc(sizeof(Dog));
    d->base.vtable = &dog_vtable;
    d->base.name = strdup(name);
    d->breed = breed;
    return d;
}
```












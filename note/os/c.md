``` c
static inline struct thread_info *current_thread_info(void) __attribute_const__;
```
- static inline 表示该函数是静态内联函数，编译器在调用时可能会将其内联到调用位置，以减少函数调用开销。

- __attribute_const__ 是一个属性，指示该函数在相同的输入下总是返回相同的结果，允许编译器进行某些优化。

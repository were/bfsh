# Programming in Intrinsics

For now, we have the extended instructions integrated to the compiler infrastructure.
However, the high-level abstraction is still absent. For a quick proof of concept, manually
writing assembly code is probably a good short-term solution.

## Hardware Intrinsics

Hardware Intrinsics are essentially function wrappers of the inlined assembly code.
The format of an inlined assembly code should look like:

``` C
__asm__ __volatile__("mnemonic " "%0, %1, %2" : "=r"(a) : "r"(b), "i"(c))
```

where:

1. The `"mnemonic"` is the name of the instruction;
2. `"%0, %1, %2"` is something like C-like printf/scanf, which describes the format of the arguments;
3. The strings (`"=r"`, `"r"` and `"i"`) affilicated with expressions (`a`, `b` and `c`) represents
   the values fed to this assembly instruction, where
   1. `"=r"` indicates the destination register; the expression fed to this register must be a left value;
      this **must** appear between to colons (`:`). This is **not** enforced by the compiler, so developers
      should pay attention.
   2. `"r"` indicates the source register; for now, I only do `i64` integers; if a `float` or `double` is
      desired, probably we should use `reinterpret_cast`;
   3. `"i"` indicates an immediate value (constant); this value is a **signed** integer that matches the number
      of bits we discussed in last chapter; this value **must** be a compilation time constant.


You can use either macros of inlined function calls to wrap these assembly code. You can refer to
[this file](https://github.com/PolyArch/dsa-riscv-ext/blob/master/dsaintrin.h) for more details.
Here we adopt inline functions, because inlined functions allow us to setup default values for
some functions with a long argument list.


## Function-like Wrapper

``` C
// +: Macros implicitly inline the code in pp.
// -: No return value is allowed.
#define INTRINSIC_MACRO(a, b, c) \
   do {
     __asm__ __volatile__("asm1 " "%0, %1, %2" : "=r"(a) :);
     __asm__ __volatile__("asm2 " "%0, %1, %2" : : "r"(b));
     __asm__ __volatile__("asm3 " "%0, %1, %2" : : "i"(c));
   } while (false)
```

``` C
// +: Function allows you to use the syntax sugar default values.
// +: More flexible, either return value or reference can be used.
// -: Inline should be done by the compiler optimization.
inline void INTRINSIC_FUNC(int64_t b, int c = 1) {
   int64_t a;
   __asm__ __volatile__("asm1 " "%0, %1, %2" : "=r"(a) :);
   __asm__ __volatile__("asm2 " "%0, %1, %2" : : "r"(b));
   __asm__ __volatile__("asm3 " "%0, %1, %2" : : "i"(c));
   return a;
}
```


To make the programming interface slightly more intuitive, developers can use either macros or **inline**
functions as wrapper. This is especially useful when mulitiple instructions are required to achieve one
semantic behavior. Macros implicitly inline the "function call", but for a real function call, a specific
level of optimization should be enabled to inline the function call. This is critical for the semantics
correctness --- **even if you give a constant to the function call, within the scope of that function, it
is a declared argument int, which will not be recognized as a constant by the compiler to fill in immediate
operand for the given assembly instruction.**

## 1.1 
The situation:
```
volatile std::nullptr_t a = nullptr;
int *b; 
b = a;
```
won't be optimized correctly in gcc:

```
bar():
        mov     QWORD PTR [rsp-8], 0
        mov     rax, QWORD PTR [rsp-8]
        xor     eax, eax
        ret
```
due to this rule in standard C++23 Document number 4950:

"The result of the conversion is determined according to the following rules: 
 — If T is cv std::nullptr_t, the result is a null pointer constant.  
Note 1 : Since the conversion does not access the object to which the glvalue refers, there is no side effect even if T is volatile-qualified, and the glvalue can refer to an inactive member of a union."

and clang version will be correct:

```
bar():
        mov     qword ptr [rsp - 8], 0
        xor     eax, eax
        ret
```
## 1.3

To avoid UB in this situation:
```
int bar(int*a, int base, int off) {
    if (off > 0 && base > base + off) {
        return 42;
    }
    return a[base + off];
}
```

You should use this simple version:
```

#include <limits>

int foo(int *a, int base, int off) {
  if (off > 0 && base > std::numeric_limits<int>::max() - off) {
    return 42;
  }
  
  // to avoid undefined off + base operation
  if (off < 0 && base < std::numeric_limits<int>::min() - off) {
    return 42;
  } 
  
  // to avoid a[-1]
  if (off + base < 0) {
	  return 42;
  }
  return a[base + off];
}

```

Or this one, more complex, but without std::numeric_limits
```

#include <cstdint>

int foo(int* a, int32_t base, int32_t off) {

	int32_t result = static_cast<int32_t>(static_cast<uint32_t>(base) +              static_cast<uint32_t>(off));
	
	if (base > 0 && off > 0) {
		if (result < 0) {
			return 42;
		}
	} else if (base < 0 && off < 0) {
		if (result >= 0) {
			return 42;
		}
	}
	
	return a[result];
}

```

Or this version with builtin, but this is not C++ at all, just for fun i attached this here ^)
```
#include <stdio.h>
#include <assert.h>

int main(void)
{
  int x1 = -1073741826;
  int y1 = -1073741826;
  int z1 = x1 + y1; 
  int temp;

  printf("%d\n", __builtin_add_overflow(x1, y1, &temp));
  if (__builtin_add_overflow(x1, y1, &temp)) {
    printf("overflow detected");
  }
  printf("%d\n", z1);

  return 0;
}
```

# fe
A *tiny*, embeddable language implemented in ~~ANSI C~~ D.

**Note: this is a fork, with the implementation translated to D.**
See original repo here: https://github.com/rxi/fe


```clojure
(= reverse (fn (lst)
  (let res nil)
  (while lst
    (= res (cons (car lst) res))
    (= lst (cdr lst))
  )
  res
))

(= animals '("cat" "dog" "fox"))

(print (reverse animals)) ; => ("fox" "dog" "cat")
```

## Overview
* Supports numbers, symbols, strings, pairs, lambdas, macros
* Lexically scoped variables, closures
* Small memory usage within a fixed-sized memory region — no mallocs
* Simple mark and sweep garbage collector
* Easy to use C API
* Portable ANSI C — works on 32 and 64bit
* Concise — less than 800 sloc

---

* **[Demo Scripts](scripts)**
* **[C API Overview](doc/capi.md)**
* **[Language Overview](doc/lang.md)**
* **[Implementation Overview](doc/impl.md)**


## Contributing
The library focuses on being lightweight and minimal; pull requests will
likely not be merged. Bug reports and questions are welcome.


## License
This library is free software; you can redistribute it and/or modify it under
the terms of the MIT license. See [LICENSE](LICENSE) for details.


## Full D interpreter

```d
import std.stdio;
import std.file;
import fe;
import core.stdc.stdlib;

int main(string[] args)
{
    if (args.length != 2)
    {
        writeln("Usage: fe-interp <source.fe>");
        return 1;
    }

    int size = 1024 * 1024;
    void* data = malloc(size);

    fe_Context* ctx = fe_open(data, size);
    const(char)[] source = cast(char[]) std.file.read(args[1]);

    struct UserContext
    {
        const(char)[] remain;
    }

    static char readInBuf(fe_Context *ctx, void* udata)
    {
        UserContext* uc = cast(UserContext*)udata;
        if (uc.remain.length == 0)
            return '\0';
        else
        {
            char c = uc.remain[0];
            uc.remain = uc.remain[1..$];
            return c;
        }
    }

    int gc = fe_savegc(ctx);
    UserContext uc;
    uc.remain = source;
    while(true)
    {
        fe_Object *obj = fe_read(ctx, &readInBuf, &uc);

        // break if there's nothing left to read
        if (!obj) break;

        // evaluate read object
        fe_eval(ctx, obj);
    }
    // restore GC stack which would now contain 
    // both the read object and result from evaluation
    fe_restoregc(ctx, gc);

    fe_close(ctx);
    return 0;
}
```
class: middle, center

# C# 7 Closure Conversion

---

# Agenda

1. Introduction
   - Theory of Closure Conversion
2. Design for Lambdas
3. Design for Local Functions

---

# Introduction

*Goal*: Turn nested functions (local/anonymous functions) into
top-level methods.

Many terms for roughly the same thing:
 * Closure conversion
 * Lambda Lifting
 * Lambda Rewriting (our term)

---

# Example

.left-column[
**Original**

```csharp
class C {
    void M() {
        Func<int> f = () => 0;
        f();
    }
}
```
]

.right-column[
**Generated**

```csharp
class C {
    // <> makes name 'unspeakable' in C#
    int <>lambda() => 0;
    void M() { 
        <>lambda();
    }
}
```
]

---

# Complications: Capturing

Tricky part is captured variables -- variables must be passed in somehow 

**Approach 1**: Rewrite closure method to take captured vars as parameters
- Often called "Lambda Lifting"
- Pro: Efficient.
- Con: Must be able to rewrite all callers.

**Approach 2**: Create "environment" class to hold vars and put closure method inside 
- Often called "Closure Conversion"
- Pro: Type-safe.
- Con: Causes allocation to hold the variables

---

# Example

.left-column[
```csharp
class C {
    void M() {
        int x = 0;
        Func<int> f = () => x + 1;
        f();
    }
}
```
]

.right-column[
```csharp
class C {
    class Environment {
        int x;
        static int <>lambda()
            => this.x + 1;
    }

    void M()
    {
        var env = new Environment();
        env.x = 0;
        env.<>lambda();
    }
}
```
]

---

# Fundamentals of Lambdas

- Lambdas are always converted to delegates on construction
- Delegates cannot have different arity after compilation
  (violates type safety)

Therefore: all lambdas must use class closures

How to assign variables to environments?
- Scope. Create a new environment for each scope and put
  all variables declared in that scope in that environment.
- Create local reference to new environment at scope entry.
- Initialize environment.

What about nested lambdas that capture variables in parent scopes?
- We need a reference to the environment class
- But wait: we're *in* that environment class
- Let's just capture `this`

---

.left-column[
**Original**

```csharp
class C {
    void M(int x) {
        int y = 0;
        Action f = () =>
        {
            int z = x;
            Action f2 = () =>
            {
                y += z;
            };
            f2();
        }
        f();
    }
}
```
]

.right-column[
**Generated (Stage 1)**

```csharp
class C {
    class OuterEnv {
        int x;
        int y;

        void <>lambda() {
            int z = this.x;
            Action f2 = () =>
            {
                y += z; // Haven't rewritten yet
            };
            f2();
        }
    }

    void M(int x) {
        var outerEnv = new OuterEnv();
        outerEnv.x = x;
        outerEnv.y = 0;
        Action f = outerEnv.<>lambda;
        f();
    }
}
```
]

---

.left-column[
**Original**

```csharp
class C {
    void M(int x) {
        int y = 0;
*       Action f = () =>
*       {
*           int z = x;
*           Action f2 = () =>
*           {
*               y += z;
*           };
*           f2();
*       }
        f();
    }
}
```
]

.right-column[
**Generated (Stage 1)**

```csharp
class C {
    class OuterEnv {
        int x;
        int y;

*       void <>lambda() {
*           int z = this.x;
*           Action f2 = () =>
*           {
*               y += z; // Haven't rewritten yet
*           };
*           f2();
*       }
    }

    void M(int x) {
        var outerEnv = new OuterEnv();
        outerEnv.x = x;
        outerEnv.y = 0;
        Action f = outerEnv.<>lambda;
        f();
    }
}
```
]

---

.left-column[
**Generated (Stage 1)**

```csharp
class C {
    class OuterEnv {
        int x;
        int y;

        void <>lambda() {
            int z = this.x;
*           Action f2 = () =>
*           {
*               y += z;
*           };
            f2();
        }
    }

    void M(int x) {
        var outerEnv = new OuterEnv();
        outerEnv.x = x;
        outerEnv.y = 0;
        Action f = outerEnv.<>lambda;
        f();
    }
}
```
]

.right-column[
**Generated (Stage 2)**

```csharp
class C {
    class InnerEnv {
        int z;
        OuterEnv outerEnv;

*       void <>lambda() {
*           this.outerEnv.y += this.z;
*       }
    }

    class OuterEnv {
        int x;
        int y;

        void <>lambda() {
            var innerEnv = new InnerEnv();
            innerEnv.z = this.x;
            innerEnv.outerEnv = this;
            Action f2 = innerEnv.lambda;
            f2();
        }
    }

    void M(int x) {
        var outerEnv = new OuterEnv();
        outerEnv.x = x;
        outerEnv.y = 0;
        Action f = outerEnv.<>lambda;
        f();
    }
}
```
]
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
                y += z;
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
        Action f = () =>
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
*               y += z;
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

---

# Tricky Bits: Capturing fields

What if our lambda captures a field in the containing class of the top-level method?

Observations:
- An environment is a class containing fields that are used by lambdas
- An environment has a local or parameter reference that can be captured

- The containing class can contain fields that are used by lambdas
- The containing class has a parameter reference that can be captured (`'this'`)

The containing class *is* an environment. Just treat `'this'` as the reference to the parent environment and capture it if a closure captures any fields.

---

# Optimizations: Get Smart

.center[![get smart](/get-smart.jpg)]

---

.left-column[
What if we have a scope with no captured variables?

Create an empty environment? 

No. Instead, find the best place to put a closure method.
We can put *both* lambdas in the same environment.

As a rule:
  1. Find the most nested scope a lambda captures variables from
  2. Put the closure method in that scope's environment

Also reduces indirection to captured variables. 

If a lambda only captures from higher scopes, it can avoid walking parent environment pointer
]

.right-column[
```csharp
void M() {
    int x = 0;
    Func<Func<bool>> f1 = () =>
    {
        return () => x == 0;
    }
    f1();
}
```

```csharp
class Environment {
    int x;
    Func<bool> <>lambda1() => <>lambda2;
    bool <>lambda2() => this.x == 0;
}
```
]

---

# Algorithm

Two phase

**Phase 1 (Analysis)**:
1. Walk tree to find and record all captured vars by lambda.
  - Record captured in all lambdas between use and declaration.
2. For each lambda
  1. Find lowest scope that contains variables captured by lambda
    - This will be the lambda's containing environment
  2. Mark all scopes between this scope and last scope containing variables captured by lambda as capturing their parent environment
    - If 'this' is captured, mark the highest environment scope as capturing its parent environment, as well

---

**Phase 2 (Rewriting)**:
1. Set the current environment pointer to `'this'`.
1. Walk the tree
    - For scope containing captured variables calculated in phase 1
        1. Create environment type to hold variables with alpha-renamed type parameters matching the top-level method's type parameters.
        2. Hoist all captured variables to fields on environment
        3. Introduce synthesized statement to create environment with type arguments from containing method or lambda
        4. Introduce synthesized statements to initialize hoisted variables
        5. If scope is marked as capturing parent, hoist and initialize a synthesized field for the current environment pointer.
    - For lambda expression
        1. Synthesize a closure method on the containing type decided in analysis. Mark it static if the top-level method is static.
    - For expression
        1. If expression contains hoisted variable, rewrite to field access

---

The Curve Ball: Local Functions

Local functions break a lot of assumptions:

1. Captured variables are fields, locals, or parameters
   - You can now capture a local function, e.g. `() => Local()`
2. Calling a nested function is always through a delegate
   - You can call a local function without converting it to a delegate
3. Calls are always to a parent.
   - Local functions can call other local functions in the same scope, e.g.
    ```csharp
    void M() {
            void Local1() { Local2(); }
            void Local2() {}
    }
    ```
4. Environments can be captured in fields
    - Local functions can use `struct` environments passed as `ref` parameters, which cannot be captured in fields

---

# The Design: Nested Functions

Closure conversion vs. Lambda lifting:
  - We often *can* rewrite all calls to local functions, meaning we can rewrite arguments

When can we use a `struct` environment? When we can add `ref` parameters, i.e.
  - Not converted to a delegate
  - Not async or iterator

How do we capture `struct` environments? 
  - We don't, we pass all environments as a list of `ref` parameters to every closure

Local functions can 'call forward' to other local functions
  - We must calculate all environments and signatures ahead of time, so whenever we rewrite a call we know its final target

---

# Introducing: The Scope Tree

Problem: Local functions need more analysis and up-front info

Solution: Build intermediate representations of semantic info 

Scope Tree:

Closure:


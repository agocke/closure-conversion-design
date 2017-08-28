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

.center[![get smart](get-smart.jpg)]

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

# The Curve Ball: Local Functions

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

# Design (cont.)

Closures capture `struct` environments as parameters, `class` environments as before
- If a closure can't take `ref` parameters, all captured environments must be classes
- A closure can be lowered onto a `class` environment and take `struct` environments as parameters
- Rule: Find lowest class environment in scope; that's the containing type

.left-column[
```csharp
void M() {
    int x = 0;

    {
        int y = 0;
        Action a = () => y++;
        void Local() {
            y += x;
        }
        a();
        Local();
    }
}
```
]

.right-column[
```csharp
struct Env1 {
    int x;
}
class Env2 {
    int y;
    void <>lambda() => this.y++;
    void <>Local(ref Env1 env1) => this.y + env1.x;
}
void M()
{
    var env1 = new Env1();
    env1.x = 0;
    var env2 = new Env2();
    env2.y = 0;
    Action a = env2.<>lambda;
    a();
    env2.<>Local(ref env1);
}
```
]

---

# `This`: Again

**Problem**: What about `this`? Struct environments don't capture environment pointers

**Solution**: Treat it more like a parameter: just capture it in an environment like any other parameter.

---

# Introducing: The Scope Tree

**Problem**: Local functions need more analysis and up-front info

**Solution**: Build intermediate representations of semantic info 

Data Structures:

**`Scope` Class** - Represents a node in the `Scope` tree and holds info on declared captured variables and closures

**`Closure` Class** - Represents a nested function (lambda or anonymous function). Holds info on variables captured by this function, captured environments, and the containing environment.

**`ClosureEnvironment` Class** - Represents an environment that contains captured variables. Also tracks `struct`-ness of final synthesized type, as well if this environment captures a parent environment.

---

# Micropasses

`Scope` tree is much smaller than `BoundNode` tree
- So we can run multiple passes cheaply

**Pass 1**: Assign variables to environment based on scope (same as before) and assign captured environments
- If any closure can't take ref params and captures vars from this environment, environment must be `class`. Otherwise, `struct`.
- A closure captures an environment if it captures any of its variables OR if it captures a closure which captures the environment. (This may require fixed-point iteration)

**Pass 2**: Find the closure's containing environment (if any) and linearize class environments.
- A closure's containing environment is the nearest class environment in scope
- If the closure captures any more class environments, mark that the current class environment as capturing its parent, and continue up the tree until we capture no more class environments. ("Linearization")

---

# Micropass: Optimization

.left-column[
Unnecessary indirection -- we may end up with a closure environment containing only `this`

Optimize by removing environment and "inlining" `this`
- If env is struct and everything which captures `this` lives in the containing type, just delete the environment and all its references.
- If env is a class, remove environment, move all contained closures to the top level, and rewrite nested environment captures to capture `this` instead.
]

.right-column[
```csharp
class C {
    int _x;    
    void M() {
        Action a = () =>
        { 
            int y = _x++;
            Func<int> b = () => y + _x;
        }
    }
}
```
]

---

# Micropass: Optimization

.left-column[
Unnecessary indirection -- we may end up with a closure environment containing only `this`

Optimize by removing environment and "inlining" `this`
- If env is struct and everything which captures `this` lives in the containing type, just delete the environment and all its references.
- If env is a class, remove environment, move all contained closures to the top level, and rewrite nested environment captures to capture `this` instead.
]

.right-column[
```csharp
class C {
    int _x;    
    class Env1 {
        C thisCapture;
        void <>lambda() {
            var env2 = new Env2();
            env2.y = thisCapture._x++;
            env2.env1 = this;
            Func<int> = env2.<>lambda;
        }
    }
    class Env2 {
        int y;
        Env1 env1;
        void <>lambda() => this.y + env1.thisCapture._x;
    }
    void M() {
        var env = new Env1();
        Action a = env.<>lambda;
    }
}
```
]

---

# Micropass: Optimization

.left-column[
Unnecessary indirection -- we may end up with a closure environment containing only `this`

Optimize by removing environment and "inlining" `this`
- If env is struct and everything which captures `this` lives in the containing type, just delete the environment and all its references.
- If env is a class, remove environment, move all contained closures to the top level, and rewrite nested environment captures to capture `this` instead.
]

.right-column[
```csharp
class C {
    int _x;    
    void <>lambda() {
        var env2 = new Env2();
        env2.y = this._x++;
        env2.thisCapture = this;
        Func<int> = env2.<>lambda;
    }
    class Env2 {
        int y;
        C thisCapture;
        void <>lambda() => this.y + thisCapture._x;
    }
    void M() {
        var env = new Env1();
        Action a = env.<>lambda;
    }
}
```
]

---

# Code Generation Micropasses

Now we need to synthesize some real code. Actual signatures must be calculated before rewriting.

1. Synthesize closure environments
- For each `ClosureEnvironment`, create a synthesized type to represent that environment, with alpha renamed type parameters and kind based on earlier struct/class calculation.

2. Synthesize lowered methods
- For each closure, create a synthesized method on its calculated containing environment (or the containing type if there is no containing environment)
- Mark the method as static or instance depending on whether or not it captured a `this` closure before optimization
- Assemble all captured struct environments and add synthesized ref parameters to the end of the method

---

# Rewriting

Now revisit the entire bound tree, rewriting similarly to lambda algorithm.

Differences:
- Only change the "current environment pointer" for class environments.
- Also look through environment `ref` parameters for hoisted locals, not just through a synthesized environment local variable
- When rewriting calls to closures with `ref` environment parameters, pass synthesized arguments -- meaning a local if this is the scope where the environment was created, or a parameter if the environment was created in a higher scope 

.center[
# Et Voilà!
]

---
class: middle, center

# Questions?

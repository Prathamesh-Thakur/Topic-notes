# GIL (Global Interpreter Lock)
The GIL is a mutex (lock) which allows only a single thread to hold control over the Python interpreter at a time.

## Why it exists:
- Python uses reference counting for memory management. When an object's reference count hits zero, it is removed from memory.
- Allowing mutlithread access to the interpreter can lead to race conditions and corrupt values of reference count.

## Effect on execution
If a program is doing heavy math (CPU-bound), the GIL prevents multiple threads from speeding it up. However, if a thread is waiting on I/O (network, database), it voluntarily drops the GIL, allowing other threads to run in the meantime.

## Execution components:

1. Compiler
    - The code is first converted to tokens, followed by the generation of an Abstract Syntax Tree.
    - The compiler converts this tree into bytecode. 
    - This happens at **compile time**.

2. Interpreter
    - The interpreter executes the compiled bytecode.
    - This includes creation of memory, which holds ***Pyobjects*** and all related execution memory.
    - This happens at **runtime**.
    - This is what the GIL locks, which is to protect the memory.

3. Program

    Written script.

4. Process
    On executing the program, a **Process** is created by the OS.

5. Thread
    A thread is an actual worker in the process that is executing it. Each Python program starts with a single main thread.

## Compile & Run time
- Unlike languages like C and Java which have the compilation and runtime steps as different, Python lumps them into a single step.
- When the python script.py command is run, the script is compiled and immediately given to the PVM for execution.
- Since it lumps them into a single step, development speed is increased.
- However, it does not provide the same rigorous compile-time checking like C and Java. This is also partly in attribution to Python being a dynamically typed language. 

## Pyobjects
- Under the hood, each typed token in Python is mapped to a C structure, known as **Pyobject**.
- Each Pyobject has two components, a reference count and the type of the object.
- This is done by the interpreter at **runtime**, which is why Python is known as **dynamically** typed.
- This is also becuase reassignment of variables, and changing of types simply involves reassignment of the pointers to the new value.
- It is also **strongly** typed, because even if variables are flexible in types, the underlying objects themselves have a specific type.
- Python stores global and module-level namespaces in a dictionary. The dictionary maps variable names (as strings) to their respective PyObject pointers in memory. Looking up variables in this hash table is part of what makes Python slower than compiled languages.

### Functions
- One caveat to this is functions.
- When Python compiles functions, it stores all it's variables in a fixed size C array.
- This leads to faster loading of variables, which can speed up your code.

## Garbage collection
- When an object's reference count becomes zero, it is cleaned up by the garbage collector.
- However, there can be instances of cyclical pointers, or other instances where there is a non zero reference count.
- In such cases, the **Generational** garbage collector runs and cleans up these variables.
- Python assigns three generations to variables. New variables start at generation 0.
- On every successive garbage collection survival, variables are promoted to a higher generation.
- Python assumes newer variables will be cleaned up more often, and scans gen 0 most often, and gen 2 most rarely.
- Intermediate operations (like a + b) create temporary objects on the evaluation stack. These are instantly destroyed (reference count drops to 0) the moment the operation is finished.
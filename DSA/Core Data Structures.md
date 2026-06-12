A **data structure in Java** is a specialized format for [organizing, processing, accessing, and storing data](https://www.mygreatlearning.com/blog/data-structures-using-java/) in a computer's memory so that operations can be performed efficiently. Instead of working with isolated variables, data structures group information into cohesive containers like lists, tables, or hierarchies. Java provides a powerful, pre-built ecosystem for these structures through its [[Java Collections Framework]] (`java.util`), alongside basic built-in types and capabilities for user-defined custom structures.

Data structures in Java are broadly categorized based on how memory is allocated and how data elements are linked together:
```
DATA STRUCTURES
│
├── PRIMITIVE (built into language)
│   ├── int, long, float, double
│   ├── char, boolean, byte, short
│   └── These are the BUILDING BLOCKS
│
└── NON-PRIMITIVE (constructed from primitives)
    │
    ├── LINEAR (data arranged in sequence)
    │   ├── Static  → Array (fixed size)
    │   └── Dynamic → Linked List, Stack, Queue
    │
    └── NON-LINEAR (data arranged hierarchically/network)
        ├── Trees  (hierarchical)
        └── Graphs (network)
```

#### Linear Data Structures
**Linear data structures** organize elements sequentially, meaning each item connects directly to its previous and next adjacent elements. In Java, these structures are either built natively into the core language or provided via classes within the Java Collections Framework.

Core Linear Data Structures:
1. Arrays
2. Lists (ArrayList & LinkedList)
3. Stacks
4. Queues

#### Non-Linear Data Structures
**Non-linear data structures** organize elements hierarchically or interconnectedly rather than sequentially. Unlike linear structures (such as arrays or linked lists), elements in a non-linear structure can connect to multiple other elements, making it impossible to traverse them fully in a single linear pass.

Core Non-Linear Data Structures:
1. Trees
2. Graphs
3. Heaps
4. Hash-Based Structures (Maps / Sets)
5. Tries (Prefix Trees)
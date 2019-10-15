# Language

## 目录：

```
├── README.md          # This README
├── bytecode_verifier  # The bytecode verifier
├── e2e_tests          # infrastructure and tests for the end-to-end flow
├── functional_tests   # Testing framework for the Move language
├── compiler           # The IR to Move bytecode compiler
├── stdlib             # Core Move modules and transaction scripts
├── test.sh            # Script for running all the language tests
└── vm
    ├── cost_synthesis # Cost synthesis for bytecode instructions
    ├── src            # Bytecode language definitions, serializer, and deserializer
    ├── tests          # VM tests
    ├── vm_genesis     # The genesis state creation, and blockchain genesis writeset
    └── vm_runtime     # The bytecode interpreter
```

## 内容：

**Move语言包括bytecode_verifier，compiler ，VM等各个部分，在这份教程中我们将不会研究Move的编译与用于验证安全性的bytecode_verifier等部分，只是把他们当作包装好的api。**

**我们关心的是VM如何调度各个部分，最终完成他的功能。因此我们将参照研究execution部分的做法，直接从看VM如何启动并提供服务，并逐渐深入到各个模块。**


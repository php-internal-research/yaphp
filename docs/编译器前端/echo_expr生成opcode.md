# echo_expr生成opcode

上篇文章，我们讲解了如何从`yaphp`源代码生成`AST`。这篇文章，我们来讲解如何从`AST`生成`opcode`。

一句话总结起来就是，我们对`AST`进行深度遍历，然后以每一个子`AST`来生成一条`opline`，最终得到`op_array`。我们来看一下具体如何去实现它。

首先，我们需要定义一下和`op_array`相关的数据结构：

```cpp
#define IS_UNUSED 0 /* Unused operand */
#define IS_CONST (1 << 0)
#define IS_TMP_VAR (1 << 1)
#define IS_VAR (1 << 2)
#define IS_CV (1 << 3) /* Compiled variable */

#define INITIAL_OP_ARRAY_SIZE 64

typedef struct _zend_op_array zend_op_array;
typedef struct _zend_oparray_context zend_oparray_context;
typedef struct _zend_op zend_op;
typedef union _znode_op znode_op;
typedef struct _znode znode;

union _znode_op {
    uint32_t num;
    uint32_t var;
};

struct _znode {
    char op_type;
    znode_op op;
};

struct _zend_op {
    znode_op op1;
    znode_op op2;
    znode_op result;
    unsigned char opcode;
    char op1_type;
    char op2_type;
    char result_type;
};

struct _zend_op_array {
    zend_op *opcodes;
    uint32_t last; /* number of opcodes */
    uint32_t T;    /* number of temporary variables */
};

struct _zend_oparray_context {
    uint32_t opcodes_size;
};
```

其中，`_zend_op_array`是核心结构，其他的结构都是以它为中心展开的。这些结构的具体含义我们可以在很多`PHP`源码分析的文章里面找到，所以我们不过多介绍。

现在，让我们来实现一下对`AST`的处理流程，在文件`zend_language_parser.y`里面：

```bison
op_array = (zend_op_array *) malloc(sizeof(zend_op_array));
init_op_array(op_array, INITIAL_OP_ARRAY_SIZE);
CG(active_op_array) = op_array;

zend_oparray_context_begin();
zend_compile_top_stmt(CG(ast));
```

这就是我们的核心了，这里的步骤可以总结为：

```bash
1. 初始化op_array，并且赋值给CG(active_op_array)
2. 调用zend_oparray_context_begin初始化CG(context)
3. 调用zend_compile_top_stmt生成opcode
```

`OK`，以这个为思路，我们来看看具体如何实现的。

首先是函数`init_op_array`：

```cpp
void init_op_array(zend_op_array *op_array, int initial_ops_size) {
    op_array->opcodes = (zend_op *) malloc(initial_ops_size * sizeof(zend_op));
    op_array->last = 0;
    op_array->T = 0;
}
```

这个函数非常的简单，首先为`op_array`里面保存的`opcodes`分配初始化的内存；然后设置`last`（即`opline`的条数）为`0`；然后设置`T`（即临时变量的个数为`0`，后面，我们的临时变量的名字是按照`T`来递增命名的，例如第一个临时变量叫做`1`，第二个临时变量叫做`2`，依次类推）为0。

然后是`zend_oparray_context_begin`：

```cpp
void zend_oparray_context_begin() {
    CG(context).opcodes_size = INITIAL_OP_ARRAY_SIZE;
}
```

这个`CG(context)`会与我们的`CG(active_op_array)`挂钩，例如这里的`CG(context).opcodes_size`表示，我们的`CG(active_op_array)`总容量。可想而知，当我们编译的`opline`条数达到`CG(context).opcodes_size`的时候，需要进行`CG(active_op_array)`的扩容。

最后就是我们的核心函数`zend_compile_top_stmt`：

```cpp
void zend_compile_top_stmt(zend_ast *ast) {
    if (!ast) {
        return;
    }

    if (ast->kind == ZEND_AST_STMT_LIST) {
        zend_ast_list *list = zend_ast_get_list(ast);
        for (uint32_t i = 0; i < list->children; ++i) {
            zend_compile_top_stmt(list->child[i]);
        }
        return;
    }

    zend_compile_stmt(ast);
}
```

这个函数是用来编译我们的`ZEND_AST_STMT_LIST`的。然后，每一条语句，我们会调用`zend_compile_stmt`来进行编译。

我们来看看`zend_compile_stmt`：

```cpp
void zend_compile_stmt(zend_ast *ast) {
    if (!ast) {
        return;
    }

    switch (ast->kind) {
    case ZEND_AST_ECHO:
        zend_compile_echo(ast);
        break;
    default:
        break;
    }
}
```

因为我们目前只实现了`echo`语句，所以，我们这里只有一个`ZEND_AST_ECHO`的类型。我们来看看我们是如何编译`echo`语句的：

```cpp
void zend_compile_echo(zend_ast *ast) {
    zend_ast *expr_ast = ast->child[0];

    znode expr_node;
    zend_compile_expr(&expr_node, expr_ast);

    zend_emit_op(ZEND_ECHO, &expr_node, nullptr);
}
```

这个函数做了两件事情，第一件事情是编译`echo`节点使用的`expr ast`，第二件事情为`echo ast`（也就是我们的`echo`语句）生成一条`opline`。

我们来看看我们是如何编译`expr_ast`的：

```cpp
void zend_compile_expr(znode *result, zend_ast *ast) {
    switch (ast->kind) {
    case ZEND_AST_LNUM:
        result->op.num = zend_ast_get_lnum(ast)->lnum;
        result->op_type = IS_CONST;
        break;
    case ZEND_AST_BINARY_OP:
        zend_compile_binary_op(result, ast);
        return;
    default:
        break;
    }
}
```

可以看到，如果是一个`ZEND_AST_LNUM`类型的节点（也就是一个数字），那么我们直接返回它作为编译`ast`后的结果；如果是一个`ZEND_AST_BINARY_OP`类型的节点（也就是操作符），那么我们需要继续编译。

```cpp
void zend_compile_binary_op(znode *result, zend_ast *ast) {
    zend_ast *left_ast = ast->child[0];
    zend_ast *right_ast = ast->child[1];
    uint32_t opcode = ast->attr;

    znode left_node, right_node;

    zend_compile_expr(&left_node, left_ast);
    zend_compile_expr(&right_node, right_ast);

    zend_emit_op_tmp(result, opcode, &left_node, &right_node);
}
```

可以看到，对`ZEND_AST_BINARY_OP`类型的编译实际上就是一个递归的函数。我们发现，只有在`AST`的子节点都是终结符的时候，我们才会调用`zend_emit_op_tmp`生成一条`opline`。我们看看`zend_emit_op_tmp`的实现：

```cpp
/**
 * generate an opline
 */
static zend_op *zend_emit_op(unsigned char opcode, znode *op1, znode *op2) {
    zend_op *opline = get_next_op();
    opline->opcode = opcode;

    if (op1 != nullptr) {
        opline->op1_type = op1->op_type;
        opline->op1 = op1->op;
    }

    if (op2 != nullptr) {
        opline->op2_type = op2->op_type;
        opline->op2 = op2->op;
    }

    return opline;
}

static inline uint32_t get_temporary_variable(void) {
    return ++CG(active_op_array)->T;
}

static zend_op *zend_emit_op_tmp(znode *result, unsigned char opcode, znode *op1, znode *op2) {
    zend_op *opline = zend_emit_op(opcode, op1, op2);

    if (result) {
        zend_make_tmp_result(result, opline);
    }

    return opline;
}
```

这几个函数就非常的简单了，设置`opline`的`op1`和`op2`，然后`opline`的`result`我们都用一个临时变量。

这样，我们就实现了`AST`到`opcode`的转化，我们来编写一个测试脚本：

```php
echo 1 + 2 * 3;
echo 1 + 2 + 3;
```

执行结果如下：

```bash
*********************gennerate opcode*********************
#0		ZEND_MUL		2		3		~1
#1		ZEND_ADD		1		~1		~2
#2		ZEND_ECHO		~2
#3		ZEND_ADD		1		2		~3
#4		ZEND_ADD		~3		3		~4
#5		ZEND_ECHO		~4
```

可以看到，是符合我们的预期的。

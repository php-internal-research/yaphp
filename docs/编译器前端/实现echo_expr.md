# 实现echo_expr

到目前为止，我们已经实现了`yaphp`源代码到`AST`的生成工作。但是，我们对`echo`语句的实现还是简单处理了，之前的实现如下：

```bison
statement:
    T_ECHO expr ';'     { $$ = $2; }
;
```

可以看到，这里实际上我们没有体现出`T_ECHO`的功能，我们仅仅处理了`expr`。所以，这里我们需要去处理一下。

有了前面的基础之后，我们可以非常轻松的来实现这个`AST`的生成。

我们修改后的规则如下：

```bison
statement:
    T_ECHO echo_expr ';'     { $$ = $2; }
;

echo_expr:
    expr {
        std::cout << "create echo zend_ast" << std::endl;
        $$ = zend_ast_create_1(ZEND_AST_ECHO, 0, $1);
    }
;
```

可以看到，我们加了一个`echo_expr`，用来创建一个`ZEND_AST_ECHO`类型的`AST`。这里，我们的语法和`php-src`的有点不同，`php-src`的语法功能更加的丰富一点，它支持`echo`后面跟一个`ZEND_AST_STMT_LIST`，这个的实现也是比较简单的，和我们之前创建`top_statement_list`的思路是一致的。这里小伙伴们可以自己去实现一下，我们的`yaphp`就不支持这种可有可无的语法了。

其中，`zend_ast_create_1`的实现如下：

```cpp
zend_ast *zend_ast_create_1(zend_ast_kind kind, zend_ast_attr attr, zend_ast *child) {
    zend_ast *ast;

    ast = (zend_ast *) malloc(zend_ast_size(1));
    ast->kind = kind;
    ast->attr = attr;
    ast->child[0] = child;

    return ast;
}
```

然后，我们需要在`_zend_ast_kind`中增加一个`ZEND_AST_ECHO`：

```cpp
/* 1 child node */
ZEND_AST_ECHO,
```

最后，我们再修改一下我们的`dump_compiler_globals`函数：

```cpp
else if (ast->kind > ZEND_AST_0_NODE_END && ast->kind < ZEND_AST_1_NODE_END) {
    queue.push_back(ast->child[0]);
} else if (ast->kind > ZEND_AST_1_NODE_END && ast->kind < ZEND_AST_2_NODE_END) {
    queue.push_back(ast->child[0]);
    queue.push_back(ast->child[1]);
}
```

其中，`ZEND_AST_0_NODE_END`, `ZEND_AST_1_NODE_END`, `ZEND_AST_2_NODE_END`这三个`zend_ast`节点类型`php-src`是没有的，这里是为了方便调试给加上的，否则，我们每次增加一个`zend_ast`节点类型，就需要写一个`if`语句。

现在，让我们来编译一下`yaphp`，并且运行，结果如下：

```bash
create * zend_ast
create + zend_ast
create echo zend_ast
kind: 129, attr: 0
kind: 131, attr: 0
kind: 515, attr: 1
kind: 65, attr: 0, value: 1
kind: 515, attr: 3
kind: 65, attr: 0, value: 2
kind: 65, attr: 0, value: 3
```

符合我们的预期。

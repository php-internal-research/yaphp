# 执行opcode

上一篇文章，我们成功的把`AST`翻译成了`opcode`，这样有一个好处，就是它是线性的，连续的，这和我们的`CPU`去一条一条的执行机器指令是保持一致的，非常便于人类理解。但是，我们还没有去设置这些`opcode`对应的`handler`。

这篇文章，我们来实现对这些`opcode`的执行，这一节还是比较难的。

首先，我们来捋一捋`opcode`和`handler`的关系。我们参考`PHP`的实现。首先是我们的`_zend_op`：

```cpp
struct _zend_op {
    znode_op op1;
    znode_op op2;
    znode_op result;
    unsigned char opcode;
    char op1_type;
    char op2_type;
    char result_type;
};
```

这种结构实际上是一种三地址码的组织形式，这种结构可以方便我们后续进行数据流分析。

我们知道，变量和字面量等等是有类型的，既然有类型，我们的操作数`1`和操作数`2`就可能多种组合。所以，这实际上就是一种笛卡尔积的表现形式了。再加上`opcode`的种类也不止一种，所以，我们有如下笛卡尔积：

```bash
opcode × op1 × op2
```

举个例子画个图：

```bash
+----------------------+              +----------------------+        +----------------------+
|                      |              |                      |        |                      |
|       ZEND_ADD       |              |       IS_CONST       |        |       IS_CONST       |
|                      |              |                      |        |                      |
+----------------------+              +----------------------+        +----------------------+
                                                                                              
                                                                                              
                                                                                              
                                                                                              
+----------------------+             +----------------------+         +----------------------+
|                      |             |                      |         |                      |
|       ZEND_SUB       |             |      IS_TMP_VAR      |         |      IS_TMP_VAR      |
|                      |             |                      |         |                      |
+----------------------+             +----------------------+         +----------------------+
                                                                                              
                                                                                              
                                                                                              
                                                                                              
                                                                                              
+----------------------+                                                                      
|                      |                                                                      
|       ZEND_MUL       |                                                                      
|                      |                                                                      
+----------------------+                                                                      
                                                                                              
                                                                                              
                                                                                              
                                                                                              
+----------------------+                                                                      
|                      |                                                                      
|       ZEND_DIV       |                                                                      
|                      |                                                                      
+----------------------+                                                                      
```

那么，我们就会有`4 * 2 * 2`种`spec handler`：

```bash
ZEND_ADD_IS_CONST_IS_CONST
ZEND_ADD_IS_CONST_IS_TMP_VAR
ZEND_ADD_IS_TMP_VAR_IS_CONST
ZEND_ADD_IS_TMP_VAR_IS_TMP_VAR
# 以此类推
```

假设，我们的`opcode`是按照顺序从`0`开始编号的，并且操作数的类型也是从`0`开始进行编号，并且，我们的`spec handler`也是严格按照顺序在内存中进行排序的。那我，我们就可以通过`opcode`、`op1_type`、`op2_type`找到`spec handler`的位置了，这个有点像一个三维的数组。对应的算法如下：

```cpp
opcode * op1_type的数量 * op2_type的数量 + opt_type的编号 * op2_type的数量 + op2_type的编号
```

我们的实现都是围绕着这个算法来进行的。

首先，我们来定义一下操作数的类型：

```cpp
#define OP_TYPE_MAP(XX)                                                                                                \
    XX(IS_UNUSED, 0)                                                                                                   \
    XX(IS_CONST, 1 << 0)                                                                                               \
    XX(IS_TMP_VAR, 1 << 1)                                                                                             \
    XX(IS_VAR, 1 << 2)                                                                                                 \
    XX(IS_CV, 1 << 3)

enum op_type_e {
#define OP_TYPE_GEN(name, value) name = value,
    OP_TYPE_MAP(OP_TYPE_GEN)
#undef OP_TYPE_GEN
};

enum op_type_code_e {
#define OP_TYPE_CODE_GEN(name, value) _##name##_CODE,
    OP_TYPE_MAP(OP_TYPE_CODE_GEN)
#undef OP_TYPE_CODE_GEN
};
```

接着，我们可以来编写我们的`spec handler`了。从上面可以看出，我们的操作数有好几个。但是，实际上，对于同一个`opcode`，它要执行的动作是一样的，只不过操作数的类型不同，获取操作数的方式需要改变。如果我们手写每一种`opcode`对应的所有`handler`，那么这个维护成本是非常的高的，所以，我们应该是有一个代码生成的机制，写好通用的模板代码，然后直接生成即可。

下面，我们来给出模板代码：

```cpp
ZEND_VM_HANDLER(0, ZEND_NOP, CONST|TMPVAR, CONST|TMPVAR)
{
    return 0;
}

ZEND_VM_HANDLER(1, ZEND_ADD, CONST|TMPVAR, CONST|TMPVAR)
{
    int64_t op1, op2;

    op1 = GET_OP1();
    op2 = GET_OP2();
    op_array->literals[opline->result.var] = op1 + op2;
    return 0;
}

ZEND_VM_HANDLER(2, ZEND_SUB, CONST|TMPVAR, CONST|TMPVAR)
{
    int64_t op1, op2;

    op1 = GET_OP1();
    op2 = GET_OP2();
    op_array->literals[opline->result.var] = op1 - op2;
    return 0;
}

ZEND_VM_HANDLER(3, ZEND_MUL, CONST|TMPVAR, CONST|TMPVAR)
{
    int64_t op1, op2;

    op1 = GET_OP1();
    op2 = GET_OP2();
    op_array->literals[opline->result.var] = op1 * op2;
    return 0;
}

ZEND_VM_HANDLER(4, ZEND_DIV, CONST|TMPVAR, CONST|TMPVAR)
{
    int64_t op1, op2;

    op1 = GET_OP1();
    op2 = GET_OP2();
    op_array->literals[opline->result.var] = op1 / op2;
    return 0;
}

ZEND_VM_HANDLER(136, ZEND_ECHO, CONST|TMPVAR, UNUSED)
{
    int64_t op1;

    op1 = GET_OP1();
    printf("%lld", op1);
    return 0;
}
```

可以看到，非常的简单。其中，这里的数字`0, 1, 2, 3, 4, 136`是这个`opcode`的编号。

接着，我们来用`PHP`代码来完成这个代码生成的脚本：

```php
<?php

#define IS_UNUSED 0 /* Unused operand */
#define IS_CONST (1 << 0)
#define IS_TMP_VAR (1 << 1)
#define IS_VAR (1 << 2)
#define IS_CV (1 << 3) /* Compiled variable */

define('ZEND_VM_OP_UNUSED', 1 << 0);
define('ZEND_VM_OP_CONST', 1 << 1);
define('ZEND_VM_OP_TMPVAR', 1 << 2);
define('ZEND_VM_OP_VAR', 1 << 3);
define('ZEND_VM_OP_CV', 1 << 4);

$op_types_map = array(
    "UNUSED"               => ZEND_VM_OP_UNUSED,
    "CONST"                => ZEND_VM_OP_CONST,
    "TMPVAR"               => ZEND_VM_OP_TMPVAR,
    "VAR"                  => ZEND_VM_OP_VAR,
    "CV"                   => ZEND_VM_OP_CV,
);

$op1_get = array(
    "UNUSED"   => "nullptr",
    "CONST"    => "opline->op1.num",
    "TMPVAR"   => "op_array->literals[opline->op1.var]",
    "VAR"      => "nullptr",
    "CV"       => "nullptr",
);

$op2_get = array(
    "UNUSED"   => "nullptr",
    "CONST"    => "opline->op2.num",
    "TMPVAR"   => "op_array->literals[opline->op2.var]",
    "VAR"      => "nullptr",
    "CV"       => "nullptr",
);

$opcodes = [];
$max_opcode = 0;
$spec_names = [];

function parse_operand_spec($def, $lineno, $str, &$flags)
{
    global $op_types_map;

    $flags = 0;
    $a = explode("|", $str);
    foreach ($a as $val) {
        if (isset($op_types_map[$val])) {
            $flags |= $op_types_map[$val];
        } else {
            die("ERROR ($def:$lineno): Wrong operand type '$str'\n");
        }
    }

    return array_flip($a);
}

function gen_handler($f, $opcode)
{
    global $op1_get, $op2_get, $spec_names, $op_types_map;

    $opTypes = array_keys($op_types_map);

    foreach ($opTypes as $op1Type) {
        foreach ($opTypes as $op2Type) {
            if (isset($opcode['op1'][$op1Type]) && isset($opcode['op2'][$op2Type])) {
                $specialized_replacements = [
                    "/GET_OP1\(([^)]*)\)/" => $op1_get[$op1Type],
                    "/GET_OP2\(([^)]*)\)/" => $op2_get[$op2Type],
                ];

                $name = $opcode['op'];
                $templateCode = $opcode['code'];

                $spec_name = $name."_SPEC"."_".$op1Type."_".$op2Type;
                $spec_names[] = $spec_name;
                fputs($f, "static int $spec_name(zend_op_array *op_array, zend_op *opline) ");
                $code = preg_replace(array_keys($specialized_replacements), array_values($specialized_replacements), $templateCode);
                fputs($f, $code);
            } else {
                $spec_names[] = 'nullptr';
            }
        }
    }
}

function gen_spec_handlers($f)
{
    global $spec_names;

    fputs($f, "\tstatic const void * const spec_handlers[] = {\n");
    foreach ($spec_names as $spec_name) {
        fputs($f, "\t\t(void *) $spec_name,\n");
    }
    fputs($f, "\t};\n");

    fputs($f, "\tzend_spec_handlers = spec_handlers;\n");
}

function gen_vm_execute_code($f)
{
    fputs($f, "void zend_execute(zend_op_array *op_array) {\n");
    fputs($f, "\tfor (size_t i = 0; i < op_array->last; i++) {\n");
    fputs($f, "\t\tzend_op *opline = &(op_array->opcodes[i]);\n");
    fputs($f, "\t\t((opcode_handler_t)opline->handler)(op_array, opline);\n");
    fputs($f, "\t}\n");
    fputs($f, "}\n\n");
}

function gen_vm_init_code($f)
{
    fputs($f, "void zend_vm_init() {\n");

    gen_spec_handlers($f);

    fputs($f, "}\n");
}

function gen_executor_code($f)
{
    global $opcodes, $max_opcode;

    // define
    fputs($f, "const void * const *zend_spec_handlers;\n");
    fputs($f, "typedef int (*opcode_handler_t) (zend_op_array *op_array, const zend_op *opline);\n\n");

    // Generate zend_vm_get_opcode_handler() function

    fputs($f, "static uint32_t zend_vm_get_opcode_handler_idx(const zend_op *opline)\n");
    fputs($f, "{\n");
    fputs($f, "\tstatic int zend_vm_decode[IS_CV + 1] = {0};\n\n");
    fputs($f, "\t#define OP_TYPE_CODE_GEN(name, value) zend_vm_decode[name] = _##name##_CODE;\n");
    fputs($f, "\t\tOP_TYPE_MAP(OP_TYPE_CODE_GEN)\n");
    fputs($f, "\t#undef OP_TYPE_CODE_GEN\n\n");
    fputs($f, "\tuint32_t offset = 0;\n");
    fputs($f, "\toffset += opline->opcode * 5 * 5;\n");
    fputs($f, "\toffset += zend_vm_decode[(int) opline->op1_type] * 5;\n");
    fputs($f, "\toffset += zend_vm_decode[(int) opline->op2_type];\n");
    fputs($f, "\treturn offset;\n");
    fputs($f, "}\n\n");

    fputs($f, "const void *zend_vm_get_opcode_handler(const zend_op *opline)\n");
    fputs($f, "{\n");
    fputs($f, "\tuint32_t offset = zend_vm_get_opcode_handler_idx(opline);\n");
    fputs($f, "\treturn zend_spec_handlers[offset];\n");
    fputs($f, "}\n\n");

    fputs($f, "void zend_vm_set_opcode_handler(zend_op *opline)\n");
    fputs($f, "{\n");
    fputs($f, "\topline->handler = zend_vm_get_opcode_handler(opline);\n");
    fputs($f, "}\n\n");

    $num = 0;

    for ($i = 0; $i <= $max_opcode; $i++) {
        if (isset($opcodes[$num])) {
            gen_handler($f, $opcodes[$num], $num);
        } else {
            gen_handler($f, [], $num);
        }
        $num++;
    }

    gen_vm_execute_code($f);

    gen_vm_init_code($f);
}

function gen_vm(string $def)
{
    global $opcodes, $max_opcode;

    $in = file($def);

    $lineno = 0;
    $handler = 0;

    foreach ($in as $line) {
        if (strpos($line, "ZEND_VM_HANDLER(") === 0) {
            if (preg_match(
                "/^ZEND_VM_HANDLER\(\s*([0-9]+)\s*,\s*([A-Z_]+)\s*,\s*([A-Z_|]+)\s*,\s*([A-Z_|]+)\s*(,\s*([A-Z_|]+)\s*)?(,\s*SPEC\(([A-Z_|=,]+)\)\s*)?\)/",
                $line,
                $m
            ) == 0) {
                die("ERROR ($def:$lineno): Invalid ZEND_VM_HANDLER definition.\n");
            }

            $code = (int)$m[1];
            $op   = $m[2];
            $op1  = parse_operand_spec($def, $lineno, $m[3], $flags1);
            $op2  = parse_operand_spec($def, $lineno, $m[4], $flags2);
            $flags = $flags1 | ($flags2 << 8);

            if ($code > $max_opcode) {
                $max_opcode = $code;
            }

            if (isset($opcodes[$code])) {
                die("ERROR ($def:$lineno): Opcode with code '$code' is already defined.\n");
            }
            if (isset($opnames[$op])) {
                die("ERROR ($def:$lineno): Opcode with name '$op' is already defined.\n");
            }
            $handler = $code;

            $opcodes[$code] = array("op"=>$op,"op1"=>$op1,"op2"=>$op2,"code"=>"","flags"=>$flags);
        } else {
            $opcodes[$handler]['code'] .= $line;
        }
    }

    ksort($opcodes);

    $f = fopen(__DIR__ . "/zend_vm_opcodes.h", "w+") or die("ERROR: Cannot create zend_vm_opcodes.h\n");
    fputs($f, "#pragma once\n\n");

    foreach ($opcodes as $code => $dsc) {
        $op = str_pad($dsc["op"], 20);
        fputs($f, "#define $op $code\n");
    }
    fclose($f);
    echo "zend_vm_opcodes.h generated successfully.\n";

    $f = fopen(__DIR__ . "/zend_vm_opcodes.cc", "w+") or die("ERROR: Cannot create zend_vm_opcodes.c\n");
    fputs($f, "#include \"zend_vm_opcodes.h\"\n\n");

    fputs($f, "static const char *zend_vm_opcodes_names[".($max_opcode + 1)."] = {\n");
    for ($i = 0; $i <= $max_opcode; $i++) {
        fputs($f, "\t".(isset($opcodes[$i]["op"])?'"'.$opcodes[$i]["op"].'"':"nullptr").",\n");
    }
    fputs($f, "};\n\n");

    fputs($f, "const char* zend_get_opcode_name(char opcode) {\n");
    fputs($f, "\treturn zend_vm_opcodes_names[opcode];\n");
    fputs($f, "}\n");

    fclose($f);
    echo "zend_vm_opcodes.cc generated successfully.\n";

    $f = fopen(__DIR__ . "/zend_vm_execute.h", "w+") or die("ERROR: Cannot create zend_vm_execute.h\n");
    fputs($f, "#pragma once\n\n");
    fputs($f, "#include <stdint.h>\n");
    fputs($f, "#include <stddef.h>\n");
    fputs($f, "#include \"zend_compile.h\"\n\n");

    gen_executor_code($f);
    echo "zend_vm_execute.h generated successfully.\n";
}

gen_vm(__DIR__ . "/zend_vm_def.h");
```

接着，我们执行这个脚本，就会生成文件`zend_vm_opcodes.h`、`zend_vm_opcodes.cc`、`zend_vm_execute.h`。

这里面有两个核心的函数`zend_vm_init`、`zend_execute`。

其中`zend_vm_init`会用一块内存来存放我们的`spec handler`的地址，这样，我们就可以通过上面所说的算法，来找到`spec handler`了。

`zend_execute`就非常的简单了，执行`opline`就好了。

接下来，我们只需要设置好每一个`opline`对应的`handler`即可。代码如下：

```cpp
// set opcode spec handler
void pass_two(zend_op_array *op_array) {
    for (size_t i = 0; i < op_array->last; i++) {
        zend_op *opline = &(op_array->opcodes[i]);
        zend_vm_set_opcode_handler(opline);
    }
}
```

最后，我们在文件`zend_language_parser.y`里面调用`zend_vm_init`、`pass_two`、`zend_execute`即可。

MLIR Operation的详解(ODS方式)

## 前言
本文是对MLIR [Operation Definition Specification (ODS)](https://mlir.llvm.org/docs/DefiningDialects/Operations/#operation-definition)文档的翻译和补充

 - 预备知识：有关MLIR和Dialect概述需要先在了解下

## 正文

除了`mlir::Op`的C++外，`MLIR`还支持以表驱动的方式定义`operations`和数据类型。这是通过`TableGen`实现的，它既是一种通用语言，也是用于生成特定领域信息(DSL)的工具。有关OP的可以简洁地定义到`TableGen`的record中，会在编译器构建时展开为等效的 `mlir::Op`C++模板文件（声明：.h.inc和定义：.cpp.inc）。

本手册详细解释了以表驱动方式定义Op的所有可用实现。至于如何添加MLIR graph，请参阅[入门教程](https://mlir.llvm.org/docs/Tutorials/QuickstartRewrites/)。

## 动机

`MLIR`实现了一种可扩展的`dialects`，`dialects`包含一组`operations`等内容。但是这种可扩展的生态导致了"stringly"的IR问题，例如，在优化和分析过程中重复的字符串比较，不直观的访问方法（比如，易出错的getOperand(3)方法，而不是通过自己属性命名的getStride()方法），冗长而通用的构造函数没有默认参数，冗长的文本IR dump方法等等。此外，Op验证分为：

最佳情况：集中的字符串到验证函数的映射，
中等情况：代码库中重复的验证，
最坏情况：没有验证函数。

一个好的方法是支持以表驱动的方式定义Op。对于每个 dialect，我们可以把所有的Op定义到一个地方，其中包含每个Op的所有成员和方法，包括其constraints，自定义汇编形式等。这个record也用于生成building, verification, parsing, printing, analysis类和方法等等。

> 什么是constraints？


## 优势

与C++模板相比，这种表驱动方法有许多好处，包括但不限于：

 - 单一的来源：我们致力于将关于op的所有信息都定义到`TableGen`中，这样读者就不需要在代码片段之间跳来跳去才能完全理解一个Op。

 - 消除模板代码：我们可以从记录中自动生成operand/attribute/result getter方法、Op build方法、Op verify方法以及许多其他实用方法。这极大地减少了定义新Op所需的模板代码。
 - 促进自动生成：这些Op信息记录的使用绝不仅限于Op定义本身。我们可以利用它们来驱动许多其他组件的自动生成，比如计算图的序列化。

## TableGen语法



使用`TableGen`作为定义Op信息的语言。`TableGen`本身只提供记录编写的语法; 可以在TableGen文件中（通常具有.td后缀的文件）找到的语法和构造。
 - TableGen的`class`类似于C++类；它可以模板化，也可以被继承和实例化。
 - TableGen的`def`类似于C++对象；它可以通过TableGen的class实例化来声明（例如，`def MyDef：MyClass<...>;`），也可以完全独立地声明（例如，`def MyDef;`）。它不能进一步模板化或实例化。
 - TableGen的`dag`是一种有向无环图的专用类型。一个dag包括一个运算符和零个或多个参数。其语法为（`operator arg0，arg1，argN`）。运算符可以是任何TableGen的`def`；参数可以是任何东西，包括`dag`本身。我们可以在运算符和参数上附加名称，如（`MyOp：$op_name MyArg：$arg_name`）。
 
 详细介绍请参阅TableGen[文档](https://llvm.org/docs/TableGen/ProgRef.html)。


## Operation定义

MLIR 定义了几个常见的构造来帮助Op定义，并通过相应的[TableGen backend](https://llvm.org/docs/TableGen/BackEnds.html#introduction)生成器:[OpDefinitionsGen](https://github.com/llvm/llvm-project/blob/main/mlir/tools/mlir-tblgen/OpDefinitionsGen.cpp) 生成它们的cpp语义。


位置：
[clangir/mlir/include/mlir/IR
/OpBase.td](https://github.com/Laity000/clangir/blob/main/mlir/include/mlir/IR/OpBase.td#L321)中的def。 主要包括：

 - Op class：它是定义operations的主要构造。在特化此类时，所有关于Op的信息都是定义在这个类中的。
 - Dialect class：属于同一个逻辑的Op放在同一Dialect中。Dialect类包含方言级别的信息。
 - OpTrait class hierarchy：它们用于指定这个Op需要的特殊属性和约束，比如Op是否具有副作用或者其输出与输入是否具有相同的shape。
 - ins/outs标记：这两个特殊标记内置到OpDefinitionsGen后端中。它们分别代表operands/attributes（Op的输入参数）和results（Op的输出）的定义。
- TypeConstraint class hierarchy：它们用于指定操作数参数或输出的约束。一个值得注意的子类是`Type`，它代表常见C++ type的约束。
 - AttrConstraint class hierarchy：它们用于指定属性的约束。一个值得注意的子类是`Attr`，它代表公共的属性约束。


 
通过实例化Op类的方式来定义operation，以满足其所需的所有字段。例如，tf.AvgPool被定义为:
```c++
def TF_AvgPoolOp : TF_Op<"AvgPool", [NoMemoryEffect]> {
  let summary = "Performs average pooling on the input.";

  let description = [{
Each entry in `output` is the mean of the corresponding size `ksize`
window in `value`.
  }];

  let arguments = (ins
    TF_FpTensor:$value,

    ConfinedAttr<I64ArrayAttr, [ArrayMinCount<4>]>:$ksize,
    ConfinedAttr<I64ArrayAttr, [ArrayMinCount<4>]>:$strides,
    TF_AnyStrAttrOf<["SAME", "VALID"]>:$padding,
    DefaultValuedAttr<TF_ConvertDataFormatAttr, "NHWC">:$data_format
  );

  let results = (outs
    TF_FpTensor:$output
  );

  TF_DerivedOperandTypeAttr T = TF_DerivedOperandTypeAttr<0>;
}
```


在接下来的内容中，将详细介绍Op所有所需的字段。完整字段列表如下。

[clangir/mlir/include/mlir/IR
/OpBase.td](https://github.com/Laity000/clangir/blob/main/mlir/include/mlir/IR/OpBase.td#L321)
```c++
// Base class for all ops.
class Op<Dialect dialect, string mnemonic, list<Trait> props = []> {
  // The dialect of the op.
  Dialect opDialect = dialect;

  // The mnemonic of the op.
  string opName = mnemonic;

  // The C++ namespace to use for this op.
  string cppNamespace = dialect.cppNamespace;

  // One-line human-readable description of what the op does.
  string summary = "";

  // Additional, longer human-readable description of what the op does.
  string description = "";

  // Optional. The group of ops this op is part of.
  OpDocGroup opDocGroup = ?;

  // Dag containing the arguments of the op. Default to 0 arguments.
  dag arguments = (ins);

  // The list of results of the op. Default to 0 results.
  dag results = (outs);

  // The list of regions of the op. Default to 0 regions.
  dag regions = (region);

  // The list of successors of the op. Default to 0 successors.
  dag successors = (successor);

  list<OpBuilder> builders = ?;

  bit skipDefaultBuilders = 0;

  string assemblyFormat = ?;

  bit hasCustomAssemblyFormat = 0;

  bit hasVerifier = 0;

  bit hasRegionVerifier = 0;

  // Whether this op has associated canonicalization patterns.
  bit hasCanonicalizer = 0;

  // Whether this op has a static "canonicalize" method to perform "match and
  // rewrite patterns".
  bit hasCanonicalizeMethod = 0;

  // Whether this op has a folder.
  bit hasFolder = 0;

  bit useCustomPropertiesEncoding = 0;

  // Op traits.
  // Note: The list of traits will be uniqued by ODS.
  list<Trait> traits = props;

  code extraClassDeclaration = ?;

  code extraClassDefinition = ?;
}
```

例子
```c++
def TF_AvgPoolOp : TF_Op<"AvgPool", [NoMemoryEffect]> {
  let summary = "Performs average pooling on the input.";

  let description = [{
Each entry in `output` is the mean of the corresponding size `ksize`
window in `value`.
  }];

  let arguments = (ins
    TF_FpTensor:$value,

    ConfinedAttr<I64ArrayAttr, [ArrayMinCount<4>]>:$ksize,
    ConfinedAttr<I64ArrayAttr, [ArrayMinCount<4>]>:$strides,
    TF_AnyStrAttrOf<["SAME", "VALID"]>:$padding,
    DefaultValuedAttr<TF_ConvertDataFormatAttr, "NHWC">:$data_format
  );

  let results = (outs
    TF_FpTensor:$output
  );

  TF_DerivedOperandTypeAttr T = TF_DerivedOperandTypeAttr<0>;
}
```

### Operation name


操作名是 MLIR 中操作的唯一标识符，例如 TensorFlow 方言中的加法操作 `tf.Add`。这相当于汇编语言中的助记符。它用于在文本格式中进行解析和打印。它还用于图重写中的模式匹配。

完整的操作名称由方言名称和操作名称组成，前者通过方言提供，后者作为 Op 类的第二个模板参数提供。

### Operation documentation

包括一句简要的`summary`和一个更长、易读的`description`。它们将用于驱动方言文档的自动生成。

```c++
let summary = "...";

let description = [{
...
}];
```
 - description应该以 Markdown 语法编写。
 - 将文档放在操作定义的开始处。
 - 摘要应该简短而简洁。它应该是以大写字母开头，没有尾随标点符号的一句话。在描述中提供扩展解释。

 ### Operation arguments



有两种类型的参数：`operands`和`attributes`。操作数是由其他Op产生的运行时值；而属性是编译时已知的常量值，包括两个类别：

自然属性：这些属性影响操作的行为（例如，卷积的填充）；

派生属性：这些属性不是必须用来来定义Op，而是从Op的信息中派生出来的。例如，类型的输出shape。这主要用于便利接口生成或与其他框架进行交互。

所有派生属性都应该能够作为属性实现。也就是说，即使它们没有实现，也应该可以存储为属性。

操作数和属性都是属于Tablegen dag-typed节点：

```c++
let arguments = (ins
  <type-constraint>:$<operand-name>,
  ...
  <attr-constraint>:$<attr-name>,
  ...
);
```

这里的 `<type-constraint>` 是来自 `TypeConstraint class hierarchy`的 TableGen def。同样地，`<attr-constraint>` 是来自 `AttrConstraint class hierarchy`的 TableGen def。更多信息请参阅下一节的约束。

操作数和属性之间的相对顺序没有要求；它们可以自由组合。操作数本身的相对顺序很重要。对于每个命名参数，将生成一个该命名的getter方法，返回具有返回类型的参数（对于属性来说，返回类型将从存储类型构造，而对于操作数来说，将为 Value）。每个属性的原始值（例如，存储的值）也可以通过生成的 <name>Attr 获取器进行访问，以在转换通道中使用，其中更用户友好的返回类型不太合适。

所有的参数应该被命名为：

提供文档，
驱动自动生成的获取方法，
为其他地方（如约束条件）提供引用的手柄。

#### Variadic operands

要声明可变长的操作数，可以使用`Variadic<...>`来包装操作数的TypeConstraint。

通常Op要么没有可变长的操作数，或者只有一个可变长的操作数。对于后一种情况，很容易推断出动态操作数用于静态可变数量操作数定义的情况。但是，如果一个Op有多个可变长的操作数（或者是optional），则无法在没有来自操作的进一步信息的情况下将动态操作数归因于相应的静态可变数量操作数定义。因此，需要`SameVariadicOperandSize`或`AttrSizedOperandSegments` trait来指示所有可变长操作数具有相同数量的动态值。
```c++
def MixedVOperandOp1 : TEST_Op<"mixed_variadic_in1",
                               [SameVariadicOperandSize]> {
  let arguments = (ins
    Variadic<I32>:$input1,
    F32:$input2,
    Variadic<I32>:$input3
  );
}
```

#### VariadicOfVariadic operands

声明一个具有可变数量的子范围的可变操作数，将操作数的TypeConstraint用`VariadicOfVariadic<..., "<segment-attribute-name>">`进行包装。

VariadicOfVariadic的第二个字段是一个I32ElementsAttr参数的名称，该参数包含可变子范围的大小。在确定子范围的大小或更新子范围的大小时，将使用此属性。

```tablegen
def TestOpFoldWithFoldAdaptor
  : TEST_Op<"fold_with_fold_adaptor",
      [AttrSizedOperandSegments, NoTerminator]> {
  let arguments = (ins
    I32:$op,
    DenseI32ArrayAttr:$attr,
    Variadic<I32>:$variadic,
    VariadicOfVariadic<I32, "attr">:$var_of_var
  );
```

#### Optional operands

To declare an optional operand, wrap the `TypeConstraint` for the operand with
`Optional<...>`.

Normally operations have no optional operands or just one optional operand. For
the latter case, it is easy to deduce which dynamic operands are for the static
operand definition. However, if an operation has more than one variable length
operands (either optional or variadic), it would be impossible to attribute
dynamic operands to the corresponding static variadic operand definitions
without further information from the operation. Therefore, either the
`SameVariadicOperandSize` or `AttrSizedOperandSegments` trait is needed to
indicate that all variable length operands have the same number of dynamic
values.

#### Optional attributes

To declare an optional attribute, wrap the `AttrConstraint` for the attribute
with `OptionalAttr<...>`.

#### Attributes with default values

To declare an attribute with a default value, wrap the `AttrConstraint` for the
attribute with `DefaultValuedAttr<..., "...">`.

The second parameter to `DefaultValuedAttr` should be a string containing the
C++ default value. For example, a float default value should be specified as
like `"0.5f"`, and an integer array default value should be specified as like
`"{1, 2, 3}"`.

The generated operation printing function will not print default-valued
attributes when the attribute value is equal to the default.

#### Confining attributes

`ConfinedAttr` is provided as a general mechanism to help modelling further
constraints on attributes beyond the ones brought by value types. You can use
`ConfinedAttr` to compose complex constraints out of more primitive ones. For
example, a 32-bit integer attribute whose minimum value must be 10 can be
expressed as `ConfinedAttr<I32Attr, [IntMinValue<10>]>`.

Right now, the following primitive constraints are supported:

*   `IntMinValue<N>`: Specifying an integer attribute to be greater than or
    equal to `N`
*   `IntMaxValue<N>`: Specifying an integer attribute to be less than or equal
    to `N`
*   `ArrayMinCount<N>`: Specifying an array attribute to have at least `N`
    elements
*   `IntArrayNthElemEq<I, N>`: Specifying an integer array attribute's `I`-th
    element to be equal to `N`
*   `IntArrayNthElemMinValue<I, N>`: Specifying an integer array attribute's
    `I`-th element to be greater than or equal to `N`
*   `IntArrayNthElemMaxValue<I, N>`: Specifying an integer array attribute's
    `I`-th element to be less than or equal to `N`
*   `IntArrayNthElemInRange<I, M, N>`: Specifying an integer array attribute's
    `I`-th element to be greater than or equal to `M` and less than or equal to `N`

TODO: Design and implement more primitive constraints

### Operation regions

The regions of an operation are specified inside of the `dag`-typed `regions`,
led by `region`:

```tablegen
let regions = (region
  <region-constraint>:$<region-name>,
  ...
);
```

#### Variadic regions

Similar to the `Variadic` class used for variadic operands and results,
`VariadicRegion<...>` can be used for regions. Variadic regions can currently
only be specified as the last region in the regions list.

### Operation results

Similar to operands, results are specified inside the `dag`-typed `results`, led
by `outs`:

```tablegen
let results = (outs
  <type-constraint>:$<result-name>,
  ...
);
```

#### Variadic results

Similar to variadic operands, `Variadic<...>` can also be used for results. And
similarly, `SameVariadicResultSize` for multiple variadic results in the same
operation.

### Operation successors

For terminator operations, the successors are specified inside of the
`dag`-typed `successors`, led by `successor`:

```tablegen
let successors = (successor
  <successor-constraint>:$<successor-name>,
  ...
);
```

#### Variadic successors

Similar to the `Variadic` class used for variadic operands and results,
`VariadicSuccessor<...>` can be used for successors. Variadic successors can
currently only be specified as the last successor in the successor list.

### Operation traits and constraints

Traits are operation properties that affect syntax or semantics. MLIR C++ models
various traits in the `mlir::OpTrait` namespace.

Both operation traits, [interfaces](../Interfaces.md/#utilizing-the-ods-framework),
and constraints involving multiple operands/attributes/results are provided as
the third template parameter to the `Op` class. They should be deriving from
the `OpTrait` class. See [Constraints](#constraints) for more information.

### Builder methods

For each operation, there are a few builders automatically generated based on
the arguments and returns types. For example, given the following op definition:

```tablegen
def MyOp : ... {
  let arguments = (ins
    I32:$i32_operand,
    F32:$f32_operand,
    ...,

    I32Attr:$i32_attr,
    F32Attr:$f32_attr,
    ...
  );

  let results = (outs
    I32:$i32_result,
    F32:$f32_result,
    ...
  );
}
```

The following builders are generated:

```c++
// All result-types/operands/attributes have one aggregate parameter.
static void build(OpBuilder &odsBuilder, OperationState &odsState,
                  TypeRange resultTypes,
                  ValueRange operands,
                  ArrayRef<NamedAttribute> attributes);

// Each result-type/operand/attribute has a separate parameter. The parameters
// for attributes are of mlir::Attribute types.
static void build(OpBuilder &odsBuilder, OperationState &odsState,
                  Type i32_result, Type f32_result, ...,
                  Value i32_operand, Value f32_operand, ...,
                  IntegerAttr i32_attr, FloatAttr f32_attr, ...);

// Each result-type/operand/attribute has a separate parameter. The parameters
// for attributes are raw values unwrapped with mlir::Attribute instances.
// (Note that this builder will not always be generated. See the following
// explanation for more details.)
static void build(OpBuilder &odsBuilder, OperationState &odsState,
                  Type i32_result, Type f32_result, ...,
                  Value i32_operand, Value f32_operand, ...,
                  APInt i32_attr, StringRef f32_attr, ...);

// Each operand/attribute has a separate parameter but result type is aggregate.
static void build(OpBuilder &odsBuilder, OperationState &odsState,
                  TypeRange resultTypes,
                  Value i32_operand, Value f32_operand, ...,
                  IntegerAttr i32_attr, FloatAttr f32_attr, ...);

// All operands/attributes have aggregate parameters.
// Generated if return type can be inferred.
static void build(OpBuilder &odsBuilder, OperationState &odsState,
                  ValueRange operands, ArrayRef<NamedAttribute> attributes);

// (And manually specified builders depending on the specific op.)
```

The first form provides basic uniformity so that we can create ops using the
same form regardless of the exact op. This is particularly useful for
implementing declarative pattern rewrites.

The second and third forms are good for use in manually written code, given that
they provide better guarantee via signatures.

The third form will be generated if any of the op's attribute has different
`Attr.returnType` from `Attr.storageType` and we know how to build an attribute
from an unwrapped value (i.e., `Attr.constBuilderCall` is defined.)
Additionally, for the third form, if an attribute appearing later in the
`arguments` list has a default value, the default value will be supplied in the
declaration. This works for `BoolAttr`, `StrAttr`, `EnumAttr` for now and the
list can grow in the future. So if possible, the default-valued attribute should be
placed at the end of the `arguments` list to leverage this feature. (This
behavior is essentially due to C++ function parameter default value placement
restrictions.) Otherwise, the builder of the third form will still be generated
but default values for the attributes not at the end of the `arguments` list
will not be supplied in the builder's signature.

ODS will generate a builder that doesn't require the return type specified if

*   Op implements InferTypeOpInterface interface;
*   All return types are either buildable types or are the same as a given
    operand (e.g., `AllTypesMatch` constraint between operand and result);

And there may potentially exist other builders depending on the specific op;
please refer to the
[generated C++ file](#run-mlir-tblgen-to-see-the-generated-content) for the
complete list.

#### Custom builder methods

However, if the above cases cannot satisfy all needs, you can define additional
convenience build methods in the `builders` field as follows.

```tablegen
def MyOp : Op<"my_op", []> {
  let arguments = (ins F32Attr:$attr);

  let builders = [
    OpBuilder<(ins "float":$val)>
  ];
}
```

The `builders` field is a list of custom builders that are added to the Op
class. In this example, we provide a convenience builder that takes a floating
point value instead of an attribute. The `ins` prefix is common to many function
declarations in ODS, which use a TableGen [`dag`](#tablegen-syntax). What
follows is a comma-separated list of types (quoted string) and names prefixed
with the `$` sign. This will generate the declaration of a builder method that
looks like:

```c++
class MyOp : /*...*/ {
  /*...*/
  static void build(::mlir::OpBuilder &builder, ::mlir::OperationState &state,
                    float val);
};
```

Note that the method has two additional leading arguments. These arguments are
useful to construct the operation. In particular, the method must populate
`state` with attributes, operands, regions and result types of the operation to
be constructed. `builder` can be used to construct any IR objects that belong to
the Op, such as types or nested operations. Since the type and name are
generated as is in the C++ code, they should be valid C++ constructs for a type
(in the namespace of the Op) and an identifier (e.g., `class` is not a valid
identifier).

Implementations of the builder can be provided directly in ODS, using TableGen
code block as follows.

```tablegen
def MyOp : Op<"my_op", []> {
  let arguments = (ins F32Attr:$attr);

  let builders = [
    OpBuilder<(ins "float":$val), [{
      $_state.addAttribute("attr", $_builder.getF32FloatAttr(val));
    }]>
  ];
}
```

The equivalents of `builder` and `state` arguments are available as `$_builder`
and `$_state` special variables. The named arguments listed in the `ins` part
are available directly, e.g. `val`. The body of the builder will be generated by
substituting special variables and should otherwise be valid C++. While there is
no limitation on the code size, we encourage one to define only short builders
inline in ODS and put definitions of longer builders in C++ files.

Finally, if some arguments need a default value, they can be defined using
`CArg` to wrap the type and this value as follows.

```tablegen
def MyOp : Op<"my_op", []> {
  let arguments = (ins F32Attr:$attr);

  let builders = [
    OpBuilder<(ins CArg<"float", "0.5f">:$val), [{
      $_state.addAttribute("attr", $_builder.getF32FloatAttr(val));
    }]>
  ];
}
```

The generated code will use default value in the declaration, but not in the
definition, as required by C++.

```c++
/// Header file.
class MyOp : /*...*/ {
  /*...*/
  static void build(::mlir::OpBuilder &builder, ::mlir::OperationState &state,
                    float val = 0.5f);
};

/// Source file.
MyOp::build(::mlir::OpBuilder &builder, ::mlir::OperationState &state,
            float val) {
  state.addAttribute("attr", builder.getF32FloatAttr(val));
}
```

### Custom parser and printer methods

Functions to parse and print the operation's custom assembly form.

### Custom verifier code

Verification code will be automatically generated for
[constraints](#constraints) specified on various entities of the op. To perform
_additional_ verification, you can use

```tablegen
let hasVerifier = 1;
let hasRegionVerifier = 1;
```

This will generate `LogicalResult verify()`/`LogicalResult verifyRegions()`
method declarations on the op class that can be defined with any additional
verification constraints. For verificaiton which needs to access the nested
operations, you should use `hasRegionVerifier` to ensure that it won't access
any ill-formed operation. Except that, The other verifications can be
implemented with `hasVerifier`. Check the next section for the execution order
of these verification methods.

#### Verification Ordering

The verification of an operation involves several steps,

1. StructuralOpTrait will be verified first, they can be run independently.
2. `verifyInvariants` which is constructed by ODS, it verifies the type,
   attributes, .etc.
3. Other Traits/Interfaces that have marked their verifier as `verifyTrait` or
   `verifyWithRegions=0`.
4. Custom verifier which is defined in the op and has been marked `hasVerifier=1`

If an operation has regions, then it may have the second phase,

1. Traits/Interfaces that have marked their verifier as `verifyRegionTrait` or
   `verifyWithRegions=1`. This implies the verifier needs to access the
   operations in its regions.
2. Custom verifier which is defined in the op and has been marked
   `hasRegionVerifier=1`

Note that the second phase will be run after the operations in the region are
verified. Verifiers further down the order can rely on certain invariants being
verified by a previous verifier and do not need to re-verify them.

#### Emitting diagnostics in custom verifiers

Custom verifiers should avoid printing operations using custom operation
printers, because they require the printed operation (and sometimes its parent
operation) to be verified first. In particular, when emitting diagnostics,
custom verifiers should use the `Error` severity level, which prints operations
in generic form by default, and avoid using lower severity levels (`Note`,
`Remark`, `Warning`).

### Declarative Assembly Format

The custom assembly form of the operation may be specified in a declarative
string that matches the operations operands, attributes, etc. With the ability
to express additional information that needs to be parsed to build the
operation:

```tablegen
def CallOp : Std_Op<"call", ...> {
  let arguments = (ins FlatSymbolRefAttr:$callee, Variadic<AnyType>:$args);
  let results = (outs Variadic<AnyType>);

  let assemblyFormat = [{
    $callee `(` $args `)` attr-dict `:` functional-type($args, results)
  }];
}
```

The format is comprised of three components:

#### Directives

A directive is a type of builtin function, with an optional set of arguments.
The available directives are as follows:

*   `attr-dict`

    -   Represents the attribute dictionary of the operation.

*   `attr-dict-with-keyword`

    -   Represents the attribute dictionary of the operation, but prefixes the
        dictionary with an `attributes` keyword.

*   `custom` < UserDirective > ( Params )

    -   Represents a custom directive implemented by the user in C++.
    -   See the [Custom Directives](#custom-directives) section below for more
        details.

*   `functional-type` ( inputs , outputs )

    -   Formats the `inputs` and `outputs` arguments as a
        [function type](../Dialects/Builtin.md/#functiontype).
    -   The constraints on `inputs` and `outputs` are the same as the `input` of
        the `type` directive.

*   `oilist` ( \`keyword\` elements | \`otherKeyword\` elements ...)

    -   Represents an optional order-independent list of clauses. Each clause
        has a keyword and corresponding assembly format.
    -   Each clause can appear 0 or 1 time (in any order).
    -   Only literals, types and variables can be used within an oilist element.
    -   All the variables must be optional or variadic.

*   `operands`

    -   Represents all of the operands of an operation.

*   `ref` ( input )

    -   Represents a reference to the a variable or directive, that must have
        already been resolved, that may be used as a parameter to a `custom`
        directive.
    -   Used to pass previously parsed entities to custom directives.
    -   The input may be any directive or variable, aside from `functional-type`
        and `custom`.

*   `regions`

    -   Represents all of the regions of an operation.

*   `results`

    -   Represents all of the results of an operation.

*   `successors`

    -   Represents all of the successors of an operation.

*   `type` ( input )

    -   Represents the type of the given input.
    -   `input` must be either an operand or result [variable](#variables), the
        `operands` directive, or the `results` directive.

*   `qualified` ( type_or_attribute )

    -   Wraps a `type` directive or an attribute parameter.
    -   Used to force printing the type or attribute prefixed with its dialect
        and mnemonic. For example the `vector.multi_reduction` operation has a
        `kind` attribute ; by default the declarative assembly will print:
        `vector.multi_reduction <minf>, ...` but using `qualified($kind)` in the
        declarative assembly format will print it instead as:
        `vector.multi_reduction #vector.kind<minf>, ...`.

#### Literals

A literal is either a keyword or punctuation surrounded by \`\`.

The following are the set of valid punctuation:

`:`, `,`, `=`, `<`, `>`, `(`, `)`, `{`, `}`, `[`, `]`, `->`, `?`, `+`, `*`

The following are valid whitespace punctuation:

`\n`, ` `

The `\n` literal emits a newline an indents to the start of the operation. An
example is shown below:

```tablegen
let assemblyFormat = [{
  `{` `\n` ` ` ` ` `this_is_on_a_newline` `\n` `}` attr-dict
}];
```

```mlir
%results = my.operation {
  this_is_on_a_newline
}
```

An empty literal \`\` may be used to remove a space that is inserted implicitly
after certain literal elements, such as `)`/`]`/etc. For example, "`]`" may
result in an output of `]` it is not the last element in the format. "`]` \`\`"
would trim the trailing space in this situation.

#### Variables

A variable is an entity that has been registered on the operation itself, i.e.
an argument(attribute or operand), region, result, successor, etc. In the
`CallOp` example above, the variables would be `$callee` and `$args`.

Attribute variables are printed with their respective value type, unless that
value type is buildable. In those cases, the type of the attribute is elided.

#### Custom Directives

The declarative assembly format specification allows for handling a large
majority of the common cases when formatting an operation. For the operations
that require or desire specifying parts of the operation in a form not supported
by the declarative syntax, custom directives may be specified. A custom
directive essentially allows for users to use C++ for printing and parsing
subsections of an otherwise declaratively specified format. Looking at the
specification of a custom directive above:

```
custom-directive ::= `custom` `<` UserDirective `>` `(` Params `)`
```

A custom directive has two main parts: The `UserDirective` and the `Params`. A
custom directive is transformed into a call to a `print*` and a `parse*` method
when generating the C++ code for the format. The `UserDirective` is an
identifier used as a suffix to these two calls, i.e., `custom<MyDirective>(...)`
would result in calls to `parseMyDirective` and `printMyDirective` within the
parser and printer respectively. `Params` may be any combination of variables
(i.e. Attribute, Operand, Successor, etc.), type directives, `attr-dict`, and
strings of C++ code. The type directives must refer to a variable, but that
variable need not also be a parameter to the custom directive.

The arguments to the `parse<UserDirective>` method are firstly a reference to
the `OpAsmParser`(`OpAsmParser &`), and secondly a set of output parameters
corresponding to the parameters specified in the format. The mapping of
declarative parameter to `parse` method argument is detailed below:

*   Attribute Variables
    -   Single: `<Attribute-Storage-Type>(e.g. Attribute) &`
    -   Optional: `<Attribute-Storage-Type>(e.g. Attribute) &`
*   Operand Variables
    -   Single: `OpAsmParser::UnresolvedOperand &`
    -   Optional: `Optional<OpAsmParser::UnresolvedOperand> &`
    -   Variadic: `SmallVectorImpl<OpAsmParser::UnresolvedOperand> &`
    -   VariadicOfVariadic:
        `SmallVectorImpl<SmallVector<OpAsmParser::UnresolvedOperand>> &`
*   Ref Directives
    -   A reference directive is passed to the parser using the same mapping as
        the input operand. For example, a single region would be passed as a
        `Region &`.
*   Region Variables
    -   Single: `Region &`
    -   Variadic: `SmallVectorImpl<std::unique_ptr<Region>> &`
*   Successor Variables
    -   Single: `Block *&`
    -   Variadic: `SmallVectorImpl<Block *> &`
*   Type Directives
    -   Single: `Type &`
    -   Optional: `Type &`
    -   Variadic: `SmallVectorImpl<Type> &`
    -   VariadicOfVariadic: `SmallVectorImpl<SmallVector<Type>> &`
*   `attr-dict` Directive: `NamedAttrList &`

When a variable is optional, the value should only be specified if the variable
is present. Otherwise, the value should remain `None` or null.

The arguments to the `print<UserDirective>` method is firstly a reference to the
`OpAsmPrinter`(`OpAsmPrinter &`), second the op (e.g. `FooOp op` which can be
`Operation *op` alternatively), and finally a set of output parameters
corresponding to the parameters specified in the format. The mapping of
declarative parameter to `print` method argument is detailed below:

*   Attribute Variables
    -   Single: `<Attribute-Storage-Type>(e.g. Attribute)`
    -   Optional: `<Attribute-Storage-Type>(e.g. Attribute)`
*   Operand Variables
    -   Single: `Value`
    -   Optional: `Value`
    -   Variadic: `OperandRange`
    -   VariadicOfVariadic: `OperandRangeRange`
*   Ref Directives
    -   A reference directive is passed to the printer using the same mapping as
        the input operand. For example, a single region would be passed as a
        `Region &`.
*   Region Variables
    -   Single: `Region &`
    -   Variadic: `MutableArrayRef<Region>`
*   Successor Variables
    -   Single: `Block *`
    -   Variadic: `SuccessorRange`
*   Type Directives
    -   Single: `Type`
    -   Optional: `Type`
    -   Variadic: `TypeRange`
    -   VariadicOfVariadic: `TypeRangeRange`
*   `attr-dict` Directive: `DictionaryAttr`

When a variable is optional, the provided value may be null. When a variable is
referenced in a custom directive parameter using `ref`, it is passed in by
value. Referenced variables to `print<UserDirective>` are passed as the same as
bound variables, but referenced variables to `parse<UserDirective>` are passed
like to the printer.

A custom directive can take a string of C++ code as a parameter. The code is
pasted verbatim in the calls to the custom parser and printers, with the
substitutions `$_builder` and `$_ctxt`. String literals can be used to
parameterize custom directives.

#### Optional Groups
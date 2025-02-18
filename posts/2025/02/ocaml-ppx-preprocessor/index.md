# OCaml的PPX预处理机制解析


一般编程语言都具备预处理机制，用于在编译前对源代码进行转换。OCaml中存在两种预处理器机制：一种基于纯文本替换，另一种则基于抽象语法树（AST）的变换，后者便是著名的PPX预处理机制，全称为Pre-Processor eXtension。本文将深入解析PPX预处理机制的原理及其应用。

## 什么是PPX

PPX是一种OCaml的预处理机制，允许开发者在编译前对源代码进行转换。它通过定义转换器（称为“扩展”）来处理特定的语法或代码模式，从而生成新的代码。一般的传统预处理器，比如C语言预处理器cpp，直接在源码层面进行文本替换，而PPX与之最大的不同在于，PPX直接操作OCaml的抽象语法树（AST），使得代码转换更加精确和安全。

## 为什么使用PPX
PPX具有以下显著优势：
- **增强语言功能**：通过PPX，开发者可以引入新的语法和功能，例如自定义的控制结构或领域特定语言（DSL），从而扩展OCaml的表达能力。
- **方便获取类型信息**：PPX处理阶段位于编译前期，能够方便地获取类型信息，有效弥补了OCaml程序在运行时类型信息被擦除的不足。
- **类型安全**：由于PPX操作的是OCaml的AST，转换过程是类型安全的，避免了传统预处理器可能引入的类型错误。

## PPX原理解析

在OCaml编译器将源代码编译为字节码或机器码的过程中，源代码首先被转换为一种称为抽象语法树（AST）的数据结构，OCaml实现中称之为Parsetree。PPX转换器本质上是对Parsetree进行变换，它以一个Parsetree作为输入，返回一个变换后的Parsetree，再交给编译器的后续步骤继续处理。具体而言，开发者可以在源代码中加入特定的注释或语法，PPX转换器会识别这些标记并应用相应的转换规则，将代码转换为标准的OCaml代码。这种转换是在编译前静态完成的，因此不会对编译后的性能产生影响。
### Attributes和Derivers
Attributes是附加在AST节点上的附加信息，其语法形式为`[@attribute_name payload]`，其中`attribute_name`是该attribute的名字，`payload`是合法的OCaml源码。`@`的数量决定了该attribute具体附加到哪一个AST节点：`@`表示最近的节点，`@@`表示最近的block，`@@@`表示它是一个独立的浮动attribute，不附加到任何AST节点。例如，`[@@deriving yojson]`是一个附加在type定义上的attribute：
```ocaml
type int_pair = (int * int) [@@deriving yojson]
```
与attributes紧密相连的一个常见概念是derivers，它本质上是一种特殊的PPX。对于这种PPX，我们可以用attribute语法将其添加到程序中，附加到某个代码块上。在编译程序时，该PPX会处理这个代码块，并将处理后生成的代码添加到该代码块的后面。常见的derivers包括：
- `ppx_show`：可以为某种type生成pretty printer。
- `ppx_yojson_conv`：可以自动生成将某种type转换为json格式的转换函数。
### Extension nodes和extenders
Extension nodes相当于Parsetree中的“空洞”，其语法形式为`[%extension_name payload]`，其中`%`的数量决定了extension node的类型：`%`表示内部节点，例如表达式或模式；`%%`表示顶层节点，例如structure中的条目。例如：
```ocaml
let v = [%html &#34;&lt;a href=&#39;ocaml.org&#39;&gt;OCaml!&lt;/a&gt;&#34;]
```
一个extension node相当于一个空位，它会被一种特殊的PPX填充，而这种PPX就叫extender。具体而言，extenders会匹配extension node的名字，匹配到之后就会根据它的`payload`来生成代码，并将生成的代码替换到extension node本来的位置。需要注意的是，这个生成代码的过程只依赖于`payload`，而不会考虑该node的上下文信息。常见的extenders例子包括：`ppx_expect`可以从`payload`直接生成CRAM tests。

## PPX的使用示例

假设我们希望在OCaml中实现为任意类型自动生成打印函数。我们可以使用`ppx_deriving`扩展来实现这一目标。

1. 安装`ppx_deriving`：
```bash
opam install ppx_deriving
```

2. 定义type，并加上`[@@deriving show]`
```ocaml
type person = {
  name : string;
  age : int;
}
[@@deriving show]
type shape =
  | Circle of float
  | Rectangle of float * float
  | Person of person
[@@deriving show]
let () =
  let p = { name = &#34;Alice&#34;; age = 30 } in
  let s = Circle 3.0 in
  let s2 = Person p in
  (* Printf.printf &#34;%s\n&#34; (show_person p); *)
  Printf.printf &#34;%s\n&#34; (show_shape s);
  Printf.printf &#34;%s\n&#34; (show_shape s2)
```

这将自动为`person`类型和shape类型生成`show`函数，用于将其转换为字符串表示。这种自动化的代码生成大大提高了开发效率。
3. 编写代码调用show
```ocaml
let () =
  let p = { name = &#34;Alice&#34;; age = 30 } in
  let s = Circle 3.0 in
  let s2 = Person p in
  (* Printf.printf &#34;%s\n&#34; (show_person p); *)
  Printf.printf &#34;%s\n&#34; (show_shape s);
  Printf.printf &#34;%s\n&#34; (show_shape s2)
```
4. 调用ocamlopt编译
```bash
$ ocamlfind ocamlopt -o main -package ppx_deriving.show -linkpkg main.ml
```
5. 运行输出
```
$ ./main
(Circle 3.)
(Person { name = &#34;Alice&#34;; age = 30 })
```

## 编写自定义PPX扩展

编写自定义PPX扩展需要深入了解OCaml的抽象语法树（AST）和编译器的工作原理。通常，开发者需要定义一个转换器，该转换器接受一个AST并返回一个新的AST。
### ppxlib
历史上曾出现过一些用于帮助编写PPX扩展的库，如`ppx_tools`和`ppx_deriving`。然而，随着OCaml生态系统的发展，`ppxlib`逐渐成为主流。`ppxlib`是一个现代化的OCaml库，旨在简化和增强OCaml编译器扩展，尤其是PPX重写器的开发。它通过提供更高层次的抽象，使得编写和管理PPX重写器变得更加简洁和高效。`ppxlib`主要由Jane Street开发，综合了多个旧有PPX项目的能力，致力于提高PPX重写器的性能、可维护性和互操作性。

从工作原理上看，`ppxlib`对PPX重写器的核心概念进行了显著改进。传统的PPX重写器往往将整个AST视为一个黑盒子进行处理，而`ppxlib`则将扩展点视为可在编译时求值的函数，采用自上而下的求值方式。这样，重写过程被拆分为多个小函数，这些函数按照指定的顺序在编译时执行，从而提供了更高效、更灵活的处理能力。

### 用ppxlib编写PPX扩展
本节通过两个示例来直观展示PPX扩展的编写方法。示例1针对extension nodes，示例2针对attributes。

#### 示例1
假设我们要通过PPX机制，将源码中的`[%get_env &#34;SOME_ENV_VAR&#34;]`在编译期替换成环境变量`SOME_ENV_VAR`的值，我们可以这样做：
1. 安装ppxlib
```bash
opam install ppxlib
```
2. 用OCaml编写扩展文件ppx_get_env.ml
```ocaml
open Ppxlib
let expand ~ctxt env_var =
  let loc = Expansion_context.Extension.extension_point_loc ctxt in
  match Sys.getenv env_var with
  | value -&gt; Ast_builder.Default.estring ~loc value
  | exception Not_found -&gt;
      let ext =
        Location.error_extensionf ~loc &#34;The environement variable %s is unbound&#34;
          env_var
      in
      Ast_builder.Default.pexp_extension ~loc ext
let my_extension =
  Extension.V3.declare &#34;get_env&#34; Extension.Context.expression
    Ast_pattern.(single_expr_payload (estring __))
    expand
let rule = Ppxlib.Context_free.Rule.extension my_extension
let () = Driver.register_transformation ~rules:[ rule ] &#34;get_env&#34;
```
解释一下，最后一行`Driver.register_transformation`将自定义的转换规则注册到ppxlib中。此外，通过`Extension.V3.declare`来声明一个PPX extension，其第一个参数就是extension名，最后一个参数是一个函数expand，它实现真正的转换逻辑。
3. 编写main.ml
```ocaml
let () = print_string [%get_env &#34;MY_VAR&#34;]
```
4. 编写构建脚本dune文件
```dune
(library
 (name ppx_get_env)
 (kind ppx_rewriter)
 (libraries ppxlib))
(executable
 (public_name dppx)
 (name main)
 (preprocess (pps ppx_get_env)))
```
5. 构建，执行
```bash
dune build
dune exec bin/main.exe
```

#### 示例2
假设我们要通过PPX机制，为record类型定义自动生成每个field的访问函数，我们可以这样做：
1. 编写ppx_deriving_accessors.ml
```ocaml
open Ppxlib
module List = ListLabels
open Ast_builder.Default
let accessor_impl (ld : label_declaration) =
  let loc = ld.pld_loc in
  pstr_value ~loc Nonrecursive
    [
      {
        pvb_pat = ppat_var ~loc ld.pld_name;
        pvb_expr =
          pexp_fun ~loc Nolabel None
            (ppat_var ~loc { loc; txt = &#34;x&#34; })
            (pexp_field ~loc
               (pexp_ident ~loc { loc; txt = lident &#34;x&#34; })
               { loc; txt = lident ld.pld_name.txt });
        pvb_attributes = [];
        pvb_loc = loc;
      };
    ]
let accessor_intf ~ptype_name (ld : label_declaration) =
  let loc = ld.pld_loc in
  psig_value ~loc
    {
      pval_name = ld.pld_name;
      pval_type =
        ptyp_arrow ~loc Nolabel
          (ptyp_constr ~loc { loc; txt = lident ptype_name.txt } [])
          ld.pld_type;
      pval_attributes = [];
      pval_loc = loc;
      pval_prim = [];
    }
let generate_impl ~ctxt (_rec_flag, type_declarations) =
  let loc = Expansion_context.Deriver.derived_item_loc ctxt in
  List.map type_declarations ~f:(fun (td : type_declaration) -&gt;
      match td with
      | {
       ptype_kind = Ptype_abstract | Ptype_variant _ | Ptype_open;
       ptype_loc;
       _;
      } -&gt;
          let ext =
            Location.error_extensionf ~loc:ptype_loc
              &#34;Cannot derive accessors for non record types&#34;
          in
          [ Ast_builder.Default.pstr_extension ~loc ext [] ]
      | { ptype_kind = Ptype_record fields; _ } -&gt;
          List.map fields ~f:accessor_impl)
  |&gt; List.concat
let generate_intf ~ctxt (_rec_flag, type_declarations) =
  let loc = Expansion_context.Deriver.derived_item_loc ctxt in
  List.map type_declarations ~f:(fun (td : type_declaration) -&gt;
      match td with
      | {
       ptype_kind = Ptype_abstract | Ptype_variant _ | Ptype_open;
       ptype_loc;
       _;
      } -&gt;
          let ext =
            Location.error_extensionf ~loc:ptype_loc
              &#34;Cannot derive accessors for non record types&#34;
          in
          [ Ast_builder.Default.psig_extension ~loc ext [] ]
      | { ptype_kind = Ptype_record fields; ptype_name; _ } -&gt;
          List.map fields ~f:(accessor_intf ~ptype_name))
  |&gt; List.concat
let impl_generator = Deriving.Generator.V2.make_noarg generate_impl
let intf_generator = Deriving.Generator.V2.make_noarg generate_intf
let my_deriver =
  Deriving.add &#34;accessors&#34; ~str_type_decl:impl_generator
    ~sig_type_decl:intf_generator
```
最后一句通过`Deriving.add`将自定义的生成器告知ppxlib。代码中我们通过Deriving.Generator.V2.make_noarg定义了两个生成器，其中generate_impl为record定义生成访问fields的函数体，generate_intf为之生成访问fields函数的类型签名。
2. 编写main.ml
```ocaml
type t =
  { a : string
  ; b : int
  }
  [@@deriving accessors]
let x = { a = &#34;ssss&#34;; b = 1 }
let () = a x |&gt; print_endline; b x |&gt; string_of_int |&gt; print_endline;
```
3. 编写构建脚本dune
```dune
(library
 (name ppx_deriving_accessors)
 (kind ppx_deriver)
 (libraries ppxlib))
(executable
 (public_name simplederiver)
 (name main)
 (preprocess (pps ppx_deriving_accessors)))
```
4. 构建，执行
```bash
dune build
dune exec bin/main.exe
```

## PPX的缺点
尽管PPX预处理机制具有类型安全、能够获取更丰富的源程序信息等优势，但与基于文本的预处理机制相比，它也存在一些缺点：

- **更高的复杂度**：PPX扩展需要对OCaml的抽象语法树（AST）进行解析和转换，涉及更多的编译器内部机制。这使得PPX的学习曲线和调试难度相对较高。
- **性能开销**：尽管PPX是在编译时处理代码的，但由于其处理的是整个抽象语法树（AST），它可能比基于文本的预处理更耗费计算资源，尤其是在处理复杂的扩展时。
- **灵活性不足**：基于文本的预处理机制允许开发者自由定义宏，并进行简单的条件编译，非常适合在代码中进行灵活的字符串替换和条件操作。相比之下，PPX扩展机制更依赖于编译器的结构和规范，灵活性相对较低。虽然功能强大，但对于某些简单的预处理任务，可能会显得过于复杂。

## 总结
OCaml的PPX预处理机制提供了一种强大且类型安全的方式来增强语言的功能。它允许开发者在编译前对代码进行复杂的转换和扩展，从而显著提高代码的表达能力和开发效率。通过PPX，开发者可以引入新的语法和功能，生成类型安全的代码，并充分利用OCaml编译器提供的类型信息。这种机制不仅为OCaml的生态系统带来了极大的灵活性，也为开发者提供了强大的工具来扩展语言的边界。

然而，PPX并非没有缺点。它需要开发者对OCaml的抽象语法树（AST）和编译器的工作原理有深入的了解，这无疑增加了学习和使用成本。此外，PPX的处理过程可能会引入额外的编译开销，尤其是在处理复杂的扩展时。尽管如此，PPX的这些缺点并没有掩盖它的巨大价值。在现代OCaml开发中，PPX已经成为不可或缺的一部分，它为语言的扩展和创新提供了无限可能。

在使用OCaml进行软件开发时，开发者需要合理利用PPX机制。一方面，应根据具体的业务场景，充分利用OCaml社区提供的丰富PPX扩展库。在必要时，开发者可以自行编写PPX扩展，以提升代码的表达能力和开发效率，从而更好地实现业务目标。另一方面，开发者应避免过度依赖PPX机制，以免引入不必要的复杂性，进而降低代码的可维护性和可读性。

总之，OCaml的PPX机制是一个功能强大的工具。作为开发者，我们应充分了解其优缺点，并根据实际需求合理使用。只有这样，才能在享受PPX带来的便利的同时，避免其可能带来的问题，从而更好地提升开发效率和代码质量。

---

> 作者: [lazypanda](https://github.com/wanghuibin0)  
> URL: https://simplecoding.fun/posts/2025/02/ocaml-ppx-preprocessor/  


[[❮]](ch06-00-nightly-rust.md)
[[❯]](ch06-02-inline-assembly.md)
&nbsp;&nbsp;
[El Lenguaje de Programación Rust](_index.md) >
[6. Rust Nocturno](ch06-00-nightly-rust.md) > 6.1. Plugins del Compilador

# 6.1. Plugins del Compilador

## Introducción

`rustc` puede cargar plugins de compilador, que son bibliotecas de usuario que
extienden el comportamiento del compilador con nuevas extensiones de sintaxis,
lints, etc.

Un plugin puede ser una biblioteca dinámica con una función *registradora*
designada se encarga registrar la extension con `rustc`. Otros crates pueden
cargar estas extensiones a través del uso del atributo `#![plugin(...)]`.
Dirígete a la documentación de [`rustc_plugin`](../rustc_plugin/index.html) para
mayor detalle acerca de la mecánica de la definición y carga de un plugin.

De estar presentes, los argumentos pasados como `#![plugin(foo(... args ...))]`
no son interpretados por rustc. Son proporcionados al plugin a través del
[método `args`](../rustc_plugin/registry/struct.Registry.html#method.args) de
`Registry`.

En la gran mayoría de los casos, un plugin solo debería ser usado a través de
`#![plugin]` y no por medio de un item `extern crate`. Enlazar un plugin traería
a libsyntax y librustc como dependencias de tu crate. Esto es generalmente
indeseable a menos que estés construyendo un plugin. El lint `plugin_as_library`
chequea estos lineamientos.

La practica usual es colocar a los plugins del compilador en su propio crate,
separados de cualquier macro `macro_rules!` o código Rust ordinario que vayan a
ser usados por los consumidores de la biblioteca.

## Extensiones de sintaxis

Los plugins pueden extender la sintaxis de Rust de varias maneras. Una forma de
extension de sintaxis son las macros procedurales. Estas son invocadas de la
misma forma que las [macros ordinarias](macros.html), pero la expansión es
llevada a cabo por código Rust arbitrario que manipula
[arboles de sintaxis](../syntax/ast/index.html) en tiempo de compilación.

Escribamos un plugin
[`roman_numerals.rs`](https://github.com/rust-lang/rust/tree/master/src/test/auxiliary/roman_numerals.rs)
que implementa literales de enteros con números romanos.

```ignore
#![crate_type="dylib"]
#![feature(plugin_registrar, rustc_private)]

extern crate syntax;
extern crate rustc;
extern crate rustc_plugin;

use syntax::codemap::Span;
use syntax::parse::token;
use syntax::ast::TokenTree;
use syntax::ext::base::{ExtCtxt, MacResult, DummyResult, MacEager};
use syntax::ext::build::AstBuilder;  // trait para expr_usize
use rustc_plugin::Registry;

fn expand_rn(cx: &mut ExtCtxt, sp: Span, args: &[TokenTree])
        -> Box<MacResult + 'static> {

    static NUMERALS: &'static [(&'static str, usize)] = &[
        ("M", 1000), ("CM", 900), ("D", 500), ("CD", 400),
        ("C",  100), ("XC",  90), ("L",  50), ("XL",  40),
        ("X",   10), ("IX",   9), ("V",   5), ("IV",   4),
        ("I",    1)];

    if args.len() != 1 {
        cx.span_err(
            sp,
            &format!("argument should be a single identifier, but got {} arguments", args.len()));
        return DummyResult::any(sp);
    }

    let text = match args[0] {
        TokenTree::Token(_, token::Ident(s, _)) => s.to_string(),
        _ => {
            cx.span_err(sp, "argument should be a single identifier");
            return DummyResult::any(sp);
        }
    };

    let mut text = &*text;
    let mut total = 0;
    while !text.is_empty() {
        match NUMERALS.iter().find(|&&(rn, _)| text.starts_with(rn)) {
            Some(&(rn, val)) => {
                total += val;
                text = &text[rn.len()..];
            }
            None => {
                cx.span_err(sp, "invalid Roman numeral");
                return DummyResult::any(sp);
            }
        }
    }

    MacEager::expr(cx.expr_usize(sp, total))
}

#[plugin_registrar]
pub fn plugin_registrar(reg: &mut Registry) {
    reg.register_macro("rn", expand_rn);
}
```

Entonces podemos hacer uso de `rn!()` como cualquier otra macro:

```ignore
#![feature(plugin)]
#![plugin(roman_numerals)]

fn main() {
    assert_eq!(rn!(MMXV), 2015);
}
```

Las ventajas de esto por encima de una simple `fn(&str) -> u32` son:

* La conversión (arbitrariamente compleja) es efectuada en tiempo de
compilación.
* La validación de entrada es también llevada a cabo en tiempo de compilación.
* Puede ser extendida para su uso en patrones, lo caul efectivamente proporciona
una forma de definir una sintaxis literal nueva para cualquier tipo de dato.

En adicion a las macros procedurales, puedes definir attributos
[`derive`](../reference.html#derive) y otros tipos de expansiones. Dirigete a
[`Registry::register_syntax_extension`](../rustc_plugin/registry/struct.Registry.html#method.register_syntax_extension)
y a el
[enum `SyntaxExtension`](https://doc.rust-lang.org/syntax/ext/base/enum.SyntaxExtension.html).
Para un ejemplo mas involucrado con macros, ve a
[`regex_macros`](https://github.com/rust-lang/regex/blob/master/regex_macros/src/lib.rs).

## Tips y trucos

Algunos de los [tips de depuración de macros](macros.html#debugging-macro-code)
son aplicables.

Puedes usar [`syntax::parse`](../syntax/parse/index.html) para transformar
arboles de tokens en elementos de sintaxis de mas alto nivel, como expresiones:

```ignore
fn expandir_foo(cx: &mut ExtCtxt, sp: Span, args: &[TokenTree])
        -> Box<MacResult+'static> {

    let mut parser = cx.new_parser_from_tts(args);

    let expr: P<Expr> = parser.parse_expr();
```

Mirar a el
[código del parser `libsyntax`](https://github.com/rust-lang/rust/blob/master/src/libsyntax/parse/parser.rs)
te dara una idea de como funciona la infraestructura de parseo.

Mantén los [`Span`s](../syntax/codemap/struct.Span.html) de todo lo que parsees,
para un mejor reporte de errores. Puedes envolver
[`Spanned`](../syntax/codemap/struct.Spanned.html) alrededor de tus estructuras
de datos,

Llamar a
[`ExtCtxt::span_fatal`](../syntax/ext/base/struct.ExtCtxt.html#method.span_fatal)
abortara de manera inmediata la compilación. Es mejor llamar a
[`ExtCtxt::span_err`](../syntax/ext/base/struct.ExtCtxt.html#method.span_err)
retornando un [`DummyResult`](../syntax/ext/base/struct.DummyResult.html), de
manera que el compilador pueda continuar encontrando mas errores.

Para imprimir fragmentos de sintaxis para depuración, pudes usar
[`span_note`](../syntax/ext/base/struct.ExtCtxt.html#method.span_note) en
conjunción con
[`syntax::print::pprust::*_to_string`](https://doc.rust-lang.org/syntax/print/pprust/index.html#functions).

El ejemplo anterior produjo un literal de entero usando
[`AstBuilder::expr_usize`](../syntax/ext/build/trait.AstBuilder.html#tymethod.expr_usize).
Como alternativa a el trait `AstBuilder`, `libsyntax` proporciona un conjunto de
[macros quasiquote](../syntax/ext/quote/index.html). Estos están indocumentados
y un poco rústicos en los bordes. Sin embargo, la implementación puede ser un
buen punto de partida para un quasiquote mejorado como una biblioteca plugin
ordinaria.

## Plugins lint

Los plugins pueden extender la
[infraestructura de lints de Rust](../reference.html#lint-check-attributes) con
chequeos adicionales para estilo de código, seguridad, etc. Escribamos ahora un
plugin
[`lint_plugin_test.rs`](https://github.com/rust-lang/rust/blob/master/src/test/auxiliary/lint_plugin_test.rs)
que nos advierte acerca de cualquier item llamado `lintme`.

```ignore
#![feature(plugin_registrar)]
#![feature(box_syntax, rustc_private)]

extern crate syntax;

// Cargando rustc como un plugin para obtener las macros
#[macro_use]
extern crate rustc;
extern crate rustc_plugin;

use rustc::lint::{EarlyContext, LintContext, LintPass, EarlyLintPass,
                  EarlyLintPassObject, LintArray};
use rustc_plugin::Registry;
use syntax::ast;

declare_lint!(TEST_LINT, Warn, "Warn about items named 'lintme'");

struct Pass;

impl LintPass for Pass {
    fn get_lints(&self) -> LintArray {
        lint_array!(TEST_LINT)
    }
}

impl EarlyLintPass for Pass {
    fn check_item(&mut self, cx: &EarlyContext, it: &ast::Item) {
        if it.ident.name.as_str() == "lintme" {
            cx.span_lint(TEST_LINT, it.span, "item is named 'lintme'");
        }
    }
}

#[plugin_registrar]
pub fn plugin_registrar(reg: &mut Registry) {
    reg.register_early_lint_pass(box Pass as EarlyLintPassObject);
}
```

Entonces código como

```ignore
#![plugin(lint_plugin_test)]

fn lintme() { }
```

producirá una advertencia del compilador:

```txt
foo.rs:4:1: 4:16 warning: item is named 'lintme', #[warn(test_lint)] on by default
foo.rs:4 fn lintme() { }
         ^~~~~~~~~~~~~~~
```

Los componentes de un plugin lint son:

* una o mas invocaciones `declare_lint!`, las cuales defines structs
[`Lint`](../rustc/lint/struct.Lint.html).
* un struct manteniendo el estado necesario para el pass lint (aquí, ninguno);
* una implementación [`LintPass`](../rustc/lint/trait.LintPass.html) definiendo
como chequear cada elemento de sintaxis. Un único `LintPass` puede llamar a
`span_lint` para diferentes `Lint`s, pero debe registrarlos a todos a través del
método `get_lints`.

Los passes lint son recorridos de sintaxis, pero corren en una etapa de la
compilación en la que la información de tipos esta disponible. Los lints
[integrados de Rust](https://github.com/rust-lang/rust/blob/master/src/librustc/lint/builtin.rs)
usan mayormente la infraestructura de los plugins lint, y proveen ejemplos de
como acceder a la información de tipos.

Los lints definidos por plugins son controlados por los [flags y atributos
usuales del compilador](../reference.html#lint-check-attributes), e.j
`#[allow(test_lint)]` o `-A test-lint`. estos identificadores son derivados del
primer argumentos a `declare_lint!`, con las conversiones de capitalización y
puntuación apropiadas.

Puedes ejecutar `rustc -W help foo.rs` para ver una lista de los lints que
`rustc` conoce, incluyendo aquellos proporcionados por plugins cargados por
`foo.rs`.

[❮ 6. Rust Nocturno](ch06-00-nightly-rust.md)
&nbsp;|&nbsp;[Tabla de contenido](_index.md)&nbsp;|&nbsp;
[6.2. Ensamblador en Linea ❯](ch06-02-inline-assembly.md)

[[❮]](ch05-12-enums.md)
[[❯]](ch05-14-patterns.md)
&nbsp;&nbsp;
[El Lenguaje de Programación Rust](_index.md) >
[5. Sintaxis y Semantica](ch05-00-syntax-and-semantics.md) > 5.13. Match

# 5.13. Match

A menudo, un simple [`if`][if]/`else` no es suficiente, debido a que tienes mas
de dos opciones posibles. También, las condiciones pueden ser complejas. Rust
posee una palabra reservada, `match`, que te permite reemplazar construcciones
`if`/`else` complicadas con algo mas poderoso. Echa un vistazo:

```rust
let x = 5;

match x {
    1 => println!("uno"),
    2 => println!("dos"),
    3 => println!("trees"),
    4 => println!("cuatro"),
    5 => println!("cinco"),
    _ => println!("otra cosa"),
}
```

[if]: if.html

`match` toma una expresión y luego bifurca basado en su valor. Cada `brazo` de
la rama tiene la forma `valor => expresión`. Cuando el valor coincide, la
expresión del brazo es evaluada. Es llamado `match` por el termino ‘pattern
matching’, del cual `match` es una implementación. Hay una [sección entera
acerca de los patrones][patterns] que cubre todos los patrones posibles.

[patterns]: patterns.html

Entonces, cual es la gran ventaja? Bueno, hay unas pocas. Primero que todo,
`match` impone `chequeo de agotamiento` (‘exhaustiveness checking’).  Ves el
ultimo brazo, el que posee el guión bajo (`_`)? Si removemos ese brazo, Rust nos
proporcionara un error:

```text
error: non-exhaustive patterns: `_` not covered
```

En otras palabras, Rust esta tratando de decirnos que olvidamos un valor. Debido
a que `x` es un entero, Rust sabe que `x` puede tener un numero de valores
diferentes - por ejemplo, `6`. Sin el `_`, sin embargo, no hay brazo que pueda
coincidir, y en consecuencia Rust se rehusa a compilar el código. `_` actúa como
un ‘brazo que atrapa todo’. Si ninguno de los otros brazos coincide, el brazo
con el `_` lo hará, y puesto que tenemos dicho ‘brazo que atrapa todo’, tenemos
ahora un brazo para cada valor posible de `x`, y como resultado, nuestro
programa compilara satisfactoriamente.

`match` es también una expresión, lo que significa que lo podemos usar en el
lado derecho de un `let` o directamente en donde una expresión sea usada:

```rust
let x = 5;

let number = match x {
    1 => "uno",
    2 => "dos",
    3 => "tres",
    4 => "cuatro",
    5 => "cinco",
    _ => "otra cosa",
};
```

Algunas veces es una buena forma de convertir algo de un tipo a otro.

## Haciendo `match` en enums

Otro uso importante de la palabra reservada `match` es para procesar las
posibles variantes de una enum:

```rust
enum Mensaje {
    Salir,
    CambiarColor(i32, i32, i32),
    Mover { x: i32, y: i32 },
    Escribir(String),
}

fn salir() { /* ... */ }
fn cambiar_color(r: i32, g: i32, b: i32) { /* ... */ }
fn mover_cursor(x: i32, y: i32) { /* ... */ }

fn procesar_mensaje(msj: Mensaje) {
    match msj {
        Mensaje::Salir => salir(),
        Mensaje::CambiarColor(r, g, b) => cambiar_color(r, g, b),
        Mensaje::Mover { x: x, y: y } => mover_cursor(x, y),
        Mensaje::Escribir(s) => println!("{}", s),
    };
}
```

Otra vez, el compilador de Rust chequea agotamiento, demandando que tengas un
brazo para cada variante de la enumeración. Si dejas uno por fuera, Rust
generara un error en tiempo de compilación, a menos que uses `_`.

A diferencia de los usos previos de `match`, no puedes usar de la sentencia `if`
para hacer esto. Puedes hacer uso de una sentencia [`if let`][if-let] que puede
ser vista como una forma abreviada de `match`.

[if-let]: if-let.html

[❮ 5.12. Enumeraciones](ch05-12-enums.md)
&nbsp;|&nbsp;[Tabla de contenido](_index.md)&nbsp;|&nbsp;
[5.14. Patrones ❯](ch05-14-patterns.md)

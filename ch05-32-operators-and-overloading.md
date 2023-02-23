[Indice general](_index.md) >
[Sintaxis y Semantica](ch05-00-syntax-and-semantics.md) > Operadores y
Sobrecarga

## El Lenguaje de Programación Rust

### 5.32. Operadores y Sobrecarga

Rust permite una forma limitada de sobrecarga de operadores. Existen ciertos
operadores que pueden ser sobrecargados. Para soportar un operador particular
entre tipos, hay un trait en especifico que puedes implementar, el cual
sobrecarga el operador.

Por ejemplo, el operador `+` puede ser sobrecargado con el trait `Add`:

```rust
use std::ops::Add;

#[derive(Debug)]
struct Punto {
    x: i32,
    y: i32,
}

impl Add for Punto {
    type Output = Punto;

    fn add(self, otro: Punto) -> Punto {
        Punto { x: self.x + otro.x, y: self.y + otro.y }
    }
}

fn main() {
    let p1 = Punto { x: 1, y: 0 };
    let p2 = Punto { x: 2, y: 3 };

    let p3 = p1 + p2;

    println!("{:?}", p3);
}
```

En `main`, podemos hacer uso de `+` en nuestros dos `Puntos`s, puesto que hemos
implementado `Add<Output=Punto>` para `Punto`.

Existe un numero de operadores que pueden ser sobrecargados de esta manera, y
todos sus traits asociados viven en el modulo [`std::ops`][stdops]. Echa un
vistazo a la documentación para una lista completa.

[stdops]: ../std/ops/index.html

Implementar dichos traits sigue un patron. Analicemos a [`Add`][add] con mas
detalle:

```rust
# mod foo {
pub trait Add<RHS = Self> {
    type Output;

    fn add(self, rhs: RHS) -> Self::Output;
}
# }
```

[add]: ../std/ops/trait.Add.html

Hay en total tres tipos involucrados aqui: El tipo para el cual estas
implementado `Add`, `RHS` (de right hand side), que por defecto es `Self`, y
`Output`. Para una expresion `let z = x + y`, `x` es el tipo `Self`, `y` es el
`RHS`, y `z` es el tipo `Self::Output`.

```rust
# struct Punto;
# use std::ops::Add;
impl Add<i32> for Punto {
    type Output = f64;

    fn add(self, rhs: i32) -> f64 {
        // sumar un i32 a un Punto obteniendo un f64
# 1.0
    }
}
```

te permitira hacer lo siguiente:

```rust,ignore
let p: Point = // ...
let x: f64 = p + 2i32;
```

### Usando los traits de operador en estructuras genericas

Ahora que sabemos como están definidos los traits de operador, podemos definir
nuestro trait `TieneArea` y nuestra estructura `Cuadrado` del
[capitulo de traits][traits] de una forma mas genérica:

[traits]: traits.html

```rust
use std::ops::Mul;

trait TieneArea<T> {
    fn area(&self) -> T;
}

struct Cuadrado<T> {
    x: T,
    y: T,
    lado: T,
}

impl<T> TieneArea<T> for Cuadrado<T>
        where T: Mul<Output=T> + Copy {
    fn area(&self) -> T {
        self.lado * self.lado
    }
}

fn main() {
    let s = Cuadrado {
        x: 0.0f64,
        y: 0.0f64,
        lado: 12.0f64,
    };

    println!("Area de s: {}", s.area());
}
```

Para `TieneArea` y `Cuadrado`, solo declaramos un parametro de tipo `T` y
reemplazamos el `f64`. El bloque `impl` necesita mas modificaciones:

```ignore
impl<T> TieneArea<T> for Cuadrado<T>
        where T: Mul<Output=T> + Copy { ... }
```

El método `area` requiere que podamos multiplicar los lados, es por ello que
declaramos que `T` debe implementar `std::ops::Mul`. Como `Add`, mencionado
anteriormente, `Mul` toma un parámetro `Output`: debido a que sabemos que los
números no cambian de tipo cuando son multiplicados, también lo definimos como
`T`. `T` también debe soportar copiado, de manera que Rust no trate de mover
`self.lado` a el valor de retorno.

[❮ anterior](ch05-31-unsized-types.md)&nbsp;|&nbsp;
[Indice general](_index.md)&nbsp;|&nbsp;
[siguiente ❯](ch05-33-deref-coercions.md)

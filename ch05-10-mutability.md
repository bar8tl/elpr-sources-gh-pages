[[❮]](ch05-09-lifetimes.md)
[[❯]](ch05-11-structs.md)
&nbsp;&nbsp;
[El Lenguaje de Programación Rust](_index.md) >
[5. Sintaxis y Semantica](ch05-00-syntax-and-semantics.md) > 5.10. Mutabilidad

# 5.10. Mutabilidad

Mutabilidad, es la habilidad que una cosa posee para ser cambiada, funciona un
poco diferente en Rust que en otros lenguajes. El primer aspecto de la
mutabilidad es que no esta habilitada por defecto:

```rust,ignore
let x = 5;
x = 6; // error!
```

Podemos introducir mutabilidad con la palabra reservada `mut`:

```rust
let mut x = 5;

x = 6; // no hay problema!
```

Esto es un enlace a variable mutable. Cuando un enlace a variable es mutable,
significa que tienes permitido cambiar a lo que el enlace apunta. Entonces, en
el ejemplo anterior, no esta cambiando el valor en `x`, en cambio, el enlace
cambio de un `i32` a otro.

[vb]: variable-bindings.html

Si deseas cambiar a que apunta el enlace a variable, necesitaras una
[referencia mutable][mr]:

```rust
let mut x = 5;
let y = &mut x;
```

[mr]: references-and-borrowing.html

`y` es un enlace a variable inmutable a una referencia mutable, lo que significa
que no puedes asociar `y` a otra cosa (`y = &mut z`), pero puedes mutar lo que
sea a lo que `y` esta asociado (`*y = 5`). Una diferencia muy sutil.

Por supuesto, si necesitas ambas cosas:

```rust
let mut x = 5;
let mut y = &mut x;
```

Ahora `y` puede ser asociado a otro valor, y el valor que esta referenciando
puede ser cambiado.

Es importante notar que `mut` es parte de un [patron][pattern], de manera tal
que puedas hacer cosas como:


```rust
let (mut x, y) = (5, 6);

fn foo(mut x: i32) {
# }
```

[pattern]: patterns.html

## Mutabilidad Interior vs. Mutabilidad Exterior

Sin embargo, cuando decimos que algo es ‘immutable’ en Rust, esto no significa
que no pueda ser cambiado: lo que decimos es que algo tiene ‘mutabilidad
exterior’. Considera, por ejemplo, [`Arc<T>`][arc]:

```rust
use std::sync::Arc;

let x = Arc::new(5);
let y = x.clone();
```

[arc]: ../std/sync/struct.Arc.html

Cuando llamamos a `clone()`, el `Arc<T>` necesita actualizar el contador de
referencias. A pesar de que no hemos usado ningún `mut` aquí, `x` es un enlace
inmutable, tampoco tomamos `&mut 5` o alguna mas. Entonces, que esta pasando?

Para entender esto, debemos volver al núcleo de la filosofía que guía a Rust,
seguridad en el manejo de memoria, y el mecanismo a través del cual Rust la
garantiza, el sistema de [pertenencia][ownership], y mas específicamente, el
[préstamo][borrowing]:

> Puedes tener uno u otro de estos dos tipos de prestamo, pero no los dos al
mismo tiempo:
>
> * una o mas referencias (`&T`) a un recurso,
> * exactamente una referencia mutable (`&mut T`).

[ownership]: ownership.html
[borrowing]: references-and-borrowing.html#borrowing

Entonces, esa es la definición real de ‘inmutabilidad’: es seguro tener dos
apuntadores? En el caso de `Arc<T>`’s, si: la mutación esta completamente
contenida dentro de la estructura en si misma. No esta disponible al usuario.
Por esta razón, retorna `clone()` `&T`s. Si proporcionase `&mut T`s, seria un
problema.

Otros tipos como los del modulo [`std::cell`][stdcell], poseen lo opuesto:
mutabilidad interior. Por ejemplo:

```rust
use std::cell::RefCell;

let x = RefCell::new(42);

let y = x.borrow_mut();
```

[stdcell]: ../std/cell/index.html

RefCel proporciona referencias `&mut` a lo que contienen a traves del metodo
`borrow_mut()`. No es esto peligroso? Que tal si hacemos:

```rust,ignore
use std::cell::RefCell;

let x = RefCell::new(42);

let y = x.borrow_mut();
let z = x.borrow_mut();
# (y, z);
```

Esto, en efecto, hará pánico en tiempo de ejecución. Esto es lo que `RefCell`
hace: aplica las reglas de préstamo de Rust en tiempo de ejecución, y hace
`panic!`os si dichas reglas son violadas. Lo anterior nos permite acercarnos a
otro aspecto de las reglas de mutabilidad de Rust. Hablemos acerca de ello
primero.

## Mutabilidad a nivel de campos

La mutabilidad es una propiedad de un préstamo (`&mut`) o un enlace a variable
(`let mut`). Esto se traduce en que, por ejemplo, no puedes tener un
[`struct`][struct] con algunos campos mutables y otros inmutables:

```rust,ignore
struct Punto {
    x: i32,
    mut y: i32, // nope
}
```

La mutabilidad de un struct esta en su enlace a variable:

```rust,ignore
struct Punto {
    x: i32,
    y: i32,
}

let mut a = Punto { x: 5, y: 6 };

a.x = 10;

let b = Punto { x: 5, y: 6};

b.x = 10; // error: cannot assign to immutable field `b.x`
```

[struct]: structs.html

Sin embargo, usando [`Cell<T>`][cell], puedes emular mutabilidad a nivel de
campos:

```rust
use std::cell::Cell;

struct Punto {
    x: i32,
    y: Cell<i32>,
}

let punto = Punto { x: 5, y: Cell::new(6) };

punto.y.set(7);

println!("y: {:?}", punto.y);
```

[cell]: ../std/cell/struct.Cell.html

Esto imprimirá `y: Cell { value: 7 }`. Hemos actualizado `y` de manera
satisfactoria.

[❮ 5.9. Tiempos de Vida](ch05-09-lifetimes.md)
&nbsp;|&nbsp;[Tabla de contenido](_index.md)&nbsp;|&nbsp;
[5.11. Estructuras ❯](ch05-11-structs.md)

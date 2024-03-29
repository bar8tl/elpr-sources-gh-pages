[[❮]](ch06-08-box-syntax-and-patterns.md)
[[❯]](ch06-10-associated-constants.md)
&nbsp;&nbsp;
[El Lenguaje de Programación Rust](_index.md) >
[6. Rust Nocturno](ch06-00-nightly-rust.md) > 6.9. Patrones Slice

# 6.9. Patrones Slice

Si deseas hacer coincidencia de patrones en un slice o un arreglo, puedes usar
`&` con la facilidad `slice_patterns`:

```rust
#![feature(slice_patterns)]

fn main() {
    let v = vec!["coincide", "1"];

    match &v[..] {
        ["coincide", segundo] => println!("El segundo elemento es {}", segundo),
        _ => {},
    }
}
```

El feature gate `advanced_slice_patterns` te permite usar `..` para indicar
cualquier numero de elementos dentro de una coincidencia de patrones aplicada a
un slice. Este comodín puede ser solo usado una vez para un determinado arreglo.
De haber un identificador antes de el `..`, el resultado de el slice sera
asociado a ese nombre. Por ejemplo:

```rust
#![feature(advanced_slice_patterns, slice_patterns)]

fn es_simetrico(list: &[u32]) -> bool {
    match list {
        [] | [_] => true,
        [x, dentro.., y] if x == y => es_simetrico(dentro),
        _ => false
    }
}

fn main() {
    let sim = &[0, 1, 4, 2, 4, 1, 0];
    assert!(es_simetrico(sim));

    let asim = &[0, 1, 7, 2, 4, 1, 0];
    assert!(!es_simetrico(asim));
}
```

[❮ 6.8. Sintaxis Box y Patrones](ch06-08-box-syntax-and-patterns.md)
&nbsp;|&nbsp;[Tabla de contenido](_index.md)&nbsp;|&nbsp;
[6.10. Constantes Asociadas ❯](ch06-10-associated-constants.md)

---
layout: post
title: Compilar Rust para Linux desde macOS
---

Uno de los problemas que me he encontrado con Rust (a diferencia de lo que ocurre con Go) es la generación de un ejecutable para un OS diferente al del equipo donde se esta desarrollando.

Recopilando información de varias fuentes he logrado hacerlo así:

<!--end_excerpt-->
## Paso 1

En la carpeta del proyecto crear dir .cargo

```bash
mkdir .cargo
```

## Paso 2

Crear un fichero config en .cargo y agregarlo datos para el linker que corresponda al target elegido.

```toml
[target.x86_64-unknown-linux-musl]
linker = "x86_64-linux-musl-gcc"
```

Luego, si miramos en la estructura de nuestro proyecto, veremos que en target nos faltan los archivos del que queremos generar. Por eso, debemos agregar dicho target para permitir la compilación.

```bash
rustup target add x86_64-unknown-linux-musl
```

Para este target deberemos instalar su linker

```bash
brew install filosottile/musl-cross/musl-cross
```

Luego ya estamos listos para generar la aplicación aunque hay que pasar una variable de entorno para que el proceso pueda encontrar el linker (al menos en macOS)

```bash
CROSS_COMPILE=x86_64-linux-musl cargo build --release --target=x86_64-unknown-linux-musl
```

Cuando termine el proceso, encontraremos nuestro ejecutable en:

```bash
target
  └── x86_64-unknown-linux-musl
                               └── release
```


Y listo. 🙂
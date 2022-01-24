---
layout: post
title: Go y Docker en macOS
---

Luego de mucho tiempo me he decidido a escribir sobre Docker, Kubernetes, etc.

Dado que trabajo sobre macOS, me he encontrado con una serie de dificultades añadidas que he ido resolviendo y apuntando por diferentes lugares hasta que decidí concentrar toda la información en un solo lugar y, al mismo tiempo, hacer pública mi «investigación» ya que podría servir a alguien que este pasando por esa misma etapa.
<!--end_excerpt-->
Dado que la información en inglés es abundante, he preferido escribir esto en castellano. 🙂

La idea consiste en crear una mínima aplicación que pueda ser encapsulada en una imagen Docker y luego desplegada en la nube. Para esto, me decidí por usar Go, que no requiere de un máquina virtual (como si ocurre con Java) y por lo tanto permite generar imágenes Docker más pequeñas.

El primer paso es, pues, crear una aplicación en Golang.

## Aplicación en Go

La aplicación en Go será bastante simple ya que la idea no es aprender a programar en Go sino crear una aplicación rápidamente.

La aplicación tiene su propio servidor escuchando en el puerto 8081 (por lo que tampoco necesita un Apache u otro servidor web) y responde a las peticiones http con el mensaje «Hello world»

```go
package main

import (
        "fmt"
        "log"
        "net/http"
        "strconv"
        "sync"
)

var counter int
var mutex = &sync.Mutex{}

func echoString(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "hello")
}

func incrementCounter(w http.ResponseWriter, r *http.Request) {
        mutex.Lock()
        counter++
        fmt.Fprintf(w, strconv.Itoa(counter))
        mutex.Unlock()
}

func main() {
        http.HandleFunc("/", echoString)

        http.HandleFunc("/increment", incrementCounter)

        http.HandleFunc("/hi", func(w http.ResponseWriter, r *http.Request) {
                fmt.Fprintf(w, "Hi")
        })

        log.Println("Starting server....")

        log.Fatal(http.ListenAndServe(":8081", nil))
        log.Println("Server started at port 8081")

}
```

Una vez hecho esto, procedo a compilar ejecutando go build.
Allí me dí cuenta de que el código que estaba generando esa compilación (recordemos que Go genera código nativo) era para macOS. Sin embargo, mi plan es desplegar en Docker en una imagen linux por lo que necesito generar código unix.

## Compilar para linux
Luego de investigar por la red, logré encontrar el comando para compilar para Linux:

```bash 
env GOOS=linux GOARCH=amd64 go build main.go
6503338 Feb 22 10:55 main
```

Ahora, necesito generar la imagen Docker con mi aplicación

## Crear Dockerfile

Para crear la imagen usaré como base iron/go (https://github.com/iron-io/dockers/tree/master/go) que es básicamente una versión Alpine preparada para correr aplicaciones desarrolladas en Golang.

Para ello, crearé un fichero llamado Dockerfile con el siguiente contenido:

```yaml
FROM iron/go
WORKDIR /app
ADD main /app/.
# Launch it
ENTRYPOINT ["./main"]
```

El fichero indica cual es la imagen que usaremos como base, cuál es el directorio de trabajo dentro de la imagen Docker, copiará a mi programa main a la carpeta app e indicará cuál es la aplicación que debe ejecutarse cuando alguien ejecute la imagen Docker.

Crear imagen Docker
Teniendo ya todo listo solo resta crear la imagen Docker.

```bash
Javiers-MacBook-Air:src jleyba$ docker build -t antonof/gotest .
Sending build context to Docker daemon  6.515MB
Step 1/4 : FROM iron/go
 ---> ed7df0451f6c
Step 2/4 : WORKDIR /app
 ---> Using cache
 ---> 5cacdfc74a97
Step 3/4 : ADD main /app/.
 ---> Using cache
 ---> 112ebd6ce76a
Step 4/4 : ENTRYPOINT ["./main"]
 ---> Using cache
 ---> d3b5b2e00d85
Successfully built d3b5b2e00d85
Successfully tagged antonof/gotest:latest
```

Aparentemente, la imagen ha sido creada pero solo nos rendiremos ante la evidencia de ver correr nuestra aplicación  🙂

## Ejecutar aplicación en Docker

Para ejecutar la aplicación dentro del docker:

```bash
Javiers-MacBook-Air:src jleyba$ docker run -p 8081:8081 antonof/gotest
2019/02/22 10:09:13 Starting server....
```

Dado que parece que se ha ejecutado correctamente, vamos a probar usando un browser:


Como podemos ver, la aplicación ha respondido de acuerdo a lo esperado y solo nos resta subir nuestra imagen a un repositorio Docker.

Subir el docker al repo
Para subir nuestra imagen usaremos el comando:

```bash
docker push hub-user/repo-name:tag
```

Ya tenemos todo listo y ahora pasaremos a ver como instalar Kubernetes en el siguiente artículo.


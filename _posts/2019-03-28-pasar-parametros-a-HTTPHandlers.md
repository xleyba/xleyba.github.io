---
layout: post
title: Pasar parámetros a HTTPHandlers
---

Luego de varios experimentos creando servicios Rest, utilizando tanto el servidor http por defecto como Fasthttp, he visto que hay un par de manderas diferentes de pasar parámetros a un handler.

¿Cuándo necesitamos eso?

<!--end_excerpt-->

Supongamos que tenemos un servicio Rest que funciona tanto como servidor cómo como cliente. Por ejemplo, un servicio debe responder a las peticiones que recibe en base a hacer llamadas GET a otro servicio.

Un handler normal se definiría como algo así:

```go
func customerAccountsHandler(w http.ResponseWriter, r *http.Request) {
```

y sería llamado así:

```go
r := mux.NewRouter()
r.Path("/customer/accounts").Queries().HandlerFunc(
	options.customerAccountsHandler).Methods("GET")
```

¿Qué ocurre si quiero parametrizar la url del servicio Rest al que debe consultar? ¿Qué ocurre si quiero pasarle la conexión cliente para poder reutilizarla (una buena práctica en Go)? Pues no puedo. No al menos así con ese formato estándar.

Una forma, como he mostrado en un artículo anterior, consiste en usar closures.
En la idea es crear una función que haga de wrapper y me permita pasar los parámetros sin dejar de ser un handler. Por ejemplo:

```go
func customerAccountsHandler(client *http.Client, backendURL string) func(w http.ResponseWriter, r *http.Request) {
	return func(w http.ResponseWriter, r *http.Request) {
```

Como vemos en el código, mi función acepta como parámetros el cliente y la URL y aún así sigue devolviendo un handler.

La llamada en ese caso quedaría así:

```go
r := mux.NewRouter()
	r.Path("/customer/accounts").Queries().HandlerFunc(
		customerAccountsHandler(client, "http://localhost:9090/accounts")).Methods("GET")
```

Este método suele ser útil, pero también puede dar problemas, por ejemplo, al momento de crear los casos de prueba.

En ese caso, otra forma muy útil y práctica es usar interfaces. Por ejemplo, creo una estructura así:

```go
type HandlerOptions struct {
	Client     *http.Client
	BackendURL string
}
```

Luego puedo entonces crear una variable con esa estructura y los datos que necesitaré pasar como parámetros:

```go
options := HandlerOptions{
	Client:     client,
	BackendURL: "http://localhost:9596",
}
```

Puedo entonces pedir que mi handler original cumpla con esa interface:

```go
func (o HandlerOptions) customerAccountsHandler(w http.ResponseWriter, r *http.Request) 
```

Y ahora puedo ya pasar los parámetros en la llamada del handler:

```go
r := mux.NewRouter()
r.Path("/customer/accounts").Queries().HandlerFunc(
	options.customerAccountsHandler).Methods("GET")
```

De esta forma, el código queda más claro y trae menos problemas al momento de realizar tests.


---
layout: post
title: Go y las conexiones HTTP
---

Un tema curioso y que me ha traído problemas es el de crear un cliente http para pruebas de carga.

La idea consistía en tener un servicio Rest como frontend que básicamente llamara a otro servicio backend para luego retornar las respuestas al llamador.

El problema se da porque el cliente de http de Go solo tiene por defecto 100 conexiones concurrentes, pero de ese pool, hay solo dos conexiones por host. Dado que mis pruebas se realizaban con SoapUI desde mi máquina, pues solo contaba con un host por ende dos conexiones concurrentes.

<!--end_excerpt-->

Sin embargo, las pruebas las estaba haciendo tanto con SoapUI como con Apache ab o wrk, configurados con 500 conexiones concurrentes y 50000 peticiones.

Esta claro que no podría funcionar bien…. y no lo hacía. 🙂

¿Cómo se arregla eso?

Pues hay que configurar un nuevo transport y modificar allí los valores por defecto. Por ejemplo, lo que he hecho es definir una función que hace toda la configuración y devuelve un puntero a un cliente http, para compartirlo entre todos los handlers (práctica recomendada):

```go
func SetClient() *http.Client {

	defaultRoundTripper := http.DefaultTransport
	defaultTransportPointer, ok := defaultRoundTripper.(*http.Transport)
	if !ok {
		panic(fmt.Sprintf("defaultRoundTripper not an *http.Transport"))
	}
	defaultTransport := *defaultTransportPointer // dereference it to get a copy of the struct that the pointer points to
	defaultTransport.MaxIdleConns = viper.GetInt("MaxIdleConns")
	defaultTransport.MaxIdleConnsPerHost = viper.GetInt("MaxIdleConnsPerHost")

	myClient := &http.Client{Transport: &defaultTransport}

	return myClient

}
```

Ese cliente y la url del servicio al que el cliente debe llamar, hay que pasárselos a los handlers y allí también hay truco.
Para resolver ese problema usamos Closures que son funciones anónimas dentro de funciones, pero que pueden usar valores que vengan como parámetro en las funciones que las contienen.

Por ejemplo, la definición normal de un handler es así:

```go
func myHandler(w http.ResponseWriter, r *http.Request)
```
¡Allí no hay forma de pasar un parámetro!
Pero usando closures, podemos hacer:

```go
func helloHandler(client *http.Client, backendURL string) func(w http.ResponseWriter, r *http.Request) {
	return func(w http.ResponseWriter, r *http.Request) {
```

y de esa forma estamos pudiendo pasar parámetros al handler y evitamos definir variables globales (que de hecho no existen en Go).

La llamada al handler aparecería entonces así:

```go
r := mux.NewRouter()
r.Path("/hello").HandlerFunc(helloHandler(SetClient(), viper.GetString(
	"backendURL"))).Methods("GET")
	

srv := &http.Server{
	Addr:        viper.GetString("port"),
	Handler:     r,
	ReadTimeout: 10 * time.Second,
	//WriteTimeout:   10 * time.Second,
	//MaxHeaderBytes: 1 << 20,
}

// Lanuch server in a thread
go func() {
	fmt.Println("Starting server...")
	log.Info().Msg("Starting server...")
	if err := srv.ListenAndServe(); err != nil {
		log.Panic().Msgf("%s", err)
	}
}()

```

(*) cabe destacar que como se ve arriba estoy usando el paquete viper para configuración de mi aplicación.

Con esto ya podemos entonces usar el cliente definido para llamar al servicio backend sin tener problemas de falta de sockets.

```go
func helloHandler(client *http.Client, backendURL string) func(w http.ResponseWriter, r *http.Request) {
	return func(w http.ResponseWriter, r *http.Request) {

	url := backendURL + "/hello"
	log.Debug().Msgf("URL to call: %s", url)

	response, errResp := client.Get(url)
	if errResp != nil {
		log.Error().Msgf("Error en response: %s", errResp.Error())
		http.Error(w, "Error en response: %s", http.StatusInternalServerError)
		return
	}

	defer response.Body.Close()

	respBody, errData := ioutil.ReadAll(response.Body)
	if errData != nil {
		log.Error().Msgf("failed to read response body %s", errData.Error())
		http.Error(w, "failed to read response body %s", http.StatusInternalServerError)
		return
	}

	log.Debug().Msgf("Received: %v", respBody)

	w.WriteHeader(http.StatusOK)
	w.Write(respBody)
}
}

```

Dentro de ese código hay también dos temas importantes que pueden traer problemas.

Uno es cerrar el Body de la response:

```go
defer response.Body.Close()
```

De lo contrario las conexiones quedan abiertas y comenzaremos a quedarnos sin recursos y con conexiones en estado WAIT.

Otro tema es leer la respuesta, incluso sino recibiéramos datos

```go
respBody, errData := ioutil.ReadAll(response.Body)
if errData != nil {
	log.Error().Msgf("failed to read response body %s", errData.Error())
	http.Error(w, "failed to read response body %s", http.StatusInternalServerError)
	return
}
```

Con esto he resuelto todos los problemas de conectividad y performance que tenía.

La idea de esas soluciones las he tomado de este blog:
[Tuning the Go HTTP Client Settings for Load Testing](http://tleyden.github.io/blog/2016/11/21/tuning-the-go-http-client-library-for-load-testing/)


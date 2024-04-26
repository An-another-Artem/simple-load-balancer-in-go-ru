<h1>Простой балансировщик нагрузки в Go</h1>

**Английская версия с исходным кодом:** [здесь](https://github.com/An-another-Artem/simple-load-balancer-in-go)

**Что такое балансировщик нагрузки?**

Балансировщик нагрузки - это программа, которая распределяет запросы клиентов между несколькими серверами, чтобы предотвратить перегрузку любого из них. Это помогает обеспечить высокую доступность и производительность вашего приложения.

**Алгоритмы балансировки нагрузки**

Существует несколько алгоритмов балансировки нагрузки, каждый из которых имеет свои преимущества и недостатки. Вот три распространенных:

* **Round Robin (включая Weighted Round Robin и Sticky Round Robin)**: Этот алгоритм равномерно распределяет запросы между всеми доступными серверами. Weighted Round Robin присваивает больший вес серверам с более высокой производительностью, а Sticky Round Robin направляет запросы от одного и того же клиента на один и тот же сервер для обеспечения согласованности сеансов.
* **Least Connections (реализовано)**: Этот алгоритм отслеживает количество активных соединений на каждом сервере и направляет новые запросы на сервер с наименьшим количеством соединений, стремясь к равномерному распределению нагрузки.
* **Least Time**: Этот алгоритм измеряет время отклика каждого сервера и направляет запросы на сервер с наименьшим временем отклика. Он требует метода периодической проверки времени отклика сервера, что может добавить сложности.

**Алгоритм Round Robin**

<img src="https://www.jscape.com/hubfs/images/round_robin_algorithm-1.png">

В Round Robin запросы циклически проходят через все доступные серверы, независимо от их текущей загрузки.

**Алгоритм Least Connections (реализовано)**

<img src="https://www.codereliant.io/content/images/2023/06/d1-1-1.png">

Этот алгоритм отслеживает количество **активных соединений** на каждом сервере и направляет новые запросы на сервер с **наименьшим количеством соединений**, стремясь к **равномерному распределению нагрузки**.

**Алгоритм Least Time**

Алгоритм Least Time измеряет время отклика каждого сервера и направляет запросы на сервер с наименьшим временем отклика. Он требует метода периодической проверки времени отклика сервера, что может добавить сложности.

**Примечание:**

Я не смог найти подходящего изображения для алгоритма Least Time. Вы можете найти изображение самостоятельно.

## Что дальше?

**Что нам нужно знать:**

* Балансировщик нагрузки использует пакет proxy из `httputil` для отправки клиентских запросов на серверы.
* В настоящее время он не использует конфигурационные файлы или другие механизмы для добавления или изменения серверов. Вы можете добавить эту функцию в качестве улучшения.

**Теперь мы готовы создать балансировщик нагрузки!**

Балансировщик нагрузки реализует алгоритм "Least connections". Как упоминалось ранее, он направляет клиентские запросы на сервер, который имеет наименьшее количество соединений.

```Go
package main

import (
	"fmt"
	"log"
	"net/http"
	"net/http/httputil"
	"net/url"
	"sync"
)

var (
	BestServer       *Server
	LeastConnections uint = 0
	FirstURL, _           = url.Parse("http://127.0.0.1:8081")
	SecondURL, _          = url.Parse("http://127.0.0.1:8082")
)

type Server struct {
	URL         url.URL
	Connections uint
	Alive       bool
	Mutex       sync.Mutex
}
type Servers []*Server

func (servers Servers) ChooseServer() (*Server, error) {
	for _, server := range servers {
		server.Mutex.Lock()
		if resp, err := http.Get(server.URL.String()); err != nil || resp.StatusCode >= 500 {
			server.Alive = false
			return BestServer, err
		} else {
			server.Alive = true
		}
		if (server.Connections < LeastConnections || LeastConnections == 0) && server.Alive {
			LeastConnections = server.Connections
			BestServer = server
		}
		if BestServer != nil {
			server.Connections++
		}
		server.Mutex.Unlock()
	}
	return BestServer, nil
}
func HandleRequest(w http.ResponseWriter, r *http.Request, servers Servers) {
	server, _ := servers.ChooseServer()
	if server == nil {
		fmt.Println("Currently there is no available servers there.")
		return
	}
	defer func() {
		server.Mutex.Lock()
		server.Connections--
		server.Mutex.Unlock()
	}()
	proxy := httputil.NewSingleHostReverseProxy(&server.URL)
	proxy.ServeHTTP(w, r)
}
func main() {
	servers := Servers{{URL: *FirstURL}, {URL: *SecondURL}}
	http.HandleFunc("/", func(writer http.ResponseWriter, request *http.Request) {
		HandleRequest(writer, request, servers)
	})
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```
<h2>Если тебе понравилось то поставь звездочку на этот репозиторий и на его английскую версию!</h2>

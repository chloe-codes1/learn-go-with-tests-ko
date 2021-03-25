# JSON, 라우팅 & 임베딩

**[이 장의 모든 코드는 여기에서 찾을 수 있다](https://github.com/quii/learn-go-with-tests/tree/main/json)**

[이전 장에서](http-server.md) 우리는 사용자가 자신이 얼마나 많은 게임에서 이겼는지 확인할 수 있는 웹서버를 만들었다.

우리의 프로덕트 오너에게 새로운 요구 사항이 생겼다; 저장되어 있는 모든 player들을 리턴하는 엔드 포인트 `/league`를 만드는 것이다. 그녀는 JSON 형식으로 값을 리턴 받는 것을 요구했다.

## Here is the code we have so far

```go
// server.go
package main

import (
	"fmt"
	"net/http"
)

type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
}

type PlayerServer struct {
	store PlayerStore
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := r.URL.Path[len("/players/"):]

	switch r.Method {
	case http.MethodPost:
		p.processWin(w, player)
	case http.MethodGet:
		p.showScore(w, player)
	}
}

func (p *PlayerServer) showScore(w http.ResponseWriter, player string) {
	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}

func (p *PlayerServer) processWin(w http.ResponseWriter, player string) {
	p.store.RecordWin(player)
	w.WriteHeader(http.StatusAccepted)
}
```

```go
// InMemoryPlayerStore.go
package main

func NewInMemoryPlayerStore() *InMemoryPlayerStore {
	return &InMemoryPlayerStore{map[string]int{}}
}

type InMemoryPlayerStore struct {
	store map[string]int
}

func (i *InMemoryPlayerStore) RecordWin(name string) {
	i.store[name]++
}

func (i *InMemoryPlayerStore) GetPlayerScore(name string) int {
	return i.store[name]
}

```

```go
// main.go
package main

import (
	"log"
	"net/http"
)

func main() {
	server := &PlayerServer{NewInMemoryPlayerStore()}

	if err := http.ListenAndServe(":5000", server); err != nil {
		log.Fatalf("could not listen on port 5000 %v", err)
	}
}
```

관련된 테스트들은 해당 챕터 상단의 링크에서 확인할 수 있다.

우리는 이제 league 테이블 엔드 포인트를 만들 것이다.

## Write the test first

우리는 유용한 테스트 함수와 가짜 `PlayerStore`를 활용하기 위해 기존에 만들어 둔 구조체를 확장할 것이다.

```go
//server_test.go
func TestLeague(t *testing.T) {
	store := StubPlayerStore{}
	server := &PlayerServer{&store}

	t.Run("it returns 200 on /league", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

실제 점수와 JSON에 대해 고민하기 전에, 우리는 작은 변화들을 되풀이하여 목표를 향해 나아갈 것이다. 가장 간단하게 확인해볼 것은 `/league`를 호출하여 `OK`를 받아오는 것이다.

## Try to run the test

```
=== RUN   TestLeague/it_returns_200_on_/league
panic: runtime error: slice bounds out of range [recovered]
    panic: runtime error: slice bounds out of range

goroutine 6 [running]:
testing.tRunner.func1(0xc42010c3c0)
    /usr/local/Cellar/go/1.10/libexec/src/testing/testing.go:742 +0x29d
panic(0x1274d60, 0x1438240)
    /usr/local/Cellar/go/1.10/libexec/src/runtime/panic.go:505 +0x229
github.com/quii/learn-go-with-tests/json-and-io/v2.(*PlayerServer).ServeHTTP(0xc420048d30, 0x12fc1c0, 0xc420010940, 0xc420116000)
    /Users/quii/go/src/github.com/quii/learn-go-with-tests/json-and-io/v2/server.go:20 +0xec
```

당신의 `PlayerServer`는 위와 같이 패닉이 일어날 것이다. Stack trace의 해당 코드 라인을 확인해보면 `server.go`를 가리키는 것을 확인할 수 있다.

```go
player := r.URL.Path[len("/players/"):]
```
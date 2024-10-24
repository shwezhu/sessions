## sessions-go

An in-memory concurrent-safe package for go web session management. 

This session management library primarily utilizes an in-memory store to ensure optimal performance and rapid access. While a Redis store is also supported as an alternative, the implementation of mutex locks for each session operation can lead to unnecessary resource consumption when using Redis. 

```shell
$ go get -u github.com/shwezhu/sessions
```

## feature

Get expired sessions after removed from session store. The codes below will enable Store send expired session into `ExpiredSession` channel which is a field of struct `Store`:

```go
store := NewStore(WithExpiredGc())
```

e.g.,
```go
store := NewStore(WithExpiredGc())
session, _ := store.Get(r, "session-id")
session.InsertValue("name", "Coco")
session.Save(rsp)
...
// If you telled store you want get expired session,
// you should use another goroutine to keep listen the channel,
go func() {
	for {
		select {
		case sessions := <-store.ExpiredSession:
			for _, se := range sessions {
				b.Log(se.values)
			}
		case errSe := <-store.ExpiredSessionErr:
			b.Error(errSe)
		}
	}
}()
```

## usage

```go
func doNothing(_ http.ResponseWriter, _ *http.Request) {}

func handler(w http.ResponseWriter, r *http.Request) {
	// Get will return a cached session if exists, if not return a new one.
	// For simplicity, ignore error here
	session, _ := store.Get(r, "session-id")
	// If it's a new session, set some info into it.
	if session.IsNew() {
		session.InsertValue("name", "Coco")
		session.InsertValue("age", 18)
		// Set max age for session in seconds.
		session.SetMaxAge(30)
		// Save session into response to client.
		// You should save session after you make change on a session.
		session.Save(w)
		return
	}
	// If not a new session.
	name, _ := session.GetValueByKey("name")
	_, _ = fmt.Fprint(w, fmt.Sprintf("hello %v\n", name))
}

// You just need only one store instance on global.
var store *sessions.MemoryStore

func main() {
	// You need specify the id length of a session, don't make it too big.
	store := NewStore(WithDefaultGc())
	http.HandleFunc("/", handler)
	http.HandleFunc("/favicon.ico", doNothing)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

```shell
$ curl localhost:8080/ -v 
*   Trying 127.0.0.1:8080...
...
< HTTP/1.1 200 OK
< Set-Cookie: session-id=F7j4Ftn5jgdKYyAxfVdR6lc=E=IdJKdx; Path=/; Expires=Tue, 05 Sep 2023 14:36:53 GMT; Max-Age=30
...

# Send get request with cookie
$ curl localhost:8080/ --cookie "session-id=F7j4Ftn5jgdKYyAxfVdR6lc=E=IdJKdx"
hello Coco
```
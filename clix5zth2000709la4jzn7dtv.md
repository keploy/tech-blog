---
title: "Go Mocks and Stubs Made Easy"
datePublished: Thu Jun 15 2023 13:16:21 GMT+0000 (Coordinated Universal Time)
cuid: clix5zth2000709la4jzn7dtv
slug: go-mocks-and-stubs-made-easy
canonical: https://keploy.io/blog/technology/go-mocks-and-stubs-made-easy
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/fd9mIBluHkA/upload/efdc31650d0ba0f4ce5da5965e2dbd05.jpeg
tags: tdd, unit-testing, stub, gomock, test-generator

---

Testing network stuff like APIs and database calls can be a real pain:

* I find myself burning way too much time just making mock data, instead of *actually* doing the tests or assertions.
    
* When you make fake mocks, you might end up using wrong guesses or data that's just too unreal or vague.
    
* When things (contract) change, you have to dig around and update everything by hand. It's a bit of a headache.
    

I was searching for a more efficient way to imitate HTTP dependencies. I found a great library on Google's GitHub page. The URL is [github.com/google/go-replayers](http://github.com/google/go-replayers).

It piqued my interest because it let me record my HTTP dependencies, and hey, it's working out for Google, right? Still, it wasn't all smooth sailing - a couple of issues popped up:

* Reading and editing the recorded stubs was difficult. Scrubbing sensitive information such as personal details and keys was especially challenging, especially when recording from live production sources.
    
* Keeping these stubs fresh was a DIY job, with API developers rarely ever giving the API mocks a second glance. So, there was always a risk of slipping into wrong assumptions.
    

> Wouldn't it be awesome to make believable, easy-to-understand mock-ups or stubs that can double as API test cases?

So, we got down to coding and built our own mock/stub library in Keploy to help with TDD workflows. This tool has a unique capability. It can create tests and mocks from real API or database calls. In contrast, [gomock](https://github.com/golang/mock) only creates types.

The best part? You can use API call tests as mocks or stubs, and vice versa!

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1665064188591/o2HOvKXcL.png align="left")

Let's roll up our sleeves and dive into a unit-testing example I found on a pretty nifty [**blog**](https://clavinjune.dev/en/blogs/mocking-http-call-in-golang-a-better-way/). We'll be swapping out the handcrafted mocking behaviour with lifelike recorded stubs 🌟. First, let's get our directory structure in shape:

```plaintext
$ go mod init mocking
go: creating new go.mod: module mocking
$ mkdir -p external
$ touch external/{external.go,external_test.go}
$ tree .
.
├── external
│   ├── external.go
│   └── external_test.go
└── go.mod

1 directory, 3 files
```

You can swipe the starting code from these Github gists: [**example.go**](https://hashnode.com/draft/633ecda757b647c4f72b091a) & [**example\_test.go**](https://gist.github.com/slayerjain/1c6e82bf5c3cee4760b57155d886be60)

## **Let's whip up our stubs**

First things first, we need to download and fire up [**keploy**](https://github.com/keploy/keploy).

**Mac**

```bash
curl --silent --location "https://github.com/keploy/keploy/releases/latest/download/keploy_darwin_all.tar.gz" | tar xz -C /tmp

sudo mv /tmp/keploy /usr/local/bin && keploy
```

**Linux**

```bash
curl --silent --location "https://github.com/keploy/keploy/releases/latest/download/keploy_linux_amd64.tar.gz" | tar xz -C /tmp

sudo mv /tmp/keploy /usr/local/bin && keploy
```

That should download and kick-start the keploy server. You should see something like this:

```bash
 ➜  mocking curl --silent --location "https://github.com/keploy/keploy/releases/latest/download/keploy_darwin_all.tar.gz" | tar xz -C /tmp

sudo mkdir -p /usr/local/bin && sudo mv /tmp/keploy /usr/local/bin && keploy
Password:

       ▓██▓▄
    ▓▓▓▓██▓█▓▄
     ████████▓▒
          ▀▓▓███▄      ▄▄   ▄               ▌
         ▄▌▌▓▓████▄    ██ ▓█▀  ▄▌▀▄  ▓▓▌▄   ▓█  ▄▌▓▓▌▄ ▌▌   ▓
       ▓█████████▌▓▓   ██▓█▄  ▓█▄▓▓ ▐█▌  ██ ▓█  █▌  ██  █▌ █▓
      ▓▓▓▓▀▀▀▀▓▓▓▓▓▓▌  ██  █▓  ▓▌▄▄ ▐█▓▄▓█▀ █▓█ ▀█▄▄█▀   █▓█
       ▓▌                           ▐█▌                   █▌
        ▓

keploy 0.9.1

.2023-06-15T15:30:58.736+0530	INFO	server/server.go:217	keploy started at port 6789
```

Now that the Keploy server is humming along, we can bring in the Keploy Go SDK and start building our stubs. Keploy usually requires all network clients (like HTTP clients or DB drivers) to be wrapped up so it can catch them. So here, we'll wrap the HTTP client with Keploy's `net/http` wrapper.

```go
	import "github.com/keploy/go-sdk/integrations/khttpclient"

...
    // wrap the http client with the Keploy SDK
	interceptor := khttpclient.NewInterceptor(http.DefaultTransport)
	client := &http.Client{
		Transport: interceptor,
	}

	ext = external.New(server.URL, client, time.Second)
...
```

We have finished wrapping the HTTP client. Now, we can start the Keploy SDK. We can use it to record or stub our HTTP calls.

```go
	ctx := mock.NewContext(mock.Config{
		Name: "hello",            // It is unique for every mock/stub. If you dont provide during record it would be generated. Its compulsory during tests.
		Mode: keploy.MODE_RECORD, // It can be MODE_TEST or MODE_OFF. Default is MODE_TEST
	})

	for i := range tt {
		tc := tt[i]

		t.Run(tc.name, func(t *testing.T) {
			t.Parallel()
            // Ensure to pass the context all the way to the http client
			gotData, gotErr := ext.FetchData(ctx, tc.id)

			if !errors.Is(gotErr, tc.wantErr) {
				fatal(t, tc.wantErr, gotErr)
			}

			if !reflect.DeepEqual(gotData, tc.wantData) {
				fatal(t, tc.wantData, gotData)
			}
		})
	}
```

"Like most profiling tools in Golang, Keploy also needs you to pass the `context.Context` objects to all the dependencies. The SDK uses this context to keep all the different parts of your app in sync. Ready to record our stubs with `go test`?

```bash
mocking go test -v -count=1 ./...
mocking server
mocking external
run tests
=== RUN   TestExternal_FetchData

💡⚡️ Keploy created new mocking context in record mode  for hello.
 If you dont see any logs about your dependencies below, your dependency/s are NOT wrapped.
=== RUN   TestExternal_FetchData/response_not_ok
=== PAUSE TestExternal_FetchData/response_not_ok
=== RUN   TestExternal_FetchData/data_found
=== PAUSE TestExternal_FetchData/data_found
=== CONT  TestExternal_FetchData/response_not_ok
=== CONT  TestExternal_FetchData/data_found
🟠 Captured the mocked outputs for Http dependency call with meta:  map[name:Http operation:GET type:HTTP_CLIENT]
🟠 Captured the mocked outputs for Http dependency call with meta:  map[name:Http operation:GET type:HTTP_CLIENT]
--- PASS: TestExternal_FetchData (0.00s)
    --- PASS: TestExternal_FetchData/data_found (0.00s)
    --- PASS: TestExternal_FetchData/response_not_ok (0.00s)
PASS
ok      mocking/external        0.165s
```

Voila! Our stubs are ready to roll. By default, they're created in the `mocks` folder, but you can easily customize this. After these stubs are created, we can switch the mode in the Keploy object defined in our test file to test.

```bash
ctx := mock.NewContext(mock.Config{
		Name: "hello",          // It is unique for every mock/stub. If you dont provide during record it would be generated. Its compulsory during tests.
		Mode: keploy.MODE_TEST, // It can be MODE_TEST or MODE_OFF. Default is MODE_TEST
	})
```

With that out of the way, we can rerun our tests:

```bash
➜  mocking go test -v -count=1 ./...
mocking server
mocking external
run tests
=== RUN   TestExternal_FetchData

💡⚡️ Keploy created new mocking context in test mode  for hello.
 If you dont see any logs about your dependencies below, your dependency/s are NOT wrapped.
=== RUN   TestExternal_FetchData/response_not_ok
=== PAUSE TestExternal_FetchData/response_not_ok
=== RUN   TestExternal_FetchData/data_found
=== PAUSE TestExternal_FetchData/data_found
=== CONT  TestExternal_FetchData/response_not_ok
=== CONT  TestExternal_FetchData/data_found
🤡 Returned the mocked outputs for Http dependency call with meta:  map[name:Http operation:GET type:HTTP_CLIENT]
🤡 Returned the mocked outputs for Http dependency call with meta:  map[name:Http operation:GET type:HTTP_CLIENT]
--- PASS: TestExternal_FetchData (0.00s)
    --- PASS: TestExternal_FetchData/response_not_ok (0.00s)
    --- PASS: TestExternal_FetchData/data_found (0.00s)
PASS
ok      mocking/external        0.247s
```

Magic, isn't it? Keploy automatically serves up the previously recorded stub responses! 🪄 You can find the complete code for this blog [**here**](https://github.com/keploy/go-mocking-blog).

You can make realistic stubs (service virtualization) for any dependency supported by Keploy. This includes Postgres, MySQL, gRPC client/server, and more. All you need to do is follow the same steps. Don't hesitate to give it a shot and share your experience on the Keploy [**slack channel**](https://join.slack.com/t/keploy/shared_invite/zt-12rfbvc01-o54cOG0X1G6eVJTuI_orSA).

We're also excited about our upcoming version of Keploy (possibly TestGPT?). It will use the magic of Generative AI to generate test code that actually works! No more dealing with those half-baked, semi-working tests produced by most GPT-based test generator tools.
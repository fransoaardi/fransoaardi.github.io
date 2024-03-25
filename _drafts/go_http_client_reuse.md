golang 의 오래된 convention 이면서 논란(?)인 코드가 있습니다.
connection 을 재사용하기 위해서 resp.Body.Close() 를 해주는 코드입니다.
재사용이 되지 않으면 다시 TCP connection 을 맺을거고,
매번 Header 에 Close 혹은 Client 의 Transport 쪽에 KeepAlive 를 disable 하는 것과 같아서 효율이 떨어지는게 당연합니다.
성능을 빠르게 하는 튜닝중 하나인 셈입니다.

defer resp.Body.Close()
As described in godoc, by default the http.Client would reuse the TCP connection, but only if you read all the data from the http.Response Body and close the http.Response Body as well:
If the returned error is nil, the Response will contain a non-nil Body which the user is expected to close. If the Body is not both read to EOF and closed, the Client’s underlying RoundTripper (typically Transport) may not be able to re-use a persistent TCP connection to the server for a subsequent “keep-alive” request.

https://medium.com/@p408865/til-go-http-client-reuse-connection-5d8c48731dec

go-github 코드에서 제시됐고, 결국 net/http 개발자인 bradfitz 를 소환하게 만든 전설(?)의 이슈입니다.

https://github.com/google/go-github/pull/317
resp.Body.Close() 를 해주는것만으로는 충분하지 않았고, drain 까지 했을때 비약적인 성능 향상이 있었다고 하는 글입니다.

body 가 다 읽히지 않았다면, http.Client{} 가 가지고 있는 RoundTripper 가 가지고있는 Transport 가 재사용 될 수 없다고 해서 아래와 같은 극단적인 선택을 하기도 합니다.

defer func() {
		// Drain and close the body to let the Transport reuse the connection
		io.Copy(ioutil.Discard, resp.Body)
}()
저는 늘 위와 같이 resp.Body 를 명시적으로 관리해주는 코드를 적용해왔는데, 가끔 body 를 func 넘나들며 전달하는 경우에는.. 적용 시점이나 매번 구현하는게 상당히 귀찮았는데, 관련해서 우연히 아래와 같이 client 에 RoundTripper 를 구현 + resp.Body ( io.ReadCloser interface ) 의 Close 를 io.EOF 를 만날때까지 컨슘해서 닫아서 재사용 될 수 있게 하는 똑똑한 코드를 발견했습니다.

https://github.com/Netflix/spectator-go/pull/47

```golang
package client

import (
	"io"
	"net"
	"net/http"
	"time"

	"go.uber.org/atomic"
)

// NewHTTP 는 http response body 를 io.Discard 를 이용해서 끝까지 consume 하지 않으면 RoundTripper 가 reuse 안되는 이슈가 있어서,
// body.Close 할때 자연스럽게 consume 하도록 구현해놓은 client 이다.
// &http.Client{} 생성을 대신해서 사용한다.
// 아래의 링크에서 가져왔다.
// https://github.com/Netflix/spectator-go/pull/47
func NewHTTP() *http.Client {
	// http.Client 선언시 nil Transport 를 사용하면 net/http/transport.go 에 정의된 DefaultTransport 가 사용된다.
	// 아래의 setting 은  의 DefaultTransport 를 기준으로 생성 하였다.
	// 단, maxIdleConns(default: 100), maxIdleConnsPerHost(default: 2),
	// IdleConnTimeout(default: 90 seconds), TLSHandshakeTimeout(default: 10 seconds) 값이 다르다
	return &http.Client{
		Transport: &keepAliveTransport{wrapped: &http.Transport{
			Proxy: http.ProxyFromEnvironment,
			DialContext: (&net.Dialer{
				Timeout:   5 * time.Second,
				KeepAlive: 30 * time.Second,
			}).DialContext,
			ForceAttemptHTTP2:     true,
			MaxIdleConns:          50,
			MaxIdleConnsPerHost:   50,
			IdleConnTimeout:       30 * time.Second,
			TLSHandshakeTimeout:   2 * time.Second,
			ExpectContinueTimeout: 1 * time.Second,
		}},
	}
}

type keepAliveTransport struct {
	wrapped http.RoundTripper
}

func (k *keepAliveTransport) RoundTrip(r *http.Request) (*http.Response, error) {
	resp, err := k.wrapped.RoundTrip(r)
	if err != nil {
		return resp, err
	}
	resp.Body = &drainingReadCloser{rdr: resp.Body}
	return resp, nil
}

type drainingReadCloser struct {
	rdr     io.ReadCloser
	seenEOF atomic.Bool
}

func (d *drainingReadCloser) Read(p []byte) (n int, err error) {
	n, err = d.rdr.Read(p)
	if err == io.EOF || n == 0 {
		d.seenEOF.Store(true)
	}
	return
}

func (d *drainingReadCloser) Close() error {
	// drain buffer
	if !d.seenEOF.Load() {
		//#nosec
		_, _ = io.Copy(io.Discard, d.rdr)
	}
	return d.rdr.Close()
}

```
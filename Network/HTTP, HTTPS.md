---

---
---

# HTTP

월드 와이드 웹(WWW)의 기반이 되는 데이터 통신 규칙으로, 클라이언트(브라우저)와 서버가 HTML, 이미지 등 다양한 자원을 주고받기 위해 사용하는 요청 응답 프로토콜

## HTTP 요청

HTTP 요청은 웹 브라우저와 같은 인터넷 통신 플랫폼에서 웹 사이트를 로드하는 데 필요한 정보를 요청하는 방법
인터넷을 통해 이루어진 각 HTTP 요청은 서로 다른 유형의 정보를 전달하는 일련의 인코딩된 데이터를 전달

일반적인 HTTP 요청에는 다음이 포함됨
- HTTP 버전 유형
- URL
- HTTP 메서드
- HTTP 요청 헤더
- 선택 사항인 HTTP 본문

### HTTP 메서드

HTTP 동사라고도 불리는 HTTP 메서드는 HTTP 요청이 쿼리된 서버에서 기대하는 작업을 나타냄

예를 들어, 가장 일반적인 두 가지 HTTP 메서드는 'GET'과 'POST'
'GET' 요청은 응답으로 정보를 기대하는 반면(일반적으로 웹 사이트 형식으로) 
'POST' 요청은 일반적으로 클라이언트가 웹 서버에 정보를 제출하고 있음을 나타냄(양식 정보 등. 예: 제출된 사용자 이름 및 비밀번호)

### HTTP 요청 헤더

HTTP 헤더에는 키값 쌍에 저장된 텍스트 정보가 포함되어 있으며 헤더는 모든 HTTP 요청(및 응답, 추후 자세히 설명 예정)에 포함
이러한 헤더는 클라이언트가 사용하는 브라우저 및 요청되는 데이터와 같은 핵심 정보를 전달

![[Pasted image 20260218154207.png]]

### HTTP 요청 본문

요청의 본문은 요청에서 전송되는 정보의 '본문'을 포함하는 부분

HTTP 요청의 본문에는 사용자 이름 및 비밀번호 또는 양식에 입력된 기타 데이터와 같이 웹 서버에 제출되는 모든 정보가 포함

![[Pasted image 20260218155931.png]]


## HTTP 응답

HTTP 응답은 웹 클라이언트(종종 브라우저)에서 HTTP 요청에 대한 응답으로 인터넷 서버로부터 수신하는 응답
이러한 응답은 HTTP 요청에서 요청된 내용을 기반으로 중요한 정보를 전달함

일반적인 HTTP 응답에는 다음이 포함됩니다.

- HTTP 상태 코드
- HTTP 응답 헤더
- 선택 사항인 HTTP 본문

### HTTP 응답 헤더

HTTP 요청과 마찬가지로 HTTP 응답에는 응답 본문에서 전송되는 데이터의 언어 및 형식과 같은 중요한 정보를 전달하는 헤더가 함께 제공됨

![[Pasted image 20260218154637.png]]

### HTTP 응답 본문

'GET'요청에 대한 성공적인 HTTP 응답에는 일반적으로 요청된 정보가 포함된 본문이 있음
대부분의 웹 요청의 경우 이는 웹 브라우저에서 웹 페이지로 변환되는 HTML 데이터임

![[Pasted image 20260218160016.png]]

### 상태 코드
|코드|의미|
|---|---|
|200|OK|
|201|Created|
|204|No Content|
|400|Bad Request|
|401|Unauthorized|
|403|Forbidden|
|404|Not Found|
|500|Internal Server Error|

# 1. HTTP의 핵심 특성: Stateless & Connectionless

단순히 데이터를 주고받는 규칙을 넘어, HTTP가 네트워크 자원을 어떻게 관리하는지에 대한 이해가 필요

- **Stateless (무상태성):** 서버가 클라이언트의 이전 상태를 기억하지 않음        
- **Connectionless (비연결성):** 요청-응답이 끝나면 연결을 바로 끊음

HTTP 자체는 상태를 저장하지 않음. 
`요청 1 → 응답 / 요청 2 → 응답 / 요청 3 → 응답`
각 요청은 서로 독립적이기 때문에 서버는 이 사용자가 아까 로그인했는지 기본적으로 모름

👉 **그래서 ‘상태를 HTTP 바깥에서 유지’하는 메커니즘이 필요**

**세션 유지 방식**

1️⃣ Cookie + Session (Stateful 인증)
- 서버가 세션 저장   
- 클라이언트는 세션 ID만 보관
	- HTTP는 Stateless지만, 서버는 Stateful

2️⃣ JWT (Stateless 인증)
- 서버가 상태 저장 안 함
- 토큰 자체에 정보 포함
	- 서버가 상태를 저장하지 않음 → 진짜 Stateless 인증

# 2. HTTP 메서드의 '안전성'과 '멱등성'

단순히 GET/POST의 차이를 넘어, API를 설계할 때 반드시 고려해야 하는 성질

- **안전성 (Safe):** 호출해도 서버의 리소스를 변경하지 않는가? (GET 등)
- **멱등성 (Idempotent):** 동일한 요청을 여러 번 보내도 결과가 항상 같은가?
    
    - **GET, PUT, DELETE:** 멱등함 (삭제를 1번 하나 100번 하나 서버의 해당 데이터는 삭제된 상태 그대로임)
    - **POST:** 멱등하지 않음 (호출할 때마다 새로운 데이터가 생성될 수 있음)


# 3. HTTP 버전의 진화: 1.1에서 3까지


- **HTTP/1.1:** 가장 대중적이지만, 앞선 요청이 밀리면 뒤가 다 밀리는 **HOLB(Head-of-Line Blocking)** 문제 존재
- **HTTP/2:** 하나의 연결에 여러 요청을 동시에 보내는 **Multiplexing**으로 속도를 개선
- **HTTP/3:** TCP 대신 **UDP 기반의 QUIC** 프로토콜을 사용하여 연결 설정 시간을 극단적으로 줄임

# HTTP 페이지를 만들어보자

```
pip install flask
```

```python
from flask import Flask, request, jsonify
app = Flask(__name__)

# 1. GET 요청 테스트: 브라우저에서 바로 확인 가능
@app.route('/', methods=['GET'])
def home():
# 이 문자열이 '응답 본문(Response Body)'이 됩니다.
	return "<h1>소크라테스 서버에 오신 것을 환영합니다!</h1>"

# 2. POST 요청 테스트: '요청 본문(Payload)'을 받아서 처리
@app.route('/api/data', methods=['POST'])
def receive_data():
	# 클라이언트가 보낸 JSON 형태의 '요청 본문'을 가져옵니다.
	payload = request.json
	print(f"받은 데이터(Payload): {payload}")

	# 서버가 클라이언트에게 보내는 응답
	return jsonify({
		"status": "success",
		"message": "데이터를 성공적으로 받았습니다!",
		"your_data": payload
	}), 201 # 201은 'Created' 상태 코드

if __name__ == '__main__':
	app.run(debug=True, port=5000)
```

![[Pasted image 20260218170526.png]]

# HTTPS

하이퍼텍스트 전송 프로토콜 보안(HTTPS)은 웹 브라우저와 웹 사이트 간에 데이터를 전송하는 데 사용되는 기본 프로토콜인 HTTP의 보안 버전

 HTTPS는 데이터 전송의 보안을 강화하기 위해 암호화됨
 이는 사용자가 은행 계좌, 이메일 서비스, 의료 보험 공급자에 로그인하는 등 중요한 데이터를 전송할 때 특히 중요

### HTTPS 작동

HTTPS는 암호화 프로토콜을 사용하여 통신을 암호화함

이 프로토콜은 이전에는 보안 소켓 계층(SSL)으로 알려졌지만, Transport Layer Security(TLS)라고 불립니다. 이 프로토콜 비대킹 공개키 인프라로 알려진 것을 사용하여 통신을 보호

이 유형의 보안 시스템에서는 두 개의 서로 다른 키를 사용하여 두 당사자 간의 통신을 암호화함

1. 개인 키 - 이 키는 웹 사이트 소유자가 관리하며, 독자께서 짐작할 수 있듯이 비공개로 유지됨
		 이 키는 웹 서버에 있으며 공개 키로 암호화된 정보를 해독하는 데 사용됨
2. 공개 키 - 이 키는 안전한 방식으로 서버와 상호 작용하고자 하는 모든 사람이 사용할 수 있음
		 공개 키로 암호화된 정보는 개인 키로만 해독할 수 있음

> [!info] SSL/TLS
> SSL(Secure Socket Layer) 프로토콜은 웹서버와 브라우저 사이의 보안을 위해 만들어진 프로토콜로 Certificate Authority(CA)를 통해 서버와 클라이언트의 인증을 진행 하는데 사용된다.
> 
> SSL의 상위버전이 TLS이다. 
> TLS 버전 1.0은 SSL 버전 3.1로서 개발을 시작했지만 Netscape와 더 이상 연관이 없음을 명시하기 위해 발표 전에 프로토콜의 이름이 변경되었다.

### TLS 핸드셰이커 동작 벙식

1. Client Hello
2. Server Hello 
3. 인증서 전달
4. 공개키 기반 키 교환
5. 세션 키 생성
6. 이후 대칭키 암호화 통신

# HTTP vs HTTPS

HTTPS는 웹 사이트에서 네트워크를 스누핑하는 사람이 쉽게 볼 수 있는 방식으로 정보를 브로드캐스트하는 것을 방지함

일반 HTTP를 통해 정보를 전송할 때, 정보는 무료 소프트웨어를 사용하여 쉽게 '스니핑'할 수 있는 데이터 패킷으로 나뉘기에 공용 Wi-Fi와 같이 안전하지 않은 매체를 통한 통신은 도청에 매우 취약함 
실제로 HTTP를 통해 발생하는 모든 통신은 일반 텍스트로 이루어지므로 올바른 도구만 있으면 누구나 쉽게 접근할 수 있으며 경로상 공격에 취약함

엄밀히 말하면 HTTPS는 HTTP와 별개의 프로토콜이 아님
HTTPS는 단순히 HTTP 프로토콜을 통해 TLS/SSL 암호화를 사용하는 것!

사용자가 웹 페이지에 연결하면 웹 페이지에서 보안 세션을 시작하는 데 필요한 공개키가 포함된 SSL 인증서를 전송함
그런 다음 두 컴퓨터, 즉 클라이언트와 서버가 보안 연결을 설정하는 데 사용되는 일련의 주고받는 통신인 SSL/TLS 핸드셰이크라는 프로세스를 거침

# HTTPS 페이지를 만들어보자

```
pip install pyOpenSSL
```

```python
from flask import Flask, request, jsonify
app = Flask(__name__)

# 1. GET 요청 테스트: 브라우저에서 바로 확인 가능
@app.route('/', methods=['GET'])
def home():
# 이 문자열이 '응답 본문(Response Body)'이 됩니다.
	return "<h1>소크라테스 서버에 오신 것을 환영합니다!</h1>"

# 2. POST 요청 테스트: '요청 본문(Payload)'을 받아서 처리
@app.route('/api/data', methods=['POST'])
def receive_data():
	# 클라이언트가 보낸 JSON 형태의 '요청 본문'을 가져옵니다.
	payload = request.json
	print(f"받은 데이터(Payload): {payload}")

	# 서버가 클라이언트에게 보내는 응답
	return jsonify({
		"status": "success",
		"message": "데이터를 성공적으로 받았습니다!",
		"your_data": payload
	}), 201 # 201은 'Created' 상태 코드

if __name__ == '__main__':
	# ssl_context='adhoc'을 추가하면 Flask가 임시 TLS 인증서를 생성합니다.
	app.run(debug=True, port=5000, ssl_context='adhoc')
```

로컬에서 CA 발급을 흉내내기 위해 openssl로 직접 파일 만들기
```
openssl req -x509 -newkey rsa:4096 -nodes -out cert.pem -keyout key.pem -days 365
```

![[Pasted image 20260218171339.png]]
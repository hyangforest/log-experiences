# 2026-06-29 mkcert를 이용한 IIS 개발 인증서 구성

## 배경

개발 환경에서 HTTPS를 사용하기 위해 IIS 자체 서명(Self-Signed) 인증서를 생성하여 사용하고 있었다.
새로운 개발 사이트(`member-local.test.co.kr`)를 추가한 후 기존 방식대로 인증서를 연결했으나 HTTPS 통신 시 인증서 검증 오류가 발생하였다.
기존 방식 대신 `mkcert`를 이용하여 개발 인증서를 관리하는 방법을 검토하였다.


## 문제

`HttpClient` 호출 시 다음과 같은 예외가 발생하였다.

```text
AuthenticationException
원격 인증서가 잘못되었습니다.

SSL/TLS 보안 채널에 대한 트러스트 관계를 설정할 수 없습니다.
```

처음에는 TLS 버전 문제를 의심하였다.
하지만 실제로는 다음과 같은 상황이었다.
- `mkcert -pkcs12`로 `.pfx` 인증서를 생성하였다.
- 해당 `.pfx` 파일을 Windows 인증서 저장소의 Personal(개인용)에 Import 하였다.
- IIS에서 HTTPS Binding으로 해당 인증서를 연결하였다.

하지만 브라우저 및 클라이언트에서 **신뢰할 수 없는 인증서** 오류가 발생하였다.
그래서, MMC 통해서가 아닌 IIS 서버 인증서 가져오기를 통해서 진행하였고 이 방법은 오류가 발생하지 않았다.

같은 `.pfx`파일인데도 인증서 가져오기 방식에 따라 저장소에 등록되는 인증서 개수가 다른 것을 발견하였다.

- MMC에서 Personal(개인용) 저장소에 Import -> 인증서 2개 등록
- IIS 에서 "서버 인증서 가졍괴" -> 인증서 1개 등록


## 고민
* 왜 .pfx를 MMC로 Import하면 인증서가 2개 생기는가? IIS Import는 왜 1개만 생기는가? 두 방식은 내부적으로 무엇이 다른가?
* `mkcert`는 자체 서명 인증서를 생성하는 도구인가?
* Windows는 어떤 인증서를 신뢰하는 것일까?
* 인증서 체인은 정확히 무엇을 의미하는가? Root CA를 말하는 것인가?
* Trusted Root 와 Personal에 Root CA가 같이 존재하는 것이 문제를 유발하는가?

## 현재 결론
### `mkcert`는 루트 기관 + 서버 인증서 구조를 만든다.
```text
mkcert -install
        │
        ▼
로컬 Root CA 생성
        │
        ▼
Windows 신뢰 저장소 등록
        │
        ▼
Trusted Root Certification Authorities 등록
        │
        ▼
mkcert로 생성하는 모든 인증서는 이 Root CA가 서명한 "사용자(서버) 인증서
```

- `mkcert`로 신뢰할 수 있는 루트 기관을 등록하면, 그 다음에 생성되는 인증서는 그 루트가 서명한 사용자 인증서가 생성된다.
- 즉 서버 인증서 `.pfx` 파일을 생성할 때마다 새로운 Root CA가 만들어지는 것이 아니라, 이미 존재하는 Root CA를 이용해서 인증서를 발급하는 구조이다.
- 브라우저는 서버 인증서를 직접 신뢰하는 것이 아니라, **해당 인증서를 발급한 Root CA를 신뢰하기 때문에 서버 인증서도 신뢰**하게 된다.

### `mkcert`를 사용하여 여러 개발 도메인을 하나의 인증서로 생성할 수 있다.
```cmd
mkcert -pkcs12 ^
  -p12-file local-dev.pfx ^
  www-local.test.co.kr ^
  m-local.test.co.kr ^
  member-local.test.co.kr
```

- `-p12-file` 옵션은 출력 파일명만 지정하는 옵션이다.
- `.p12`와 `.pfx`는 동일한 PKCS#12 형식이다.
- `mkcert v1.4.4` 기준 PKCS#12 기본 암호는 `changeit` 이다. (버전에 따라 기본 암호 정책이 달라질 수 있으므로 사용하는 버전을 확인해야 한다.)
> cmd 결과 창에 출력되는 메시지: **The legacy PKCS#12 encryption password is the often hardcoded default "changeit" ℹ️** 

### 인증서 체인이란 무엇인가
```text
[서버 인증서 (leaf)]
    │
    ▼
[중간 인증서 (Intermediate CA)] (있을 수도 있고 없을 수도 있음)
    │
    ▼
[Root CA]
```
- 서버 인증서는 Root CA가 직접 서명하거나 또는 Intermediate CA를 거쳐 서명될 수 있다.
- 서버 인증서는 체인의 시작, Root CA 체인의 끝(최상위)으로 인증서 체인은 이 둘을 연결하는 전체 경로이다.

### `.pfx` 구성
`mkcert -pkcs12` 로 생성한 `.pfx` 파일에는 다음이 포함되어 있다.
1. 서버 인증서
2. Root CA 인증서
3. 개인 키

`.pfx`는 단일 인증서가 아니라 인증서 체인 + 개인 키로 묶은 파일이다.

### MMC Import 시 발생하는 일
MMC를 이용하여 Personal(개인용)에 `.pfx`를 Import하면 서버 인증서, Root CA 인증서 둘 다 등록되게 된다. Root CA가 Trusted Root가 아니라 개인용에 저장된다. 
Trusted Root에 없어서가 아니라, Personal에도 Root CA가 같이 들어와 있는 상태다.

이 상태는 다음과 같은 혼란을 만든다.

1. 체인 구성 시 어떤 인증서를 참조하는지 모호해짐
2. 동일한 Root CA가 다른 저장소에 중복 존재
3. 일부 클라이언트/환경에서 체인 검증이 꼬일 가능성 존재


### IIS Import 시 발생하는 일
.pfx 내부 구성에 Root CA 인증서가 포함되어 있어도 서버 인증서만 Personal(개인용)에 등록한다. 

## 의문
* MMC Import 시 Root CA를 자동으로 Trusted Root로 분리할 수는 없는가?

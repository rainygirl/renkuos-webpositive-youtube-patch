# WebPositive에서 YouTube: HaikuWebKit용 Media Source Extensions

English: [`README.md`](README.md).

HaikuWebKit 1.9.19에 **Media Source Extensions**(MSE)를 추가하는 패치입니다.
YouTube를 비롯한 대부분의 동영상 사이트가 이 API로 영상을 스트리밍합니다. MSE가
없으면 WebPositive는 YouTube 라이브를 전혀 재생하지 못하고, 일반 영상도 360p 단일
파일로만 재생합니다. WebPositive의 user agent는 바꾸지 않습니다.

대상은 32비트 Haiku(x86_gcc2 hybrid)이며, Sony VAIO P(Atom Z520, Intel
GMA500)에서 개발하고 측정했습니다.

## 패치 내용

- **Media Source Extensions** (`ENABLE_MEDIA_SOURCE`): H.264 영상과 AAC 음성의
  fragmented MP4를 받는 SourceBuffer와, 이를 Media Kit로 디코딩하는 미디어
  엔진. GMA500이 있으면 하드웨어 H.264 디코더([haiku-webpositive-msvdx](https://github.com/rainygirl/renkuos-webpositive-msvdx)),
  없으면 ffmpeg 플러그인을 씁니다. 영상은 오디오 클록에 맞춰 내보내고, 드롭된
  프레임 수를 페이지에 보고합니다. WebM/VP9/AV1은 지원하지 않는다고 답해 사이트가
  H.264를 보내게 합니다.
- HaikuWebKit 미디어 엔진을 **처음으로 등록**합니다(그동안 등록되지 않아
  `<video>`/`<audio>`가 전혀 재생되지 않았음). 재생 클록은 디코더 타임스탬프 대신
  실제 재생한 오디오 양으로 셉니다.
- `<video src="...m3u8">`용 **HTTP Live Streaming**.
- 미디어 열기를 **메인 스레드 밖으로** 옮겼습니다(전에는 열리는 동안 페이지가
  수 초씩 멈췄음).
- `canPlayType()`이 **코덱별로** 답합니다(존재하지 않는 코덱에도 "예"라고 했음).
- 재생 중이나 재생 후 WebPositive를 **종료할 때의 크래시**를 고쳤습니다.
- CSS 필터 결과를 **레이어가 바뀌지 않는 동안 재사용**합니다. 전에는 근처가
  다시 그려질 때마다 블러를 새로 계산했습니다. YouTube 플레이어 뒤의 흐린 배경이
  메인 스레드의 12.7%를 쓰던 것이 0.5%로 줄었습니다.
- 32비트 x86 JavaScriptCore용 **어셈블리 인터프리터**
  (`haikuwebkit-1.9.19-x86-llint.patch`). 업스트림에는 이식형 C 루프만 있어서
  offlineasm의 x86-32 백엔드를 새로 만들었습니다. 인터프리터로 도는 JavaScript가
  4분의 1~3분의 1 빨라집니다.

## 현재 상태

VAIO P에서 원래 user agent로 측정한 결과입니다.

- `https://www.youtube.com/watch?v=vOTiJkg1voo`(ABC NEWS Australia 라이브)가
  MSE로 **재생됩니다**. 640x360 H.264를 GMA500으로 디코딩하고 AAC 음성도
  나오며, 데이터가 있는 동안은 30fps에 **드롭 0**입니다.
- **아직 끊김 없이 이어지지는 않습니다.** YouTube 플레이어가 다음 조각을
  8~30초 간격으로만 요청해서 몇 초 재생하고 기다리기를 반복하고, 첫 화면까지
  약 3분이 걸립니다. 요청 자체는 0.3~2초면 끝나고, 느린 것은 요청 시점을 정하는
  YouTube JavaScript입니다. 어셈블리 인터프리터가 도움은 되지만(첫 화면 170초,
  C 루프는 190~240초) 인터프리터는 JIT가 아니고, 업스트림에는 32비트 x86용 JIT가
  없습니다. 자세한 내용은 [`AGENTS.md`](AGENTS.md)에 있습니다.
- 일반 YouTube 영상은 이전처럼 재생됩니다.

## 빌드

`webkit/build-haikuwebkit-x86.sh`가 Linux에서 `haikuwebkit_x86` 1.9.19-22를
크로스 빌드합니다.
[haiku-rwebpositive-arm64](https://github.com/rainygirl/renkuos-rwebpositive-arm64)의
x86 빌드(메모리 누수 수정)에 이 저장소의 패치 두 개를 얹어 실행하므로, `RWP`에 그
체크아웃 경로를 지정하세요.

```sh
RWP=/path/to/haiku-rwebpositive-arm64 webkit/build-haikuwebkit-x86.sh
```

또는 `haikuwebkit-1.9.19-x86.patch`가 적용된 HaikuWebKit 1.9.19 트리에 직접
적용합니다.

```sh
patch -p0 < webkit/haikuwebkit-1.9.19-youtube.patch
patch -p0 < webkit/haikuwebkit-1.9.19-x86-llint.patch
```

이미 있는 빌드 디렉터리를 쓴다면 `-DENABLE_MEDIA_SOURCE=ON`과
`-DENABLE_C_LOOP=OFF`도 주세요. CMake가 예전 캐시 값을 유지하면 MSE 코드가
통째로 빠지고 C 루프가 그대로 남습니다. LLInt 빌드에는 Ruby(offlineasm)가
필요합니다(모든 WebKit 빌드와 같음).

## 디버깅

WebPositive 환경변수에 `WEBKIT_MEDIA_DEBUG=1`을 주면 다음을 stderr에 출력합니다.
- 모든 `canPlayType`/`isTypeSupported` 응답
- 파싱한 트랙과 조각
- 선택된 디코더
- 2초마다 상태 한 줄(클록, 큐 길이, 표시/드롭 프레임)

페이지 콘솔도 stderr로 복사합니다.

## 라이선스

패치는 HaikuWebKit을 수정하므로 그 라이선스(GNU LGPL v2, BSD)를 따릅니다. 새
파일에는 LGPL 헤더가 붙어 있습니다. [`LICENSE`](LICENSE) 참고.

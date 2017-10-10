---
layout: post
title: "어렵게 WebAssembly 써보기"
description: "WebAssembly를 어려운 방법으로 한번 써보기."
date: 2017-03-18 17:16:43
---

최근 (아마도) 뜨고 있는 웹 기술 중 하나는 역시 WebAssmebly(이하 wasm). 한번 써보기로 했다.

보통은 C/C++ 등으로 코드를 짠 뒤 Emscripten을 이용해 asm.js로 컴파일된 코드를 다시 wasm으로 컴파일하는 방식이겠지만, 나는 C에 젬병이기 때문에 다른 방법을 쓰기로 했다.

방법 중 하나는 asm.js로 코드를 짠 뒤 [Binaryen](https://github.com/WebAssembly/binaryen)의 `asm2wasm`으로 컴파일하는 방법, 다른 하나는 텍스트 포맷으로 코드를 짜는 것이다.

asm.js로 코드를 짜보는 것을 시도해봤지만, 이쪽은 제대로 된 툴이 존재하질 않았기 때문에 다른 방법을 사용했다.

## Text Format, S-expression

Wasm에는 바이너리 포맷뿐만 아니라 [텍스트 포맷](https://github.com/WebAssembly/design/blob/master/TextFormat.md)이 존재한다. 즉 텍스트 포맷으로 코드를 짜서 바이너리 포맷으로 변환이 가능하다는 것이다.

두 포맷간의 변환에는 Binaryen의 `wasm-as`, `wasm-dis`를 사용할 수도 있고, [WABT](https://github.com/WebAssembly/wabt)의 `wast2wasm`, `wasm2wast`를 사용할 수도 있다. 다만 [MDN에서는](https://developer.mozilla.org/en-US/docs/WebAssembly/Understanding_the_text_format) WABT의 툴을 사용하고 있는 것도 있고, `wast2wasm`은 바이너리 포맷을 자세하게 보여주는 기능이 존재하기 때문에 공부에 도움이 되는 것도 있고 해서 `wast2wasm`을 사용하기로 했다.

## WABT

WABT를 설치한다. 설치는 굉장히 쉬우며, GitHub 페이지에 있는대로 하면 된다.

```
$ git clone --recursive https://github.com/WebAssembly/wabt
$ cd wabt
$ make
/bin/sh: 1: cmake: not found
make: *** [out/clang/Debug/Makefile] Error 127
```

...

```
$ sudo apt-get install cmake
$ make
CMake Error: your C compiler: "clang" was not found.   Please set CMAKE_C_COMPILER to a valid compiler path or name.
CMake Error: your CXX compiler: "clang++" was not found.   Please set CMAKE_CXX_COMPILER to a valid compiler path or name.
```

clang은 없지만 gcc로 컴파일 하는 것도 가능하다. README에 있다.

```
$ make gcc-release
$ out/gcc/Release/wast2wasm
usage: wast2wasm [options] filename
```

완료.

## 첫 WASM 모듈

텍스트 에디터를 켜서 간단한 모듈을 만든다.

```
$ cat module1.wat
(module)
```

간단하기 그지없다. 아무것도 없는 모듈이다.
컴파일도 간단하다.

```
$ ls
module1.wat  wabt

$ wabt/out/gcc/Release/wast2wasm module1.wat -o module1.wasm

$ ls
module1.wat  module1.wasm  wabt
```

혹은 바이너리 포맷을 공부하고 싶다면,

```
$ wabt/out/gcc/Release/wast2wasm module1.wat -v
0000000: 0061 736d                                 ; WASM_BINARY_MAGIC
0000004: 0100 0000                                 ; WASM_BINARY_VERSION
```

이렇게.

모듈을 만들었으니 아무 기능도 없지만 돌려봐야 한다.
Node.js에는 wasm이 기본으로 활성화되어 있진 않지만, 플래그를 주면 써볼 수 있다. 정확히 어느 버전부터 있는지는 잘 모르겠다.

```
$ node -v
v7.7.3

$ node
> WebAssembly
ReferenceError: WebAssembly is not defined

$ node --expose-wasm
> WebAssembly
{ compile: [Function],
  validate: [Function],
  Module: [Function],
  Instance: [Function],
  Table: [Function],
  Memory: [Function] }
```

이런 식으로 하면 된다.

```
> WebAssembly.compile(fs.readFileSync("module1.wasm"))
Promise {
  <rejected> Error: WebAssembly.compile(): Wasm decoding failedResult = expected version 0c 00 00 00, found 01 00 00 00 @+4
(...)
```

...???

현재 버전의 WABT는 wasm의 최신 버전, `0x01`에 맞는 코드를 생성하지만, 현재 Node.js stable에 포함된 V8에서는 지원되지 않는 버전이다.

```
$ node -v
v8.0.0-nightly20170317704263139b

$ node --expose-wasm
> WebAssembly.compile(fs.readFileSync("module1.wasm"))
Promise {
  <rejected> CompileError: WebAssembly.compile(): Wasm decoding failedResult = expected version 0d 00 00 00, found 01 00 00 00 @+4
```

Nightly에서도 안된다.

이걸 넘는 방법은 몇가지가 있는데, Polyfill을 사용하거나 브라우저에서 돌리는 것이다.

Binaryen에는 `binaryen.js`라는 JS Polyfill이 존재하는데, 이걸 어떻게 써야 하는지 알 수가 없었다. 어쩔 수 없이 브라우저를 쓰기로 했다.

Firefox에서는 `about:config` 페이지의 `javascript.options.wasm`를 `true`로, Chrome에서는 `chrome://flags/#enable-webassembly`에서 사용 설정을 선택한다. Firefox Nightly와 Chrome Canary를 써야만 되는지는 잘 모르겠다. 나는 Firefox Developer Edition 54.0a2를 사용중이다.

아래의 HTML을 만든다.

```
<meta charset="UTF-8"><script>
function loadWasmModule(filename) {
  fetch(filename)
    .then(res => res.arrayBuffer())
    .then(buf => WebAssembly.compile(buf))
    .then(module => {
      let imports = {};
      return new WebAssembly.Instance(module, imports)
    })
    .then(instance => {
      window.instance = instance;
      console.log(`${filename} loaded and usable from %cwindow.instance`, "font-weight:bold;text-decoration:underline");
    });
}
</script>
```

브라우저에서 html 파일을 연 뒤 콘솔에서,

```
> loadWasmModule("module1.wasm")
< undefined
  module1.wasm loaded and usable from window.instance
> window.instance
< WebAssembly.Instance { exports: Object }
```

드디어 써볼 수 있다.

```
> window.instance.exports
< Object {  }
```

당연히 아무 것도 없다. 애초에 들어간 코드가 없으니까.

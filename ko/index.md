<div class="hero">
<img src="../logo.svg" alt="">

# Rok 문서

<div class="lead">Rok의 가이드, 예제, 언어 참조와 표준 라이브러리입니다. 위 메뉴에서 고른 버전의 문서입니다.</div>
<div class="install"><pre><code>curl -fsSL https://rok-lang.org/<wbr>install.sh | sh</code></pre><p>macOS와 Linux용입니다. Windows에서는 <a href="learn/01-getting-started.html#설치-스크립트">install.ps1</a>을 씁니다.</p></div>
</div>

<div class="cards">
<a class="card" href="learn/index.html"><span class="tag">learn/</span><h3>Rok 배우기</h3><p>설치부터 동시성, FFI, stone, 쿡북까지 스무 장입니다. 모든 코드 예제는 사이트를 빌드할 때 검사합니다.</p></a>
<a class="card" href="examples/index.html"><span class="tag">examples/</span><h3>예제</h3><p>작고 완결된 프로그램입니다. CLI 도구, 파일 처리, TCP 에코 서버, 병렬 map, 인터프리터 등이 있습니다.</p></a>
<a class="card" href="spec/refs-lang/index.html"><span class="tag">spec/</span><h3>언어 참조</h3><p>어휘 구조부터 타입, 표현식, 모듈, 메타 시스템, 런타임, ABI까지 열 장입니다.</p></a>
<a class="card" href="spec/refs-standard-libraries/index.html"><span class="tag">std/</span><h3>표준 라이브러리</h3><p><code>bare</code> / <code>core</code> / <code>std</code> 세 계층의 사양입니다.</p></a>
<a class="card" href="../std/index.html"><span class="tag">api/</span><h3>API</h3><p>표준 라이브러리의 모든 모듈, 타입, 함수입니다. 소스에서 생성합니다.</p></a>
<a class="card" href="spec/refs-lang/009-tools/index.html"><span class="tag">tools/</span><h3>도구</h3><p><code>rokc</code> 컴파일러, <code>rok</code> 패키지 도구, REPL, Jupyter 커널, LSP 서버, 디버깅 도구입니다.</p></a>
<a class="card" href="errors/index.html"><span class="tag">errors/</span><h3>에러 코드</h3><p><code>rokc</code> 에러 코드마다 한 쪽씩, 에러가 나는 예와 고치는 방법을 보여 줍니다.</p></a>
</div>

## 버전

릴리스마다 문서가 따로 있습니다. Rok 0.1.x는 `docs.rok-lang.org/0.1/`에 있는 식입니다. `latest/`는 최신 릴리스이고 `dev/`는 main 브랜치를 따라갑니다. 언어의 동기와 철학은 [사양 소개](spec/index.html)에 있습니다. 패키지(stone)는 [rokstones.io](https://rokstones.io)에 있습니다.

# 📝 Markdown Preview 기술 문서

> Marked.js와 Highlight.js를 활용한 실시간 마크다운 미리보기 구현

## 📋 목차

1. [프로젝트 개요](#프로젝트-개요)
2. [Marked.js 마크다운 파싱](#markedjs-마크다운-파싱)
3. [Highlight.js 코드 하이라이팅](#highlightjs-코드-하이라이팅)
4. [GitHub Flavored Markdown](#github-flavored-markdown)
5. [실시간 미리보기](#실시간-미리보기)
6. [파일 다운로드 구현](#파일-다운로드-구현)
7. [화면 동일 PDF 저장](#화면-동일-pdf-저장)
8. [LocalStorage 활용](#localstorage-활용)
9. [UI/UX 구현](#uiux-구현)

---

## 프로젝트 개요

### 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                    Markdown Preview                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────────────┐        ┌─────────────────┐            │
│   │   📝 Editor     │        │   👁️ Preview    │            │
│   │   (textarea)    │  ───▶  │   (rendered)    │            │
│   │                 │        │                 │            │
│   │   Markdown      │        │   HTML          │            │
│   └─────────────────┘        └─────────────────┘            │
│            │                          │                      │
│            ▼                          ▼                      │
│   ┌─────────────────┐        ┌─────────────────┐            │
│   │   Marked.js     │        │  Highlight.js   │            │
│   │   (Parser)      │        │  (Syntax HL)    │            │
│   └─────────────────┘        └─────────────────┘            │
│                                                              │
│   ┌─────────────────────────────────────────────┐           │
│   │              LocalStorage                    │           │
│   │   (content, theme persistence)               │           │
│   └─────────────────────────────────────────────┘           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 사용된 라이브러리

| 라이브러리 | 버전 | 용도 | CDN |
|-----------|------|------|-----|
| **Marked.js** | 12.0.0 | Markdown → HTML 변환 | cdnjs |
| **Highlight.js** | 11.9.0 | 코드 문법 하이라이팅 | cdnjs |

```html
<!-- Marked.js -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/marked/12.0.0/marked.min.js"></script>

<!-- Highlight.js -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/highlight.min.js"></script>
<link rel="stylesheet" href="...highlight.js/.../github-dark.min.css">
```

---

## Marked.js 마크다운 파싱

### Marked.js란?

**Marked.js**는 JavaScript로 작성된 빠른 마크다운 파서입니다.

- ⚡ **빠름**: C로 작성된 파서와 비슷한 속도
- 📦 **경량**: 압축 시 ~35KB
- 🌐 **표준 준수**: CommonMark 명세 지원
- 🐙 **GFM 지원**: GitHub Flavored Markdown

### 기본 사용법

```javascript
// 마크다운 → HTML 변환
const markdown = '# Hello World';
const html = marked.parse(markdown);
// 결과: '<h1>Hello World</h1>'
```

### 옵션 설정

```javascript
marked.setOptions({
    gfm: true,        // GitHub Flavored Markdown 활성화
    breaks: true,     // 줄바꿈을 <br>로 변환
    highlight: function(code, lang) {
        // 코드 하이라이팅 함수
        if (lang && hljs.getLanguage(lang)) {
            try {
                return hljs.highlight(code, { language: lang }).value;
            } catch (e) {}
        }
        return hljs.highlightAuto(code).value;
    }
});
```

### 파싱 과정

```
Markdown 입력
    │
    ▼
┌─────────────────┐
│    Lexer        │  ← 토큰화 (Tokenization)
│   (토큰 생성)    │
└────────┬────────┘
         │
    Token Array
    [heading, paragraph, code, ...]
         │
         ▼
┌─────────────────┐
│    Parser       │  ← 토큰 → AST 변환
│   (구문 분석)    │
└────────┬────────┘
         │
    AST (추상 구문 트리)
         │
         ▼
┌─────────────────┐
│   Renderer      │  ← AST → HTML 변환
│   (HTML 생성)    │
└────────┬────────┘
         │
         ▼
    HTML 출력
```

### 커스텀 렌더러

Marked.js는 **커스텀 렌더러**를 통해 출력을 커스터마이징할 수 있습니다:

```javascript
const renderer = new marked.Renderer();

// 체크박스 목록 커스터마이징
renderer.listitem = function(text) {
    if (text.includes('type="checkbox"')) {
        // 체크박스가 있는 항목은 특별한 클래스 추가
        return `<li class="task-list-item">${text}</li>`;
    }
    return `<li>${text}</li>`;
};

marked.setOptions({ renderer });
```

**왜 커스텀 렌더러가 필요한가?**

GFM의 체크박스 목록(`- [x] 완료`)을 위해 CSS 스타일링을 적용하려면
목록 항목에 특별한 클래스가 필요합니다.

---

## Highlight.js 코드 하이라이팅

### 작동 원리

Highlight.js는 코드를 분석하여 **토큰별로 `<span>` 태그**를 추가합니다:

```javascript
// 입력
function hello() {
    return "world";
}

// 출력 (하이라이팅된 HTML)
<span class="hljs-keyword">function</span> 
<span class="hljs-title">hello</span>() {
    <span class="hljs-keyword">return</span> 
    <span class="hljs-string">"world"</span>;
}
```

### Marked.js와 통합

```javascript
marked.setOptions({
    highlight: function(code, lang) {
        // 언어가 지정되었고 Highlight.js가 지원하는 경우
        if (lang && hljs.getLanguage(lang)) {
            try {
                return hljs.highlight(code, { language: lang }).value;
            } catch (e) {
                console.error('Highlight error:', e);
            }
        }
        // 언어 미지정 시 자동 감지
        return hljs.highlightAuto(code).value;
    }
});
```

### 동적 하이라이팅

Marked.js의 `highlight` 옵션은 초기 파싱 시에만 적용됩니다.
**동적으로 추가된 코드 블록**은 별도로 하이라이팅해야 합니다:

```javascript
function updatePreview() {
    const markdown = editor.value;
    preview.innerHTML = marked.parse(markdown);
    
    // 이미 파싱된 HTML 내의 코드 블록을 다시 하이라이팅
    preview.querySelectorAll('pre code').forEach(block => {
        hljs.highlightElement(block);
    });
}
```

### 테마 전환

Highlight.js는 **CSS 파일로 테마**를 제공합니다. 다크/라이트 모드 전환 시:

```html
<!-- 두 테마를 모두 로드하고, disabled 속성으로 전환 -->
<link rel="stylesheet" href=".../github-dark.min.css" id="hljs-theme-dark">
<link rel="stylesheet" href=".../github.min.css" id="hljs-theme-light" disabled>
```

```javascript
function setTheme(theme) {
    const hljsDark = document.getElementById('hljs-theme-dark');
    const hljsLight = document.getElementById('hljs-theme-light');
    
    if (theme === 'dark') {
        hljsDark.disabled = false;
        hljsLight.disabled = true;
    } else {
        hljsDark.disabled = true;
        hljsLight.disabled = false;
    }
}
```

### 지원 언어

Highlight.js는 **190개 이상의 언어**를 지원합니다:

| 카테고리 | 언어 |
|---------|------|
| 웹 | JavaScript, TypeScript, HTML, CSS, JSON |
| 백엔드 | Python, Java, Go, Rust, Ruby, PHP |
| 시스템 | C, C++, Rust, Assembly |
| 쉘 | Bash, PowerShell, Zsh |
| 데이터 | SQL, YAML, XML, Markdown |
| 기타 | Dockerfile, Makefile, Nginx |

---

## GitHub Flavored Markdown

### GFM이란?

**GitHub Flavored Markdown (GFM)**은 GitHub이 확장한 마크다운 명세입니다.

### 표준 마크다운 vs GFM

| 기능 | 표준 Markdown | GFM |
|------|--------------|-----|
| 테이블 | ❌ | ✅ |
| 체크박스 | ❌ | ✅ |
| 취소선 | ❌ | ✅ `~~text~~` |
| 자동 링크 | ❌ | ✅ |
| 코드 블록 (펜스) | 일부 | ✅ ` ``` ` |
| 이모지 | ❌ | ✅ `:emoji:` |

### 테이블 문법

```markdown
| 왼쪽 | 가운데 | 오른쪽 |
|:-----|:------:|-------:|
| L    |   C    |      R |
```

**정렬 규칙**:
- `:---` 왼쪽 정렬
- `:---:` 가운데 정렬
- `---:` 오른쪽 정렬

### 체크박스 (Task List)

```markdown
- [x] 완료된 항목
- [ ] 미완료 항목
```

**파싱 결과**:
```html
<ul>
  <li class="task-list-item">
    <input type="checkbox" checked disabled> 완료된 항목
  </li>
  <li class="task-list-item">
    <input type="checkbox" disabled> 미완료 항목
  </li>
</ul>
```

### 취소선

```markdown
~~취소된 텍스트~~
```

→ `<del>취소된 텍스트</del>`

### GFM 활성화

```javascript
marked.setOptions({
    gfm: true,     // GFM 활성화
    breaks: true   // 줄바꿈을 <br>로 (GFM 스타일)
});
```

---

## 실시간 미리보기

### 이벤트 기반 업데이트

```javascript
const editor = document.getElementById('editor');
const preview = document.getElementById('preview');

function updatePreview() {
    const markdown = editor.value;
    
    // 1. 마크다운 파싱
    preview.innerHTML = marked.parse(markdown);
    
    // 2. 코드 하이라이팅 재적용
    preview.querySelectorAll('pre code').forEach(block => {
        hljs.highlightElement(block);
    });
    
    // 3. 글자/단어 수 업데이트
    updateWordCount(markdown);
    
    // 4. 자동 저장
    localStorage.setItem('markdown-content', markdown);
}

// input 이벤트: 모든 입력에 반응
editor.addEventListener('input', updatePreview);
```

### 왜 `input` 이벤트인가?

| 이벤트 | 발생 시점 | 적합성 |
|--------|----------|--------|
| `keydown` | 키 누를 때 | ❌ 값 변경 전 |
| `keyup` | 키 뗄 때 | △ 복붙 미감지 |
| `change` | 포커스 잃을 때 | ❌ 실시간 아님 |
| **`input`** | **값 변경 시** | ✅ **모든 변경 감지** |

`input` 이벤트는 키보드 입력, 복사/붙여넣기, 드래그앤드롭 등
**모든 값 변경**을 감지합니다.

### 글자/단어 수 계산

```javascript
function updateWordCount(markdown) {
    // 글자 수: 문자열 길이
    const chars = markdown.length;
    
    // 단어 수: 공백으로 분리
    const words = markdown.trim() 
        ? markdown.trim().split(/\s+/).length 
        : 0;
    
    wordCount.textContent = `${chars.toLocaleString()} 자 · ${words.toLocaleString()} 단어`;
}
```

**정규표현식 `\s+` 설명**:
- `\s`: 공백 문자 (스페이스, 탭, 줄바꿈)
- `+`: 1개 이상 연속

### Tab 키 지원

기본적으로 `Tab` 키는 포커스를 이동시킵니다. 에디터에서는 들여쓰기로 사용:

```javascript
editor.addEventListener('keydown', (e) => {
    if (e.key === 'Tab') {
        e.preventDefault();  // 기본 동작 방지
        
        const start = editor.selectionStart;
        const end = editor.selectionEnd;
        
        // 현재 커서 위치에 4칸 스페이스 삽입
        editor.value = editor.value.substring(0, start) 
                     + '    ' 
                     + editor.value.substring(end);
        
        // 커서를 삽입된 스페이스 뒤로 이동
        editor.selectionStart = editor.selectionEnd = start + 4;
        
        updatePreview();
    }
});
```

---

## 파일 다운로드 구현

### Blob API

**Blob (Binary Large Object)**은 파일과 같은 불변 원시 데이터를 나타냅니다.

```javascript
// 텍스트로 Blob 생성
const blob = new Blob(
    [editor.value],           // 데이터 배열
    { type: 'text/markdown' } // MIME 타입
);
```

### 다운로드 트리거

```javascript
document.getElementById('download-md').addEventListener('click', () => {
    // 1. Blob 생성
    const blob = new Blob([editor.value], { type: 'text/markdown' });
    
    // 2. Blob URL 생성
    const url = URL.createObjectURL(blob);
    
    // 3. 임시 링크 생성
    const a = document.createElement('a');
    a.href = url;
    a.download = 'document.md';  // 다운로드 파일명
    
    // 4. 프로그래밍 방식으로 클릭
    a.click();
    
    // 5. 메모리 정리
    URL.revokeObjectURL(url);
    
    showToast('다운로드 완료!');
});
```

### 단계별 설명

```
1. Blob 생성
   ┌─────────────────┐
   │ "# Hello World" │ → Blob 객체
   └─────────────────┘

2. Blob URL 생성
   blob:http://localhost/abc-123-def

3. 임시 <a> 태그 생성
   <a href="blob:..." download="document.md">

4. 프로그래밍 클릭
   → 브라우저 다운로드 시작

5. URL 해제
   → 메모리 누수 방지
```

### MIME 타입

| 파일 형식 | MIME 타입 |
|----------|----------|
| Markdown | `text/markdown` |
| HTML | `text/html` |
| Plain Text | `text/plain` |
| JSON | `application/json` |

---

## 화면 동일 PDF 저장

PDF 버튼은 브라우저 인쇄 기능을 호출하지 않고 `html2canvas`로 현재 미리보기 패널을 시각적으로 캡처합니다. 패널 전체를 복제해 캡처하므로 일반 모드와 전체화면 읽기 모드의 스타일을 유지합니다. 캔버스를 A4 비율의 페이지 단위로 잘라 `jsPDF`에 넣고 파일을 즉시 다운로드합니다. `document.fonts.ready`와 미리보기 이미지 로딩을 기다리므로 웹폰트가 기본 글꼴로 바뀌거나 이미지가 빈칸으로 출력되는 문제를 줄입니다.

이 방식은 텍스트를 PDF의 선택 가능한 텍스트 레이어로 변환하지 않고 화면을 이미지로 보존합니다. 따라서 브라우저별 PDF 레이아웃 차이 없이 시각적 형태를 우선적으로 유지합니다.

## LocalStorage 활용

### 자동 저장

모든 입력은 **LocalStorage에 자동 저장**됩니다:

```javascript
function updatePreview() {
    const markdown = editor.value;
    preview.innerHTML = marked.parse(markdown);
    
    // 자동 저장
    localStorage.setItem('markdown-content', markdown);
}
```

### 복원

페이지 로드 시 저장된 내용 복원:

```javascript
const savedContent = localStorage.getItem('markdown-content');
if (savedContent) {
    editor.value = savedContent;
    updatePreview();
}
```

### 테마 저장

```javascript
// 저장
localStorage.setItem('markdown-preview-theme', 'dark');

// 불러오기
const savedTheme = localStorage.getItem('markdown-preview-theme') || 'dark';
```

### LocalStorage vs SessionStorage

| 특성 | LocalStorage | SessionStorage |
|------|-------------|----------------|
| 지속성 | **영구** (명시적 삭제 전까지) | 탭/창 닫으면 삭제 |
| 범위 | 같은 도메인 모든 탭 | 현재 탭만 |
| 용량 | ~5-10MB | ~5-10MB |
| 용도 | 설정, 저장된 문서 | 임시 상태 |

### 저장 데이터 구조

```javascript
// 이 앱에서 사용하는 키들
{
    'markdown-content': '# 문서 내용...',
    'markdown-preview-theme': 'dark' | 'light'
}
```

---

## UI/UX 구현

### 분할 레이아웃

CSS Grid를 사용한 **50:50 분할**:

```css
.editor-container {
    display: grid;
    grid-template-columns: 1fr 1fr;  /* 동일한 너비 */
    gap: 20px;
    flex: 1;
    min-height: 0;  /* 중요: flex 자식의 오버플로우 처리 */
}
```

### 반응형 디자인

모바일에서는 **세로 분할**:

```css
@media (max-width: 900px) {
    .editor-container {
        grid-template-columns: 1fr;         /* 1열 */
        grid-template-rows: 1fr 1fr;        /* 2행 */
    }
}
```

```
Desktop (가로 분할)          Mobile (세로 분할)
┌─────────┬─────────┐        ┌─────────────────┐
│ Editor  │ Preview │        │     Editor      │
│         │         │   →    ├─────────────────┤
│         │         │        │     Preview     │
└─────────┴─────────┘        └─────────────────┘
```

### 전체화면 모드

```javascript
document.getElementById('fullscreen-editor').addEventListener('click', () => {
    document.getElementById('editor-panel').classList.toggle('fullscreen');
});

// ESC 키로 종료
document.addEventListener('keydown', (e) => {
    if (e.key === 'Escape') {
        document.querySelectorAll('.panel.fullscreen').forEach(panel => {
            panel.classList.remove('fullscreen');
        });
    }
});
```

```css
.panel.fullscreen {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    z-index: 1000;
    border-radius: 0;
}
```

### 드롭다운 메뉴

```javascript
const templatesBtn = document.getElementById('templates-btn');
const templatesMenu = document.getElementById('templates-menu');

// 버튼 클릭 시 토글
templatesBtn.addEventListener('click', (e) => {
    e.stopPropagation();  // 이벤트 버블링 방지
    templatesMenu.classList.toggle('show');
});

// 외부 클릭 시 닫기
document.addEventListener('click', () => {
    templatesMenu.classList.remove('show');
});
```

**`stopPropagation()` 필요 이유**:

```
버튼 클릭
    │
    ├─ 버튼 리스너: 메뉴 열기
    │
    └─ 버블링 → document 리스너: 메뉴 닫기  ← 이걸 방지!
```

### 토스트 알림

```javascript
function showToast(message) {
    // 기존 토스트 제거
    const existing = document.querySelector('.toast');
    if (existing) existing.remove();
    
    // 새 토스트 생성
    const toast = document.createElement('div');
    toast.className = 'toast';
    toast.textContent = message;
    document.body.appendChild(toast);
    
    // 2초 후 제거
    setTimeout(() => toast.remove(), 2000);
}
```

```css
.toast {
    position: fixed;
    bottom: 30px;
    left: 50%;
    transform: translateX(-50%);
    background: var(--accent-primary);
    color: white;
    padding: 12px 24px;
    border-radius: 8px;
    animation: slideUp 0.3s ease;
}

@keyframes slideUp {
    from { 
        opacity: 0; 
        transform: translateX(-50%) translateY(20px); 
    }
    to { 
        opacity: 1; 
        transform: translateX(-50%) translateY(0); 
    }
}
```

### 클립보드 복사

```javascript
// Markdown 복사
document.getElementById('copy-md').addEventListener('click', () => {
    navigator.clipboard.writeText(editor.value).then(() => {
        showToast('Markdown 복사 완료!');
    });
});

// 렌더링된 HTML 복사
document.getElementById('copy-html').addEventListener('click', () => {
    navigator.clipboard.writeText(preview.innerHTML).then(() => {
        showToast('HTML 복사 완료!');
    });
});
```

---

## 성능 고려사항

### 디바운싱 (선택적)

대용량 문서의 경우 **디바운싱**을 적용할 수 있습니다:

```javascript
function debounce(func, wait) {
    let timeout;
    return function(...args) {
        clearTimeout(timeout);
        timeout = setTimeout(() => func.apply(this, args), wait);
    };
}

// 100ms 대기 후 업데이트
const debouncedUpdate = debounce(updatePreview, 100);
editor.addEventListener('input', debouncedUpdate);
```

**현재 구현에서는 사용하지 않음**: 일반적인 문서 크기에서는 즉각적인 피드백이 더 나은 UX를 제공합니다.

### Marked.js 비동기 처리

대용량 문서의 경우:

```javascript
// 비동기 파싱 (대용량 문서용)
marked.parse(markdown, { async: true })
    .then(html => {
        preview.innerHTML = html;
    });
```

---

## 마치며

### 핵심 기술 요약

| 기능 | 기술 |
|------|------|
| Markdown 파싱 | Marked.js |
| 코드 하이라이팅 | Highlight.js |
| 실시간 업데이트 | `input` 이벤트 |
| 파일 다운로드 | Blob API |
| 데이터 저장 | LocalStorage |
| 테마 전환 | CSS 변수 + disabled 속성 |

### 확장 가능성

1. **이미지 업로드**: Base64 인코딩 또는 외부 서비스 연동
2. **협업 편집**: WebSocket 실시간 동기화
3. **오프라인 지원**: Service Worker로 PWA 구현
4. **내보내기**: PDF 변환 (html2pdf.js)

### 참고 자료

- [Marked.js 공식 문서](https://marked.js.org/)
- [Highlight.js 공식 문서](https://highlightjs.org/)
- [GitHub Flavored Markdown 명세](https://github.github.com/gfm/)
- [CommonMark 명세](https://commonmark.org/)

---

*작성일: 2026년 1월*

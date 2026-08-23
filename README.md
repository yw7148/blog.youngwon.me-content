# Youngwon Tech Blog Posts

`youngwon-tech-blog`에 게시할 기술 글을 Markdown으로 모아 관리하는 저장소입니다.

## 구조

```text
ideas/          AI와 발전시키기 전의 간단한 아이디어 메모
posts/          글 초안과 발행 원고
templates/      새 글 작성용 템플릿
```

파일명이 블로그의 URL slug가 됩니다. 예를 들어 `posts/my-post.md`는 블로그에서
`/posts/my-post/`로 발행합니다.

## 글 작성

`templates/post.md`를 복사해 `posts/<slug>.md`로 만든 뒤 작성합니다. 초안은
`draft: true`, 발행할 글은 `draft: false`로 설정합니다.

지원하는 frontmatter 필드는 [SCHEMA.md](./SCHEMA.md)에 정리되어 있습니다.

아직 글의 형태가 잡히지 않은 주제는 `ideas/`에 먼저 기록합니다. 핵심 아이디어와
bullet point만 적어 두었다가 AI와 함께 `posts/`의 정식 원고로 발전시킬 수 있습니다.

## 블로그에 반영

이 저장소의 `posts/*.md`를 `youngwon-tech-blog/src/content/posts/`에 동기화한 뒤
블로그 저장소에서 다음 명령으로 검증합니다.

```sh
npm run build
```

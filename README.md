# Youngwon Tech Blog Posts

`youngwon-tech-blog`에 게시할 기술 글을 Markdown으로 모아 관리하는 저장소입니다.

## 구조

```text
posts/          글 초안과 발행 원고
templates/      새 글 작성용 템플릿
```

파일명이 블로그의 URL slug가 됩니다. 예를 들어 `posts/my-post.md`는 블로그에서
`/posts/my-post/`로 발행합니다.

## 글 작성

`templates/post.md`를 복사해 `posts/<slug>.md`로 만든 뒤 작성합니다. 초안은
`draft: true`, 발행할 글은 `draft: false`로 설정합니다.

지원하는 frontmatter 필드는 [SCHEMA.md](./SCHEMA.md)에 정리되어 있습니다.

## 블로그에 반영

이 저장소의 `posts/*.md`를 `youngwon-tech-blog/src/content/posts/`에 동기화한 뒤
블로그 저장소에서 다음 명령으로 검증합니다.

```sh
npm run build
```

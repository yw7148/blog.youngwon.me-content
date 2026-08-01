# Frontmatter schema

| 필드 | 필수 | 설명 |
| --- | --- | --- |
| `title` | 예 | 글 제목과 Open Graph 제목 |
| `description` | 예 | 검색, RSS, 공유 메타 설명 |
| `publishedAt` | 예 | `YYYY-MM-DD` 형식의 발행일 |
| `updatedAt` | 아니오 | `YYYY-MM-DD` 형식의 수정일 |
| `draft` | 아니오 | 초안 여부. 기본값은 `false` |
| `tags` | 아니오 | 태그 목록. 기본값은 빈 목록 |
| `series` | 아니오 | 연재 글을 묶는 이름 |
| `canonicalUrl` | 아니오 | 외부 원문이 있을 때 사용하는 canonical URL |

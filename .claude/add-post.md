# 새 포스트 추가

## 파일

`_posts/YYYY-MM-DD-slug.md` 생성. slug는 영문 소문자 + 하이픈.

## Frontmatter

```yaml
---
layout: post
title: "포스트 제목"
summary: "포스트 요약"
author: woodypark
date: 'YYYY-MM-DD HH:MM:SS +0900'
category: 카테고리명
thumbnail: /assets/img/posts/썸네일파일명
keywords: keyword1, keyword2, keyword3
permalink: /blog/slug-name/
---
```

- `category`: 단수형, 하나만. 현재: `architecture`, `go`, `redis`, `sql`
- `permalink`: `/blog/` 접두사
- frontmatter 닫는 `---` 아래에 빈 줄 + `---` 추가 후 본문 시작

## 새 카테고리가 필요하면

`categories/카테고리명.md` 파일도 함께 생성:

```markdown
---
layout: page
title: 카테고리표시명
permalink: /blog/categories/카테고리명/
---

<h5> Posts by Category : {{ page.title }} </h5>

<div class="card">
{% for post in site.categories.카테고리명 %}
 <li class="category-posts"><span>{{ post.date | date_to_string }}</span> &nbsp; <a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}
</div>
```

## 글쓰기 스타일

`.claude/writing-style.md` 참조.

## 커밋

`post: MMDD`

# 새 저자 추가

3곳에 등록 필요.

## 1. `_authors/username.md`

```yaml
---
name: 표시 이름
username: username
bio: "자기소개"
site: https://github.com/username
avatar: username.png
email: email@example.com
social:
    - title: "github"
      url: "https://github.com/username"
    - title: "linkedin"
      url: "https://www.linkedin.com/in/..."
---
```

## 2. `_data/authors.yml`에 추가

```yaml
username:
   name: 표시 이름
   username: username
   site: https://github.com/username
   avatar: username.png
   bio: "자기소개"
   email: email@example.com
   social:
      - title: "github"
        url: "https://github.com/username"
      - title: "linkedin"
        url: "https://www.linkedin.com/in/..."
```

두 곳의 정보가 일치해야 한다.

## 3. 아바타 이미지

`assets/img/authors/username.png`에 배치.

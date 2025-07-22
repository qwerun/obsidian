## [!info] Основная идея

`GOMODCACHE` — это каталог, где Go хранит скачанные версии модулей (по умолчанию `$GOPATH/pkg/mod`).

- Загружается один раз через `GOPROXY`, сохраняет `.zip`, `.mod`, `.info` файлы и проверяет их SHA-256 хэши из `go.sum`.
    
- Общий кеш ускоряет сборки, позволяет работать офлайн и экономит место за счёт дедупликации между проектами.
    
- Очищается командой `go clean -modcache`; управляется путём `go env GOMODCACHE` и флагом `-modcacherw` (делает файлы записываемыми).

---

## Краткий конспект

- **Назначение GOMODCACHE** — локальный кеш исходников модулей.
    
- **Преимущества** — ускорение сборок, оффлайн-режим, дедупликация кода.
    
- **Практическое применение** — шэринг кеша в Docker, приватные модули.
---

### Нюансы и подводные камни

[!warning]

- **Огромный размер** — сотни версий занимают десятки гигабайт.
    
- **Hash mismatch** — повреждённый кеш требует `go clean -modcache && go mod download`.
---

## Полезные ссылки

- [https://go.dev/ref/mod#mod-cache](https://go.dev/ref/mod#mod-cache)
    
- [https://pkg.go.dev/cmd/go](https://pkg.go.dev/cmd/go)
    
- [https://blog.go.dev/module-mirror-security](https://blog.go.dev/module-mirror-security)
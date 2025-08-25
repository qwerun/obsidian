> [!info] **Основная идея**  
> JWT (JSON Web Token) — это компактный и безопасный способ представления утверждений (claims) между двумя сторонами. Он широко применяется для авторизации и аутентификации в веб-приложениях, особенно в распределённых и микросервисных архитектурах.


TODO Разница между авторизацией и аутенфикацией
TODO Для получения двух токенов используется одна ручка, а вторая для рефреша
## Краткий конспект

- JWT — это формат токена, основанный на JSON и стандартизованный в RFC 7519.
- Состоит из трёх частей: Header, Payload и Signature.
- Используется для авторизации, OAuth 2.0, SSO и передачи клеймов.
- Поддерживает симметричную и асимметричную подпись (HS256, RS256 и др.).
- В Go можно легко генерировать и валидировать JWT через библиотеку `golang-jwt/jwt/v5`.
- Требует осторожности: токены нельзя отозвать, нужно правильно обрабатывать `exp`, `iat`, `kid`.
- Не шифруется по умолчанию — только подписывается.

---

## Что такое JWT

JWT (JSON Web Token) — это открытый стандарт для безопасной передачи информации в виде JSON-объекта. Этот формат определён в [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519?utm_source=chatgpt.com).

Особенности:
- Компактен (base64url).
- Самодостаточен: содержит все данные для валидации.
- Подписан (а иногда и зашифрован).
- Работает на любом языке.

---

## Зачем нужен

JWT активно используется для:

- **Авторизации**: проверка, имеет ли пользователь доступ к ресурсу.
- **OAuth 2.0**: как access token и ID token.
- **SSO (Single Sign-On)**: одна авторизация на разные системы.
- **Микросервисы**: безопасный обмен клеймами между сервисами.

---

## Из чего состоит

JWT состоит из трёх частей, разделённых точками:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0IiwibmFtZSI6IkFsaWNlIiwiaWF0IjoxNjkxNjQ1MjAwfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

1. **Header** — описывает алгоритм и тип токена:

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

2. **Payload** — полезные данные (клеймы):

```json
{
  "sub": "1234",
  "name": "Alice",
  "iat": 1691645200
}
```

3. **Signature** — подпись токена, используется для валидации:

```text
HMACSHA256(
  base64url(header) + "." + base64url(payload),
  secret
)
```

---

## Алгоритмы подписи и шифрования

JWT поддерживает:

| Алгоритм | Тип          | Пример использования |
|----------|--------------|----------------------|
| HS256    | Симметричный | Shared secret        |
| RS256    | Асимметричный| public/private ключи |
| ES256    | ЭЦП (ECDSA)  | компактные подписи   |

> [!tip]
> RS256 предпочтительнее для production: позволяет отделить подпись и валидацию между сервисами.

---

## Практический пример на Go

> Используем библиотеку [`github.com/golang-jwt/jwt/v5`](https://github.com/golang-jwt/jwt?utm_source=chatgpt.com)

### Генерация JWT

```go
package main

import (
	"time"
	"log"
	"github.com/golang-jwt/jwt/v5"
)

var secret = []byte("my-secret-key")

func generateToken() string {
	claims := jwt.MapClaims{
		"sub": "1234",
		"name": "Alice",
		"exp": time.Now().Add(time.Hour * 1).Unix(),
	}

	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	signed, err := token.SignedString(secret)
	if err != nil {
		log.Fatal(err)
	}
	return signed
}
```

### Валидация JWT

```go
func validateToken(tokenStr string) (*jwt.Token, error) {
	return jwt.Parse(tokenStr, func(token *jwt.Token) (interface{}, error) {
		if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, jwt.ErrSignatureInvalid
		}
		return secret, nil
	})
}
```

> [!example]
> Если `exp` истекло, `Parse()` вернёт ошибку. Проверяйте `token.Valid`.

---

> [!warning] **Ошибки**  
> - ❌ Использование `none` в алгоритме — отключает подпись и создаёт уязвимость.  
> - ❌ Отсутствие `exp` и `iat` может привести к бессрочным токенам.  
> - ❌ Не валидируйте токены только по `sub` — всегда проверяйте подпись.  
> - ❌ Нельзя «отозвать» токен — они остаются валидными до истечения.

---

> [!tip] **Рекомендации**  
> - Устанавливайте `exp` и `iat` на каждый токен.  
> - При использовании RS256 — храните приватный ключ безопасно.  
> - Используйте `kid` для управления несколькими ключами.  
> - Храните токены **в cookie с HttpOnly+Secure**, если это frontend.  
> - Не храните access token в localStorage — XSS легко доберётся.

---

> [!faq] **Часто задаваемые вопросы**  
> **Q:** Можно ли в JWT хранить пароль?  
> **A:** Нет, никогда. Только безопасные клеймы без чувствительных данных.  
>  
> **Q:** JWT — это шифрование?  
> **A:** Нет. JWT по умолчанию только подписан, не зашифрован.  
>  
> **Q:** Можно ли отозвать JWT?  
> **A:** Нет. Лучше использовать короткий TTL и чёрные списки.

---

## Полезные ссылки

- [RFC 7519 — JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519?utm_source=chatgpt.com)
- [jwt.io — интерактивный парсер токенов](https://jwt.io/introduction?utm_source=chatgpt.com)
- [github.com/golang-jwt/jwt](https://github.com/golang-jwt/jwt?utm_source=chatgpt.com)

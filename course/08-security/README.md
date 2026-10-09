# 08 — Security

**Prerequisites блока:** [04 — HTTP, сети и API design](../04-http-networking-api/) (cookies, CORS, TLS)
**Следующий блок:** [09 — Redis и performance](../09-redis-performance/)
**Уроков:** 6 — 1 🟢 · 3 🟡 · 2 🔴

## Цель блока

Разбирать безопасность по схеме «угроза → механизм атаки → защита → ограничения защиты». Понимать аутентификацию и авторизацию как протоколы с конкретными гарантиями, а не как набор библиотек. Разделы, которые зависят от редакций стандартов (OWASP Top 10, OAuth 2.1), сверяются с первоисточником.

## Уроки

| # | Тема | Уровень | Ключевой вопрос урока | Текст |
|---|---|---|---|---|
| 8.1 | Сессии, cookies, пароли | 🟢 | как хранить пароли и чем сессия отличается от токена | — |
| 8.2 | JWT, access и refresh токены | 🟡 | что гарантирует подпись JWT и как отозвать токен | — |
| 8.3 | OAuth 2.0 и OpenID Connect | 🔴 | какой flow выбрать и чем ID token отличается от access token | — |
| 8.4 | Авторизация | 🟡 | как проверить, что пользователь имеет право именно на этот объект | — |
| 8.5 | OWASP и типовые уязвимости | 🟡 | как устроена атака и почему защита работает | — |
| 8.6 | Операционная безопасность | 🔴 | как защитить сервис от перебора, утечки секретов и зависимостей | — |

## Состав уроков

**8.1 Сессии, cookies, пароли.** Stateful сессии против stateless токенов · атрибуты cookie: `HttpOnly`, `Secure`, `SameSite`, `Domain`, `Path`, префиксы `__Host-` · хеширование паролей: почему не SHA-256, bcrypt против scrypt против Argon2id, соль и pepper, параметры стоимости · MFA и TOTP.

**8.2 JWT, access и refresh токены.** Структура JWS, подпись против шифрования (JWE) · HS256 против RS256 против EdDSA · обязательная валидация: `alg`, `exp`, `aud`, `iss` · атаки: `alg: none`, algorithm confusion · отзыв токенов и denylist · пара access/refresh, rotation и reuse detection · где хранить токены в браузере.

**8.3 OAuth 2.0 и OpenID Connect.** Роли и термины · Authorization Code + PKCE, Client Credentials, Device Authorization · почему Implicit и Password grant устарели · статус OAuth 2.1 · OIDC: ID token, UserInfo, discovery · access token как opaque или JWT, introspection.

**8.4 Авторизация.** RBAC, ABAC, ReBAC · policy engines (OPA, Casbin, CASL) · IDOR / BOLA и почему проверка роли не равна проверке доступа к объекту · реализация через guards и policies в NestJS · multi-tenancy и row-level security в PostgreSQL.

**8.5 OWASP и типовые уязвимости.** OWASP Top 10 (актуальная редакция сверяется с owasp.org) · SQL и NoSQL injection · XSS: stored, reflected, DOM, CSP · CSRF и роль SameSite · SSRF и metadata endpoints облаков · mass assignment · path traversal · prototype pollution · ReDoS · небезопасная десериализация.

**8.6 Операционная безопасность.** Rate limiting: fixed window, sliding window, token bucket, распределённый лимит на Redis · защита от brute force и credential stuffing · secrets management: env против vault, ротация · security headers (helmet) · supply chain: lockfile, `npm audit`, install scripts, typosquatting · PII в логах.

## Что должно быть получено

- Проектирую аутентификацию для SPA и мобильного клиента и обосновываю, где хранится каждый токен.
- Объясняю, почему JWT нельзя «просто отозвать», и предлагаю рабочий компромисс.
- По описанию endpoint'а называю вероятные уязвимости и защиту от каждой.

## Связи с другими блоками

- **04 HTTP:** cookies, CORS, TLS (4.1, 4.5).
- **07 NestJS:** guards и metadata (7.4, 7.5).
- **09 Redis:** распределённый rate limiter (9.4).

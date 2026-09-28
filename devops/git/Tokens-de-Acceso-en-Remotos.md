# Tokens de acceso en remotos

## ¿Qué es?

Un token de acceso es una contraseña específica para Git (y otras integraciones), generada por la plataforma que aloja el repositorio (Bitbucket, GitHub, GitLab...), que sustituye a tu contraseña de usuario normal para autenticar `push`, `pull` y `fetch` por HTTPS.

## ¿Por qué existe?

Usar la contraseña de tu cuenta directamente para Git sería un riesgo: si un token queda expuesto o hay que revocarlo, invalidar solo esa credencial es trivial; invalidar tu contraseña de cuenta te desconecta de todo lo demás. El token además se puede limitar por permisos (solo lectura, o lectura y escritura de un repositorio concreto) y por caducidad, cosa que una contraseña de cuenta no ofrece.

---

## Cómo se genera y se usa

Cada plataforma tiene su propio nombre y pantalla para esto (Bitbucket: *API tokens* en la cuenta de Atlassian; GitHub: *Personal Access Tokens* en Settings → Developer settings), pero el patrón es el mismo:

1. Generarlo desde la web de la plataforma, con los permisos mínimos que hagan falta (normalmente lectura + escritura del repositorio).
2. Copiarlo en el momento: la mayoría de plataformas solo lo muestran una vez.
3. Usarlo en la URL del remoto en vez de una contraseña:

```bash
git remote set-url origin https://<usuario-o-marcador>:<TOKEN>@dominio-del-host/<workspace>/<repositorio>.git
```

Bitbucket, por ejemplo, usa un usuario fijo `x-token-auth` en vez del nombre de usuario real:

```bash
git remote set-url origin https://x-token-auth:ATCTT3xxxx...@bitbucket.org/mi-equipo/mi-repo.git
```

Tras cambiarlo, `git fetch` es la forma más rápida de comprobar que autentica bien, sin arriesgar un `push`.

## Buenas prácticas avanzadas

- **No lo pegues en un chat, un ticket o un README.** Un token en texto plano en cualquier sitio que no sea la propia URL del remoto (guardada localmente) es un token comprometido; si ocurre, revócalo y genera uno nuevo, no lo reutilices.
- **Dale la caducidad más corta que puedas tolerar**, no la más larga que te ofrezca la plataforma. Un token que caduca en semanas limita el daño si se filtra sin que nadie se entere.
- **Un token por máquina o integración, no uno compartido.** Así, si hay que revocar el de un portátil perdido, el resto de integraciones (CI, otro equipo) siguen funcionando.

---

*En resumen: un token de acceso es una llave que puedes recortar y tirar sin tener que cambiar la cerradura entera.*

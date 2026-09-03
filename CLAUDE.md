# Jogo da Memória da Rafaela — Guia para Agentes de IA

Jogo da memória infantil (Laravel) com painel administrativo de partidas.
Este documento é a referência operacional para agentes de IA neste projeto.

> **Fonte de verdade**: o código manda. Em divergência, use `composer.json` /
> `php artisan about`. A especificação original está em
> [`docs/roteiro-jogo-rafaela.md`](docs/roteiro-jogo-rafaela.md).

---

## 🔄 Antes de começar: `git pull`

**SEMPRE** verifique atualizações remotas antes de escrever ou alterar qualquer coisa neste repositório:

```bash
git pull          # já está pré-autorizado (allow)
```

Trabalhar sobre uma base desatualizada gera conflitos. Puxe primeiro, sempre. Para só inspecionar antes: `git fetch && git status`.

---

## Stack

| Camada | Tecnologia |
|---|---|
| Backend | Laravel 11 / PHP 8.2+ |
| Frontend | Blade + Vite + CSS/JS **puro** (sem Bootstrap/Tailwind, sem libs JS) |
| Banco | MariaDB (prod) · SQLite (dev opcional) |
| Web server | Nginx + PHP-FPM (Debian 12) |
| Auth admin | Sessão simples (flag), credenciais via `.env` |

Não há autenticação de usuário final — o jogo é público e anônimo. O único
login do sistema é o do **admin**, baseado em flag de sessão.

---

## Versão e Commits

Versão em [`version.md`](version.md) (raiz), lida via `config('app.version')`
(primeiro semver do arquivo). Padrão `X.Y.Z`:

- **Z** sobe a cada entrega: criar tela, criar tabela, mudar layout, renomear
  label/rota, alterar regra do jogo ou config de segurança.
- **Y / X** são manuais (mudança estrutural / release estável).

**Formato obrigatório de commit**: `X.Y.Z - description in English (US)`.
O bump do `version.md` vai em **um** commit por entrega; registre o changelog
no próprio `version.md`.

---

## Convenções de Código

### Organização

- **Controllers finos**: apenas request handling e montagem de resposta.
- **Form Requests** (`app/Http/Requests/`): toda validação de entrada.
- **Models** (`app/Models/`): `$fillable` explícito, casts, scopes.
- **Sem libs JS**: a lógica do jogo é vanilla JS em `resources/js/game.js`.

### Estrutura de views

```
resources/views/
├── layouts/
│   ├── game.blade.php       # layout do jogo (colorido, infantil)
│   └── admin.blade.php      # layout do admin (sóbrio, roxo escuro)
├── game/
│   └── index.blade.php      # telas do jogo (inicial, jogo, vitória, final)
├── admin/
│   ├── login.blade.php
│   └── dashboard.blade.php
└── errors/                  # páginas de erro genéricas (sem stacktrace)
    ├── 404.blade.php
    ├── 419.blade.php
    ├── 429.blade.php
    └── 500.blade.php
```

### Assets (Vite)

- CSS/JS ficam em `resources/css/` e `resources/js/`, carregados via
  `@vite([...])` no layout. **Não** usar `<style>`/`<script>` inline com lógica
  nas views — apenas a injeção de constantes (`window.LEVELS`, etc.) é permitida
  via `@json`.
- Build de produção: `npm run build`. Dev com HMR: `npm run dev`.

---

## Banco de Dados & Migrations

**Banco padrão de produção: MariaDB.** SQLite é permitido apenas para dev local
rápido. PostgreSQL não é usado.

### Idempotência (obrigatória)

```php
public function up(): void
{
    if (! Schema::hasTable('game_logs')) {
        Schema::create('game_logs', function (Blueprint $table) {
            $table->id();
            // ...
            $table->timestamps();
        });
    }
}

public function down(): void
{
    Schema::dropIfExists('game_logs');
}
```

### Boas práticas

- Índices em colunas usadas em `WHERE` / `ORDER BY` (ex.: `level`, `created_at`).
- Evite `ENUM` no schema — use `string` e valide no Form Request.
- Sempre `$table->timestamps()` (`created_at` = momento da partida).
- **NUNCA** `migrate:fresh` em produção. Use `migrate:rollback --step=N`.

---

## UI & Frontend

### Identidade visual

Dois contextos visuais **distintos** (não há dark mode global aqui):

| Contexto | Paleta | Tom |
|---|---|---|
| **Jogo** | rosa `#FF6B9D`, roxo `#A855F7`, fundo `#FFF0F5` | infantil, lúdico, animado |
| **Admin** | header roxo escuro `#2D1B69`, conteúdo branco | sóbrio, funcional |

- Fonte do jogo: `'Segoe UI', 'Comic Sans MS', cursive`.
- Animações do jogo: flip 3D (`transform: rotateY`), bounce do mascote, confete.
- Responsividade do tabuleiro: `min(90vw, 550px)`, células proporcionais.
- **Nenhum framework CSS.** CSS puro para controle total.

### Regras de Blade

- Output sempre escapado com `{{ }}`. `{!! !!}` é proibido com dado de usuário.
- Dados do servidor para o JS: `@json($var)` (nunca interpolação em `<script>`).
- `@csrf` em **todos** os formulários.
- `@vite([...])` no layout para CSS/JS.

---

## Registro de Partidas (fluxo central)

```
Rafaela completa um nível
   → game.js chama saveLog(data)
   → POST /api/log (JSON + header X-CSRF-TOKEN, rota com throttle)
   → StoreGameLogRequest valida
   → GameLogController grava GameLog (+ ip/user-agent do servidor)
   → resposta {"ok": true}
   → a tela de vitória aparece independentemente do resultado da API
```

O frontend **nunca** trava por falha de log — `saveLog` falha em silêncio.

---

## Segurança

Regras completas em [SECURITY_GUIDELINES.md](SECURITY_GUIDELINES.md). Resumo:

- **Admin**: senha em **hash** no `.env` (`ADMIN_PASSWORD_HASH`), comparada com
  `Hash::check`/`hash_equals`; rate-limit no login; `session()->regenerate()`
  no login e `invalidate()` + `regenerateToken()` no logout.
- **CSRF**: `@csrf` em forms; `/api/log` usa header `X-CSRF-TOKEN`.
- **Mass Assignment**: `$fillable` explícito; IP/user-agent definidos no servidor,
  **fora** do input do usuário.
- **Validação**: sempre via Form Request; nunca confiar no JS.
- **Throttle**: `/api/log` limitado; login admin limitado.
- **SQL/XSS**: Eloquent/Query Builder parametrizado; `{{ }}` no output.
- **Prod**: `APP_DEBUG=false`, `.env` fora do git, headers de segurança no Nginx.

---

## Comandos Rápidos

```bash
php artisan serve
npm run dev / npm run build
php artisan migrate / migrate:status / migrate:rollback --step=1
php artisan pint            # se instalado
php artisan about
php -l caminho/arquivo.php
php artisan optimize:clear
```

---

## DEV Files (não vão para produção)

`.env`, `.env.*`, `storage/`, `bootstrap/cache/`, `.git/`, `vendor/`,
`node_modules/`, `public/build/`, `.claude/`, `README.md`,
`CLAUDE.md`, `SECURITY_GUIDELINES.md`, `.vscode/`.

---

## Checklist Pré-Commit

- [ ] `php -l` nos arquivos PHP alterados
- [ ] `php artisan view:cache && php artisan view:clear` — valida Blade
- [ ] Jogo testado no navegador (virada, vitória, avanço de nível, log)
- [ ] Migrations com `hasTable()`/`dropIfExists()` e `down()` funcional
- [ ] `$fillable` explícito em models
- [ ] `@csrf` em todos os formulários; `/api/log` com CSRF no header
- [ ] `.env.example` atualizado se adicionou variável
- [ ] `version.md` com bump + changelog se aplicável
- [ ] `APP_DEBUG` continua `false` em PROD

---

<!-- COMMIT-RULE:repodocs -->

## Commits — you commit, and nothing is delivered until you have

> Marked echo. The single source is **[samirhvbr/repodocs](https://github.com/samirhvbr/repodocs/blob/master/docs/versioning.md#who-commits-and-when)**
> — change it there, not here. This block is regenerated.

**Committing is your job.** Not "leave the tree ready and something downstream
packages it" — you run `git commit`, and `git push`, as the last step of the work
you were asked to do. The COMMITTER skill that used to commit on an agent's
behalf is `enabled: false` in every repository of this fleet since 03/09/2026;
what is left of it is a kill-switch, not a scheduler. **If you do not commit,
nobody does.**

**Do not report a task as finished before the commit exists.** "Done",
"delivered", "concluded" mean the work is in `git log` — never that it is sitting
uncommitted where only this session can see it. The commit is the last step *of
the task*, not a follow-up for someone else. If you are about to write
"finished", commit first, then write it.

**Every commit obeys the versioning rules**, with no exception:

- Subject `X.Y.Z - short description in English (US)`, the version taken from
  `version.md` and **bumped in the same commit**.
- The `CHANGELOG.md` entry is written first — its `## X.Y.Z - description`
  heading *is* the subject.
- No Conventional Commits prefix (`feat:`, `fix:`, `chore:`) and no vague
  subject ("update", "ajuste", "wip", "changes", "several improvements").

**The bump is the one clause a repository may override — in writing.** If this
repository's own documentation says the version is stamped some other way, and says
why, follow that. Otherwise the line above applies to you. An override nobody wrote
down is not an exception. Nothing else in this block bends: the changelog entry, the
subject, the language, one subject per commit, and committing before you report done
all hold regardless.

**One subject per commit.** The subject has to describe the whole commit
honestly. The moment your description needs an "and" to be true, it is two
commits.

**Split a large delivery into blocks.** A complex task is committed as a series
of commits grouped by subject, each small enough to be described in one line and
read on its own. They may share a version — bump `version.md` in the first and
repeat the number in the rest; two commits carrying one version is expected, not
a mistake. **Splitting is the default** for anything non-trivial, because the
history is the documentation of *how* the work was done, and one commit touching
six unrelated subjects documents none of them.

**The standard you are keeping:** someone reading `git log` alone — a year from
now, without the conversation that produced the work — can say what happened,
when, why, and at which version. If your commit would fail that test, it is too
big or its subject is too vague, and both are fixed the same way.

<!-- /COMMIT-RULE -->

---

<!-- RELEASES-RULE:repodocs -->

## Releases — the `version.md` on GitHub is what the Releases show

> Marked echo. The single source is **[samirhvbr/repodocs](https://github.com/samirhvbr/repodocs/blob/master/docs/versioning.md)**
> — change it there, not here. This block is regenerated.

**The `version.md` of the default branch, on GitHub, is what the GitHub Releases
must show.** The local checkout does not enter the calculation: it can be behind,
ahead or mid-work, and none of that is published — GitHub cannot tag a commit it
does not have.

**The bump and the Release are one act.** A commit that bumps `version.md` is not
finished until that version has a tag, a published Release, and the **`Latest`
badge on it** — the same push, not "later". A badge sitting on an older release
tells whoever looks that the project is at a version it is not.

- `.github/workflows/release.yml` does it on any push that touches `version.md`.
- `./tools/release.sh` does it by hand. It is **idempotent and self-healing**:
  it publishes whatever is missing and moves a drifted badge back. Running it is
  always safe, so it is both the check and the fix.

A PR publishes nothing while it is a PR. The moment it merges, the push moves
`version.md` on the default branch and the Release becomes that version.

Tag and Release title are the **bare version — no `v` prefix**.

## Language — English (US), everywhere in the repository

**Everything that lives in this repository, or in GitHub's interface around it,
is written in English (US)**: documents, **commit messages**, pull request titles
and bodies, issues, code comments, changelog entries, release notes.

Commit format: `X.Y.Z - short description in English`. The version comes from
`version.md` and is bumped in the same commit. Conventional Commits prefixes
(`feat:`, `fix:`, `chore:`) and vague one-word messages are forbidden.

**Exactly one carve-out:** end-user-facing strings — UI text, transactional
email, product copy. That is product i18n for a Brazilian audience, not
repository content.

History is not rewritten: Portuguese messages already in the log stay as they
are.

<!-- /RELEASES-RULE -->

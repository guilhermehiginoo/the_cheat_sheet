---
title: Laravel
description: Referência prática para criar um projeto Laravel, comandos Artisan do dia a dia, estrutura MVC e os erros que mais pegam iniciantes.
tags:
  - php
  - web
  - backend
  - mvc
---

## Criando um Projeto

```bash
# Criar um projeto novo (o Composer é o único requisito obrigatório)
composer create-project laravel/laravel meu-app
cd meu-app

# Ou, com o instalador do Laravel
composer global require laravel/installer
laravel new meu-app
```

Todo clone de um projeto existente precisa dos mesmos quatro passos:

```bash
composer install          # instala as dependências PHP (vendor/ está no gitignore)
cp .env.example .env      # cria o arquivo de ambiente local
php artisan key:generate  # escreve a APP_KEY no .env
php artisan migrate       # cria as tabelas no banco
```

Depois, suba os dois servidores que você precisa em desenvolvimento:

```bash
composer run dev    # servidor + worker de fila + Vite, tudo num terminal só
```

Ou suba as peças na mão, em terminais separados:

```bash
php artisan serve   # aplicação PHP em http://127.0.0.1:8000
npm install         # só na primeira vez
npm run dev         # servidor Vite para CSS/JS
```

- O `.env` **não** é commitado — por isso um projeto clonado não tem nenhum.
- A `APP_KEY` é usada para criptografar sessões e cookies. Sem ela, nada sobe.
- `php artisan serve` é só para desenvolvimento; produção roda atrás de Nginx/Apache ou FrankenPHP.
- Um projeto novo já vem com SQLite: o `laravel new` cria o
  `database/database.sqlite` e roda as migrations sozinho.

## Starter Kits

O `laravel new` pergunta qual starter kit você quer. Um starter kit é uma
aplicação Laravel normal que já vem com autenticação — login, cadastro,
recuperação de senha, verificação de e-mail, dois fatores — mais um frontend
montado. Todo esse código fica **dentro do seu projeto**, então ele é seu e você
edita à vontade; não existe atualização de kit depois.

- **React** — Inertia, React 19, TypeScript, Tailwind, [shadcn/ui](https://ui.shadcn.com). Frontend em `resources/js/`.
- **Vue** — Inertia, Vue 3 Composition API, TypeScript, Tailwind, shadcn-vue. Frontend em `resources/js/`.
- **Svelte** — Inertia, Svelte 5, TypeScript, Tailwind, shadcn-svelte. Frontend em `resources/js/`.
- **Livewire** — Livewire + Tailwind + [Flux UI](https://fluxui.dev), UI reativa escrita em PHP, sem framework JS. Frontend em `resources/views/`.
- **Nenhum** — o esqueleto puro. Escolha esse para aprender o framework em si, ou quando for construir uma API.

Os três kits com Inertia deixam você escrever componentes React/Vue/Svelte
mantendo o roteamento e os controllers no servidor — sem precisar de uma camada
de API separada. O Livewire é a escolha se você prefere ficar no Blade e no PHP.

```bash
laravel new meu-app                            # pergunta qual kit
laravel new meu-app --using=vendor/starter-kit # kit da comunidade, via Packagist
```

Cada kit ainda oferece duas opções no mesmo prompt:

- **Teams** — usuários pertencem a times, com telas de gerenciamento e rotas escopadas por time (`/{current_team}/dashboard`).
- **WorkOS AuthKit** — troca a autenticação nativa por um provedor hospedado, com login social, passkeys, magic link e SSO. Precisa de `WORKOS_CLIENT_ID`, `WORKOS_API_KEY` e `WORKOS_REDIRECT_URL` no `.env`.

A autenticação de todos os kits é feita pelo **Laravel Fortify**, que registra
as rotas por você. Ligue e desligue funcionalidades em `config/fortify.php`:

```php
use Laravel\Fortify\Features;

'features' => [
    Features::registration(),          // remova esta linha para fechar o cadastro público
    Features::resetPasswords(),
    Features::emailVerification(),
    Features::twoFactorAuthentication(['confirm' => true]),
],
```

- A lógica de cadastro fica em `app/Actions/Fortify/CreateNewUser.php` — é ali
  que você adiciona campos extras ao formulário de registro.
- Nos kits com Inertia, desligar uma funcionalidade exige também remover as
  referências às rotas dela nos componentes, senão o build do frontend quebra.
- **Breeze** e **Jetstream** eram a geração anterior de starter kits. Muito
  tutorial ainda usa os dois, mas a documentação atual não os lista mais — para
  um projeto novo, use os kits acima.
- Nota de versão: essa geração de kits chegou no **Laravel 12**; o kit Svelte e
  a autenticação via Fortify vieram no **Laravel 13**. Tutoriais antigos com
  `laravel new --breeze` são anteriores a tudo isso.

## Ambiente Local: Herd e Sail

`php artisan serve` funciona, mas parte do princípio de que você já tem PHP,
Composer e um banco instalados e configurados na máquina. Duas ferramentas
oficiais eliminam esse trabalho de setup.

### Herd (macOS / Windows)

Um app nativo que já vem com PHP, Composer e nginx. Qualquer projeto dentro do
diretório "parkeado" é servido automaticamente em `<pasta>.test` — sem comando
`serve`, sem número de porta.

```bash
herd park ~/Sites          # serve todos os projetos dessa pasta
herd link meu-app          # serve um projeto específico como meu-app.test
herd php -v                # o binário PHP gerenciado pelo Herd
herd open                  # abre o projeto atual no navegador
```

- Troque a versão do PHP por projeto pela interface ou com `herd use php@8.3`.
- O Herd já traz os binários `php`, `composer`, `laravel`, `node`, `npm` e `nvm` —
  se o `node` não aparece no `PATH` fora do seu shell, é por isso.
- A versão paga adiciona bancos de dados, captura de e-mails e visualização de
  logs; a gratuita cobre PHP + nginx.

### Sail (Docker)

Uma casca fina sobre o Docker Compose que entrega PHP, MySQL, Redis e outros
como containers, sem instalar nada na máquina.

```bash
php artisan sail:install     # adiciona o Sail a um projeto existente
./vendor/bin/sail up -d      # sobe os containers
./vendor/bin/sail down       # derruba os containers
```

O detalhe para se acostumar: **sua aplicação roda dentro de um container, então
os comandos precisam rodar lá também.** Prefixe tudo com `sail`:

```bash
php artisan serve   →  sail artisan serve
php artisan migrate →  sail artisan migrate
composer install    →  sail composer install
npm run dev         →  sail npm run dev
php artisan tinker  →  sail artisan tinker
```

Deixe curto com um alias no shell:

```bash
alias sail='[ -f sail ] && sh sail || sh vendor/bin/sail'
```

- Rodar `php artisan migrate` puro na máquina enquanto usa Sail é um erro
  comum — isso usa o PHP do host, que não enxerga o banco do container.
- No `.env`, `DB_HOST` é o **nome do serviço** (`mysql`), não `127.0.0.1`.

Todo o resto deste cheat sheet é igual nos dois casos. Herd e Sail mudam *onde*
os comandos rodam, não o que eles fazem.

## Arquitetura: MVC

O Laravel segue o padrão **MVC** — Model, View, Controller. A requisição
percorre a aplicação em uma direção:

```text
Requisição → routes/web.php → Controller → Model (Eloquent) → Banco
                                  ↓
                          View (Blade) → Resposta
```

| Camada         | Fica em                     | Responsabilidade                              |
| -------------- | --------------------------- | --------------------------------------------- |
| **Rota**       | `routes/web.php`, `api.php` | Liga uma URL + verbo HTTP a um código         |
| **Controller** | `app/Http/Controllers/`     | Recebe a requisição, orquestra, responde      |
| **Model**      | `app/Models/`               | Representa uma tabela, guarda queries e relações |
| **View**       | `resources/views/`          | Templates Blade que renderizam o HTML         |

Outras pastas que vale conhecer:

```text
app/Http/Middleware/     # código que roda antes/depois de cada requisição
bootstrap/app.php        # bootstrap da app: rotas, middleware, exceções
config/                  # arquivos de configuração, que leem do .env
database/migrations/     # mudanças de schema versionadas
database/seeders/        # dados iniciais/de exemplo
public/                  # raiz web, ponto de entrada (index.php) e assets buildados
resources/js|css/        # código-fonte do frontend, compilado pelo Vite
routes/                  # definição das rotas
storage/                 # logs, cache, views compiladas, arquivos enviados
```

- Mantenha os controllers magros: valide, delegue, responda. Regra de negócio
  vive nos models ou em classes dedicadas, não no controller.

## Comandos Artisan do Dia a Dia

`artisan` é a CLI. `php artisan list` mostra tudo; `php artisan help <comando>` explica um.

```bash
php artisan serve                   # sobe o servidor de desenvolvimento
php artisan serve --port=8001       # em outra porta
php artisan tinker                  # REPL interativo com a aplicação carregada
php artisan route:list              # todas as rotas registradas
php artisan route:list --path=user  # filtrado
php artisan about                   # ambiente, versões, estado dos caches
```

### Geradores

```bash
php artisan make:model Post -mcr    # model + migration + controller + rotas resource
php artisan make:model Post -m      # model + migration
php artisan make:controller PostController --resource
php artisan make:migration create_posts_table
php artisan make:request StorePostRequest
php artisan make:middleware EnsureUserIsAdmin
php artisan make:seeder PostSeeder
php artisan make:factory PostFactory
php artisan make:command SyncPosts
```

### Banco de Dados

```bash
php artisan migrate                 # roda as migrations pendentes
php artisan migrate:status          # o que rodou e o que não rodou
php artisan migrate:rollback        # desfaz o último lote
php artisan migrate:fresh --seed    # dropa tudo, re-migra e roda os seeders
php artisan db:seed                 # roda só os seeders
```

- `migrate:fresh` **apaga todos os dados**. Nunca rode em produção.

### Cache e Manutenção

```bash
php artisan optimize:clear   # limpa cache de config, rotas, views e aplicação de uma vez
php artisan config:clear
php artisan cache:clear
php artisan view:clear
php artisan storage:link     # cria o link public/storage → storage/app/public
php artisan queue:work       # processa os jobs da fila
php artisan down             # modo manutenção
php artisan up
```

- Em produção você faz o contrário — *cacheia*: `php artisan config:cache route:cache view:cache`.
- Em desenvolvimento, mantenha limpo — config cacheada ignora mudanças no `.env`.

## Rotas

```php
// routes/web.php
use App\Http\Controllers\PostController;

Route::get('/', fn () => view('welcome'));

Route::get('/posts', [PostController::class, 'index'])->name('posts.index');
Route::post('/posts', [PostController::class, 'store']);
Route::get('/posts/{post}', [PostController::class, 'show']);
Route::put('/posts/{post}', [PostController::class, 'update']);
Route::delete('/posts/{post}', [PostController::class, 'destroy']);

// As sete rotas de CRUD de uma vez
Route::resource('posts', PostController::class);

// Agrupamento
Route::middleware('auth')->prefix('admin')->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
});
```

- Tipar o model no método (`show(Post $post)`) ativa o **route model binding** —
  o Laravel busca o registro e devolve 404 automaticamente se não existir.
- Use rotas nomeadas e `route('posts.index')` em vez de URLs escritas na mão.
- `routes/web.php` tem sessão e CSRF; `routes/api.php` é stateless.

## Controllers

```php
namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\Request;

class PostController extends Controller
{
    public function index()
    {
        return view('posts.index', ['posts' => Post::latest()->paginate(15)]);
    }

    public function show(Post $post)   // route model binding
    {
        return view('posts.show', compact('post'));
    }

    public function store(Request $request)
    {
        $data = $request->validate([
            'title' => ['required', 'string', 'max:255'],
            'body'  => ['required'],
        ]);

        $post = Post::create($data);

        return redirect()->route('posts.show', $post)->with('status', 'Post criado!');
    }
}
```

## Eloquent

```php
// Leitura
Post::all();
Post::find(1);
Post::findOrFail(1);                       // lança 404 se não existir
Post::where('published', true)->get();
Post::where('views', '>', 100)->orderBy('views', 'desc')->take(5)->get();
Post::first();
Post::count();

// Escrita
$post = Post::create(['title' => 'Oi', 'body' => '...']);   // precisa de $fillable
$post->update(['title' => 'Atualizado']);
$post->delete();

// Relações
class Post extends Model
{
    protected $fillable = ['title', 'body'];

    public function user()     { return $this->belongsTo(User::class); }
    public function comments() { return $this->hasMany(Comment::class); }
}

$post->comments;                       // carregamento sob demanda
Post::with('comments')->get();         // eager loading — evita o problema N+1
```

- `create()` e `update()` só atribuem campos listados em `$fillable`. Uma coluna
  que "sumiu" silenciosamente quase sempre é um `$fillable` incompleto.
- O model `Post` mapeia para a tabela `posts` por convenção (plural, snake_case).

## Migrations

```php
// database/migrations/xxxx_create_posts_table.php
public function up(): void
{
    Schema::create('posts', function (Blueprint $table) {
        $table->id();
        $table->foreignId('user_id')->constrained()->cascadeOnDelete();
        $table->string('title');
        $table->text('body');
        $table->boolean('published')->default(false);
        $table->timestamps();
    });
}

public function down(): void
{
    Schema::dropIfExists('posts');
}
```

- Nunca edite uma migration que já rodou em produção — crie uma nova.
- Localmente, `php artisan migrate:fresh` é o jeito rápido de refazer tudo.

## Templates Blade

```blade
{{-- resources/views/posts/index.blade.php --}}
@extends('layouts.app')

@section('content')
    <h1>{{ $title }}</h1>          {{-- saída escapada --}}
    {!! $htmlConfiavel !!}         {{-- saída crua — só para conteúdo confiável --}}

    @if ($posts->isEmpty())
        <p>Nenhum post ainda.</p>
    @else
        @foreach ($posts as $post)
            <a href="{{ route('posts.show', $post) }}">{{ $post->title }}</a>
        @endforeach
    @endif

    @auth
        <p>Olá, {{ auth()->user()->name }}</p>
    @endauth

    <form method="POST" action="{{ route('posts.store') }}">
        @csrf
        <input name="title" value="{{ old('title') }}">
        @error('title') <span>{{ $message }}</span> @enderror
        <button>Salvar</button>
    </form>
@endsection
```

- Todo formulário não-GET precisa de `@csrf`, senão a requisição é rejeitada com 419.
- Use `@method('PUT')` / `@method('DELETE')` para verbos que o HTML não envia.

## Validação

```php
$request->validate([
    'email'    => ['required', 'email', 'unique:users,email'],
    'password' => ['required', 'confirmed', 'min:8'],
    'age'      => ['nullable', 'integer', 'between:18,120'],
]);
```

Em caso de falha o Laravel redireciona de volta automaticamente com os erros e
os dados antigos — sem `try/catch`. Para conjuntos maiores de regras, use um
Form Request:

```bash
php artisan make:request StorePostRequest
```

## Vite e Assets

```blade
{{-- No <head> do seu layout --}}
@vite(['resources/css/app.css', 'resources/js/app.js'])
```

```bash
npm run dev     # desenvolvimento: hot reload, precisa ficar rodando
npm run build   # produção: gera public/build/ + manifest.json
```

- Em desenvolvimento, o `npm run dev` roda **junto** com o `php artisan serve` —
  dois terminais, os dois rodando. O `composer run dev` sobe os dois (mais o
  worker de fila) de uma vez.
- Em produção (ou sempre que o servidor de dev não estiver de pé), você precisa
  ter rodado `npm run build` pelo menos uma vez.

## Erros Comuns

| Erro                                                    | Causa                                          | Correção                                                |
| ------------------------------------------------------- | ---------------------------------------------- | ------------------------------------------------------- |
| `No application encryption key has been specified`      | `APP_KEY` vazia ou `.env` faltando             | `cp .env.example .env && php artisan key:generate`      |
| Página em branco / **500** genérico                     | O erro real está escondido                     | Ponha `APP_DEBUG=true` no `.env`, ou leia `storage/logs/laravel.log` |
| `SQLSTATE[HY000] [2002] Connection refused`             | Banco não está rodando, ou host/porta errados  | Suba o banco; confira `DB_HOST`/`DB_PORT` no `.env`     |
| `SQLSTATE[HY000] [1045] Access denied for user`         | Credenciais do banco erradas                   | Ajuste `DB_USERNAME`/`DB_PASSWORD` e rode `php artisan config:clear` |
| `SQLSTATE[42S02] Base table or view not found`          | As migrations nunca rodaram                    | `php artisan migrate`                                   |
| `Unable to open database file` (SQLite)                 | O arquivo não existe                           | `touch database/database.sqlite`                        |
| `Vite manifest not found at public/build/manifest.json` | Assets nunca foram buildados e o dev server está desligado | `npm run build`, ou rode `npm run dev`      |
| Página carrega mas sem CSS/JS                           | O `npm run dev` parou                          | Reinicie, ou rode `npm run build`                       |
| **419** Page Expired                                    | Falta `@csrf` ou a sessão expirou              | Adicione `@csrf` no formulário; limpe os cookies        |
| **404** numa rota que você acabou de criar              | Rotas cacheadas, ou verbo errado               | `php artisan route:clear`; confira `php artisan route:list` |
| `Target class [FooController] does not exist`           | Namespace errado ou typo na rota               | Importe o controller e use `[FooController::class, 'metodo']` |
| `Class "App\Models\Post" not found`                     | Autoloader desatualizado                       | `composer dump-autoload`                                |
| Mudança no `.env` não surte efeito                      | Config está cacheada                           | `php artisan config:clear` (ou `optimize:clear`)        |
| `The stream or file .../laravel.log could not be opened` | Sem permissão de escrita                      | `chmod -R 775 storage bootstrap/cache`                  |
| Imagens enviadas dão 404 em `/storage/...`              | Falta o symlink                                | `php artisan storage:link`                              |
| `Add [title] to fillable property`                      | Mass assignment bloqueado                      | Adicione a coluna ao `$fillable` do model               |
| `Attempt to read property on null`                      | Uma query retornou `null`                      | Use `findOrFail()`, ou proteja com `@if`/`?->`          |
| `composer install` falha em `ext-...`                   | Extensão do PHP faltando                       | Instale (ex.: `php-mbstring`, `php-xml`, `php-curl`)    |
| Usando Sail: `Connection refused` em `127.0.0.1`        | Rodou o comando no host, ou `DB_HOST` errado   | Prefixe com `sail`; use `DB_HOST=mysql` no `.env`       |

### A Ordem de Depuração

Quando algo quebra e você não sabe o motivo, nesta ordem:

```bash
tail -f storage/logs/laravel.log   # 1. leia o erro de verdade
php artisan optimize:clear         # 2. limpe todos os caches
php artisan migrate:status         # 3. o schema está atualizado?
php artisan route:list             # 4. a rota existe como você imagina?
php artisan about                  # 5. ambiente, conexão com o banco, caches
```

A maioria dos 500 de iniciante é uma de três coisas: falta de `APP_KEY`, falta
de conexão com o banco, ou migrations que nunca rodaram.

## References

- [Documentação do Laravel](https://laravel.com/docs)
- [Artisan Console](https://laravel.com/docs/artisan)
- [Eloquent ORM](https://laravel.com/docs/eloquent)
- [Templates Blade](https://laravel.com/docs/blade)
- [Empacotamento de Assets (Vite)](https://laravel.com/docs/vite)
- [Starter Kits](https://laravel.com/docs/starter-kits)
- [Laravel Fortify](https://laravel.com/docs/fortify)
- [Laravel Herd](https://herd.laravel.com/docs)
- [Laravel Sail](https://laravel.com/docs/sail)
- [Laravel Bootcamp](https://bootcamp.laravel.com)

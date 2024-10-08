+++
title = "Como eu implementei comentários no meu blog usando o Mastodon"
date = "2024-10-08T17:30:00-03:00"
draft = false
toc = true
comments = true
fediverse = 113273777798787692
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = ["pt","tech","hugo","mastodon"]
+++

## Introdução

Hoje mudei (novamente) o sistema de comentários do meu site. Antes usava o Giscus - que é *muito* bacana e funciona através de discussões no GitHub - e agora estou usando o Fediverso! Fiz isso com base [neste post](https://danielpecos.com/2022/12/25/mastodon-as-comment-system-for-your-static-blog/) aqui, do [Daniel Pecos Martínez](https://fosstodon.org/@dpecos).

Nada contra o Giscus, pelo contrário: Ele é ótimo para quem quer fugir do Disqus (que [não é muito legal para privacidade](https://nrkbeta-no.translate.goog/2019/12/18/disqus-delte-persondata-om-titalls-millioner-internettbrukere-uten-at-nettsidene-visste-om-det/?_x_tr_sl=auto&_x_tr_tl=en&_x_tr_hl=pt-BR), e coloca anúncios no seu site), e não exige que você tenha um servidor funcionando para operar os comentários, tal qual eu tinha com o [Isso](https://isso-comments.de/).

Recentemente eu estive em uma onda muito boa de criatividade (mais sobre isso em um post futuro) e resolvi começar a usar o Mastodon para escrever alguns pensamentos do dia a dia. A qualidade das conversas lá é **muito** boa. Então decidi resgatar uma ideia antiga: A de usar posts no Mastodon para um sistema de comentários. Infelizmente não manjo o suficiente de programação para implementar do zero. Mas se tem uma coisa que eu sou bom é adaptar esse tipo de implementação para minhas necessidades. Foi assim que encontrei o post linkado acima. E fiz o seguinte aqui no meu site (lembrando que o código completo do site [está no meu GitHub](https://github.com/patrickcamillo/patrickcamillo.com)):

## Parte técnica

### Conteúdo dos arquivos

**No arquivo `config.toml`**

```
[params.comment]
  [params.comment.fediverse]
    host = "ursal.zone"
    user = "patrickcamillo"
```

É só colocar a instância e o nome de usuário. A princípio será para o seu site pessoal, então será sempre fixo. Mas dá pra adaptar em outro lugar caso esteja lidando com um site multi-autor ou algo assim. INCLUSIVE, bela ideia.

**No arquivo (novo) `themes/just-another-bear/layouts/partials/comments.html`**

```html
<hr>

<h1>{{T `comments`}}</h1>

<noscript>{{T `noscriptWarning`}}</noscript>

<p style="font-size: large; ">
    {{T `commentsDesc1`}}
    <a class="link"
        href="https://{{ .Site.Params.comment.fediverse.host }}/@{{ .Site.Params.comment.fediverse.user }}/{{ .Params.fediverse }}">
        {{T `commentsDesc2`}}
    </a>
    {{T `commentsDesc3`}}
</p>

<p id="mastodon-comments-list"></p>

<script src="https://cdnjs.cloudflare.com/ajax/libs/dompurify/2.4.1/purify.min.js" integrity="sha512-uHOKtSfJWScGmyyFr2O2+efpDx2nhwHU2v7MVeptzZoiC7bdF6Ny/CmZhN2AwIK1oCFiVQQ5DA/L9FSzyPNu6Q==" crossorigin="anonymous" referrerpolicy="no-referrer"></script>
<script type="text/javascript">
  var host = '{{ .Site.Params.comment.fediverse.host }}';
  var user = '{{ .Site.Params.comment.fediverse.user }}';
  var id = '{{ .Params.fediverse }}'

  function escapeHtml(unsafe) {
    return unsafe
      .replace(/&/g, "&amp;")
      .replace(/</g, "&lt;")
      .replace(/>/g, "&gt;")
      .replace(/"/g, "&quot;")
      .replace(/'/g, "&#039;");
  }

  var commentsLoaded = false;

  function toot_active(toot, what) {
    var count = toot[what+'_count'];
    return count > 0 ? 'active' : '';
  }

  function toot_count(toot, what) {
    var count = toot[what+'_count'];
    return count > 0 ? count : '';
  }

  function user_account(account) {
    var result =`@${account.acct}`;
    if (account.acct.indexOf('@') === -1) {
      var domain = new URL(account.url)
      result += `@${domain.hostname}`
    }
    return result;
  }

  function render_toots(toots, in_reply_to, depth) {
    var tootsToRender = toots
      .filter(toot => toot.in_reply_to_id === in_reply_to)
      .sort((a, b) => a.created_at.localeCompare(b.created_at));
    tootsToRender.forEach(toot => render_toot(toots, toot, depth));
  }

  function render_toot(toots, toot, depth) {
    toot.account.display_name = escapeHtml(toot.account.display_name);
    toot.account.emojis.forEach(emoji => {
      toot.account.display_name = toot.account.display_name.replace(`:${emoji.shortcode}:`, `<img src="${escapeHtml(emoji.static_url)}" alt="Emoji ${emoji.shortcode}" height="20" width="20" />`);
    });
    mastodonComment =
      `<div class="mastodon-comment" style="margin-left: calc(var(--mastodon-comment-indent) * ${depth})">
        <div class="author">
          <div class="avatar">
            <img src="${escapeHtml(toot.account.avatar_static)}" height=60 width=60 alt="">
          </div>
          <div class="details">
            <a class="name" href="${toot.account.url}" rel="nofollow">${toot.account.display_name}</a>
            <a class="user" href="${toot.account.url}" rel="nofollow">${user_account(toot.account)}</a>
          </div>
          <a class="date" href="${toot.url}" rel="nofollow">${toot.created_at.substr(0, 10)} ${toot.created_at.substr(11, 8)}</a>
        </div>
        <div class="content">${toot.content}</div>
        <div class="attachments">
          ${toot.media_attachments.map(attachment => {
            if (attachment.type === 'image') {
              return `<a href="${attachment.url}" rel="nofollow"><img src="${attachment.preview_url}" alt="${attachment.description}" /></a>`;
            } else if (attachment.type === 'video') {
              return `<video controls><source src="${attachment.url}" type="${attachment.mime_type}"></video>`;
            } else if (attachment.type === 'gifv') {
              return `<video autoplay loop muted playsinline><source src="${attachment.url}" type="${attachment.mime_type}"></video>`;
            } else if (attachment.type === 'audio') {
              return `<audio controls><source src="${attachment.url}" type="${attachment.mime_type}"></audio>`;
            } else {
              return `<a href="${attachment.url}" rel="nofollow">${attachment.type}</a>`;
            }
          }).join('')}
        </div>
        <div class="status">
          <div class="replies ${toot_active(toot, 'replies')}">
            <a href="${toot.url}" rel="nofollow"><i class="fa fa-reply fa-fw"></i>${toot_count(toot, 'replies')}</a>
          </div>
          <div class="reblogs ${toot_active(toot, 'reblogs')}">
            <a href="${toot.url}" rel="nofollow"><i class="fa fa-retweet fa-fw"></i>${toot_count(toot, 'reblogs')}</a>
          </div>
          <div class="favourites ${toot_active(toot, 'favourites')}">
            <a href="${toot.url}" rel="nofollow"><i class="fa fa-star fa-fw"></i>${toot_count(toot, 'favourites')}</a>
          </div>
        </div>
      </div>`;
    document.getElementById('mastodon-comments-list').appendChild(DOMPurify.sanitize(mastodonComment, {'RETURN_DOM_FRAGMENT': true}));

    render_toots(toots, toot.id, depth + 1)
  }

  function loadComments() {
    if (commentsLoaded) return;

    document.getElementById("mastodon-comments-list").innerHTML = "{{T `commentLoad`}}";

    fetch('https://' + host + '/api/v1/statuses/' + id + '/context')
      .then(function(response) {
        return response.json();
      })
      .then(function(data) {
        if(data['descendants'] && Array.isArray(data['descendants']) && data['descendants'].length > 0) {
            document.getElementById('mastodon-comments-list').innerHTML = "";
            render_toots(data['descendants'], id, 0)
        } else {
          document.getElementById('mastodon-comments-list').innerHTML = "<p>{{T `noComments`}}</p>";
        }

        commentsLoaded = true;
      });
  }

  function respondToVisibility(element, callback) {
    var options = {
      root: null,
    };

    var observer = new IntersectionObserver((entries, observer) => {
      entries.forEach(entry => {
        if (entry.intersectionRatio > 0) {
          callback();
        }
      });
    }, options);

    observer.observe(element);
  }

  var comments = document.getElementById("mastodon-comments-list");
  respondToVisibility(comments, loadComments);
</script>
```

Algumas dessas linhas estão com um código referente ao i18n para possibilitar a tradução do meu site para inglês. Se você não usa internacionalização, é só substituir as chaves abaixo no código acima.

**No arquivo em português `themes/just-another-bear/i18n/pt-br.toml`**

```toml
[commentsDesc1]
other = "Você pode usar sua conta no Fediverso (por exemplo Mastodon, e muitas outras redes) e responder a "

[commentsDesc2]
other = "este post"

[commentsDesc3]
other = "para comentar aqui."

[comments]
other = "Comentários"

[commentLoad]
other = "Carregando comentários do Fediverso..."

[noComments]
other = "Nenhum comentário."
```

**No arquivo para tradução para o inglês `themes/just-another-bear/i18n/en.toml`**

```toml
[commentsDesc1]
other = "You can use your Fediverse (i.e. Mastodon, among many others) account to reply to "

[commentsDesc2]
other = "this post"

[commentsDesc3]
other = "to leave a comment here."

[comments]
other = "Comments"

[commentLoad]
other = "Loading comments from the Fediverse..."

[noComments]
other = "No comments found."
```

**No arquivo `themes/just-another-bear/layouts/partials/style.html`**

Por fim, algumas alterações no estilo:

```css
    /* Mastodon comments */

    .mastodon-comment {
      background-color: var(--block-background-color);
      border-radius: var(--block-border-radius);
      border: 1px var(--block-border-color) solid;
      padding: 20px;
      margin-bottom: 1.5rem;
      display: flex;
      flex-direction: column;
      color: var(--font-color);
      font-size: var(--font-size);
    }
    .mastodon-comment p {
      margin-bottom: 0px;
    }
    .mastodon-comment .author {
      padding-top:0;
      display:flex;
    }
    .mastodon-comment .author a {
      text-decoration: none;
    }
    .mastodon-comment .author .avatar img {
      margin-right:1rem;
      min-width:60px;
      border-radius: 5px;
    }
    .mastodon-comment .author .details {
      display: flex;
      flex-direction: column;
    }
    .mastodon-comment .author .details .name {
      font-weight: bold;
    }
    .mastodon-comment .author .details .user {
      color: #5d686f;
      font-size: medium;
    }
    .mastodon-comment .author .date {
      margin-left: auto;
      font-size: small;
    }
    .mastodon-comment .content {
      margin: 15px 20px;
    }
    .mastodon-comment .attachments {
      margin: 0px 10px;
    }
    .mastodon-comment .attachments > * {
      margin: 0px 10px;
    }
    .mastodon-comment .content p:first-child {
      margin-top:0;
      margin-bottom:0;
    }
    .mastodon-comment .status > div {
      display: inline-block;
      margin-right: 15px;
    }
    .mastodon-comment .status a {
      color: #5d686f;
      text-decoration: none;
    }
    .mastodon-comment .status .replies.active a {
      color: #003eaa;
    }
    .mastodon-comment .status .reblogs.active a {
      color: #8c8dff;
    }
    .mastodon-comment .status .favourites.active a {
      color: #ca8f04;
    }
```

### Como usar

Agora, os posts e páginas só precisam ter a propriedade `comments = true` para habilitar o bloco de comentários. Do contrário, nada do arquivo `comments.html` será renderizado.

E para apontar o bloco de comentários, basta preencher a propriedade `fediverse` com o id do seu post no Fediverso. Assim:

![Captura da tela de um cabeçalho de post](/images/comentarios/2024-10-08-173910.png)

O truque aqui é que você precisa postar primeiro (para gerar um id) e então editar o post para adicionar o id na propriedade. Coisa rápida.

Então o link é adicionado automaticamente no final do post, bem como todas as respostas são renderizadas corretamente. Eba!

![Captura de tela de um comentário de exemplo](/images/comentarios/2024-10-08-174610.png)
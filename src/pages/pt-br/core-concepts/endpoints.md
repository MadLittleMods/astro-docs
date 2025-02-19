---
layout: ~/layouts/MainLayout.astro
title: Endpoints
description: Aprenda a criar endpoints que podem processar todo tipo de dados
i18nReady: true
---

O Astro permite que você crie endpoints customizados para servir e processar todo tipo de dados. Isso pode ser usado para gerar imagens, expor um arquivo RSS ou os usar como rotas de API para construir uma API completa para o seu site.

Em sites gerados de forma estática, seus endpoints customizados são chamados durante a fase de build para produzir arquivos estáticos. Já em sites usando o [modo SSR](/pt-br/guides/server-side-rendering/#habilitando-o-ssr-em-seu-projeto) seus endpoints customizados se tornarão endpoints reais executados a cada requisição.

Endpoints estáticos e SSR são definidos de maneira similar, mas os endpoints SSR suportam funcionalidades adicionais.

## Endpoints de Arquivos Estáticos
Para criar um endpoint customizado, adicione um arquivo `.js` ou `.ts` no diretório `/pages`. A extensão do arquivo será removida durante o processo de build, portanto o nome do arquivo deve conter a extensão que você deseja que os dados usem, por exemplo `src/pages/data.json.ts` se tornará a rota `/data.json`.

Seus endpoints devem exportar uma função `get` (opcionalmente assíncrona) que recebe um objeto com as propriedades (`param` e `request`) como único parâmetro e retorna um objeto contendo a propriedade `body`. Essa função será chamada pelo Astro durante a build, que utilizará os conteúdos da propriedade `body` para gerar o arquivo.

```js title="src/pages/builtwith.json.ts"
// Se tornará: /builtwith.json
export async function get({params, request}) {
  return {
    body: JSON.stringify({
      name: 'Astro',
      url: 'https://astro.build/',
    }),
  };
}
```

O objeto retornado também pode conter a propriedade `encoding`. Ela deve ser uma string válida do tipo [`BufferEncoding`](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/bdd02508ddb5eebcf701fdb8ffd6e84eabf47885/types/node/buffer.d.ts#L169) aceita pelo método `fs.writeFile` do Node.js. Por exemplo, para gerar uma imagem no formato png retornamos:

```ts title="src/pages/builtwith.json.ts"
export async function get({ params, request }) => {
  const response = await fetch("https://astro.build/assets/press/full-logo-light.png");
  const buffer = Buffer.from(await response.arrayBuffer());

  return {
    body: buffer,
    encoding: 'binary',
  };
};
```

Também é possível adicionar validação de tipo à sua função com o tipo `APIRoute`:

```ts
import type { APIRoute } from 'astro';

export const get: APIRoute = async function get ({params, request}) {
  /* ... */
}
```


### Roteamento dinâmico e a propriedade `params`

Os endpoints suportam as mesmas funcionalidades de [roteamento dinâmico](/pt-br/core-concepts/routing/#rotas-dinâmicas) que as páginas. Nomeie seu arquivo com um nome de parâmetro entre colchetes e exporte uma função chamada [`getStaticPaths()`](/pt-br/reference/api-reference/#getstaticpaths). Assim será possível acessar o parâmetro utilizando a propriedade `params` passada para a função do endpoint.

```ts title="src/pages/[id].json.ts"
import type { APIRoute } from 'astro';

const usernames = ["Sarah", "Chris", "Dan"]

export const get: APIRoute = ({ params, request }) => {
  const id = params.id;
  return {
    body: JSON.stringify({
      name: usernames[id]
    })
  }
};

export function getStaticPaths () {
    return [ 
        { params: { id: "0"} },
        { params: { id: "1"} },
        { params: { id: "2"} },
    ]
}
```

Dessa forma serão gerados três endpoints JSON durante a build: `/api/1.json`, `/api/2.json` e `/api/3.json`. O roteamento dinâmico com endpoints funciona da mesma forma que nas páginas, porém, como um endpoint é uma função e não uma página, [props](/pt-br/reference/api-reference/#passagem-de-dados-com-props) não são suportadas.

### `request`
Todos os endpoints recebem uma propriedade `request`, porém no modo estático você só tem acesso a propriedade `request.url`. Ela retorna o URL completo do endpoint atual e funciona da mesma forma que [Astro.request.url](/pt-br/reference/api-reference/#astrorequest) funciona em páginas.

```ts title="src/pages/request-path.json.ts"
import type { APIRoute } from 'astro';

export const get: APIRoute = ({ params, request }) => {
  return {
    body: JSON.stringify({
      path: new URL(request.url).pathname
    })
  };
}
```

## Endpoints do Servidor (Rotas de API)
Tudo descrito na seção de endpoints de arquivos estáticos também pode ser utilizado no modo SSR: arquivos podem exportar uma função `get` que recebe um objeto com as propriedades `params` e `request`.

Porém, diferente do modo `static`, quando você configura o modo `server`, os endpoints serão construídos no momento em que são requisitados. Isso desbloqueia novas funcionalidades que estão indisponíveis durante a build e permite que você construa rotas de API que respondem requisições e seguramente executam código no servidor em runtime.

:::note
Não se esqueça de [habilitar o modo SSR no seu projeto](/pt-br/guides/server-side-rendering/#habilitando-o-ssr-em-seu-projeto) antes de testar esses exemplos.
:::

Os endpoints do servidor tem acesso a propriedade `params` sem exportar a função `getStaticPaths` e podem retornar um objeto [`Response`](https://developer.mozilla.org/pt-BR/docs/Web/API/Response), permitindo que você defina códigos de status e cabeçalhos HTTP.

```js title="src/pages/[id].json.js"
import { getProduct } from '../db';

export async function get({ params }) {
  const id = params.id;
  const product = await getProduct(id);

  if (!product) {
    return new Response(null, {
      status: 404,
      statusText: 'Não encontrado'
    });
  }

  return new Response(JSON.stringify(product), {
    status: 200,
    headers: {
      "Content-Type": "application/json"
    }
  });
}
```

Esse código responderá a qualquer requisição que corresponda à rota dinâmica. Por exemplo, se navegarmos para `/capacete.json`, `params.id` será `capacete`. Se `capacete` existir no banco de dados, o endpoint irá criar um objeto `Response` para responder com JSON e retornar um [código de status HTTP](https://developer.mozilla.org/en-US/docs/Web/API/Response/status) de sucesso. Caso contrário, ele usará o objeto `Response` para responder com um erro `404`.

### Métodos HTTP
Além da função `get`, você pode exportar uma função com o nome de qualquer [método HTTP](https://developer.mozilla.org/pt-BR/docs/Web/HTTP/Methods). Assim, quando uma requisição for recebida, o Astro irá checar o método e chamar a função correspondente.

Também é possível exportar uma função `all` para corresponder a todos os métodos que já não tenham suas respectivas funções exportadas. Se houver uma requisição sem método correspondente, ela será redirecionada para a sua [página de 404](/pt-br/core-concepts/astro-pages/#página-customizada-de-erro-404).

:::note
Como `delete` é uma palavra reservada do JavaScript, exporte uma função chamada `del` para corresponder ao método delete.
:::

```ts title="src/pages/methods.json.ts"
import type { APIRoute } from 'astro';

export const get: APIRoute = ({ params, request }) => {
  return {
    body: JSON.stringify({
      message: "Método GET"
    })
  }
};

export const post: APIRoute = ({ request }) => {
  return {
    body: JSON.stringify({
      message: "Método POST!"
    })
  }
}

export const del: APIRoute = ({ request }) => {
  return {
    body: JSON.stringify({
      message: "Método DELETE!"
    })
  }
}

export const all: APIRoute = ({ request }) => {
  return {
    body: JSON.stringify({
      message: `Método ${request.method}!`
    })
  }
}
```

### `request`
No modo SSR, a propriedade `request` retorna um objeto [`Request`](https://developer.mozilla.org/pt-BR/docs/Web/API/Request) completamente utilizável que se refere a requisição atual. Isso permite que você aceite dados e cheque cabeçalhos:

```ts title="src/pages/test-post.json.ts"
export const post: APIRoute = async ({ request }) => {
  if (request.headers.get("Content-Type") === "application/json") {
    const body = await request.json();
    const name = body.name;

    return new Response(JSON.stringify({
      message: `Seu nome é: ${name}`
    }), {
      status: 200
    })
  }
  return new Response(null, { status: 400 });
}
```

### Redirecionamentos
Como `Astro.redirect` não está disponível em rotas de API, você pode usar o método [`Response.redirect`](https://developer.mozilla.org/en-US/docs/Web/API/Response/redirect) para redirecionar:

```js title="src/pages/links/[id].js" {14}
import { getLinkUrl } from '../db';

export async function get({ params }) {
  const { id } = params;
  const link = await getLinkUrl(id);

  if (!link) {
    return new Response(null, {
      status: 404,
      statusText: 'Não encontrado'
    });
  }

  return Response.redirect(link, 307);
}
```

### Exemplo: Verificando um desafio captcha
Endpoints do servidor podem ser usados como uma API REST para executar funções de autenticação, acesso ao banco de dados e verificação sem expor dados sensíveis ao cliente.

No exemplo abaixo, uma rota de API é usada para verificar um desafio Google reCAPTCHA v3 sem expor os segredos dele aos usuários.

No servidor nós definimos uma função correspondente ao método POST que aceita os dados do desafio e então verifica seu resultado com a API do reCAPTCHA. Aqui nós podemos definir valores secretos ou ler variáveis de ambiente de maneira segura.

```js title="src/pages/recaptcha.js"
export async function post({ request }) {
  const data = await request.json();

  const recaptchaURL = 'https://www.google.com/recaptcha/api/siteverify';
  const requestBody = {
    secret: "SUA_CHAVE_SECRETA",   // Isso pode ser uma variável de ambiente
    response: data.recaptcha       // O token recebido pela requisição
  };

  const response = await fetch(recaptchaURL, {
    method: "POST",
    body: JSON.stringify(requestBody)
  });

  const responseData = await response.json();

  return new Response(JSON.stringify(responseData), { status: 200 });
}
```

Então podemos acessar o endpoint utilizando `fetch` a partir de uma página do seu site:

```astro title="src/pages/index.astro"
<html>
  <head>
    <script src="https://www.google.com/recaptcha/api.js"></script>
  </head>

  <body>
    <button class="g-recaptcha" 
      data-sitekey="PUBLIC_SITE_KEY" 
      data-callback="onSubmit" 
      data-action="submit"
     >
       Clique aqui para verificar o desafio reCAPTCHA!
     </button>

    <script is:inline>
      function onSubmit(token) {
        fetch("/recaptcha", {
          method: "POST",
          body: JSON.stringify({ recaptcha: token })
        })
        .then((response) => response.json())
        .then((gResponse) => {
          if (gResponse.success) {
            // Checagem bem-sucedida, captcha válido
          } else {
            // Checagem malsucedida
          }
        })
      }
    </script>
  </body>
</html>
```
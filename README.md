# groove-apis — backend (Groovy) e frontend (Angular) separados

```
groove-apis/
├── backend/    API REST em Groovy + Spring Boot + SQLite  (porta 8081)
├── frontend/   Groove, loja de vinis em Angular            (porta 4200)
├── postman/    Coleção com TODOS os endpoints + testes
└── tools/      Dublê da API e testes de contrato (só para teste)
```

## Backend

Requisitos: JDK 21.

```bash
cd backend
./mvnw spring-boot:run      # Windows: mvnw.cmd spring-boot:run
```

A API sobe em `http://localhost:8081`. O banco é o arquivo `backend/lojinha.db`
(criado/atualizado automaticamente se não existir).

Na primeira execução com a tabela `produto` vazia, o backend cadastra sozinho os 20 discos
do catálogo (códigos 1 a 20, iguais aos das capas e áudios). **Se você já tem um
`lojinha.db` antigo com produtos de teste, apague-o** (ou esvazie a tabela `produto`) para o
catálogo ser criado com os códigos certos.

CORS: as origens liberadas ficam em `lojinha.cors.allowed-origins`
(`application.properties`), por padrão `http://localhost:4200`.

> No Linux/macOS, se o `./mvnw` reclamar de permissão depois de descompactar o zip:
> `chmod +x backend/mvnw`.

### Endpoints

| Recurso | Endpoints | Tela que usa |
|---------|-----------|--------------|
| Cliente | `GET /api/cliente/{codigo}` | Minha conta |
| | `POST /api/cliente` | Cadastro |
| | `PUT /api/cliente` | Minha conta |
| | `DELETE /api/cliente/{codigo}` | Minha conta |
| | `POST /api/cliente/login` | Login |
| | `PATCH /api/cliente/resetarSenha` | Esqueci minha senha |
| | `PATCH /api/cliente/alterarSenha` | Minha conta |
| Produto | `GET /api/produtos` | Vitrine, Catálogo (admin) |
| | `GET /api/produto/{codigo}` | Detalhe, sincronização da cesta |
| | `GET /api/produto/vitrine` | Vitrine (destaques) |
| | `GET /api/produto/busca/{termo}` | Resultado da busca |
| | `POST /api/produto` | Catálogo (admin) |
| | `PUT /api/produto` | Catálogo (admin) |
| | `DELETE /api/produto/{codigo}` | Catálogo (admin) |
| Pedido | `POST /api/pedido/finalizar` | Cesta (fechar a compra) |
| | `GET /api/pedidos/cliente/{codigo}` | Meus pedidos |
| | `GET /api/pedido/{codigo}/detalhe` | Detalhe do pedido |
| | `GET /api/pedido/{codigo}` | — (CRUD cru, Postman) |
| | `POST /api/pedido` | — (CRUD cru, Postman) |
| | `PUT /api/pedido` | — (CRUD cru, Postman) |
| | `DELETE /api/pedido/{codigo}` | — (CRUD cru, Postman) |

Observações sobre o contrato (vale para o Postman e para o frontend):

- `POST` sempre cria um registro novo: o `codigo` enviado é ignorado. Para alterar, use `PUT`.
- Erros de regra de negócio voltam como **HTTP 400** com `{"mensagem": "..."}`.
- `GET` de um registro inexistente devolve **200** com o objeto vazio (`codigo: 0`), não 404.
- `PUT` de um registro inexistente devolve **200** com o texto `... não encontrado!`.
- A **senha nunca volta** nas respostas de cliente (nem no login).
- Em `PUT /api/cliente`, senha em branco **mantém** a senha atual.
- Em `PATCH /api/cliente/alterarSenha`, o campo **`documento` carrega a senha ATUAL**
  (herdado do backend Java original) e `senha` é a nova.
- Em `POST /api/pedido/finalizar`, o preço cobrado é sempre o **do banco**
  (promoção quando houver), nunca o enviado pelo cliente. Linhas repetidas do mesmo produto
  são somadas, e tudo roda em uma transação: se um item não tem estoque, nada é gravado.
- `dataPedido` sai como `yyyy-MM-dd` nos dois endpoints que a devolvem.

## Frontend

```bash
cd frontend
npm install
npm start
```

Abre em `http://localhost:4200`. A URL da API está em
`frontend/src/app/service/api.config.ts` (já aponta para `http://localhost:8081`).

Telas: vitrine (`/`), promoções (`/promo`), detalhe do disco (`/detalhe?codigo=N`),
busca (`/busca?q=`), cesta (`/cesta`), cadastro (`/cadastro`), login (`/login`),
esqueci a senha (`/reenvio`), meus pedidos (`/pedidos`), detalhe do pedido
(`/pedido?codigo=N`), **minha conta (`/conta`)** e **catálogo/admin (`/admin`)**.

Capas e prévias são arquivos estáticos nomeados pelo código do produto:
`frontend/public/{codigo}.png` e `frontend/public/audio/{codigo}.mp3`. Um disco
cadastrado pelo `/admin` só terá capa e prévia se você adicionar esses arquivos.

> A tela `/admin` não é uma barreira de segurança: a API **não tem autenticação nem
> papel de administrador**, então qualquer um que alcance a API pode cadastrar, alterar e
> excluir produtos, com ou sem essa tela. Para valer, o controle precisa existir no backend.

## Testes

```bash
cd frontend
npm test                 # testes de tela/unidade (Angular + vitest)
npm run build            # compila tudo (SSR + prerender)
```

Contrato da API, com o backend rodando na 8081:

```bash
node tools/testar-api.mjs                        # 71 verificações, sem dependências
node tools/testar-api.mjs --base http://host:porta
```

Ele cria os próprios dados (cliente, produtos e pedidos marcados com `[teste ...]`) e
apaga tudo no fim, então pode rodar contra o backend real sem sujar o catálogo.

Pelo Postman (ou Newman), importe os dois arquivos de `postman/` e rode as pastas em ordem
— **Produto → Cliente → Pedido → Limpeza**:

```bash
npx newman run postman/groove-apis.postman_collection.json --env-var baseUrl=http://localhost:8081
# 49 requests, 98 asserções
```

### Rodar o frontend sem JDK

`tools/api-stub.mjs` é um **dublê da API só para teste/desenvolvimento**: reproduz em
memória o mesmo contrato do backend (status, mensagens e regras), com o mesmo catálogo de
20 discos. Serve para mexer no frontend sem instalar JDK/Maven e para rodar os testes em CI.

```bash
node tools/api-stub.mjs          # sobe na 8081, igual ao backend
```

Não é o backend de produção: não tem banco e não deve ser publicado.

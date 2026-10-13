<p align="center">
  <img src="docs/logo.png" alt="imovCG" width="440">
</p>

<p align="center">
  Busca de imóveis para aluguel em Campina Grande, reunindo anúncios de OLX e Facebook num mapa só.
</p>

<p align="center">
  <a href="http://18.216.234.111"><b>Aplicação em produção</b></a>
</p>

---

## O problema

Quem procura aluguel em Campina Grande — em especial estudante da UFCG, UEPB e IFPB — precisa
vasculhar a OLX e vários grupos de Facebook, um por um. Os anúncios não têm formato comum, ninguém
consegue comparar preço com localização de forma confiável e é fácil perder um imóvel bom porque
ele apareceu num grupo que a pessoa não acompanha.

O imovCG coleta esses anúncios automaticamente, normaliza tudo num mesmo esquema e apresenta num
mapa com filtros de preço, tipo, quartos, bairro e distância de um campus. O contato continua sendo
feito direto com o anunciante, no anúncio de origem — não há intermediação nem taxa.

Além do que é coletado, proprietários e imobiliárias de Campina Grande podem cadastrar imóveis
diretamente na plataforma: eles entram com a conta Google, publicam pelo painel do anunciante e o
imóvel passa a aparecer no mesmo mapa, marcado com a fonte `imovcg`. Esse inventário é exclusivo —
não existe nos portais nacionais.

## Arquitetura

Três serviços independentes, cada um no seu repositório, integrados por HTTP:

| Serviço | Stack | Responsabilidade |
|---|---|---|
| [`frontend`](https://github.com/ImovCG/frontend) | React 19, TypeScript, Vite, Leaflet, Tailwind | Mapa, filtros, favoritos |
| [`backend`](https://github.com/ImovCG/backend) | Java 21, Spring Boot 4, JPA, MySQL | API REST, persistência, deduplicação |
| [`webscrapping`](https://github.com/ImovCG/webscrapping) | Python, Selenium | Coleta, normalização e envio em lote |

Este repositório é o guarda-chuva: reúne os três como submódulos e contém o `docker-compose.yml`
que sobe o conjunto.

Na frente de tudo fica um **Caddy**, que atende as portas 80 e 443 e emite e renova sozinho o
certificado HTTPS do Let's Encrypt para o domínio em `DOMINIO`. Ele repassa as requisições ao
`nginx` do frontend, que serve o SPA e faz proxy reverso de `/api/` para o backend pela rede
interna do Docker.

HTTPS com domínio não é opcional: o login com Google recusa origens `http` e endereços IP.

## Executando

Requisitos: Docker e Docker Compose.

```bash
git clone --recurse-submodules git@github.com:ImovCG/imovcg.git
cd imovcg
cp .env.example .env
```

Edite o `.env` com as senhas do banco e suba:

```bash
docker compose up -d
```

A aplicação fica em `http://localhost` e o banco persiste no volume `mysql_data`.

### Variáveis de ambiente

| Variável | Descrição | Padrão |
|---|---|---|
| `MYSQL_ROOT_PASSWORD` | Senha de root do MySQL | — |
| `DB_USERNAME` / `DB_PASSWORD` | Credenciais da aplicação | — |
| `DOMINIO` | Domínio público; o Caddy emite o certificado HTTPS para ele | `localhost` |
| `CORS_ALLOWED_ORIGINS` | Origens aceitas pelo backend | `*` |
| `SCRAPE_INTERVAL_SECONDS` | Intervalo entre coletas | `604800` (7 dias) |
| `GOOGLE_CLIENT_ID` | Client ID OAuth usado para validar o login do anunciante | — |
| `JWT_SECRET` | Segredo HMAC que assina a sessão do anunciante | — |
| `JWT_EXPIRATION_SECONDS` | Validade do token de sessão | `604800` (7 dias) |
| `SCRAPER_API_KEY` | Chave de serviço da carga em lote. Vazio deixa o endpoint aberto | vazio |

## Anunciante

O anunciante entra com a conta Google (Google Identity Services, gratuito). O frontend troca o ID
token do Google por um JWT próprio no `POST /api/auth/google`; esse JWT protege apenas as rotas de
`/api/anunciante/**`. Todo o resto da API continua público — quem só quer procurar imóvel não
precisa de conta.

| Rota | Acesso |
|---|---|
| `GET /api/imoveis`, `GET /api/imoveis/{id}` | Público |
| `POST /api/auth/google` | Público |
| `GET /api/auth/eu`, `PATCH /api/auth/eu` | JWT |
| `GET/POST/PUT/DELETE /api/anunciante/imoveis` | JWT, escopo do dono |
| `POST /api/imoveis/lote` | `X-Api-Key`, se `SCRAPER_API_KEY` estiver definida |

Cada anúncio pertence a um anunciante: as rotas do painel só enxergam os imóveis do dono do token,
e tentar ler ou editar o anúncio de outra pessoa devolve 404.

**Contato.** Imóvel coletado leva para o anúncio de origem, na OLX ou no Facebook. Imóvel
cadastrado aqui não tem origem para onde apontar: o botão abre o WhatsApp do anunciante. Como o
Google não fornece telefone, o anunciante informa o número no painel (`PATCH /api/auth/eu`) — sem
ele o botão de contato fica desativado, e o painel avisa isso.

**Fotos.** O imovCG guarda apenas a URL da imagem, nunca o arquivo. Links de compartilhamento do
Google Drive são convertidos para o endereço direto da imagem; o arquivo precisa estar como
"Qualquer pessoa com o link".

### Configurando o login do Google

1. No [Google Cloud Console](https://console.cloud.google.com/), crie um projeto.
2. Em **APIs e Serviços → Tela de permissão OAuth**, escolha **Externo** e preencha nome e e-mail
   de suporte. Com os escopos básicos (`openid`, `email`, `profile`) não há processo de verificação.
3. Em **Credenciais → Criar credenciais → ID do cliente OAuth → Aplicativo da Web**, adicione as
   origens JavaScript autorizadas: `http://localhost:5173`, `http://localhost` e a URL de produção.
4. Coloque o Client ID em `GOOGLE_CLIENT_ID` (backend, no `.env`) e na variável de repositório
   `VITE_GOOGLE_CLIENT_ID` do GitHub Actions do frontend, que a injeta no build da imagem.

O Client ID é público por natureza e vai no bundle do frontend. O `JWT_SECRET` não: ele fica só no
`.env` da VM.

### Coleta do Facebook

A coleta da OLX funciona sem configuração. O Facebook exige sessão autenticada: o scraper reusa
cookies gerados **manualmente uma única vez**, e nenhuma senha é digitada de forma automatizada.

```bash
cd webscrapping
python salvar_cookies.py     # abre o Chrome, você faz login, gera artifacts/fb_cookies.json
```

Copie o arquivo para `secrets/fb_cookies.json` na máquina onde o container roda — o
`docker-compose.yml` monta esse diretório como somente leitura. Sem o arquivo, o scraper do
Facebook é ignorado e apenas a OLX é coletada.

### Submódulos

Os submódulos apontam para commits fixos. Depois de alterar um serviço, atualize o ponteiro aqui:

```bash
git submodule update --remote frontend
git add frontend && git commit -m "chore: atualiza ponteiro do frontend"
```

## Deploy

As imagens são publicadas no GitHub Container Registry como
`ghcr.io/imovcg/{frontend,backend,webscrapping}`. Na VM, o
[Watchtower](https://github.com/nicholas-fedor/watchtower) observa os containers marcados e
substitui automaticamente quando uma tag `latest` é atualizada, dispensando deploy manual.

## Licença

MIT — veja [LICENSE](LICENSE).

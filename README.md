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

## Arquitetura

Três serviços independentes, cada um no seu repositório, integrados por HTTP:

| Serviço | Stack | Responsabilidade |
|---|---|---|
| [`frontend`](https://github.com/ImovCG/frontend) | React 19, TypeScript, Vite, Leaflet, Tailwind | Mapa, filtros, favoritos |
| [`backend`](https://github.com/ImovCG/backend) | Java 21, Spring Boot 4, JPA, MySQL | API REST, persistência, deduplicação |
| [`webscrapping`](https://github.com/ImovCG/webscrapping) | Python, Selenium | Coleta, normalização e envio em lote |

Este repositório é o guarda-chuva: reúne os três como submódulos e contém o `docker-compose.yml`
que sobe o conjunto.

O `nginx` do frontend serve o SPA e faz proxy reverso de `/api/` para o backend pela rede interna
do Docker, de modo que a porta 8080 não precisa ficar exposta.

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
| `CORS_ALLOWED_ORIGINS` | Origens aceitas pelo backend | `*` |
| `SCRAPE_INTERVAL_SECONDS` | Intervalo entre coletas | `604800` (7 dias) |

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

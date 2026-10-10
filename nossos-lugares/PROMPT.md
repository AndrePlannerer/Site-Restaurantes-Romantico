# Nossos Lugares — prompt de construção

Prompt para colar num construtor de app (Claude Code, Cursor, v0, Lovable) e gerar a versão completa, hospedada e compartilhada entre os dois.

> O `prototipo.html` desta pasta é a primeira versão, feita como artifact do Claude. Ele usa o armazenamento do Claude (`claude.use("db")` e `claude.use("assets")`), então **não funciona aberto direto do GitHub Pages** — serve de referência visual e de fluxo (upload de foto → leitura do EXIF → mapa e galeria).

## Avisos antes de usar

- **Instagram é o ponto fraco.** O Instagram bloqueia leitura automática de perfis e posts. O prompt resolve buscando o @ no Google Places, o que funciona bem para restaurante conhecido e falha para lugar pequeno.
- **Google Places API exige cartão cadastrado** no Google Cloud. Tem cota gratuita mensal, mas o cadastro é necessário para a captação de endereço e horário funcionar.

---

```
Construa um web app chamado "Nossos Lugares" — um diário de lugares para um
casal (eu e a Emily). Dois usuários apenas, os dois com acesso total de
leitura e escrita a tudo. O app precisa ficar hospedado na internet com URL
própria e login.

## Stack
- Next.js (App Router) + TypeScript + Tailwind, deploy na Vercel.
- Supabase para banco (Postgres), auth e storage de fotos.
- Auth por magic link, com allowlist de exatamente dois e-mails. Qualquer
  outro e-mail recebe tela de "acesso restrito".
- Mapa: MapLibre GL JS com tiles do MapTiler (ou Mapbox GL). Mapa real,
  com zoom, arraste e clusters quando houver muitos pinos.

## Modelo de dados
place
  id, status ('visitado' | 'quero_ir'), name, address, lat, lng,
  city, neighborhood, category (enum), cuisine_tags (array de texto),
  price_level (1-4), source_url, phone, website, instagram_handle,
  opening_hours (jsonb: por dia da semana), google_place_id,
  rating_andre (1-5, null), rating_emily (1-5, null),
  visited_at (date, null), created_by, created_at, updated_at
photo
  id, place_id, storage_path, taken_at, lat, lng, uploaded_by, created_at
note
  id, place_id, author (qual dos dois), body, created_at, updated_at
  → cada pessoa tem sua própria nota por lugar, editável só por ela,
    visível para os dois
comment
  id, place_id, author, body, created_at, parent_id (para respostas)
  → thread cronológica, os dois comentam no lugar do outro

## Telas

1. MAPA (home)
   - Pinos de todos os lugares. Cor diferente para "visitado" e "quero ir".
   - Filtros no topo: status, categoria, cuisine tag, cidade/bairro,
     faixa de preço, "só os que a Emily ainda não avaliou".
   - Clicar no pino abre um card lateral (desktop) ou bottom sheet (mobile)
     com foto, nota dos dois, e botão "ver detalhe".

2. TABELA
   - Tabela real, densa, ordenável e filtrável: nome, categoria, cuisine,
     bairro, preço, nota dele, nota dela, média, data da visita, nº de fotos.
   - Busca por texto. Exportar CSV.
   - Alternar entre visão tabela e visão galeria de fotos (cards com foto).

3. QUERO IR
   - Campo único de "colar link". Aceita: link do Google Maps (inclusive
     encurtado maps.app.goo.gl), site do restaurante, ou perfil/post do
     Instagram.
   - Ao colar, o backend resolve o link e preenche automaticamente:
     nome, endereço, lat/lng, telefone, site, horário de funcionamento,
     faixa de preço, categoria e cuisine tags.
   - Fluxo obrigatório: mostrar os dados extraídos num formulário
     editável antes de salvar, com a origem de cada campo indicada.
     Nunca salvar direto sem revisão.
   - Estratégia de extração, nesta ordem:
     a) Google Maps link → expandir o redirect, extrair place_id ou
        coordenadas, e chamar a Google Places API (Place Details) para
        nome, endereço, horários, telefone, site, price_level, types.
        Esta é a fonte principal e a mais confiável.
     b) Site próprio → buscar a página no servidor e ler JSON-LD
        schema.org/Restaurant (name, address, openingHoursSpecification,
        telephone, servesCuisine) e, na ausência dele, meta tags OpenGraph.
        Se achou nome + cidade mas não achou endereço/horário, usar Google
        Places Text Search com esses dados para completar.
     c) Instagram → NÃO tentar scraping do HTML, o Instagram bloqueia.
        Extrair só o @handle da URL, usar o handle como busca no Google
        Places Text Search, e mostrar os 3 melhores resultados para a
        pessoa escolher qual é. Se nenhum servir, cair no preenchimento
        manual com o handle já salvo.
   - Qualquer falha de extração cai em formulário manual com mensagem
     clara do que não deu para achar — nunca um erro genérico.
   - Botão "já fomos": move para status visitado, pede data e foto.

4. DETALHE DO LUGAR
   - Fotos (upload múltiplo, com leitura de EXIF para data e GPS;
     se a foto tiver GPS e o lugar não tiver coordenada, sugerir usar).
   - Endereço, horário de funcionamento formatado com indicador
     "aberto agora / fecha às X", telefone, link do site e do Insta,
     botão "abrir no Google Maps".
   - Duas colunas de nota pessoal, uma de cada, editáveis inline,
     cada um só edita a sua.
   - Avaliação em estrelas de cada um, separada, com a média calculada.
   - Thread de comentários abaixo, com respostas e edição.
   - Categoria e cuisine tags editáveis.

## Categorização
- category (enum, um por lugar): restaurante, bar, café, padaria,
  sorveteria, doceria, food truck, mercado, passeio, hospedagem, outro.
- cuisine_tags (vários por lugar, multi-select com busca e criação de
  tag nova): japonesa, italiana, brasileira, nordestina, mineira,
  árabe, mexicana, peruana, chinesa, coreana, tailandesa, indiana,
  francesa, portuguesa, hambúrguer, pizza, churrasco, frutos do mar,
  vegetariana, vegana, brunch, café especial, confeitaria, sorvete,
  cervejaria, vinho, coquetelaria.
- Ao importar de um link, mapear os `types` da Google Places API para
  category automaticamente, e sugerir cuisine tags a partir do nome e
  da descrição — mas sempre como sugestão editável, nunca fixa.
- Na tabela e no mapa, os filtros de cuisine são combináveis.

## Detalhes que importam
- Mobile-first de verdade: a maior parte do uso é no celular, na rua.
- Atualização em tempo real entre os dois (Supabase Realtime): se ela
  adiciona um lugar, aparece na minha tela sem refresh.
- Toda ação destrutiva (apagar lugar, apagar foto) pede confirmação.
- Fotos comprimidas no cliente antes do upload (lado maior 1600px, JPEG 0.82).
- Estados vazios com instrução do que fazer, não só "nenhum resultado".
- Horários guardados em formato estruturado, não string, para o
  "aberto agora" funcionar no fuso de São Paulo.
- Português do Brasil em toda a interface.

## Design
Tom de diário de viagem, não de dashboard SaaS. Uma família tipográfica
com personalidade para títulos e uma neutra para texto. Fotos são o
elemento principal — deixe elas respirarem. Evite cards idênticos
arredondados com sombra cinza em tudo. Suporte a tema claro e escuro.

## Entregue também
- README com os passos de deploy e as variáveis de ambiente necessárias
  (Supabase URL e keys, Google Places API key, MapTiler key), incluindo
  como restringir cada key por domínio.
- Migrations SQL do schema e as políticas RLS do Supabase, com as
  políticas garantindo que só os dois e-mails da allowlist leem e
  escrevem, e que cada um só edita a própria nota e os próprios
  comentários.
```

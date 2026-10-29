# Minas Mais Ranking · Documentação do frontend

> Documento para guiar a construção no Cursor. Base: protótipo HTML navegável (arquivo `minas-mais-ranking.html`) e a especificação original. Slogan: **"Venda. Supere. Conquiste."**

## 1. Visão geral

Aplicativo mobile/web exclusivo da Rede Minas Mais. Transforma campanhas comerciais de laboratórios em uma competição entre lojas.

**Regra fundamental (nunca quebrar):** as lojas são divididas em quatro grupos de porte (P, M, G, GG). Cada grupo tem seu próprio ranking e seu próprio campeão. **Lojas de grupos diferentes nunca são comparadas**, em nenhuma tela, cálculo ou notificação.

## 2. Stack recomendada

| Camada | Escolha | Motivo |
|---|---|---|
| Framework | Next.js 14+ (App Router) + TypeScript | Rotas por perfil, mobile e web no mesmo código |
| Estilo | Tailwind CSS + variáveis CSS (tokens) | Tema claro/escuro por tokens |
| Estado de servidor | TanStack Query | Cache e atualização periódica do ranking |
| Estado de UI | Zustand (ou Context) | Perfil, campanha e grupo selecionados |
| Formulários | React Hook Form + Zod | Validação de campanha e importação |
| Gráficos | SVG próprio ou Recharts | Evolução da posição e vendas por laboratório |
| Planilhas | SheetJS (`xlsx`) + PapaParse | Importação de Excel/CSV no navegador |
| Animação | Motion (motion.dev) | Pódio, barras, troca de posições e Telão (ver seção 16) |
| Testes | Vitest + Testing Library + Playwright | Regras de ranking e fluxos críticos |
| Mock de API | MSW | Frontend funciona antes do backend |

Fontes: **Barlow Condensed** (títulos e números) e **Barlow** (texto), via `next/font/google`.

## 3. Estrutura de pastas

```
src/
  app/
    (app)/
      layout.tsx              # topo + barra de abas + menus
      page.tsx                # Início
      ranking/page.tsx
      loja/page.tsx           # Minha loja (vendedor, gerente)
      supervisao/page.tsx     # supervisor
      admin/
        page.tsx              # campanhas
        importar/page.tsx
        lojas/page.tsx
      campeoes/page.tsx
    telao/page.tsx            # tela cheia, sem layout do app
  components/
    ranking/                  # RankRow, Podium, GroupCard, GroupChip, Medal, DeltaBadge
    campanha/                 # CampaignCard, CampaignChips, CampaignMeta
    loja/                     # ClimbCard, GoalMeter, NeighborList, PositionChart
    supervisao/               # StoreRow, LabSalesBars, SummaryTiles
    admin/                    # CampaignForm, ImportPreview, StoreGroupTable
    ui/                       # Button, Card, Chip, Segmented, Sheet, Toast, Field
  domain/
    ranking.ts                # funções puras (ver seção 6)
    format.ts                 # moeda, unidades, pontos, % e datas pt-BR
    types.ts
  services/                   # chamadas à API (hoje: mock)
  mocks/                      # dados fictícios + handlers MSW
  styles/tokens.css
```

## 4. Perfis e navegação

| Perfil | Abas | Telão | Vê |
|---|---|---|---|
| Administrador | Início, Ranking, Admin, Campeões | Sim | Tudo; cria campanhas, importa vendas, gerencia lojas |
| Supervisor | Início, Ranking, Supervisão, Campeões | Sim | Suas lojas, vendas por laboratório |
| Gerente | Início, Ranking, Minha loja, Campeões | Sim | Posição, meta e como subir |
| Vendedor | Início, Ranking, Minha loja, Campeões | Não | Posição e como subir |

Proteja as rotas por perfil (middleware ou guard no layout). Barra de abas fixa embaixo no celular; no desktop pode virar lateral.

## 5. Modelo de dados (tipos)

```ts
type GrupoId = 'P' | 'M' | 'G' | 'GG';
type Metrica = 'valor' | 'unidades' | 'pontos' | 'crescimento';
type StatusCampanha = 'rascunho' | 'ativa' | 'encerrada';

interface Loja { id: number; codigo: string; nome: string; grupo: GrupoId; supervisorId: number }

interface Campanha {
  id: string; nome: string; laboratorio: string;
  inicio: string; fim: string;          // ISO yyyy-mm-dd
  metrica: Metrica; grupos: GrupoId[];   // grupos participantes
  desempate: 'unidades' | 'valor' | 'mix';
  status: StatusCampanha;
  metaPorGrupo?: Partial<Record<GrupoId, number>>;
  produtos: { id: string; nome: string; pontos: number }[];
}

interface Venda { data: string; lojaId: number; produtoId: string; laboratorio: string; quantidade: number; valor: number }

interface PosicaoRanking {
  campanhaId: string; grupo: GrupoId; lojaId: number;
  posicao: number; valor: number; posicaoAnterior: number;
}
```

Entidades completas no backend (para referência): usuários, lojas, grupos, supervisores, laboratórios, produtos, campanhas, campanha_produtos, vendas, rankings, pontuações, histórico_ranking, notificações, premiações.

## 6. Regras de negócio (funções puras em `domain/ranking.ts`)

1. **`rank(campanha, grupo)`**: lista as lojas **só do grupo**, ordenadas por pontuação decrescente. Empate: critério da campanha; se persistir, id da loja.
2. **`posicao(campanha, loja)`**: índice da loja dentro de `rank(campanha, loja.grupo)`.
3. **`variacao(campanha, loja)`**: posição anterior menos posição atual (positivo = subiu). A posição anterior vem do fechamento da semana anterior.
4. **`faltaParaSubir(campanha, loja)`**: pontuação da loja acima menos a da loja. Se for a 1ª, mostrar vantagem sobre a 2ª.
5. **`equivalenciaEmUnidades`**: valor → unidades ÷ preço médio; pontos → unidades ÷ pontos por unidade; unidades → o próprio número; crescimento → sem equivalência.
6. **`meta(campanha, grupo)`**: usar a meta definida; se não houver, média do grupo × 1,2 (crescimento: 20 %).
7. **Métricas**: valor = R$ vendido; unidades = soma de quantidades; pontos = unidades × pontos do produto; crescimento = % sobre o período de referência.
8. **Campanha encerrada**: somente leitura. Define campeão, 2º e 3º por grupo.
9. **Mudar loja de grupo**: a loja sai do ranking antigo e entra no novo em todas as campanhas ativas. Pedir confirmação.
10. **Campanhas simultâneas**: cada uma tem ranking independente; a loja participa de várias.

Cubra tudo isso com testes unitários, principalmente a regra fundamental (nenhuma função pode receber lojas de grupos diferentes).

## 7. Telas

### Início
Cartão de destaque por perfil (loja: campanhas ativas, melhor posição, variação na semana; supervisor: lojas, campanhas, posições no top 3; admin: totais e última importação). Alerta para campanhas que terminam em até 3 dias. Lista de campanhas ativas (posição da loja ou dias restantes, barra de tempo decorrido). Atalho para o Telão e bloco "Campeões recentes".

### Ranking da campanha
Seletor de campanha (chips). Cabeçalho com métrica, período, dias restantes e desempate. Quatro cards de grupo (líder, valor, nº de lojas; grupo fora da campanha fica desativado). Pódio do grupo selecionado e lista completa com barra relativa ao líder, variação e destaque da própria loja. Aviso fixo: só há comparação dentro do mesmo grupo.

### Minha loja
Posição grande, "de N lojas", variação, valor da métrica, barra de meta. Card escuro **"Suba no ranking"** com quanto falta para ultrapassar a loja de cima e a equivalência em unidades. Lista de vizinhos (acima, você, abaixo, com diferença). Gráfico de evolução da posição (5 semanas, eixo invertido, 1º no topo). Produtos da campanha.

### Supervisão
Seletor de campanha. Quatro indicadores (melhor loja, loja que pede atenção, lojas na campanha, crescimento médio). Lojas agrupadas por porte, cada uma com posição no próprio grupo e distância para o 1º. Barras de vendas por laboratório.

### Admin
Três abas: **Campanhas** (listar, criar, ativar, encerrar com confirmação), **Importar vendas** (arquivo → prévia com linhas reconhecidas e erros → confirmar → rankings recalculados e resumo de mudanças), **Lojas e grupos** (trocar grupo).

### Campeões
Quadro de títulos e, por campanha encerrada, 1º, 2º e 3º de cada grupo com valor final.

### Modo Telão (`/telao`)
Tela cheia, sempre escura. Top 5 dos quatro grupos, troca de campanha ativa a cada 9 s, barra de progresso, relógio, Esc fecha. Atualização periódica dos dados e animação das barras.

### Notificações
Subiu posições, entrou no top 3, falta X para o 1º, caiu de posição, campanha encerrando, líder do grupo.

## 8. Importação de vendas

Colunas: **Data, Código/Nome da loja, Código/Produto, Laboratório, Quantidade, Valor**.

1. Ler o arquivo no navegador (SheetJS ou PapaParse).
2. Para cada linha: identificar loja (código ou nome), produto, laboratório e a campanha ativa cujo grupo inclui o grupo da loja.
3. Marcar erros (loja não encontrada, sem campanha ativa, valor inválido) sem travar o restante.
4. Mostrar prévia: "N linhas prontas, M com erro".
5. Ao confirmar, enviar as linhas válidas e mostrar a mudança de posição da loja afetada.

## 9. Sistema visual

Tokens em `styles/tokens.css`; use sempre os nomes, nunca valores soltos.

| Token | Claro | Escuro | Uso |
|---|---|---|---|
| `--bg` | #F2EFEE | #0F0E0E | fundo |
| `--surface` | #FFFFFF | #1A1818 | cards |
| `--surface2` | #F7F5F4 | #242121 | campos e blocos internos |
| `--ink` | #1B1919 | #F4F1F0 | texto |
| `--muted` | #6A6361 | #A59E9C | texto secundário |
| `--line` | #E4DFDD | #2F2B2A | bordas |
| `--red` | #D3202A | #F0444C | destaque e números |
| `--hero` | #C4161F | #B3131C | cartão de destaque e botão primário |
| `--red-soft` | #FCE7E8 | #3B1719 | fundo da própria loja |
| `--up` / `--down` | #17834A / #C2371F | #4CC585 / #FF7A5C | subiu / caiu |
| `--gP` `--gM` `--gG` `--gGG` | #2A9D5C `#2D6BDB` `#E58A1F` `#C8102E` | #4CC585 `#6FA0FF` `#F2A24E` `#F26A70` | grupos (verde, azul, laranja, vermelho) |
| `--gold` `--silver` `--bronze` | #C79A1E #98A0A6 #B06A3C | #E0B33A #B2B9BE #C98558 | pódio |

Raios sugeridos: 8, 12, 16 px. Espaços: escala de 4 px. Uma sombra suave só nos cards. Tema segue `prefers-color-scheme`. Estilo: moderno, premium e corporativo, com números grandes em Barlow Condensed e listras diagonais discretas no cartão de destaque e no pódio.

## 10. Componentes (contrato)

| Componente | Props principais | Observação |
|---|---|---|
| `GroupChip` | `grupo` | Obrigatório sempre que houver posição |
| `GroupCard` | `grupo`, `lider`, `selecionado`, `desativado` | Cor do grupo na borda quando selecionado |
| `Podium` | `itens` (top 3), `minhaLojaId` | Ordem visual 2º, 1º, 3º |
| `RankRow` | `posicao`, `loja`, `valor`, `variacao`, `maximo`, `ehMinhaLoja` | Barra relativa ao líder |
| `DeltaBadge` | `variacao` | ▲ verde, ▼ vermelho, traço neutro; adicionar texto para leitor de tela |
| `ClimbCard` | `faltaParaSubir`, `lojaAcima`, `equivalencia` | Estado alternativo "você lidera" |
| `GoalMeter` | `valor`, `meta` | `role="progressbar"` |
| `PositionChart` | `posicoes[]`, `totalLojas` | Incluir descrição acessível com os valores |
| `CampaignChips` | `campanhas`, `selecionada` | `aria-pressed` |
| `Sheet` | `aberto`, `onFechar` | Prender foco, Esc fecha, devolver foco ao botão |
| `Toast` | `mensagem` | `role="status"` |

Cada componente que mostra posição recebe o grupo, para impedir comparação entre grupos.

## 11. Estados e acessibilidade

- Todos os botões têm hover, foco visível, desativado e carregando.
- Estados vazios: "Ainda não há vendas nesta campanha."; carregando com esqueleto; erro com mensagem e ação de tentar de novo.
- Contraste de texto 4,5:1 (3:1 em texto grande). Testar o laranja do Grupo G no tema claro.
- Navegação completa por teclado; `aria-current="page"` na aba ativa.
- `prefers-reduced-motion` desliga animações.
- Layout funciona a partir de 360 px; sem rolagem horizontal da página.

## 12. Textos (pt-BR)

Termos fixos: **posição**, **grupo**, **un** (unidades), **pts** (pontos). Exemplos:

- "Falta R$ 3.366 para ultrapassar a Barbacena e chegar ao 4º lugar."
- "Você é o 1º do Grupo M. Vantagem de R$ X sobre a Nova Lima."
- Botões com o resultado: "Importar 7 linhas", "Encerrar campanha".
- Erros dizem o que houve e como resolver: "Loja L99 não encontrada. Confira o código no cadastro de lojas."

## 13. Plano de entrega

1. Projeto, tokens, layout e navegação por perfil.
2. `domain/ranking.ts` com testes.
3. Mock de dados (24 lojas: 8 P, 7 M, 5 G, 4 GG) e MSW.
4. Ranking e Início.
5. Minha loja e Supervisão.
6. Admin e importação.
7. Campeões, Notificações e Telão.
8. Acessibilidade, testes de ponta a ponta e ajustes de desempenho.

## 14. Critérios de aceite

- Nenhuma tela mistura lojas de grupos diferentes.
- Trocar o perfil muda abas e conteúdo.
- Importar o arquivo de exemplo atualiza o ranking e mostra a mudança de posição.
- Encerrar uma campanha a bloqueia e a leva para Campeões.
- Telão troca de campanha sozinho e fecha com Esc.
- Tema claro e escuro corretos; passa em teclado e leitor de tela nos fluxos principais.

## 16. Animações com Motion (motion.dev)

Biblioteca: **Motion** (pacote `motion`). Em React, importe de `motion/react`; no Next.js App Router, use em componentes com `"use client"` (ou `motion/react-client`). Em Vue/Nuxt, use a versão `motion-v`. Confirme a API na documentação oficial antes de implementar.

```bash
npm i motion
```

Centralize durações e curvas em `src/styles/motion.ts` (por exemplo: rápido 150 ms, padrão 250 ms, destaque 600 ms; mola suave para o pódio). Nenhum componente define tempo solto.

| Onde | Efeito | Recurso do Motion |
|---|---|---|
| Pódio | 3º, 2º e 1º sobem em sequência; medalha aparece por último | `initial`/`animate` com `delay` em cascata (`stagger`) |
| Barras do ranking | Crescem até o valor relativo ao líder ao entrar na tela | `whileInView` + animação de `scaleX` |
| Lista de ranking | Ao importar vendas, as linhas **trocam de lugar com animação** | `layout` / `layoutId` e `AnimatePresence` |
| Número da posição e valores | Contam até o novo valor | `animate` com `useMotionValue` e `useTransform` |
| Seta ▲/▼ | Pequeno pulo quando a loja sobe ou desce | `animate` em `y` |
| Cards da Início | Entrada escalonada | `variants` com `staggerChildren` |
| Menus e Toast | Entram e saem suaves | `AnimatePresence` |
| Gráfico de evolução | Linha se desenha da esquerda para a direita | animação de `pathLength` |
| Telão | Troca de campanha com transição; barras reanimam a cada rodada | `AnimatePresence mode="wait"` |

Regras:
- Respeitar `prefers-reduced-motion`: use `MotionConfig reducedMotion="user"` na raiz do app.
- Animar só `transform` e `opacity` sempre que possível, para manter 60 fps em celulares simples.
- Animação não pode esconder informação: o valor final precisa estar visível se a animação falhar.
- Não animar ao reordenar entre grupos: cada lista é de um único grupo, e a troca de lugar só acontece dentro dele.
- Medir antes de adicionar mais: no Telão, limite a 20 elementos animados ao mesmo tempo.

## 15. Fora do escopo desta fase

Autenticação real, backend e banco, integração com o sistema de vendas da rede, premiações, metas e desafios (previstos como expansão). O frontend consome uma API mockada com os tipos da seção 5.

# DESIGN.md · Minas Mais Ranking

Direção baseada na identidade da rede (vermelho, branco, cinza/preto) e no protótipo do app. Não foi comparada com sites reais (UXKIN não conectado).

## Sensação
Competição esportiva com acabamento corporativo: placar, pódio, números grandes. Sem gradientes roxos, sem vidro, sem emoji.

## Cor (claro / escuro)
| Token | Claro | Escuro | Papel |
|---|---|---|---|
| bg | #F2EFEE | #0F0E0E | fundo da página |
| surface | #FFFFFF | #1A1818 | cards |
| ink | #1B1919 | #F4F1F0 | texto |
| muted | #6A6361 | #A59E9C | texto secundário (contraste ≥ 4.5:1 sobre bg/surface) |
| line | #E4DFDD | #2F2B2A | bordas |
| primary | #C4161F | #B3131C | botões e destaques (texto branco) |
| red-text | #D3202A | #F0444C | números e links em destaque |
| up / down | #17834A / #C2371F | #4CC585 / #FF7A5C | subiu / caiu |
| grupo P / M / G / GG | #2A9D5C / #2D6BDB / #E58A1F / #C8102E | #4CC585 / #6FA0FF / #F2A24E / #F26A70 | só para identificar grupos |
| gold / silver / bronze | #C79A1E / #98A0A6 / #B06A3C | #E0B33A / #B2B9BE / #C98558 | pódio |

## Tipografia
- Títulos e números: Barlow Condensed 700/800, maiúsculas nos títulos de seção.
- Texto: Barlow 400/500/600, 16px mínimo, linhas de 60 a 75 caracteres.
- Escala: 14 · 16 · 20 · 28 · 40 · 64 · 96 (números de placar).

## Espaço e forma
- Escala de 4 px: 4 · 8 · 12 · 16 · 24 · 40 · 64 · 96. Seção: 64 px no celular, 96 px no desktop. Largura de conteúdo: 1120 px.
- Raios: 8 · 12 · 16. Uma sombra suave nos cards; sem sombras em outros lugares.

## Componentes
- Botão primário (vermelho), secundário (contorno) e discreto (texto), todos com hover, foco visível e desativado.
- Tag de grupo, linha de ranking, pódio e medalha vêm direto do protótipo.
- Imagens: capturas reais do app em moldura de celular; ícones de traço único.

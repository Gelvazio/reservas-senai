# Reservas SENAI Alto Vale — Instruções do Projeto

PWA de página única, estático, publicado no GitHub Pages em
`https://gelvazio.github.io/reservas-senai`. Mostra as reservas de **carros** e de **salas** da unidade
SENAI Alto Vale, lidas de uma planilha Google Sheets. Sem framework, sem build, sem backend:
HTML + CSS + JavaScript vanilla num único arquivo.

## Arquivos

| Arquivo | Papel |
|---|---|
| `index.html` | Todo o app (HTML, CSS e JS inline, ~74 KB). É a página servida em `/reservas-senai/`. |
| `reservas.html` | **Cópia idêntica** de `index.html`. Qualquer alteração deve ir nos dois arquivos (conferir com `diff -q index.html reservas.html`). |
| `manifest.json` | Manifesto PWA (`standalone`, tema `#0b4f9c`). Hoje com `icons: []`. |
| `sw.js` | Service worker. Cache `reservas-v22`, estratégia network-first com fallback para o cache. |

`index.html` e `reservas.html` precisam ficar iguais. Se o service worker listar arquivos novos, ou se o
comportamento de cache mudar, incremente a versão em `const C="reservas-vNN"` para invalidar o cache antigo.

## Como funciona

**Fonte de dados.** A planilha tem o ID em `const ID` no início do script. Cada aba da planilha é lida
no navegador do visitante:

1. `discover()` descobre a lista de abas buscando `.../htmlview` e interpretando o HTML (`parseTabList`).
   Se falhar, usa a lista guardada em `localStorage` (`reservas_tablist_*`) e, por último, a lista do snapshot.
2. `load2()` baixa cada aba em CSV por `.../export?format=csv&gid=...`, com `gviz/tq?tqx=out:csv` como
   alternativa. Se qualquer aba falhar, a atualização inteira é descartada e o app segue com o que já tem.
3. `setData()` faz o parse (`parseTab`) e redesenha. Se o CSV não mudou desde a última vez (`SIG`), não redesenha.

**Dados de reserva em cascata (do mais novo ao mais antigo):** planilha ao vivo → cache em `localStorage`
(`reservas_v3_*`, só vale se `ts >= SNAP_TS`) → `SNAP`, o snapshot embutido no próprio HTML.
`SNAP_TS` é o timestamp (ms) do snapshot e define quando o cache local fica obsoleto.

**Atualização.** Automática a cada 10 minutos (`setInterval`, só com a aba visível), ao voltar para a aba e
pelo botão "🔄 Atualizar agora". A query `?offline` desliga a busca automática (útil para testar o snapshot).

**Classificação de abas** (`classify`, pelo nome da aba, sem acento e em maiúsculas):
`CARRO` → carros · `MAPA` ou `SALA` → salas · `CONTATO` → contatos (`flat`) · demais abas são ignoradas.
Abas de tempo de deslocamento são ignoradas. O formato esperado do nome é, por exemplo,
`CARROS DE 05/10 A 10/10` e `MAPAS SALAS DE 05/10 A 10/10`.

**Formato das abas (o parser depende disso, não alterar a planilha sem ajustar `parseTab`):**
- *Carros:* linha de cabeçalho com `DATA` na 1ª coluna; colunas `DATA, COLABORADOR, CARRO, LOCAL/DESTINO,
  HORÁRIO RESERVA, CARONA`. A data aparece só na primeira linha do dia, como `05/10/2026 (Segunda-feira)`;
  as linhas seguintes do mesmo dia têm a coluna de data vazia.
- *Salas:* linhas de seção com o padrão `CURSO - TURNO - DD/MM`, seguidas de linhas com curso, turma, UC,
  docente e ambiente. Horários são extraídos por regex (`HH:MM às HH:MM`).
- *Contatos:* a primeira linha é o cabeçalho; a coluna `Nome` identifica a pessoa.

**Interface.** Navegação por hash (`#/carros`, `#/salas`, `#/carros/t/<aba>/d/<dia>`), três níveis:
seção → período (aba) → dia. Há busca de texto livre (`#q`) que atravessa todas as abas da seção atual.

**Foco no dia de hoje.** Ao entrar em Carros ou Salas (`#/carros`, `#/salas`) o app abre direto o período (aba)
que contém a data de hoje (fuso `America/Sao_Paulo`, funções `HOJE()` e `isHoje()`), via `location.replace`.
Na grade de dias o card de hoje vem primeiro, ocupa 100% da largura (`.ptile.hoje`) e tem o selo "HOJE";
os demais cards continuam abaixo. Se hoje não está em nenhuma aba, mostra a lista de semanas normal.
O botão ← volta para a lista de semanas em `#/<secao>/w` (sem esse sufixo o redirecionamento criaria um loop).

**Favorito.** O usuário escolhe um docente (`#fav`, guardado em `localStorage` na chave `reservas_fav`).
Reservas dele aparecem destacadas com ⭐, e também as em que ele está de carona: o campo `CARONA` é
texto livre, e `parts()`/`same()`/`inPool()` tentam casar nomes por heurística (palavras de parada em `STOP`,
nomes de lugares em `PLACES`). É heurístico: pode errar com apelidos, abreviações ou nomes repetidos.

## Modo PUBLIC e dados sensíveis

`const PUBLIC=true` é o modo da versão publicada. Nele:
- abas `CONTATO*` são ignoradas;
- qualquer aba cujo CSV case com a regex `SENS` (e-mail ou telefone) é **descartada por inteiro** antes de exibir.

⚠️ **Esse filtro só protege a exibição.** O repositório é público e o `SNAP` embutido no HTML contém
nomes completos de colaboradores, placas de veículos, destinos e horários de deslocamento. Isso são dados
pessoais (LGPD) visíveis para qualquer pessoa no código-fonte e no histórico do git. Regras:
- **Nunca** colocar e-mails, telefones, CPF ou documentos no snapshot nem em qualquer arquivo do repositório.
- Ao atualizar o `SNAP`, preferir dados minimizados ou anonimizados; avaliar se o snapshot precisa existir.
- Remover dados do HTML atual **não** os remove do histórico do git; tratar isso como decisão separada.

## Convenções de código

- JavaScript vanilla, sem dependências e sem build. Manter tudo inline em `index.html`.
- Escapar todo texto vindo da planilha com `esc()` antes de colocar em `innerHTML` (a planilha é entrada
  não confiável). Não concatenar valores da planilha direto em HTML.
- Layout mobile-first (uso principal é no celular): alvos de toque ≥ 44 px, sem rolagem horizontal.
- Idioma da interface e dos comentários: português do Brasil. Fuso de exibição: `America/Sao_Paulo`.

## Como testar

Não há suíte de testes. Servir a pasta e abrir no navegador:

```bash
python3 -m http.server 8000   # depois abrir http://localhost:8000/?offline
```

- `?offline` usa o snapshot e não acessa a planilha: bom para testar parsing e interface.
- Sem `?offline` o app tenta a planilha real. Em redes que bloqueiam `docs.google.com` ele cai no snapshot
  e mostra "offline (sem conexão com a planilha)".
- Depois de mexer no `sw.js` ou no HTML, recarregar com o cache limpo (DevTools → Application → Clear storage),
  porque o service worker mantém a versão anterior.

## Ao finalizar uma tarefa

1. Se alterou `index.html`, copiar para `reservas.html` e confirmar com `diff -q`.
2. Se alterou o comportamento de cache, incrementar a versão em `sw.js`.
3. Atualizar este `CLAUDE.md` se mudou estrutura, formato de dados ou regras acima.
4. Commit em português com mensagem descritiva e push.

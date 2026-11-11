# arteris

Monorepo organizado pelo boaz. Cada pasta tem uma única responsabilidade:

| Pasta | Conteúdo | Formato |
|---|---|---|
| `0_Assets/` | Recursos reutilizáveis: `projeto/` (marca, imagens, fontes), `requisitos/` (tema.md, requisito.mjs, casca.html) e `diagramas/` (tema, ícones, logos) | livre |
| `1_Requisitos/` | O produto desejado — sem backend, frontend ou tecnologia | `.md` versionado (+ `pdf/`) |
| `2_Diagramas/` | Diagramas de desenvolvimento (backend e frontend) | `.svg` versionado |
| `3_Arquitetura/` | Diagramas de alto nível: cloud, tecnologias, integrações | `.svg` versionado |
| `4_CI_CD/` | Estruturas de entrega (pipelines, deploy) | livre |
| `5_DEV/` | Código: `backend/`, `frontend/`, `qa/backend/`, `qa/frontend/` | livre |

- Requisitos são documentos `.md`, criados/alterados pela skill `requisito` no padrão de `0_Assets/requisitos/tema.md` (front matter e carimbo de topo com projeto, documento, versão, data/hora e status; histórico no rodapé; diagramas e telas como imagens `.svg` versionadas). A skill `requisito-estilo` cuida do formato, do HTML/PDF gerado por `0_Assets/requisitos/requisito.mjs` com a `casca.html`, da migração de `.html` antigos e do PDF. `./boaz.sh requisitos` lista os documentos; `./boaz.sh pdf <documento>` exporta o PDF versionado para `1_Requisitos/pdf/`.
- Diagramas são criados/alterados somente pelas skills `diagrama-dev` (2_) e `diagrama-arquitetura` (3_).
- Todo diagrama segue o tema dark de `0_Assets/diagramas/tema.svg` e embute (inline) os assets que usar de `0_Assets/`.
- Nome do diagrama: `<legenda>_v<NN>_<AAAAMMDD-HHMM>.svg`. Cada alteração gera um novo arquivo com a versão seguinte e, na mesma tarefa, a versão anterior é apagada (`git rm`) depois que todas as referências (requisitos `.md` e outros arquivos) forem atualizadas para a nova; o histórico fica no git.
- `./boaz.sh svg <pasta> <legenda>` devolve o próximo nome; `./boaz.sh diagramas` lista a versão mais recente de cada um; `./boaz.sh antigas` lista as versões antigas já sem referência (`--apagar` as remove).

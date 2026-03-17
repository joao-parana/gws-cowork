
# Usando o Google Workspace CLI com o Claude Cowork

## Você não precisa de TCP/IP — o passthrough MCP resolve tudo

**A ideia central: o Claude Cowork já faz a ponte entre sua sandbox VM e os MCP servers do host automaticamente.** Você não precisa de TCP/IP, SSH tunnels ou HTTP wrappers. Quando você configura um MCP server no Claude Desktop, a VM Linux do Cowork obtém acesso a ele via passthrough do protocolo SDK sobre Unix pipes. O Google Workspace CLI (`gws`) suporta o modo MCP server sobre stdio — o que o torna diretamente utilizável. Melhor ainda: existem MCP servers da comunidade maduros para Google Drive e Sheets que não exigem nenhum wrapping de CLI.

## A sandbox do Cowork é uma VM completa, mas o MCP a atravessa

O Claude Cowork não é um produto independente — é uma aba dentro do Claude Desktop (ao lado de Chat e Code), lançado em 12 de janeiro de 2026. Ele executa o Claude Code CLI dentro de uma **VM Linux Ubuntu 22.04 completa** sobre o Apple Virtualization Framework, e não apenas um App Sandbox do macOS. A VM recebe **4 vCPUs, ~3,8 GB de RAM e ~10 GB de disco** no seu Mac M3. Dentro da VM, as sessões são ainda mais isoladas com bubblewrap e filtros seccomp.

As restrições da sandbox são severas. O acesso à rede é limitado a uma allowlist restrita: **apenas `api.anthropic.com`, `pypi.org` e `registry.npmjs.org`** — todos os outros domínios retornam `403 Forbidden`. A VM não consegue acessar CLIs instalados no host, pacotes do Homebrew ou utilitários nativos do macOS. `curl` para URLs arbitrárias falha. É por isso que sua intuição sobre TCP/IP era razoável.

Mas a Anthropic construiu uma saída: **os MCP servers configurados no Claude Desktop são injetados dinamicamente na VM ao início de cada sessão** via protocolo SDK. A stack de comunicação entre host e VM inclui canais dedicados para MCP (Unix pipes), proxy HTTP (Unix socket → socat), arquivos compartilhados (VirtioFS) e message passing. O processo do MCP server roda no host com permissões normais — podendo executar qualquer CLI instalado no host, acessar o filesystem completo e fazer requisições de rede arbitrárias. O Cowork dentro da VM chama essas ferramentas de forma transparente.

## O Google Workspace CLI já fala MCP

O `gws` CLI (em `github.com/googleworkspace/cli`) é uma **ferramenta escrita em Rust** que gera sua superfície de comandos dinamicamente a partir do Google Discovery Service, cobrindo **mais de 50 APIs do Google Workspace**, incluindo Drive, Sheets, Gmail, Calendar, Docs e Chat. Foi lançado em 2 de março de 2026 e já acumulou **~15.700 estrelas no GitHub**. Instale via `npm install -g @googleworkspace/cli` — o pacote npm inclui binários ARM64 pré-compilados para macOS.

A ferramenta inclui (ou incluiu) um modo MCP server ativado com `gws mcp`. Isso inicia um **MCP server baseado em stdio** que expõe as APIs do Workspace como ferramentas estruturadas. O histórico do MCP é turbulento: o subcomando `mcp` foi removido na v0.8.0 em 7 de março alegando overhead de context window, mas o README atual na branch `main` ainda o documenta e o npm mostra a **v0.13.2** — bem além da versão da remoção. O modo MCP provavelmente retornou após pressão da comunidade. Para configurá-lo no `claude_desktop_config.json` do Claude Desktop:

```json
{
  "mcpServers": {
    "gws": {
      "command": "gws",
      "args": ["mcp", "-s", "drive,sheets"]
    }
  }
}
```

Isso inicia o `gws mcp` como um subprocesso no host. O Cowork acessa via passthrough do SDK — sem TCP/IP envolvido. A flag `-s` filtra por serviços específicos (Drive e Sheets neste exemplo), mantendo a quantidade de ferramentas gerenciável. Sem filtragem, o `gws` pode expor **200 a 400 ferramentas** consumindo de 40.000 a 100.000 tokens de contexto, razão pela qual a equipe removeu temporariamente o modo MCP.

Um aviso prático: o `gws` exige credenciais OAuth. Execute `gws auth login` no terminal do host primeiro para concluir o fluxo OAuth interativo. As credenciais são criptografadas com AES-256-GCM e armazenadas no keyring do macOS em `~/.config/gws/credentials.json`.

## MCP servers da comunidade podem ser a melhor escolha

Se o `gws mcp` se mostrar instável (o projeto avisa sobre breaking changes antes da v1.0), vários MCP servers da comunidade oferecem automação de Google Drive e Sheets com confiabilidade comprovada:

**`taylorwilsdon/google_workspace_mcp`** é a opção mais abrangente, cobrindo **12 serviços Google com mais de 100 ferramentas** — Drive, Sheets, Docs, Gmail, Calendar, Slides, Forms, Chat, Tasks, Contacts, Apps Script e Search. Suporta instalação com um clique via DXT para o Claude Desktop, OAuth 2.1 com suporte multi-usuário, tool tiers para controlar o uso de contexto e um modo read-only de segurança. Instale via `uvx workspace-mcp --tool-tier core` ou o pacote DXT.

**`xing5/mcp-google-sheets`** é desenvolvido especificamente para workflows com foco em Sheets, expondo **19 ferramentas dedicadas** (~13K tokens) para operações CRUD completas, batch updates, formatação e compartilhamento. Também inclui integração com Google Drive para gerenciamento de arquivos. Suporta service accounts para uso headless e automatizado.

**`isaacphi/mcp-gdrive`** estende o servidor de referência arquivado da Anthropic com suporte a escrita no Sheets — uma opção mais leve se você precisar apenas de busca/leitura no Drive e leitura/escrita no Sheets.

Os três usam transporte stdio e funcionam de forma idêntica na config do Claude Desktop. O Cowork os detecta automaticamente.

## Arquitetura recomendada para o seu Mac M3

A configuração mais simples e pronta para produção não exige TCP/IP, HTTP wrapping nem SSH tunneling:

```
┌─────────────────────────────────────────────────┐
│  Claude Desktop (macOS M3)                      │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐ │
│  │   Chat   │  │  Cowork  │  │     Code      │ │
│  └──────────┘  └────┬─────┘  └───────────────┘ │
│                     │ SDK protocol (pipes)       │
│  ┌──────────────────┴────────────────────────┐  │
│  │  MCP Server (processo no host)            │  │
│  │  Opção A: gws mcp -s drive,sheets         │  │
│  │  Opção B: uvx workspace-mcp               │  │
│  │  Opção C: uvx mcp-google-sheets@latest    │  │
│  └──────────────────┬────────────────────────┘  │
└─────────────────────┼───────────────────────────┘
                      │ HTTPS (Google APIs)
                      ▼
            Google Workspace APIs
```

**Passo a passo:**

1. **Instale a ferramenta escolhida no host.** Para o `gws`: `npm install -g @googleworkspace/cli`. Para o servidor da comunidade: `pip install workspace-mcp` ou use `uvx`.

2. **Autentique no host.** Para o `gws`: execute `gws auth setup` e depois `gws auth login` no Terminal. Para os servidores da comunidade: configure as credenciais OAuth conforme o README de cada um.

3. **Adicione o MCP server à config do Claude Desktop.** Edite `~/Library/Application Support/Claude/claude_desktop_config.json` e inclua a definição do servidor em `mcpServers`. Reinicie o Claude Desktop.

4. **Abra a aba Cowork e inicie uma tarefa.** As ferramentas MCP aparecem automaticamente. Peça ao Cowork para "listar meus arquivos recentes no Google Drive" ou "criar uma nova planilha com os dados do orçamento do Q1" — ele usará as ferramentas MCP de forma transparente.

## Quando você realmente precisaria de TCP/IP

O bridging via TCP/IP só se torna relevante em dois casos específicos. Primeiro, se você quiser rodar o MCP server em uma **máquina diferente** (por exemplo, um servidor cloud com credenciais de service account), você usaria o `mcp-remote` como bridge stdio-para-HTTP: `"args": ["npx", "mcp-remote", "https://seu-servidor.com/mcp"]`. Segundo, se você estiver fazendo wrapping de uma CLI que **não suporta MCP** em uma HTTP API usando Flask-Shell2HTTP ou um servidor Express customizado — mas como o `gws` já suporta MCP nativamente, isso se torna desnecessário.

O pacote `supergateway` (`npx -y supergateway`) converte entre qualquer combinação de transporte MCP (stdio ↔ SSE ↔ WebSocket ↔ Streamable HTTP), útil se você precisar fazer bridge entre transportes incompatíveis. E apps com sandbox no macOS conseguem conectar em `localhost:*` por padrão, então mesmo um HTTP wrapper local seria acessível — mas, novamente, desnecessário neste cenário.

## Conclusão

O enquadramento original da pergunta — precisar de TCP/IP para escapar da sandbox do Cowork — reflete uma restrição real (a VM bloqueia acesso arbitrário à rede e CLIs), mas aponta para a solução errada. **MCP é a saída planejada do Cowork.** Qualquer MCP server configurado no Claude Desktop roda no host com acesso total ao sistema e está disponível de forma transparente dentro da VM do Cowork. O modo MCP nativo do `gws` ou servidores da comunidade como o `taylorwilsdon/google_workspace_mcp` se encaixam diretamente nessa arquitetura. A recomendação prática: comece com o `xing5/mcp-google-sheets` se seu foco for automação de Sheets, ou com o `taylorwilsdon/google_workspace_mcp` se precisar de uma cobertura mais ampla do Workspace. Use o `gws mcp` apenas se precisar da superfície completa de mais de 50 APIs e aceitar a instabilidade de um projeto pré-v1.0. Sem TCP/IP necessário.


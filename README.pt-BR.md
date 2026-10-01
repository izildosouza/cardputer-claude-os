🌐 [English](README.md) · **Português (BR)**

# Cardputer Claude OS

Um bundle de "OS" feito em casa para o [M5Stack Cardputer](https://shop.m5stack.com/) —
faça o flash do firmware UIFlow, instale um launcher e rode uma pequena suíte de apps
que transformam o Cardputer num dispositivo Claude de bolso:

- **Claude Buddy** — pareie o Cardputer com o Claude Code via BLE; acompanhe
  as execuções dos agentes, o gasto de tokens e o tamanho da fila direto do bolso.
- **Push to Claude** — segure SPACE para gravar uma pergunta por voz e solte para
  enviar. O Whisper transcreve, o Claude Haiku 4.5 responde no LCD, e uma
  memória de 24 horas por dispositivo mantém o contexto entre as mensagens. Também
  tem modo de digitação para ambientes barulhentos.
- **Claude Pager** — digite uma tarefa no QWERTY, dispare como uma sessão
  de longa duração do [Managed Agents] na nuvem e acompanhe o status ao vivo
  (`bash: pytest …`, `wrote auth_test.py`, `idle ✓`) no LCD.
  A Inbox lista as sessões ativas; a tela Detail permite responder, interromper
  ou aprovar tool calls pendentes direto do bolso. Funciona em conjunto com a
  UI de navegador **Central Console** no seu Mac e com o sync de artifacts
  `claude-pull`.
- **Cardputer MCP** — transforma o dispositivo num pager de bolso para qualquer
  agente que fale Model-Context-Protocol (Claude Code, Cursor,
  Claude Desktop, Managed Agents etc.). O agente pode te chamar
  com um banner colorido + um chirp no alto-falante (`notify`), fazer uma
  pergunta de múltipla escolha que você responde no QWERTY (`ask`) ou
  exigir um gesto físico antes de operações destrutivas (`confirm`).
  Roda localmente via stdio/BLE — sem nuvem, sem precisar de Wi-Fi — **e
  agora também via [MCP tunnel], para que agentes na nuvem (Managed Agents, a
  Messages API) também consigam alcançar o dispositivo no seu bolso.** O
  gesto de segurar para confirmar vira uma **chave de aprovação em hardware**: um
  agente autônomo na nuvem fisicamente não consegue executar uma operação
  irreversível sem o seu dedo no dispositivo — nenhum prompt injection
  consegue sintetizar uma tecla mantida pressionada, e um dispositivo inacessível
  falha fechado (o agente para, nunca segue sozinho).

[MCP tunnel]: https://platform.claude.com/docs/en/agents-and-tools/mcp-tunnels/overview

- **Hello / Snake** — um app mínimo de exemplo + um jogo da cobrinha, para o bundle
  não ser só coisa séria.

[Managed Agents]: https://platform.claude.com/docs/en/managed-agents/overview

> **Fork de** [`moremas/build-with-claude`](https://github.com/moremas/build-with-claude).
> Este fork adiciona o diretório `worker/` (um Cloudflare Worker que
> cuida do STT de voz + chat com o Claude com memória de conversa) e o
> app de dispositivo `Push to Claude` que conversa com ele. Todo o resto
> vem do upstream — crédito e agradecimentos aos autores originais.

## Neste fork (izildosouza)

1. **Microfone do Cardputer original funcionando:** o `m5-onboard` fixa o firmware do modelo `cardputer` no UIFlow **v2.4.2**. As versões ≥ v2.4.3 usam ESP-IDF 5.5.x, cuja regressão no I2S faz o microfone retornar `-8` constante (a voz grava silêncio). Dá para sobrescrever com `M5_UIFLOW_VERSION=<tag>` ou `M5_UIFLOW_VERSION=latest`. Com isso, o Push to Claude por voz funciona também no Cardputer original, não só no Adv. Detalhes na [issue #8](https://github.com/dakshaymehta/cardputer-claude-os/issues/8).
2. **Gateway configurável no Worker:** o chat (`CHAT_BASE_URL`, `CHAT_MODEL`, `CHAT_API_KEY`) e a transcrição (`STT_BASE_URL`, `STT_MODEL`, `STT_API_KEY`) podem passar por um gateway compatível (OmniRoute, LiteLLM...). Sem essas variáveis, nada muda. O Pager continua exigindo `ANTHROPIC_API_KEY` direto na Anthropic. Detalhes em [`worker/README.md`](worker/README.md).
3. **Correção do `(empty)`:** o chat junta todos os blocos de texto da resposta (modelos que começam com um bloco de raciocínio mostravam `(empty)`).
4. **WiFi fora do git:** as credenciais ficam em `buddy/device/wifi_config.py` (gitignored; copie de `wifi_config.example.py`; só 2.4 GHz).
5. **Config pessoal do Worker fora do git:** IDs de KV e URLs podem ficar em `worker/wrangler.local.toml` (gitignored); o deploy fica `npx wrangler deploy -c wrangler.local.toml`.
6. **Dica para Windows:** se o esptool disser `No serial data received` mesmo com o device em modo download, confira se a porta COM do Cardputer não está duplicada com uma porta serial Bluetooth (Gerenciador de Dispositivos → Portas). Troque o número da porta do Cardputer e reinicie (Reiniciar, não Desligar, por causa da Inicialização Rápida).

Essas mudanças também foram propostas upstream nos PRs #9–#12 de [dakshaymehta/cardputer-claude-os](https://github.com/dakshaymehta/cardputer-claude-os/pulls).

## Novidades deste fork

| Adição                           | Onde                                                                                         | O que faz |
| -------------------------------- | -------------------------------------------------------------------------------------------- | --------- |
| **Cardputer MCP (bridge no host)** | [`mcp/`](mcp/)                                                                             | Servidor Model Context Protocol (baseado em `bleak`) que qualquer cliente Claude/MCP pode registrar, via **stdio ou streamable-http**. Seis tools: `notify`, `ask`, `confirm`, `show`, `progress`, `device_status`. Fala BLE com o app do dispositivo e registra cada decisão de `confirm` num log local de auditoria de consentimento (`~/.cardputer-mcp/audit.log`). |
| **MCP tunnel + daemon HTTP**     | [`mcp/auth.py`](mcp/auth.py) + [`tunnel/`](tunnel/) + [`mac/`](mac/)                         | O caminho de ponte com a nuvem. `CARDPUTER_HTTP=1` roda o mesmo servidor como um daemon streamable-http autenticado por bearer token (launchd); `tunnel/` (cloudflared + mcp-proxy) expõe esse daemon por um [MCP tunnel] da Anthropic, para que Managed Agents / a Messages API possam fazer `notify`/`ask`/`confirm` no dispositivo — só conexões de saída, fail-closed. |
| **Cardputer MCP (app do dispositivo)** | [`buddy/device/apps/cardputer_mcp.py`](buddy/device/apps/cardputer_mcp.py)             | Periférico BLE GATT num bloco novo de UUIDs de serviço (`a5cd0001-…`), separado do NUS do Buddy. Renderiza notificações, modais de pergunta e um gesto de confirmação segurando Y; envia os acks via notificações TX. |
| **Relay no Cloudflare Worker**   | [`worker/`](worker/)                                                                         | Endpoint de edge protegido por autenticação. Whisper para STT, Claude Haiku 4.5 para a resposta, Workers KV para a memória de conversa por dispositivo (últimas 8 mensagens, TTL de 24 h). |
| **App de voz + chat**            | [`buddy/device/apps/push_to_claude.py`](buddy/device/apps/push_to_claude.py)                 | Cliente no dispositivo. Faz streaming do WAV para o Worker enquanto grava (uso de RAM constante), modo de fallback por texto, respostas com rolagem, atalho `/reset`. |
| **App Pager no dispositivo**     | [`buddy/device/apps/pager.py`](buddy/device/apps/pager.py)                                   | UI de três telas (Compose / Inbox / Detail) para disparar e fazer a triagem de sessões do Managed Agents pelo QWERTY. Faz long-polling no Worker para o ticker de eventos ao vivo. |
| **SessionRouter Durable Object** | [`worker/src/router.do.js`](worker/src/router.do.js)                                         | Um DO por sessão da Anthropic. Faz polling sob demanda no endpoint `events.list` do Managed Agents, espelha os eventos no storage do DO e atende tanto o Pager (poll) quanto o Console (SSE). |
| **Central Console (navegador)**  | [`worker/src/console.html`](worker/src/console.html)                                         | Console HTML de arquivo único com tema escuro, servido pelo Worker. Stream de eventos ao vivo, bash com syntax highlighting, diffs inline para `str_replace`, pílulas de arquivo, interromper + responder. Protegido por token, sem etapa de build. |
| **Sync de artifacts no Mac**     | [`mac/claude-pull`](mac/claude-pull) + [`launchd plist`](mac/com.claude.pager.pull.plist)    | Script Python só com a stdlib, executado a cada 60 s pelo launchd. Puxa os arquivos de `/workspace/out/` de cada sessão para `~/ClaudeRuns/<title>-<id>/` e mostra uma notificação em banner quando uma sessão termina. |
| **Config do dispositivo externalizada** | [`buddy/device/apps/config.example.py`](buddy/device/apps/config.example.py)          | URL do Worker + secret do dispositivo carregados de um `config.py` no gitignore, para que segredos nunca entrem no repo. |
| **Skill Cardputer Companion**    | [`.claude/skills/cardputer-companion/SKILL.md`](.claude/skills/cardputer-companion/SKILL.md) | Agent Skill só de instruções. A contraparte comportamental do servidor MCP: ensina o Claude _quando_ recorrer a `notify`/`ask`/`confirm` e _como_ formatar para o LCD de 240×135 — exigindo `confirm` físico antes de operações irreversíveis, chamando atenção só ao fim de tarefas longas e, fora isso, ficando quieto. |

Veja [`worker/README.md`](worker/README.md) para o guia completo de deploy
no Cloudflare.

## Compre um Cardputer

O bundle foi pensado para o **M5Stack Cardputer-Adv** (a versão com microfone
PDM + alto-falante, necessária para o Push to Claude). Compre direto em
[shop.m5stack.com](https://shop.m5stack.com/) — procure por "Cardputer".
No upstream, o Cardputer original (não Adv) funciona para tudo, exceto
o app de voz. Neste fork a voz funciona nele também, com o firmware
fixado na v2.4.2 (veja [Neste fork](#neste-fork-izildosouza)).

## Início rápido — flash de um Cardputer

1. Clone este repo localmente — em qualquer lugar:
   ```bash
   git clone https://github.com/izildosouza/cardputer-claude-os.git
   ```
   A skill detecta sozinha o bundle buddy a partir do próprio local de instalação, então o caminho do clone não importa. `~/Downloads/m5stack/` e `~/Desktop/m5stack/` também são verificados como fallbacks convencionais.
2. Conecte o Cardputer ao notebook via USB-C
3. Abra o Claude Code e comece um chat novo
4. Aponte o Claude Code para a pasta do repo
5. Digite `m5-onboard go`

Só isso — o Claude faz o flash do firmware e envia os apps para o dispositivo automaticamente.

### Quando o Claude pedir para colocar o dispositivo em modo download

No meio do processo, o Claude vai pausar e pedir que você faça isto na **parte de trás** do dispositivo:

1. Segure o botão **G0** do Cardputer
2. Ainda segurando o G0, aperte o botão **Reset**
3. Solte o Reset primeiro e depois solte o G0
4. A tela apaga — o dispositivo está em modo download

A partir daí o Claude assume.

### O que acontece em seguida

- **O firmware é gravado no dispositivo** (~180 segundos)
- **Os apps são enviados para o dispositivo** (~100 segundos)
- **O dispositivo reinicia** direto no launcher — escolha um app e pronto

Feito. Ligue e desligue o dispositivo com a chave lateral.

---

## Início rápido — Cardputer MCP (deixe qualquer agente alcançar o dispositivo)

Transforme o Cardputer num pager de bolso que qualquer cliente que fale MCP
— Claude Code, Claude Desktop, Cursor, Codex, Managed Agents (via o
[MCP tunnel](#início-rápido--cardputer-via-mcp-tunnels-agentes-na-nuvem)
mais abaixo) ou qualquer coisa que suporte o Model Context Protocol — consegue
alcançar. Seis tools aparecem na primeira conexão:

- `cardputer.notify(title, body, urgency)` — mostra um banner no
  dispositivo e toca um chirp no alto-falante. A urgência define a cor do cabeçalho
  (info=escuro, warn=amarelo, crit=vermelho) e muda o padrão do bipe.
  Retorna assim que o banner aparece; some sozinho depois de 10 s. Um
  limite por agente (padrão de ~1 notify não-`crit` a cada 60 s) descarta
  banners em excesso com `rate-limited`; `crit` sempre toca.
- `cardputer.ask(question, choices, timeout_s)` — mostra uma pergunta
  de múltipla escolha numerada; o usuário aperta 1–4 no QWERTY;
  a string escolhida volta para o agente. Bloqueia o agente até
  o usuário responder, apertar ESC ou `timeout_s` expirar.
- `cardputer.confirm(title, details, timeout_s)` — exibe um banner
  vermelho de perigo e exige um gesto físico antes de resolver como
  `confirmed`. A ideia toda é que um prompt injection não consegue
  sintetizar uma tecla física mantida pressionada. Reserve isso para
  operações irreversíveis (deploys, force pushes, DROP TABLE,
  cobranças etc.). Passe `details` — o comando / SQL / diff /
  destinatário **real** — e o dispositivo renderiza isso num **diff da ação
  com rolagem acima do gesto**, para que o usuário aprove _o que leu_,
  e não só um título de 18 caracteres (o modelo das hardware wallets). No
  firmware atual o gesto é **tocar Y rapidamente** (a tela diz
  "TAP Y fast for 3s") porque o driver do teclado não tem
  auto-repeat — veja _Known limitations_ em
  [`mcp/README.md`](mcp/README.md). Toda decisão de `confirm`
  (confirmada, cancelada ou expirada) é adicionada a um
  **log local de auditoria de consentimento** em `~/.cardputer-mcp/audit.log` — quem pediu,
  o que pediram para você aprovar e o resultado — para que o gesto físico
  deixe um rastro durável que você pode revisar depois.
- `cardputer.show(text, channel)` — escreve uma **linha de status
  ambiente** na tela ociosa do dispositivo (silenciosa, não bloqueante, ignora
  o DND). Dê uma olhada no bolso para ver o que uma tarefa longa está fazendo
  (`running pytest`, `wrote auth.py`, `idle`); cada `channel` ganha
  sua própria linha, então vários agentes podem dividir a tela.
- `cardputer.progress(label, percent, channel)` — desenha uma **barra de
  progresso ao vivo** (0–100%) na tela ociosa — a irmã visual do
  `show`. Chame conforme uma tarefa longa avança (`0 → 25 → 60 → 100`) e uma
  barra verde vai enchendo no seu bolso; ela compartilha o anel de canais do `show`, então
  um canal pode alternar entre uma linha de status e uma barra. Silenciosa, ambiente,
  ignora o DND.
- `cardputer.device_status()` — uma verificação **somente leitura** de se o
  dispositivo está alcançável e do seu estado (`online; dnd=off; fw=0.4.2;
caps=…; battery=87%`). Passiva (não acorda o rádio), então um agente pode perguntar
  "meu humano está alcançável / concentrado?" antes de decidir interromper,
  graças a um heartbeat de ~10 s do dispositivo.

Toda a stack é local — MCP via stdio entre o seu cliente e a
bridge `bleak` no host, e depois BLE-GATT até o dispositivo. Sem ida
à nuvem, sem precisar de Wi-Fi. O cache de pareamento fica em
`~/.cardputer-mcp/paired.json`, então reconexões pulam o scan BLE.

### Configuração

1. **Envie o app para o dispositivo** (não precisa refazer o flash do firmware se você
   já fez o onboarding do dispositivo):

   ```bash
   python3 .claude/skills/m5-onboard/scripts/install_apps.py \
       --port <PORT> --src buddy
   ```

2. **Configure a bridge no host:**

   ```bash
   cd mcp
   python3 -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   ```

3. **Registre o servidor MCP no Claude Code:**

   ```bash
   claude mcp add cardputer \
       "$(pwd)/.venv/bin/python" \
       "$(pwd)/server.py"
   ```

   (Cursor, Codex, Claude Desktop etc. têm cada um a sua própria
   UI de registro de servidores MCP; aponte todos para o mesmo par
   `.venv/bin/python server.py`.)

4. **No dispositivo:** inicie o launcher e escolha **cardputer_mcp**
   no menu. A tela vai mostrar `waiting for bridge` e
   exibir o nome BLE `CardputerMCP_XXXXXX` do dispositivo. A primeira
   tool call do agente dispara um scan + conexão; a
   tela muda para um `READY` verde.

5. **Teste.** Numa sessão nova do Claude Code:

   > Dá um toque no Cardputer com o título "tests passing" e o corpo
   > "127 ok in 4.2s".

   Você deve ver o banner aparecer no dispositivo, ouvir um chirp,
   e o Claude recebe `"shown"` de volta. A ida e volta completa leva
   menos de um segundo depois da primeira conexão.

### Faça o Claude recorrer ao dispositivo por conta própria (a skill companion)

As três tools acima são as _mãos_ do dispositivo; de fábrica, o Claude
só usa essas tools quando você pede explicitamente ("dá um toque no Cardputer…").
A Agent Skill [`cardputer-companion`](.claude/skills/cardputer-companion/SKILL.md)
que vem no bundle adiciona os _modos_ — ensina o Claude **quando** recorrer
a elas e **como** formatar para a telinha, sem nenhum código extra:

- **Confirmar antes de operações irreversíveis.** Deploys em produção, force pushes,
  `DROP TABLE`, `rm -rf`, efeitos colaterais pagos → um gesto físico de segurar
  para confirmar, nunca um "tem certeza?" numa mensagem de chat (que um prompt injection
  poderia forjar). Se o dispositivo estiver inacessível, o Claude para em vez de
  tratar a ausência do dispositivo de segurança como aprovação.
- **Chamar atenção só quando importa.** Um `notify` ao terminar uma
  tarefa realmente longa que você está esperando longe do teclado — não uma
  narração passo a passo. A postura padrão é ficar _quieto_.
- **Perguntar quando estiver travado e você estiver longe** — uma pergunta de 2–4 opções no
  QWERTY em vez de ficar parado no chat.
- **Formatação para 240×135** embutida em toda mensagem para o dispositivo.

Ela é carregada automaticamente no Claude Code sempre que as tools MCP do `cardputer`
estão registradas (a skill fica em `.claude/skills/` neste
repo, então é descoberta junto com a `m5-onboard`). Nada para instalar;
nada para configurar. Para ver funcionando, registre o servidor MCP e
depois passe uma tarefa de verdade para o Claude — por exemplo, "roda a suíte de testes e me
avisa como foi" — e se afaste do notebook.

### Permissão de Bluetooth no macOS

O primeiro scan BLE da bridge dispara um pedido de permissão
do macOS. Aprove uma vez; o `bleak` guarda a permissão em cache. Se o Claude Code
roda dentro de um multiplexador de terminal em sandbox que não está vendo
o pedido, dê permissão de Bluetooth ao próprio app de terminal em
**System Settings → Privacy & Security → Bluetooth**.

### Verificando sem um cliente MCP

[`mcp/smoke_test.py`](mcp/smoke_test.py) exercita a bridge
diretamente (importa `Bridge` e chama as tools como Python), sem
precisar registrar o MCP. Útil para confirmar a conectividade
BLE, depurar o tratamento de gestos e validar firmware
novo antes de ligá-lo a uma sessão de verdade:

```bash
cd mcp && .venv/bin/python smoke_test.py
```

Ele roda um `notify`, depois um `ask` (você aperta 1–3 no dispositivo)
e depois um `confirm` (você toca Y rapidamente, conforme a ressalva acima).

Veja [`mcp/README.md`](mcp/README.md) para as notas completas de
arquitetura, a referência do protocolo de comunicação, as limitações conhecidas e o roadmap;
o formato do protocolo BLE fica em
[`buddy/references/mcp_protocol.md`](buddy/references/mcp_protocol.md).

---

## Início rápido — Cardputer via MCP tunnels (agentes na nuvem)

O caminho local via stdio descrito acima só funciona para um agente rodando no mesmo
notebook. Os **MCP tunnels** estendem o dispositivo ao **Claude na nuvem** — uma
sessão do [Managed Agents] no Console ou um agente da Messages API — para que um
agente autônomo encarando um job de 40 minutos na nuvem possa chamar
o dispositivo no seu bolso e, o mais importante, **exigir que você segure Y fisicamente
antes de qualquer passo irreversível**. A nuvem literalmente não consegue digitar no
teclado do Cardputer, então nenhum prompt injection ou loop descontrolado consegue forjar
consentimento; se o dispositivo estiver inacessível, o agente **falha fechado** e
para. É uma chave de aprovação em hardware para IA.

```
 cloud Claude            your Mac (always-on)
 (Managed Agent  ─tunnel─▶  cloudflared ─▶ mcp-proxy ─▶ 127.0.0.1:9000 ─BLE─▶ Cardputer
  / Messages API)  outbound-only           (Docker)     cardputer-mcp daemon
```

O mesmo daemon também atende o **Claude Code local** via loopback, então um
único dono do BLE e uma única trava física cobrem agentes na nuvem e locais.

[Managed Agents]: https://platform.claude.com/docs/en/managed-agents/overview

### Do que você precisa

- **Acesso ao beta de MCP tunnels + Managed Agents** (peça no Console).
  Os tunnels funcionam a partir do Managed Agents no Console e da Messages API — **não**
  no app de consumidor claude.ai.
- **Docker Desktop** e o dispositivo com o `cardputer_mcp` instalado.

### Configuração (≈10 min, uma vez só)

1. **Rode o daemon da bridge sempre ligado** (é dono do link BLE e atende
   streamable-http em `127.0.0.1:9000`):

   ```bash
   ./mac/install_cardputer_bridge.sh        # writes a stub env, exits
   $EDITOR ~/.config/cardputer-bridge/env   # set random tokens + tunnel domain
   ./mac/install_cardputer_bridge.sh        # renders + loads the launchd agent
   ```

   Aprove o pedido único de Bluetooth do macOS para o daemon.

2. **Aponte o Claude Code local para o mesmo daemon** (trava unificada — o
   instalador imprime isto já com o seu token preenchido):

   ```bash
   claude mcp add --transport http cardputer \
       http://127.0.0.1:9000/mcp \
       --header "Authorization: Bearer <your-local-token>"
   ```

3. **Suba o tunnel** e conecte-o a um agente na nuvem — o passo a passo
   completo (etapas no Console, geração de certificados, uso com Managed Agent e
   Messages API, e um checklist de verificação em 6 passos) está em
   [`tunnel/README.md`](tunnel/README.md):

   ```bash
   cd tunnel
   cp env.example .env && $EDITOR .env      # TUNNEL_DOMAIN + TUNNEL_TOKEN
   ./gen-certs.sh                           # CA + server cert; upload data/ca.crt in Console
   docker compose up -d
   ```

   Depois conecte `https://cardputer.<your-tunnel-domain>/mcp` (com o seu
   bearer token da nuvem) a um Managed Agent e peça para ele confirmar uma
   operação destrutiva — o dispositivo pisca em vermelho com `from:managed-agent`,
   você toca Y rapidamente por ~3s e o agente prossegue.

### O modelo de segurança num fôlego só

Só conexões de saída (nenhuma porta de entrada); o TLS interno é terminado por um certificado que **só
você** tem (a Cloudflare não consegue ler os payloads); um **bearer token** no
daemon protege o tunnel, que de outra forma não tem autenticação, e identifica qual agente
está pedindo; o **gesto físico** é o consentimento que não dá para forjar; e
**fail-closed** significa que um dispositivo apagado nunca é um sim. A skill
[`cardputer-companion`](.claude/skills/cardputer-companion/SKILL.md)
ensina o Claude a respeitar tudo isso. Toda decisão de `confirm` é
registrada num **log local de auditoria de consentimento** (`~/.cardputer-mcp/audit.log`)
— o primeiro degrau, escrito pelo daemon, da escada de comprovantes. Os diffs de ação
no dispositivo já existem hoje; comprovantes de consentimento assinados _criptograficamente_ e
quórum de várias pessoas estão documentados como uma escada futura em
[`docs/superpowers/`](docs/superpowers/).

---

## Início rápido — Push to Claude (voz + chat)

O app de voz precisa de um Cloudflare Worker controlado por você. São uns 10 minutos
de configuração única; depois disso, cada interação por voz/texto é um único toque.

1. Faça o deploy do Worker — siga o [`worker/README.md`](worker/README.md). No fim você terá uma URL do Worker e um `DEVICE_SECRET` que você mesmo gerou.
2. Aponte o dispositivo para ele:
   ```bash
   cp buddy/device/apps/config.example.py buddy/device/apps/config.py
   ```
   Edite o `config.py` e cole o seu `WORKER_BASE` e o seu `DEVICE_SECRET`.
3. Envie os apps para o Cardputer (não precisa refazer o flash do firmware):
   ```bash
   python3 .claude/skills/m5-onboard/scripts/install_apps.py --port <PORT> --src buddy
   ```
4. Inicie o dispositivo → **Push to Claude** → toque em SPACE.

O `config.py` está no gitignore — o seu secret fica na sua máquina.

---

## Início rápido — Claude Pager + Central Console (agentes na nuvem)

O Pager transforma o Cardputer num controle remoto + display de status
para sessões do [Anthropic Managed Agents]. Digite uma tarefa no QWERTY,
dispare e veja o dispositivo avançar por `bash`, `write`, `idle ✓`
em tempo real. Um console HTML autocontido no seu Mac espelha as
mesmas sessões com um log de eventos completo no estilo terminal; um job do launchd
sincroniza os artifacts que o agente salva em `~/ClaudeRuns/`.

[Anthropic Managed Agents]: https://platform.claude.com/docs/en/managed-agents/overview

O Pager usa o mesmo Worker do Push to Claude — termine aquele
início rápido primeiro. Depois:

1. **Crie um namespace KV extra** para o índice de sessões:

   ```bash
   cd worker
   npx wrangler kv namespace create INDEX
   ```

   Cole o id retornado em `worker/wrangler.toml`, substituindo
   `REPLACE_WITH_YOUR_INDEX_KV_ID`.

2. **Adicione a migração do Durable Object** (já está no `wrangler.toml`)
   e faça o deploy de novo:

   ```bash
   npx wrangler deploy
   ```

   O primeiro deploy registra o DO `SessionRouter` pelo bloco de migração
   v1; os deploys seguintes são normais.

3. **Abra o Central Console** em
   `https://<your-worker>.workers.dev/console`. No primeiro acesso ele pede
   o seu `DEVICE_SECRET` (o mesmo valor do Cardputer); o secret
   fica guardado no localStorage do navegador e nunca é enviado para lugar nenhum além do
   seu Worker. Clique em `+ New`, digite uma tarefa e veja rodar.

4. **Envie o app Pager** para o Cardputer. Ele vai como bytecode
   pré-compilado (`.mpy`) porque o código-fonte é grande demais para ser parseado
   dentro da heap que sobra no launcher:

   ```bash
   pip3 install --user --break-system-packages mpy-cross
   python3 buddy/scripts/push_pager_mpy.py --port <PORT>
   ```

   Reinicie o dispositivo e escolha **Pager** no menu do launcher.

5. **(Opcional) Sync de artifacts no Mac.** Os agentes salvam os artifacts
   voltados ao usuário em `/workspace/out/` dentro do container. O
   script `mac/claude-pull` espelha esses arquivos em `~/ClaudeRuns/<title>-<id>/`
   num agendamento de 60 segundos do launchd e te avisa com um banner quando
   uma sessão termina.
   ```bash
   ./mac/install_launchd.sh        # writes a stub config and exits
   $EDITOR ~/.config/claude-pager/config.json   # paste worker_base + device_secret
   ./mac/install_launchd.sh        # second run actually loads launchd
   ```
   Os logs ficam em `/tmp/claude-pull.{out,err}.log`. Para rodar manualmente, use
   `./mac/claude-pull -v`.

### Usando o Pager

Três telas, alternadas com o cluster de setas:

```
COMPOSE   ← →   INBOX   →   DETAIL
                            (Enter on a row)
```

- **Compose** — digite uma tarefa e aperte Enter para disparar. `→` pula para a Inbox
  sem enviar.
- **Inbox** — lista ao vivo das sessões recentes com indicador de status + uma linha com a última
  tool. Atualiza a cada 4 s. Up/Down para rolar, Enter para abrir o
  Detail, `D` para apagar, `N` para voltar ao Compose.
- **Detail** — ticker ao vivo de uma sessão. Faz long-polling no Worker, então
  as mudanças aparecem em ~1 s depois que o agente age.
  - `R` responder (envia uma mensagem de follow-up)
  - `I` interromper (envia `user.interrupt`)
  - `Y/N` aprovar/negar a confirmação de tool pendente (quando houver)
  - `Esc` volta para a Inbox

As notificações disparam em **todas** as telas — o Pager consulta o
Worker a cada 15 s em segundo plano. Quando um agente muda de estado:

| Gatilho                          | Bipe                  | Banner                          |
| -------------------------------- | --------------------- | ------------------------------- |
| `running` → `idle`               | chirp A5 → E6         | verde **DONE: <title>**         |
| confirmação de tool pendente     | D6 triplo urgente     | amarelo **NEEDS YOU: <title>**  |
| → `terminated`                   | A4 → A3 descendente   | vermelho **ERROR: <title>**     |

O estado é persistido em `/flash/.pager_notif.json`, então o mesmo DONE
não dispara de novo depois de um reboot.

### Usando o Central Console

Aba do navegador em `<your-worker>/console`. Protegido por token, tema escuro,
monoespaçado. Coluna da esquerda = sessões, painel principal = stream de eventos com:

- blocos de bash com syntax highlighting
- diffs inline para tool calls de `str_replace`
- blocos de resultado de tool recolhíveis
- botões `y`/`n` de confirmação pendente no compositor
- pílulas de arquivo na parte de baixo — clique para baixar

Aperte `n` (quando nenhum campo estiver em foco) para disparar uma tarefa nova. Use `⌘/Ctrl-Enter`
no compositor ou no modal de criação para enviar.

### Proteção de custo

Cada sessão do Managed Agents mantém um container na nuvem ativo durante toda a sua
vida — normalmente de alguns centavos a uns poucos dólares por tarefa.
O Worker impõe um limite diário de sessões criadas por dispositivo (`PAGER_DAILY_SPAWN_CAP`
no `wrangler.toml`, padrão 30). Aumente ou diminua a gosto.

---

## Usando o Claude Buddy (BLE)

1. Ligue o Cardputer
2. Escolha **Claude Buddy** no menu do launcher
3. No Claude Desktop: **Help → Troubleshooting → Enable Developer Tools** (uma vez só, fica salvo)
4. Depois **Developer menu → Hardware Buddy → Connect**

## Conexão automática ao WiFi

O launcher tenta conectar ao WiFi a cada boot e mostra o resultado
na tela — `Connected · IP: 192.168.x.x` em caso de sucesso, `WiFi: offline`
em caso de falha (o launcher sempre continua, de um jeito ou de outro). De fábrica
não há credenciais, então você vai ver `WiFi: offline`. Copie
[`buddy/device/wifi_config.example.py`](buddy/device/wifi_config.example.py)
para `buddy/device/wifi_config.py` (no gitignore) e preencha o seu
SSID + senha (só 2.4 GHz), depois rode o `install_apps.py` de novo. Para pular
a conexão automática de vez, remova a chamada `_connect_wifi_with_splash()`
perto do início de `main()`.

## Adicionando seu próprio app

1. Coloque um arquivo `.py` em `buddy/device/apps/`
2. Envie só os apps, sem refazer o flash:
   ```bash
   python3 .claude/skills/m5-onboard/scripts/install_apps.py --port <PORT> --src buddy
   ```
3. O launcher descobre o app novo sozinho no próximo boot

Use o `buddy/device/apps/hello_cardputer.py` como base — é o menor exemplo das convenções (polling do teclado, fonte, comportamento de saída).

## Voltando ao UIFlow original

O bundle buddy assume o fluxo de boot via `/flash/main.py`. Remova
esse arquivo e o launcher original do UIFlow inicia normalmente no próximo reset.
Pelo REPL do dispositivo:

```python
import os
os.remove('/flash/main.py')
import machine; machine.reset()
```

Para também remover os apps em `/flash/apps/`, percorra esse diretório
do mesmo jeito e apague o que você não quiser.

Se quiser um firmware UIFlow limpo por cima, rode `m5-onboard go` de novo
_sem_ `--apps`: a skill faz o flash do UIFlow e para, sem mexer no
sistema de arquivos.

---

## Pré-requisitos

Você precisa de **Python 3.10+**, **git** e **Claude Code** no seu notebook. O `pyserial` vem vendorizado em `.claude/skills/m5-onboard/scripts/vendor/`. O `esptool` tem licença GPL e **não** é vendorizado — a skill instala ele via pip na primeira execução se ainda não estiver no seu ambiente, então para o usuário a experiência continua sendo um único comando. Para pré-instalar explicitamente: `python3 -m pip install --user -r requirements.txt`.

Para o Worker do Push to Claude você também precisa de **Node.js 18+** e de uma
conta na Cloudflare; as instruções completas estão em [`worker/README.md`](worker/README.md).

Se precisar preparar o ambiente:

- **macOS** — o `python3` normalmente já vem instalado; se não vier, `brew install python`
- **Linux (Debian/Ubuntu)** — `sudo apt-get install -y python3 python3-pip git`
- **Windows** — `winget install -e --id Python.Python.3.13` e `winget install -e --id Git.Git`

**Só para Windows + placas mais antigas:** o driver USB-UART CH9102 é necessário para Basic / Fire / Core2 / StickC. Baixe em [WCH](https://www.wch.cn/downloads/CH343SER_EXE.html). O Cardputer-Adv e o CoreS3 usam o driver USB composto nativo do sistema e não precisam de nada extra.

**Quer que o `--apps buddy` aponte para outro bundle?** O padrão resolve para o diretório `buddy/device/` ao lado da skill neste repo, com `~/Downloads/m5stack/` e `~/Desktop/m5stack/` verificados como fallbacks. Para sobrescrever (por exemplo, se você mantém um fork ou tem um bundle customizado em outro lugar), defina `M5_BUDDY_DIR`:

```bash
export M5_BUDDY_DIR=/path/to/buddy/device
```

## Solução de problemas

- **O pedido de modo download fica se repetindo** — você está soltando o G0 cedo demais. Solte o Reset primeiro, continue segurando o G0 por mais ou menos um segundo e só então solte.
- **"No USB-UART bridge found" (placas mais antigas)** — instale o driver CH9102 no Windows; no macOS/Linux, desconecte e conecte de novo.
- **O Claude Buddy nunca conecta via BLE** — confira se o launcher do buddy (e não o do UIFlow) é o dono do `/flash/main.py`. A skill cuida disso automaticamente na instalação.
- **O Push to Claude mostra "Not configured"** — copie o `config.example.py` para `config.py`, preencha `WORKER_BASE` + `DEVICE_SECRET` e envie os apps de novo.
- **O Push to Claude retorna "unauthorized"** — o `DEVICE_SECRET` do `config.py` não bate com o que está configurado no Worker. Rode `wrangler secret put DEVICE_SECRET` de novo e atualize o `config.py` para ficar igual.
- **Alguma outra coisa parece quebrada** — rode `python3 .claude/skills/m5-onboard/scripts/smoke_test.py --port <PORT>` para uma verificação de I2C + LCD + alto-falante + botões.

## O que tem neste repo

- **`.claude/skills/m5-onboard/`** — a skill de onboarding. Detecta a porta, faz o flash do UIFlow, instala os apps. Veja [`.claude/skills/m5-onboard/SKILL.md`](.claude/skills/m5-onboard/SKILL.md) para o playbook completo e todas as pegadinhas já tratadas nos scripts.
- **`.claude/skills/cardputer-companion/`** — a skill de etiqueta em tempo de execução. Ensina o Claude quando e como usar as tools MCP do Cardputer (`notify`/`ask`/`confirm`). Só instruções — nenhum script. Veja [`.claude/skills/cardputer-companion/SKILL.md`](.claude/skills/cardputer-companion/SKILL.md).
- **`buddy/`** — o bundle de apps MicroPython que é instalado. Veja [`buddy/README.md`](buddy/README.md) para a estrutura do lado do dispositivo e as ferramentas de iteração.
- **`worker/`** — o Cloudflare Worker que faz o Push to Claude funcionar (voz + memória do chat). Veja [`worker/README.md`](worker/README.md) para as instruções de deploy.

Os três são desacoplados de propósito: a skill `m5-onboard` consegue instalar qualquer bundle via `--apps <path>`, o `buddy` é só o que vem neste repo, e o worker é opcional (só o app Push to Claude usa).

## Contribuindo

PRs são bem-vindos — principalmente apps novos para o launcher, placas novas e melhorias
no fluxo de onboarding. Abra uma issue antes se estiver planejando algo
não trivial. O código é pequeno e tenta, de propósito, continuar legível
de ponta a ponta.

## Licença

O código próprio deste projeto está licenciado sob **Apache 2.0** — veja [`LICENSE`](LICENSE) e [`NOTICE`](NOTICE).

O `pyserial` (BSD-3-Clause, compatível com Apache) é o único pacote de terceiros incluído em `.claude/skills/m5-onboard/scripts/vendor/`. O `esptool` (GPLv2+) intencionalmente não é vendorizado; ele é declarado como dependência pip em [`requirements.txt`](requirements.txt) para que o repositório em si continue limpo sob Apache-2.0. Veja [`LICENSE-THIRD-PARTY.md`](LICENSE-THIRD-PARTY.md) para mais detalhes.

Fork de [`moremas/build-with-claude`](https://github.com/moremas/build-with-claude); a licença Apache-2.0 do upstream é preservada em `LICENSE` e `NOTICE`.

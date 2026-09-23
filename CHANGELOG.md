# Changelog — arx-painel

Histórico de versões do painel web de administração do Arx OS, da mais
recente para a mais antiga. Segue aproximadamente o formato
[Keep a Changelog](https://keepachangelog.com/) — sem datas por versão,
já que o projeto não rastreia isso individualmente; a ordem abaixo é a
ordem cronológica real de lançamento.

---

### 0.36.1 — CORREÇÃO URGENTE: gevent 24.2.1 não tinha wheel pro Python 3.13
- **Instalação da 0.36.0 falhou de verdade em produção**: `pip
  install` tentava compilar `gevent` do zero (sem wheel pronta pra
  Python 3.13 nessa versão), e a compilação Cython do próprio gevent
  24.2.1 quebra em interpretador novo (`undeclared name not builtin:
  long` — código pensado pra Python 2, trecho de compatibilidade
  desatualizado). `postinst` abortou no meio, pacote ficou marcado
  como não configurado no dpkg
- **Efeito colateral bom**: como a falha aconteceu ANTES da etapa de
  reiniciar o serviço no `postinst`, o painel continuou rodando
  normalmente na versão anterior — não derrubou nada
- Corrigido: `gevent==24.2.1` → `gevent==25.5.1`, que já publica wheel
  pronta pra `cp313-manylinux2014_x86_64` (Python 3.13 no Debian
  x86_64) — instala direto, sem precisar compilar nada

### 0.36.0 — Endpoint `/api/v1/eventos` (pendente do app companion arx-monitor)
- **`GET /api/v1/eventos`** — mesma autenticação por token da API
  já existente, dois modos na mesma rota:
  - `?desde=<timestamp unix>` → resposta única (polling): tudo que
    aconteceu depois desse instante. Pensado pro modo "Economia" do
    app (polling periódico via WorkManager)
  - sem `desde` → SSE (Server-Sent Events), conexão fica aberta e
    empurra evento novo assim que aparece. Pensado pro modo
    "Instantâneo" do app e pra versão desktop (conexão persistente)
- **Gunicorn trocado pro worker `gevent`** (antes: síncrono, 2
  workers fixos) — com worker síncrono, cada conexão SSE aberta
  prenderia um worker inteiro pelo tempo todo que ficasse conectado;
  com só 2, dois clientes do app já travariam o painel pra qualquer
  outro acesso. gevent permite um worker lidar com centenas de
  conexões simultâneas via cooperação, sem mudar o resto do código —
  monkey-patch é feito pelo próprio worker do gunicorn na
  inicialização
- **Arquitetura mantida**: mesmo padrão do resto do projeto — webui
  nunca lê arquivo sensível direto, mesmo pra isso (`alertas.jsonl` é
  escrito pelo helper privilegiado). Ação nova
  `alertas.listar_desde(desde)` no helper, chamada via socket a cada
  2s de dentro do generator do SSE — sem esse cuidado, seria preciso
  dar acesso de leitura de arquivo direto pro processo sem
  privilégio, quebrando o isolamento
- `X-Accel-Buffering: no` no header da resposta — sem isso, o nginx
  bufferizaria o stream inteiro antes de entregar, matando a
  vantagem do SSE (chegaria tudo de uma vez só, não em tempo real)
- Testado: polling filtra corretamente por timestamp, `desde`
  inválido recusado (400), sem token recusado (401), modo SSE
  responde com mimetype/headers corretos
- **Pendente pro lado do app** (fora do escopo do painel): consumir
  esse endpoint de verdade no Flutter

### 0.35.2 — Tolerância de boot pro NTP (nível "info" usado pela 1ª vez)
- **Relatado pelo usuário**: NTP demorava pra sincronizar depois do
  boot e disparava alerta crítico + notificação, que sumia sozinho
  minutos depois — ruído que pode fazer alerta de verdade passar
  despercebido com o tempo
- **NTP com menos de 5 minutos de boot e ainda não sincronizado**
  agora usa o nível "info" ("iniciando — sincronizando desde o boot
  (Ns atrás), normal nos primeiros minutos") em vez de "critico" —
  só escala pra crítico de verdade se continuar sem sincronizar
  depois desse prazo
- **"info" nunca tinha sido usado de verdade** no sistema de 6
  níveis (existia desde o início, só não tinha caso de uso ainda) —
  `alertas.py` ajustado pra não gerar alerta nem notificação pra
  esse nível (mesma lógica de "ok"), matching a definição original
  ("informativo, sem ação corretiva"). O badge visual (azul, "INFO")
  já existia pronto no macro compartilhado, nunca exercitado até
  agora
- Sincronizar DENTRO da janela de tolerância (info → ok) não gera
  "recuperado" nem notificação — nunca foi alertado como problema,
  não faz sentido "recuperar" de nada
- Testado: fronteira exata da janela (299s ainda info, 301s já
  critico se continuar sem sincronizar), ciclo completo de
  transições (ok→info→critico→ok), e o caso específico de sincronizar
  a tempo (info→ok sem alarme)

### 0.35.1 — Bug real: página de espera ficava sem CSS/JS (corrida de verdade)
- **Achado testando em produção**: a página de espera da 0.35.0
  carregava sem estilo nenhum e sem funcionalidade de reconexão —
  causa real: CSS/JS externos são requisições SEPARADAS depois do
  HTML principal, e o servidor começa a desligar serviços (nginx
  incluso) quase imediatamente após o reboot ser disparado. A
  corrida entre "navegador busca os arquivos separados" e "nginx cai"
  claramente perdia
- **Corrigido pra 100% autossuficiente**: CSS e JS agora embutidos
  na PRÓPRIA resposta HTML (`<style>` e `<script>` inline) — zero
  requisição adicional depois da primeira, não importa quão rápido o
  servidor comece a desligar depois
- Isso quebra a regra geral do projeto (nunca inline, sempre CSP
  restrito) de propósito, só nessas duas rotas — nginx ganhou um
  `location` específico pra `/sistema/reiniciar` e
  `/sistema/desligar` com CSP mais permissivo
  (`script-src 'unsafe-inline'`) só ali; `location` mais específico
  não herda os headers do `server{}`, então os outros (HSTS,
  X-Frame-Options, etc) foram repetidos explicitamente
- Removido o JS/CSS externo criado na 0.35.0 (agora obsoletos,
  substituídos pelo inline)
- Testado: resposta HTTP real confirma ZERO referência a arquivo
  externo (nem `style.css`, nem `arx-icon-mark.svg`, nem `.js`) nos
  dois modos

### 0.35.0 — Página de espera ao reiniciar/desligar
- **Antes**: clicar em reiniciar/desligar redirecionava pro
  dashboard, que precisa do servidor já estar de pé pra renderizar —
  resultado, o navegador caía numa página quebrada durante a janela
  em que o servidor estava fora do ar
- **Agora**: a própria resposta do clique já entrega uma página de
  espera autossuficiente (sem precisar de outra requisição depois).
  No modo reiniciar, um JS externo (`reiniciando.js` — CSP não
  permite inline) fica tentando reconectar em `/login` a cada poucos
  segundos, e redireciona sozinho assim que o servidor volta a
  responder. No modo desligar, mensagem final sem tentativa de
  reconexão (não vai voltar sozinho)
- Sessão anterior não sobrevive a um restart do serviço (chave de
  sessão aleatória por padrão) — a página manda pro login de
  propósito, não tenta voltar pro dashboard direto
- Testado: resposta HTTP real do POST já vem com a página de espera
  (200, sem redirect), sem script inline, JS balanceado, os dois
  modos renderizando

### 0.34.9 — Bug real: corrida de boot do isc-dhcp-server acontecendo de novo
- **Achado testando depois do reboot** (não relacionado ao AppArmor
  — confirmado sem nenhum `DENIED` no log do kernel pros perfis do
  painel): `isc-dhcp-server` falhou ao iniciar com "No subnet
  declaration... Not configured to listen on any interfaces!" — a
  interface ainda não tinha IP na hora que o dhcpd tentou subir
- O drop-in `arx-espera-rede.conf` (que já existia justamente pra
  evitar isso) dependia só de `network-online.target` — esse alvo
  pode ser considerado "pronto" sem que a interface específica já
  tenha IP de verdade, dependendo de como o
  `systemd-networkd-wait-online` está configurado. Garantia fraca
  demais, falhou nessa reinicialização
- Reforçado com dependência explícita em
  `systemd-networkd-wait-online.service` (mais forte que o alvo
  genérico) **e** uma checagem ativa (`ExecStartPre`) que espera até
  30s por IPv4 de verdade em alguma interface antes do dhcpd tentar
  subir — sem fixar nome de interface (funciona em qualquer
  instalação, não só nessa VM). Se esgotar os 30s, loga aviso e
  segue mesmo assim (não trava o serviço pra sempre)
- Nenhuma mudança de AppArmor envolvida — o `ExecStartPre` roda via
  `/bin/sh` fora dos perfis do painel

### 0.34.8 — CORREÇÃO URGENTE: bugs reais de verdade, achados com enforce mode ativo
- **Bug real (erro de edição anterior)**: `/sbin/ethtool Ux,` nunca bateu de
  verdade — `/sbin` é link simbólico pro `/usr/sbin` (usrmerge do
  Debian), o AppArmor casa pelo caminho que o KERNEL resolve
  (`/usr/sbin/ethtool`), não pelo link. Em complain mode isso ficava
  mascarado (permitia mesmo sem bater a regra); virou bloqueio de
  verdade assim que passou pra enforce. Mesmo problema em
  `/bin/ping` (faltava `/usr/bin/ping`). Corrigido adicionando o
  caminho `/usr/` correspondente nos dois — revisei TODAS as outras
  regras `Ux` do perfil procurando o mesmo padrão; as demais já
  tinham os dois caminhos cobertos
- **`aa-status` faltava por completo** — usado por `apparmor.status`
  (a página de AppArmor do próprio painel), nunca tinha sido
  declarado. Adicionado `/usr/sbin/aa-status Ux,`
- **`/etc/arx/grupos.json` (mknod negado)** — grupos locais nunca
  tinham cobertura, só `/etc/arx/canal` (usado por updates.py)
  estava coberto. Generalizado pra `/etc/arx/** rwk,` (cobre os dois
  arquivos existentes e qualquer um futuro)
- Testado: sintaxe validada com `apparmor_parser -Q`
- **Se você já rodou `aa-complain` nos dois binários pra contornar o
  bloqueio**: essa atualização também restaura o enforce mode
  automaticamente (o pacote reescreve o perfil e recarrega via
  `apparmor_parser`), não precisa reativar manualmente

### 0.34.7 — Última lacuna de AppArmor + migração pra ENFORCE MODE
- **Última lacuna achada** (mesmo padrão de sempre): `/etc/systemd/network/` (nó do diretório em si), no helper
- **Script de teste (61 ações) rodado de novo: 0 falhas reais.** Log
  revisado mais uma vez: tudo coberto, só ruído esperado
  (`aa-status`/`ethtool`)
- **MIGRAÇÃO PRA ENFORCE MODE** — depois de 4 rodadas de revisão de
  log real de produção + teste sistemático de 61 ações de leitura +
  uso real acumulado ao longo de vários dias, os dois perfis
  (`painel-helper`, `painel-webui`) saem de complain e passam a
  **bloquear de verdade** qualquer coisa fora do que já foi mapeado
- **Isso é uma mudança de comportamento real, não só documentação.**
  Se algo escapou da validação (principalmente em ações de
  ESCRITA — VLAN, firewall, DNS, domínio — que foram menos
  exercitadas que as de leitura pelo script), vai bloquear em vez de
  só logar
- **Rollback instantâneo se algo quebrar**, sem precisar reinstalar
  pacote nenhum:
  ```bash
  aa-complain /opt/painel/helper/helper.py
  aa-complain /opt/painel/webui/venv/bin/gunicorn
  ```
  Isso volta os dois pra complain na hora (mudança em memória,
  sobrevive até o próximo reload do perfil — não precisa reiniciar
  serviço nenhum). Depois é só me mandar o log do que bloqueou
- Testado: sintaxe validada com `apparmor_parser -Q` nos dois em modo
  enforce, confirmado que não sobrou nenhum `flags=(complain)` em
  nenhum dos dois arquivos

### 0.34.6 — Mais 2 lacunas reais de AppArmor (terceira rodada) + ponto arquitetural anotado
- **2 lacunas novas, só apareceram depois do serviço reiniciar com o
  pacote mais recente**:
  - `/proc/*/cgroup` e `/sys/fs/cgroup/system.slice/cpu.max` (webui)
    — gunicorn/Python checando limite de CPU via cgroup v2
    (decide quantos workers rodar)
  - `/var/backups/painel/*.tar.gz` (webui) — o botão "Baixar" da
    página de Backups lê o arquivo direto (`send_from_directory`),
    nunca tinha sido coberto no perfil do webui
- **Ponto arquitetural anotado, não corrigido agora**: o download de
  backup lê o arquivo direto no webui, sem passar pelo helper — foge
  do padrão "webui nunca toca arquivo sensível direto, só o helper
  privilegiado" usado no resto do projeto. Não é uma mudança de
  postura introduzida agora (a regra só alcança o que já existia,
  senão quebraria em enforce mode), mas fica registrado como melhoria
  futura: rotear esse download pelo helper também
- Testado: sintaxe validada com `apparmor_parser -Q`, perfil completo
  passa limpo

### 0.34.5 — Lacuna de recuperação fechada: admin agora revoga token de outro
- **Achado testando fluxos de recuperação**: MFA já tinha
  recuperação assistida por outro admin (`/admins`), mas token de
  API não — se vazasse e a pessoa não estivesse disponível, ninguém
  mais conseguia revogar. `auth.api_token_revogar()` já aceitava
  qualquer username como parâmetro desde que foi escrita — só
  faltava uma rota/página admin-only expondo isso, a limitação
  estava só no `app.py`
- **Página nova `/admins/<username>/tokens`** (`@requer_admin`) —
  outro admin vê apelido/data/último uso dos tokens de qualquer
  conta e pode revogar. Texto puro do token nunca aparece aqui, nem
  pra admin nenhum — mesma garantia de sempre
- Link "Ver/revogar" novo na tabela de `/admins`
- Testado: admin revoga token de outra conta com sucesso (token
  realmente para de autenticar depois), papel leitura é bloqueado
  tentando acessar essa página (RBAC de verdade, não só escondido na
  UI)

### 0.34.4 — Mais 4 lacunas reais de AppArmor, achadas na segunda rodada de log
- **Ruído identificado e descartado** (não precisa de regra nova):
  `aa-status` rodado na mão pelo usuário e a transição `Ux` do
  `ethtool` — as duas aparecem no log como bookkeeping normal do
  kernel durante transição pra unconfined, não são gap real
- **4 lacunas reais corrigidas**:
  - `/var/backups/painel/` (nó do diretório em si, mesmo padrão já
    visto — `**` não cobre)
  - `/opt/painel/webui/** r,` → `mr,` — bibliotecas `.so` das
    dependências Python (`cbor2`, `cryptography`, `markupsafe`)
    precisam de "m" (mapear em memória como executável) pra carregar
    a extensão nativa, "r" sozinho não bastava
  - `/proc/*/stat` (webui) — processo se inspecionando (uso de
    CPU/memória)
  - `/usr/share/terminfo/d/dumb` e `/etc/inputrc` (webui) — Python
    checando capacidade de terminal ao rodar sem TTY interativo
    (normal via systemd)
- Testado: sintaxe validada com `apparmor_parser -Q` de verdade nos
  dois perfis completos, ambos passam limpo

### 0.34.3 — Timestamp legível no Dashboard e Logs
- **"Eventos recentes" (Dashboard) e Logs** mostravam o timestamp cru
  da auditoria (ISO-8601 UTC com microssegundos, tipo
  "2026-09-20T22:17:51.900177+00:00") — mesmo espírito do que já foi
  ajustado antes pro uptime. Agora mostra em hora local, formato
  curto ("2026-09-20 22:17:51"), mesmo padrão já usado em Alertas e
  no resto do painel
- Função nova `_formatar_ts_auditoria()` — reutilizável, com reserva
  segura pro caso de o timestamp vir num formato inesperado (mostra o
  valor cru em vez de quebrar a página)
- Testado: formatação correta, valor inválido não derruba nada,
  templates renderizando

### 0.34.2 — Última lacuna do AppArmor fechada, com validação real pela primeira vez
- **`aa-logprof` não funciona nesse sistema** — depende de
  `/var/log/syslog` (rsyslog clássico), que não existe aqui (só
  journal do systemd). Resolvido sem precisar dele: os dados exatos
  já estavam no log que a gente já tinha revisado junto
- **Corrige a regra de `/tmp` da 0.34.1, que estava com sintaxe
  ERRADA** (`rwcd` não existe — instalei o `apparmor-utils` nesse
  ambiente e validei com o `apparmor_parser` de verdade pela
  primeira vez nesse projeto, não só contagem de chaves como antes.
  As letras "c"/"d" no log são como o kernel categoriza a operação,
  não a sintaxe de regra do perfil). Corrigido pra `/tmp/* rw,`
  (sintaxe validada), que já cobre criar/escrever/apagar arquivo
  comum — "capability mknod" só seria necessário pra dispositivo de
  verdade, não é o caso dos temporários do gunicorn
- **Validação real confirma retroativamente todas as adições de
  AppArmor da sessão inteira** — os dois perfis completos (com tudo
  acumulado: métricas de DNS, notificações, VLAN, MFA, WebAuthn, API,
  etc) passam limpo no `apparmor_parser -Q`, não só no balanceamento
  de chaves usado até agora
- Com isso, as duas lacunas conhecidas de AppArmor (achadas revisando
  o log real de produção) estão fechadas — modo complain segue ativo,
  mas sem gap identificado pendente pra considerar enforce mais pra
  frente

### 0.34.1 — Lacunas reais de AppArmor, achadas revisando o log de produção
- **Primeira revisão de verdade do log de complain mode desde que os
  perfis existem** — os dois (`painel-helper`, `painel-webui`) rodam
  em complain há toda a Fase 2+5, nunca bloqueando nada, só
  registrando. Com tanta coisa nova testada nessa sessão, era a hora
  certa de conferir lacunas antes de cogitar enforce
- **Bug real encontrado**: o perfil cobria `/sys/block/*/stat`
  (link simbólico), mas o acesso de verdade resolve pro caminho
  físico via virtio (`/sys/devices/pci.../virtio3/block/vda/stat`)
  — em modo enforce, isso bloquearia a leitura de disco do
  dashboard. Corrigido com curinga (`/sys/devices/**/block/*/stat`),
  sem depender do endereço de barramento PCI específico dessa VM
  (varia por hardware)
- Lacunas adicionais achadas no log real, todas corrigidas:
  `/proc/uptime` e `/proc/*/net/dev` (não cobertos pela abstração
  base do AppArmor, usados por `system.resource_status`), leitura
  do nó do próprio diretório (`/opt/painel/helper/` e
  `/opt/painel/webui/` — `**` não cobre o diretório em si, só o que
  tem dentro), `/etc/mime.types` (gunicorn detectando tipo de
  arquivo estático), capability `mknod` (probe opcional do próprio
  interpretador Python, benigna)
- **Pendente, não corrigido ainda de propósito**: arquivos
  temporários do gunicorn em `/tmp` (operação `mknod` de verdade,
  não só probe) — a permissão exata é sutil o bastante pra preferir
  deixar o `aa-logprof` gerar a regra certa a partir do log real, em
  vez de eu adivinhar os bits de permissão
- Testado: chaves balanceadas nos dois perfis

### Fase 5 — item 6/6 (Plugins): adiado pra pós-1.0, decisão consciente
- **Não implementado agora, de propósito** — plugin é código que roda
  com privilégio real (via helper), e fazer isso com segurança de
  verdade (isolamento, modelo de permissão, manifesto validado,
  plugin manager) é um projeto à parte, bem maior que os outros 5
  itens da Fase 5 (que reaproveitaram RBAC/helper/ações já
  existentes). Uma versão "rápida e suja" (importar `.py` solto com
  acesso total ao helper) seria pior que não ter plugin nenhum —
  falsa sensação de segurança
- Análise detalhada trazida pelo usuário (arquitetura de 3 camadas —
  Core / Módulos oficiais / Plugins isolados via API controlada,
  manifesto declarando permissões/dependências, 3 categorias de
  plugin — oficial/comunitário/interno) concorda com essa decisão e
  fica registrada como referência pra quando isso for retomado
- **Ponto forte confirmado**: a arquitetura atual (webui sem
  privilégio ↔ socket Unix ↔ helper privilegiado, tudo por ação
  nomeada em `ACTIONS`) já É a fundação certa pra plugins de verdade
  mais tarde — nada precisa ser refeito, só estendido
- Mesmo padrão de outros itens adiados conscientemente no projeto
  (RAID, PXE, SNMP, Prometheus — ver Fase 2)

### Fase 5 do painel — fechamento: 5/6 implementados (RBAC, MFA,
### Notificações, WebAuthn, API completa), 1/6 adiado (Plugins,
### pós-1.0, decisão documentada acima)

### 0.34.0 — Fase 5 do painel (item 5/6): API completa (por token)
- **`POST /api/v1/executar`** — endpoint único, genérico, autenticado
  por token (`Authorization: Bearer <token>`) — em vez de recriar
  rota REST por recurso (trabalho enorme e duplicado pras 130+
  ações que já existem), expõe as MESMAS ações que o webui já usa
  internamente: `{"acao": "system.resource_status", "params": {}}`.
  Decisão de escopo confirmada com o usuário antes de implementar
- **Token herda o papel de quem criou** (admin/leitura) — reaproveita
  o RBAC já existente sem nenhum código novo de permissão: o mesmo
  `handle_request` do helper que bloqueia ação mutante pra papel
  leitura via webui bloqueia igual via token de API. Testado
  confirmando o bloqueio de verdade
- **`auth.py`** ganhou gerenciamento de tokens (`api_token_criar`,
  `_listar`, `_revogar`, `_verificar`) — token é SHA-256 (não o hash
  lento tipo senha — tokens já são aleatórios de alta entropia,
  hash rápido é o certo aqui e permite achar o dono rápido). Texto
  puro do token só existe uma vez, na hora de criar, nunca mais
  recuperável depois — nem o painel guarda
- **Página `/api-tokens`** — criar (com apelido), listar (com
  último uso), revogar. Exemplo de uso com `curl` direto na página
- **CSRF ajustado** — rotas `/api/` isentas da checagem de sessão
  (token no header já é a autenticação; não existe cookie de sessão
  nem risco de CSRF nesse modelo)
- `helper_client.call()` ganhou `_contexto_override` — permite
  informar usuário/papel manualmente (a API não tem sessão do
  Flask-Login pra tirar isso automaticamente, diferente do resto do
  painel)
- Testado: ciclo completo de token (criar/listar/verificar/último
  uso atualizado/revogar/recusa de inválido), fluxo HTTP real
  ponta a ponta com cliente SEM sessão nenhuma (só o token), recusa
  de token ausente/inválido/corpo malformado, e RBAC bloqueando
  token de papel leitura numa ação mutante — igual bloquearia pela
  interface

### 0.33.2 — Documentação: exigência de certificado pro WebAuthn
- **CONFIRMADO EM PRODUÇÃO**: WebAuthn funcionando de ponta a ponta,
  em conjunto com TOTP — usuário escolhe qual método usar na hora do
  login quando tem os dois configurados. Fecha o item 4/6 da Fase 5
  com validação real, não só testes automatizados
- Aviso adicionado na página `/seguranca`, junto da seção de chaves
  de segurança: WebAuthn exige certificado HTTPS confiável (não
  autoassinado) e acesso por hostname (não por IP) — restrição do
  próprio protocolo, não do painel. Documentado ali porque é
  exatamente onde quem for configurar precisa ver isso

### 0.33.1 — Bug real corrigido: WebAuthn recusava por causa da porta sumindo no proxy
- **Causa raiz, achada testando ao vivo**: o nginx usava
  `proxy_set_header Host $host;` — a variável `$host` do nginx
  **sempre remove a porta** por definição. O Flask, atrás do proxy,
  nunca via a porta 9006 de verdade, então `request.host_url`
  reconstruía a URL assumindo a porta padrão (443) e omitia ela —
  virava "https://localhost" em vez de "https://localhost:9006". O
  navegador manda o origin completo com porta, a lib WebAuthn recusa
  a diferença (proteção correta contra origin forjado, só que os dois
  lados precisam bater)
- Corrigido: `$host` trocado por `$http_host` (preserva o Host header
  exatamente como o navegador mandou, porta incluída) — variável
  padrão do nginx pra esse tipo de situação
- `postinst` já recarrega o nginx automaticamente
  (`nginx -t && systemctl reload nginx`), não precisou de mudança
  adicional
- **Jornada até aqui, documentada porque pode ajudar quem mais testar
  WebAuthn nesse ambiente**: acessar por IP dá "invalid domain"
  (WebAuthn exige hostname de verdade) → acessar por hostname sem
  DNS configurado precisa de entrada no hosts do cliente → certificado
  autoassinado dá "TLS certificate errors" (WebAuthn recusa cert não
  confiável) → túnel SSH pra `localhost` contorna isso (exceção
  especial do navegador) → mas aí a porta sumia no proxy, esse bug.
  Com essa correção, o túnel SSH pra localhost deve funcionar de
  ponta a ponta agora

### 0.33.0 — Fase 5 do painel (item 4/6): WebAuthn (chave de segurança/biometria)
- **Segundo fator alternativo ao TOTP** — chave física (YubiKey,
  etc) ou biometria da plataforma (Windows Hello, Touch ID). Se a
  conta tiver os dois configurados, escolhe qual usar na hora de
  logar
- **Biblioteca `webauthn` (py_webauthn)** via pip/venv — diferente do
  TOTP (fórmula simples, RFC bem definida), WebAuthn envolve
  verificação de assinatura, parsing CBOR/COSE e attestation —
  criptografia complexa de mais pra reimplementar na mão com
  segurança, então usei biblioteca testada. Dependências leves
  (`cbor2`, `cryptography`, `pyasn1`, `pyOpenSSL`) — nada como Pillow
- **`auth.py`** ganhou registro/listagem/remoção de credenciais
  (`webauthn_iniciar_registro`, `webauthn_confirmar_registro`,
  `webauthn_listar_credenciais`, `webauthn_remover_credencial`) e
  autenticação (`webauthn_iniciar_autenticacao`,
  `webauthn_verificar_autenticacao`, com contador anti-clone
  atualizado a cada login)
- **Página `/seguranca`** ganhou seção de chaves — adicionar (com
  apelido), listar, remover
- **Login em `/login/mfa`** mostra as opções disponíveis — só TOTP,
  só WebAuthn, ou os dois com escolha
- **rp_id/origem calculados dinamicamente** por servidor
  (`request.host_url`) — cada Arx OS usa o próprio hostname, sem
  precisar configurar nada fixo
- JS novo (`webauthn-common.js`, conversões base64url↔ArrayBuffer que
  a API do navegador exige; `webauthn-registro.js`;
  `webauthn-login.js`) — nenhum inline, tudo externo (CSP)
- **Bug real corrigido durante o desenvolvimento**: os decorators
  `@app.route("/seguranca")` `@login_required` ficaram grudados na
  função errada numa edição (`_webauthn_rp_id` em vez de
  `seguranca_pagina`) — `/seguranca` sumiu do mapa de rotas, e a
  função auxiliar virou sem querer uma rota protegida por login, o
  que quebrava ela quando chamada durante o fluxo de login (usuário
  ainda não autenticado). Achado e corrigido antes de empacotar,
  confirmado com teste HTTP completo
- **CSRF ajustado pra aceitar token no corpo JSON também** — a
  checagem global só olhava formulário, e os endpoints novos mandam
  JSON
- Testado: tudo que dá pra testar sem navegador real (geração de
  opções, armazenamento, remoção, user_id estável, proteção
  anti-clone, recusa de credencial desconhecida, erro de verificação
  não derruba o processo) e o fluxo HTTP completo de login com
  verificação criptográfica mockada (register→listar, login com
  TOTP+WebAuthn juntos, só WebAuthn, sessão expirada). **A
  verificação criptográfica de verdade (attestation real de
  navegador/chave) só se confirma testando ao vivo** — não dá pra
  fabricar isso em teste automatizado sem um navegador de verdade

### 0.32.5 — Bug real corrigido (o de verdade): thread morta junto com o script
- **Causa raiz final, depois de investigação extensa com testes
  reais**: `verificar_alertas.py` (script do timer, `Type=oneshot`)
  chama `alertas.verificar_e_registrar()`, que dispara a notificação
  numa thread em background — mas diferente do `helper.py` (processo
  que fica vivo pra sempre), o script termina e o **processo Python
  inteiro morre** logo em seguida. Quando um processo Python morre,
  TODAS as threads (mesmo `daemon=True`) morrem junto, mesmo no meio
  de um envio de e-mail/webhook ainda em andamento. Por isso a
  notificação só funcionava quando disparada por uma página do
  painel (o processo do helper continua vivo depois) — nunca quando
  disparada pelo timer isolado
- Corrigido: `alertas.py` agora rastreia as threads de notificação
  disparadas (`_threads_notificacao`) e ganhou
  `aguardar_notificacoes_pendentes(timeout=15)` — chamado pelo
  `verificar_alertas.py` logo depois de `verificar_e_registrar()`,
  dá até 15s pra qualquer notificação em andamento terminar antes do
  processo poder sair. O caminho do webui continua sem chamar isso
  (helper.py não precisa, fica vivo pra sempre) — não trava nada
- De quebra: `verificar_alertas.py` e `coletar_metricas.py` agora
  configuram logging básico — sem isso, qualquer log dentro de
  `commands/` chamado por esses scripts ia pra lugar nenhum visível
  (helper.py configura isso só pra si mesmo, processo separado não
  herda). Vai pro journal do próprio serviço agora
- Testado: cenário exato do bug (processo "morrendo" logo depois de
  `verificar_e_registrar()`, sem a espera, notificação não termina) e
  a correção (com a espera, termina certinho); caminho do webui
  confirmado continuando instantâneo, sem regressão

### 0.32.4 — Bug real corrigido: timer de 10s não disparava no ritmo certo
- **Causa raiz confirmada com dados reais**: os intervalos entre
  execuções do `arx-painel-alertas.timer` variavam de forma errática
  (14s, 29s, 46s, 60s...) em vez de bater 10s certinho. Não era
  rate-limiting do systemd (nenhuma mensagem de start-limit no
  journal) — era o `AccuracySec` **padrão** do systemd, que é
  **1 minuto**: um recurso de economia de energia que agrupa
  disparos de timers próximos, pra acordar o sistema com menos
  frequência. Faz sentido pra um timer de 5 minutos (como o de
  métricas), mas destrói completamente um timer de 10 segundos — o
  systemd "arredondava" o disparo dentro dessa janela larga de 1
  minuto
- Corrigido: `AccuracySec=1sec` adicionado ao
  `arx-painel-alertas.timer` — sem isso, nenhum ajuste de
  `OnUnitActiveSec` sozinho resolveria, por menor que fosse o valor
- Timer de métricas (5 min) não precisa dessa mudança — a imprecisão
  de até 1 minuto é irrelevante numa janela de 5 minutos

### 0.32.3 — Investigação fechada + checagem de alertas bem mais rápida
- **Mistério resolvido, testando com calma passo a passo**: não era
  bug nenhum, nem de notificação, nem de detecção. A checagem só
  acontece quando algo chama `alertas.verificar_e_registrar()` —
  timer periódico OU carregamento de página do painel. Como o timer
  rodava a cada 5 minutos (pensado pra métricas, não pra alertas), e
  o teste do usuário esperava só 10-15s sem tocar o painel, nenhum
  dos dois gatilhos tinha disparado ainda nesse intervalo curto —
  assim que uma página carregava, a checagem rodava na hora e a
  notificação disparava certinho. `firewall.status()` também
  confirmado correto (consulta a ruleset real do kernel via `nft`,
  não o status do serviço systemd — mais preciso, mas explica por que
  `systemctl stop nftables` sozinho nem sempre reflete no teste)
- **Checagem de alertas separada da coleta de métricas** — timer
  próprio novo (`arx-painel-alertas.timer`), rodando a cada **10
  segundos** (pedido explícito do usuário, avisado do trade-off: mais
  chamadas de sistema constantes, mas ainda leve). Métricas continuam
  em 5 minutos, não precisam ser tão frequentes
  - Script novo `verificar_alertas.py`, mesmo padrão de
    `coletar_metricas.py` (não passa pelo socket do helper — só
    leitura, sem privilégio necessário)
  - `coletar_metricas.py` não chama mais `alertas.verificar_e_registrar()`
    (evita checagem duplicada agora que tem timer dedicado)
  - `postinst` habilita o timer novo junto com o de métricas
- Testado: sintaxe, import, script novo importa `alertas` corretamente

### 0.32.2 — Diagnóstico: log de tentativa/sucesso (investigação em andamento)
- **Reportado pelo usuário**: notificação do "crítico" (serviço
  caindo) não chega, só a do "recuperado" chega. Suspeita
  arquitetural (fork matando a thread) descartada — `helper.py` usa
  `socketserver.UnixStreamServer` com `serve_forever()`, processo
  único e contínuo, threads de background não deveriam ser
  interrompidas por isso
- **Gap real encontrado**: o código só logava quando dava ERRO —
  nunca quando tentava ou quando dava certo. Sem isso, não tinha como
  saber, só olhando o log, se o disparo pro caso crítico sequer
  chegou a acontecer
- Adicionado log de tentativa e sucesso em cada etapa (disparo da
  thread, início do envio por canal, sucesso por canal) — fecha a
  lacuna de observabilidade, sem mudar nenhum comportamento
- Testado: cadeia completa de log aparecendo (disparo → tentativa →
  sucesso/falha), inclusive com caracteres especiais reais do payload
  (ç, ã, travessão) sem quebrar nada
- **Investigação ainda em aberto** — essa versão é diagnóstica, não a
  correção final. Precisa do log real do próximo teste (parar o
  serviço, sem restaurar ainda) pra confirmar se a thread é disparada
  pro caso crítico e, se for, o que acontece com ela

### 0.32.1 — Bug real corrigido: notificação nunca disparava em uso real
- **Causa raiz confirmada, testando de propósito (parar o firewall)**:
  `alertas.verificar_e_registrar()` é chamado em dois lugares — pelo
  script periódico E por toda página do webui (pro sininho de
  alertas no topo). Só o disparo de notificação estava ligado ao
  script periódico, mas o ESTADO de "isso é novo" é compartilhado
  entre os dois — como o webui é chamado com muito mais frequência,
  era bem provável que uma página carregada logo depois do problema
  "roubasse" a detecção do alerta como novo antes do timer perceber.
  O alerta ficava registrado certinho no histórico, mas a notificação
  nunca disparava, porque quando o timer finalmente rodava o alerta
  já não era mais "novo"
- **Corrigido**: notificação agora dispara de dentro do próprio
  `verificar_e_registrar()`, em QUALQUER lugar que detecte o alerta
  como novo — webui ou timer, o que acontecer primeiro. Roda numa
  thread em background (`daemon=True`), nunca trava a página nem o
  script que chamou (email/webhook continuam sendo I/O lenta, mas
  agora isso não é mais um risco pra requisição do webui, já que
  dispara e segue em frente na hora)
- Removida a chamada duplicada que existia em `coletar_metricas.py`
  (ficaria notificando duas vezes se não removida)
- Testado: `verificar_e_registrar()` retorna quase instantâneo mesmo
  com notificação simulada lenta (1.5s), e principalmente — **o
  cenário exato do bug relatado**: duas chamadas próximas (simulando
  webui + timer) só disparam notificação uma vez, na primeira

### 0.32.0 — Fase 5 do painel (item 3/6): Notificações (e-mail/webhook)
- **Notifica por e-mail (SMTP) e/ou webhook (POST JSON)** toda vez
  que aparece um alerta novo — página `/notificacoes` com os dois
  canais, cada um pode ser ativado independente
- **Envio SEMPRE pelo script periódico**
  (`coletar_metricas.py`/timer), **nunca** pela requisição
  interativa do webui — email/webhook são I/O de rede lenta e não
  confiável, não podia travar carregamento de página de ninguém só
  porque um alerta apareceu
- **Canais isolados**: um SMTP fora do ar não impede o webhook de
  disparar (e vice-versa) — cada um falha e loga por conta própria,
  nunca propaga pro outro
- **Senha SMTP criptografada em disco** (Fernet) — generalizei o
  padrão que só existia pra credencial de serviço do DNS
  (`domain.py`) num utilitário reusável em `_util.py`
  (`criptografar_segredo`/`descriptografar_segredo`). Campo de senha
  em branco no formulário mantém a senha já salva (não precisa
  redigitar toda vez que só quer trocar o destinatário)
- Botões de "testar" pros dois canais — usam a config já salva,
  mostram o erro real na tela (não é silencioso como o disparo
  automático)
- `alertas.verificar_e_registrar()` agora devolve a lista de alertas
  novos também, não só a contagem (nenhum lugar dependia do formato
  antigo, verificado antes de mudar)
- Testado: cripto/descripto do segredo, config persistindo (inclusive
  preservando a senha ao atualizar só outro campo), isolamento entre
  os dois canais (webhook dispara mesmo com email falhando), recusa
  clara ao testar sem configurar antes

### 0.31.4 — QR code centralizado + uptime legível
- **QR code descentralizado corrigido** — o SVG vem com dimensão fixa
  em `mm` (não se adapta sozinho ao container), e a URI ficou mais
  longa depois de incluir o hostname (0.31.3), gerando um QR code
  maior que passou a estourar o `max-width` do jeito que estava.
  Corrigido com CSS dedicado (`.qrcode-mfa`) que força o SVG a 100%
  do container e centraliza com flexbox — resistente a mudanças
  futuras de tamanho do QR code
- **Uptime em formato legível** — antes sempre em horas decimais
  ("45.7h"), agora "1 dia, 21 horas". Nova função central
  `_util.formatar_duracao()` (dias/horas/minutos/segundos, só as duas
  maiores unidades, evita listas longas tipo "1 ano, 2 meses, 3 dias,
  4 horas"), reutilizável por qualquer módulo que precise mostrar
  duração pro admin. Aplicado em `system.resource_status()`
  (`uptime_horas` → `uptime_legivel`) e no relatório em texto
- Varredura no resto do projeto não achou outro lugar precisando do
  mesmo tratamento — certificados (dias até expirar) e período das
  métricas de DNS (dias) já usam a unidade certa pro contexto deles
- Testado: formatação com vários valores (incluindo o caso real
  relatado, 45.7h), sem import circular, templates renderizando

### 0.31.3 — Hostname no rótulo do MFA
- **Rótulo do QR code agora inclui o hostname** — antes era só
  "Arx OS: usuário", que funciona bem com um servidor só, mas quem
  administra vários servidores Arx OS no mesmo app de autenticação
  não tinha como diferenciar as entradas (reportado pelo usuário).
  Agora fica "Arx OS (hostname): usuário" — cada servidor aparece
  identificado, mesmo com múltiplas entradas no mesmo app
- `platform.node()` chamado direto no webui (sem privilégio
  necessário, mesmo raciocínio já usado pra gerar senha provisória —
  não precisa passar pelo helper pra isso)
- Testado: geração do QR code com o novo rótulo

### 0.31.2 — Corrigido: qrcode via pip/venv, não pacote Debian
- **Erro na 0.31.1**: a dependência `python3-qrcode` foi adicionada
  como pacote Debian, mas esse pacote traz `python3-pil` (Pillow) junto — puxa
  ~20 bibliotecas de imagem (libjpeg, libtiff, libwebp, harfbuzz,
  etc), 22 pacotes ao todo, só pra gerar SVG (que não precisa de
  Pillow nenhum). O projeto já tinha o padrão certo pra isso
  (`/opt/painel/webui/venv` + `requirements.txt` via pip), eu só não
  usei
- Corrigido: `qrcode` (sem `[pil]`) no `requirements.txt` — pip
  confirma zero dependências obrigatórias (`Requires:` vazio), só
  puxa a biblioteca em si, nada de Pillow
- Tirado o `python3-qrcode` do `Depends` do `deb/control`
- **Se você já adicionou `python3-qrcode` ao filtro do
  `arx-main-mirror`, pode reverter** — não é mais necessário, e
  mantém o mirror mais enxuto (22 pacotes a menos, alinhado com o
  espírito do projeto de minimizar o que entra no sistema)
- Testado: SVG gera normalmente sem Pillow instalado no ambiente

### 0.31.1 — QR code pra configurar MFA
- **QR code no lugar de só a chave em texto** — reportado pelo
  usuário testando: o "O" (letra) e "0" (número) na chave manual
  eram fáceis de confundir na hora de digitar. Agora escaneia com o
  app de autenticação, a chave em texto continua disponível como
  alternativa (caso o celular não tenha câmera disponível ali)
- **Dependência nova: `python3-qrcode`** — adicionada no
  `deb/control`. Gera o desenho como SVG inline (sem precisar de
  Pillow/processamento de imagem) — QR code tem correção de erro
  Reed-Solomon e outras partes complexas de mais pra reimplementar
  na mão com segurança, ao contrário do TOTP (fórmula simples,
  RFC 6238), então aqui usei biblioteca de verdade em vez de escrever
  do zero
  - **Atenção**: precisa confirmar que `python3-qrcode` está
    disponível no mirror local (`arx-security-mirror`/main) antes de
    instalar — é pacote padrão do Debian, mas ainda não confirmado no
    mirror desse ambiente especificamente
- Testado: geração do SVG, renderização no template, fluxo completo
  via requisição HTTP real (login → iniciar configuração → QR code
  aparece na página)

### 0.31.0 — Fase 5 do painel (item 2/6): MFA (TOTP)
- **Verificação em duas etapas via TOTP** — compatível com Google
  Authenticator, Authy, e qualquer app que siga o padrão (RFC 6238).
  Algoritmo implementado direto com `hmac`/`hashlib`/`struct` da
  biblioteca padrão do Python — sem dependência nova, e **testado
  contra os 5 vetores de teste oficiais da RFC antes de entrar em
  uso** (bate 100%)
- **Login em duas etapas**: `/login` (usuário+senha) → se a conta tem
  MFA ativo, não loga ainda — vai pra `/login/mfa` (pede o código de
  6 dígitos). Estado da etapa intermediária fica na sessão assinada
  do Flask (`SECRET_KEY`), não dá pra falsificar vindo do navegador;
  expira em 5 minutos se abandonado no meio
- **Página `/seguranca`** (qualquer usuário, papel admin ou leitura)
  — ativar mostra a chave em texto (sem QR code, mais simples,
  qualquer app aceita digitar na mão), confirma com um código antes
  de valer pra login de verdade (prova que sincronizou certo antes de
  exigir em todo acesso). Desativar pede a senha de novo
- **Recuperação**: outro admin pode desativar o MFA de alguém que
  perdeu o celular, pela página `/admins` (agora mostra status de MFA
  de cada conta)
- Testado: os 5 vetores oficiais da RFC 6238, ciclo completo
  (configurar → confirmar código errado recusa → código certo ativa →
  desativar), e o fluxo de login inteiro via requisição HTTP real
  (cookies de sessão + CSRF de verdade) — login não completa antes do
  código, código errado não loga, código certo loga, conta sem MFA
  continua em uma etapa só (sem regressão)

### 0.30.7 — CSP: scripts inline removidos (reintroduzidos por engano)
- **Bug real corrigido — página `/dominio` (e `/jobs/<id>`) quebrada
  pela CSP**: a política de segurança do painel
  (`default-src 'self'`, sem `unsafe-inline` pra script) bloqueia
  QUALQUER `<script>` inline — regra do projeto desde cedo (já
  corrigido antes, em 0.11.6/0.11.7), mas dois scripts inline foram
  reintroduzidos sem querer nas últimas features: o de acompanhar job
  (`/jobs/<id>`, saída ao vivo) e o de gerar senha provisória
  (`/dominio`, adicionado nessa mesma sessão)
- Corrigido: os dois movidos pra arquivos `.js` externos
  (`job-status.js`, `dominio-gerar-senha.js`) — o `job.id`, que antes
  vinha injetado direto no script via Jinja (`{{ job.id | tojson }}`,
  não dava pra externalizar do jeito que estava), agora vai como
  atributo `data-job-id` num elemento HTML, lido pelo JS externo
- Varredura completa confirmou que não sobrou nenhum `<script>`
  inline em template nenhum do projeto
- A parte de fonte externa (OpenSans via Google Fonts) mencionada
  junto no relato não foi encontrada nesse estado do projeto — não
  precisou de correção
- Testado: sintaxe, templates renderizando, JS balanceado, grep
  confirma zero scripts inline restantes

### 0.30.6 — Gerar senha provisória automática
- **Botão "Gerar senha provisória"** no formulário de criar usuário
  de domínio — preenche os dois campos de senha com uma senha
  aleatória que já atende à política de complexidade do AD (maiúscula,
  minúscula, número, caractere especial), revela o texto (deixa de
  ser `type="password"`) pra dar pra copiar e passar pro usuário
- Não existe "modo relaxado" pra senha provisória no Active
  Directory — confirmado em produção (o comando que falhou já incluía
  `--must-change-at-next-login` e mesmo assim foi rejeitado por
  complexidade). Gerar automaticamente é a forma de tirar esse
  trabalho do admin sem abrir mão da política do domínio
- Endpoint novo `/dominio/gerar-senha` — não passa pelo helper (não
  mexe no sistema, só gera uma string), usa `secrets` (aleatoriedade
  criptográfica) excluindo caracteres ambíguos (`l/I/O/0/1`) pra
  reduzir erro de transcrição na hora de repassar a senha
- Testado: senha gerada sempre tem as 4 categorias, sempre 12
  caracteres, cruzado contra a validação real do `domain.py`
  (30 gerações, todas passam)

### 0.30.5
- **Causa raiz confirmada**: o erro de "criar usuário" real que
  disparou toda essa sequência de correções era a política padrão de
  complexidade de senha do Active Directory — a senha testada só
  tinha letra minúscula + número (2 categorias), o AD por padrão
  exige pelo menos 3 de 4 (maiúscula, minúscula, número, caractere
  especial). Comportamento correto do Samba, não um bug
- **`domain.create_user` agora checa isso ANTES de chamar o
  `samba-tool`** — mensagem clara em português
  ("senha não atende a política de complexidade...") em vez do texto
  cru do LDAP (`0000052D: Constraint violation...`) que só aparecia
  depois de tentar de verdade
- Testado: a mesma senha que falhou em produção é pega na hora,
  senha com 3+ categorias passa

### 0.30.4 — Correção de segurança: senha em texto puro no log
- **Bug grave corrigido — senha aparecia em texto puro no
  `helper.log`**: a auditoria (`auditoria.jsonl`) já mascarava senha
  corretamente (`"password": "***"`), mas o log de INFO
  ("executando ação: X params=...") usava os parâmetros **crus**,
  sem passar pela mesma sanitização — toda vez que alguém criava um
  usuário (de domínio, local, ou administrador do painel), a senha
  ficava gravada em texto puro em disco, legível por qualquer um com
  acesso de leitura a esse arquivo
- Corrigido: o log de INFO agora usa a mesma função de sanitização já
  usada na auditoria (`_sanitizar_params`) — nenhum código novo,
  só reaproveitar o que já existia certo num lugar só e faltava no
  outro
- **Isso não apaga retroativamente o que já foi gravado antes da
  correção** — só evita gravação nova. Quem tiver log antigo com
  senha em texto puro precisa limpar/redigir manualmente
- Testado: senha não aparece mais no log gerado, `***` aparece no
  lugar

### 0.30.3
- **Bug real corrigido — criar/bloquear/desbloquear usuário de
  domínio mostrava só "erro interno" quando falhava**:
  `create_user`, `disable_user` e `enable_user` usavam
  `subprocess.run(..., check=True)` sem capturar o resultado — quando
  o `samba-tool` falhava (confirmado em produção: `samba-tool user
  create` retornou código 255), a exceção crua (`CalledProcessError`,
  que não inclui o `stderr` no texto por padrão) caía no tratamento
  genérico do `helper.py`, virando "erro interno" pro usuário — a
  mensagem REAL do Samba nem aparecia no log
- Corrigido: as três agora capturam o `stderr` de verdade e levantam
  `RuntimeError` com a mensagem real do `samba-tool` (mesmo padrão já
  usado no resto do arquivo via `_limpar_stderr_samba_tool`, só
  tinham ficado de fora)
- Testado: erro do `samba-tool` propaga com a mensagem real agora,
  não mais genérica

### 0.30.2 — RBAC: defesa em profundidade na interface
- **Papel "leitura" não vê mais botão nenhum que muda algo** — antes,
  o bloqueio existia só no backend (`handle_request`), mas a
  interface mostrava os botões normalmente, como se desse pra
  clicar. Ruim por dois motivos: confunde o usuário leitura, e
  depende de TODO formulário passar pelo caminho central de checagem
  (já achamos um caso real, `/admins`, que não passava — daí essa
  camada extra não ser opcional)
- Resolvido de forma central, sem mexer template por template: JS
  novo (`rbac-leitura.js`), carregado globalmente, troca o botão de
  qualquer formulário POST por um aviso "Somente leitura" e desabilita
  os campos — automático em qualquer página, atual ou futura
- Bloqueia o `submit` de verdade também (não só esconde o botão) —
  cobre outros jeitos de disparar o envio, tipo tecla Enter num campo
  de texto
- Testado: classe `papel-leitura` aparece no HTML só pra usuário com
  esse papel, JS balanceado

### 0.30.1
- **Confirmar senha ao criar usuário/admin** — faltava em dois
  formulários (usuário de domínio, administrador do painel); já
  existia em "Usuários locais", só não tinha sido replicado nos
  outros dois. Mesmo padrão (campo duplo + checagem no servidor antes
  de chamar a ação)
- **Menu lateral reorganizado em grupos expansíveis** — 26 links
  soltos viraram 5 grupos (Rede, Domínio e Contas, Sistema,
  Monitoramento, Administração) usando `<details>`/`<summary>` do
  HTML — colapsa/expande nativo do navegador, sem JS nenhum. O grupo
  da página atual abre sozinho; os outros ficam fechados até clicar.
  Cada ícone e link preservado exatamente como estava, só reagrupado
- Testado: sintaxe, templates renderizando com o menu novo (inclusive
  papel "leitura" continuando sem ver "Administradores"), `<details>`
  balanceado, CSS balanceado

### 0.30.0 — Fase 5 do painel (item 1/6): RBAC v1
- **Dois papéis**: `admin` (acesso total, comportamento de sempre) e
  `leitura` (só visualiza, qualquer ação que altere algo é recusada)
- **Bloqueio centralizado no `handle_request` do helper.py** — mesmo
  ponto único já usado pro modo manutenção, reaproveitando
  `_eh_acao_de_leitura()` que já existia. Evitou ter que adicionar
  checagem de permissão em cada uma das 113+ rotas do webui
- **`auth.py`** ganhou `criar_ou_atualizar_admin`, `remover_admin`,
  `list_admins`, `obter_role` — suporta formato antigo (hash direto,
  sempre "admin" implícito) e novo (`{hash, role}`) ao mesmo tempo,
  sem quebrar instalação existente
- **Proteção contra ficar sem admin**: recusa remover ou rebaixar o
  último usuário com papel "admin" — sem isso, seria possível trancar
  todo mundo fora da administração do próprio painel
- Página `/admins` (só visível/acessível pra quem já é admin — usa um
  decorator separado, `requer_admin`, já que essa página mexe direto
  no `auth.py` e não passa pelo helper, então o bloqueio central de
  RBAC não a cobre sozinho)
- **Limitação conhecida da v1, de propósito**: papéis mais granulares
  (ex: admin só de rede, só de domínio) ficam pra depois — essa
  versão é um corte binário simples (pode mudar tudo / não pode mudar
  nada), decisão consciente pra não precisar reescrever rota por rota
  agora
- Testado: os dois formatos de admin.json, proteção do último admin
  (remover e rebaixar), bloqueio no handle_request nos 4 cenários
  (leitura+mutante recusa, admin+mutante passa, sessão antiga sem
  role não quebra, leitura+leitura sempre passa), decorator
  requer_admin redirecionando de verdade

### 0.29.2
- **Bug real corrigido — consultas ficavam presas na memória do
  daemon indefinidamente**: o cronômetro do flush (grava no disco a
  cada 5 min) só era checado DENTRO do loop que processa linha nova
  do log — se não chegasse nenhuma consulta depois, a checagem nunca
  rodava, e os dados já contados ficavam presos na memória até a
  *próxima* consulta qualquer "acordar" a checagem (podia ser muito
  tempo depois, numa rede mais parada). Confirmado em produção:
  consultas reais de outra máquina não apareciam no painel por isso
- Corrigido: o loop principal foi reestruturado pra checar o flush a
  cada volta (a cada 0.5s quando ocioso), independente de chegar
  linha nova ou não — `seguir_log()` (gerador separado) foi removido,
  a lógica de acompanhar o arquivo ficou direto no `main()`
- Testado: flush acontece mesmo em período sem nenhuma consulta nova
  depois (cenário exato do bug relatado), parsing continua intacto

### 0.29.1 — Métricas de DNS, parte 2 (página completa)
- **Página nova `/dns-metricas`** — botão de ligar/desligar o log de
  consultas (com confirmação, avisando que o BIND reinicia), filtro
  de período (1h / 24h / 7 dias / 30 dias), tabelas de domínios mais
  consultados e IPs com mais requisições, total de consultas no
  período
- Link novo no menu — **de quebra, corrigido um bug real de
  navegação**: o link "DNS" usava `startswith('/dns')`, que também
  batia com `/dns-metricas` (e destacava os dois links ao mesmo tempo
  por engano). Ajustado pra `== '/dns' or startswith('/dns/')`,
  específico o bastante pra não pegar a página nova
- Aviso claro na página sobre a limitação conhecida (rankings de
  domínio e IP são independentes, não a combinação dos dois)
- **Fecha a funcionalidade completa de métricas de DNS** — log do
  BIND, daemon contínuo, agregação por período, e agora a interface
- Testado: sintaxe, rotas sem colisão, template nos dois estados
  (ligado com dados / desligado)

### 0.29.0 — Métricas de DNS, parte 1 (base + daemon, SEM página ainda)
- **Log de consultas do BIND**: `dns.habilitar_querylog()` /
  `desabilitar_querylog()` / `status_querylog()` — liga/desliga via
  arquivo de include separado (`/etc/bind/querylog.conf`), nunca mexe
  direto no `named.conf.local` além de uma linha de include. Log com
  rotação própria do BIND (5 arquivos de 50MB, nunca cresce sem
  limite)
- **Daemon novo, processo contínuo dedicado**
  (`arx-dns-stats.service`, roda como usuário `bind`) — acompanha o
  log de consultas ao vivo (tipo `tail -f`, reabre sozinho se o BIND
  rotacionar o arquivo), agrega em janelas de 5 minutos (top-50
  domínios + top-50 IPs por janela), grava em
  `/var/log/named/dns-stats.jsonl`, com poda automática de 30 dias
- **`dns_stats.resumo(horas, top_n)`** — soma as janelas dentro do
  período pedido, devolve os rankings consolidados. Primeira peça de
  filtro (por período) já funcionando
- **Limitação conhecida, documentada no código**: o daemon guarda os
  dois rankings (domínios, IPs) separados — não a combinação dos
  dois. Não dá pra responder "quais sites o IP X visitou" com o que é
  coletado hoje, só os rankings isolados. Se isso fizer falta, dá pra
  evoluir depois
- **Formato do log do BIND parseado com base na documentação, AINDA
  NÃO validado contra log real** — mesmo aviso de outras vezes nesse
  projeto que lidaram com formato de saída externo: primeira
  atualização do log de verdade vai confirmar se bate ou precisa
  ajuste
- Testado: liga/desliga o querylog preservando o resto do
  `named.conf.local`, parsing da linha contra o formato documentado,
  flush + poda de 30 dias, soma de múltiplas janelas respeitando o
  filtro de período
- **Pendente pra próxima etapa**: página no painel com os filtros
  visuais, botão de ligar/desligar o querylog pela interface

### 0.28.0 — Remover VLAN pelo painel
- **`network.py`** ganhou `list_vlans()` e `remove_vlan()` — apaga os
  arquivos `.netdev`/`.network` da VLAN, tira a linha `VLAN=<nome>`
  do arquivo da interface pai (sem apagar o arquivo pai inteiro —
  pode ter outras VLANs ou ser a config base da interface, busca em
  todos os `.network` pra achar o certo, não assume), derruba a
  interface do kernel (`ip link delete`) e recarrega. Mesmo rollback
  automático por timeout das outras mudanças de rede
- Aba "VLAN" na página de Rede agora lista as VLANs criadas com botão
  de remover (confirmação antes)
- **De quebra, uma inconsistência foi encontrada e corrigida**: `create_vlan` e o
  reversor de rede ainda usavam `systemctl restart systemd-networkd`
  (o jeito antigo, mais disruptivo) em vez do `networkctl reload` já
  corrigido em outras partes há algumas versões — só essas duas
  funções tinham ficado de fora daquela correção. Agora as três
  (criar, remover, reverter) usam `networkctl reload` consistentemente
- Testado: listar, remover preservando o resto do arquivo pai,
  interface derrubada do kernel, recarregamento chamado, recusa de
  VLAN inexistente, validação de nome perigoso

### 0.27.1
- **Bug real corrigido — página de acompanhamento ficava "inerte"
  depois que o helper reiniciava**: quando a atualização inclui o
  próprio `arx-painel`, o helper reinicia (esperado) e o processo
  novo não tem memória nenhuma do job antigo (estado fica só em
  memória). A página tentava reconectar pra sempre, achando que era
  só uma queda temporária — mas esse job específico nunca mais volta.
  Corrigido: `/api/jobs/<id>` agora distingue "job genuinamente sumido"
  (404) de "helper temporariamente inacessível" (502), e a página
  para de tentar e explica claramente o que aconteceu, com link pra
  conferir a versão atual — em vez de girar pra sempre
- Aviso proativo na página de Atualizações: selecionar `arx-painel`
  mostra um alerta explicando que o painel vai reiniciar no meio, então
  a tela de acompanhamento pode perder esse job (não é falha)
- Testado: sintaxe, template, JS balanceado, mensagem de erro
  "não encontrado" propagando corretamente do helper até o navegador

### 0.27.0 — Atualização com saída ao vivo, tipo terminal
- **Página de acompanhar atualização agora mostra a saída do `apt-get`
  em tempo real** — baixando, desempacotando, configurando, linha por
  linha, igual rodando no terminal de verdade
- `jobs.iniciar_job_streaming()` — variante de job que aceita um
  callback chamado a cada linha de saída, acumulando num campo
  `saida` que cresce ao longo da execução (em vez de só devolver tudo
  de uma vez no final)
- `updates.apply()` trocou `subprocess.run` (esperava tudo terminar)
  por `subprocess.Popen` lendo linha por linha — mantém o isolamento
  de cgroup (`systemd-run --scope`) já corrigido antes
- Página de status (`/jobs/<id>`) ganhou uma caixa estilo terminal,
  atualizada a cada 1.5s via `fetch`, com auto-scroll (só desce
  sozinho se você já estava no final — não atrapalha se estiver
  rolando pra cima pra ler algo)
- Testado: streaming linha por linha, job acumulando a saída em tempo
  real, template renderizando nos estados rodando/concluído

### 0.26.3 — Causa raiz real do dpkg corrompido em atualização
- **Bug crítico corrigido — atualizar o próprio `arx-painel` corrompia
  o `dpkg`, exigindo correção manual**: o `apt-get` rodava como
  processo FILHO do `arx-painel-helper`, dentro do cgroup do próprio
  `arx-painel-helper.service`. Quando a atualização incluía o próprio
  pacote, o `postinst` reiniciava o helper no meio da instalação — e
  o `systemd` mata o cgroup inteiro do serviço ao reiniciar,
  **incluindo o `apt`/`dpkg` que ainda estava rodando lá dentro**.
  Resultado real em produção: `dpkg foi interrompido`, depois
  `dpkg --configure -a` recusando (\"estado de inconsistência muito
  ruim\"), só resolvido reinstalando o pacote na mão
- **A correção da 0.26.2 (página de status resiliente a 500) ficou
  válida, mas não era a causa raiz** — só suavizava o sintoma na tela;
  o `dpkg` continuava corrompendo por trás
- Corrigido de verdade: `apt-get` agora roda dentro de
  `systemd-run --scope --collect` — um cgroup **independente**, fora
  da árvore do `arx-painel-helper.service`. Mesmo que o helper
  reinicie no meio (porque a atualização inclui ele mesmo), o
  `apt`/`dpkg` continua rodando sem ser morto junto
  - AppArmor: `systemd-run` adicionado (quem o helper chama
    diretamente agora; `apt-get` continua coberto, roda dentro do
    scope)
- Testado: construção do comando isolado confirmada
  (`systemd-run --scope --collect --quiet apt-get ...`). O
  comportamento real do isolamento de cgroup só se confirma de
  verdade numa atualização real em produção — é o próprio mecanismo
  que só se manifesta nesse cenário exato

### 0.26.2
- **Bug real corrigido — erro 500 "quebrando" a atualização do
  próprio painel (era só aparência)**: a página de acompanhar job
  (`/jobs/<id>`) usava `<meta http-equiv="refresh">` pra recarregar a
  cada 2s — quando a atualização inclui o `arx-painel` em si, os
  serviços reiniciam no meio, e se o refresh caísse bem nesse
  instante, o navegador mostrava a página de erro crua e parava de
  tentar. **A atualização em si sempre tinha terminado certinho por
  trás** (confirmado: `dpkg --audit` limpo, serviços de volta no ar) —
  só a página de acompanhamento não sabia se recuperar
- Corrigido: endpoint JSON novo `/api/jobs/<id>` + polling via JS
  (`fetch`) que trata falha de conexão como "painel reiniciando,
  tentando de novo" em vez de erro definitivo — continua sondando até
  conseguir, então recarrega a página só quando o job de fato termina
- Testado: sintaxe, rotas sem colisão, template renderiza nos estados
  rodando/concluído

### 0.26.1
- **Bug real corrigido — IP do admin nunca aparecia nos logs**: atrás
  do proxy reverso (nginx), `request.remote_addr` do Flask sempre
  retornava `127.0.0.1` (o próprio nginx), nunca o IP real de quem
  estava logado — o nginx já mandava `X-Forwarded-For` certinho, mas
  o Flask nunca foi configurado pra confiar nesse header. Corrigido
  com `ProxyFix` (`werkzeug.middleware.proxy_fix`), confiando só no
  último salto (o nginx local). Testado passando pelo `wsgi_app` de
  verdade (o jeito certo de testar isso — `test_request_context`
  sozinho não pega esse tipo de bug, já que pula o middleware)
- **Modo manutenção — desativar agora exige senha do admin**: ativar
  continua sem senha (não é destrutivo), mas desativar reabre TODA
  ação administrativa do painel, então agora pede confirmação por
  senha num modal (reaproveitando o modal de confirmação já existente
  — a senha nunca é comparada no JS, só coletada e conferida no
  servidor via `auth.verificar_credenciais`)
- **Ícones específicos pros alertas** — "Reinício pendente" e "Modo
  manutenção" tinham ícone genérico (não tinham ícone nenhum, na
  real). Corrigido, e de quebra ficou claro que nenhum dos dois virava
  alerta de VERDADE — o sistema de alertas só rastreia os itens de
  `health.get_status()`, e esses dois só existiam no `arx doctor`.
  Movidos pra `health.py` (o `doctor.py` não duplica mais, já vem de
  lá) — agora aparecem no histórico de alertas e no sino, com ícone
  próprio (reaproveitado dos mesmos SVGs do menu lateral)
- **Botão de reiniciar no aviso de reinício pendente** — a função e a
  rota já existiam (`system.reboot`), só faltava o botão no lugar
  certo, com confirmação
- Testado: os dois itens aparecendo em `health.get_status()`, sem
  duplicar no doctor; templates renderizando com os ícones novos

### 0.26.0 — Fechamento da Fase 2 (itens 7/9, 8/9, 9/9): últimos três de uma vez
**Fase 2 dos 9 itens planejados está COMPLETA.**

- **Item 9 — Modo manutenção**: novo `manutencao.py` +
  `_util.status_manutencao/ativar_manutencao/desativar_manutencao`
  (estado em disco, sobrevive a restart do helper). Bloqueio
  **centralizado** no `handle_request` do `helper.py` — qualquer ação
  mutante de QUALQUER módulo é recusada com o modo ativo, exceto
  `manutencao.desativar` (senão ninguém escaparia do modo). Banner
  visível em toda página quando ativo. Página `/manutencao`
- **Item 7 — Atualização mais segura**: `jobs.ha_job_rodando()` +
  `updates.apply_async()` agora recusa iniciar uma segunda atualização
  enquanto outra roda (duas ao mesmo tempo podem corromper o estado
  do apt/dpkg). `updates.reboot_necessario()` (`/var/run/
  reboot-required`, padrão do Debian) — banner na página de
  Atualizações e item novo no arx doctor
- **Item 8 — Exportar/importar config**: novo `config_portavel.py` —
  versão **portátil** da config (rede/firewall/DHCP/DNS), diferente
  do backup completo (que é pra restaurar na MESMA máquina): isso é
  pra levar pra **outro** servidor Arx OS, sem chave TLS/banco do
  Samba/leases. Import protegido por allowlist de caminho (testado
  recusando escrever em `/etc/shadow` com pacote adulterado) e backup
  automático de cada arquivo antes de sobrescrever. Página
  `/config-portavel`, exige digitar "IMPORTAR" pra confirmar
- Testado: ciclo completo do modo manutenção (bloqueia mutante,
  libera com desativar, libera de novo depois), recusa de atualização
  concorrente, exportar/importar com proteção de allowlist,
  integração do reboot_necessario no arx doctor, todos os templates
  novos renderizando nos dois cenários (ativo/inativo)
- AppArmor: só precisou de `/var/run/reboot-required` novo — o resto
  (`/var/lib/painel/**`, `/var/backups/painel/**`, `/etc/dhcp/**`,
  `/etc/bind/**`, `/etc/nftables.conf*`) já estava coberto com
  wildcard desde antes

### 0.25.0 — Fechamento da Fase 2 (item 6/9): Certificados
- **Módulo novo `certificados.py`** — validade, emissor, titular, SAN
  e fingerprint do certificado HTTPS do painel (`openssl x509`), com
  alerta por proximidade de expiração (crítico &lt;7 dias, alto
  &lt;30, atenção &lt;60)
- Página nova `/certificados`
- **Integrado ao arx doctor** — mais um item na checagem agregada
- Parsing testado contra certificado real gerado na hora (formato
  exato confirmado — `sha256 Fingerprint=` vem em minúsculo,
  diferente do que a documentação às vezes sugere) e contra cenário
  de expiração próxima (certificado de 5 dias, detectado como crítico
  corretamente) e arquivo inexistente (degrada sem quebrar)
- Sem ACME/Let's Encrypt ainda, de propósito — fica pra depois, como
  já estava combinado desde o planejamento desse item
- Novos caminhos no AppArmor: `openssl` (exec) e leitura do
  certificado

### 0.24.0 — Fechamento da Fase 2 (item 5/9): Armazenamento básico
- **Scrollbar no mesmo tema escuro do painel** — a barra clara padrão
  do navegador destoava de tudo
- **`disco.py`** ganhou `list_particoes` (todas as partições/volumes,
  não só discos inteiros, com sistema de arquivos e flag de LUKS),
  `list_espaco` (uso por ponto de montagem real, ignora tmpfs/
  overlay/etc) e `status_luks` (dispositivos LUKS desbloqueados agora)
- Página nova `/armazenamento` — discos físicos (linka pro SMART já
  existente em Diagnóstico, sem duplicar), espaço por ponto de
  montagem (cor de alerta em 80%/90%), partições com flag de LUKS,
  LUKS aberto agora
- `list_particoes` usa `lsblk -P` (pares chave=valor) em vez do
  formato em árvore padrão — evita ter que lidar com os caracteres de
  desenho da árvore (├─, └─) na hora de parsear
- Testado com dados reais desse ambiente (partições, espaço, LUKS
  ausente tratado sem quebrar — `dmsetup` nem sempre está instalado)

### 0.23.2
- **Bug real corrigido — `isc-dhcp-server` falhava no boot, de forma
  intermitente**: o serviço subia ANTES do `systemd-networkd` terminar
  de configurar a interface (que também recebe IP via DHCP do lado de
  cima) — o `dhcpd` via "no IPv4 addresses" em `enp1s0` e recusava
  escutar, derrubando o serviço inteiro. Corrida de inicialização
  clássica: falhava ou não dependendo de qual serviço terminava
  primeiro. Corrigido com um drop-in systemd
  (`/etc/systemd/system/isc-dhcp-server.service.d/arx-espera-rede.conf`)
  fazendo o `isc-dhcp-server` esperar `network-online.target` antes de
  subir — não editamos o unit do pacote original (isso se perderia na
  próxima atualização), é um override em cima
- Diagnosticado com `dhcpd -t` (config OK) e `dhcpd -f -d` (rodando na
  mão funcionava, confirmando que não era a config) até achar a causa
  real no log do `journalctl`

### 0.23.1 — Dashboard ao vivo (parte 1: sistema e rede)
- **Novo: seção "Ao vivo" no Dashboard** — CPU, memória, rede (por
  interface, incluindo VLAN) e disco, atualizando sozinho a cada 3s,
  sem recarregar a página
- **`metrics.live_snapshot()`** — leitura instantânea dos contadores
  crus do kernel (`/proc/stat`, `/proc/meminfo`, `/proc/net/dev`,
  `/sys/block/*/stat`). Não calcula taxa nenhuma no backend — devolve
  os contadores cumulativos com timestamp, o JS calcula a variação
  entre duas sondagens sucessivas
- Endpoint novo `/api/metricas-vivas` (JSON), consumido pelo
  `dashboard-vivo.js` — sem biblioteca externa nenhuma (o CSP do painel
  não permite CDN), tudo em JS puro
- Disco lido via `/sys/block/*` (não `/proc/diskstats`) — só lista
  dispositivos inteiros, evita ter que adivinhar padrão de nome de
  partição
- Testado: leitura real contra o `/proc`/`/sys` desse próprio
  ambiente (CPU, memória, interfaces, discos todos lidos
  corretamente)
- **Pendente, de propósito, pra não fazer correndo**: métricas de
  DNS, sites mais acessados, IPs com mais requisições — precisa
  habilitar log de consultas no BIND e construir um agregador do
  zero, é praticamente um projeto à parte

### 0.23.0 — Fechamento da Fase 2 (item 4/9): arx doctor
- **Módulo novo `doctor.py`** — checagem agregada, reaproveita
  `health.get_status()` (firewall, DHCP, DNS, NTP, Samba AD, disco,
  serviços do painel), soma o status do AppArmor (por perfil) e as
  atualizações pendentes, tudo num índice só de 0 a 100
- Página nova `/doctor` — pontuação grande, veredito geral, tabela
  com todos os itens (mesmo badge de gravidade já usado em Saúde)
- Não inventa checagem nova nenhuma — só junta o que já existia numa
  visão consolidada, exatamente como o roadmap pedia
- Testado: cenário bom (100 - pequenas penalidades) e cenário ruim
  (múltiplos problemas simultâneos), pontuação calculada corretamente
  nos dois

### 0.22.2
- **Bug real corrigido — `OU=Domain Controllers` aparecia na lista
  como se fosse uma OU do admin**: é criada automaticamente pelo
  próprio Samba no provisionamento, guarda só objetos de computador
  dos controladores de domínio — não é pra uso geral, e não deveria
  aparecer como destino pra mover usuário. Mesmo padrão de filtro já
  usado pras contas de serviço, agora pras OUs do sistema. Testado

### 0.22.1
- **Bug real corrigido — contas de serviço apareciam dentro dos
  grupos, removíveis com um clique**: `list_group_members()` não
  tinha o mesmo filtro que `list_users()` já tinha — dentro de
  "Domain Users", por exemplo, dava pra ver e remover `arx-dns-svc` e
  `dns-<hostname>`, um jeito real de quebrar o DNS do domínio sem
  querer. Corrigido: filtro compartilhado (`_filtrar_contas_servico`)
  aplicado nos dois lugares agora
- **OUs viram úteis de verdade — `move_user_to_ou`**: antes dava pra
  criar OU, mas não tinha como colocar ninguém dentro dela. Adicionado
  `samba-tool user move`, com um seletor de "Mover pra OU" direto na
  tabela de usuários (só aparece quando existe alguma OU criada)
- Testado: filtro funcionando em `list_group_members`, comando de
  mover montado corretamente, validações de nome/OU rejeitando
  entrada inválida

### 0.22.0 — Fechamento da Fase 2 (item 3/9): Grupos e OUs do Samba AD
- **`domain.py`** ganhou gerenciamento completo de grupos
  (`create_group`, `delete_group`, `list_groups`,
  `list_group_members`, `add_group_member`, `remove_group_member`) e
  de OUs — unidades organizacionais (`create_ou`, `delete_ou`,
  `list_ous`), via `samba-tool group`/`samba-tool ou`
- Sintaxe confirmada contra documentação oficial antes de implementar
- **Nome de grupo restrito a um conjunto seguro de caracteres** —
  existe um bug documentado no `samba-tool` onde nome de grupo com
  caractere especial (parênteses, aspas) faz `delete`/`listmembers`
  não acharem o grupo de volta depois de criado. Evitado de
  propósito, não só documentado
- Abas novas "Grupos" e "OUs" na página de Domínio, página dedicada
  de membros por grupo (`/dominio/grupo/<nome>`)
- Remover OU usa `--full` (recursivo) — sem isso o samba-tool recusa
  apagar uma OU não-vazia
- **Checagem forte reforçada**: passou a cruzar a lista de ações
  só-leitura contra o dicionário principal de ações — pegou um erro
  real nesse próprio commit (3 funções de leitura registradas só na
  lista de auditoria, não no dispatcher, o que faria toda chamada
  falhar com "ação desconhecida")
- Testado: comandos corretos (incluindo confirmação de que
  `addmembers` não usa aspas extras, evitando outro bug documentado
  do samba-tool), validação de nome perigoso rejeitada, DN de sub-OU
  montado corretamente

### 0.21.7
- **Bug real corrigido — remover registro DNS do Samba não pedia
  confirmação**: apagava direto no clique, sem nenhum aviso — os
  outros dois lugares do painel que removem algo parecido (zona do
  BIND, regra de firewall) já usam um modal de confirmação
  (`data-confirmar`), esse aqui tinha ficado de fora. Corrigido, mesmo
  padrão, tratado globalmente pelo `componentes.js` (não precisou de
  JS novo)

### 0.21.6
- **Bug real corrigido — listagem de registros do Samba sempre
  vazia**: o regex de `Name=` usava `re.match` exigindo bater exato
  do início da linha, mas a saída real do `samba-tool dns query` vem
  indentada (`  Name=...`, `    SOA:...`) — o exemplo usado antes pra
  testar não tinha essa indentação preservada. Corrigido, testado
  contra a saída real capturada em produção (bate 100%, incluindo o
  registro que o usuário tinha acabado de adicionar)

### 0.21.5
- **Mensagens de erro do `samba-tool dns` mais limpas**: o aviso
  fixo de "senha na linha de comando" (aparece em toda chamada com
  `-U user%senha`) estava poluindo a mensagem de erro real mostrada
  pro admin. Filtrado nos 4 pontos que chamam `samba-tool dns`
- **Bug real corrigido — contas de serviço aparecendo na lista de
  usuários do domínio**: `arx-dns-svc` (a que a gente cria) e
  `dns-<hostname>` (que o próprio Samba cria sozinho no
  provisionamento com BIND9_DLZ) apareciam misturadas com usuários
  humanos de verdade, com botão de "Bloquear" e tudo — confuso e
  arriscado (alguém podia bloquear sem querer a conta que o Samba
  precisa pra funcionar). Corrigido: `list_users()` filtra as duas
- Testado: limpeza de mensagem preserva o erro real, filtro remove só
  as contas de serviço, usuários reais continuam aparecendo normal

### 0.21.4 — Editar registros das zonas do Samba/AD
- **Zonas do Samba agora são editáveis pelo painel**, igual as do
  BIND: `domain.list_dns_records_samba`, `add_dns_record_samba`,
  `remove_dns_record_samba` — via `samba-tool dns query/add/delete`,
  reaproveitando a conta de serviço já configurada (mesma senha
  criptografada, nada novo pra guardar)
- Sintaxe confirmada contra documentação oficial antes de implementar
  (`samba-tool dns add/delete <servidor> <zona> <nome> <TIPO> <dado>`)
  — o parsing do `query` foi testado contra um exemplo real de saída
  documentado (agrupamento por "Name=X, Records=N"), bateu certinho
- Página nova `/dns/samba/<zona>` — lista registros, formulário de
  adicionar, botão de remover. Link direto a partir da lista de zonas
  em `/dns`
- Testado: parsing bate com saída real, comandos add/delete montados
  corretamente, validação de tipo e campos obrigatórios

### 0.21.3
- **Confirmado em produção**: rota adicionada corretamente, DHCP
  intacto (`ip route` mostrando os dois lado a lado) — a correção da
  0.21.2 funcionou
- **Bug real corrigido — remover rota não pedia confirmação**: a
  rota `/rede/rota/remover` chamava `network.remove_route` mas
  descartava o resultado sem checar `pendente_confirmacao` — o
  backend aplicava o rollback automático certinho (a remoção de rota
  usa o mesmo mecanismo de segurança que adicionar), só que o modal de
  contagem regressiva nunca aparecia, então o admin não tinha como
  saber que a remoção seria desfeita sozinha em 15s se nada
  confirmasse. Corrigido — mesmo padrão de `add_route`. Varredura no
  resto do `app.py` não achou mais nenhuma chamada nessa situação
  (uma outra suspeita, `firewall.init_defaults`, checada e confirmada
  que não usa rollback de propósito — sempre libera SSH/painel,
  não precisa)

### 0.21.2
- **Bug real corrigido — mudança de rede sequestrava a interface de
  outro arquivo, derrubando o DHCP**: `add_route`/`set_static`/
  `set_dhcp` sempre escreviam em `10-{iface}.network`, mas a interface
  já podia estar sendo governada por outro arquivo com nome diferente
  (ex: `20-wired.network`, comum em imagens base) — o
  systemd-networkd usa o PRIMEIRO arquivo que bate em ordem
  alfabética, então criar um `10-...` roubava a interface do arquivo
  de verdade. Aconteceu em produção: adicionar uma rota derrubou o
  DHCP inteiro. Corrigido: as três funções agora usam
  `_arquivo_governante()` (que já existia no projeto, só não estava
  sendo usada aqui) pra descobrir e editar o arquivo que REALMENTE
  está em vigor, só caindo pro nome padrão `10-{iface}.network`
  quando a interface nunca teve config própria ainda
- Testado: cenário exato do usuário (interface com `20-wired.network`
  preexistente + DHCP) — acha o arquivo certo, preserva DHCP, não
  cria arquivo paralelo. Regressão: fluxo antigo (já usando
  `10-{iface}.network`) continua funcionando

### 0.21.1
- **Bug real corrigido — mudança de rede "travava a rede" inteira**:
  toda mudança de rede (IP fixo, VLAN, e agora rotas) reiniciava o
  `systemd-networkd` **inteiro** via `systemctl restart` — isso
  derruba TODAS as interfaces por um instante, não só a que está
  sendo alterada. Corrigido: troca pra `networkctl reload`, o comando
  certo pra recarregar config sem reiniciar o serviço nem derrubar
  links. `networkctl` adicionado ao perfil AppArmor
- **Bug real corrigido — modal de confirmação reaparecendo à toa**:
  a página confiava cegamente no parâmetro `?pendente=N` da URL pra
  decidir se mostra o modal — se a página fosse recarregada DEPOIS do
  rollback automático já ter acontecido, o modal reaparecia mesmo sem
  nada pendente de verdade. Corrigido em Rede **e** Firewall (mesmo
  mecanismo, mesmo bug) — agora confere o estado real antes de
  mostrar o modal, via `tem_mudanca_pendente()` novo em `_util.py`
- Testado: `tem_pendente` distingue escopos corretamente,
  `_escrever_e_reiniciar` confirmado usando `networkctl reload`

### 0.21.0 — Fechamento da Fase 2 (item 2/9): Rotas estáticas
- **`network.py`** ganhou `add_route`, `remove_route`, `list_routes`
  — rotas estáticas por interface, via `[Route]` no arquivo
  `.network` do systemd-networkd
- **Preserva a config existente**: como o arquivo é reescrito por
  inteiro (mesmo padrão do `set_static`), lê endereço/gateway/DNS
  atuais antes de adicionar/remover uma rota — sem isso, perderia o
  resto da configuração da interface
- Mesmo rollback automático por timeout das outras mudanças de rede
  (rota errada pode cortar o acesso do próprio admin)
- Aba nova "Rotas" na página de Rede — lista por interface, formulário
  de adicionar, botão de remover
- Testado: preserva config existente ao adicionar, recusa duplicata,
  recusa remover rota inexistente, valida destino (CIDR) e via (IP)

### 0.20.8 — Conta de serviço dedicada pra consultar DNS do Samba
- **Kerberos implícito não funciona pra essa chamada RPC específica**
  (confirmado em produção — `-k`/`--use-kerberos=required` falham com
  `NT_STATUS_INVALID_PARAMETER` mesmo como root na própria DC; só
  usuário/senha explícitos funcionam). Em vez de usar a senha do
  Administrator, criada uma **conta de domínio dedicada**
  (`arx-dns-svc`), privilégio mínimo (grupo `DnsAdmins`, só consulta),
  senha aleatória de 32 caracteres gerada na hora
- **Senha guardada criptografada** (Fernet/`cryptography`), nunca em
  texto puro — chave e senha em arquivos separados, `chmod 600`,
  `/etc/painel/dns_secret.{key,enc}`
- Botão novo "Configurar conta de consulta DNS" na página de DNS,
  aparece quando a lista de zonas do Samba vier vazia
- Parsing confirmado batendo com a saída real do `samba-tool` (não
  precisou de ajuste — só a autenticação estava errada)
- Nova dependência: `python3-cryptography`
- Testado: criação da conta (+ reconfiguração se já existir),
  criptografia/descriptografia da senha, permissões 600, comando
  final usa a conta dedicada (nunca o Administrator)

### 0.20.7
- **Bug real corrigido — página Configurações/Serviços só mostrava 6
  serviços**: a lista fixa (`SERVICOS_PERMITIDOS`) nunca foi
  atualizada conforme os módulos foram sendo construídos — faltava
  `named`, `isc-dhcp-server`, `chrony`, `samba-ad-dc`, `smbd`/`nmbd`,
  `apt-cacher-ng`. Agora tem os 13 serviços que o painel realmente
  gerencia hoje, cada um com descrição
- Testado: lista completa confirmada

### 0.20.6 — Saúde consciente do backend, zonas do Samba visíveis
- **Bug real corrigido — crítico falso de DNS com SAMBA_INTERNAL**: a
  checagem de saúde sempre conferia se o `named` estava ativo, mas
  com backend SAMBA_INTERNAL ele fica parado DE PROPÓSITO (o próprio
  Samba serve DNS) — gerava alerta crítico falso toda vez. Corrigido:
  a checagem agora sabe qual backend está em uso e só cobra o `named`
  quando ele é quem deveria estar respondendo de verdade
- **Página de DNS agora mostra as duas fontes de zona**: zonas
  estáticas do BIND (como sempre foi) **e** zonas gerenciadas pelo
  Samba/AD (nova aba "Zonas (Samba/AD)", só aparece quando há domínio
  provisionado) — os dados do domínio vivem no AD independente do
  backend, por isso aparecem mesmo com SAMBA_INTERNAL. Mostra
  claramente qual backend está ativo no topo da página
- Nova função `domain.list_dns_zones_samba()` — usa Kerberos da
  própria máquina (`-k yes`), sem precisar guardar senha de admin no
  painel. **Formato exato da saída do samba-tool não validado ao
  vivo** — parsing defensivo, pode precisar ajuste depois do teste
  real
- Testado: health.py não dá crítico falso com SAMBA_INTERNAL mas
  ainda pega o problema real com BIND9_DLZ; parsing das zonas do
  Samba; template renderiza com e sem domínio provisionado

### 0.20.5
- **Bug real corrigido — mesmo conflito de zona, agora pela porta do
  módulo DNS comum**: `dns.create_zone()` deixava criar uma zona
  estática com o mesmo nome do realm do domínio, mesmo quando o
  backend é BIND9_DLZ — o `named` só recusava subir no próximo
  restart, não na hora de criar. Corrigido: `create_zone()` agora
  recusa na hora, com mensagem clara, se a zona colide com o realm de
  um domínio provisionado em BIND9_DLZ
- `domain.status()` ganhou o campo `realm` (faltava, usado pela
  checagem nova)
- Testado: recusa a zona conflitante, não bloqueia zona com nome
  diferente

### 0.20.4
- **Mais um gap real achado no log de produção**: `/etc/krb5.conf`
  (renomear pra backup, criar o novo, `chmod`) não estava coberto no
  perfil do helper — o reprovisionamento de domínio mexe bastante
  nele (pra `kinit`/Kerberos reconhecerem o realm recém-criado).
  Adicionado `/etc/krb5.conf* rw,`
- Resto do log conferido sem mais nada de novo — `__pycache__` não
  reapareceu (confirma a correção da 0.19.1 segurando)

### 0.20.3
- **Bug real corrigido — detecção de backend de DNS sempre mostrava
  "não detectado"**: suposição errada de que o `samba-tool` grava uma
  chave `dns backend =` separada no `smb.conf` — confirmado contra o
  arquivo real gerado em produção, ele não grava. Corrigido: detecta
  pela presença do `named.conf` gerado (só existe no modo BIND9_DLZ)
  em vez de procurar uma chave que nunca existiu. Testado: os três
  cenários (não provisionado, SAMBA_INTERNAL, BIND9_DLZ)
- **Confirmado em produção**: `samba-ad-dc` e `named` rodando juntos,
  `active` e `enabled` nos dois — a integração BIND9_DLZ está de pé

### 0.20.2
- **Bug real corrigido — `named` não ficava habilitado pro boot**: o
  código só reiniciava o `named` (`systemctl restart`), nunca
  habilitava (`enable`) — funcionava na hora, mas sumia no próximo
  reboot. Corrigido: `enable` e `restart` chamados separadamente (só
  `enable --now` não bastaria, já que não reinicia se o serviço já
  estiver ativo — e precisamos que ele releia a config nova sempre).
  Testado: os dois comandos confirmados, na ordem certa

### 0.20.1
- **Bug real corrigido — zona estática conflitando com a zona do
  DLZ**: reprovisionar com BIND9_DLZ quando já existia uma zona
  estática com o mesmo nome do realm (criada antes, manualmente, pelo
  módulo DNS comum) fazia o `named` recusar subir ("already exists"),
  com erro genérico de "falhou ao reiniciar" sem dizer o motivo real.
  Corrigido: detecta esse conflito específico ANTES de tentar
  reiniciar, e aponta exatamente qual bloco remover de
  `named.conf.local`. Testado: detecta o conflito real, não mexe em
  outras zonas, e não afeta o fluxo quando não há conflito

### 0.20.0 — Provisionamento unificado, escolha de backend de DNS
- **Provisionar (primeira vez) e reprovisionar (destrutivo) agora
  compartilham a mesma lógica** — antes eram dois caminhos separados
  e divergentes. Os dois aceitam escolher o backend de DNS
  (`SAMBA_INTERNAL` ou `BIND9_DLZ`) desde a primeira vez, não só no
  reprovisionamento
- **`status()` agora detecta o backend de DNS configurado de
  verdade** — lê o `smb.conf` gerado pelo samba-tool, em vez de supor.
  Mostrado no topo da página de Domínio
- Aba "Avançado" atualizada: reprovisionar aceita trocar de backend
  nos dois sentidos (Samba interno ⇄ BIND9_DLZ), não só pra BIND9_DLZ
- Formulário de provisionamento inicial ganhou os dois rádios de
  escolha, com explicação de cada um
- Testado: provisionar com cada backend, recusa provisionar de novo
  se já existe, reprovisionar trocando de backend, status detecta
  corretamente em cada caso

### 0.19.9 — Bug fundamental corrigido: bind9 → named
- **Descoberta importante**: o Debian empacota o serviço real do
  bind9 como `named.service` — `bind9.service` era só um **alias**
  (symlink criado ao habilitar). Todo o código do painel (desde o
  início) chamava `systemctl ... bind9`, que só funcionava enquanto
  esse alias existia. Quando o fix da 0.19.6 mandou desativar o bind9
  (`systemctl disable bind9`), isso REMOVEU o alias — e a partir daí
  "bind9.service" parou de existir de verdade, quebrando toda
  chamada seguinte
- **Corrigido em todo o projeto**: `dns.py` (status, restart, reload
  — usado toda vez que se muda uma zona/registro), `domain.py`
  (provisionamento e reprovisionamento com BIND9_DLZ), `backup.py`
  (restauração de backup completo). Varredura confirmou que não
  sobrou nenhuma chamada `systemctl` com "bind9" em lugar nenhum do
  código
- Testado: comandos gerados confirmam "named" em vez de "bind9"

### 0.19.8
- **Bug real corrigido — `tdbbackup` (pacote `tdb-tools`) faltando**:
  o provisionamento com BIND9_DLZ precisa dessa ferramenta pra copiar
  o banco do AD pro BIND ler — sem ela, o `samba-tool` falha bem no
  meio, DEPOIS de já ter apagado o domínio antigo (backup feito, mas
  ainda assim uma surpresa ruim). Corrigido: `tdb-tools` adicionado
  como dependência do pacote, e checagem movida pra ANTES de mexer em
  qualquer coisa destrutiva, falhando rápido com mensagem clara em
  vez de no meio do processo. Testado

### 0.19.7 — Reprovisionar domínio com BIND9_DLZ
- **Solução definitiva pro conflito de porta 53** (0.19.6 só
  contornava desativando o bind9 — essa versão faz os dois
  trabalharem JUNTOS): novo fluxo `domain.reprovisionar_com_bind9_dlz`
  — reprovisiona o domínio do zero com `--dns-backend=BIND9_DLZ`, o
  BIND vira o servidor de DNS de verdade (alimentado pelos dados do
  AD via plugin), sem brigar com o DNS interno do Samba
- **Extremamente destrutivo** — apaga o domínio atual (usuários,
  grupos) e recria do zero. Exige digitar "REPROVISIONAR" (palavra
  diferente do "CONFIRMAR" do provisionamento normal, pra não
  confundir as duas ações)
- **Honestidade sobre incerteza real**: o caminho exato do
  `named.conf` que o `samba-tool` gera varia por empacotamento — o
  código procura nos caminhos conhecidos em vez de fixar um só, e
  avisa claramente (sem fingir sucesso) se não achar em nenhum
- Aba nova "Avançado" na página de Domínio, separada das ações do
  dia a dia
- Testado: exige confirmação, acha o named.conf certo entre vários
  candidatos, inclui no bind9 sem duplicar em reconfigurações, avisa
  claramente quando não encontra nenhum candidato

### 0.19.6
- **Bug real corrigido — conflito de porta 53 derrubava o samba-ad-dc
  inteiro**: `bind9` (módulo DNS do painel) e o DNS interno do Samba
  disputam a mesma porta — quando os dois tentam subir juntos, o
  Samba AD morre inteiro (não só a parte de DNS dele), porque "dns
  failed to setup interfaces" derruba o processo todo. Isso já era
  uma limitação conhecida/documentada, mas nunca tinha causado queda
  real em produção até agora. Corrigido: `domain.provision()` agora
  desativa o `bind9` automaticamente ao provisionar o domínio (o
  Samba assume o papel de DNS a partir daí), com aviso claro no
  resultado do job quando isso acontece. Testado: desativa e avisa
  quando bind9 estava ativo, não avisa quando já estava inativo

### 0.19.5 — Saúde expandida: DHCP, DNS, NTP, Samba AD
- **Investigação de "firewall caído não detectado" concluída — falso
  alarme, não era bug**: teste controlado (start → confirma regras →
  stop → confirma tabela sumiu → checa `/saude`) confirmou a detecção
  funcionando corretamente. A confusão veio do estado de um teste
  anterior, não do código
- **Cobertura de saúde expandida** — antes só cobria serviços do
  próprio painel + disco + firewall. Agora também:
  - **DHCP**: parado = "alto" (máquina nova não pega IP, quem já tem
    lease continua funcionando por um tempo)
  - **DNS**: parado = "crítico" (afeta na hora quem depende desse
    servidor pra resolver nome)
  - **NTP**: não basta o serviço estar ativo, precisa estar
    **sincronizado de verdade** (reaproveita `ntp.get_sync_status()`)
    — dessincronizado = "crítico", por causa do risco real pro
    Kerberos/AD
  - **Samba AD**: só entra na checagem se o domínio **já foi
    provisionado** — não gera alerta sobre um serviço que nunca foi
    configurado de propósito
- Testado: 6 cenários (tudo ok, cada serviço caído individualmente,
  domínio não-provisionado não aparece na lista, Samba parado quando
  provisionado)

### 0.19.4
- **Mais dois gaps reais achados no log de produção**: `traceroute`
  no Debian na verdade executa `/usr/bin/traceroute.db` por trás
  (`update-alternatives`) — o perfil cobria só o nome do link.
  Trocado pra `/usr/bin/traceroute* Ux,` (cobre os dois). E a limpeza
  de cache do repositório local (`expire-caller.pl` + sua dependência
  `acngtool`) não estava coberta — adicionados

### 0.19.3
- **Bug real corrigido — trocar canal de atualização dava 404**: ao
  inserir as rotas novas de AppArmor mais cedo, o decorador
  `@app.route("/atualizacoes/canal", ...)` sumiu por engano (a
  linha foi usada como texto de referência na edição e não recolocada). Sem CSP
  ou AppArmor envolvidos — erro de edição do arquivo, nada a ver
  com as mudanças de segurança dessa leva. Corrigido, e o
  arquivo inteiro foi varrido procurando o mesmo padrão em outro lugar — não
  achou mais nenhum caso

### 0.19.2
- **Confirmado no log real**: a correção do `__pycache__` da 0.19.1
  funcionou — nenhuma escrita nova apareceu depois do reload do
  perfil
- **Mais um gap real achado no log**: `/usr/bin/apt` (diferente de
  `apt-get`, que já estava liberado) — `updates.py` usa `apt list
  --upgradable` pra checar atualizações disponíveis. Confirmado no
  código (linha 69) antes de corrigir. Adicionado ao perfil do helper

### 0.19.1
- **Ajustes reais no perfil do helper, achados no log de produção**
  (o próprio processo do `aa-logprof`, feito na mão lendo o
  `journalctl`): faltavam `/etc/arx/canal` e
  `/etc/apt/sources.list.d/arx.list` (lidos/gravados pelo módulo de
  atualizações)
- **`__pycache__` desligado** (`PYTHONDONTWRITEBYTECODE=1` nas duas
  units) — o Python tentava escrever bytecode compilado dentro da
  própria pasta de instalação, algo que nenhum serviço deveria
  precisar fazer. Mais limpo desligar o cache do que abrir permissão
  de escrita ali

### 0.19.0 — Fechamento da Fase 2 (item 1/9): AppArmor
- **Perfis do painel reescritos do zero** — os que existiam (de bem
  antes) assumiam um binário compilado e SQLite; a arquitetura real é
  Python + JSON. Caminhos e binários atualizados pra bater com tudo
  que foi construído desde então (jobs, backup, alertas, relatório,
  repositório, diagnóstico, etc)
- **`painel-helper`** e **`painel-webui`**: carregados automaticamente
  no postinst, sempre em modo **complain** (só loga violação, nunca
  bloqueia) — nascem assim de propósito. `AppArmorProfile=` adicionado
  nas units systemd, atrelando o perfil ao serviço
- Página nova (`/apparmor`): status de cada perfil, botão pra
  promover pra `enforce` (com aviso forte, modal de confirmação) ou
  voltar pra `complain`
- **nginx/Samba/SSH NÃO incluídos** de propósito — são serviços de
  terceiros complexos, arriscado demais confinar sem observação real.
  Rascunhos ficam em `/usr/share/doc/arx-painel/apparmor-perfis/` pro
  admin revisar e aplicar na mão, seguindo o processo documentado
  (complain → uso real → `aa-logprof` → enforce)
- Testado: parse do `aa-status`, validação rejeita perfil de
  terceiro e modo inválido, comando `aa-enforce`/`aa-complain` correto
- Novas dependências: `apparmor`, `apparmor-utils`
- **Primeiro de 9 itens combinados pra fechar a Fase 2 de vez** antes
  de partir pra Fase 3 (Desktop) — ordem definida pelo usuário

### 0.18.0 — Repositório local: de espelho completo pra modo cache
- **Mudança de arquitetura** (motivo real: hardware fraco, 2 núcleos,
  travando tentando espelhar o Debian inteiro mesmo filtrado — testado
  em produção, o processo consumia quase 50% de CPU e não cabia)
- **Saiu**: `aptly` como espelho completo (unstable→stable, jobs
  assíncronos de sincronização, agendamento configurável, toda a saga
  de correções de chave GPG das versões 0.17.2 a 0.17.10)
- **Entrou**: `apt-cacher-ng` — proxy de cache que só baixa (e guarda)
  o que uma máquina de domínio pede de verdade, crescendo aos poucos.
  Já vem com Debian pré-configurado de fábrica; só precisamos adicionar
  o repositório central do Arx como fonte extra
- **Bônus real**: elimina de vez a dor de cabeça de chave GPG — o
  `apt-cacher-ng` não verifica assinatura, é só um proxy passivo. Quem
  verifica continua sendo o `apt` de cada cliente, do jeito que sempre
  foi
- Página `/repositorio` bem mais simples: status, configurar repo
  central (sem chave nenhuma), limpar cache obsoleto
- **Ativo por padrão em Modo Servidor** (requisito original mantido)
- Dependências trocadas: `aptly`+`debian-archive-keyring` saem,
  `apt-cacher-ng` entra
- Testado: status, configuração idempotente (reconfigurar substitui a
  linha, não duplica), validações de URL

### 0.17.10
- **Bug real corrigido — `mirror update` não reaproveitava a keyring
  do `mirror create`**: são comandos separados no `aptly`, cada um com
  sua própria flag `-keyring=` — passar só na criação não bastava,
  toda sincronização seguinte falhava com o mesmo erro "sem chave
  pública". Corrigido adicionando `-keyring=` também nos dois
  `mirror update` (Debian e Arx) — como os caminhos são sempre fixos,
  funciona tanto na configuração inicial quanto nas sincronizações
  periódicas depois, sem precisar passar como parâmetro. Testado:
  os dois comandos de update confirmam a flag

### 0.17.9
- **Bug real corrigido — `gpg --export` recusava sobrescrever a
  keyring de uma tentativa anterior**: mesma família dos bugs
  "already exists" de reexecução — o arquivo `arx-central.gpg` já
  existia de uma tentativa passada, e `gpg --export` recusa
  sobrescrever por padrão. Corrigido com `--yes`. Testado: comando
  final confirma a flag

### 0.17.8
- **Bug real corrigido — `gpg` tentando abrir `/dev/tty`**: rodando
  como serviço systemd (sem terminal interativo nenhum), o `gpg` tenta
  abrir `/dev/tty` por padrão em várias operações e falha
  (`cannot open '/dev/tty'`). Corrigido adicionando `--batch --no-tty`
  em **todas** as quatro chamadas de `gpg` do módulo (listar chave
  local, gerar chave local, importar chave central, exportar chave
  central) — não só na que já tinha (geração de chave). Testado:
  confirma que as quatro chamadas incluem as duas flags

### 0.17.7
- **As duas correções de keyring (0.17.5/0.17.6) confirmadas
  funcionando em produção** — assinaturas do Debian validando certinho
- **Bug real corrigido — `configurar_inicial()` não era
  reexecutável**: tentativas anteriores que falharam no meio do
  caminho (pelos bugs já corrigidos) deixavam o mirror "preso" no
  banco do `aptly`, e a tentativa seguinte batia em "already exists"
  mesmo já com tudo corrigido. Agora sempre limpa qualquer mirror
  órfão do mesmo nome antes de criar de novo — a função pode ser
  chamada quantas vezes precisar sem exigir limpeza manual. Testado
  com o cenário exato (mirror órfão simulado, confirma remoção antes
  da recriação)

### 0.17.6
- **Bug real corrigido — mesma causa raiz da 0.17.5, agora na chave
  do repositório central**: `gpg --import` guarda a chave no "cofre"
  moderno do GPG (keybox), mas o `aptly` verifica assinatura via
  `gpgv`, que não enxerga esse cofre sozinho — precisa de um arquivo
  de keyring explícito. Corrigido: depois de importar, a chave
  também é exportada pra um arquivo dedicado
  (`/etc/painel-repo/arx-central.gpg`), passado via `-keyring=` no
  `aptly mirror create` do repositório central. Testado: exportação
  chamada corretamente, comando final com `-keyring=` incluído

### 0.17.5
- **Bug real corrigido — verificação de assinatura do Debian
  falhava**: faltava confiar nas chaves oficiais do Debian (diferente
  da chave do repositório central do Arx, que já era importada) —
  `aptly mirror create` não conseguia verificar a assinatura do
  `Release` do Debian. Corrigido usando a keyring oficial mantida pelo
  pacote `debian-archive-keyring` (`-keyring=` no `aptly mirror
  create`), em vez de importar chave por chave manualmente (essas
  chaves rotacionam com o tempo). Checagem clara se o pacote não
  estiver instalado, em vez de deixar o `aptly` falhar com erro
  confuso. `debian-archive-keyring` adicionado como dependência do
  `arx-painel`. Testado: recusa sem a keyring, comando correto com ela

### 0.17.4
- **Bug real corrigido — `wget` não instalado na `arx-servidor`**: o
  download da chave GPG central chamava o binário `wget` externo, que
  não era uma dependência declarada do pacote (só estava no filtro do
  mirror, não no `Depends`). Corrigido trocando por `urllib` (biblioteca
  padrão do Python) — elimina essa classe inteira de bug, já que não
  depende de nenhum binário externo estar instalado. Testado: 404, host
  inalcançável, e sucesso

### 0.17.3
- **Causa raiz real do erro de configurar repositório encontrada**:
  não era a chave (estava servindo certinho, confirmado com `curl -I`)
  — era a URL enviada sem `http://` na frente. Sem o esquema, o
  `wget` trata o endereço como **arquivo local**, não como endereço
  de rede, e falha silenciosamente (a 0.17.2 já tinha corrigido a
  mensagem genérica, foi isso que permitiu enxergar a causa real)
- **Corrigido dos dois lados**: validação no backend (rejeita URL sem
  `http://`/`https://` com mensagem clara) e o campo do formulário
  agora é `type="url"` — o navegador já impede enviar sem esquema,
  antes mesmo de chegar no servidor

### 0.17.2
- **Bug real corrigido — erro genérico ao importar a chave GPG do
  repositório central**: o download+import rodava como um pipe só via
  shell, escondendo a mensagem real de erro (wget 404, arquivo vazio,
  chave inválida — tudo virava o mesmo "returned non-zero exit status
  2" sem contexto nenhum). Corrigido separando os dois passos, cada
  um com mensagem de erro específica. Testado os três cenários (404,
  vazio, conteúdo inválido) — mensagens agora mostram o motivo real

### 0.17.1
- **Bug visual corrigido — sino de alertas desalinhado no header**:
  o `.topo` usa `justify-content: space-between`, e com três itens
  (título, sino, Sair) isso distribui os três igualmente — o sino
  ficava flutuando no meio do header em vez de junto do "Sair".
  Corrigido agrupando sino e "Sair" num container só (`.topo-direita`),
  voltando a ser só dois grupos pro `space-between` (título de um
  lado, sino+Sair do outro)

### 0.17.0 — Repositório local (espelho independente por servidor)
- **Módulo novo**: cada servidor em Modo Servidor vira um espelho
  próprio e INDEPENDENTE (Debian oficial + repositório central do
  Arx, duas fontes separadas — decisão explícita de não depender do
  repositório central estar sempre no ar). Máquinas de domínio
  atualizam localmente em vez de saírem pra internet toda vez
- Usa `aptly` (confirmado disponível direto no `main` do Debian
  trixie, sem fonte externa) — mesmo filtro de pacotes já curado na
  `arx-build`, mantido num só lugar
- **Configuração inicial e sincronização** rodam como job assíncrono
  (a primeira sincronização é pesada — Debian inteiro, mesmo filtrado)
- **Agendamento configurável por servidor** — cada admin escolhe
  horário e dias da semana, sem padrão forçado. Gera um timer systemd
  dinamicamente
- Página nova (`/repositorio`): status, configuração inicial,
  sincronizar agora, agendamento
- **Importante — limite real de teste**: a lógica de encadeamento de
  comandos (`mirror update` → `snapshot create` ×2 → `merge` →
  `publish snapshot`/`switch`) foi testada e validada com `aptly`
  mockado — mas uma sincronização de verdade (rede real, GPG real,
  Debian completo) só dá pra validar no `arx-servidor`, não no
  ambiente de desenvolvimento. Testar com cuidado antes de confiar em
  produção

### 0.16.0 — sistema de 6 níveis de gravidade + sino de alertas
- **Bug real corrigido — alerta perdido quando o problema é de curta
  duração**: a checagem de alertas só rodava no timer de 5 minutos —
  se o firewall (ou qualquer coisa) quebrasse e fosse corrigido antes
  do próximo ciclo, a janela ruim nunca era vista, nunca ia pro
  histórico. Corrigido: a checagem agora também dispara sempre que
  qualquer página autenticada é carregada (via context processor) —
  se um admin está olhando bem na hora que algo está ruim, isso nunca
  escapa mais
- **Sistema de gravidade revisado — 6 níveis, não mais 3**: ok
  (verde) / info (azul) / atenção (âmbar) / alto (laranja) / crítico
  (vermelho) / emergência (roxo). Reclassificação real: firewall
  **não-inicializado** (zero proteção) era "alerta" (amarelo) — agora
  é **crítico** (vermelho), porque servidor sem nenhuma defesa é falha
  grave de segurança, não risco moderado. Firewall ativo-mas-sem-
  persistência fica em "alto" (laranja) — ainda protegido agora, mas
  frágil a um reboot. Disco ganhou um degrau novo (emergência, >98%,
  risco real de corromper banco de dados como o do Samba AD)
- **Sino de alertas no menu**, ao lado do "Sair" — número de alertas
  ativos, cor de acordo com a pior gravidade entre eles. Testado com
  login real simulado, confirma contagem e cor corretas
- Macro Jinja novo (`_macros.html`) pro badge de gravidade — reutilizado
  em Saúde, Alertas e Dashboard, evita repetir a lógica de cor em
  cada template
- Testado: os 6 níveis de disco (ok→atenção→alto→crítico→emergência),
  os dois casos de firewall (crítico vs alto), sino com múltiplos
  alertas pegando a pior gravidade entre eles

### 0.15.0 — Fase 4 do roadmap completa: alertas + relatório de diagnóstico
- **Alertas**: histórico persistido de degradações de saúde (reaproveita
  health.get_status()) — página nova mostra o que está ruim **agora**
  e o histórico dos últimos 30 dias, incluindo quando um item se
  recupera. Checagem roda junto com a coleta de métricas (a cada 5
  min). Cuidado real no design: o "último veredito conhecido" precisa
  ficar em disco, não em memória — esse script reinicia do zero a
  cada execução, guardar só em memória faria todo alerta parecer
  "novo" sempre. Testado simulando processos separados de verdade
  (reimportando o módulo), confirmando que não duplica
- **Relatório de diagnóstico**: baixa um `.txt` juntando sistema,
  saúde, serviços, firewall, NTP, DHCP, DNS e uso de recursos — cada
  seção isolada (uma falhando não derruba o relatório inteiro, já que
  o objetivo dele é ajudar justamente quando algo está quebrado).
  Testado com uma seção falhando de propósito, resto continuou saindo
- Botão de confirmação da Restauração de backup virou modal (pedido
  do usuário) — mecanismo generalizado no `componentes.js`, reutilizável
  em qualquer form que precise de confirmação **digitada**, não só
  sim/não

Com isso, a **Fase 4 do roadmap de robustez está completa (5/5)**.

### 0.14.1
- **Nome dos backups mudou**: `backup-...` → `arx-backup-...` (mesmo
  resto do formato)
- **Upload de backup**: dá pra enviar um `.tar.gz` de outra máquina
  (ou um já baixado) pra poder restaurar dele. O nome original do
  arquivo enviado é **sempre ignorado** por segurança — salvo com um
  nome novo gerado por nós, e o conteúdo é validado como `.tar.gz` de
  verdade antes de aceitar (rejeita lixo/upload vazio/base64 malformado)
- **Bug de segurança real corrigido no caminho**: não existia
  **nenhum** limite de tamanho de upload no Flask — agora limitado a
  200MB (generoso pra um backup de config, mas com teto)
- Testado de ponta a ponta: nome sempre novo mesmo pro arquivo
  enviado, validação de conteúdo tar.gz, rejeição de vazio/base64
  inválido/conteúdo que não é tar.gz de verdade

### 0.14.0 — Fase 4 do roadmap (parte 1): jobs assíncronos, backup completo, restauração
- **Jobs assíncronos**: ações demoradas (provisionar domínio, aplicar
  atualizações) agora rodam em background — retornam na hora com um
  ID, acompanhamento numa página nova (`/jobs`) que atualiza sozinha
  a cada 2s (via meta-refresh, sem JS, respeitando o CSP) enquanto
  roda. Testado: sucesso, falha, consulta de job inexistente
- **Backup completo**: empacota toda a config relevante (DHCP, DNS,
  Firewall, NTP, Samba, dados do painel) num `.tar.gz`, baixável
  direto do painel
- **Restauração**: exige digitar "CONFIRMAR" (mesmo padrão do
  provisionamento de domínio) — sempre cria um backup de segurança do
  estado ATUAL antes de sobrescrever, nunca restaura sem ter como
  voltar atrás. Reinicia os serviços afetados depois
- **Bug real corrigido durante o teste**: nome de arquivo de backup
  colidia se dois backups aconteciam no mesmo segundo (exatamente o
  cenário de "backup de segurança antes de restaurar") — um
  sobrescrevia o outro silenciosamente. Corrigido com milissegundos
  no nome, testado com o cenário de colisão explícito
- Ainda faltam os itens 4 e 5 da Fase 4 (Alertas, Relatórios de
  diagnóstico) — ficam pro próximo bloco

### 0.13.1 — rollback automático sobrevive a restart do helper
- **Lacuna real corrigida**: o agendamento do rollback automático
  (firewall e rede) vivia só na memória do processo `helper` — se ele
  reiniciasse durante a janela de 15s (ex: uma atualização do próprio
  pacote acontecendo bem nessa hora), o agendamento se perdia e a
  mudança arriscada ficava aplicada **pra sempre**, sem reverter.
  Corrigido: todo pendente agora também é salvo em disco
  (`/var/lib/painel/pendentes/`) antes de aplicar. Se o helper subir
  de novo e achar um pendente órfão (processo anterior morreu antes
  de confirmar OU de reverter), reverte sozinho **no boot**, antes
  até de aceitar qualquer requisição nova
- Testado de ponta a ponta simulando o cenário exato (aplica mudança,
  simula o processo morrendo sem confirmar, chama a recuperação como
  um boot novo faria) — pros dois casos: firewall e rede (IP fixo e
  VLAN). Confirmar também testado, limpa o arquivo em disco
  corretamente

### 0.13.0 — Fase 3 do roadmap: contadores, DNS completo, leases, sincronização real
- **Firewall — contadores de pacote/byte**: regras novas incluem
  `counter`, mostra tráfego real por regra na tabela. Regras criadas
  antes dessa versão continuam funcionando, só sem contador (até
  serem recriadas)
- **DNS — AAAA, PTR, zona reversa**: registros IPv6 (validados),
  registros PTR, e criação de zona reversa clássica (`/8`, `/16`,
  `/24` — delegação classless fica fora de escopo). Nome da zona
  reversa calculado automaticamente (ex: `192.168.1.0/24` →
  `1.168.192.in-addr.arpa`)
- **DHCP — leases ativos de verdade**: aba nova mostrando quem
  realmente pegou IP agora (lido de `/var/lib/dhcp/dhcpd.leases`),
  diferente das reservas — pega corretamente a ocorrência mais
  recente de cada IP no arquivo (que é append-only, tem histórico de
  renovações)
- **NTP — status real de sincronização**: card novo com stratum,
  offset, leap status via `chronyc tracking` — diferencia
  "configurado" de "sincronizando de verdade" (um servidor pode estar
  configurado mas sem sincronizar por trás, ex: upstream fora do ar)
- Testado de ponta a ponta: contador com/sem counter, zona reversa +
  AAAA + PTR + validações de IP/prefixo, leases com cenário de
  renovação + expirada, sincronização real vs. só-configurado

### 0.12.1
- **Bug real corrigido — rollback de VLAN deixava a interface órfã
  ativa**: reverter os arquivos `.netdev`/`.network` e reiniciar o
  `systemd-networkd` não remove a interface VLAN que já foi criada no
  kernel — o daemon só para de *gerenciar* ela, não a apaga. Corrigido
  adicionando `ip link delete <vlan>` explícito no rollback, antes do
  restart. Testado: comando chamado com o nome certo da interface no
  timeout real

### 0.12.0 — Rollback automático generalizado pra Rede
- **Sistema de rollback-com-contagem extraído do firewall.py pra um
  utilitário compartilhado** (`commands/_util.py`), reutilizável por
  qualquer módulo. `firewall.py` refatorado pra usar ele (mesmo
  comportamento, testado — só menos código duplicado)
- **Aplicado em Rede**: `set_static` (IP fixo) e `create_vlan` — os
  dois cenários que já causaram lockout real hoje. Mudar IP fixo ou
  criar VLAN agora também abre o modal de contagem regressiva, com
  rollback automático se não confirmar
- Modal de contagem generalizado no `componentes.js` — funciona em
  qualquer página com um `<dialog id="modal-contagem">`, lendo as
  URLs de confirmar/recarregar dos `data-attributes` (não fica mais
  preso só ao firewall)
- Testado de ponta a ponta com timeout REAL (15s completos, não só
  simulado): `set_static` reverte pro DHCP original se não confirmar,
  `create_vlan` remove o arquivo da VLAN sozinho se não confirmar

### 0.11.7
- **Bug real corrigido — CSP bloqueava estilo inline no painel
  inteiro**: `Content-Security-Policy: default-src 'self'` sem
  `style-src` explícito bloqueava TODO `style="..."` inline usado em
  praticamente todo template do painel — silenciosamente, sem quebrar
  o visual geral (a maior parte do design vem do `style.css`
  externo), mas descartando detalhes de ajuste fino (a centralização
  do número na contagem regressiva foi o primeiro caso visível).
  Corrigido liberando `style-src 'self' 'unsafe-inline'`
  especificamente — mantém `script` inline bloqueado (proteção real
  contra XSS), libera só estilo (risco bem menor, prática comum em
  sites com CSP rigoroso). Arquivo de config do nginx não é
  `conffile`, então a correção aplica sozinha na próxima atualização

### 0.11.6
- **Bug real corrigido — modal de contagem regressiva nunca abria**:
  causa raiz era o próprio CSP (`default-src 'self'`, configurado lá
  na Fase 1 de segurança) bloqueando um `<script>` inline
  colocado direto no template — primeiro caso de JS inline
  nesse projeto, e o navegador bloqueou certinho (CSP funcionando
  como devia). Corrigido movendo toda a lógica pro
  `componentes.js` (arquivo externo, permitido), passando os valores
  via `data-attributes` no HTML em vez de interpolação Jinja dentro
  de `<script>`. Confirmado: zero script inline no HTML renderizado

### 0.11.5
- **Limpeza de UX**: removido o banner permanente "Acabou de mudar
  uma regra?" (aparecia sempre, mesmo sem nenhuma mudança pendente) e
  a mensagem de flash duplicada — agora só o modal de contagem
  regressiva mostra o aviso, e só quando existe mesmo uma mudança
  pendente de confirmação
- **Ainda investigando**: o modal não abriu automaticamente no teste
  do usuário — renderização do template e sintaxe do JS validadas
  sem erro no ambiente de desenvolvimento, causa real ainda não
  identificada, aguardando console do navegador pra diagnosticar

### 0.11.4
- **Contagem regressiva visual no rollback automático do firewall**:
  em vez de só um banner estático de texto, agora abre um modal
  automaticamente logo depois de liberar porta ou remover regra, com
  o número de segundos restantes contando ao vivo. Se o tempo acabar
  sem confirmar, a página recarrega sozinha — mostra visualmente se a
  regra reverteu ou não, sem precisar adivinhar ou clicar em nada. O
  banner estático continua como reforço (caso o JS falhe, ou o admin
  saia e volte pra página dentro da janela de tempo)

### 0.11.3
- **Bug crítico corrigido — rollback automático (dead-man's switch)
  também duplicava regras**: mesma causa raiz do bug corrigido na
  0.11.2 (falta de `flush ruleset;`), só que escondida num segundo
  lugar — a captura do estado "antes" usada pra reverter uma mudança
  não confirmada. Reportado pelo usuário: adicionou uma porta, não
  confirmou, e a reversão automática duplicou as 4 regras base em vez
  de só desfazer a porta nova. Corrigido e testado com o cenário
  exato reportado

### 0.11.2
- **Bug crítico corrigido — regras de firewall duplicando a cada
  restart do `nftables`**: o arquivo persistido em
  `/etc/nftables.conf` nunca tinha um `flush ruleset;` no início —
  toda vez que o serviço reiniciava com as regras já carregadas, o
  conteúdo do arquivo era SOMADO ao que já estava rodando em vez de
  substituir. Corrigido: `_persistir()` sempre prefixa `flush
  ruleset;`, garantindo que recarregar o arquivo sempre parte do
  zero. Testado — arquivo persistido agora sempre começa com o flush

### 0.11.1
- **Bug real corrigido — `updates.apply()` escondia o erro real do
  `apt`**: quando `apt-get install --only-upgrade` falhava, a saída
  real (motivo de verdade da falha) ficava presa dentro da exceção
  sem ser usada — o painel só mostrava "erro interno" genérico. Agora
  captura e mostra a mensagem real do `apt` na tela, sem precisar
  abrir `/var/log/painel/helper.log` pra descobrir o motivo

### 0.11.0 — Fase 3 do roadmap: rollback automático do firewall
- **Firewall — rollback automático por timeout (dead-man's switch)**:
  toda mudança que pode cortar acesso (liberar porta, remover regra)
  agora aplica e **desfaz sozinha em 15 segundos** se o admin não
  confirmar — mesmo padrão que switch/roteador profissional usa.
  Teria evitado os dois lockouts reais de hoje (porta 9006, VLAN).
  Botão "Confirmar última mudança" na página de Firewall cancela a
  reversão. Testado de ponta a ponta: reverte sozinho sem
  confirmação, mantém a mudança quando confirmado
- Esse é o primeiro item de uma leva maior (Fase 3 completa envolve
  também: contadores de pacote/byte no firewall, AAAA/PTR/zona
  reversa no DNS, leases ativos no DHCP, status de sincronização real
  no NTP, administração mais completa do Samba AD) — os próximos
  vêm em blocos separados, este era o mais crítico de segurança

### 0.10.0 — Fase 2 do roadmap de robustez: diagnóstico de rede, SMART, histórico de métricas
- **Diagnóstico de rede** (item 15): nova página com ping, traceroute,
  consulta DNS (A/AAAA/MX/TXT/NS/CNAME) e teste de porta TCP. Validação
  rígida de host/porta antes de qualquer subprocess, protegido contra
  injeção — testado
- **Discos/SMART** (item 17): lista discos físicos, mostra saúde SMART
  (saudável/falha/indisponível) com atributos-chave, nunca quebra em
  disco virtual sem suporte a SMART
- **Histórico de métricas** (item 16 + item 28 da Fase 4): timer
  systemd novo (`arx-painel-metrics.timer`) grava uma amostra de
  CPU/memória/disco a cada 5 minutos, com poda automática (48h de
  retenção). Card novo em Saúde do sistema mostra mín/média/máx das
  últimas 24h. Essa é a peça que faltava pros gráficos do dashboard
  que ficaram adiados desde a 0.4.0 — a base agora existe de verdade
- **Novas dependências**: `traceroute`, `smartmontools`, `dnsutils`
  (essa já estava no filtro do mirror; as duas primeiras precisam ser
  adicionadas manualmente, mesmo processo do `ethtool`)
- Testado de ponta a ponta: histórico de métricas grava e lê
  corretamente, validações de segurança rejeitam host/porta inválidos,
  todos os templates novos renderizam sem erro (com e sem dado
  preenchido)

### 0.9.2
- **Bug visual real corrigido — retângulos pretos gigantes no card do
  hostname (dashboard)**: essa era a causa do "painel quebrando"
  reportado na 0.9.0 (não era o serviço caindo — era só um visual
  quebrado feio o bastante pra parecer erro grave, achado via
  DevTools). O ícone SVG do card não tinha tamanho definido porque a
  regra de CSS que limita isso só valia dentro de um wrapper
  `.card-icone`, e esse ícone específico não tinha esse wrapper — sem
  a regra bater, o navegador usava o tamanho padrão dele pra SVG sem
  medida (300×150px). Corrigido tornando a regra geral (não depende
  mais do wrapper), evitando essa classe inteira de bug se eu
  esquecer o wrapper em outro lugar de novo

### 0.9.1 — apelido fixo "arx.os" pra acessar o painel
- **`https://arx.os` funciona além do IP** — protegido, não editável
  pelo admin (mesmo padrão das regras estruturais do firewall):
  existe de verdade como zona DNS interna, mas nunca aparece na lista
  de zonas pra editar/apagar. Criado e mantido sozinho (verifica e
  atualiza automaticamente o IP toda vez que a página DNS carrega,
  sem gerar ruído na auditoria)
- **Depende de**: o DNS deste servidor estar ativo, e as máquinas da
  rede usarem ele como servidor DNS (configurável na aba "Opções
  padrão" do DHCP) — sem isso, só funciona acessando via IP mesmo,
  como sempre funcionou
- Certificado autoassinado agora inclui `arx.os` como nome
  alternativo (evita aviso extra do navegador além do esperado por
  ser autoassinado) — só em instalação nova, não sobrescreve
  certificado já existente
- `nginx` responde tanto por IP quanto por `arx.os`
- Testado de ponta a ponta: cria, fica invisível pro admin, é
  idempotente, atualiza sozinho se o IP mudar

### 0.9.0 — Dashboard e Serviços redesenhados (referência visual do usuário)
- **Página de Serviços**: busca em tempo real, filtro por status
  (Todos/Ativos/Inativos), descrição de cada serviço, badges
  coloridos (ATIVO/HABILITADO), botões em português (Iniciar/Parar/
  Reiniciar), contador de itens exibidos
- **Dashboard**: card de IP principal + uptime, tabela de serviços
  essenciais embutida, painel de "Ações rápidas" (Reiniciar sistema,
  Desligar sistema — **ações reais novas**, protegidas por modal de
  confirmação bem explícito; Verificar atualizações, Configurar rede,
  Logs, Gerenciar usuários — linkam pras páginas reais), Eventos
  recentes (reaproveita o log de auditoria)
- **Ações novas no backend**: `system.reboot()`/`system.shutdown()`
  — reiniciam/desligam a máquina de verdade via `systemctl`
- **Adiado de propósito** (mesma lacuna documentada desde a 0.4.0):
  mini-gráficos de histórico (sparklines) e "Uso de recursos últimas
  24h" — precisam de um componente de histórico de métricas que
  ainda não existe. Não implementado com dado inventado
- **Checagem de colisão de rotas** agora roda sempre antes de
  empacotar (achou o bug da 0.8.2, mantida pra sempre)

### 0.8.2
- **Bug crítico corrigido — causa raiz REAL do "Internal Server Error"
  em Ver registros de zona DNS**, que persistia desde a 0.6.0: um
  decorador `@app.route("/dns/zona/<zona>")` tinha vazado por engano
  em cima de `dns_zona_criar()` numa edição anterior, duplicando essa
  rota — como foi registrada primeiro, o Flask sempre chamava
  `dns_zona_criar()` pra qualquer GET em `/dns/zona/<qualquer coisa>`,
  e essa função não aceita o parâmetro `zona`, gerando o erro cru.
  A rota certa (`dns_zona()`) nunca era alcançada. Só foi possível
  achar com o traceback real (`journalctl -u arx-painel-webui`) — a
  blindagem genérica de erros da 0.7.4 não pegava isso porque o
  problema é de ROTEAMENTO, não de comunicação com o helper
- **Checagem nova adotada antes de empacotar**: importa o `app.py`
  de verdade e confere se existe alguma combinação rota+método
  registrada duas vezes apontando pra endpoints diferentes — teria
  pego esse bug específico antes de qualquer versão publicada

### 0.8.1 — abas em todas as páginas + formulários organizados
- **Abas aplicadas em todas as páginas com múltiplas seções**: DHCP
  (Sub-redes / Reservas / Opções padrão), DNS (Zonas / Forwarders),
  Firewall (Regras ativas / Liberar porta), Domínio (Usuários / Criar
  usuário), Usuários locais (Usuários / Criar usuário), Configurações
  (Sistema / Serviços), Grupos (Grupos existentes / Criar grupo), Hora
  NTP (Servidor upstream / Redes liberadas), Atualizações (Ações /
  Trocar canal). Dashboard e páginas de conteúdo único (Logs, Saúde,
  Interfaces de rede) continuam sem abas, como combinado
- **Formulários padronizados**: todo campo agora usa `.campo` (label
  numa linha acima do input, espaçamento consistente, largura máxima),
  em vez do padrão antigo "Rótulo: campo" na mesma linha — aplicado
  em ~45 campos, em todos os templates
- **Confirmação por modal estendida**: remover usuário local, remover
  reserva DHCP, remover grupo, remover rede liberada no NTP — mesma
  proteção que já tinha em firewall/subnet/zona

### 0.8.0 — DHCP: opções de serviço + componentes de página do design system
- **DHCP — opções padrão vs por sub-rede**: lease time (padrão/máximo)
  e DNS agora configuráveis globalmente, com override opcional por
  sub-rede (lease time específico na criação). Bug real corrigido no
  processo: `get_default_options()` usava regex sem `re.MULTILINE`,
  só achava a opção se estivesse literalmente na primeira linha do
  arquivo — testado e corrigido antes de empacotar
- **Modal de confirmação** (`<dialog>` nativo, sem framework): aplicado
  nas ações mais arriscadas — remover regra de firewall, remover
  sub-rede DHCP, remover zona DNS
- **Abas**: página de Rede reorganizada (Interfaces / IP fixo / VLAN)
- **Paginação client-side**: página de Logs (200 entradas, 20 por
  página) — primeiro lugar com volume real de dados pra isso valer a pena
- **Chips removíveis**: campo de forwarders do DNS, Enter ou vírgula
  pra adicionar, X pra remover
- Componentes compartilhados em `static/js/componentes.js`, prontos
  pra reutilizar em outras páginas quando fizer sentido

### 0.7.8
- **Correção real do fix de DHCPv6 da 0.7.7** (que estava incompleto):
  a lógica do `/etc/init.d/isc-dhcp-server` só evita subir IPv6 quando
  `INTERFACESv4` tem algo (não vazio) — deixar só `INTERFACESv6=""`
  vazio não bastava, porque a regra "os dois vazios = sobe os dois
  juntos" continuava valendo. `postinst` agora detecta as interfaces
  físicas reais da máquina (excluindo lo/veth/docker/etc) e preenche
  `INTERFACESv4` com elas, em vez de deixar vazio ou chutar um nome
  fixo

### 0.7.7
- **Causa raiz real do "Serviço: failed" persistente encontrada**: o
  init do `isc-dhcp-server` tenta subir IPv4 **e** IPv6 juntos por
  padrão — o painel nunca configurou DHCPv6 em lugar nenhum, então
  essa metade sempre falhava, e o `systemd` reportava o **serviço
  inteiro** como falho mesmo com o IPv4 certinho (o processo `dhcpd`
  IPv4 ficava rodando de verdade em background, só o status geral que
  mentia). `postinst` agora desativa `INTERFACESv6` na primeira
  instalação, já que não é suportado mesmo

### 0.7.6
- **Range de sub-rede DHCP agora é opcional**: o `dhcpd` exige uma
  declaração de sub-rede (mesmo vazia) pra CADA interface real da
  máquina, senão recusa subir inteiro ("No subnet declaration for
  ..."). Antes, o formulário exigia range obrigatório, então não
  dava pra declarar uma rede "só de aviso" sem servir DHCP nela —
  exatamente a causa real por trás do "Serviço: failed" reportado.
  Agora dá pra deixar o range em branco

### 0.7.5
- **Bug crítico corrigido — criar VLAN podia trancar o admin de fora**:
  o arquivo que associa a VLAN à interface física (`15-{iface}-vlans.network`)
  só tinha `VLAN=X`, sem nenhum endereçamento — e como o nome dele
  ordena antes do arquivo padrão da base (`20-wired.network`), o
  systemd-networkd aplicava ele primeiro, derrubando o IP da
  interface na hora. Corrigido: agora sempre localiza o arquivo que
  já governa a interface antes de criar a VLAN — se for específico
  dela, edita no lugar (preserva IP fixo/DHCP existente); se só um
  wildcard cobrir (`en*`), nunca mexe nele direto (afetaria outras
  interfaces também) e cria um arquivo específico novo replicando o
  mesmo endereçamento; se nada cobrir, usa DHCP=yes como fallback
  seguro. Testado nos três cenários
- **Bug real corrigido — DHCP reportava "erro interno" mesmo quando a
  mudança já tinha sido aplicada**: `create_subnet`/`delete_subnet`
  escreviam o arquivo com sucesso, mas se o `systemctl restart
  isc-dhcp-server` falhasse depois (ex: falta declarar sub-rede pra
  alguma interface real da máquina), o erro não era tratado — virava
  "erro interno" genérico E deixava o arquivo com a mudança gravada
  mesmo com o serviço quebrado. Corrigido: agora reverte o arquivo
  também nesse caso, e a mensagem de erro mostra a causa real (via
  `journalctl`), não mais genérica

### 0.7.4
- **Suporte a zona `forward` (redirecionamento)**: antes só dava pra
  criar zona `master`. Agora o formulário tem seletor de tipo — zona
  forward não tem arquivo próprio, só encaminha consultas pra outro
  servidor DNS. "Ver registros" só aparece pra zona master (forward
  não tem registros, mostra mensagem clara em vez de erro)
- **Bug real corrigido**: o regex de leitura de zonas exigia uma
  cláusula `file`, que zona forward não tem — mesmo se o tipo
  existisse, nunca apareceria na listagem. Corrigido (cláusula file
  agora é opcional no regex)
- **Robustez geral**: `helper_client.py` agora converte QUALQUER erro
  inesperado de comunicação (socket recusado, timeout, helper
  reiniciando no meio de uma chamada) em erro tratável — protege
  todas as ~40 rotas do painel de uma vez, não só uma corrigida
  isoladamente. Testado de ponta a ponta: criar zona master, criar
  zona forward, ver registros, remover as duas

### 0.7.3
- **Bug real corrigido — DHCP mostrava sub-redes de EXEMPLO comentadas
  como se fossem ativas**: o regex de leitura (`_SUBNET_RE`) achava o
  texto `subnet X netmask Y { ... }` em qualquer lugar do arquivo, sem
  conferir se a linha começava com `#` (comentário) — por isso as 6
  sub-redes de exemplo padrão do `dhcpd.conf` do Debian (todas
  comentadas, nenhuma ativa de verdade) apareciam na listagem, com
  Range "None - None" porque o parse dentro delas também vinha
  quebrado. Corrigido: `list_subnets()` e `delete_subnet()` agora
  conferem se cada bloco está numa linha comentada antes de
  considerar ele — testado com um arquivo simulado (blocos comentados
  + um real), confirmando que só a sub-rede real é reconhecida

### 0.7.2
- **Bug crítico corrigido — helper não subia de jeito nenhum**: um
  erro de edição na 0.7.0 apagou a linha `def list_reservations`
  do `dhcp.py`, deixando o corpo da função "grudado" como código morto
  dentro de `delete_subnet` — sintaticamente válido (por isso passou
  no `py_compile`), mas a função deixou de existir de verdade no
  módulo. O helper falhava no import logo na inicialização
  (`AttributeError`), em loop de restart. Corrigido, e agora testado
  de verdade: toda ação registrada no `helper.py` é checada contra
  o módulo real antes de empacotar (importando de fato, não só
  validando sintaxe — sintaxe válida não garante que a função existe)

### 0.7.1
- **Bug real corrigido — serviços não voltavam sozinhos depois de
  atualizar**: o `prerm` sempre parava os dois serviços antes de trocar
  os arquivos (correto), mas nada no `postinst` os religava depois. O
  "systemctl start ..." que aparecia na mensagem final da instalação
  era só texto pro admin ler, nunca foi comando executado de verdade.
  Resultado: toda atualização deste pacote até agora (0.4.x até 0.7.0)
  exigiu reiniciar manualmente — não era o admin esquecendo, era o
  pacote que nunca fazia isso sozinho. Corrigido: numa atualização
  (nunca na primeira instalação, que continua manual de propósito),
  os serviços reiniciam sozinhos no final do `postinst`

### 0.7.0 — itens de rede/DHCP que ficaram adiados
- **Interfaces de rede**: botões up/down/reconfigure direto na tabela
  (via `networkctl`, nativo do systemd-networkd)
- **Detecção de interface nova**: coluna "Configurada" mostra se a
  interface já bate com algum `[Match] Name=` existente (incluindo
  wildcard tipo `en*`) ou se é uma interface nunca vista pela config
- **Diferenciação fibra vs par trançado**: via `ethtool` (nova
  dependência do pacote) — mostra "desconhecido" com segurança se a
  ferramenta não estiver disponível ou a interface não suportar a
  consulta, nunca quebra a listagem inteira
- **DHCP — CRUD de sub-rede por campo estruturado**: criar e remover
  sub-rede sem precisar do modo avançado, com a mesma validação de
  sintaxe + backup/reversão automática do resto do módulo

### 0.6.4
- **Bug real corrigido — rodapé da sidebar "vazando" pra fora da
  tela**: faltava `box-sizing: border-box` no CSS. Sem isso, o padding
  do sidebar (20px em cima + embaixo) **somava** aos 100vh de altura
  em vez de ficar contido dentro dela — o sidebar ficava uns 40px mais
  alto que a tela, e como é `position: fixed`, essa sobra "vazava" pra
  baixo do que dava pra ver, sem scroll pra alcançar. Corrigido
  aplicando `box-sizing: border-box` globalmente (prática padrão que
  evita essa classe inteira de bug em qualquer elemento do site)

### 0.6.3
- **Bug real corrigido na Auditoria**: `system.resource_status` e
  `updates.check` apareciam no log mesmo sendo consultas de só-leitura
  — a checagem usava prefixo do nome (`status`/`list_`/`get_`), e
  esses dois nomes não batiam com nenhum (já era o segundo bug desse
  tipo, depois de `audit.tail`/`health.verificar`). Trocado por lista
  EXPLÍCITA de ações só-leitura, com o padrão invertido pra ser mais
  seguro: qualquer ação nova, se esquecida de listar, **entra** no
  log por padrão (nunca perde auditoria de uma mudança real por
  acidente de nome)

### 0.6.2
- **Bug de UX corrigido**: `nftables` aparecia como "Inativo" na lista
  genérica de Serviços (Configurações), contradizendo a página
  Firewall/Saúde que mostrava "ativo e persistente" — os dois estavam
  certos, só que `nftables.service` é um serviço *oneshot* (roda uma
  vez no boot e termina, fica "inactive" mesmo com as regras
  carregadas e funcionando). Removido da lista genérica — a página
  Firewall já é o lugar certo pra ver o estado real, com os sinais
  corretos (regras carregadas + persistência), não o `is-active` bruto

### 0.6.1
- **Regras estruturais do firewall (Loopback, Conexão estabelecida)
  não podem mais ser removidas** — nem pelo botão (agora escondido
  pra essas duas) nem por requisição direta ao helper (checagem real
  no backend). Removê-las quebraria o firewall inteiro na hora,
  inclusive a sessão de quem estivesse mexendo — risco pior que o da
  porta 9006 que já causou um bloqueio acidental antes

### 0.6.0 — Fase 1 completa + pacote grande de melhorias
- **Health Checks** (fecha a Fase 1 — 10/10): módulo novo agrega
  serviços críticos + disco + firewall num veredito único
  (ok/alerta/crítico), nova página **Saúde do sistema**, badge real
  no topo do dashboard (linkando pra lá)
- **Bug crítico corrigido — firewall não persistia**: `systemctl
  enable nftables` rodava sem checar se funcionou de verdade; se
  falhasse silenciosamente, as regras pareciam persistir mas somiam
  no próximo reboot. Corrigido: agora verifica com `is-enabled` e
  avisa alto se falhar. Status do firewall mostra se está realmente
  persistente, com botão "Reforçar persistência" se não estiver
- **DHCP vinha com sub-redes de exemplo do Debian já ativas** — não
  era o painel, é o pacote `isc-dhcp-server` que já vem assim.
  Corrigido: `postinst` zera pra uma base mínima genuína, só na
  primeira instalação (nunca num upgrade, pra não apagar config real)
- **DNS**: criar e remover zona agora funciona pela interface (antes
  só dava pra ver zonas já existentes, nunca criar uma nova)
- **Modo avançado (DHCP/DNS)**: três botões — Salvar sem aplicar /
  Salvar e aplicar / Cancelar — antes só existia "aplicar direto"
- **Sidebar fixa de verdade** — trocado `sticky` por `fixed`, o
  rodapé não "foge" mais pra baixo em páginas com pouco conteúdo
- **Versão do painel no rodapé** — lida automaticamente do pacote
  instalado, sem precisar editar nada manualmente
- **Confirmação de senha** ao criar usuário local (campo duplicado,
  checado no servidor)
- **Hostname editável** em Configurações
- **Botões de serviço** (start/stop/restart) — layout corrigido pra
  garantir mesma linha, cores diferentes por ação (azul/vermelho/amarelo)
- Regras de firewall mais legíveis (traduzidas da sintaxe nft crua)

**Adiado de propósito pra próxima rodada** (grande demais pra
encaixar com qualidade nessa mesma leva):
- CRUD completo de sub-rede DHCP por campo estruturado (hoje só dá
  pra criar/editar via modo avançado ou reserva dentro de sub-rede
  já existente)
- Configurar opções de DHCP padrão vs por sub-rede (DNS/gateway/lease)
- Start/stop/restart de interface de rede pela interface web
- Detectar interfaces novas ainda não configuradas
- Diferenciar interfaces de fibra vs par trançado

### 0.5.0 — Auditoria + Backup/Rollback generalizado
- **Auditoria**: toda ação que muda algo no sistema fica registrada em
  `/var/log/painel/auditoria.jsonl` — quem (usuário logado), de onde
  (IP), o quê (ação + parâmetros, senha sempre mascarada), quando, e
  se deu certo. Consultas de status não entram no log, pra não afogar
  o histórico. Nova página **Logs** no menu mostra as últimas 200
  entradas
- **Backup/Rollback generalizado**: `network.py` (IP fixo, DHCP, VLAN)
  e `ntp.py` (servidor upstream, redes liberadas) agora fazem backup
  antes de escrever e revertem sozinhos se o `systemctl restart`
  falhar — mesmo padrão que já existia isolado em `dhcp.py`/`dns.py`
  modo avançado, agora com um utilitário compartilhado
  (`commands/_util.py`)
- **Regras de firewall mais legíveis** — em vez de mostrar só a
  sintaxe crua do `nft`, a tabela agora traduz cada regra (ex: "Porta
  liberada — TCP porta 22 — de qualquer lugar"), com a regra técnica
  ainda visível embaixo, em texto pequeno, pra quem quiser conferir

### 0.4.5
- **Bug real corrigido**: `firewall.status()` nunca listava as regras
  ativas, mesmo existindo de verdade no kernel — faltava a flag `-a`
  no comando `nft list table` (sem ela, o nft não inclui os números de
  handle, e o parser não reconhecia nenhuma linha como regra). Achado
  testando na prática: `nft list ruleset` mostrava regras reais, mas
  o painel mostrava lista vazia

### 0.4.3 — refinamento visual (componentes compartilhados)
- Alertas redesenhados — ícone + título + mensagem, por categoria
  (sucesso/erro), propaga pra todo o painel via `base.html`
- Hierarquia de botões — `.btn-perigo` nas ações destrutivas
  (Remover/Bloquear) em todas as páginas, visualmente diferente dos
  botões de ação normal
- Status com ponto colorido (`.status-dot`) na tabela de interfaces
- `<h1>` duplicado removido de todas as páginas — o título já aparece
  na barra do topo via `block titulo`, ter os dois era redundante
- Componentes novos adicionados ao CSS pra uso futuro: switch
  (liga/desliga) e card compacto (item de lista com ícone+status)
- **Ainda não implementado** (fica pra depois de fechar a Fase 1):
  abas, paginação, modal de confirmação, chips/tags removíveis,
  dropdown com descrição

### 0.4.2 — revisão de código
- **Bug corrigido**: `dns.get_zone_records`/`add_record`/`remove_record`
  assumiam que todo arquivo de zona seguia o padrão
  `/etc/bind/zonas/db.<zona>` — agora usam o caminho REAL, lido do
  `named.conf.local` (via `list_zones()`), então funciona com qualquer
  convenção de nome
- **Limitação documentada e erro melhorado**: `groups.publish_group`
  só funciona na VM que hospeda o repositório — agora dá erro claro
  em vez de traceback confuso quando rodado num servidor comum
- `groups.delete_group` agora avisa se o grupo removido já estava
  publicado (o pacote fica órfão no repo, não é limpo automaticamente)
- Documentada a desconexão entre os módulos Domínio (Samba AD) e DNS
  (BIND9) — não se integram hoje, é decisão de arquitetura pra depois

### 0.4.1
- **Bug real corrigido**: `firewall.init_defaults()` liberava SSH por
  padrão mas não a porta do próprio painel (9006) — ativar o firewall
  pelo painel bloqueava o acesso ao próprio painel (só sobrava SSH
  pra corrigir na mão). Corrigido: porta do painel entra na lista
  segura padrão, mesmo motivo do SSH
- **Bug real corrigido**: regras do `nftables` só existiam em memória,
  nunca eram salvas em disco — um reboot apagava tudo silenciosamente,
  sem aviso nenhum. Corrigido: toda mudança agora persiste em
  `/etc/nftables.conf` e garante que carrega sozinho no boot

### 0.4.0
- **Reformulação visual completa** — sidebar fixa com ícones (troca o
  menu dropdown no topo), cards com ícone + badge de status colorido,
  barra de progresso nos recursos do sistema, seguindo referência
  visual mais "futurista"/profissional
- Dashboard mostra hostname + kernel, e status de todos os módulos
  (Firewall, Atualizações, Domínio, DHCP, DNS, NTP) com badge colorido
- **Adiado de propósito**: mini-gráficos de histórico (sparklines) nos
  cards de CPU/memória — precisam de um componente de histórico de
  métricas ao longo do tempo, que ainda não existe. Boa sinergia com
  os itens pendentes de Auditoria/Health checks da Fase 1 — fica pra
  quando atacarmos isso

### 0.3.3
- Tela de login redesenhada — logo centralizada no topo, campos com
  ícone embutido, fundo com vinheta radial escura, mantendo a paleta
  de cores já usada no resto do painel

### 0.3.1
- Corrigido: `domain.py` desatualizado publicado junto com `helper.py`
  mais novo (referenciava `install_samba` que não existia na versão
  empacotada) — causava `AttributeError` e o helper nunca subia.
  Adotada prática de sempre subir o patch version a cada correção,
  mesmo em desenvolvimento ativo da mesma "versão maior" — evita
  colisão no pool do `aptly` (que recusa sobrescrever arquivo com
  mesmo nome e conteúdo diferente, proteção de integridade correta)
- Adotado processo de empacotar `painel/` inteira como `.tar.gz` em
  vez de copiar arquivo por arquivo — elimina essa classe de bug de
  dessincronia entre arquivos relacionados

### 0.3.0
- **HTTPS de fábrica** — porta **9006** (não a 443 padrão, decisão
  deliberada — reduz ruído de scanner automatizado, mas NUNCA deve ser
  tratado como segurança de verdade; a defesa real continua sendo
  TLS + autenticação + rate limiting + firewall restringindo acesso à
  VLAN de gerência), certificado autoassinado gerado sozinho no
  `postinst` (navegador avisa "não confiável" na primeira vez, é
  esperado pra painel interno sem domínio público), headers de
  segurança (HSTS, X-Frame-Options, CSP restrito)
- **Sessão endurecida** — cookie `Secure` (só HTTPS) + `HttpOnly` +
  `SameSite=Lax`, expira sozinha depois de 30min sem atividade
- **Módulo DHCP** (isc-dhcp-server) — subnets e reservas já
  interpretados, com filtro por VLAN/MAC/IP; modo avançado com config
  completa (checa sintaxe com `dhcpd -t` antes de aplicar, reverte
  sozinho se falhar)
- **Módulo DNS** (BIND9) — forwarders, zonas, registros A/CNAME/MX/TXT
  (com incremento automático do Serial do SOA); modo avançado com
  `named-checkconf`/`named-checkzone` antes de aplicar
- **Módulo Hora (NTP)** (chrony) — define servidor upstream, libera
  redes/VLANs pra sincronizar com este servidor
- **Proteção contra força bruta no login** — bloqueio temporário após
  5 tentativas erradas em 5 minutos (15 min de bloqueio), rastreado
  em arquivo compartilhado entre os workers do gunicorn (com lock via
  `fcntl`, já que memória de processo não é compartilhada entre eles)
- **Menu reorganizado em grupos** (Rede / Contas / Sistema) — uma
  barra única não aguentava mais o número de módulos
- **Dashboard reescrito**: resumo de recursos do sistema (CPU/memória/
  disco/uptime) + status resumido de cada módulo, tudo somente
  leitura, cada card linkando pra página de configuração real
- `build-deb.sh` lê a versão automaticamente do `control` — não
  precisa mais sincronizar o número em dois arquivos manualmente

### 0.2.0
- nginx empacotado e ativado sozinho no `postinst` (porta 8080, HTTP —
  TLS fica pra próxima versão)
- `ARX_PAINEL_SECRET_KEY` gerada automaticamente na instalação
- Corrigido bug de arquitetura: `RuntimeDirectory` compartilhado entre
  os dois serviços agora sobrevive a restart de qualquer um dos dois
  (`RuntimeDirectoryPreserve=yes` + mesmo `Group=` nos dois `.service`)
- Corrigido: mensagens de erro reais (`ValueError`/`RuntimeError`) do
  helper agora chegam até a interface — antes qualquer erro virava um
  "erro interno ao executar a ação" genérico, mesmo quando o código já
  tinha uma mensagem específica e útil pronta
- `criar_admin.py` agora ajusta o dono do arquivo de credencial pro
  usuário `painel-web` (antes ficava `root:root 600`, webui não
  conseguia nem ler o próprio arquivo)
- Timeout do `helper_client` subiu de 10s → 120s (ações como
  `apt-get update` ou `samba-tool domain provision` são lentas)
- `updates.set_channel` corrigido — reaproveita a URL real já
  configurada em vez de gravar um hostname inventado
- Dependências completas do Samba AD DC: `samba-ad-dc`,
  `samba-ad-provision`, `samba-common-bin`, `krb5-user`
- `domain.status()` não dá mais falso positivo — checa a existência de
  `sam.ldb` em vez de procurar texto no `smb.conf` (que já vem com
  exemplo comentado que batia com a busca antiga)
- `domain.provision()` agora faz backup do `smb.conf` existente antes
  de provisionar (samba-tool exige gerar do zero), copia o `krb5.conf`
  gerado pro lugar certo, e ativa o serviço `samba-ad-dc` correto
- Botão "Instalar Samba" na página de Domínio, pra quando a dependência
  não veio junto por algum motivo
- Primeiro Samba AD DC provisionado e validado com sucesso

### 0.1.x
- Esqueleto inicial (webui + helper), módulos de Rede e Atualizações
- Autenticação, CSRF, módulos de Firewall/Domínio/Usuários/Grupos
- Primeiro empacotamento `.deb`

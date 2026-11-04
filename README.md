# Instalação — Five Monitor

## 1. Arquivos e requisitos

Este pacote usa o TURN incluído para Windows, a NUI existente com a logo Five Community e código Lua aberto.

- FXServer para Windows com suporte ao runtime Node 22 e à permissão `add_unsafe_child_process_permission`.
- Base vRP/CreativeNW iniciada antes deste resource, para a configuração de permissões fornecida.
- Acesso administrativo ao Windows da VPS para configurar o firewall.
- Não precisa instalar Node.js ou executar `npm install` na VPS: o resource usa o runtime do FXServer.
- Não precisa criar conta na Cloudflare para usar o TURN incluído.

Extraia a pasta `five_monitor` do ZIP dentro de `resources` da sua base. Evite uma pasta duplicada como `five_monitor/five_monitor`.

Estrutura esperada:

```text
resources/
└── five_monitor/
    ├── fxmanifest.lua
    ├── client/
    ├── server/
    ├── shared/
    ├── config/
    ├── nui/
    └── turn/
```

Mantenha o nome `five_monitor`. Não é necessário iniciar o script original junto com a reconstrução.

## 2. Configurar o server.cfg

No `server.cfg` realmente carregado pelo FXServer, coloque este bloco **depois da inicialização do vRP da sua base**:

```cfg
# Permite que este resource inicie os executáveis TURN incluídos.
add_unsafe_child_process_permission five_monitor

# Opcional: webhook privado para enviar os clipes ao Discord.
set fm_clip_webhook "https://discord.com/api/webhooks/SEU_ID/SEU_TOKEN"

# Inicia o resource.
ensure five_monitor
```

Substitua a URL de exemplo pelo webhook real. Se não quiser enviar clipes, remova a linha do webhook.

Use `set`, sem `setr`: o webhook deve permanecer no servidor. Não coloque a URL na NUI ou em arquivos compartilhados.

Na primeira instalação, reinicie o servidor para carregar o bloco completo. Se a pasta não aparecer na lista de resources, execute `refresh` no console do FXServer antes de iniciar.

## 3. Permissões e ID da sua base

O pacote está configurado em `config/server.lua` para a API encontrada na base fornecida:

```lua
Framework = "creativeNW",
AdminPermissionName = "Admin",
TargetIdentifier = "framework",
VrpVariant = "creativeNW",
ProxyInterface = "vRP",
ProxyProtocol = "request-id",
Permission = nil,
```

- O acesso usa o grupo `Admin` da base.
- O campo de busca da interface usa o **ID/passaporte da base**.
- Internamente, o servidor converte esse ID para a conexão FiveM do jogador.
- O jogador procurado precisa estar conectado.
- `AllowSelfWatch = false` impede assistir à própria tela; teste com dois jogadores.

Nesta configuração, adicionar apenas uma ACE não substitui a permissão `Admin` do framework. O campo `AcePermission` é usado pelo modo standalone.

Não use `Permission = function(source) return source == 1 end` para liberar o passaporte 1: `source` representa o ID temporário de conexão. Não duplique a chave `Permission` na mesma tabela.

Se sua instalação já tem permissões personalizadas funcionando, preserve o `config/server.lua` local.

## 4. Configurar o TURN

Arquivo: `config/turn.json`.

O modo automático inicia os executáveis da pasta `turn` junto com o resource. Exemplo com descoberta automática do IP público:

```json
{
  "enabled": true,
  "autoStart": true,
  "publicIp": "",
  "discoverPublicIp": true,
  "port": 3478,
  "realm": "five_monitor",
  "username": "stream",
  "password": "",
  "udp": true,
  "tcp": true
}
```

**Se mudar de VPS, substitua `publicIp` pelo IP público correto.** Como alternativa, use `publicIp` vazio e `discoverPublicIp: true` para descoberta automática.

Com `autoStart: true`, o campo `password` vazio permite gerar a senha automaticamente durante a inicialização. Não precisa preencher `IceServers` em `config/server.lua` para usar o TURN gerenciado incluído.

Não inicie manualmente outra cópia dos EXEs na mesma porta enquanto o modo automático estiver ativo.

### Firewall do Windows

Depois de iniciar o resource, abra o **PowerShell como Administrador na VPS** e execute este comando inteiro em uma única linha:

```powershell
Get-CimInstance Win32_Process | Where-Object { $_.Name -in @('turn-server-simple.exe','turn-server-tcp.exe') } | ForEach-Object { if ($_.ExecutablePath -and (Test-Path -LiteralPath $_.ExecutablePath)) { New-NetFirewallRule -DisplayName ("Five Monitor TURN - " + $_.Name) -Direction Inbound -Action Allow -Profile Any -Program $_.ExecutablePath } else { Write-Error ("Caminho indisponível: " + $_.Name) } }
```

Esse comando libera a entrada de rede para os dois programas, incluindo suas portas UDP de relay. A porta de escuta é 3478 UDP/TCP; os executáveis também usam portas UDP dinâmicas para transmitir a mídia.

- Se as duas regras já existem e a transmissão funciona, não precisa criá-las novamente.
- Se mover a pasta do resource, confira as regras: elas apontam para o caminho dos EXEs.
- Se não houver saída, verifique se o resource iniciou os dois processos TURN.
- Se o provedor tem firewall próprio, ele também precisa permitir a comunicação do TURN. Liberar apenas 3478 pode não ser suficiente para o relay de mídia.
- Não desative o firewall inteiro.

A mensagem `managed TURN ready` confirma a inicialização local; sozinha, não comprova que a rede externa consegue acessar o serviço.

## 5. Webhook e envio dos vídeos

O webhook de vídeos é configurado exclusivamente pela linha no `server.cfg`:

```cfg
set fm_clip_webhook "URL_COMPLETA_DO_WEBHOOK"
```

Abra a interface com `/telar`, assista a um jogador e aguarde acumular alguns segundos. Clique no botão vermelho para enviar o clipe ao canal do webhook.

- O clipe contém até os últimos 10 segundos já recebidos.
- A preparação leva aproximadamente a duração do vídeo; aguarde a confirmação ou a mensagem de erro.
- O clipe é configurado por padrão para 640×360, 10 FPS, sem áudio.
- A qualidade do clipe é independente da qualidade da transmissão ao vivo.
- O transporte padrão de upload é `latent`; não precisa configurar `fm_upload_base_url` para esse modo.
- `WebhookStarted`, em `config/server.lua`, é opcional e serve para avisos de início, não para enviar vídeos.

## 6. Qualidade da transmissão

O pacote de produção vem com transmissão ao vivo em 720p, meta de 60 FPS e bitrate máximo de 6 Mbps.

Em `config/shared.lua`:

```lua
Capture = { width = 1280, height = 720, fps = 60 },
```

O limite de bitrate fica em `nui/rtc.js`, em `maxBitrate = 6000000`.

60 FPS é a meta da captura, não uma garantia. O desempenho do PC de quem transmite, a rede e o TURN afetam a taxa efetiva. O WebRTC pode reduzir o bitrate; não há troca automática de preset de resolução implementada.

Se engasgar, experimente 720p/30 FPS ou 960×540/30 FPS. Reinicie o resource e abra uma nova transmissão depois de alterar as configurações.

## 7. Uso e conferência da instalação

1. Entre com dois jogadores na base.
2. No administrador com permissão `Admin`, execute `/telar`.
3. Informe o ID/passaporte do outro jogador e confirme.
4. Confira se a imagem acompanha os movimentos do jogador.
5. Aguarde alguns segundos e teste o botão vermelho.
6. Confira se o clipe chegou ao canal do Discord.

O pacote de produção não inclui os comandos antigos `fm_nui` e `fm_permissao`. Mensagens essenciais de falha e estado do TURN permanecem disponíveis.

## 8. Atualizar sem perder a configuração

1. Faça backup da instalação atual.
2. Pare o resource pelo console do FXServer:

```text
stop five_monitor
```

3. Preserve `config/server.lua` e `config/turn.json` da instalação que já funciona.
4. Substitua a pasta pela versão nova e recoloque os dois arquivos preservados. Se personalizou comandos, teclas ou qualidade, compare também `config/shared.lua` antes de substituí-lo.
5. Mantenha o webhook e a permissão de processos no `server.cfg`.
6. Inicie novamente:

```text
ensure five_monitor
```

Para alterações simples de configuração, use:

```text
restart five_monitor
```

Feche e abra novamente a interface depois do reinício. Se mudou o caminho de instalação, confira as regras de firewall dos EXEs.

## 9. Problemas comuns

| Sintoma | O que conferir |
| --- | --- |
| Sem permissão | Grupo `Admin`, framework correto, recurso `vrp` iniciado e `Permission` sem duplicatas. |
| Jogador não encontrado | Use o passaporte da base e confirme que o jogador está conectado. |
| Não consegue assistir a si mesmo | É o comportamento padrão de `AllowSelfWatch = false`. |
| TURN não inicia | Permissão `add_unsafe_child_process_permission` antes do `ensure`, pasta `turn` completa e porta livre. |
| Tempo limite ou `connection_failed` | IP público, firewall do Windows, firewall do provedor e acesso às portas de relay. |
| Interface não carrega | Extração correta, `fxmanifest.lua`, pasta `nui` completa e arquivos atualizados. |
| Vídeo engasga | Reduza resolução/FPS e confira o desempenho e upload do transmissor. |
| Clipe não chega ao Discord | URL do webhook, configuração carregada, saída HTTPS da VPS e mensagem de erro retornada. |

## 10. Conteúdo e limites da validação

O pacote mantém os módulos e executáveis necessários e não inclui testes, `node_modules` ou comandos de diagnóstico. O código próprio foi formatado; os módulos e EXEs de terceiros foram preservados.

A versão de produção passou na validação de sintaxe/imports e em 369 verificações Lua. Essa validação não substitui o teste de vídeo e webhook na sua base. Não foi medido 60 FPS sustentado no FiveM remoto.

## Créditos

Five Community  
Discord: [discord.gg/fivecommunity](https://discord.gg/fivecommunity)


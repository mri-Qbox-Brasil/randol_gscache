# randol_gscache — Manual

Recriação do colecionável "G's Cache" do GTA Online: a cada ciclo, um pacote é escondido em um local aleatório do mapa, marcado por um blip de raio aproximado, e o primeiro jogador a achá-lo leva a recompensa.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Configuração](#configuração)
4. [Fluxo do evento](#fluxo-do-evento)
5. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
6. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qb-core`, `es_extended` **ou** `ND_Core` | Sim | Bridge automático em `bridge/`. O bridge ND exige ND_Core 2.0.0+ |
| `ox_lib` | Sim | `lib.require`, `lib.callback`, `lib.points`, `lib.showTextUI`, `lib.requestAnimDict` |

**Game build**: requer build **2372 (Tuners)** ou superior (animação e banco de áudio do colecionável) e **OneSync Infinity** — o objeto é criado no servidor.

---

## Instalação

1. Copie a pasta `randol_gscache` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure randol_gscache
   ```
3. Não há SQL nem itens de inventário a cadastrar.
4. O primeiro pacote só aparece após `CycleTimer` minutos contados a partir do start do recurso — ele não nasce imediatamente.

---

## Configuração

### `config.lua` (cliente)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `TextUIMessage` | string | Sim | Texto do TextUI exibido a menos de 1.5 metro do pacote |
| `DropNotify` | string | Sim | Notificação enviada a todos quando um novo pacote surge |
| `Blip.alpha` | número | Sim | Opacidade do blip de raio (250 metros) |
| `Blip.color` | número | Sim | Cor do blip de raio |

### `sv_config.lua` (servidor)

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `GiveRewards` | função `(player, src)` | Sim | Hook de recompensa. Por padrão sorteia entre 130 e 175 em dinheiro e notifica. Troque por itens ou outros valores aqui |
| `Debug` | bool | Sim | Imprime no console do servidor as coordenadas do pacote gerado |
| `CycleTimer` | número (minutos) | Sim | Intervalo entre gerações de pacote (padrão `90`) |
| `Locations` | lista de `{coords = vec3, rot = vec3}` | Sim | Todos os 75 locais do GTA Online. Um é sorteado por ciclo |
| `Models` | lista de strings | Sim | Modelos de prop possíveis: `prop_mp_drug_package`, `prop_mp_drug_pack_blue`, `prop_mp_drug_pack_red` |

---

## Fluxo do evento

1. A cada `CycleTimer` minutos, o servidor sorteia local e modelo, cria o objeto com `CreateObjectNoOffset` (congelado, com a rotação salva no config) e liga a GlobalState `GsCacheActive`.
2. Todos os jogadores recebem a notificação e um **blip de raio de 250 metros** — o centro é deslocado aleatoriamente em até 100 unidades nos eixos X e Y, então o blip é apenas uma pista da região, não a posição exata.
3. Ao entrar no raio de 30 metros do pacote, o blip pontual é removido e um bipe começa a tocar da coordenada do pacote a cada 3 segundos.
4. A menos de 1.5 metro aparece o TextUI; a tecla **E** coleta.
5. O servidor valida (pacote ativo e jogador a menos de 3 metros), deleta o objeto, desliga a GlobalState, limpa blips e pontos de todos os clientes e chama `GiveRewards`. Só um jogador consegue coletar.
6. Se um novo ciclo começar com um pacote ainda ativo, o pacote antigo é removido antes.

---

## Entrypoints para outros recursos

O recurso publica o estado do evento numa GlobalState, legível de qualquer recurso (cliente ou servidor):

```lua
if GlobalState.GsCacheActive then
    -- existe um pacote ativo no mapa
end
```

---

## Estrutura de arquivos

```
randol_gscache/
├── bridge/
│   ├── client/
│   │   ├── esx.lua        — notificação e estado de login no ESX
│   │   ├── nd.lua         — notificação e estado de login no ND_Core
│   │   └── qb.lua         — notificação, PlayerData e limpeza no unload (QB)
│   └── server/
│       ├── esx.lua        — GetPlayer e AddMoney no ESX
│       ├── nd.lua         — GetPlayer e AddMoney no ND_Core
│       └── qb.lua         — GetPlayer e AddMoney no QB
├── cl_cache.lua           — blips, ponto de coleta, TextUI, bipe, animação de pegar
├── sv_cache.lua           — ciclo de geração, spawn do objeto, validação e recompensa
├── config.lua             — config do cliente (textos e blip)
├── sv_config.lua          — config do servidor (ciclo, locais, modelos, recompensa)
└── fxmanifest.lua
```

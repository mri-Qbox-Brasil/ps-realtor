# ps-realtor — Manual

Tablet de corretor de imóveis para o `ps-housing`: cadastra propriedades e MLOs, define portas, garagens e jardins no mundo, gerencia vendas e aloca apartamentos.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Configuração](#configuração)
4. [Permissões por grade](#permissões-por-grade)
5. [Comandos](#comandos)
6. [Criação de propriedades](#criação-de-propriedades)
7. [Blips](#blips)
8. [Integrações](#integrações)
9. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
10. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qb-core` | Sim | O recurso usa `exports['qb-core']:GetCoreObject()` diretamente |
| `ps-housing` | Sim | Fonte de todas as propriedades, apartamentos e shells. Sem ele o recurso não carrega dados |
| `ox_lib` | Sim | Callbacks, notificações, TextUI, raycast, `glm` |
| `fivem-freecam` | Sim | Câmera livre usada no editor de polyzone (jardim / zona de MLO) |
| `ox_doorlock` | Não | Ao marcar a porta de um MLO, checa se ela já está cadastrada. Exige v1.16.0 ou superior |

---

## Instalação

1. Copie a pasta `ps-realtor` para `resources/`.
2. Adicione ao `server.cfg`, **depois** do `ps-housing`:
   ```
   ensure ps-realtor
   ```
3. Crie o job de corretor no `qb-core` (por padrão `realestate`) e adicione o nome dele em `Config.RealtorJobNames`.
4. Se for usar o tablet como item (`Config.UseItem = true`), cadastre o item `tablet` no `qb-core/shared/items.lua` como usável.

Não há SQL próprio — a persistência é toda do `ps-housing`.

---

## Configuração

Arquivo: `shared/config.lua`.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `Config.RealtorJobNames` | string[] | Sim | Jobs autorizados a vender e gerenciar propriedades. Padrão: `{ 'realestate' }` |
| `Config.UseCommand` | bool | Sim | Registra o comando `/housing`. Padrão: `true` |
| `Config.UseItem` | bool | Sim | Registra `Config.ItemName` como item usável que abre o tablet. Padrão: `false` |
| `Config.ItemName` | string | Sim | Nome do item usável. Padrão: `tablet` |
| `Config.PlayAnimation` | bool | Sim | Toca a animação de tablet ao abrir a UI. Padrão: `true` |
| `Config.AnimationProp` | string | Sim | Prop usado na animação. Padrão: `prop_cs_tablet` |
| `Config.RealtorPerms` | table | Sim | Grade mínima do job para cada ação. Ver abaixo |

As comissões de venda **não** ficam aqui — são configuradas no `ps-housing`.

---

## Permissões por grade

`Config.RealtorPerms` define o grade mínimo do job de corretor para cada operação. A UI esconde as ações que o jogador não pode executar.

| Chave | Padrão | Ação |
|---|---|---|
| `changePropertyForSale` | 0 | Colocar/tirar uma propriedade da lista de venda |
| `sellProperty` | 0 | Vender uma propriedade |
| `manageProperty` | 1 | Editar dados da propriedade (descrição, preço, interior, porta, garagem) |
| `listNewProperty` | 2 | Cadastrar uma propriedade nova |
| `deleteProperty` | 2 | Apagar uma propriedade |
| `setApartments` | 2 | Alocar apartamentos para jogadores |

O servidor valida apenas se o job do jogador está em `RealtorJobNames`. A checagem de grade é feita no lado da UI.

---

## Comandos

| Comando | Permissão | Descrição |
|---|---|---|
| `/housing` | Qualquer jogador (só registrado se `Config.UseCommand = true`) | Abre/fecha o tablet. Bloqueado se o personagem estiver morto, em last stand, algemado, ou com o menu de pause aberto |

O que o corretor consegue fazer dentro do tablet depende do job e da grade dele.

### Teclas do editor de zonas

| Tecla | Contexto | Ação |
|---|---|---|
| `E` | Posicionar porta/garagem | Confirma a posição atual |
| `H` | Posicionar porta/garagem, ou editor de polígono | Cancela / finaliza |
| `C` | Editor de polígono | Adiciona ou remove um ponto |
| `K` | Editor de polígono | Edita um ponto existente |
| `N` | Editor de polígono | Cancela a edição do ponto |
| Scroll | Editor de polígono | Muda a altura da zona |

---

## Criação de propriedades

O fluxo é conduzido pela UI e varia conforme o tipo:

**Shell (propriedade normal)**
- **Porta** — marcada com a posição do personagem. Zona de 1.5 x 2.2.
- **Garagem** — marcada com a posição do personagem. Zona de 3.0 x 5.0. Recomendado marcar dentro de um veículo para ver a área real.
- **Jardim** — polígono desenhado em câmera livre com o cursor de raycast.
- **Interior (shell)** — escolhido na lista de shells do `ps-housing`. A UI valida se o modelo existe (`IsModelInCdimage`) antes de aplicar.

**MLO**
- **Porta** — apontada diretamente para a porta do mundo (raycast em objeto). Suporta porta dupla. Se o `ox_doorlock` estiver rodando, avisa quando a porta já está cadastrada lá.
- **Zona** — polígono desenhado em câmera livre.
- **Garagem** — igual à do shell.

A rua e a região são resolvidas automaticamente pelas coordenadas (`GetStreetNameAtCoord`, `GetNameOfZone`). Ao confirmar, o recurso dispara `bl-realtor:server:registerProperty`, que repassa para `exports['ps-housing']:registerProperty`.

---

## Blips

A UI tem dois toggles de blip, aplicados nas propriedades (apartamentos são ignorados):

- **À venda** — todas as propriedades com `for_sale`.
- **Vendidas** — todas as propriedades com dono e fora da lista de venda.

Os blips usam o sprite `375`, cor `2`, escala `0.7`, e são short range. O nome traz o tipo, a rua e o `property_id`.

---

## Integrações

### ps-housing

Dependência central. O `ps-realtor` é apenas a interface — todas as escritas passam pelo `ps-housing`:

- `exports['ps-housing']:GetProperties()`, `GetApartments()`, `GetApartment()`, `GetProperty()`, `GetShells()` — carregamento inicial da UI.
- `exports['ps-housing']:registerProperty(data, nil, src)` — cadastro de propriedade nova.
- Evento `ps-housing:server:updateProperty` — edição.
- Evento `ps-housing:server:addTenantToApartment` — alocação de apartamento.

O cliente ouve `ps-housing:client:initialisedProperties`, `ps-housing:client:updatedProperty`, `ps-housing:client:updateApartment` e `ps-housing:client:addProperty` para manter a UI em dia sem recarregar.

Em todos os eventos de escrita, o `ps-realtor` injeta `data.realtorSrc = src` — é assim que o `ps-housing` sabe quem foi o corretor e paga a comissão.

### fivem-freecam

O editor de polígono (jardim e zona de MLO) roda em câmera livre, com os multiplicadores de movimento reduzidos enquanto está ativo.

### ox_doorlock

Só é consultado ao marcar a porta de um MLO, para avisar se a porta já pertence a uma door cadastrada. Requer a versão `1.16.0` ou superior; abaixo disso o recurso avisa e cancela.

---

## Entrypoints para outros recursos

### Evento `bl-realtor:client:toggleUI`

Abre/fecha o tablet no cliente. É o gancho usado pelo item usável e pode ser disparado por qualquer outro recurso.

```lua
TriggerClientEvent('bl-realtor:client:toggleUI', source)
```

### Evento `bl-realtor:server:registerProperty`

Cadastra uma propriedade nova. Só executa se o job do jogador estiver em `Config.RealtorJobNames`.

```lua
TriggerServerEvent('bl-realtor:server:registerProperty', data)
```

### Evento `bl-realtor:server:updateProperty`

Atualiza uma propriedade existente. `type` é o tipo de alteração (`UpdateShell`, `UpdateDoor`, `UpdateGarage`, etc.), repassado ao `ps-housing`.

```lua
TriggerServerEvent('bl-realtor:server:updateProperty', type, property_id, data)
```

### Evento `bl-realtor:server:addTenantToApartment`

Aloca um jogador em um apartamento.

```lua
TriggerServerEvent('bl-realtor:server:addTenantToApartment', data)
```

### Callback `bl-realtor:server:getNames`

Recebe um array de `citizenid` e devolve os nomes completos (online ou offline). Retorna `Unknown` para quem não existir.

```lua
local names = lib.callback.await('bl-realtor:server:getNames', false, { citizenid1, citizenid2 })
```

---

## Estrutura de arquivos

```
ps-realtor/
├── client/
│   ├── client.lua           — abertura da UI, animação do tablet, comando /housing, blips, callbacks NUI
│   ├── createProperty.lua   — estado da propriedade em criação, posicionamento de porta/garagem/jardim
│   ├── sync.lua             — carrega propriedades, apartamentos e shells do ps-housing e mantém a UI sincronizada
│   └── zoneCreator.lua      — editor de polígono em câmera livre e seleção de porta de MLO por raycast
├── server/
│   └── server.lua           — checagem de job, repasse dos eventos ao ps-housing, item usável
├── shared/
│   └── config.lua           — jobs, comando/item, animação e permissões por grade
├── html/                    — UI compilada (index.html, index.js, index.css, images/)
├── ui/                      — código-fonte da UI em Svelte + Vite (build gera html/)
└── fxmanifest.lua
```

A UI é um projeto Svelte em `ui/`. Para alterá-la, edite os componentes em `ui/src/` e rode o build do Vite; a saída vai para `html/`.

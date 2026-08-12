# Direct Link — MetLife Brasil

Página estática de smart redirect: tenta abrir o app MetLife Brasil e, se não conseguir, envia o usuário para a loja correta.

Arquivo principal: [`index.html`](index.html).

## O que resolve

Links compartilhados (WhatsApp, e-mail, SMS, etc.) precisam:

1. Abrir o app se estiver instalado
2. Ir para a loja se não estiver
3. Funcionar em Chrome Android sem cair direto na Play Store
4. Evitar o aviso “o site quer abrir o app” do scheme customizado no Android
5. No iOS, não deixar o botão da App Store “roubar” o toque enquanto o prompt nativo “Abrir?” estiver aberto

## Deep link e lojas

| Item | Valor |
|------|--------|
| Scheme | `cssapp` |
| Path | `brSelfServiceapp` |
| Deep link | `cssapp://brSelfServiceapp` (+ query string da página, se houver) |
| Package Android | `com.metlife.brazil.business.css` |
| Play Store | [MetLife Brasil](https://play.google.com/store/apps/details?id=com.metlife.brazil.business.css&hl=pt_BR) |
| App Store | [MetLife Brasil](https://apps.apple.com/br/app/metlife-brasil/id1625752674) |

## Comportamento esperado

### Android (Chrome / navegador real)

1. Tenta abrir o app com **Intent sem fallback de loja**:
   ```
   intent://brSelfServiceapp#Intent;scheme=cssapp;package=com.metlife.brazil.business.css;end
   ```
2. Se o app abrir, a página perde o foco e o redirect para a loja é cancelado.
3. Se após ~2s a página ainda estiver visível, redireciona para a **Play Store**.
4. Em retry manual (“Tentar abrir novamente”), usa Intent **com** `S.browser_fallback_url` da loja (gesto do usuário).

### iOS (Safari / navegador real)

1. Tenta `cssapp://brSelfServiceapp` e já liga o botão **Abrir o app** a esse deep link.
2. O Safari pode mostrar o prompt nativo “Abrir esta página no 'MetLife Brasil'?”. Esse prompt **não esconde** a página.
3. O botão **Baixar na App Store** fica **invisível e intocável** (`visibility` + `pointer-events: none`) enquanto a tentativa está em andamento — reserva o espaço no layout (botões fixos no rodapé) para nada se mover sob o dedo, mas não aceita toque.
4. Se após ~3s a página ainda estiver visível, mostra a UI de falha com **Tentar abrir novamente** (deep link). O botão da App Store fica tocável **~1s depois** dessa revelação (arming delay), para um toque em voo no momento do prompt não cair na loja.
5. **~1.5s depois** da UI de falha, redireciona para a **App Store** — mas só se a página ainda estiver visível. Esse *grace* é o que impede o bug antigo: quem confirmou “Abrir” tem o app abrindo, a página fica oculta nesse intervalo e o redirect é cancelado. Sem o app instalado, o Safari só mostra “o endereço é inválido”, a página segue visível e a loja abre sozinha.
6. Toque em “Abrir o app” cancela todos os timeouts pendentes.
7. Se o app abrir e o usuário voltar ao Safari (ex.: app “pisca” e fecha), a página mostra a UI de falha em vez de ficar no spinner infinito — e **não** vai à loja (o app existe).
8. Layout dos botões é `position: fixed` no rodapé — trocar texto/esconder spinner **não** move os botões.

### WebView in-app (WhatsApp, Instagram, Facebook, etc.)

Não faz auto-open (deep links costumam falhar nesses browsers).

Mostra instrução para **Abrir no navegador**, com botões para tentar o app e ir à loja.

### Desktop

Mostra os dois botões de loja (Play Store e App Store), sem tentar deep link.

### Proteção anti-loop

Há um cooldown de ~8s (`sessionStorage`) após uma tentativa automática. Se o usuário voltar rápido demais, a página mostra a UI de falha com CTAs em vez de disparar outro auto-redirect.

### Override de deep link (`?dl=`)

Para o time mobile testar variantes sem redeploy:

```
https://matheusbecatini.github.io/directl_ink/?dl=cssapp%3A%2F%2Fhome
https://matheusbecatini.github.io/directl_ink/?dl=cssapp%3A%2F%2FbrSelfServiceapp%2F
```

O parâmetro `dl` é removido da query repassada ao app; demais query params continuam sendo anexados ao deep link padrão.

## Por que Android não usa `cssapp://` no auto-open

- `cssapp://…` no Chrome costuma mostrar o diálogo “Este site deseja abrir o app…”.
- `intent://…` com `package` abre o app de forma mais direta.

## Por que o Intent automático não leva `S.browser_fallback_url`

No Chrome, Intent **sem gesto do usuário** + `S.browser_fallback_url` apontando para a Play Store frequentemente **pula o app** e vai direto à loja.

Por isso:

- **Auto-open** → Intent **sem** fallback de loja
- **Falha (timeout)** → redirect explícito para a loja no JavaScript
- **Toque do usuário** → Intent **com** fallback de loja (gesto válido)

## Por que o redirect iOS para a loja passa por um *grace*

Os dois diálogos nativos do Safari **mantêm a página visível** (`document.hidden` continua `false`):

- com o app instalado → “Abrir esta página no 'MetLife Brasil'?”
- sem o app → “O Safari não pode abrir a página porque o endereço é inválido.”

Do ponto de vista do JavaScript os dois estados são idênticos, então um `location.replace(AppStore)` disparado direto no timeout ganhava a corrida de quem tocou em **Abrir** e mandava para a loja quem *tem* o app.

A solução é ir à loja em duas etapas: timeout de ~3s renderiza a UI de falha e agenda o redirect para ~1.5s depois. Nesse intervalo, quem confirmou “Abrir” já teve o app aberto e a página ficou oculta — `goToStore` verifica `document.hidden` / `left` / `ac.signal.aborted` e desiste. Quem não tem o app continua visível e vai para a loja sozinho.

Se, mesmo assim, alguém **com o app instalado** confirmar o prompt e ainda cair na App Store, a causa está no **app nativo** (ver checklist iOS abaixo).

## Como testar

1. Abrir a página no **Chrome Android** com o app instalado → deve abrir o app.
2. Sem o app (ou se falhar) → após ~2s deve ir à Play Store.
3. Abrir pelo **WhatsApp** → deve pedir para abrir no navegador.
4. Abrir no **desktop** → deve mostrar as duas lojas.
5. iOS Safari **com** o app → prompt “Abrir?”; confirmar deve abrir o app e **não** cair na loja.
6. iOS Safari **sem** o app → alerta “endereço é inválido”; após o OK, deve ir sozinho para a App Store em ~4.5s.
7. Override: `?dl=cssapp%3A%2F%2Fhome` → deve tentar esse deep link em vez do padrão.

## Observação para o time mobile

### Android

O app nativo precisa registrar no AndroidManifest:

- scheme `cssapp`
- host/path compatível com `brSelfServiceapp`
- package `com.metlife.brazil.business.css`

Se o intent-filter mudar, atualize a constante `APP` em `index.html`.

### iOS — checklist (causa restante do “Abrir → App Store”)

Evidência: com o app instalado, confirmar “Abrir” no prompt do Safari **lança o app** (ele “pisca”) e em seguida o usuário acaba na App Store. Nesta página **não há** redirect automático para a loja no iOS. Validar no app nativo:

1. **`CFBundleURLTypes` / `CFBundleURLSchemes`** contém `cssapp`.
2. **`application(_:open:options:)`** ou **`scene(_:openURLContexts:)`** recebe `cssapp://brSelfServiceapp` e **roteia** a URL em vez de cair em um fallback genérico.
3. **Não** há checagem de versão mínima / forced update que abra a App Store no cold launch por deep link.
4. Confirmar a forma exata da URL aceita pelo app:
   - `cssapp://brSelfServiceapp` → host `brSelfServiceapp`, path vazio
   - Variantes úteis para teste via `?dl=`: `cssapp://brSelfServiceapp/`, `cssapp://home` (já usado no histórico deste repo)
5. Solução definitiva no iOS: **Universal Link** (arquivo AASA no domínio MetLife + entitlement `applinks:`), que abre o app **sem** o prompt “Abrir?” do Safari.

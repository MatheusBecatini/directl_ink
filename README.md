# Direct Link — MetLife Brasil

Página estática de smart redirect: tenta abrir o app MetLife Brasil e, se não conseguir, envia o usuário para a loja correta.

Arquivo principal: [`index.html`](index.html).

## O que resolve

Links compartilhados (WhatsApp, e-mail, SMS, etc.) precisam:

1. Abrir o app se estiver instalado
2. Ir para a loja se não estiver
3. Funcionar em Chrome Android sem cair direto na Play Store
4. Evitar o aviso “o site quer abrir o app” do scheme customizado no Android

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

1. Tenta `cssapp://brSelfServiceapp`.
2. Se após ~2s a página ainda estiver visível, redireciona para a **App Store**.

### WebView in-app (WhatsApp, Instagram, Facebook, etc.)

Não faz auto-open (deep links costumam falhar nesses browsers).

Mostra instrução para **Abrir no navegador**, com botões para tentar o app e ir à loja.

### Desktop

Mostra os dois botões de loja (Play Store e App Store), sem tentar deep link.

### Proteção anti-loop

Há um cooldown de ~8s (`sessionStorage`) após uma tentativa automática. Se o usuário voltar rápido demais, a página mostra a UI de falha com CTAs em vez de disparar outro auto-redirect.

## Por que Android não usa `cssapp://` no auto-open

- `cssapp://…` no Chrome costuma mostrar o diálogo “Este site deseja abrir o app…”.
- `intent://…` com `package` abre o app de forma mais direta.

## Por que o Intent automático não leva `S.browser_fallback_url`

No Chrome, Intent **sem gesto do usuário** + `S.browser_fallback_url` apontando para a Play Store frequentemente **pula o app** e vai direto à loja.

Por isso:

- **Auto-open** → Intent **sem** fallback de loja
- **Falha (timeout)** → redirect explícito para a loja no JavaScript
- **Toque do usuário** → Intent **com** fallback de loja (gesto válido)

## Como testar

1. Abrir a página no **Chrome Android** com o app instalado → deve abrir o app.
2. Sem o app (ou se falhar) → após ~2s deve ir à Play Store.
3. Abrir pelo **WhatsApp** → deve pedir para abrir no navegador.
4. Abrir no **desktop** → deve mostrar as duas lojas.
5. iOS Safari com app → deve tentar o scheme; sem app → App Store.

## Observação para o time mobile

O app nativo precisa registrar no AndroidManifest:

- scheme `cssapp`
- host/path compatível com `brSelfServiceapp`
- package `com.metlife.brazil.business.css`

Se o intent-filter mudar, atualize a constante `APP` em `index.html`.

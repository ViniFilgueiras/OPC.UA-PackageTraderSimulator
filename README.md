# OPC·UA SCADA — Simulador de Servidor, Clientes e Lógica Ladder

Simulador interativo de uma arquitetura **OPC-UA** rodando inteiramente no navegador, em HTML/CSS/JavaScript puro, sem dependências, frameworks ou build. Desenvolvido para a cadeira de **Redes Industriais** (Prof. George Thé).

O projeto demonstra, de forma visual e em tempo real, como o padrão OPC integra um servidor com múltiplos clientes, como a lógica de um CLP age sobre o sistema, e como dados de sensores físicos reais entram no *address space* e se propagam por toda a rede.

---

## O que ele faz

Um único arquivo (`index.html`) coloca em funcionamento, ao mesmo tempo:

- **1 Servidor OPC-UA simulado** com um *address space* de 7 nós tipados. Cada nó carrega `NodeId`, valor, unidade, **qualidade** (`Good` / `Uncertain` / `Bad`) e **timestamp** — o mesmo modelo de `DataValue` do OPC-UA real. Quando um valor muda, o servidor notifica as assinaturas.
- **4 clientes OPC** independentes, com papéis distintos, todos lendo o mesmo servidor: painel do operador (HMI), historiador, gestor de alarmes e supervisório móvel. Cada um assina um subconjunto de nós e reage de forma diferente — é aí que a integração do OPC fica visível.
- **2 sensores físicos reais** que puxam temperatura e vento da API pública **Open-Meteo** (sem chave). Eles escrevem nos nós do servidor, então o dado real entra no *address space* e se propaga para clientes e para a lógica de controle.
- **Diagrama Ladder animado** (lógica de CLP): os contatos *leem* nós do servidor e as bobinas *escrevem* nós de volta. Vê-se a corrente percorrer os trilhos, contatos fecharem e bobinas energizarem em tempo real.

O fluxo completo fica visível na tela:

```
sensor real → nó OPC → contato (ladder) → bobina → nó OPC → clientes
```

---

## Como rodar

> **Importante:** este projeto faz chamadas a uma API externa (Open-Meteo). Navegadores **bloqueiam** essas chamadas quando a página é aberta direto do disco (`file://`). Por isso, abrir o arquivo com duplo clique faz os sensores caírem em **modo simulado**. Para obter os dados reais, sirva a página por `http://` ou `https://`.

### Opção 1 — GitHub Pages (recomendado)

Funciona em `https://`, então os sensores reais funcionam sem configuração extra.

1. Suba o arquivo no repositório (renomeie para `index.html` para uma URL limpa).
2. Vá em **Settings → Pages**.
3. Em *Source*, escolha **Deploy from a branch**, selecione a branch `main` e a pasta `/ (root)`, e salve.
4. Aguarde 1–2 minutos. A URL será algo como `https://seu-usuario.github.io/seu-repo/`.

### Opção 2 — Servidor local

Na pasta onde está o arquivo:

```bash
python3 -m http.server 8000
```

Depois abra `http://localhost:8000/`.

Alternativas equivalentes: `npx serve` (Node) ou a extensão **Live Server** do VS Code (botão direito no arquivo → *Open with Live Server*).

### Opção 3 — Duplo clique (modo demonstração)

Abrir o arquivo direto no navegador funciona, mas os sensores rodam em **modo simulado** (valores plausíveis gerados localmente, marcados com qualidade `Uncertain`). Útil para demonstrar a lógica sem depender de rede.

---

## Modo real vs. modo simulado

| | Servido por http/https | Aberto via `file://` |
|---|---|---|
| Sensores Open-Meteo | Dados reais (`Good`) | Simulados (`Uncertain`) |
| Servidor, clientes, ladder | Funcionam | Funcionam |
| Qualidade dos nós | Reflete a rede | Marcada como incerta |

O fallback simulado garante que a apresentação nunca fique parada, mesmo se a rede do local bloquear o domínio do Open-Meteo (algo comum em redes institucionais).

---

## Arquitetura

```
┌─────────────────────────────────────────────────────────────┐
│                    SERVIDOR OPC-UA (JS)                       │
│   address space: 7 nós tipados                               │
│   { NodeId, valor, unidade, qualidade, timestamp }          │
│                                                              │
│   writeNode()  →  notifica todas as subscriptions           │
└───────┬───────────────────────┬──────────────────┬──────────┘
        │                       │                  │
   subscriptions           lê / escreve         escreve
        │                       │                  │
┌───────▼────────┐      ┌───────▼───────┐   ┌──────▼─────────┐
│  4 CLIENTES    │      │    LADDER     │   │  2 SENSORES    │
│  HMI-01        │      │  contatos lêem│   │  Open-Meteo    │
│  HIST-02       │      │  bobinas      │   │  (temp / vento)│
│  ALARM-03      │      │  escrevem     │   │                │
│  MOBILE-04     │      └───────────────┘   └────────────────┘
└────────────────┘
```

### Os nós (address space)

| NodeId | Descrição | Origem |
|---|---|---|
| `ns=2;s=Temp_Forno` | Temperatura | Sensor real (Fortaleza) |
| `ns=2;s=Vento_Torre` | Velocidade do vento | Sensor real (São Paulo) |
| `ns=2;s=Nivel_Tanque` | Nível do tanque | Processo simulado |
| `ns=2;s=Pressao` | Pressão | Processo simulado |
| `ns=2;s=Bomba_Cmd` | Comando da bomba | Lógica ladder |
| `ns=2;s=Aquecedor` | Comando do aquecedor | Lógica ladder |
| `ns=2;s=Alarme_Alto` | Alarme de alta | Lógica ladder |

### A lógica ladder

Três degraus (*rungs*):

- **Aquecedor** — liga quando a temperatura está baixa **e** não há alarme ativo (contato normalmente fechado como intertravamento de segurança).
- **Bomba** — liga quando o nível está baixo **e** a pressão está abaixo do limite. Usa **histerese** (liga abaixo de 40%, desliga acima de 60%) para evitar o chaveamento contínuo (*chattering*).
- **Alarme** — dispara quando a temperatura ultrapassa 36° e só se limpa abaixo de 33° (também com histerese).

---

## Controles da interface

- **Pausar / retomar varredura** — para o ciclo de scan do CLP.
- **Passo único** — executa um único scan; útil para explicar a lógica degrau a degrau.
- **Buscar sensores agora** — força uma nova leitura do Open-Meteo.
- **Injetar falha de qualidade** — marca um nó como `Bad`, demonstrando como a qualidade se propaga pela rede.
- **Velocidade** — ajusta o intervalo do ciclo de scan.

---

## Conceitos de Redes Industriais demonstrados

- **Address space** e nós tipados do OPC-UA.
- **Subscriptions** (modelo publish/subscribe) versus leitura por *polling*.
- **Qualidade e timestamp** em cada `DataValue` — o que distingue o OPC de um protocolo cru.
- **Integração de múltiplos clientes** sobre uma fonte única de verdade.
- **Lógica ladder** de CLP, com contatos, bobinas e intertravamentos.
- **Histerese** como solução para o *chattering* de atuadores, um problema clássico de controle.

---

## Ferramentas

- **HTML, CSS e JavaScript puro** — sem frameworks, sem build, sem instalação. Escolhido pela portabilidade: abre em qualquer navegador e embarca em qualquer site.
- **Open-Meteo** — API pública e gratuita de dados meteorológicos, sem necessidade de chave. Dados sob licença CC BY 4.0.

---

## Licença e créditos

Projeto acadêmico desenvolvido para a cadeira de Redes Industriais (Prof. George Thé).

Dados meteorológicos fornecidos por [Open-Meteo](https://open-meteo.com/) sob licença CC BY 4.0.

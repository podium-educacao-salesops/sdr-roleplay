# Simulador SDR (roleplay de ligação por áudio)

> Documento de passagem — atualizado em 25/11/2026.

## Para que serve

Treino de ligação por voz. A pessoa do comercial (executivo de contas / SDR) fala no microfone com um "dono de barbearia" simulado por IA, em três níveis (fácil, médio, difícil), e no fim recebe uma avaliação automática pela régua do script. Desde 25/11/2026 existe também o Simulador de Eventos, para quem vende Summit, Podium pelo Brasil e Cash Experience. A gestão acompanha as notas de todo mundo num painel. O uso é sob demanda (treino de quem está entrando e reforço); de 04/09 a 25/11/2026 foram gravadas 117 simulações.

## Onde está

| O quê | Endereço | Hospedagem | Conta | Banco |
|---|---|---|---|---|
| Simulador comercial | https://podium-educacao-salesops.github.io/sdr-roleplay/ | GitHub Pages (branch `main`, raiz) | GitHub `podium-educacao-salesops` | — |
| Simulador de Eventos | https://podium-educacao-salesops.github.io/sdr-roleplay/eventos.html | idem | idem | — |
| Painel da gestão (comercial) | https://podium-educacao-salesops.github.io/sdr-roleplay/historico.html | idem | idem | — |
| Painel da gestão (eventos) | https://podium-educacao-salesops.github.io/sdr-roleplay/historico-eventos.html | idem | idem | — |
| Backend (IA + gravação) | https://withered-wildflower-db78.daiane-silvasvc.workers.dev | Cloudflare Workers | Cloudflare `daiane-silvasvc` (pessoal) | D1 `sdr_roleplay_db` |

Repositório deste site: https://github.com/podium-educacao-salesops/sdr-roleplay
Repositório do backend: https://github.com/podium-educacao-salesops/sdr-roleplay-worker (tem README próprio).

## Como funciona

Site estático, sem build: quatro arquivos HTML com CSS e JS dentro.

```
navegador (Chrome)                      Worker withered-wildflower-db78          Google Gemini
 fala -> Web Speech API -> texto  --->  POST /   (persona / avaliação)  ---->    gera a fala / a nota
 texto <- speechSynthesis (voz)   <---  {content:[{text}]}
 fim da ligação                   --->  POST /save  ----> D1 sdr_roleplay_db (tabela simulations)
 historico*.html                  --->  GET /history?limit=500
```

- `index.html` — simulador comercial. Monta a persona, conversa por voz, e no fim pede a avaliação (`task: "avaliacao"`, régua v4.1 NEPQ + BANT). Grava com `rubrica_versao: "v4.1"`.
- `eventos.html` — simulador de eventos: 3 eventos × 3 níveis, régua de 10 etapas do script oficial da equipe de eventos. Grava com `rubrica_versao: "eventos-v1/<summit|ppb|cash>"`.
- `historico.html` — painel da gestão do comercial. Lê `GET /history` e descarta as linhas `eventos-v1/...`. Filtros por setor, SDR, nível, persona e período.
- `historico-eventos.html` — painel da equipe de eventos; só mostra as linhas `eventos-v1/...`.
- A chave da IA não está no site: fica como secret no Worker. O endereço do Worker está fixo em `API_URL` nos quatro arquivos.
- Se o `/save` falhar, o resultado fica numa fila no navegador (`localStorage`, chave `roleplay_pendentes`, até 10) e é reenviado na próxima simulação.

## Rodar no computador

```bash
cd sdr-roleplay
python3 -m http.server 4180 --bind 127.0.0.1
# abrir http://127.0.0.1:4180/  (e /eventos.html, /historico.html)
```

Mesmo rodando local, a página chama o Worker publicado: as simulações feitas em teste entram no banco de produção. Usar um nome de teste que dê para reconhecer depois.

Precisa de Chrome ou Edge no computador (ou Chrome no Android). Em iPhone/iPad nenhum navegador tem reconhecimento de fala — a página avisa.

## Publicar uma alteração

Push na `main` publica. O GitHub Pages reconstrói em cerca de 1 minuto; não há build nem CLI. Criar branch não publica nada.

```bash
git add index.html && git commit -m "..." && git push origin main
```

Se o deploy do Pages falhar (aba Actions do repositório), um commit vazio força outro: `git commit --allow-empty -m "Redeploy" && git push`.

Mudança no backend (IA, banco) é no repositório `sdr-roleplay-worker`, com `wrangler deploy` — ver o README de lá.

## Variáveis de ambiente

Nenhuma. O site é estático e não lê variáveis. A única configuração é a constante `API_URL` (endereço do Worker) em `index.html`, `eventos.html`, `historico.html` e `historico-eventos.html`. As chaves ficam no Worker (ver README do `sdr-roleplay-worker`).

## Rotinas automáticas

Nenhuma. Nada roda sozinho; tudo acontece quando alguém usa a página.

## Regras e cuidados

- **Painel sem senha, por decisão da Daiane (03/09/2026).** Antes de mudar isso, ver o documento de passagem do time.
- **Nome é texto livre.** O painel agrupa o SDR só ignorando maiúscula/minúscula, nunca acento: `Vinicius` e `Vinícius` são duas pessoas diferentes. Não unificar grafias parecidas sem confirmar com a gestão.
- **A nota varia.** A mesma transcrição avaliada 5 vezes variou até 10 pontos, mesmo com `temperature: 0` (medido em 03/09/2026). O julgamento costuma cair na mesma faixa da régua; a diferença de poucos pontos entre duas pessoas não é evidência de nada.
- **O app não usa o campo `score` que o modelo devolve**: ele divergia da média dos próprios critérios em 10/10 rodadas. A nota é calculada a partir dos critérios.
- **Duas escalas diferentes por simulação**: média dos critérios (0–5) e % do script seguido (0–100). Não somar nem fazer média entre elas.
- **Nota não é comparável entre modelos de IA.** Na troca do `gemini-2.5-pro` para o `gemini-3.1-pro-preview` a mesma transcrição ganhou cerca de +1,2 de aderência e +20 pontos de script. O painel registra `modelo_avaliador` em cada linha e avisa quando o período mistura modelos. Trocar o modelo de avaliação cria um degrau que parece evolução do time.
- **Pergunta de visão**: no roteiro de eventos é obrigatória; no roteiro de produto (comercial) é proibida. Não copiar critério de uma régua para a outra.
- **Simulador de Eventos não fala de valores**, por decisão da Daiane.
- O painel lê no máximo 500 simulações (`limit=500` no front, teto de 500 no Worker). Em 25/11/2026 eram 117; quando passar de 500, as mais antigas somem do painel (continuam no banco).
- O onboarding (`onboarding-sdr`) aponta para `.../sdr-roleplay/` e `.../sdr-roleplay/eventos.html` com endereço absoluto.

## Pendências da passagem

- [x] Repositório transferido para `podium-educacao-salesops` (25/11/2026).
- [ ] O endereço vai mudar para `https://<org>.github.io/sdr-roleplay/`. Atualizar os links absolutos no `onboarding-sdr/index.html` e avisar o time. Não contar com redirecionamento automático do endereço antigo do Pages.
- [ ] Na mudança de endereço, a fila local de resultados não enviados (`roleplay_pendentes`) fica no endereço antigo e se perde. Impacto pequeno.
- [ ] Transferir o Worker e o banco D1 da conta Cloudflare pessoal `daiane-silvasvc` (ver pendências no README do `sdr-roleplay-worker`). Se o endereço do Worker mudar, trocar `API_URL` nos quatro HTML.
- [ ] Decidir se o painel continua sem senha depois da saída da Daiane.

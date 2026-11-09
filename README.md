# Calçada Viva — IA Generativa para Engenharia

**GS – Generative AI for Engineering | Desafio CitizenTech | Engenharia de Software — FIAP**

**Integrantes:** Gustavo Felex (RM554242), Vinicius Issa Gois (RM553814), Gustavo Bonani (RM553493), Wesley Vieira (RM553496)

---

## 1. Problema escolhido

Calçadas com buracos, desníveis, falta de rampa ou obstáculos impedem a circulação de pessoas com deficiência, idosos e quem empurra carrinhos de bebê. Muitos cidadãos querem relatar o problema, mas escrevem relatos soltos e incompletos ("tem um negócio atrapalhando aqui"), sem tipo de barreira nem localização, o que dificulta o encaminhamento.

**Calçada Viva** é um helper com IA generativa que organiza esses relatos: classifica o tipo de barreira, resume apenas os fatos informados, devolve os dados em formato estruturado e pergunta o que está faltando. O cidadão revisa o resultado antes de compartilhá-lo.

## 2. Arquitetura da solução

```
 Cidadão (relato em texto livre)
            │
            ▼
 ┌──────────────────────────────┐
 │ Notebook Python (Jupyter)    │
 │  • prompts de sistema        │
 │  • gerar_resposta()          │──── modo "api" ───► OpenRouter (Chat Completions)
 │                              │                     principal: nvidia/nemotron-3-ultra-550b-a55b:free
 │                              │                     reservas:  xing4.0-29b, flint-v1.0
 │                              │──── modo "demo" ──► resposta fixa de exemplo (sem chave)
 │  • parsear_json() valida     │
 │  • fallback em caso de falha │
 └──────────────────────────────┘
            │
            ▼
 Resultado para revisão humana
 (categoria, resumo, JSON, pergunta de esclarecimento)
```

| Componente | Descrição |
|---|---|
| `chamar_api()` | Envia a conversa ao endpoint `https://openrouter.ai/api/v1/chat/completions` usando só a biblioteca padrão (`urllib`). Usa `temperature=0.2` para respostas consistentes, envia modelos reserva no campo `models` e transforma respostas de erro da API em uma exceção clara. |
| `gerar_resposta()` | Escolhe entre o modo `api` (chamada real ao modelo) e o modo `demo` (sem chave e sem custo). |
| `parsear_json()` | Valida se a resposta é um objeto JSON; não tenta "consertar" respostas inválidas. |
| `resposta_de_fallback()` | Mensagem segura ao cidadão quando a API falha. Não inventa resultados e não apaga o relato. |

## 3. Modificações feitas no notebook

Todas as mudanças estão marcadas com `# MODIFICACAO-ALUNO` (código) ou `## Modificação do aluno` (markdown).

| # | Modificação | Motivo |
|---|---|---|
| 1 | **Troca do modelo via API** para `nvidia/nemotron-3-ultra-550b-a55b:free` no OpenRouter, com dois modelos reserva | Gratuito, grande (550B parâmetros, 55B ativos) e estável nos testes. O `google/gemma-4-31b-it:free`, primeira opção, retornou erro 429 (limite do provedor) em todas as tentativas; outros modelos gratuitos retornaram erro 502. Os modelos reserva evitam que a demonstração pare se o principal ficar sobrecarregado. |
| 2 | **Prompt de classificação e resumo** com categorias fechadas | Relatos livres variam muito; categorias fixas e a regra de "não inventar fatos" deixam a saída consistente e segura. |
| 3 | **Prompt com saída estruturada em JSON** | Campos previsíveis permitem integração futura com formulários; `local` fica `null` se não informado e `requer_revisao_humana` é sempre `true`. |
| 4 | **Prompt para informação ausente** com linguagem acessível | Em vez de inventar a localização, o assistente pergunta de forma simples e não pede dados pessoais (nome, telefone, CPF). |
| 5 | **Experimento de tratamento de erros do modelo** | Passa 4 respostas inválidas reais (JSON cortado, JSON em bloco markdown, lista em vez de objeto, texto livre) pela função `parsear_json()` e confirma que todas são detectadas; também mostra a mensagem de contingência quando a API falha. |

## 4. Exemplos de prompts e saídas (modelo real)

As saídas abaixo foram geradas pelo modelo `nvidia/nemotron-3-ultra-550b-a55b:free`, chamado de verdade pela API do OpenRouter.

### Classificação e resumo
**Prompt de sistema (trecho):** "Classifique em: buraco, desnível, bloqueio, falta de rampa, piso danificado, outro ou insuficiente. Resuma somente fatos informados. Não invente endereço, causa ou responsável."

**Relato:** "Tem caixas ocupando a calçada na Rua das Flores, perto do número 120. Uma pessoa de cadeira de rodas não consegue passar."

**Saída do modelo:**
```
**Classificação:** bloqueio

**Resumo dos fatos:**
- Caixas estão ocupando a calçada na Rua das Flores, próximo ao número 120.
- Uma pessoa em cadeira de rodas não consegue passar pelo local.

**Informações essenciais presentes:** local (rua e número aproximado), tipo de obstáculo
(caixas) e impacto na acessibilidade (impedimento de passagem de cadeira de rodas).
Nenhuma pergunta adicional necessária no momento.
```

### Saída estruturada em JSON
A resposta do modelo passou pela validação de `parsear_json()`:
```json
{
  "categoria": "bloqueio",
  "descricao_resumida": "Caixas ocupando a calçada, impedindo passagem de cadeira de rodas.",
  "local": "Rua das Flores, perto do número 120",
  "impacto_acessibilidade": "alto",
  "informacoes_pendentes": ["Número exato do imóvel", "Fotos do local", "Horário da ocorrência"],
  "requer_revisao_humana": true
}
```

### Informação ausente
**Relato:** "Tem um negócio atrapalhando a calçada aqui e está difícil passar."

**Saída do modelo:**
```
Qual é o bairro ou um ponto de referência próximo?
```

### Tratamento de erros do modelo
```
Teste 1 — validação de respostas inválidas do modelo
  [OK] JSON cortado no meio: Unterminated string starting at: line 1 column 36 (char 35)
  [OK] JSON dentro de bloco markdown: Expecting value: line 1 column 1 (char 0)
  [OK] lista em vez de objeto: A resposta JSON precisa ser um objeto.
  [OK] texto livre: Expecting value: line 1 column 1 (char 0)
  [OK] JSON válido aceito: {'categoria': 'buraco', 'local': None}
Resultado: todas as respostas inválidas foram detectadas.

Teste 2 — falha na chamada à API
  Falha simulada: TimeoutError
  Mensagem ao cidadão: Não consegui organizar o relato agora. Tente novamente mais tarde
  ou revise as informações manualmente. Nenhum relato foi enviado.
```

### Análise dos resultados
- **Classificação correta** nos três relatos e **nenhum dado inventado** sobre endereço, causa ou responsável.
- **JSON válido** na primeira tentativa, com todas as chaves pedidas.
- **Pontos a melhorar observados:** no JSON, o modelo listou como pendentes "fotos" e "horário", que o prompt não pedia; no relato vago, perguntou só a localização e não o tipo de obstáculo. Os dois casos reforçam a necessidade da revisão humana e indicam ajustes futuros no prompt (por exemplo, limitar `informacoes_pendentes` a localização e tipo de barreira).

## 5. Evidências de execução

Prints do notebook executado no JupyterLab, com chamadas reais ao modelo e todas as células rodadas sem erro:

| Print | Conteúdo |
|---|---|
| ![](evidencias/01_cabecalho_e_modelo.jpg) | Cabeçalho e Modificação 1 — troca do modelo e justificativa |
| ![](evidencias/02_funcoes_api_saida.jpg) | Funções de chamada à API e saída com o modelo configurado (`Modo: api`) |
| ![](evidencias/03_prompt_classificacao.jpg) | Modificação 2 — prompt de classificação e resposta do modelo |
| ![](evidencias/04_prompt_json.jpg) | Modificação 3 — saída estruturada em JSON gerada pelo modelo |
| ![](evidencias/05_prompt_info_ausente.jpg) | Modificação 4 — pergunta de esclarecimento gerada pelo modelo |
| ![](evidencias/06_tratamento_erro_codigo.jpg) | Experimento — tratamento de erros do modelo (código) |
| ![](evidencias/07_tratamento_erro_saida.jpg) | Experimento — tratamento de erros do modelo (saída dos testes) |

## 6. Como executar

1. Instale o ambiente: `pip install -r requirements.txt` (ou abra o notebook no Google Colab ou no JupyterLite).
2. Crie uma chave gratuita em [openrouter.ai](https://openrouter.ai) e configure as variáveis de ambiente:
   ```bash
   export OPENROUTER_API_KEY="sua-chave"   # nunca coloque a chave no notebook nem no GitHub
   export CALCADA_VIVA_MODE=api
   export OPENROUTER_MODEL=nvidia/nemotron-3-ultra-550b-a55b:free   # opcional
   ```
   No Colab, guarde a chave em **Secrets** e carregue com `userdata.get("OPENROUTER_API_KEY")`.
3. Abra `calcada_viva_generative_ai.ipynb` e execute todas as células.
4. Sem chave, o notebook roda em **modo demonstrativo**, com respostas fixas de exemplo e sem custo.

## 7. Limitações

- Modelos gratuitos têm limite de uso e podem ficar indisponíveis (vimos erros 429 e 502 nos testes); os modelos reserva reduzem, mas não eliminam, esse risco.
- As respostas podem variar a cada execução, mesmo com `temperature=0.2`.
- A classificação é uma triagem preliminar, não um laudo técnico nem um protocolo oficial.
- O modelo pode classificar ou resumir de forma incorreta, acrescentar pendências desnecessárias ou devolver JSON inválido (nesse caso o notebook avisa e pede nova tentativa ou revisão humana).
- O protótipo não envia relatos à prefeitura nem a nenhum órgão público.

## 8. Aspectos éticos

- **Revisão humana obrigatória:** toda saída é marcada para revisão antes de ser compartilhada.
- **Sem invenção de dados:** os prompts proíbem inventar endereço, causa ou responsável; dados ausentes são perguntados ou ficam como `null`.
- **Privacidade:** o assistente não pede nome, telefone ou CPF, e o usuário é orientado a não incluir dados pessoais nos relatos. Os relatos são enviados a um provedor externo (OpenRouter), o que reforça a orientação de não incluir dados pessoais.
- **Transparência:** o sistema não afirma que o relato foi recebido ou resolvido pela prefeitura e não substitui canais oficiais nem serviços de emergência.
- **Segurança da chave:** a chave da API fica em variável de ambiente e nunca é versionada.
- **Acessibilidade:** respostas em português simples, com frases curtas e tom respeitoso.

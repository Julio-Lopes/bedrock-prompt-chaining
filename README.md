# bedrock-prompt-chaining

Exemplo de **prompt chaining** com **AWS Step Functions** e **Amazon Bedrock**. A state machine envia uma sequência de prompts para um modelo de linguagem, acumulando o histórico da conversa a cada etapa, para que cada resposta leve em conta as anteriores.

## Como funciona

```
Invoke model with first prompt
        │
Add first result to conversation history
        │
Invoke model with second prompt
        │
Add second result to conversation history
        │
Invoke model with third prompt
```

1. O primeiro prompt pede uma introdução sobre o tema recebido na entrada.
2. Um estado `Pass` salva o prompt e a resposta na variável `$history`.
3. O segundo prompt é enviado junto com o histórico e pede os pontos principais do tema.
4. O histórico é atualizado novamente.
5. O terceiro prompt pede uma conclusão com base em toda a conversa, e o resultado é retornado como saída da execução.

A definição usa **JSONata** como linguagem de consulta, o que permite manipular o histórico com `$append` e variáveis via `Assign`.

## Estrutura

```
.
├── prompt-chaining-bedrock.asl.json   # Definição da state machine (ASL)
└── README.md
```

## Pré-requisitos

- Conta AWS com acesso ao Amazon Bedrock
- Acesso habilitado ao modelo usado (por padrão, `anthropic.claude-3-haiku-20240307-v1:0`)
- Uma IAM role para a state machine com permissão `bedrock:InvokeModel`

Exemplo de política mínima:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "arn:aws:bedrock:*::foundation-model/anthropic.claude-3-haiku-20240307-v1:0"
    }
  ]
}
```

## Deploy

### Pelo console

1. Abra o **AWS Step Functions** e clique em **Create state machine**.
2. Na aba **Código**, cole o conteúdo de `prompt-chaining-bedrock.asl.json`.
3. Selecione ou crie a IAM role com a permissão acima.
4. Salve a state machine.

### Pela AWS CLI

```bash
aws stepfunctions create-state-machine \
  --name bedrock-prompt-chaining \
  --definition file://prompt-chaining-bedrock.asl.json \
  --role-arn arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
```

## Execução

Entrada esperada:

```json
{ "topic": "computação em nuvem" }
```

Pela CLI:

```bash
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:<REGION>:<ACCOUNT_ID>:stateMachine:bedrock-prompt-chaining \
  --input '{"topic": "computação em nuvem"}'
```

Saída:

```json
{ "resultado_final": "..." }
```

## Personalização

- **Modelo:** altere o campo `ModelId` nos três estados `Task`. Modelos mais recentes da Anthropic podem exigir um *inference profile* (ex.: `us.anthropic.claude-...`).
- **Prompts:** os textos dos prompts estão fixos na definição. Edite-os nos estados de invocação e nos estados `Pass` correspondentes, que repetem o texto ao montar o histórico.
- **Tamanho das respostas:** ajuste `max_tokens` em cada chamada.
- **Mais etapas:** basta repetir o par `Invoke model` → `Add result to conversation history`.

## Custos

Cada execução faz três chamadas ao Bedrock, e o histórico cresce a cada etapa, aumentando o número de tokens de entrada. Consulte a página de preços do Amazon Bedrock para o modelo escolhido.

# Testes de Integração com GenAI: Pedidos, Estoque e Pagamentos

Atividade do fórum da Unidade III. Uso de prompts com uma LLM para apoiar testes de integração em um cenário REST.

## Cenário

Fluxo: `POST /orders` → `GET /stock/{id}` (Estoque) → `POST /payments` (Pagamentos) → pedido confirmado.

Riscos de integração: falha parcial, timeout, resposta malformada, idempotência e contratos entre serviços.

## Tipos de prompt usados

1. **Prompt A:** geração de cenários de integração (papel + contexto + formato de saída).
2. **Prompt B:** casos de erro e contrato de API (contrato OpenAPI + exemplo few-shot + restrições).

## Prompt A

**v1**
```
Gere casos de teste de integração para um serviço de Pedidos que usa Estoque e Pagamentos.
```
Problema: saída genérica, poucos casos, sem falhas parciais e sem formato definido.

**v2**
```
Você é um engenheiro de QA sênior. O serviço de Pedidos chama Estoque (GET /stock/{id})
e Pagamentos (POST /payments). Gere 10 casos de teste de integração em tabela com:
ID, objetivo, pré-condições, passos, resultado esperado e tipo (sucesso, falha, borda).
Inclua falha parcial (pagamento aprovado mas estoque indisponível), timeout,
resposta malformada e idempotência.
```

### Casos gerados (v2)

| ID | Objetivo | Resultado esperado | Tipo |
|----|----------|--------------------|------|
| TC01 | Pedido válido (estoque ok, pagamento aprovado) | 201, status CONFIRMED | Sucesso |
| TC02 | Estoque insuficiente | 409, Pagamentos não é chamado | Falha |
| TC03 | Produto inexistente | 404 | Falha |
| TC04 | Pagamento recusado | 402, estoque não é reservado | Falha |
| TC05 | Timeout no Pagamentos | 504, pedido PENDING, retry permitido | Borda |
| TC06 | Timeout no Estoque | 503, nenhuma cobrança feita | Borda |
| TC07 | Pagamentos retorna JSON sem campo `status` | 502, erro tratado | Borda |
| TC08 | Falha parcial: pagamento aprovado, reserva de estoque falha | Estorno e pedido CANCELLED | Falha |
| TC09 | Mesmo POST repetido com mesma Idempotency-Key | Apenas 1 cobrança | Borda |
| TC10 | Payload inválido (quantidade = -1) | 400, nenhum serviço externo chamado | Falha |

## Prompt B

**v1**
```
Gere testes em Python para a API de Pedidos.
```
Problema: testes sem padrão, sem validação de status code e dependendo de serviços reais.

**v2**
```
Com base neste contrato (Estoque retorna {"disponivel": int}; Pagamentos retorna
{"status": "APPROVED" | "DECLINED"}), gere testes em pytest + requests que validem
status code, campos obrigatórios e erros 402, 404 e 409. Use a biblioteca `responses`
para simular os serviços externos. Siga este formato de teste: [1 teste modelo].
```

### Código gerado (v2)

```python
import requests
import responses

ESTOQUE = "http://estoque/stock"
PAGAMENTOS = "http://pagamentos/payments"


def criar_pedido(produto_id, qtd):
    r = requests.get(f"{ESTOQUE}/{produto_id}", timeout=2)
    if r.status_code == 404:
        return 404
    if r.json()["disponivel"] < qtd:
        return 409
    p = requests.post(PAGAMENTOS, json={"produto": produto_id, "qtd": qtd}, timeout=2)
    if p.json().get("status") != "APPROVED":
        return 402
    return 201


@responses.activate
def test_pedido_valido():
    responses.add(responses.GET, f"{ESTOQUE}/1", json={"disponivel": 5})
    responses.add(responses.POST, PAGAMENTOS, json={"status": "APPROVED"})
    assert criar_pedido(1, 2) == 201


@responses.activate
def test_estoque_insuficiente():
    responses.add(responses.GET, f"{ESTOQUE}/1", json={"disponivel": 1})
    assert criar_pedido(1, 2) == 409


@responses.activate
def test_produto_inexistente():
    responses.add(responses.GET, f"{ESTOQUE}/99", status=404)
    assert criar_pedido(99, 1) == 404


@responses.activate
def test_pagamento_recusado():
    responses.add(responses.GET, f"{ESTOQUE}/1", json={"disponivel": 5})
    responses.add(responses.POST, PAGAMENTOS, json={"status": "DECLINED"})
    assert criar_pedido(1, 2) == 402
```

Para rodar: `pip install pytest requests responses` e depois `pytest`.

## Análise

**Influência de cada prompt**
- O Prompt A gerou boa diversidade de cenários (sucesso, falha parcial, timeout, idempotência), mas em nível alto, sem código.
- O Prompt B gerou testes executáveis com validação de status e campos, porém com menos cenários de borda.

**Ajustes necessários**
1. Definir o formato de saída (tabela ou pytest) eliminou respostas vagas.
2. Pedir explicitamente falha parcial, timeout e idempotência ampliou a cobertura.
3. Dar um exemplo (few-shot) padronizou o estilo dos testes.
4. Restringir o uso de mocks evitou testes que dependiam de ambiente real.

**Conclusão:** a GenAI funciona bem como copiloto, mas o resultado esperado de cada caso precisa de revisão humana. Os casos redundantes foram removidos e as regras de negócio, conferidas manualmente.# integracao-pedidos-genai
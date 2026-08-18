# ADR-003 — Autenticação HMAC-SHA256 com secret por endpoint

- **Status:** Aceita
- **Data:** 2026-08-18 (reunião técnica)
- **Decisores:** Sofia (Segurança), Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)

## Contexto

Os webhooks enviam dados de pedidos para um URL fora da nossa infraestrutura. O cliente precisa (1) autenticar que a requisição veio de nós e (2) detectar adulteração do payload em trânsito. Já houve caso de cliente que vazou secret em log da aplicação dele; uma secret global da plataforma amplificaria esse incidente.

A feature é **somente outbound** (nós enviamos; o cliente não envia webhooks para nós).

## Decisão

- Assinar o **corpo da requisição** com **HMAC-SHA256** e enviar a assinatura no header `X-Signature`.
- **Secret única por endpoint** de webhook (não uma secret global da plataforma). A secret é **gerada por nós** na criação do cadastro e devolvida na resposta de criação.
- **Rotação pela API**, com **grace period de 24 horas**: secret antiga e nova válidas em paralelo; depois a antiga expira.
- **TLS obrigatório**: URL cadastrada deve ser `https`. URL `http` é recusada na validação (schema Zod). Não é uma decisão de stack à parte — é regra de validação do cadastro.

A tabela de configuração armazena `url`, `secret`, `customer_id` e estado ativo.

## Alternativas consideradas

- **Secret global da plataforma.** Descartada: vazamento em um cliente comprometeria todos os outros. ([09:21] Sofia)
- **Assinatura com algoritmo diferente de SHA-256.** HMAC-SHA256 foi escolhido por ser padrão de mercado e ter biblioteca em qualquer cliente sério. ([09:20] Sofia)

HTTPS obrigatório e limite de 64 KB do payload (erro se ultrapassar) foram registrados como restrições de validação/NFR, não como ADRs separados.

## Consequências

**Positivas**

- Cliente consegue verificar origem e integridade com biblioteca padrão.
- Blast radius de um vazamento fica limitado a um endpoint.
- Rotação com 24 h permite migrar o lado do cliente sem downtime da verificação.

**Negativas / trade-off**

- Operação de secret (geração, armazenamento, rotação, grace) é mais complexa que uma chave única.
- O cliente precisa implementar verificação HMAC e, na janela de rotação, aceitar duas secrets.
- Secret precisa ser tratada como dado sensível em logs (redação), analogamente a senha/token já redigidos pelo Pino do projeto.

O trade-off aceito: complexidade de secret por endpoint e rotação em troca de isolamento de vazamento e padrão de mercado verificável pelo cliente.

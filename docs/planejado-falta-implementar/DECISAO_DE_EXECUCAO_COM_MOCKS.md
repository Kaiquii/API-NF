# Decisão de execução inicial com mocks

> Status: decisão registrada em 05/08/2026.

## Contexto

O integrador do ERP, Jorge, informou que o ERP também está em desenvolvimento e autorizou o avanço da API-NF sem uma empresa emitente piloto disponível neste momento.

## Decisão

O desenvolvimento inicial da API-NF seguirá com dados mockados e fixtures fiscais controladas.

Essa decisão permite iniciar a fundação técnica do projeto e implementar o fluxo interno de emissão sem assumir dados fiscais fictícios como se fossem regras de produção.

## O que pode avançar

- estrutura de API, worker e comando administrativo;
- configuração por ambiente;
- PostgreSQL, migrations e isolamento multi-tenant;
- autenticação por API key;
- contratos de entrada, validação e mapping;
- modelo fiscal canônico e módulo `nfe/v1`;
- idempotência, estados, numeração e processamento assíncrono;
- geração de XML, validação, assinatura em ambiente controlado e testes automatizados;
- armazenamento de artefatos, consulta, webhooks, logs, métricas e auditoria;
- integração entre o ERP em desenvolvimento e os endpoints de homologação mockados.

## O que permanece pendente para homologação real

A homologação com a SEFAZ e qualquer ativação em produção dependem de uma empresa emitente piloto e das informações reais do seu cenário:

- UF, regime tributário e operação fiscal inicial;
- dados cadastrais e habilitação da empresa;
- certificado A1 válido e disponibilizado por canal seguro;
- produtos de teste com NCM, unidade e tributação revisados;
- responsável fiscal para aprovar o cenário e as evidências;
- credenciais e disponibilidade do ambiente de homologação da SEFAZ.

## Regra de segurança

Mocks, fixtures e respostas simuladas aceleram a construção e a integração do ERP, mas não comprovam autorização fiscal. Nenhuma emissão real ou ativação em produção será realizada antes de validar o cenário com dados verdadeiros, certificado A1 e homologação controlada na SEFAZ.

## Próximo passo

Iniciar o Marco 1 - Fundação de engenharia, enquanto o integrador evolui o ERP e disponibiliza os dados necessários para o Marco 0 e para a homologação fiscal real.

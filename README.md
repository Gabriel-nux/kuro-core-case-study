# Kuro Industries — Edge AI Data Processing Platform

**Estudo de Caso Técnico:** Arquitetura de processamento de dados sensíveis com IA local e licenciamento *Zero Trust*, projetada para operar inteiramente dentro da infraestrutura do cliente.

![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Polars](https://img.shields.io/badge/Polars-000000?style=for-the-badge&logo=polars&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-000000?style=for-the-badge&logo=flask&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=for-the-badge&logo=github-actions&logoColor=white)

---

## O Problema

Processar folhas de cálculo de negócio com inferência de IA quando os dados **não podem**, por exigência contratual, sair da máquina do cliente. 

Isto descarta qualquer arquitetura SaaS centralizada convencional e exige repensar de onde para onde cada responsabilidade se move: o processamento de dados e a inferência de IA devem ocorrer **100% localmente**. A nuvem existe apenas como um *gatekeeper* para governar quem tem o direito de executar o software.

---

## Visão Geral da Arquitetura

A solução foi desenhada em três serviços fisicamente isolados, cada um com uma única responsabilidade e uma única audiência:

```mermaid
graph LR
    subgraph Cliente
    A["EDGE NODE<br>Processamento + IA Local"]
    end
    
    subgraph Nuvem Kuro
    B(("CLOUD API<br>Licenciamento e Telemetria"))
    end
    
    subgraph Interno Kuro
    C["ADMIN PANEL<br>Gestão de Licenças"]
    end

    A -->|"Valida HWID e Token<br>Zero Data Egress"| B
    C -->|"Gere Bloqueios e HWID"| B

    style A fill:#1e1e1e,stroke:#00aaff,stroke-width:2px,color:#fff
    style B fill:#1e1e1e,stroke:#ffaa00,stroke-width:2px,color:#fff
    style C fill:#1e1e1e,stroke:#00ffaa,stroke-width:2px,color:#fff

Regra de Isolamento Físico: Esta segregação não é apenas concetual, é uma regra de processo. Os três repositórios nunca são aninhados um dentro do outro. Isto garante que o código-servidor e as ferramentas administrativas jamais sejam acidentalmente empacotados no instalador distribuído ao cliente final.
Motor de Processamento

O núcleo do sistema foi projetado para eficiência de memória e resiliência contra dados mal formatados:

    Avaliação Preguiçosa (Lazy Evaluation): A leitura, validação e deduplicação de folhas de cálculo via Polars constrói um plano de execução antes de ler qualquer linha, evitando carregar o ficheiro inteiro em memória.

    Inferência de IA em Blocos: A etapa mais cara (LLMs locais) consome dados em lotes de tamanho fixo. O pico de memória permanece constante independentemente de o ficheiro ter 1.000 ou 500.000 registos, mantendo o uso de RAM achatado.

    Fila de Mensagens Mortas (Dead Letter Queue): Nenhum erro de formatação interrompe o processamento do lote. Registos problemáticos são isolados numa DLQ com o motivo exato do erro para auditoria, nunca descartados silenciosamente.

    Disjuntor (Circuit Breaker): Falhas consecutivas na IA são tratadas como indisponibilidade do serviço (não erro de dado), interrompendo a tarefa de forma controlada para evitar ruído.

    Motor de Regras Plugável: A lógica de negócio é carregada dinamicamente. Colunas, formatos e vocabulário são definidos por configuração declarativa, não no código-fonte, permitindo uma alternativa (fallback) para regras genéricas.

Pipeline Condicional de IA e Visão Computacional

Abstraímos a complexidade da interação com modelos locais para o utilizador final:

    Auto-Prompting Inteligente: O operador descreve a sua intenção em linguagem natural no frontend. Um modelo menor redige e guarda um prompt profissional otimizado no perfil do cliente para inferências futuras.

    Extração Multimodal (LLaVA): Para registos com imagens anexas, o sistema aciona uma segunda etapa de inferência com um modelo de visão computacional local. A arquitetura garante que esta etapa pesada seja ignorada em registos puramente textuais.

Camada de Licenciamento (Zero Trust)

Sem ofuscação de binário e sem visibilidade dos dados processados, a proteção da propriedade intelectual depende 100% do protocolo Edge-Cloud:

    Vínculo por Hardware (Trust-on-First-Use): Cada licença vincula-se ao hardware (UUID) na primeira ativação válida. Tentativas de uso noutras máquinas são recusadas automaticamente pela API.

    Cache Offline Assinada (HMAC-SHA256): O sistema tolera instabilidade de rede operando num período de carência. A integridade do ficheiro .kuro_sync é garantida por assinatura associada ao HWID da máquina. Mudar a data ou copiar o ficheiro invalida o acesso via comparação de tempo constante (hmac.compare_digest).

    Resposta Assimétrica a Falhas (Killswitch): A thread de licenciamento distingue erros definitivos (401/403) de instabilidades (5xx/Timeout). Apenas recusas definitivas encerram o processo na hora, prevenindo que o sistema vá abaixo por instabilidade da API.

    Princípio do Menor Privilégio: O Edge Node possui apenas tokens limitados ao próprio cliente, nunca trafegando a chave mestra (X-API-KEY) administrativa nas validações.

Privacidade por Conceção (Privacy by Design)

O maior inibidor de adoção de IA no ambiente corporativo é o risco de fuga de dados. Esta arquitetura trata isso como um requisito estrutural:

    Zero-Data Egress: O processamento ocorre 100% no cliente. A Cloud API atua exclusivamente como gatekeeper de licenças e nunca recebe dados, folhas de cálculo ou conteúdo de negócio.

    Superfície de Auditoria Reduzida: Ao garantir inferência no Edge, eliminamos a necessidade de Acordos de Processamento de Dados (DPAs) com provedores de nuvem (ex: OpenAI, AWS), facilitando a conformidade com frameworks como LGPD e GDPR.

Automação de Suporte via WhatsApp

Integrado diretamente ao mesmo backend de licenciamento, sem infraestrutura adicional (via Meta Cloud API):

    Notificação Proativa: Alertas automáticos ao fim de pipelines longos, eliminando acompanhamento manual.

    Suporte Técnico Auto-Atendido: Os utilizadores enviam códigos de erro e recebem vídeos curtos de resolução.

    Gist Cache Fallback: A base de conhecimento corre num Gist público com TTL em memória. O suporte é atualizável em tempo real, sem necessidade de novas implementações, e degrada graciosamente servindo a última cache válida caso o GitHub fique indisponível.

Engenharia de Qualidade

A estabilidade da ponte Edge-Cloud é rigorosamente testada:

    81 Testes Automatizados: Cobrem os backends contra vetores adversários (adulteração de cache offline, clonagem de HWID, simulação de timeouts e payloads mal formatados).

    Integração Real: O CI/CD corre contra instâncias reais de PostgreSQL (via GitHub Actions), garantindo que o comportamento reflita o ambiente de produção.

    Regressão de Performance: Testes empíricos com dados sintéticos em escala para comprovar (e não apenas assumir) o comportamento de memória do motor de processamento.

Stack Técnica

    Edge Node: Python, Polars, CustomTkinter, LLaVA / LLMs Locais (Inferência on-device)

    Cloud API: Flask, SQLAlchemy, PostgreSQL, Meta Cloud API

    DevOps / Qualidade: pytest, GitHub Actions, Render

Maturidade e Limitações Conhecidas

Este projeto documenta tanto as suas decisões deliberadas quanto as suas limitações conhecidas, atualmente priorizadas no roadmap técnico:

    Migração de Schema: Sem integração formal com o Alembic; alterações de base de dados em produção dependem de comandos ALTER TABLE manuais e idempotentes no arranque (suporta novas colunas, mas falha em mutações complexas).

    Rate Limiting: As rotas públicas da API atualmente não possuem limitação de taxa estrita.

    Painel Admin: Ações de manutenção de infraestrutura (repor HWID, atualizar telefone) são feitas diretamente via chamadas de API, aguardando implementação de interface gráfica no SaaS Admin.
![Demonstração do Sistema](screenshots/mains.png)
![Demonstração do Sistema](screenshots/command.png)

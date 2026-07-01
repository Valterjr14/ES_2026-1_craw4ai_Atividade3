# Auditoria de Testes de Software e Plano de Evolução da Qualidade

---

## Sumário

- Sobre o Repositório
- Projeto Analisado
- Equipe
- Estrutura da Auditoria
- Metodologia
- Vídeo de Apresentação

---

## Sobre o Repositório

Este repositório registra todas as análises, evidências e conclusões produzidas pela equipe na **Atividade Avaliativa 3 (A3)** da disciplina de Engenharia de Software.

O objetivo desta etapa é realizar uma **auditoria da estratégia de testes** do projeto **Crawl4AI**, avaliando a qualidade da suíte de testes existente, sua cobertura, nível de automação, riscos associados e propondo um plano de evolução para aumentar a confiabilidade e a maturidade do processo de testes.

Esta atividade dá continuidade às Atividades 1 e 2, utilizando os diagnósticos anteriormente realizados sobre requisitos, arquitetura e dívida técnica como base para avaliar a capacidade do projeto de evoluir com segurança.

---

## Projeto Analisado

O projeto analisado foi o **Crawl4AI** — https://github.com/unclecode/crawl4ai

O Crawl4AI é uma biblioteca open source desenvolvida em Python para web crawling e web scraping com saída otimizada para modelos de linguagem. Em vez de retornar HTML bruto, o projeto transforma páginas em Markdown limpo e estruturado, removendo elementos desnecessários e facilitando seu uso em aplicações de IA, RAG (Retrieval-Augmented Generation), agentes inteligentes e pipelines de processamento de dados.

---

- **Linguagem principal:** Python
- **Licença:** Apache 2.0
- **Versão analisada:** v0.8.7 (01 de junho de 2026)

---

## Equipe

- Pedro Afonso Tavares Barreto da Silva — 202300027580
- José Valter de Oliveira Junior — 202300083678
- Yasmin Silva Santos — 202300084058
- Beatriz Eduarda Pires da Cruz — 202200092647
- João Guilherme Santos Florêncio — 202200025546

---

## Estrutura da Auditoria

A auditoria está organizada em quatro eixos de investigação, seguindo o roteiro proposto na atividade.

```
ES_2026-2_Crawl4AI_Atividade3/
├── README.md
├── Eixo_A_Estrategia_de_Testes.md
├── Eixo_B_Qualidade_Tecnica_dos_Testes.md
├── Eixo_C_Lacunas_e_Riscos.md
├── Eixo_D_Plano_de_Evolucao.md

```

---

## Metodologia

A análise foi conduzida diretamente no repositório do Crawl4AI no GitHub, com inspeção de issues, pull requests, releases, histórico de commits, estrutura do código-fonte e documentação. Cada eixo foi avaliado segundo os critérios do MPS.BR Nível G e dos padrões de projeto GoF.

### Eixos da Auditoria

**I. Eixo A — Estratégia Atual de Testes**

Avaliação da organização dos testes, tipos existentes, documentação, automação e integração contínua.

**II. Eixo B — Qualidade Técnica dos Testes**

Análise da qualidade da suíte de testes quanto à clareza, isolamento, independência, uso de mocks, cobertura funcional e facilidade de manutenção.

**III. Eixo C — Lacunas, Riscos e Falhas Potenciais**

Identificação das funcionalidades críticas sem cobertura adequada, módulos com maior risco de regressão e fragilidades na estratégia de testes.

**IV. Eixo D — Plano de Evolução da Qualidade**

Proposta priorizada para evolução da estratégia de testes, contemplando níveis de teste, automação, implantação gradual e critérios de maturidade.

---

## Vídeo de Apresentação

**Link do vídeo:** 
https://youtu.be/IFghsHXxrFA

---


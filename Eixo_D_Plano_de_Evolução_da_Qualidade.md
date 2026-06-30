## D.1 — Priorização: Top 3 Prioridades de Teste
 
---
 
### Prioridade 1 — Testes de Contrato para a Camada de LLM
 
**Evidência do problema:**
 
```
https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/extraction_strategy.py
```
 
A dependência do LiteLLM é implícita dentro de `LLMExtractionStrategy`. O supply chain attack da v0.8.5 exigiu um hotfix urgente porque não há testes de contrato que verifiquem a interface entre o projeto e a biblioteca externa.
 
**Por que é prioridade máxima:**
 
Sem testes de contrato, qualquer atualização de dependência, intencional ou maliciosa, pode quebrar silenciosamente o componente mais crítico do projeto (extração via LLM) sem que a suíte de testes detecte. O risco é classificado como **ALTO** pelos eixos anteriores.
 
**Recomendação:**
 
Implementar testes de contrato com `pytest` e `unittest.mock` que verifiquem a interface entre `LLMExtractionStrategy` e o LiteLLM, sem realizar chamadas reais à API:
 
```python
# tests/unit/test_llm_extraction_contract.py
from unittest.mock import AsyncMock, patch
import pytest
from crawl4ai.extraction_strategy import LLMExtractionStrategy
from crawl4ai.models import LLMConfig
 
@pytest.mark.asyncio
async def test_llm_extraction_calls_litellm_with_correct_interface():
    """Garante que LLMExtractionStrategy usa a interface correta do LiteLLM."""
    config = LLMConfig(provider="openai/gpt-4o", api_token="test-key")
    strategy = LLMExtractionStrategy(llm_config=config, instruction="Extraia dados.")
 
    with patch("crawl4ai.extraction_strategy.litellm.acompletion") as mock_llm:
        mock_llm.return_value = AsyncMock(
            choices=[AsyncMock(message=AsyncMock(content='{"result": "ok"}'))]
        )
        result = await strategy.extract(url="https://example.com", html="<p>teste</p>")
        mock_llm.assert_called_once()
        assert result is not None
```
 
**Critério de sucesso:** nenhuma chamada real à API nos testes unitários; qualquer mudança na interface do LiteLLM quebra o teste antes de chegar à produção.
 
---
 
### Prioridade 2 — Testes Unitários do Pipeline Principal (arun)
 
**Evidência do problema:**
 
```
https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/async_webcrawler.py
```
 
O método `arun()` orquestra 7 etapas sem separação formal. Os testes existentes (`docs/examples/docker_example.py`) são E2E e dependem de browser real e URLs externas, tornam-se frágeis e lentos em CI.
 
**Por que é prioridade 2:**
 
A dívida técnica apontada na A2 (violação de SRP e OCP no `AsyncWebCrawler`) torna qualquer refatoração arriscada sem uma suíte de testes unitários que sirva de rede de segurança. Sem esses testes, a refatoração proposta pela Chain of Responsibility não pode ser executada com segurança.
 
**Recomendação:**
 
Criar testes unitários por etapa do pipeline, mockando dependências externas:
 
```python
# tests/unit/test_pipeline_stages.py
from unittest.mock import AsyncMock, MagicMock, patch
import pytest
from crawl4ai import AsyncWebCrawler, CrawlerRunConfig, BrowserConfig
 
@pytest.mark.asyncio
async def test_cache_hit_skips_browser():
    """Se cache HIT, o browser não deve ser acionado."""
    config = BrowserConfig(headless=True)
    run_config = CrawlerRunConfig()
 
    with patch("crawl4ai.async_webcrawler.BrowserManager") as mock_bm:
        with patch("crawl4ai.async_webcrawler.CrawlCache") as mock_cache:
            mock_cache.return_value.read.return_value = MagicMock(success=True)
            async with AsyncWebCrawler(config=config) as crawler:
                result = await crawler.arun("https://example.com", config=run_config)
            mock_bm.return_value.get_browser_context.assert_not_called()
```
 
**Critério de sucesso:** cada etapa do pipeline testável de forma isolada, sem browser real, em menos de 5 segundos por teste.
 
---
 
### Prioridade 3 — Testes de Regressão para ExtractionStrategy
 
**Evidência do problema:**
 
```
https://github.com/unclecode/crawl4ai/blob/main/crawl4ai/extraction_strategy.py
```
 
A `ExtractionStrategy` é o componente mais bem projetado do projeto (Strategy pattern correto), mas não possui testes unitários que garantam a intercambialidade das implementações. Um bug em `JsonCssExtractionStrategy` ou `RegexExtractionStrategy` não seria detectado antes de afetar usuários.
 
**Por que é prioridade 3:**
 
Esses componentes são LLM-free, testáveis de forma rápida, determinística e sem dependências externas. São o ponto de maior custo-benefício para começar a construir cobertura real.
 
**Recomendação:**
 
```python
# tests/unit/test_extraction_strategies.py
import pytest
from crawl4ai.extraction_strategy import (
    JsonCssExtractionStrategy,
    RegexExtractionStrategy
)
 
HTML_SAMPLE = """
<div class="product">
    <span class="name">Produto A</span>
    <span class="price">R$ 99,90</span>
</div>
"""
 
def test_json_css_extracts_correct_fields():
    schema = {
        "name": "Produtos",
        "baseSelector": ".product",
        "fields": [
            {"name": "name", "selector": ".name", "type": "text"},
            {"name": "price", "selector": ".price", "type": "text"},
        ]
    }
    strategy = JsonCssExtractionStrategy(schema=schema)
    result = strategy.extract(url="https://example.com", html=HTML_SAMPLE)
    assert len(result) == 1
    assert result[0]["name"] == "Produto A"
    assert result[0]["price"] == "R$ 99,90"
 
def test_strategies_are_interchangeable():
    """Garante que diferentes strategies respeitam o mesmo contrato."""
    regex_strategy = RegexExtractionStrategy()
    css_strategy = JsonCssExtractionStrategy(schema={"name": "t", "baseSelector": "p", "fields": []})
 
    for strategy in [regex_strategy, css_strategy]:
        result = strategy.extract(url="https://example.com", html="<p>test</p>")
        assert isinstance(result, list)
```
 
---
 
## D.2 — Camadas de Foco: Onde Investir Primeiro
 
---
 
A estratégia de camadas é construída em pirâmide invertida de urgência, das camadas mais rápidas e críticas para as mais custosas:
 
| Camada | Foco | Ferramentas | Timing |
|---|---|---|---|
| **Unitária** | ExtractionStrategies (LLM-free), modelos de dados, chunking | pytest + unittest.mock | Imediato |
| **Contrato** | Interface LLMExtractionStrategy ↔ LiteLLM | pytest + pytest-mock | Semana 1-2 |
| **Integração** | Pipeline completo com browser mockado | pytest-asyncio + AsyncMock | Semana 3-4 |
| **Regressão** | Fluxos centrais do arun() contra HTML estático | pytest + fixtures HTML | Semana 5-6 |
| **E2E** | Testes contra URLs controladas (mockserver) | pytest + pytest-httpserver | Fase 2 |
 
**Justificativa da ordem:**
 
Os testes unitários das strategies LLM-free têm custo zero de infraestrutura e cobertura imediata dos componentes mais intercambiáveis. Os testes de contrato resolvem diretamente o risco ALTO do supply chain attack. Os testes de integração com pipeline mockado criam a rede de segurança necessária para a refatoração proposta na A2.
 
---
 
## D.3 — Implantação Gradual: Por Onde Começar
 
---
 
O Crawl4AI é um projeto open-source ativo com releases mensais. A estratégia de implantação deve ser não-intrusiva:
 
**Fase 1 — Semanas 1 a 2: Fundação (zero risco de quebra)**
 
1. Criar pasta `/tests/unit/` separada de `/docs/examples/`
2. Adicionar `conftest.py` com fixtures HTML estáticas reutilizáveis
3. Implementar testes das strategies LLM-free (`JsonCss`, `Regex`, `Cosine` com modelo mockado)
4. Configurar `pytest.ini` com marcadores: `@pytest.mark.unit`, `@pytest.mark.integration`, `@pytest.mark.e2e`
```
# Estrutura proposta
tests/
  unit/
    test_extraction_strategies.py
    test_chunking_strategies.py
    test_llm_extraction_contract.py
    test_markdown_generation.py
  integration/
    test_pipeline_stages.py
    test_browser_pool.py
  fixtures/
    sample_html/
      news_page.html
      ecommerce_page.html
    conftest.py
```
 
**Fase 2 — Semanas 3 a 4: Contratos e Pipeline**
 
1. Implementar testes de contrato para LiteLLM com mocks
2. Criar testes de integração do pipeline com `BrowserManager` mockado
3. Adicionar `pyproject.toml` com configuração de cobertura mínima por módulo
**Fase 3 — Semanas 5 a 6: Regressão e CI**
 
1. Implementar `pytest-httpserver` para servir HTML estático nos testes E2E leves
2. Configurar GitHub Actions para executar testes unitários em todo PR
3. Adicionar relatório de cobertura com `pytest-cov`
---
 
## D.4 — Automação: Ferramentas, Pipeline e Política
 
---
 
### Ferramentas Recomendadas
 
| Ferramenta | Papel | Justificativa |
|---|---|---|
| `pytest` + `pytest-asyncio` | Framework principal | Já usado informalmente no projeto; suporte nativo a async |
| `unittest.mock` / `pytest-mock` | Isolamento de dependências | Mockar LiteLLM, Playwright, APIs externas sem custo |
| `pytest-cov` | Cobertura de código | Relatório por módulo; integração com GitHub Actions |
| `pytest-httpserver` | Servidor HTTP estático para testes E2E | Elimina dependência de URLs externas nos testes |
| `Codecov` | Visualização de cobertura no GitHub | Badge público + comentário automático em PRs |
 
### Pipeline GitHub Actions Proposto
 
```yaml
# .github/workflows/tests.yml
name: Test Suite
 
on:
  pull_request:
    branches: [main, dev]
  push:
    branches: [main]
 
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install -e ".[all]" pytest pytest-asyncio pytest-mock pytest-cov
      - name: Run unit tests
        run: |
          pytest tests/unit/ \
            -m "unit" \
            --cov=crawl4ai \
            --cov-report=xml \
            --cov-fail-under=60
      - uses: codecov/codecov-action@v5
        with:
          files: coverage.xml
 
  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install -e ".[all]" pytest pytest-asyncio pytest-mock playwright
      - run: playwright install chromium
      - name: Run integration tests
        run: pytest tests/integration/ -m "integration" --maxfail=3
```
 
### Política Mínima de Execução
 
| Evento | Testes Executados | Gate |
|---|---|---|
| Pull Request aberto | Unitários + Contrato | Obrigatório para merge |
| Push na `main` | Unitários + Integração | Obrigatório |
| Release tag | Unitários + Integração + E2E leve | Obrigatório |
| Nightly (agendado) | Suíte completa incluindo E2E real | Informativo |
 
---
 
## D.5 — Critérios de Sucesso
 
---
 
Os critérios abaixo definem o que constitui avanço real de maturidade em testes, diferenciando quantidade de arquivos de qualidade efetiva de cobertura:
 
### Critérios de Entrada (condição mínima para iniciar cada fase)
 
| Fase | Critério de Entrada |
|---|---|
| Fase 1 (Unitários) | `conftest.py` com fixtures HTML estáticas criado; `pytest.ini` configurado com marcadores |
| Fase 2 (Contratos) | 100% das strategies LLM-free cobertas por testes unitários |
| Fase 3 (Integração) | Testes de contrato do LiteLLM passando sem chamadas reais à API |
 
### Critérios de Saída (definição de "feito" por fase)
 
| Fase | Critério de Saída | Métrica |
|---|---|---|
| Fase 1 | Strategies LLM-free testadas e passando em CI | Cobertura ≥ 80% em `extraction_strategy.py` para métodos não-LLM |
| Fase 2 | Testes de contrato implementados e integrados ao CI | Nenhuma chamada real à API nos testes unitários |
| Fase 3 | Pipeline com stages testáveis isoladamente | Tempo de execução dos unitários < 60 segundos no CI |
 
### Indicadores de Maturidade Real
 
| Indicador | Situação Atual | Meta |
|---|---|---|
| Cobertura de testes unitários | ~0% (apenas E2E) | ≥ 60% nos módulos críticos |
| Testes executados sem browser | 0 | > 80% da suíte |
| Tempo médio de CI em PRs | Sem CI de testes | < 3 minutos para unitários |
| Detecção de regressão em LLM interface | Nenhuma (descoberta em produção) | Antes do merge via contrato |
| Badge de cobertura no README | Ausente | Presente e atualizado automaticamente |
 
---
 
## Conclusão
 
O Crawl4AI possui uma arquitetura com boas decisões de design — especialmente no padrão Strategy, que facilitam a adoção de testes unitários. O principal obstáculo hoje não é a complexidade do código, mas a ausência de separação entre testes E2E (que dependem de browser e URLs reais) e testes unitários (que podem rodar em milissegundos sem infraestrutura).
 
O plano proposto é conservador e não-intrusivo: começa pelos componentes LLM-free, que são determinísticos e imediatamente testáveis, e evolui gradualmente para contratos de LLM e integração de pipeline. As três prioridades, testes de contrato para LLM, unitários do pipeline e regressão das strategies, respondem diretamente às dívidas técnicas identificadas nas Atividades 1 e 2, criando a rede de segurança necessária para que as refatorações propostas possam ser executadas com confiança.

## Ferramentas de IA Utilizadas

Para a elaboração desta seção, foi utilizado o Claude como ferramenta de apoio na estruturação e redação do texto, bem como na análise das evidências obtidas a partir do repositório GitHub do Crawl4AI.
 
---
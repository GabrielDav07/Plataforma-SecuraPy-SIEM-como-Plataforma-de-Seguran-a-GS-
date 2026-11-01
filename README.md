# Plataforma SecuraPy: SIEM como-Plataforma de Seguran a GS
# SecuraPy — SIEM Simplificado

> **Global Solution — Coding for Security**  
> CyberShield Ltda. · MVP de um SIEM (*Security Information and Event Management*)

## Sobre o projeto

O **SecuraPy** é um sistema simplificado de gerenciamento de eventos e informações de segurança (SIEM), desenvolvido em Python. O projeto recebe registros de autenticação, firewall e servidor web, transforma esses registros em eventos padronizados e utiliza análises para auxiliar na identificação de atividades suspeitas.

Entre os comportamentos investigados estão:

- Brute force;
- Port scan;
- Path traversal;
- Cross-Site Scripting (XSS);
- Reconhecimento web.

## Escopo desta versão

**Esta versão do repositório contempla os módulos A, B e C da atividade.**

| Módulo | Escopo | Situação nesta versão |
|---|---|---|
| **A** | Núcleo do sistema: coleta, normalização e análise/correlação de eventos. | Incluído |
| **B** | Persistência dos eventos em MongoDB. | Incluído |
| **C** | Machine Learning, classificação de ameaças e integridade SHA-256 do modelo. | Incluído |
| D | API REST com Flask. | Não incluído nesta versão |
| E | Dashboard web. | Não incluído nesta versão |
| F | Docker e conteinerização. | Não incluído nesta versão |
| G | Assistente com LLM. | Não incluído nesta versão |

Os módulos D a G não são apresentados como funcionalidades implementadas nesta versão.

## Arquitetura dos módulos A, B e C

```text
                  Arquivos de log
             auth / firewall / web
                       |
                       v
                  coletor.py
                       |
                       v
              Eventos normalizados
                 /           \
                v             v
         banco_dados.py   correlacao.py
                |             |
                v             v
             MongoDB        Alertas
                              |
                              v
                      Machine Learning
                              |
                              v
                    Classificação e score
                       de risco por IP
```

## Módulos incluídos

### Módulo A — Núcleo do sistema

O núcleo realiza a leitura e interpretação dos logs e converte cada linha válida em um evento padronizado.

**Arquivos principais:**

- `coletor.py`: leitura, parsing e normalização dos logs;
- `correlacao.py`: análise temporal de falhas de autenticação;
- `gerador_logs.py`: geração de logs de teste, incluindo tráfego normal e ataques simulados.

O evento normalizado pelo coletor utiliza os campos:

```python
{
    "timestamp": "2025-02-20 08:15:01",
    "fonte": "auth",
    "tipo": "FAIL",
    "ip": "185.220.101.1",
    "detalhes": "usuario=admin",
    "linha_original": "..."
}
```

As fontes utilizadas são `auth`, `firewall` e `web`. O campo `detalhes` contém informações específicas da origem, como usuário, protocolo, destino, porta, método, URL e status HTTP.

A correlação temporal identifica possíveis ataques de brute force ao analisar falhas do mesmo IP dentro de uma janela de tempo. Os alertas possuem os campos `timestamp`, `regra`, `severidade`, `ip` e `descricao`.

### Módulo B — Banco de dados

O arquivo `banco_dados.py` realiza a persistência dos eventos no **MongoDB**, utilizando o banco `securapy` e a coleção `eventos`.

Principais operações:

- Conectar ao MongoDB;
- Salvar eventos;
- Evitar duplicidade de registros;
- Buscar eventos por endereço IP;
- Contar eventos por fonte.

A identificação de eventos utiliza a fonte e a linha original para evitar que a execução repetida do processo gere documentos duplicados.

### Módulo C — Machine Learning e integridade

O módulo C complementa a análise por regras utilizando um classificador supervisionado.

**Arquivos:**

| Arquivo | Responsabilidade |
|---|---|
| `features.py` | Extrai características dos eventos e de resumos de ameaças por IP. |
| `treinar.py` | Prepara os dados, treina e avalia o classificador e salva o artefato. |
| `classificar.py` | Verifica a integridade do modelo, classifica e ordena ameaças por score. |
| `test_ml.py` | Testes automatizados do módulo. |
| `requirements-ml.txt` | Dependências específicas do Machine Learning. |

#### Características utilizadas

O módulo extrai seis características por endereço IP:

- `falhas_auth`: quantidade de falhas de autenticação;
- `portas_distintas`: quantidade de portas de destino distintas observadas no firewall;
- `ip_blacklist`: indica se o IP pertence à blacklist fornecida;
- `erros_web`: quantidade de respostas HTTP com status igual ou superior a 400;
- `eventos_total`: quantidade total de eventos associados ao IP;
- `bloqueios_firewall`: quantidade de eventos de bloqueio do firewall.

#### Treinamento e avaliação

O treinamento utiliza um Pipeline do Scikit-learn com:

- `StandardScaler`, para padronização das características;
- `RandomForestClassifier`, para classificação entre `ataque` e `normal`;
- `StratifiedKFold` e `cross_validate`, para validação cruzada estratificada.

São calculadas as métricas de precisão (*precision*), recall e F1-score. Os rótulos por IP são fornecidos separadamente em um arquivo JSON e devem representar uma referência revisada; o módulo não cria automaticamente a verdade-terreno.

#### Classificação e score de risco

A classificação acrescenta aos registros:

- `score_risco`: probabilidade estimada da classe ataque, expressa de 0 a 100;
- `rotulo_ml`: classe prevista (`ataque` ou `normal`);
- `risco_ml`: faixa de risco (`ALTO`, `MEDIO` ou `BAIXO`).

As faixas utilizadas são:

| Score | Faixa |
|---|---|
| 70 a 100 | ALTO |
| 40 a 69,99 | MEDIO |
| 0 a 39,99 | BAIXO |

O score é um indicador para priorização e triagem, não uma confirmação definitiva de que ocorreu um ataque.

#### Integridade SHA-256

Após o treinamento, o modelo é persistido com `joblib`. O módulo calcula seu hash SHA-256 e grava o valor em um arquivo auxiliar com a extensão `.sha256`.

Antes de carregar o modelo, `classificar.py` calcula novamente o hash e compara os valores. Se houver divergência, o carregamento é interrompido.

> **Nota de segurança:** o hash em arquivo separado detecta alterações quando o valor de referência é confiável. Se alguém puder substituir tanto o modelo quanto o arquivo de hash, poderá recalcular o hash. Para proteção contra adulteração intencional, a referência deve ser armazenada ou distribuída por um canal confiável. Como `joblib` utiliza serialização baseada em pickle, carregue apenas modelos de origem confiável.

## Estrutura do repositório

A estrutura abaixo representa os arquivos recebidos para os módulos A, B e C. Os nomes de diretórios `Logs`, `Config` e `Saida` respeitam a capitalização presente no pacote da Pessoa 1.

```text
securaPy/
├── coletor.py
├── banco_dados.py
├── correlacao.py
├── gerador_logs.py
├── Config/
├── Logs/
│   ├── auth.log
│   ├── firewall.log
│   └── web_access.log
├── logs_gerados/
├── Saida/
└── ml/
    ├── __init__.py
    ├── features.py
    ├── treinar.py
    ├── classificar.py
    ├── test_ml.py
    └── requirements-ml.txt
```

Os arquivos de modelo (`modelo.pkl`) e hash (`modelo.pkl.sha256`) são gerados pelo treinamento e não precisam estar presentes antes da primeira execução.

## Tecnologias e bibliotecas

- **Python 3** — linguagem principal;
- **MongoDB** — persistência de eventos;
- **PyMongo** — conexão com o MongoDB;
- **Scikit-learn** — treinamento e avaliação do classificador;
- **NumPy** — operações numéricas;
- **Joblib** — persistência do modelo;
- **Hashlib** — cálculo do SHA-256;
- **Pytest** — testes automatizados.

## Instalação

### Pré-requisitos

- Python 3 instalado;
- MongoDB instalado e em execução para utilizar a persistência;
- Git instalado.

### Clonar o repositório

```bash
git clone https://github.com/fabioBaraDev/coding_for_security.git
cd coding_for_security
```

> Os comandos abaixo pressupõem que os arquivos do SecuraPy estejam na raiz do repositório, com a pasta `ml/` contendo os arquivos da Pessoa 2.

### Instalar dependências

Para o banco de dados:

```bash
python -m pip install pymongo
```

Para o módulo de Machine Learning:

```bash
python -m pip install -r ml/requirements-ml.txt
```

## Execução e testes

### Coletor

Execute a partir da pasta que contém `coletor.py`:

```bash
python coletor.py
```

Com os logs de teste descritos no README de desenvolvimento da Pessoa 1, o resultado esperado é a leitura de 23 eventos de autenticação, 17 de firewall e 18 web, totalizando 58 eventos.

### Banco de dados

Com o MongoDB em execução:

```bash
python banco_dados.py
```

O módulo grava os eventos e evita duplicações em execuções posteriores.

### Correlação temporal

```bash
python correlacao.py
```

Com os logs de referência descritos pela Pessoa 1, a correlação temporal identifica o IP `185.220.101.1` com 11 falhas em 39 segundos, dentro da janela de 60 segundos.

### Geração de logs

```bash
python gerador_logs.py
```

O gerador cria arquivos de log de teste na pasta `logs_gerados/`.

### Treinamento do Machine Learning

O treinamento recebe:
1. Um JSON contendo uma lista de eventos normalizados;
2. Um JSON com rótulos revisados por IP (`ataque` ou `normal`).

Exemplo de `rotulos.json`:

```json
{
  "185.220.101.1": "ataque",
  "192.168.1.10": "normal"
}
```

Com os arquivos JSON preparados, execute:

```bash
python -m ml.treinar --eventos-json eventos.json --rotulos-json rotulos.json --modelo ml/modelo.pkl
```

É possível fornecer uma blacklist opcional em JSON, contendo uma lista de endereços IP:

```bash
python -m ml.treinar --eventos-json eventos.json --rotulos-json rotulos.json --modelo ml/modelo.pkl --blacklist-json blacklist.json
```

O treinamento exige pelo menos quatro IPs rotulados, com no mínimo dois exemplos em cada classe, para realizar a validação cruzada.

### Classificação de ameaças

Após treinar o modelo, ele pode ser carregado e utilizado por outro módulo:

```python
from ml.classificar import carregar_modelo, pontuar

modelo = carregar_modelo("ml/modelo.pkl")
ameacas_ordenadas = pontuar(ameacas, modelo=modelo)
```

A função `pontuar()` recebe uma lista de resumos de ameaças no formato esperado pelo adaptador do módulo C, acrescenta a classificação e o score e devolve os registros ordenados do maior para o menor risco.

### Testes do módulo C

Instale as dependências de `ml/requirements-ml.txt` e execute:

```bash
python -m pytest ml/test_ml.py
```

## Integração entre os módulos

O fluxo de integração previsto para os módulos incluídos é:

1. O coletor do módulo A lê e normaliza os registros;
2. O módulo B persiste os eventos no MongoDB;
3. A correlação do módulo A identifica padrões temporais e gera alertas;
4. O módulo C extrai características, treina o classificador e calcula scores de risco;
5. Os resultados do módulo C podem ser utilizados pelos componentes seguintes da plataforma quando os contratos de integração forem definidos.

**Ponto de integração a validar:** o módulo C recebe eventos normalizados no formato do coletor, mas o treinamento exige que esses eventos sejam exportados para JSON e que exista um arquivo separado com rótulos revisados por IP. A interface entre os alertas/resumos produzidos pelo núcleo e a função `pontuar()` também deve ser confirmada na integração do grupo. Portanto, não se presume que essa integração esteja automatizada nesta versão.

## Integrantes e responsabilidades

| Integrante | Responsabilidade |
|---|---|
| Pessoa 1 | Módulo A — Núcleo do sistema e Módulo B — Banco de dados |
| Pessoa 2 | Módulo C — Machine Learning e integridade SHA-256 |
| Pessoa 3 | Módulos D e F — API REST com Flask e Docker (não incluídos nesta versão) |
| Pessoa 4 | Módulos E e G — Dashboard web e assistente com LLM (não incluídos nesta versão) |

## Limitações e observações

- Os logs utilizados na atividade têm finalidade educacional e volume limitado.
- As métricas do classificador dependem da quantidade, diversidade e qualidade dos rótulos fornecidos.
- O score de risco é um apoio à priorização, não um veredito.
- A blacklist é fornecida ao módulo de Machine Learning; sua origem e atualização devem ser definidas pelo grupo.
- A integração automatizada entre a exportação de eventos, os rótulos e o classificador deve ser validada.
- Os módulos D a G não estão incluídos nesta versão do repositório.

## Contexto acadêmico

Projeto desenvolvido para a **Global Solution da disciplina Coding for Security**, com o objetivo de aplicar conceitos de programação, análise de logs, persistência de dados, detecção de ameaças e Machine Learning em uma solução simplificada de SIEM.

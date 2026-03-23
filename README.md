# Curso de SQLAlchemy

Este repositório contém o material e exercícios para o curso de SQLAlchemy. O objetivo é aprender a usar o `SQLAlchemy`, uma biblioteca Python para trabalhar com bancos de dados relacionais.

## Gerenciamento de Ambiente Virtual

Estamos usando o **Poetry** como gerenciador de dependências e ambientes virtuais. O Poetry simplifica o gerenciamento de pacotes Python, resolve conflitos de dependências e cria ambientes isolados automaticamente.

### Por que Poetry?
- **Isolamento**: Cada projeto tem seu próprio ambiente virtual.
- **Gerenciamento de dependências**: Arquivo `pyproject.toml` centraliza tudo.
- **Reprodutibilidade**: Lockfile (`poetry.lock`) garante versões exatas.
- **Simplicidade**: Comandos intuitivos para instalar, adicionar e remover pacotes.

## Pré-requisitos

- Python 3.8 ou superior instalado.

### Instalação do Poetry

- **No Windows**: Abra o PowerShell como administrador e execute:
  ```powershell
  (Invoke-WebRequest -Uri https://install.python-poetry.org -UseBasicParsing).Content | python -
  ```
  Ou instale via pip: `pip install poetry`

- **No Linux/Mac**:
  ```bash
  curl -sSL https://install.python-poetry.org | python3 -
  ```

- Para mais detalhes, consulte [poetry.eustace.io](https://python-poetry.org/docs/#installation).

## Configuração do Projeto

1. **Clone o repositório** (se aplicável):
   ```bash
   git clone git@github.com:dforoni/sqlalchemy-curso.git
   cd sqlalchemy-curso
   ```

2. **Instale as dependências**:
   ```bash
   poetry install
   ```
   Isso criará um ambiente virtual e instalará todos os pacotes listados no `pyproject.toml`.

3. **Ative o ambiente virtual** (opcional, Poetry faz isso automaticamente em muitos comandos):
   ```bash
   poetry shell
   ```

## Como Usar

- Para executar scripts Python: `poetry run python seu_script.py`
- Para adicionar novas dependências: `poetry add nome-do-pacote`
- Para remover dependências: `poetry remove nome-do-pacote`
- Para atualizar dependências: `poetry update`

## Estrutura do Projeto

- `pyproject.toml`: Configuração do Poetry e dependências.
- `.gitignore`: Arquivos ignorados pelo Git (incluindo arquivos `.db` para bancos de dados).
- Outros arquivos: Scripts e materiais do curso serão adicionados aqui.

## Licença

Este projeto é para fins educacionais. 

---


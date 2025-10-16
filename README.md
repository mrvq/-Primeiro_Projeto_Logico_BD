# 🛒 Projeto de Banco de Dados - E-commerce

## 📘 Descrição do Projeto

Este projeto apresenta o **modelo lógico e físico** de um banco de dados para um **sistema de e-commerce**, desenvolvido como parte de um desafio prático de modelagem de dados.

O objetivo é demonstrar desde a **modelagem conceitual (EER)** até a **implementação em SQL**, incluindo a criação do esquema, inserção de dados de exemplo e consultas complexas (queries) que respondem a perguntas de negócio.

---

## 🧩 Estrutura do Projeto

📁 ecommerce-db/
├── README.md
├── schema.sql # Criação do esquema do banco de dados (DDL)
├── data.sql # Inserção de dados de exemplo (DML)
├── queries.sql # Consultas SQL complexas (SELECT, WHERE, JOIN, HAVING etc.)
└── diagrama-er.png # Diagrama Entidade-Relacionamento (EER)


---

## 🗃️ Modelagem do Banco de Dados

O modelo contempla:

- **Cliente (PF e PJ)** — uma conta pode ser Pessoa Física ou Jurídica, mas nunca ambas.
- **Forma de Pagamento** — clientes podem ter múltiplas formas de pagamento cadastradas.
- **Pedidos e Itens** — pedidos com múltiplos produtos e status de acompanhamento.
- **Entrega** — com status e código de rastreio.
- **Fornecedor e Vendedor** — com possibilidade de um mesmo CNPJ estar em ambas as entidades.
- **Produto e Estoque** — controle de produtos, fornecedores e disponibilidade por local.

---

## 🧠 Regras de Negócio Principais

- Um **cliente** pode ser **PF** ou **PJ**, mas não ambos.
- Cada **cliente** pode possuir **várias formas de pagamento**.
- Um **pedido** pertence a um **cliente** e pode ter um **vendedor** associado.
- Cada **pedido** possui **itens** e está vinculado a uma **entrega**.
- **Fornecedores** fornecem **produtos**, e **produtos** podem ter múltiplos fornecedores (relação N:N).
- O **estoque** é controlado por produto e local.

---

## 🧱 Diagrama Entidade-Relacionamento

![Diagrama ER](./diagrama-er.png)

---

## 🧾 Exemplos de Consultas (Queries)

Algumas perguntas de negócio respondidas pelas queries incluídas:

1. **Quantos pedidos foram feitos por cada cliente?**
2. **Qual o total gasto por cliente?**
3. **Existe algum vendedor que também é fornecedor?**
4. **Quais fornecedores fornecem cada produto e seus estoques?**
5. **Quais produtos têm estoque baixo?**
6. **Clientes com ticket médio maior que R$400.**
7. **Total faturado por fornecedor.**
8. **Entregas atrasadas.**
9. **Auditoria: pedidos com discrepância entre valor total e soma dos itens.**

---

## ⚙️ Como Executar o Projeto

1. Clone o repositório:
   ```bash
   git clone https://github.com/seuusuario/ecommerce-db.git
   cd ecommerce-db
2. Execute o script de criação do esquema:

psql -U postgres -f schema.sql


3. Popule o banco com dados de exemplo:

psql -U postgres -f data.sql


4. Rode as consultas:

psql -U postgres -f queries.sql

🧮 Tecnologias Utilizadas

PostgreSQL 15+

SQL padrão ANSI

Diagramas EER (gerados automaticamente)

Markdown para documentação

🧑‍💻 Autor

Marcio Queirantes
Analista de Sistemas Jr | Desenvolvedor Full Stack
📧 LinkedIn
 | 💻 GitHub

Projeto criado como parte de um desafio de modelagem de banco de dados — Bootcamp Randstad - Análise de Dados / Formação DIO.

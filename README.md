# -Primeiro_Projeto_Logico_BD
 Primeiro Projeto Lógico de Banco de Dados
-- Project: E-commerce Logical Schema (PostgreSQL)
-- Files included in this single document:
-- 1) README.md
-- 2) schema.sql  (DDL)
-- 3) data.sql    (inserts)
-- 4) queries.sql (sample complex queries + questions)

/* ================= README.md ================= */
-- README.md
--
-- Descrição do projeto
-- Este repositório contém a modelagem lógica de banco de dados para um cenário de e-commerce
-- contemplando: clientes (Pessoa Física e Jurídica), múltiplas formas de pagamento por cliente,
-- entregas com status e código de rastreio, fornecedores, vendedores (possível sobreposição com fornecedores),
-- produtos, estoques e pedidos. As constraints (PK, FK, CHECK, UNIQUE) foram definidas conforme o enunciado.
--
-- Instruções
-- 1) Rode o arquivo schema.sql em um banco PostgreSQL (psql) para criar as tabelas.
-- 2) Rode o arquivo data.sql para popular com dados de exemplo.
-- 3) Rode queries.sql para testar consultas.
--
-- Observações de modelagem
-- - A tabela cliente armazena dados comuns; as informações específicas de PF/PJ ficam em tabelas separadas
--   cliente_pf e cliente_pj. Uma conta só pode ser PF ou PJ (constraint de exclusão mútua implementada
--   com CHECK e regras de inserção previstas no design).
-- - Um cliente pode ter múltiplas formas de pagamento (tabela cliente_pagamento).
-- - Entregas possuem status e código de rastreio (tabela entrega).
-- - Vendedor e Fornecedor são entidades separadas; existe possibilidade de um vendedor também ser fornecedor
--   (ex.: mesmo CNPJ/CPF) — uma query demonstra como verificar esse caso.


/* ================= schema.sql ================= */

-- DROP existing objects (careful in production) ------------------------------------------------
DROP SCHEMA public CASCADE;
CREATE SCHEMA public;

-- Use UTF8
SET client_encoding = 'UTF8';

-- Tabela: cliente (conta)
CREATE TABLE cliente (
    cliente_id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    senha_hash VARCHAR(255) NOT NULL,
    telefone VARCHAR(20),
    criado_em TIMESTAMP WITHOUT TIME ZONE DEFAULT now(),
    -- flags to indicate type; ensure one and only one of pf_id or pj_id is present via constraints
    tipo CHAR(2) NOT NULL CHECK (tipo IN ('PF','PJ'))
);

-- Tabela: cliente_pf (Pessoa Física)
CREATE TABLE cliente_pf (
    cliente_id INTEGER PRIMARY KEY REFERENCES cliente(cliente_id) ON DELETE CASCADE,
    cpf CHAR(11) NOT NULL UNIQUE,
    nome_completo VARCHAR(200) NOT NULL,
    data_nascimento DATE
);

-- Tabela: cliente_pj (Pessoa Jurídica)
CREATE TABLE cliente_pj (
    cliente_id INTEGER PRIMARY KEY REFERENCES cliente(cliente_id) ON DELETE CASCADE,
    cnpj CHAR(14) NOT NULL UNIQUE,
    razao_social VARCHAR(200) NOT NULL,
    nome_fantasia VARCHAR(200)
);

-- Constraint to ensure PF/PJ exclusivity is enforced by application logic plus
-- a trigger could enforce it; here we add a partial exclusion via a check using tipo column.
-- Since cliente.tipo already indicates PF or PJ, we rely on that being consistent with presence
-- of row in cliente_pf or cliente_pj. (For strict DB-level enforcement, triggers would be added.)

-- Endereço (poderia ter mais de um por cliente)
CREATE TABLE endereco (
    endereco_id SERIAL PRIMARY KEY,
    cliente_id INTEGER NOT NULL REFERENCES cliente(cliente_id) ON DELETE CASCADE,
    logradouro VARCHAR(255) NOT NULL,
    numero VARCHAR(20),
    complemento VARCHAR(100),
    bairro VARCHAR(100),
    cidade VARCHAR(100) NOT NULL,
    estado CHAR(2) NOT NULL,
    cep CHAR(8),
    tipo_endereco VARCHAR(20) DEFAULT 'residencial'
);

-- Tabela: forma_pagamento (catalogo)
CREATE TABLE forma_pagamento (
    forma_pagamento_id SERIAL PRIMARY KEY,
    nome VARCHAR(50) NOT NULL UNIQUE, -- Ex: 'Cartao', 'Boleto', 'Pix'
    descricao TEXT
);

-- Tabela: cliente_pagamento (um cliente pode ter N formas de pagamento cadastradas)
CREATE TABLE cliente_pagamento (
    cliente_pagamento_id SERIAL PRIMARY KEY,
    cliente_id INTEGER NOT NULL REFERENCES cliente(cliente_id) ON DELETE CASCADE,
    forma_pagamento_id INTEGER NOT NULL REFERENCES forma_pagamento(forma_pagamento_id),
    titular VARCHAR(200),
    detalhes JSONB, -- armazenar máscara de cartão, banco, etc (atenção à segurança)
    criado_em TIMESTAMP DEFAULT now(),
    UNIQUE(cliente_id, forma_pagamento_id, titular)
);

-- Tabela: fornecedor
CREATE TABLE fornecedor (
    fornecedor_id SERIAL PRIMARY KEY,
    nome VARCHAR(200) NOT NULL,
    cnpj CHAR(14) UNIQUE,
    telefone VARCHAR(20),
    email VARCHAR(255)
);

-- Tabela: vendedor
CREATE TABLE vendedor (
    vendedor_id SERIAL PRIMARY KEY,
    nome VARCHAR(200) NOT NULL,
    cpf CHAR(11) UNIQUE,
    cnpj CHAR(14), -- opcional se pessoa jurídica
    email VARCHAR(255)
);

-- Tabela: produto
CREATE TABLE produto (
    produto_id SERIAL PRIMARY KEY,
    sku VARCHAR(50) NOT NULL UNIQUE,
    nome VARCHAR(255) NOT NULL,
    descricao TEXT,
    preco DECIMAL(12,2) NOT NULL CHECK (preco >= 0),
    peso_kg DECIMAL(8,3) DEFAULT 0
);

-- Tabela: produto_fornecedor (relationship M:N entre produtos e fornecedores)
CREATE TABLE produto_fornecedor (
    produto_id INTEGER NOT NULL REFERENCES produto(produto_id) ON DELETE CASCADE,
    fornecedor_id INTEGER NOT NULL REFERENCES fornecedor(fornecedor_id) ON DELETE CASCADE,
    fornecedor_sku VARCHAR(100),
    preco_fornecedor DECIMAL(12,2),
    prazo_entrega_dias INTEGER,
    PRIMARY KEY (produto_id, fornecedor_id)
);

-- Tabela: estoque (por produto e por local)
CREATE TABLE estoque (
    estoque_id SERIAL PRIMARY KEY,
    produto_id INTEGER NOT NULL REFERENCES produto(produto_id) ON DELETE CASCADE,
    local_estoque VARCHAR(100) DEFAULT 'principal',
    quantidade INTEGER NOT NULL CHECK (quantidade >= 0),
    UNIQUE(produto_id, local_estoque)
);

-- Tabela: pedido
CREATE TABLE pedido (
    pedido_id SERIAL PRIMARY KEY,
    cliente_id INTEGER NOT NULL REFERENCES cliente(cliente_id),
    vendedor_id INTEGER REFERENCES vendedor(vendedor_id),
    data_pedido TIMESTAMP DEFAULT now(),
    valor_total DECIMAL(12,2) DEFAULT 0 CHECK (valor_total >= 0),
    status_pedido VARCHAR(30) DEFAULT 'EM_PROCESSAMENTO'
);

-- Tabela: item do pedido
CREATE TABLE pedido_item (
    pedido_item_id SERIAL PRIMARY KEY,
    pedido_id INTEGER NOT NULL REFERENCES pedido(pedido_id) ON DELETE CASCADE,
    produto_id INTEGER NOT NULL REFERENCES produto(produto_id),
    quantidade INTEGER NOT NULL CHECK (quantidade > 0),
    preco_unitario DECIMAL(12,2) NOT NULL CHECK (preco_unitario >= 0),
    desconto DECIMAL(12,2) DEFAULT 0 CHECK (desconto >= 0)
);

-- Tabela: entrega (shipment)
CREATE TABLE entrega (
    entrega_id SERIAL PRIMARY KEY,
    pedido_id INTEGER NOT NULL REFERENCES pedido(pedido_id) ON DELETE CASCADE,
    endereco_id INTEGER NOT NULL REFERENCES endereco(endereco_id),
    status VARCHAR(50) NOT NULL DEFAULT 'AGUARDANDO_SEPARACAO', -- Ex: AGUARDANDO_SEPARACAO, EM_TRANSITO, ENTREGUE, DEVOLVIDO
    codigo_rastreio VARCHAR(100),
    data_envio TIMESTAMP,
    data_entrega TIMESTAMP
);

-- Tabela: pagamento (registro de pagamento do pedido)
CREATE TABLE pagamento (
    pagamento_id SERIAL PRIMARY KEY,
    pedido_id INTEGER NOT NULL REFERENCES pedido(pedido_id) ON DELETE CASCADE,
    forma_pagamento_id INTEGER NOT NULL REFERENCES forma_pagamento(forma_pagamento_id),
    cliente_pagamento_id INTEGER REFERENCES cliente_pagamento(cliente_pagamento_id),
    valor_pago DECIMAL(12,2) NOT NULL CHECK (valor_pago >= 0),
    status_pagamento VARCHAR(30) DEFAULT 'PENDENTE', -- PAGO, PENDENTE, ESTORNADO
    data_pagamento TIMESTAMP DEFAULT now()
);

-- Índices de suporte
CREATE INDEX idx_pedido_cliente ON pedido(cliente_id);
CREATE INDEX idx_pedido_data ON pedido(data_pedido);
CREATE INDEX idx_entrega_status ON entrega(status);

-- Triggers/Rules: garantir consistência entre cliente.tipo e tabelas PF/PJ
-- Implementaremos triggers simples para evitar inconsistências.

CREATE OR REPLACE FUNCTION verify_cliente_tipo_on_insert() RETURNS trigger AS $$
BEGIN
    IF NEW.tipo = 'PF' THEN
        -- Expect a row in cliente_pf to be created by the application afterwards
        RETURN NEW;
    ELSIF NEW.tipo = 'PJ' THEN
        RETURN NEW;
    ELSE
        RAISE EXCEPTION 'tipo inválido para cliente: %', NEW.tipo;
    END IF;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_verify_cliente_tipo
    BEFORE INSERT OR UPDATE ON cliente
    FOR EACH ROW
    EXECUTE FUNCTION verify_cliente_tipo_on_insert();

-- É possível adicionar triggers para garantir exclusão mútua entre cliente_pf e cliente_pj,
-- porém esse script mantém a consistência via coluna tipo e boas práticas de aplicação.


/* ================= data.sql ================= */

-- Inserção de formas de pagamento
INSERT INTO forma_pagamento(nome, descricao) VALUES
('Cartao', 'Cartão de crédito/débito'),
('Boleto', 'Boleto bancário'),
('Pix', 'PIX instantâneo');

-- Clientes
INSERT INTO cliente(email, senha_hash, telefone, tipo) VALUES
('maria.pf@example.com', 'hash1', '11999990000','PF'),
('empresa_x@example.com', 'hash2', '1133333000','PJ'),
('joao.pf@example.com', 'hash3', '11988887777','PF');

-- Dados PF / PJ
INSERT INTO cliente_pf(cliente_id, cpf, nome_completo, data_nascimento) VALUES
(1,'12345678901','Maria Silva','1985-03-10'),
(3,'98765432100','João Souza','1990-07-20');

INSERT INTO cliente_pj(cliente_id, cnpj, razao_social, nome_fantasia) VALUES
(2,'12345678000199','Empresa X Ltda','Loja X');

-- Endereços
INSERT INTO endereco(cliente_id, logradouro, numero, bairro, cidade, estado, cep, tipo_endereco) VALUES
(1,'Rua A','100','Centro','Sao Paulo','SP','01001000','residencial'),
(2,'Av. B','500','Jardim','Sao Paulo','SP','02002000','comercial'),
(3,'Rua C','12','Vila','Campinas','SP','13013010','residencial');

-- Formas de pagamento do cliente
INSERT INTO cliente_pagamento(cliente_id, forma_pagamento_id, titular, detalhes) VALUES
(1,1,'Maria Silva','{"cartao":"**** **** **** 1111","bandeira":"VISA"}'),
(1,3,'Maria Silva','{"pix":"maria.pix@example.com"}'),
(2,2,'Empresa X','{"banco":"001","agencia":"1234"}');

-- Fornecedores e vendedores
INSERT INTO fornecedor(nome, cnpj, telefone, email) VALUES
('Fornecedor A','11111111000111','1132211000','fornA@example.com'),
('Fornecedor B','22222222000122','1132211001','fornB@example.com');

INSERT INTO vendedor(nome, cpf, cnpj, email) VALUES
('Carlos Vendedor','55544433322',NULL,'carlos.v@example.com'),
('FornecedorA Vendedor','', '11111111000111','vendor_fornA@example.com'); -- same CNPJ as Fornecedor A

-- Produtos
INSERT INTO produto(sku, nome, descricao, preco, peso_kg) VALUES
('SKU-001','Teclado Mecanico','Teclado mecânico RGB',350.00,0.9),
('SKU-002','Mouse Gamer','Mouse com sensor ótimo',120.00,0.2),
('SKU-003','Monitor 24"','Monitor 24 polegadas',900.00,3.6);

-- Produto-Fornecedor
INSERT INTO produto_fornecedor(produto_id, fornecedor_id, fornecedor_sku, preco_fornecedor, prazo_entrega_dias) VALUES
(1,1,'F-A-TECLADO-01',200.00,7),
(2,1,'F-A-MOUSE-01',60.00,5),
(3,2,'F-B-MON-24',500.00,10);

-- Estoque
INSERT INTO estoque(produto_id, local_estoque, quantidade) VALUES
(1,'principal',50),
(2,'principal',120),
(3,'principal',25);

-- Pedidos
INSERT INTO pedido(cliente_id, vendedor_id, data_pedido, valor_total, status_pedido) VALUES
(1,1,'2025-10-01 10:00:00',470.00,'CONCLUIDO'),
(2,2,'2025-09-25 15:30:00',500.00,'EM_PROCESSAMENTO'),
(1,1,'2025-10-05 08:20:00',900.00,'CONCLUIDO');

-- Itens de pedido
INSERT INTO pedido_item(pedido_id, produto_id, quantidade, preco_unitario, desconto) VALUES
(1,1,1,350.00,0.00),
(1,2,1,120.00,0.00),
(2,3,1,500.00,0.00),
(3,3,1,900.00,0.00);

-- Entregas
INSERT INTO entrega(pedido_id, endereco_id, status, codigo_rastreio, data_envio, data_entrega) VALUES
(1,1,'ENTREGUE','BR123456789BR','2025-10-02 09:00:00','2025-10-03 14:30:00'),
(2,2,'EM_TRANSITO','BR987654321BR','2025-09-26 08:00:00',NULL),
(3,1,'AGUARDANDO_SEPARACAO',NULL,NULL,NULL);

-- Pagamentos
INSERT INTO pagamento(pedido_id, forma_pagamento_id, cliente_pagamento_id, valor_pago, status_pagamento, data_pagamento) VALUES
(1,1,1,470.00,'PAGO','2025-10-01 10:05:00'),
(2,2,3,500.00,'PENDENTE','2025-09-25 15:35:00'),
(3,1,1,900.00,'PAGO','2025-10-05 08:25:00');


/* ================= queries.sql ================= */
-- Arquivo: queries.sql
-- Contém várias consultas que demonstram os requisitos: SELECT, WHERE, expressões derivadas,
-- ORDER BY, HAVING, JOINs e perguntas que elas respondem.

-- 1) Quantos pedidos foram feitos por cada cliente? (GROUP BY, ORDER BY)
-- Pergunta: "Quantos pedidos cada cliente fez e qual o total gasto?"
SELECT c.cliente_id, c.email,
       COUNT(p.pedido_id) AS total_pedidos,
       COALESCE(SUM(p.valor_total),0) AS gasto_total
FROM cliente c
LEFT JOIN pedido p ON p.cliente_id = c.cliente_id
GROUP BY c.cliente_id, c.email
ORDER BY gasto_total DESC;

-- 2) Algum vendedor também é fornecedor?
-- Pergunta: "Existe algum vendedor cujo CNPJ coincide com um fornecedor?"
SELECT v.vendedor_id, v.nome AS vendedor_nome, v.cnpj AS vendedor_cnpj, f.fornecedor_id, f.nome AS fornecedor_nome
FROM vendedor v
JOIN fornecedor f ON v.cnpj IS NOT NULL AND v.cnpj = f.cnpj;

-- 3) Relação de produtos, fornecedores e estoques (JOINs e atributos derivados)
-- Pergunta: "Quais fornecedores fornecem cada produto, qual o preço do fornecedor e o estoque atual?"
SELECT pr.produto_id, pr.sku, pr.nome AS produto_nome,
       f.fornecedor_id, f.nome AS fornecedor_nome,
       pf.preco_fornecedor,
       e.quantidade AS estoque_atual,
       (pf.preco_fornecedor IS NOT NULL AND e.quantidade < 10) AS estoque_baixo_flag
FROM produto pr
LEFT JOIN produto_fornecedor pf ON pf.produto_id = pr.produto_id
LEFT JOIN fornecedor f ON f.fornecedor_id = pf.fornecedor_id
LEFT JOIN estoque e ON e.produto_id = pr.produto_id
ORDER BY pr.nome;

-- 4) Relação de nomes dos fornecedores e nomes dos produtos (simples join)
-- Pergunta: "Listar fornecedor e produtos que ele fornece"
SELECT f.nome AS fornecedor, pr.nome AS produto
FROM fornecedor f
JOIN produto_fornecedor pf ON pf.fornecedor_id = f.fornecedor_id
JOIN produto pr ON pr.produto_id = pf.produto_id
ORDER BY f.nome, pr.nome;

-- 5) Produtos com estoque menor que 30 e ordenados pelo estoque (WHERE + ORDER BY)
SELECT pr.produto_id, pr.nome, e.quantidade
FROM produto pr
JOIN estoque e ON e.produto_id = pr.produto_id
WHERE e.quantidade < 30
ORDER BY e.quantidade ASC;

-- 6) Pedidos por cliente com valor médio por item (usando expressão derivada e HAVING)
-- Pergunta: "Quais clientes têm um ticket médio por pedido maior que R$400?"
SELECT c.cliente_id, c.email,
       AVG(p.valor_total) AS ticket_medio,
       COUNT(p.pedido_id) AS numero_pedidos
FROM cliente c
JOIN pedido p ON p.cliente_id = c.cliente_id
GROUP BY c.cliente_id, c.email
HAVING AVG(p.valor_total) > 400
ORDER BY ticket_medio DESC;

-- 7) Itens vendidos por produto (JOIN + aggregation)
-- Pergunta: "Quantas unidades de cada produto foram vendidas (considerando pedidos com status CONCLUIDO)?"
SELECT pr.produto_id, pr.nome, SUM(pi.quantidade) AS unidades_vendidas, SUM(pi.quantidade * pi.preco_unitario - pi.desconto) AS receita_bruta
FROM produto pr
JOIN pedido_item pi ON pi.produto_id = pr.produto_id
JOIN pedido p ON p.pedido_id = pi.pedido_id AND p.status_pedido = 'CONCLUIDO'
GROUP BY pr.produto_id, pr.nome
ORDER BY unidades_vendidas DESC;

-- 8) Entregas atrasadas: (expressão derivada com COALESCE e WHERE)
-- Pergunta: "Quais entregas ainda não entregues e cujo envio ocorreu há mais de 7 dias?"
SELECT e.entrega_id, e.pedido_id, e.status, e.codigo_rastreio, e.data_envio,
       now() - e.data_envio AS dias_desde_envio
FROM entrega e
WHERE e.data_entrega IS NULL
  AND e.data_envio IS NOT NULL
  AND now() - e.data_envio > INTERVAL '7 days'
ORDER BY e.data_envio;

-- 9) Clientes PF vs PJ: listar clientes e tipo
SELECT c.cliente_id, c.email,
       c.tipo,
       CASE WHEN c.tipo = 'PF' THEN (SELECT nome_completo FROM cliente_pf WHERE cliente_pf.cliente_id = c.cliente_id)
            ELSE (SELECT razao_social FROM cliente_pj WHERE cliente_pj.cliente_id = c.cliente_id)
       END AS nome_exibicao
FROM cliente c
ORDER BY c.cliente_id;

-- 10) Valor total faturado por fornecedor (via produto_fornecedor -> pedido_item)
-- Pergunta: "Qual o total faturado de produtos originados de cada fornecedor (usando preço unitário do pedido) ?"
SELECT f.fornecedor_id, f.nome AS fornecedor,
       SUM(pi.quantidade * pi.preco_unitario - pi.desconto) AS faturamento_total
FROM fornecedor f
JOIN produto_fornecedor pf ON pf.fornecedor_id = f.fornecedor_id
JOIN produto pr ON pr.produto_id = pf.produto_id
JOIN pedido_item pi ON pi.produto_id = pr.produto_id
JOIN pedido p ON p.pedido_id = pi.pedido_id AND p.status_pedido = 'CONCLUIDO'
GROUP BY f.fornecedor_id, f.nome
ORDER BY faturamento_total DESC;

-- 11) Pergunta avançada: "Quais clientes usaram mais de uma forma de pagamento?"
SELECT c.cliente_id, c.email, COUNT(DISTINCT cp.forma_pagamento_id) AS formas_pagamento_cadastradas
FROM cliente c
JOIN cliente_pagamento cp ON cp.cliente_id = c.cliente_id
GROUP BY c.cliente_id, c.email
HAVING COUNT(DISTINCT cp.forma_pagamento_id) > 1;

-- 12) Pergunta de auditoria: "Listar pedidos com discrepância entre soma dos itens e valor_total do pedido"
SELECT p.pedido_id, p.valor_total AS pedido_valor_total, SUM(pi.quantidade * pi.preco_unitario - pi.desconto) AS soma_itens
FROM pedido p
LEFT JOIN pedido_item pi ON pi.pedido_id = p.pedido_id
GROUP BY p.pedido_id, p.valor_total
HAVING COALESCE(SUM(pi.quantidade * pi.preco_unitario - pi.desconto),0) <> p.valor_total;

-- Fim das queries

-- Observação: Ajuste de permissões, índices e triggers adicionais podem ser aplicados conforme necessidade.

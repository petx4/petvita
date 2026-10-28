# PetVida – Banco de Dados de uma Clínica Veterinária



---

## 1. Domínio

Gestão de uma **clínica veterinária**: tutores, animais, veterinários, consultas e serviços cobrados.

## 2. Contextualização

A **PetVida** é uma clínica de bairro com 5 veterinários. Hoje usa fichas de papel e planilhas separadas
(agenda, clientes, caixa). Problemas:

- **Dados repetidos**: o telefone do tutor é copiado em toda ficha de consulta.
- **Inconsistência**: o tutor muda de telefone e as fichas antigas ficam com o número errado.
- **Difícil consultar**: saber "quanto faturamos em agosto" exige somar fichas à mão.
- **Sem validação**: dá para registrar consulta de um animal que nem foi cadastrado.
- **Duplicidade**: o mesmo tutor aparece com grafias diferentes ("Mariana S. Lima" e "Mariana Souza Lima").
- **Perda e acesso simultâneo**: papel se perde e duas pessoas editando a mesma planilha sobrescrevem uma à outra.

## 3. Proposta de melhoria

Usar um **banco de dados relacional (MySQL)**:

- cada informação é guardada **uma vez só** (o telefone fica apenas em `tutores`);
- **chaves estrangeiras** impedem consulta sem animal ou sem veterinário;
- `UNIQUE` evita CPF e CRMV duplicados;
- perguntas como faturamento e consultas por veterinário viram **consultas SQL** de poucos segundos;
- vários atendentes usam o sistema ao mesmo tempo (InnoDB);
- backup simples com `mysqldump`.

---

## 4. Modelagem lógica

![Modelagem lógica](docs/modelagem_logica.png)

### 4.1 Tabelas, campos e tipos

**tutores**: `id_tutor` INT (PK, auto) · `nome` VARCHAR(100) · `cpf` CHAR(11) (único) · `telefone` VARCHAR(15) · `cidade` VARCHAR(60)

**especies**: `id_especie` INT (PK, auto) · `nome` VARCHAR(40) (único)

**animais**: `id_animal` INT (PK, auto) · `id_tutor` INT (FK) · `id_especie` INT (FK) · `nome` VARCHAR(50) · `data_nascimento` DATE · `peso_kg` DECIMAL(5,2)

**veterinarios**: `id_veterinario` INT (PK, auto) · `nome` VARCHAR(100) · `crmv` VARCHAR(15) (único) · `especialidade` VARCHAR(50)

**servicos**: `id_servico` INT (PK, auto) · `descricao` VARCHAR(80) (único) · `valor` DECIMAL(10,2)

**consultas**: `id_consulta` INT (PK, auto) · `id_animal` INT (FK) · `id_veterinario` INT (FK) · `data_hora` DATETIME · `motivo` VARCHAR(100) · `status` ENUM('AGENDADA','REALIZADA','CANCELADA')

**consulta_servicos**: `id_consulta` INT (PK, FK) · `id_servico` INT (PK, FK) · `quantidade` INT · `valor_unitario` DECIMAL(10,2)

### 4.2 Relacionamentos e cardinalidade

| Relacionamento | Cardinalidade | Implementação |
|---|---|---|
| tutores → animais | **1 : N** (um tutor tem vários animais) | `animais.id_tutor` |
| especies → animais | **1 : N** | `animais.id_especie` |
| animais → consultas | **1 : N** | `consultas.id_animal` |
| veterinarios → consultas | **1 : N** | `consultas.id_veterinario` |
| consultas ↔ servicos | **N : N** (uma consulta tem vários serviços e um serviço aparece em várias consultas) | tabela `consulta_servicos` |

> Neste modelo simplificado não há relacionamento 1:1. Um exemplo possível seria separar o microchip do animal
> em uma tabela própria com `id_animal` único.

---

## 5. Formas Normais

Ponto de partida, a **ficha de papel** atual:

```
FICHA (id_consulta, data, tutor_nome, tutor_cpf, tutor_telefones,
       animal_nome, animal_especie, vet_nome, vet_crmv, servicos_realizados)
```

**1FN – valores atômicos, sem listas em um campo.**
Na ficha, `servicos_realizados` era uma lista ("Consulta, Hemograma, Radiografia"), o que impede somar valores ou
filtrar por serviço. Cada serviço da consulta virou **uma linha** em `consulta_servicos`.

**2FN – nenhum atributo depende de só parte de uma chave composta.**
Em `consulta_servicos (id_consulta, id_servico, ...)`, a descrição do serviço depende só de `id_servico`.
Ela foi para a tabela `servicos`. Em `consulta_servicos` ficaram apenas `quantidade` e `valor_unitario`,
que dependem da chave inteira. (`valor_unitario` guarda o preço cobrado na época, então mudar o preço de tabela
não altera o faturamento passado.)

**3FN – nenhum atributo depende de outro atributo que não seja chave.**
`animal_especie`, dados do tutor e dados do veterinário se repetiam em cada ficha.
Foram separados em `especies`, `tutores` e `veterinarios` e ligados por chaves estrangeiras.
O **total da consulta não é gravado**: é calculado com `SUM(quantidade * valor_unitario)`,
evitando um valor que poderia ficar desatualizado.

**4FN – nenhuma tabela mistura dois fatos multivalorados independentes.**
Um tutor pode ter vários telefones e vários e-mails (fatos independentes entre si). Se ficassem juntos numa tabela
`(tutor, telefone, email)`, seria preciso gravar todas as combinações. Neste modelo cada tutor tem um telefone e
cada veterinário uma especialidade, então não há esse problema. Se a clínica quiser vários telefones e e-mails,
o correto é criar **duas tabelas separadas** (`tutor_telefones` e `tutor_emails`), nunca uma só.

---

## 6. Instruções SQL

| Arquivo | Conteúdo |
|---|---|
| `01_criar_banco_e_tabelas.sql` | `CREATE DATABASE petvida` e as 7 tabelas |
| `02_popular_tabelas.sql` | `INSERT` em todas as tabelas |
| `03_consultas.sql` | As 15 consultas |

```bash
mysql -u root -p < scripts/01_criar_banco_e_tabelas.sql
mysql -u root -p < scripts/02_popular_tabelas.sql
mysql -u root -p < scripts/03_consultas.sql
```

Registros: tutores 6 · especies 5 · animais 8 · veterinarios 5 · servicos 6 · consultas 10 · consulta_servicos 15 (mínimo exigido: 5).

### As 15 consultas, com respostas

**1) INNER JOIN + ORDER BY.** *A recepção quer a lista de animais com o tutor e o telefone.*

| tutor | telefone | animal |
|---|---|---|
| Ana Paula Ribeiro | (11) 96633-4455 | Bob |
| Ana Paula Ribeiro | (11) 96633-4455 | Nina |
| Carlos Eduardo Ferreira | (11) 97722-3344 | Mel |
| Fernanda Costa Martins | (11) 94455-6677 | Pipoca |
| João Pedro Almeida | (11) 95544-5566 | Fred |
| Mariana Souza Lima | (11) 98811-2233 | Luna |
| Mariana Souza Lima | (11) 98811-2233 | Thor |
| Roberto Nascimento | (11) 93366-7788 | Sol |

**2) WHERE + LIKE.** *Campanha de vacinação para tutores de cidades que começam com "São".*

| nome | telefone | cidade |
|---|---|---|
| Carlos Eduardo Ferreira | (11) 97722-3344 | São Paulo |
| Mariana Souza Lima | (11) 98811-2233 | São Paulo |
| Roberto Nascimento | (11) 93366-7788 | São Paulo |

**3) INNER JOIN (3 tabelas) + WHERE + BETWEEN.** *Consultas realizadas entre 01/08/2026 e 30/09/2026.*

| id_consulta | data_hora | animal | veterinario |
|---|---|---|---|
| 4 | 2026-08-04 14:00:00 | Bob | Dr. Rafael Tavares |
| 5 | 2026-08-12 09:00:00 | Fred | Dr. Henrique Matos |
| 6 | 2026-08-14 15:00:00 | Nina | Dra. Beatriz Camargo |
| 7 | 2026-09-10 10:00:00 | Pipoca | Dra. Beatriz Camargo |

**4) LEFT JOIN + GROUP BY + COUNT.** *Quantos animais por espécie (mesmo as sem nenhum).*

| especie | total_animais |
|---|---|
| Cão | 4 |
| Gato | 2 |
| Ave | 1 |
| Coelho | 1 |
| Roedor | 0 |

**5) GROUP BY + AVG + MIN + MAX.** *Peso médio, mínimo e máximo por espécie, para conferir dosagens.*

| especie | peso_medio | peso_minimo | peso_maximo |
|---|---|---|---|
| Cão | 15.55 | 6.80 | 32.50 |
| Gato | 3.90 | 3.60 | 4.20 |
| Coelho | 1.90 | 1.90 | 1.90 |
| Ave | 0.09 | 0.09 | 0.09 |

**6) GROUP BY + HAVING.** *Tutores com mais de um animal (desconto familiar).*

| tutor | qtd_animais |
|---|---|
| Ana Paula Ribeiro | 2 |
| Mariana Souza Lima | 2 |

**7) SUM + GROUP BY + ORDER BY.** *Valor total cobrado em cada consulta.*

| id_consulta | animal | total |
|---|---|---|
| 5 | Fred | 580.00 |
| 3 | Mel | 400.00 |
| 8 | Thor | 400.00 |
| 4 | Bob | 305.00 |
| 1 | Thor | 245.00 |
| 2 | Luna | 235.00 |
| 6 | Nina | 235.00 |
| 7 | Pipoca | 150.00 |

**8) SUM + GROUP BY (mês).** *Faturamento mensal.*

| mes | faturamento |
|---|---|
| 2026-07 | 880.00 |
| 2026-08 | 1120.00 |
| 2026-09 | 150.00 |
| 2026-10 | 400.00 |

**9) SUM + GROUP BY (veterinário).** *Faturamento gerado por cada veterinário.*

| veterinario | faturamento |
|---|---|
| Dr. Henrique Matos | 980.00 |
| Dra. Beatriz Camargo | 865.00 |
| Dra. Camila Duarte | 400.00 |
| Dr. Rafael Tavares | 305.00 |

**10) LEFT JOIN + COUNT.** *Consultas por veterinário (inclui quem não atendeu ninguém).*

| veterinario | especialidade | total_consultas |
|---|---|---|
| Dra. Beatriz Camargo | Clínica Geral | 4 |
| Dr. Henrique Matos | Ortopedia | 2 |
| Dr. Rafael Tavares | Dermatologia | 2 |
| Dra. Camila Duarte | Cardiologia | 2 |
| Dra. Letícia Barros | Animais Exóticos | 0 |

**11) SUM + GROUP BY + ORDER BY + LIMIT.** *Os 3 serviços que mais faturaram.*

| servico | faturamento |
|---|---|
| Consulta especializada | 880.00 |
| Radiografia | 720.00 |
| Consulta clínica geral | 600.00 |

**12) LEFT JOIN + IS NULL.** *Serviços que nunca foram realizados.*

| servico | valor |
|---|---|
| Banho e tosa higiênica | 80.00 |

**13) GROUP BY + COUNT.** *Consultas por situação.*

| status | total |
|---|---|
| REALIZADA | 8 |
| AGENDADA | 1 |
| CANCELADA | 1 |

**14) LEFT JOIN + IS NULL.** *Animais que nunca tiveram consulta (a clínica pode ligar para o tutor).*

| animal | tutor | telefone |
|---|---|---|
| Sol | Roberto Nascimento | (11) 93366-7788 |

**15) Subconsulta + HAVING + SUM.** *Consultas com valor acima da média (R$ 318,75).*

| id_consulta | animal | total |
|---|---|---|
| 5 | Fred | 580.00 |
| 3 | Mel | 400.00 |
| 8 | Thor | 400.00 |

---

## 7. Passo a passo do dump (estrutura e dados)

### Pelo terminal

1. Abra o terminal (no Windows, o Prompt de Comando; se o `mysqldump` não for reconhecido, use o caminho completo, por exemplo `"C:\Program Files\MySQL\MySQL Server 8.0\bin\mysqldump.exe"`).
2. Rode o comando e digite a senha quando pedir:

   ```bash
   mysqldump -u root -p --databases petvida --routines --triggers --single-transaction --result-file=dump_petvida.sql
   ```

   - `--databases petvida` inclui `CREATE DATABASE` e `USE` no arquivo;
   - `--single-transaction` gera um dump consistente sem travar as tabelas;
   - `--result-file` grava em UTF-8 (no PowerShell, evite `>` porque ele grava em UTF-16 e quebra a importação).
3. Só a estrutura: acrescente `--no-data`. Só os dados: acrescente `--no-create-info` (e use `petvida` no lugar de `--databases petvida`).
4. Abra o arquivo e confira se há `CREATE TABLE` e `INSERT INTO`.

### Pelo MySQL Workbench

1. **Server → Data Export**.
2. Marque o schema `petvida` e todas as tabelas.
3. Escolha **Export to Self-Contained File**, informe o caminho do `dump_petvida.sql`.
4. Selecione **Dump Structure and Data** e marque **Include Create Schema**.
5. Clique em **Start Export**.

### Restaurar (para testar)

```bash
mysql -u root -p < dump_petvida.sql
```

No Workbench: **Server → Data Import → Import from Self-Contained File → Start Import**.
Se os acentos aparecerem errados, acrescente `--default-character-set=utf8mb4` ao comando.
Conferência: `USE petvida; SELECT COUNT(*) FROM consultas;` deve retornar 10.

> O `scripts/dump_petvida.sql` deste repositório foi gerado com o `mysqldump` do MySQL 8.0.46 e restaurado com sucesso.

---

## 8. Commit dos scripts

```bash
cd petvida_mysql
git init
git add README.md docs/ scripts/
git commit -m "Banco de dados PetVida: modelagem, scripts SQL, consultas e dump"
git branch -M main
git remote add origin https://github.com/SEU-USUARIO/petvida.git
git push -u origin main
```

Antes do `push`, crie um repositório vazio no GitHub (**New repository**, sem README) e use a URL dele no `remote add`.

## 9. Entrega no Classroom

- **9.1 Link do repositório** com os scripts (URL da página principal no GitHub; deixe público ou dê acesso ao professor).
- **9.2 Print da modelagem**: use `docs/modelagem_logica.png` ou gere pelo Workbench em **Database → Reverse Engineer…**, escolhendo o schema `petvida`, e tire o print do diagrama.

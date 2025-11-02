# TP2-TADB

**Descrição:**

Este projeto tem como objetivo a prática de mapeamento de dados XML para relacional.

Considerando o documento SIGMOD Record disponível no XML Data Repository da Universidade de Washington
 
(http://www.cs.washington.edu/research/xmldatasets/)

a) proponha 10 consutas XPath que explorem diferentes eixos de caminhamento e as execute utilizando algum processador de consultas XML. Seja criativo e proponha consultas que permitam o uso de diferentes construções da linguagem.

b) crie um banco de dados relacional que modele esse seu documento XML e popule-o com os registros XML.

c) apresente as 10 consultas propostas no item ‘a’ em SQL
 
**Entregas:**
 
1- 10 (dez) Consultas XPath
 
2- Modelo Conceitual do Banco de Dados
 
3- Modelo Lógico Relacional do Banco de Dados
 
4- Funções de mapeamento (povoamento) do Banco de Dados
 
5- Consultas SQL equivalentes à XPath proposta no item 1

---

## 1. Consultas XPath

1. Busca todos os artigos.
```xpath
/sigmodrecord/issue/articles
```

2. descrição
```xpath
código
```

3. descrição
```xpath
código
```

## 2. Modelo Conceitual do Banco de Dados (MER)

![imagem](./mer.png)

## 3. Modelo Lógico Relacional do Banco de Dados

Modelo lógico em linha

- ISSUE ( <ins>volume</ins> , <ins> number</ins> )

- ARTICLE ( <ins>title</ins> , <ins>volume_issue</ins> , <ins>number_issue</ins> , initPage , endPage )

- AUTHOR ( <ins>name</ins> )

- AUTHORSHIP ( <ins> article_title</ins> , <ins>article_volume</ins> , <ins> article_number</ins> , <ins> author_name</ins> , <ins> position</ins> )

### 3.1 Modelo Físico

```sql
CREATE TABLE Issue (
    volume INTEGER NOT NULL,
    number INTEGER NOT NULL,
    
    CONSTRAINT pk_issue PRIMARY KEY (volume, number),
    CONSTRAINT chk_issue_volume_01 CHECK (volume > 0),
    CONSTRAINT chk_issue_number_01 CHECK (number > 0)
);

CREATE TABLE Article (
    title VARCHAR(500) NOT NULL,
    initPage INTEGER NOT NULL,
    endPage INTEGER NOT NULL,
    volume_issue INTEGER NOT NULL,
    number_issue INTEGER NOT NULL,
    
    CONSTRAINT pk_article PRIMARY KEY (title, volume_issue, number_issue),
    CONSTRAINT fk_article_volume_issue FOREIGN KEY (volume_issue, number_issue) 
        REFERENCES Issue(volume, number),
    CONSTRAINT chk_article_pages_01 CHECK (initPage >= 0 AND endPage >= 0 AND endPage >= initPage)
);

CREATE TABLE Author (
    name VARCHAR(200) NOT NULL,
    
    CONSTRAINT pk_author PRIMARY KEY (name)
);

CREATE TABLE Authorship (
    article_title VARCHAR(500) NOT NULL,
    article_volume INTEGER NOT NULL,
    article_number INTEGER NOT NULL,
    author_name VARCHAR(200) NOT NULL,
    position CHAR(2) NOT NULL,
    
    CONSTRAINT pk_authorship PRIMARY KEY (article_title, article_volume, article_number, author_name),
    CONSTRAINT fk_authorship_article FOREIGN KEY (article_title, article_volume, article_number) 
        REFERENCES Article(title, volume_issue, number_issue),
    CONSTRAINT fk_authorship_author FOREIGN KEY (author_name) 
        REFERENCES Author(name),
    CONSTRAINT chk_authorship_position_01 CHECK (position ~ '^[0-9]{2}$')
);
```

## 4. Funções de mapeamento (povoamento) do Banco de Dados

- Função para inserir Article
```sql

CREATE OR REPLACE FUNCTION inserir_articles()
RETURNS integer AS $$
DECLARE
    v_rows_inserted INTEGER := 0;
BEGIN
    INSERT INTO Article (title, initPage, endPage, volume_issue, number_issue)
    SELECT DISTINCT
        trim(t.title) AS title,
        (regexp_replace(trim(t.initPage), '\D', '', 'g'))::int AS initpage,
        (regexp_replace(trim(t.endPage),   '\D', '', 'g'))::int AS endpage,
        (regexp_replace(trim(t.vol),       '\D', '', 'g'))::int AS volume_issue,
        (regexp_replace(trim(t.num),       '\D', '', 'g'))::int AS number_issue
    FROM sigmod_record,
         XMLTABLE(
           '//issue/articles/article'   -- caminho corrigido
           PASSING documento
           COLUMNS
             title    TEXT PATH 'string(title)',
             initPage TEXT PATH 'string(initPage)',
             endPage  TEXT PATH 'string(endPage)',
             vol      TEXT PATH 'string(../../volume)',  -- sobe dois níveis
             num      TEXT PATH 'string(../../number)'
         ) AS t
    WHERE t.title IS NOT NULL
      AND trim(t.title) <> ''
      AND t.initPage IS NOT NULL
      AND t.endPage IS NOT NULL
      AND regexp_replace(t.initPage, '\D', '', 'g') ~ '^[0-9]+$'
      AND regexp_replace(t.endPage,  '\D', '', 'g') ~ '^[0-9]+$'
      AND t.vol IS NOT NULL
      AND t.num IS NOT NULL
      AND regexp_replace(t.vol, '\D', '', 'g') ~ '^[0-9]+$'
      AND regexp_replace(t.num, '\D', '', 'g') ~ '^[0-9]+$'
      AND (regexp_replace(t.endPage, '\D', '', 'g'))::int >= (regexp_replace(t.initPage, '\D', '', 'g'))::int
    ON CONFLICT DO NOTHING;

    GET DIAGNOSTICS v_rows_inserted = ROW_COUNT;
    RETURN v_rows_inserted;
END;
$$ LANGUAGE plpgsql;
```

- Função para inserir Issue;
```sql
CREATE OR REPLACE FUNCTION inserir_issues()
RETURNS void AS $$
BEGIN
    INSERT INTO Issue (volume, number)
    SELECT DISTINCT
        x.vol::int AS volume,
        x.num::int AS number
    FROM sigmod_record,
    XMLTABLE(
        '//issue'
        PASSING documento
        COLUMNS
            vol TEXT PATH 'string(@volume | volume)',
            num TEXT PATH 'string(@number | number)'
    ) AS x
    WHERE x.vol IS NOT NULL
      AND x.num IS NOT NULL
    ON CONFLICT DO NOTHING;
END;
$$ LANGUAGE plpgsql;
```

- Função para inserir Author
```sql
CREATE OR REPLACE FUNCTION inserir_authors()
RETURNS integer AS $$
DECLARE
    v_rows_inserted INTEGER := 0;
BEGIN
    INSERT INTO Author (name)
    SELECT DISTINCT trim(t.author_name)
    FROM sigmod_record,
         XMLTABLE(
           '//issue/articles/article/authors/author'
           PASSING documento
           COLUMNS
             author_name TEXT PATH 'string(.)'
         ) AS t
    WHERE t.author_name IS NOT NULL
      AND trim(t.author_name) <> ''
    ON CONFLICT DO NOTHING;

    GET DIAGNOSTICS v_rows_inserted = ROW_COUNT;
    RETURN v_rows_inserted;
END;
$$ LANGUAGE plpgsql;

```

- Função para inserir  Authorship
```sql
CREATE OR REPLACE FUNCTION inserir_authorships()
RETURNS integer AS $$
DECLARE
    v_rows_inserted INTEGER := 0;
BEGIN
    INSERT INTO Authorship (
        article_title,
        article_volume,
        article_number,
        author_name,
        position
    )
    SELECT DISTINCT
        trim(t.title) AS article_title,
        (trim(t.vol))::int AS article_volume,
        (trim(t.num))::int AS article_number,
        trim(t.author_name) AS author_name,
        lpad(trim(t.position), 2, '0') AS position
    FROM sigmod_record,
         XMLTABLE(
           '//issue/articles/article/authors/author'
           PASSING documento
           COLUMNS
             author_name TEXT PATH 'string(.)',
             position    TEXT PATH 'string(@position)',
             title       TEXT PATH 'string(../../title)',
             vol         TEXT PATH 'string(../../../../volume)',
             num         TEXT PATH 'string(../../../../number)'
         ) AS t
    WHERE t.author_name IS NOT NULL
      AND trim(t.author_name) <> ''
      AND t.title IS NOT NULL
      AND trim(t.title) <> ''
      AND t.vol ~ '^[0-9]+$'
      AND t.num ~ '^[0-9]+$'
      AND trim(t.position) ~ '^[0-9]{1,2}$'
    ON CONFLICT DO NOTHING;

    GET DIAGNOSTICS v_rows_inserted = ROW_COUNT;
    RETURN v_rows_inserted;
END;
$$ LANGUAGE plpgsql;

```

- Função para inserir o documento SigmodRecord
```sql
CREATE OR REPLACE FUNCTION sigmod_record()
RETURNS void AS $$
BEGIN
    RAISE NOTICE 'Iniciando inserção completa dos dados...';

    -- 1️⃣ Inserir Issues
    PERFORM inserir_issues();
    RAISE NOTICE 'Issues inseridos.';

    -- 2️⃣ Inserir Articles
    PERFORM inserir_articles();
    RAISE NOTICE 'Articles inseridos.';

    -- 3️⃣ Inserir Authors
    PERFORM inserir_authors();
    RAISE NOTICE 'Authors inseridos.';

    -- 4️⃣ Inserir Authorships
    PERFORM inserir_authorships();
    RAISE NOTICE 'Authorships inseridos.';

    RAISE NOTICE '✅ Inserção completa finalizada com sucesso.';
END;
$$ LANGUAGE plpgsql;

```

## 5. Consultas SQL equivalentes à XPath proposta no item 1
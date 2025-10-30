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

- AUTHOR ( <ins>nome</ins> )

- AUTHORSHIP ( <ins> article_title</ins> , <ins>article_volume</ins> , <ins> article_number</ins> , <ins> author_nome</ins> , <ins> position</ins> )

### 3.1 Modelo Físico

## 4. Funções de mapeamento (povoamento) do Banco de Dados

## 5. Consultas SQL equivalentes à XPath proposta no item 1
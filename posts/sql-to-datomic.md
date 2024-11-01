Title: SQL to Datalog
Date: 2024-11-01
Tags: clojure, datomic, sql, databases

<!-- Goal is to introduce datalog and tuple databases to people who understand RDBMS -->
## Introduction

## Databases
A good practice to follow, when trying to learn a new technology is to ask why it exists. In our context, we should ask why databases exist and why are they structured the way they are.

Every computing machine needs a way to store and retrieve data. Even our brain does. And we want our machine to be able to do it fast. After trying a lot of different ideas, computer scientists came up with one of storing data as collection of records. They also found this way of data storage to have several mathematical properties. These mathematical properties allowed for storage optimisations and efficient information retrieval. They called it, the relational database management system.

## Relational Database Management System (RDBMS)
Relational databases are the most widely used and understood storage systems, today. They are efficient for most purposes and easy to make sense of, for a beginner. They store data in tables and each table is built on top of columns and rows. Each row contains related information and is usually called a `record`.

Example of a table in a relational database. This table stores information about students.
```sql
                   column or field (gender values for everyone)
                      ↓
| id | name  | age | gender |
|----|-------|-----|--------|
| 1  | Ram   | 14  | M      |  <-- row or record (information about Ram)
| 2  | Sita  | 19  | F      |
| 3  | Shyam | 25  | M      |
```
> Notice how a `row` only keeps related information and `column` has a particular kind of information.

<br>
Relational databases usually offer a query language that allows us to interact with the storage system. The query language has a [declarative syntax](https://stackoverflow.com/q/1784664/5913204) again an awesome feature. Declarative systax allows us to tell the storage system what we want and it returns that data, without us worrying much about it. The langauge is called **Structured Query Language (SQL)**.

Suppose we want to find all the male students. We will tell our database exactly that.

```sql
SELECT ALL FROM students WHERE gender = 'M';

# Give me everything about the students where their gender is male.

| id | name  | age | gender |
|----|-------|-----|--------|
| 1  | Ram   | 14  | M      |
| 3  | Shyam | 25  | M      |
```

## Datalog
Now instead of thinking about the data in terms of collection of records. Let's visualize it as graphs.

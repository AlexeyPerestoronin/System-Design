# Классификация и выбор СУБД

![](./data%20bases.drawio.png)

# Пояснения

**ACID** - набор требований к транзакционной системе работы с данными (СУБД):
* Atomicity
* Consistency
* Isolation
* Durability

---

**CAP** - акроним теоремы Брюера - теорема о том, что в любой реализации распределённых вычислений (распределённой системы обработки данных РСОД) возможно обеспечить не более двух из трех следующих свойств:
* Consistency - когда данные во всех частях РСОД не противоречат друг другу
* Availabililty - когда запрос на доступ к данным РСОД и меет мгновенный отклик
* Partition Tolerance - расщепление РСОД не приводит к некорректности отклика

# Известные СУБД
1. SQL
2. NoSQL
3. Специальные:
   1. Search Data Bases:
      1. Search DB
   2. Time Series Data Bases:
      1. Influx
   3. [Hadoop](./known%20DBMS/Hadoop.md)
   4. [Amazon S3](./known%20DBMS/Amazon%20S3.md)
   5. Azure Blob
   6. Google Cloud Storage
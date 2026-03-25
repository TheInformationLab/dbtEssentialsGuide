# Testen & Debugging

1. Erstelle einen neuen Ordner im Models-Verzeichnis mit dem Namen library_loans ("/models/library_loans")

2. Erstelle im library_loans-Ordner eine neue Datei namens customers_with_late_fees.sql ("/models/library_loans/customers_with_late_fees.sql") und kopiere die SQL-Anweisungen aus der Starter-Datei customers_with_late_fees.sql hinein.

   ```sql
   WITH CTE AS (
       SELECT
           COALESCE(FAC.book_name, FIC.book_name) as book_name,
           M.member_name,
           M.discount_rate/100 as discount_applied,
           SUM(L.late_fee * (M.discount_rate/100)) as fee_applied
       FROM dbt_course.library_loans.members AS M
           INNER JOIN dbt_course.library_loans.loans AS L ON M.member_id = L.member_id
           LEFT JOIN dbt_course.library_loans.books_factual AS FAC ON FAC.book_id=L.book_id
           LEFT JOIN dbt_course.library_loans.books_fictional AS FIC ON FIC.book_id=L.book_id
       GROUP BY 1,2,3
   )
   SELECT
   member_name,
   listagg(book_name::text, ',') as late_books,
   discount_applied,
   ROUND(SUM(fee_applied),2) as fee_to_pay
   FROM CTE
   GROUP BY 1,3
   ```

3. Erstelle eine \_schema.yml-Datei im library_loans-Verzeichnis ("/models/library_loans/\_schema.yml") und kopiere die YAML-Anweisungen aus der Starter-Datei \_schema.yml hinein

   ```yml
   version: 2

   sources:
     - name: library
       database: dbt_course
       schema: library_loans
       tables:
         - name: books_factual
         - name: books_fictional
         - name: loans
         - name: members
   ```

   Dies erstellt die Quellen, die für dbt referenzieren kann

4. Aktualisiere den Models-Abschnitt der dbt_project.yml-Datei (Zeilen 38-45), damit alles in library_loans standardmäßig als Tabelle materialisiert wird

   ```yml
   models:
     my_new_project:
       # Applies to all files under models/example/
       example:
         +materialized: table

       lego:
         +materialized: table

       library_loans:
         +materialized: table
   ```

5. Aktualisiere customers_with_late_fees.sql, um unsere definierten Quellen zu verwenden, anstelle von hartcodierten Werten

   ```sql
   WITH CTE AS (
       SELECT
           COALESCE(FAC.book_name, FIC.book_name) as book_name,
           M.member_name,
           M.discount_rate/100 as discount_applied,
           SUM(L.late_fee * (M.discount_rate/100)) as fee_applied
       FROM {{ source('library', 'members') }} AS M
           INNER JOIN {{ source('library', 'loans') }} AS L ON M.member_id = L.member_id
           LEFT JOIN {{ source('library', 'books_factual') }} AS FAC ON FAC.book_id=L.book_id
           LEFT JOIN {{ source('library', 'books_fictional') }} AS FIC ON FIC.book_id=L.book_id
       GROUP BY 1,2,3
   )
   SELECT
   member_name,
   listagg(book_name::text, ',') as late_books,
   discount_applied,
   ROUND(SUM(fee_applied),2) as fee_to_pay
   FROM CTE
   GROUP BY 1,3
   ```

6. Füge Primärschlüsseltests zu unseren Primärschlüsseln hinzu:

- Unique und Not-Null-Tests
  - books(fictional/factual).Book_id
  - members.member_id
  - loans.loan_id

```yml
version: 2

sources:
  - name: library
    database: dbt_course
    schema: library_loans
    tables:
      - name: books_factual
        columns:
          - name: book_id
            data_tests:
              - unique
              - not_null
      - name: books_fictional
        columns:
          - name: book_id
            data_tests:
              - unique
              - not_null
      - name: loans
        columns:
          - name: loan_id
            data_tests:
              - unique
              - not_null
      - name: members
        columns:
          - name: member_id
            data_tests:
              - unique
              - not_null
```

7. Stelle sicher, dass wir nur 'Gold', 'Silver' und 'Bronze' Mitgliedschaftsstufen haben

```yml
version: 2

sources:
  - name: library
    database: dbt_course
    schema: library_loans
    tables:
      - name: books_factual
        columns:
          - name: book_id
            data_tests:
              - unique
              - not_null
      - name: books_fictional
        columns:
          - name: book_id
            data_tests:
              - unique
              - not_null
      - name: loans
        columns:
          - name: loan_id
            data_tests:
              - unique
              - not_null
      - name: members
        columns:
          - name: member_id
            data_tests:
              - unique
              - not_null
          - name: membership_tier
            data_tests:
              - accepted_values:
                  arguments:
                    values: ["Bronze", "Silver", "Gold"]
```

8. Führe in der Befehlszeile folgenden Befehl aus

   ```sh
   dbt test --select library_loans
   ```

   7 sollten durchlaufen werden, 2 sollten fehlschlagen

9. Erstelle ein neues Modell in library_loans mit dem Namen stg_members.sql ("/models/library_loans/stg_members.sql")

10. Kopiere die SQL-Anweisungen aus der Starter-Datei stg_members.sql

    ```sql
    SELECT *
    FROM {{ source("library", "members") }}
    WHERE member_id IS NOT NULL
    AND membership_tier IN ('Bronze', 'Silver', 'Gold')
    ```

    Dies erstellt ein Staging-Modell, das unsere Bedingungen für eingehende Mitgliederdaten einhält

11. In der library_loans \_schema.yml: Schneide Mitglieder-Tests (Zeilen 26-33) aus und wende sie auf das neue stg_members-Modell in einem models-Block an

    ```yml
    version: 2

    sources:
      - name: library
        database: dbt_course
        schema: library_loans
        tables:
          - name: books_factual
            columns:
              - name: book_id
                data_tests:
                  - unique
                  - not_null
          - name: books_fictional
            columns:
              - name: book_id
                data_tests:
                  - unique
                  - not_null
          - name: loans
            columns:
              - name: loan_id
                data_tests:
                  - unique
                  - not_null
          - name: members

    models:
      - name: stg_members
        columns:
          - name: member_id
            data_tests:
              - unique
              - not_null
          - name: membership_tier
            data_tests:
              - accepted_values:
                  arguments:
                    values: ["Bronze", "Silver", "Gold"]
    ```

12. Führe in der Befehlszeile folgenden Befehl aus

    ```sh
    dbt build --select library_loans
    ```

    Der accepted-values-Test sollte nun bestanden werden, aber der unique fictional book_id-Test sollte immer noch fehlschlagen

13. Erstelle ein neues Modell in library_loans mit dem Namen stg_books.sql ("/models/library_loans/stg_books.sql")

14. Kopiere die SQL-Anweisungen aus der Starter-Datei stg_books.sql

    ```sql
    SELECT
    book_id,
    location,
    book_name,
    'Fact' as genre
    FROM {{ source("library", "books_factual") }}
    WHERE book_id IS NOT NULL
    UNION
    SELECT
    book_id,
    location,
    book_name,
    'Fiction' as genre
    FROM {{ source("library", "books_fictional") }}
    WHERE book_id IS NOT NULL
    ```

    Dies vereinigt fiktive und sachliche Bücher in einer Tabelle, während alle Null-Werte für book_ids entfernt werden

15. In der library_loans \_schema.yml entferne die book_id-Tests (Zeilen 9-13 und 15-19) und wende einen einzigen book_id not_null und unique-Test auf das neue stg_books-Modell im models-Abschnitt an.

    ```yml
    version: 2

    sources:
      - name: library
        database: dbt_course
        schema: library_loans
        tables:
          - name: books_factual
          - name: books_fictional
          - name: loans
            columns:
              - name: loan_id
                data_tests:
                  - unique
                  - not_null
          - name: members

    models:
      - name: stg_members
        columns:
          - name: member_id
            data_tests:
              - unique
              - not_null
          - name: membership_tier
            data_tests:
              - accepted_values:
                  arguments:
                    values: ["Bronze", "Silver", "Gold"]
      - name: stg_books
        columns:
          - name: book_id
            data_tests:
              - unique
              - not_null
    ```

16. Führe in der Befehlszeile folgenden Befehl aus

    ```sh
    dbt build --select library_loans
    ```

    Alle Tests sollten nun bestanden werden

17. Füge Beziehungstests zu unserer loans-Quelle hinzu:
    - loans.book_id
      - hat eine Beziehung zu stg_books.book_id
    - loans.member_id
      - hat eine Beziehung zu stg_members.member_id

    ```yml
    version: 2

    sources:
      - name: library
        database: dbt_course
        schema: library_loans
        tables:
          - name: books_factual
          - name: books_fictional
          - name: loans
            columns:
              - name: loan_id
                data_tests:
                  - unique
                  - not_null
              - name: book_id
                data_tests:
                  - relationships:
                      arguments:
                        to: ref('stg_books')
                        field: book_id
              - name: member_id
                data_tests:
                  - relationships:
                      arguments:
                        to: ref('stg_members')
                        field: member_id
          - name: members

    models:
      - name: stg_members
        columns:
          - name: member_id
            data_tests:
              - unique
              - not_null
          - name: membership_tier
            data_tests:
              - accepted_values:
                  arguments:
                    values: ["Bronze", "Silver", "Gold"]
      - name: stg_books
        columns:
          - name: book_id
            data_tests:
              - unique
              - not_null
    ```

18. Führe in der Befehlszeile folgenden Befehl aus

    ```sh
    dbt test --select library_loans
    ```

    Einige Tests werden fehlschlagen

19. Erstelle ein neues Modell in library_loans mit dem Namen stg_loans.sql ("/models/library_loans/stg_loans.sql")

20. Kopiere die SQL-Anweisungen aus der Starter-Datei stg_loans.sql

    ```sql
    SELECT *
    FROM {{ source("library", "loans") }}
    WHERE loan_id IS NOT NULL
    AND book_id IS NOT NULL
    AND member_id IS NOT NULL
    AND book_id IN (SELECT book_id FROM {{ ref('stg_books') }})
    AND member_id IN (SELECT member_id FROM {{ ref('stg_members') }})
    ```

    Dies wendet unsere Not-Null-Tests an, überprüft auch die referenzielle Integrität mit unseren Fremdschlüsseln (book_id und member_id)

21. Aktualisiere die \_schema.yml-Datei im library_loans-Ordner, um alle source-loans-data_tests zu entfernen und sie auf das neue stg_loans-Modell anzuwenden

    ```yml
    version: 2

    sources:
      - name: library
        database: dbt_course
        schema: library_loans
        tables:
          - name: books_factual
          - name: books_fictional
          - name: loans
          - name: members

    models:
      - name: stg_members
        columns:
          - name: member_id
            data_tests:
              - unique
              - not_null
          - name: membership_tier
            data_tests:
              - accepted_values:
                  arguments:
                    values: ["Bronze", "Silver", "Gold"]
      - name: stg_books
        columns:
          - name: book_id
            data_tests:
              - unique
              - not_null
      - name: stg_loans
        columns:
          - name: loan_id
            data_tests:
              - unique
              - not_null
          - name: book_id
            data_tests:
              - relationships:
                  arguments:
                    to: ref('stg_books')
                    field: book_id
          - name: member_id
            data_tests:
              - relationships:
                  arguments:
                    to: ref('stg_members')
                    field: member_id
    ```

22. Aktualisiere die customers_with_late_fees.sql-Datei, um auf unsere neuen Staging-Tabellen zu verweisen, anstelle der ursprünglichen Quellen unter Verwendung der ref()-Funktion. Du musst auch die SQL aktualisieren, da es jetzt nur noch 1 stg_books-Tabelle im Vergleich zu den 2 Book-Quellen gibt.

    ```sql
    WITH CTE AS (
        SELECT
            B.book_name as book_name,
            M.member_name,
            M.discount_rate/100 as discount_applied,
            SUM(L.late_fee * (M.discount_rate/100)) as fee_applied
        FROM {{ ref('stg_members') }} AS M
            INNER JOIN {{ ref('stg_loans') }} AS L ON M.member_id = L.member_id
            LEFT JOIN {{ ref('stg_books') }} AS B ON B.book_id=L.book_id
        GROUP BY 1,2,3
    )
    SELECT
    member_name,
    listagg(book_name::text, ',') as late_books,
    discount_applied,
    ROUND(SUM(fee_applied),2) as fee_to_pay
    FROM CTE
    GROUP BY 1,3
    ```

23. Erstelle eine neue Datei namens customer_withdrawals.sql im library_loans-Ordner ("/models/library_loans/customer_withdrawals.sql").

24. Kopiere die CTE-Abfrage aus customers_with_late_fees.sql in customer_withdrawals.sql (Zeilen 2-10)

    ```sql
    SELECT
        B.book_name as book_name,
        M.member_name,
        M.discount_rate/100 as discount_applied,
        SUM(L.late_fee * (M.discount_rate/100)) as fee_applied
    FROM {{ ref('stg_members') }} AS M
        INNER JOIN {{ ref('stg_loans') }} AS L ON M.member_id = L.member_id
        LEFT JOIN {{ ref('stg_books') }} AS B ON B.book_id=L.book_id
    GROUP BY 1,2,3
    ```

25. Aktualisiere die CTE in customer_with_late_fees.sql (Zeilen 1-11) unter Verwendung der ref()-Funktion, um vom customer_withdrawals.sql-Modell aufzurufen

    ```SQL
    WITH CTE AS (
        SELECT *
        FROM {{ ref('customer_withdrawals') }}
    )
    SELECT
    member_name,
    listagg(book_name::text, ',') as late_books,
    discount_applied,
    ROUND(SUM(fee_applied),2) as fee_to_pay
    FROM CTE
    GROUP BY 1,3
    ```

26. Führe in der Befehlszeile folgenden Befehl aus

    ```sh
    dbt build --select library_loans
    ```

    Um alle library_loans-Modelle zu erstellen und zu testen. Alle Tests sollten bestanden werden

27. Erstelle einen Ordner im tests-Ordner mit dem Namen generic ("/tests/generic")

28. Erstelle im generic-tests-Ordner eine SQL-Datei mit dem Namen no_negative_values.sql ("/tests/generic/no_negative_values.sql")

29. Kopiere den Inhalt der Starter-Datei no_negative_values.sql in die no_negative_values.sql-Datei

    ```sql
    {% test no_negative_values(model, column_name)%}
        SELECT {{column_name}}
        FROM {{model}}
        WHERE {{column_name}} < 0
    {% endtest %}
    ```

30. Aktualisiere die \_schema.yml-Datei im library_loans-Ordner ("/models/library_loans/\_schema.yml"), um den no_negative_values-Test auf die fee_applied-Spalte des customer_withdrawals-Modells am Ende der Datei anzuwenden.

    ```yml
    - name: customer_withdrawals
      columns:
        - name: fee_applied
          data_tests:
            - no_negative_values
    ```

    <details>
    <summary>vollständige _schema.yml</summary>

    ```yml
    version: 2

    sources:
      - name: library
        database: dbt_course
        schema: library_loans
        tables:
          - name: books_factual
          - name: books_fictional
          - name: loans
          - name: members

    models:
      - name: stg_members
        columns:
          - name: member_id
            data_tests:
              - unique
              - not_null
          - name: membership_tier
            data_tests:
              - accepted_values:
                  arguments:
                    values: ["Bronze", "Silver", "Gold"]
      - name: stg_books
        columns:
          - name: book_id
            data_tests:
              - unique
              - not_null
      - name: stg_loans
        columns:
          - name: loan_id
            data_tests:
              - unique
              - not_null
          - name: book_id
            data_tests:
              - relationships:
                  arguments:
                    to: ref('stg_books')
                    field: book_id
          - name: member_id
            data_tests:
              - relationships:
                  arguments:
                    to: ref('stg_members')
                    field: member_id
      - name: customer_withdrawals
        columns:
          - name: fee_applied
            data_tests:
              - no_negative_values
    ```

    </details>

31. Führe in der Befehlszeile folgenden Befehl aus

    ```sh
    dbt test --select customer_withdrawals
    ```

    Um das customer_withdrawals-Modell zu testen. Schau dir an, wie der no_negative_values-Test angewendet wird.

32. Erstelle eine Datei mit dem Namen packages.yml im root-Verzeichnis ("/packages.yml")

33. Kopiere die Install-YAML-Anweisungen aus [dbt_utils](https://hub.getdbt.com/dbt-labs/dbt_utils/latest/) in die packages.yml

    ```yml
    packages:
      - package: dbt-labs/dbt_utils
        version: 1.3.0
    ```

    (n.b. deine Version kann variieren)

34. Führe in der Befehlszeile folgenden Befehl aus

    ```sh
    dbt deps
    ```

    Um das dbt_utils-Paket zu installieren

35. Erstelle solution.csv im seeds-Ordner ("/seeds/solution.csv")

36. Öffne die Starter-Datei solution.csv **in einem Text-Editor** und kopiere den Inhalt in /seeds/solution.csv

    <details>
    <summary>solution.csv Inhalt</summary>

    [solution.csv](../files/solution.csv)

    </details>

37. Aktualisiere die \_schema.yml-Datei im library_loans-Ordner ("/models/library_loans/\_schema.yml"), um einen seeds-Abschnitt am Ende der Datei mit solution hinzuzufügen.

    ```yml
    seeds:
      - name: solution
    ```

    <details>
    <summary>vollständige _schema.yml</summary>

    ```yml
    version: 2

    sources:
      - name: library
        database: dbt_course
        schema: library_loans
        tables:
          - name: books_factual
          - name: books_fictional
          - name: loans
          - name: members

    models:
      - name: stg_members
        columns:
          - name: member_id
            data_tests:
              - unique
              - not_null
          - name: membership_tier
            data_tests:
              - accepted_values:
                  arguments:
                    values: ["Bronze", "Silver", "Gold"]
      - name: stg_books
        columns:
          - name: book_id
            data_tests:
              - unique
              - not_null
      - name: stg_loans
        columns:
          - name: loan_id
            data_tests:
              - unique
              - not_null
          - name: book_id
            data_tests:
              - relationships:
                  arguments:
                    to: ref('stg_books')
                    field: book_id
          - name: member_id
            data_tests:
              - relationships:
                  arguments:
                    to: ref('stg_members')
                    field: member_id
      - name: customer_withdrawals
        columns:
          - name: fee_applied
            data_tests:
              - no_negative_values

    seeds:
      - name: solution
    ```

    </details>

38. Führe in der Befehlszeile folgenden Befehl aus

    ```sh
    dbt seed
    ```

    Um die seed-Datei in das Data Warehouse zu laden

39. Aktualisiere die \_schema.yml-Datei im library_loans-Ordner ("/models/library_loans/\_schema.yml"), um den [dbt_utils.equal_rowcount](https://github.com/dbt-labs/dbt-utils/tree/1.3.0/#equal_rowcount-source)-Test auf das customers_with_late_fees-Modell im Vergleich zum solution-seed hinzuzufügen. Füge auch einen [accepted_range-Test](https://github.com/dbt-labs/dbt-utils/tree/1.3.0/#accepted_range-source) auf die discount_applied-Spalte hinzu, um zu prüfen, ob sie zwischen 0 und 25% liegt

    ```yml
    - name: customers_with_late_fees
      data_tests:
        - dbt_utils.equal_rowcount:
            compare_model: ref('solution')
      columns:
        - name: discount_applied
          data_tests:
            - dbt_utils.accepted_range:
                min_value: 0
                max_value: 0.25
    ```

    <details>
    <summary>vollständige _schema.yml</summary>

    ```yml
    version: 2

    sources:
      - name: library
        database: dbt_course
        schema: library_loans
        tables:
          - name: books_factual
          - name: books_fictional
          - name: loans
          - name: members

    models:
      - name: stg_members
        columns:
          - name: member_id
            data_tests:
              - unique
              - not_null
          - name: membership_tier
            data_tests:
              - accepted_values:
                  arguments:
                    values: ["Bronze", "Silver", "Gold"]
      - name: stg_books
        columns:
          - name: book_id
            data_tests:
              - unique
              - not_null
      - name: stg_loans
        columns:
          - name: loan_id
            data_tests:
              - unique
              - not_null
          - name: book_id
            data_tests:
              - relationships:
                  arguments:
                    to: ref('stg_books')
                    field: book_id
          - name: member_id
            data_tests:
              - relationships:
                  arguments:
                    to: ref('stg_members')
                    field: member_id
      - name: customer_withdrawals
        columns:
          - name: fee_applied
            data_tests:
              - no_negative_values
      - name: customers_with_late_fees
        data_tests:
          - dbt_utils.equal_rowcount:
              compare_model: ref('solution')
        columns:
          - name: discount_applied
            data_tests:
              - dbt_utils.accepted_range:
                  min_value: 0
                  max_value: 0.25

    seeds:
      - name: solution
    ```

    </details>

40. Führe in der Befehlszeile folgenden Befehl aus

    ```sh
    dbt test --select customers_with_late_fees
    ```

    Um diese Tests auf dem customers_with_late_fees-Modell ausführen zu sehen (und zu bestehen)

41. Führe in der Befehlszeile folgenden Befehl aus

    ```sh
    dbt build --select library_loans
    ```

    Um alle library_loans-Modelle zu erstellen und zu testen. Alle Tests sollten bestanden werden

### [Zurück zur Anleitung](../ReadMe.md)

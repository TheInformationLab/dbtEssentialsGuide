# Testen & Debugging

1. Aktualisiere den models-Bereich in der `dbt_project.yml`-Datei (Zeilen 38-45), um alles in `library_loans` standardmäßig als Tabelle zu materialisieren

   ```yml
   models:
     my_new_project:
       lego:
         +materialized: table

       library_loans:
         +materialized: table
   ```

2. Erstelle einen neuen Ordner im models-Verzeichnis namens `library_loans` (`/models/library_loans`)

3. Erstelle im library_loans-Ordner eine neue Datei namens `customers_with_late_fees.sql` (`/models/library_loans/customers_with_late_fees.sql`) und kopiere die SQL-Anweisungen hinein.

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
   listagg(book_name::text, ',') within group (order by book_name) as late_books,
   discount_applied,
   ROUND(SUM(fee_applied),2) as fee_to_pay
   FROM CTE
   GROUP BY 1,3
   ```

4. Dupliziere `customer_with_late_fees.sql` und speichere es als `models/library_loans/legacy_customers_with_late_fees.sql`

5. Erstelle eine `packages.yml` im root-Verzeichnis (`/packages.yml`) und füge folgendes ein:

   ```yml
   packages:
     - package: dbt-labs/dbt_utils
       version: 1.3.3
   ```

6. Führe `dbt deps` aus, um das neue Paket zu installieren

7. Erstelle eine `_schema.yml`-Datei im `library_loans`-Verzeichnis und füge folgendes ein:

   ```yml
   models:
     - name: customers_with_late_fees
       data_tests:
         - dbt_utils.equality:
             arguments:
               compare_model: ref('legacy_customers_with_late_fees')
   ```

   Dies beweist, dass unsere Code-Änderungen die Ausgabe nicht verändern.

8. Erstelle eine `_sources.yml`-Datei im `library_loans`-Verzeichnis (`/models/library_loans/_sources.yml`) und kopiere folgendes hinein:

   ```yml
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

   Dies erstellt die Quellen, auf die dbt verweisen kann

9. Erstelle zwei Unterverzeichnisse in `/models/library_loans/`:
   `staging` (`/models/library_loans/staging`) und
   `intermediate` (`/models/library_loans/intermediate`)

10. Aktualisiere den `models`-Teil von `dbt_project.yml`, um Staging-Modelle in `library_loans/staging` als View und Intermediate-Modelle in `library_loans/intermediate` als Tabelle zu materialisieren:

    ```yaml
    models:
      my_new_project:
        lego:
          +materialized: table

        library_loans:
          +materialized: table
          staging:
            +materialized: view
          intermediate:
            +materialized: table
    ```

11. Erstelle für jede Quelle im `staging`-Unterverzeichnis ein Staging-Modell:
    - `stg_books_factual.sql` (`/models/libary_loans/staging/stg_books_factual.sql`)

      ```sql
      select * from {{ source('library', 'books_factual') }}
      ```

    - `stg_books_fictional.sql` (`/models/libary_loans/staging/stg_books_fictional.sql`)

      ```sql
      select * from {{ source('library', 'books_fictional') }}
      ```

    - `stg_loans.sql` (`/models/libary_loans/staging/stg_loans.sql`)

      ```sql
      select * from {{ source('library', 'loans') }}
      ```

    - `stg_members.sql` (`/models/libary_loans/staging/stg_members.sql`)

      ```sql
      select * from {{ source('library', 'members') }}
      ```

12. Erstelle `int_books.sql` im `intermediate`-Verzeichnis (`/models/libary_loans/intermediate/int_books.sql`):

    ```sql
    select
        book_id,
        location,
        book_name,
        'factual' as genre
    from
        {{ ref("stg_books_factual") }}

    union all

    select
        book_id,
        location,
        book_name,
        'fictional' as genre
    from
        {{ ref("stg_books_fictional") }}
    ```

    Dies kombiniert alle Bücher in einem einzelnen Modell.

13. Aktualisiere `customers_with_late_fees`, um unsere neuen Staging- und Intermediate-Modelle zu verwenden.

    ```sql
    WITH CTE AS (
        SELECT
            B.book_name,
            M.member_name,
            M.discount_rate / 100 as discount_applied,
            SUM(L.late_fee * (M.discount_rate / 100)) as fee_applied
        FROM {{ ref('stg_members') }} AS M
            INNER JOIN {{ ref('stg_loans') }} AS L ON M.member_id = L.member_id
            LEFT JOIN {{ ref('int_books') }} AS B ON B.book_id = L.book_id
        GROUP BY
            B.book_name,
            M.member_name,
            M.discount_rate
    )
    SELECT
        member_name,
        listagg(book_name::text, ',') within group (order by book_name) as late_books,
        discount_applied,
        ROUND(SUM(fee_applied),2) as fee_to_pay
    FROM
        CTE
    GROUP BY
        member_name,
        discount_applied
    ```

14. Extrahiere die CTE in ein neues `customer_withdrawals`-Modell
    - `models/library_loans/customer_withdrawals.sql`:

      ```sql
      SELECT
          B.book_name,
          M.member_name,
          M.discount_rate / 100 as discount_applied,
          SUM(L.late_fee * (M.discount_rate / 100)) as fee_applied
      FROM {{ ref('stg_members') }} AS M
          INNER JOIN {{ ref('stg_loans') }} AS L ON M.member_id = L.member_id
          LEFT JOIN {{ ref('int_books') }} AS B ON B.book_id = L.book_id
      GROUP BY
          B.book_name,
          M.member_name,
          M.discount_rate
      ```

    - `models/library_loans/customers_with_late_fees.sql`:

      ```sql
      SELECT
          member_name,
          listagg(book_name::text, ',') within group (order by book_name) as late_books,
          discount_applied,
          ROUND(SUM(fee_applied),2) as fee_to_pay
      FROM
          {{ ref('customer_withdrawals') }}
      GROUP BY
          member_name,
          discount_applied
      ```

15. Erstelle eine `_schema.yml` im `staging`-Verzeichnis (`models/library_loans/staging/_schema.yml`) mit den Staging-Tests:

    ```yaml
    models:
      - name: stg_books_factual
        columns:
          - name: book_id
            data_tests:
              - not_null
              - unique
      - name: stg_books_fictional
        columns:
          - name: book_id
            data_tests:
              - not_null
              - unique
      - name: int_members
        columns:
          - name: member_id
            data_tests:
              - not_null
              - unique
          - name: membership_tier
            data_tests:
              - accepted_values:
                  arguments:
                    values:
                      - Gold
                      - Silver
                      - Bronze
      - name: stg_loans
        columns:
          - name: loan_id
            data_tests:
              - not_null
              - unique
          - name: book_id
            data_tests:
              - relationships:
                  arguments:
                    field: book_id
                    to: ref('int_books')
          - name: member_id
            data_tests:
              - relationships:
                  arguments:
                    field: member_id
                    to: ref('stg_members')
    ```

16. Erstelle eine `_schema.yml` im `intermediate`-Verzeichnis (`models/library_loans/intermediate/_schema.yml`) mit den Intermediate-Tests:

    ```yaml
    models:
      - name: int_books
        columns:
          - name: book_id
            data_tests:
              - not_null
              - unique
    ```

17. Aktualisiere die `_schema.yml` im `library_loans`-Verzeichnis (`models/library_loans/_schema.yml`) und füge die Tests hinzu:

    ```yaml
    models:
      - name: customer_withdrawals
        columns:
          - name: discount_applied
            data_tests:
              - dbt_utils.accepted_range:
                  arguments:
                    min_value: 0
                    inclusive: true
          - name: fee_applied
            data_tests:
              - dbt_utils.accepted_range:
                  arguments:
                    min_value: 0
                    inclusive: true

      - name: customers_with_late_fees
        data_tests:
          - dbt_utils.equality:
              arguments:
                compare_model: ref('legacy_customers_with_late_fees')
    ```

18. Aktualisiere `stg_books_fictional.sql`, um die Duplikate zu entfernen:

    ```sql
    select distinct * from {{ source('library', 'books_fictional') }}
    ```

19. Mache das gleiche vorsorglich in `stg_books_factual.sql`:

    ```sql
    select distinct * from {{ source('library', 'books_factual') }}
    ```

20. Füge eine WHERE-Klausel zu `stg_members.sql` hinzu, um Zeilen mit ungültigen Mitgliedschaftsstufen zu filtern:

    ```sql
    select *
    from {{ source('library', 'members') }}
    where
        membership_tier in ('Bronze', 'Silver', 'Gold')
    ```

21. Aktualisiere `stg_loans.sql`, um Zeilen mit ungültigen Beziehungen zu filtern:

    ```sql
    select *
    from {{ source('library', 'loans') }}
    where
        member_id in (select member_id from {{ ref('stg_members') }})
        and book_id in (select book_id from {{ ref('int_books') }})
    ```

## [Zurück zur Leitfadenliste](readme.md)

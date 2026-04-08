# Testing & Debugging

1. Update the models section of the `dbt_project.yml` file (lines 38-45) to materialize everything in `library_loans` as a table by default

   ```yml
   models:
     my_new_project:
       lego:
         +materialized: table

       library_loans:
         +materialized: table
   ```

2. Make a new folder in the models directory called `library_loans` (`/models/library_loans`)

3. In library_loans make a new file called `customers_with_late_fees.sql` (`/models/library_loans/customers_with_late_fees.sql`) and copy/paste into it the SQL.

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

4. Duplicate `customer_with_late_fees.sql` and save as `models/library_loans/legacy_customers_with_late_fees.sql`

5. Create a `packages.yml` in the root directory (`/packages.yml`) and paste into it the following:

   ```yml
   packages:
     - package: dbt-labs/dbt_utils
       version: 1.3.3
   ```

6. Run `dbt deps` to install the new package

7. Create a `_schema.yml` file in the `library_loans` directory and paste into it the following:

   ```yml
   models:
     - name: customers_with_late_fees
       data_tests:
         - dbt_utils.equality:
             arguments:
               compare_model: ref('legacy_customers_with_late_fees')
   ```

   This will prove that our code changes to not change the output.

8. Create a `_sources.yml` file in the `library_loans` directory (`/models/library_loans/_sources.yml`) and copy/paste into it the following:

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

   This will create our sources for dbt to reference

9. Update `dbt_project.yml` to materialize everything in the `libary_loans/staging` sub directory as view:

   ```yaml
   models:
     my_new_project:
       lego:
         +materialized: table

       library_loans:
         +materialized: table
         staging:
           +materialized: view
   ```

10. Create the `staging` sub directory in `/models/library_loans/` (`/models/library_loans/staging`)

11. Create a staging model for every source inside the `staging` sub directory:
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

12. Update `dbt_project.yml` to materialize everything in `libary_loans/intermediate` as table:

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

13. Create a sub directory called `intermediate` under `library_loans`

14. Create `int_books.sql` inside `intermediate` (`/models/libary_loans/intermediate/int_books.sql`):

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

    This combines all books in a single model.

15. Update `customers_with_late_fees` to use our new staging and intermediate models.

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

16. Extract the CTE into a new `customer_withdrawals` model
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

17. Create a `_schema.yml` inside `staging` (`models/library_loans/staging/_schema.yml`) with the staging tests:

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
      - name: stg_members
        columns:
          - name: member_id
            data_tests:
              - not_null
              - unique
          - name: membership_tiers
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
                    to: ref('stg_books')
          - name: member_id
            data_tests:
              - relationships:
                  arguments:
                    field: member_id
                    to: ref('stg_members')
    ```

18. Create a `_schema.yml` inside `intermediate` (`models/library_loans/intermediate/_schema.yml`) with the staging tests:

    ```yaml
    models:
      - name: int_books
        columns:
          - name: book_id
            data_tests:
              - not_null
              - unique
    ```

19. Update the `_schema.yml` in `library_loans` (`models/library_loans/_schema.yml`) and add the tests:

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

20. Update `stg_books_fictional.sql` to remove the duplicates:

    ```sql
    select distinct * from {{ source('library', 'books_fictional') }}
    ```

21. Do the same as a precaution in `stg_books_factual.sql`:

    ```sql
    select distinct * from {{ source('library', 'books_factual') }}
    ```

22. Add a WHERE clause to `stg_members.sql` to filter out rows with invalid membership tiers:

    ```sql
    select *
    from {{ source('library', 'members') }}
    where
        membership_tier in ('Bronze', 'Silver', 'Gold')
    ```

23. Update `stg_loans.sql` to filter out rows with invalid relations:

    ```sql
    select *
    from {{ source('library', 'loans') }}
    where
        member_id in (select member_id from {{ ref('stg_members') }})
        and book_id in (select book_id from {{ ref('int_books') }})
    ```

## [Back to guide list](readme.md)

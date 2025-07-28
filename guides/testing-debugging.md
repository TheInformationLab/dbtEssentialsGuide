# Testing & Debugging

1. Make a new folder in the models directory called library_loans ("/models/library_loans")

2. In library_loans make a new file called customers_with_late_fees.sql ("/models/library_loans/customers_with_late_fees.sql") and copy/paste into it the SQL from the starter file customers_with_late_fees.sql.

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

3. Create a schema.yml file in the library_loans directory ("/models/library_loans/schema.yml") and copy/paste into it the yaml from the starter file schema.yml

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

   This will create our sources for dbt to reference

4. Update the models section of the dbt_project.yml file (lines 38-45) to materialize everything in library_loans as a table by default

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

5. Update customers_with_late_fees.sql to use our defined sources, rather than the hardcoded values

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

6. Add primary key tests to our primary keys:

- unique and not null tests
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

7. Check that we only have 'Gold', 'Silver' and 'Bronze' membership tiers

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
                  values: ["Bronze", "Silver", "Gold"]
```

8. In the command bar, run the command

   ```sh
   dbt test --select library_loans
   ```

   7 should pass, 2 should fail

9. Create a new model in libary_loans called stg_members.sql ("/models/library_loans/stg_members.sql")

10. Copy in the sql from the starter file stg_members.sql

    ```sql
    SELECT *
    FROM {{ source("library", "members") }}
    WHERE member_id IS NOT NULL
    AND membership_tier IN ('Bronze', 'Silver', 'Gold')
    ```

    This creates a staging model which abides by our constraints for incoming members data

11. In the library_loans schema.yml take the members tests (lines 26-33) and apply them to the new stg_members in a models block

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
                  values: ["Bronze", "Silver", "Gold"]
    ```

12. In the command bar, run the command

    ```sh
    dbt build --select library_loans
    ```

    The accepted values test should now pass, but the unique fictional book_id should still fail

13. Create a new model in libary_loans called stg_books.sql ("/models/library_loans/stg_books.sql")

14. Copy in the sql from the starter file stg_books.sql

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

    This combines both fictional and factual books into one table, while removing any null book_ids

15. In the library_loans schema.yml remove the book_id tests (lines 9-13 and 15-19) and apply a single book_id non_null and unique test to the new stg_books in the models section.

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
                  values: ["Bronze", "Silver", "Gold"]
      - name: stg_books
        columns:
          - name: book_id
            data_tests:
              - unique
              - not_null
    ```

16. In the command bar, run the command

    ```sh
    dbt build --select library_loans
    ```

    All tests should now pass

17. Add relationship tests to our loans source:

    - loans.book_id
      - has a relationship to stg_books.book_id
    - loans.member_id
      - has a relationship to stg_members.member_id

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
                      to: ref('stg_books')
                      field: book_id
              - name: member_id
                data_tests:
                  - relationships:
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
                  values: ["Bronze", "Silver", "Gold"]
      - name: stg_books
        columns:
          - name: book_id
            data_tests:
              - unique
              - not_null
    ```

18. In the command bar, run the command

    ```sh
    dbt test --select library_loans
    ```

    Some tests will fail

19. Create a new model in libary_loans called stg_loans.sql ("/models/library_loans/stg_loans.sql")

20. Copy in the sql from the starter file stg_loans.sql

    ```sql
    SELECT *
    FROM {{ source("library", "loans") }}
    WHERE loan_id IS NOT NULL
    AND book_id IS NOT NULL
    AND member_id IS NOT NULL
    AND book_id IN (SELECT book_id FROM {{ ref('stg_books') }})
    AND member_id IN (SELECT member_id FROM {{ ref('stg_members') }})
    ```

    This applies our not null tests, and also checks for referential integrity with our foreign keys (book_id and member_id)

21. Update the schema.yml file in the library_loans folder to remove all the source loans data_tests and apply them to the new stg_loans model

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
                  to: ref('stg_books')
                  field: book_id
          - name: member_id
            data_tests:
              - relationships:
                  to: ref('stg_members')
                  field: member_id
    ```

22. Update the customers_with_late_fees.sql file to refer to our new staging tables, rather than the original sources using the ref() function. You'll also need to update the SQL now that there's only 1 stg_books table compared to the 2 book sources.

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

23. Create a new file called customer_withdrawals.sql in the library_loans folder ("/models/library_loans/customer_withdrawals.sql").

24. Copy the CTE query from customers_with_late_fees.sql into customer_withdrawals.sql (lines 2-10)

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

25. Update the CTE in customer_with_late_fees.sql (lines 1-11) using the ref() function to call from the customer_withdrawals.sql model

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

26. In the command bar, run the command

    ```sh
    dbt build --select library_loans
    ```

    To make and test all the library_loans models. All tests should pass

27. Create a folder within the tests folder called generic ("/tests/generic")

28. In the generic tests folder, create a sql file called no_negative_values.sql ("/tests/generic/no_negative_values.sql")

29. Copy into the no_negative_values.sql file the contents of the starter file no_negative_values.sql

    ```sql
    {% test no_negative_values(model, column_name)%}
        SELECT {{column_name}}
        FROM {{model}}
        WHERE {{column_name}} < 0
    {% endtest %}
    ```

30. Update the schema.yml file in the library_loans folder ("/models/library_loans/schema.yml") to apply the no_negative_values test to the fee_applied column of the customer_withdrawals model at the end of the file.

    ```yml
    - name: customer_withdrawals
      columns:
        - name: fee_applied
          data_tests:
            - no_negative_values
    ```

    <details>
    <summary>full schema.yml</summary>

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
                  to: ref('stg_books')
                  field: book_id
          - name: member_id
            data_tests:
              - relationships:
                  to: ref('stg_members')
                  field: member_id
      - name: customer_withdrawals
        columns:
          - name: fee_applied
            data_tests:
              - no_negative_values
    ```

    </details>

31. In the command bar, run the command

    ```sh
    dbt test --select customer_withdrawals
    ```

    To test the customer_withdrawals model. See how the no_negative_values test is applied.

32. Create a file in the root directory called packages.yml ("/packages.yml")

33. Copy the install yaml from [dbt_utils](https://hub.getdbt.com/dbt-labs/dbt_utils/latest/) into the packages.yml

    ```yml
    packages:
      - package: dbt-labs/dbt_utils
        version: 1.3.0
    ```

    (n.b. your version may vary)

34. In the command bar, run the command

    ```sh
    dbt deps
    ```

    To install the dbt_utils package

35. Create solution.csv in the seeds folder ("/seeds/solution.csv")

36. Open the starter file solution.csv **in a text editor** and copy paste the contents into /seeds/solution.csv

    <details>
    <summary>solution.csv contents</summary>

    [solution.csv](../files/solution.csv)

    </details>

37. Update the schema.yml file in the library_loans folder ("/models/library_loans/schema.yml") to add a seeds section to the end of the file with solution named.

    ```yml
    seeds:
      - name: solution
    ```

    <details>
    <summary>full schema.yml</summary>

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
                  to: ref('stg_books')
                  field: book_id
          - name: member_id
            data_tests:
              - relationships:
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

38. In the command bar, run the command

    ```sh
    dbt seed
    ```

    To load the seed file into the data warehouse

39. Update the schema.yml file in the library_loans folder ("/models/library_loans/schema.yml") to add the [dbt_utils.equal_rowcount](https://github.com/dbt-labs/dbt-utils/tree/1.3.0/#equal_rowcount-source) test to the customers_with_late_fees model comparing to the solution seed. Also add an [accepted_range test](https://github.com/dbt-labs/dbt-utils/tree/1.3.0/#accepted_range-source) on the discount_applied column to check it's between 0 and 25%

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
    <summary>full schema.yml</summary>

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
                  to: ref('stg_books')
                  field: book_id
          - name: member_id
            data_tests:
              - relationships:
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

40. In the command bar, run the command

    ```sh
    dbt test --select customers_with_late_fees
    ```

    To see these tests running (and passing) against the customers_with_late_fees model

41. In the command bar, run the command

    ```sh
    dbt build --select library_loans
    ```

    To make and test all the library_loans models. All tests should pass

### [Back to guide list](../ReadMe.md)

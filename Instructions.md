# dbt Essentials Guide
This is our step by step instructions for the TIL dbt essentials training

## Getting started with dbt Cloud

1. Click "Initialize dbt project" in version control

    ![initialise dbt project](./images/initialise_project.png)

2. Click "Commit and sync" in version control and enter a commit message in the dialogue box

    ![commit and sync](./images/commit_and_sync.png)

3. Click "Create branch" in version control and name it "intro-training"

    ![create branch](./images/create_branch.png)

    ![name the branch](./images/branch_name.png)

4. In the command bar, run the command

    ```sh
    dbt run
    ```
    to create all models in our data warehouse

5. Make a new folder "models/lego"

6. Update dbt_project.yml lines 38-43 to materialise all models in the lego directory as tables by default

    ```yaml
    models:
      my_new_project:
        # Applies to all files under models/example/
        example:
          +materialized: table

        lego:
          +materialized: table
    ```

7. Create a new file within the legos directory called parts_per_set.sql ("models/lego/parts_per_set.sql") and paste in the contents from Original Lego Script.txt

    ```sql
    WITH UNIQUE_PARTS AS (
    SELECT 
        P.part_num
    FROM dbt_course.lego.parts as P
    INNER JOIN dbt_course.lego.inventory_parts as IP on P.part_num = IP.part_num
    INNER JOIN dbt_course.lego.inventories as I on I.id = IP.inventory_id
    INNER JOIN dbt_course.lego.sets as S on S.set_num = I.set_num
        GROUP BY P.part_num
        HAVING COUNT(*) = 1
    )
    SELECT 
        T.name as theme_name,
        S.name as set_name,
        S.year as set_year,
        CASE 
            WHEN UP.part_num IS NULL THEN 'Not Unique' 
            ELSE 'Unique' 
        END as unique_part,
        COUNT(P.part_num) as parts
    FROM dbt_course.lego.parts as P
    LEFT JOIN UNIQUE_PARTS as UP on P.part_num = UP.part_num
    INNER JOIN dbt_course.lego.inventory_parts as IP on P.part_num = IP.part_num
    INNER JOIN dbt_course.lego.inventories as I on I.id = IP.inventory_id
    INNER JOIN dbt_course.lego.sets as S on S.set_num = I.set_num
    INNER JOIN dbt_course.lego.themes as T on T.id = S.theme_id
    GROUP BY 1,2,3,4;
    ```

8. In the command bar, run the command

    ```sh
    dbt run --select parts_per_set
    ```
    to create the parts_per_set model in our data warehouse

9. Remove the semicolon from line 26 in models/lego/parts_per_set.sql

    ```sql
    GROUP BY 1,2,3,4
    ```

10. In the command bar, run the command

    ```sh
    dbt run --select lego
    ```
    to create all models in the lego folder in our data warehouse

11. Create a new file within the lego directory called sources.yml ("models/lego/sources.yml") directing dbt to where the source tables are.

    ```yaml
    version: 2
    
    sources:
      - name: lego
        database: dbt_course
        schema: lego
        tables:
          - name: parts
          - name: inventory_parts
          - name: inventories
          - name: sets
          - name: themes
    ```

12. Edit parts_per_set.sql to replace all hardcoded table names with the source function

    ```sql
    WITH UNIQUE_PARTS AS (
    SELECT 
        P.part_num
    FROM {{ source('lego', 'parts') }} as P
    INNER JOIN {{ source('lego', 'inventory_parts') }} as IP on P.part_num = IP.part_num
    INNER JOIN {{ source('lego', 'inventories') }} as I on I.id = IP.inventory_id
    INNER JOIN {{ source('lego', 'sets') }} as S on S.set_num = I.set_num
        GROUP BY P.part_num
        HAVING COUNT(*) = 1
    )
    SELECT 
        T.name as theme_name,
        S.name as set_name,
        S.year as set_year,
        CASE 
            WHEN UP.part_num IS NULL THEN 'Not Unique' 
            ELSE 'Unique' 
        END as unique_part,
        COUNT(P.part_num) as parts
    FROM {{ source('lego', 'parts') }} as P
    LEFT JOIN UNIQUE_PARTS as UP on P.part_num = UP.part_num
    INNER JOIN {{ source('lego', 'inventory_parts') }} as IP on P.part_num = IP.part_num
    INNER JOIN {{ source('lego', 'inventories') }} as I on I.id = IP.inventory_id
    INNER JOIN {{ source('lego', 'sets') }} as S on S.set_num = I.set_num
    INNER JOIN {{ source('lego', 'themes') }} as T on T.id = S.theme_id
    GROUP BY 1,2,3,4
    ```

13. Create a new file in the lego directory called unique_parts.sql ("models/lego/unique_parts.sql")

14. Copy lines 2-9 from parts_per_set.sql into unique_parts.sql

    ```sql
    SELECT 
        P.part_num
    FROM {{ source('lego', 'parts') }} as P
    INNER JOIN {{ source('lego', 'inventory_parts') }} as IP on P.part_num = IP.part_num
    INNER JOIN {{ source('lego', 'inventories') }} as I on I.id = IP.inventory_id
    INNER JOIN {{ source('lego', 'sets') }} as S on S.set_num = I.set_num
        GROUP BY P.part_num
        HAVING COUNT(*) = 1
    ```

15. Add a config block to the top of unique_parts.sql to materialise it as a view in the data warehouse

    ```sql
    {{
        config(
            materialized='view'
        )
    }}

    SELECT 
        P.part_num
    FROM {{ source('lego', 'parts') }} as P
    INNER JOIN {{ source('lego', 'inventory_parts') }} as IP on P.part_num = IP.part_num
    INNER JOIN {{ source('lego', 'inventories') }} as I on I.id = IP.inventory_id
    INNER JOIN {{ source('lego', 'sets') }} as S on S.set_num = I.set_num
        GROUP BY P.part_num
        HAVING COUNT(*) = 1
    ```

16. Update parts_per_set.sql to replace the CTE (lines 1-10) with a ref() function aimed at unique_parts.sql

    ```sql
    WITH UNIQUE_PARTS AS (
        SELECT * 
        from {{ ref('unique_parts') }}
    )
    SELECT 
        T.name as theme_name,
        S.name as set_name,
        S.year as set_year,
        CASE 
            WHEN UP.part_num IS NULL THEN 'Not Unique' 
            ELSE 'Unique' 
        END as unique_part,
        COUNT(P.part_num) as parts
    FROM {{ source('lego', 'parts') }} as P
    LEFT JOIN UNIQUE_PARTS as UP on P.part_num = UP.part_num
    INNER JOIN {{ source('lego', 'inventory_parts') }} as IP on P.part_num = IP.part_num
    INNER JOIN {{ source('lego', 'inventories') }} as I on I.id = IP.inventory_id
    INNER JOIN {{ source('lego', 'sets') }} as S on S.set_num = I.set_num
    INNER JOIN {{ source('lego', 'themes') }} as T on T.id = S.theme_id
    GROUP BY 1,2,3,4
    ```

17. In the command bar, run the command

    ```sh
    dbt run --select lego
    ```
    to create the two lego models sequentially in the data warehouse

18. Create a new file within the lego directory called schema.yml ("models/lego/schema.yml") to add in documentation and tests.

    ```yml
    version: 2
    
    models:
      - name: unique_parts
        description: The part_nums which are only used in one set
        columns:
          - name: part_num
            data_tests:
              - not_null
              
      - name: parts_per_set
        description: Shows the number of parts in each set along with their theme and whether they have unique parts
        columns:
          - name: theme_name
            data_tests:
              - not_null
          - name: set_name
            data_tests:
              - not_null
          - name: set_year
            data_tests:
              - not_null
    ```

19. In the command bar, run the command

    ```sh
    dbt build
    ```
    to create and test all models in our data warehouse

20. In the command bar, run the command

    ```sh
    dbt test
    ```
    to test all models

21. Edit the my_first_dbt_model.sql file in the example directory ("models/example/my_first_dbt_model.sql") to remove the comment on line 27

    ```sql

    /*
        Welcome to your first dbt model!
        Did you know that you can also configure models directly within SQL files?
        This will override configurations stated in dbt_project.yml

        Try changing "table" to "view" below
    */

    {{ config(materialized='table') }}

    with source_data as (

        select 1 as id
        union all
        select null as id

    )

    select *
    from source_data

    /*
        Uncomment the line below to remove records with null `id` values
    */

    where id is not null

    ```

22. In the command bar, run the command

    ```sh
    dbt build
    ```
    to create and test all models in our data warehouse. All should pass.

23. Update the sources.yml file in the lego directory ("models/lego/sources.yml") to paste in the completed version from the fileshare.

    ```yml
    version: 2
    
    sources:
      - name: lego
        database: DBT_COURSE
        schema: LEGO
        tables:
          - name: colors
            description: dimension table of lego colors
            columns:
              - name: id
                description: primary key and unique identifier of each color
                data_tests:
                  - not_null
                  - unique
              - name: name
                description: the name of the color
                data_tests:
                  - not_null
                  - unique
              - name: RGB
                description: the hex value of the color
                data_tests:
                  - not_null
              - name: is_trans
                data_tests:
                  - accepted_values:
                      values: ["TRUE", "FALSE"]
    
          - name: inventories
            description: dimension table of what we currently stock
            columns:
              - name: id
                description: primary key
                data_tests:
                  - unique
                  - not_null
              - name: version
                description: the version of each set we carry
                data_tests:
                  - not_null
              - name: set_num
                description: foreign key and the set identifier
                data_tests:
                  - not_null
                  - relationships:
                      to: source('lego','sets')
                      field: set_num
    
          - name: inventory_parts
            description: the parts within each set we stock
            columns:
              - name: inventory_id
                description: foreign key to inventories table
                data_tests:
                  - not_null
                  - relationships:
                      to: source('lego','inventories')
                      field: id
              - name: part_num
                description: foreign key to parts table - not behaving properly
                data_tests:
                  - not_null
              - name: color_id
                description: foreign key to colors table
                data_tests:
                  - not_null
                  - relationships:
                      to: source('lego','colors')
                      field: id
              - name: quantity
                description: how many of that part is in the set
                data_tests:
                  - not_null
              - name: is_spare
                description: boolean if the part is spare
                data_tests:
                  - not_null
                  - accepted_values:
                      values: ["TRUE", "FALSE"]
    
          - name: inventory_sets
            description: dimension table of sets and how many we stock
            columns:
              - name: inventory_id
                description: foreign key to inventories
                data_tests:
                  - not_null
                  - relationships:
                      to: source('lego','inventories')
                      field: id
              - name: set_num
                description: foreign key from sets
                data_tests:
                  - not_null
                  - relationships:
                      to: source('lego','sets')
                      field: set_num
              - name: quantity
                description: how many of each set we hold
    
          - name: parts
            description: dimension table of lego parts
            columns:
              - name: part_num
                description: primary key and unique identifier of each part
                data_tests:
                  - not_null
                  - unique
              - name: name
                description: the name of the part
                data_tests:
                  - not_null
              - name: part_cat_id
                description: foreign key from part_categories table
                data_tests:
                  - not_null
                  - relationships:
                      to: source('lego','part_categories')
                      field: id
    
          - name: part_categories
            description: dimension table combining parts into different categories
            columns:
              - name: id
                description: primary key
                data_tests:
                  - unique
                  - not_null
              - name: name
                description: the part category name
                data_tests:
                  - not_null
    
          - name: sets
            description: dimension table of all lego sets
            columns:
              - name: set_num
                description: primary key
                data_tests:
                  - not_null
                  - unique
              - name: name
                description: the name of the set
                data_tests:
                  - not_null
              - name: year
                description: the year the set was released
                data_tests:
                  - not_null
              - name: theme_id
                description: foreign key from themes
                data_tests:
                  - not_null
                  - relationships:
                      to: source('lego','themes')
                      field: id
              - name: num_parts
                description: the number of parts in each set
                data_tests:
                  - not_null
    
          - name: themes
            description: dimension table grouping sets into different themes
            columns:
              - name: id
                description: primary key
                data_tests:
                  - not_null
                  - unique
              - name: name
                description: the name of the theme
                data_tests:
                  - not_null
              - name: parent_id
                description: if a theme is a sub-theme, the id of its parent
                data_tests:
                  - relationships:
                      to: source('lego','themes')
                      field: id
    ```
    This completed version has full descriptions of sources and their columns. It also uses all 4 in-built data_tests (not_null, unique, accepted_values, relationships)

24. In the command bar, run the command

    ```sh
    dbt build
    ```
    to make and then test all our models sequentially in the data warehouse

25. In the command bar, run the command

    ```sh
    dbt docs generate
    ```
    to generate documentation based on the models and yaml files in our project

26. Click the little document icon to see the documentation

    ![docs icon](./images/docs_icon.png)

27. Click "Commit and sync" in version control and enter a commit message in the dialogue box

    ![commit and sync](./images/commit_and_sync.png)
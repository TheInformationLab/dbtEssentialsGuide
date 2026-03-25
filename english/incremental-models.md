# Incremental Models

1. Create a new file called fetch_weather.sql in the macros folder ("/macros/fetch_weather.sql").

2. Copy in the sql from the starter file fetch_weather.sql

   ```sql
   {% macro fetch_weather() %}
       {% do run_query("CALL DBT_COURSE.WEATHER.DBT_FETCH_WEATHER('"+ target.schema +"');")%}
   {% endmacro %}
   ```

   This runs the dbt_fetch_weather() procedure and produces its table in the schema in which we're working

3. Make a new folder in the models directory called office_weather ("/models/office_weather")

4. Create a _schema.yml file in the office_weather directory ("/models/office_weather/_schema.yml") and copy/paste into it the yaml from the starter file _schema.yml

   ```yml
   version: 2

   sources:
     - name: office_weather
       schema: "{{target.schema}}"
       tables:
         - name: weather_readings

   models:
     - name: stg_weather_data
   ```

5. Create stg_weather_data.sql in the office_weather directory ("/models/office_weather/stg_weather_data.sql")

6. Into stg_weather_data.sql ("/models/office_weather/stg_weather_data.sql") write the following query to stage our source weather_reading data

   ```SQL
   select *
   from {{ source('office_weather', 'weather_readings') }}
   ```

7. Update the _schema.yml file ("/models/office_weather/_schema.yml") to add a config block with a pre-hook to run the fetch_weather() macro before the stg_weather_data model is run

   ```yml
   version: 2

   sources:
     - name: office_weather
       schema: "{{target.schema}}"
       tables:
         - name: weather_readings

   models:
     - name: stg_weather_data
       config:
         pre_hook:
           - "{{ fetch_weather() }}"
   ```

8. Update the dbt_project.yml file ("/dbt_project.yml") to materialize everything in the office_weather folder as a view by default

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

       office_weather:
         +materialized: view
   ```

   <details>
   <summary>full dbt_project.yml</summary>

   ```yml
   # Name your project! Project names should contain only lowercase characters
   # and underscores. A good package name should reflect your organization's
   # name or the intended use of these models
   name: "my_new_project"
   version: "1.0.0"
   config-version: 2

   # This setting configures which "profile" dbt uses for this project.
   profile: "default"

   # These configurations specify where dbt should look for different types of files.
   # The `model-paths` config, for example, states that models in this project can be
   # found in the "models/" directory. You probably won't need to change these!
   model-paths: ["models"]
   analysis-paths: ["analyses"]
   test-paths: ["tests"]
   seed-paths: ["seeds"]
   macro-paths: ["macros"]
   snapshot-paths: ["snapshots"]

   target-path: "target" # directory which will store compiled SQL files
   clean-targets: # directories to be removed by `dbt clean`
     - "target"
     - "dbt_packages"

   # Configuring models
   # Full documentation: https://docs.getdbt.com/docs/configuring-models

   # In dbt, the default materialization for a model is a view. This means, when you run
   # dbt run or dbt build, all of your models will be built as a view in your data platform.
   # The configuration below will override this setting for models in the example folder to
   # instead be materialized as tables. Any models you add to the root of the models folder will
   # continue to be built as views. These settings can be overridden in the individual model files
   # using the `{{ config(...) }}` macro.

   models:
     my_new_project:
       # Applies to all files under models/example/
       example:
         +materialized: table

       lego:
         +materialized: table

       library_loans:
         +materialized: table

       office_weather:
         +materialized: view
   ```

   </details>

9. In the command bar, run the command

   ```sh
   dbt build --select office_weather
   ```

   To run our macro and then make our staging model in the data warehouse

10. Create a new file within the office_weather directory called all_weather_data.sql ("/models/office_weather/all_weather_data.sql")

11. Update all_weather_data.sql ("/models/office_weather/all_weather_data.sql") to select from the stg_weather_data model

    ```sql
    select *
    from {{ ref('stg_weather_data') }}
    ```

12. Configure all_weather_data.sql ("/models/office_weather/all_weather_data.sql") to be an incremental model by adding a config block to the start of the file. Specify that the combination of office and time is what conveys uniqueness

    ```sql
    {{
        config(
            materialized='incremental',
            unique_key=['office','time']
        )
    }}
    select *
    from {{ ref('stg_weather_data') }}
    ```

13. Update all_weather_data.sql ("/models/office_weather/all_weather_data.sql") to include an is_incremental block to compare the time field to the maximum value of the time field from its prior state

    ```sql
    {{
        config(
            materialized='incremental',
            unique_key=['office','time']
        )
    }}
    select *
    from {{ ref('stg_weather_data') }}

    {% if is_incremental() %}
        -- this filter will only be applied on an incremental run
        where time > (select max(time) from {{ this }})
    {% endif %}
    ```

14. In the command bar, run the command

    ```sh
    dbt build --select office_weather
    ```

    To make all our office_weather models. stg_weather_data and all_weather_data should both be built.

15. In the command bar, run the command

    ```sh
    dbt build --select all_weather_data
    ```

    To make the all_weather_data model in the data warehouse. You should see the incremental where clause present in the logs.

16. In the command bar, run the command

    ```sh
    dbt build --select all_weather_data --full-refresh
    ```

    To make the all_weather_data model in the data warehouse. Now you should not see the incremental where clause in the logs.

### [Back to guide list](../ReadMe.md)

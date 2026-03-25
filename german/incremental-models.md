# Inkrementelle Models

1. Erstellen Sie eine neue Datei namens `fetch_weather.sql` im macros-Verzeichnis (`/macros/fetch_weather.sql`).

2. Kopieren Sie den SQL-Code aus der Starterdatei `fetch_weather.sql`

   ```sql
   {% macro fetch_weather() %}
       {% do run_query("CALL DBT_COURSE.WEATHER.DBT_FETCH_WEATHER('"+ target.schema +"');")%}
   {% endmacro %}
   ```

   Dies führt die Prozedur dbt_fetch_weather() aus und erstellt eine Tabelle im Schema, in dem wir arbeiten

3. Erstellen Sie einen neuen Ordner im models-Verzeichnis namens office_weather (`/models/office_weather`)

4. Erstellen Sie eine `_schema.yml`-Datei im office_weather-Verzeichnis (`/models/office_weather/_schema.yml`) und kopieren/einfügen Sie den YAML-Code aus der Starterdatei `_schema.yml`

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

5. Erstellen Sie `stg_weather_data.sql` im office_weather-Verzeichnis (`/models/office_weather/stg_weather_data.sql`)

6. Schreiben Sie folgende Abfrage in die `stg_weather_data.sql`-Datei (`/models/office_weather/stg_weather_data.sql`), um unsere Quelldaten weather_readings vorzubereiten

   ```SQL
   select *
   from {{ source('office_weather', 'weather_readings') }}
   ```

7. Aktualisieren Sie die `_schema.yml`-Datei (`/models/office_weather/_schema.yml`), um einen config-Block mit einem pre-hook hinzuzufügen, der das `fetch_weather()`-Makro ausführt, bevor das Modell `stg_weather_data` ausgeführt wird

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

8. Aktualisieren Sie die `dbt_project.yml`-Datei (`/dbt_project.yml`), um alles im office_weather-Ordner standardmäßig als View zu materialisieren

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
   <summary>vollständige dbt_project.yml</summary>

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

9. Führen Sie den Befehl in der Befehlszeile aus

   ```sh
   dbt build --select office_weather
   ```

   Um unser Makro auszuführen und anschließend unser Staging-Modell im Data Warehouse zu erstellen

10. Erstellen Sie eine neue Datei im office_weather-Verzeichnis namens all_weather_data.sql (`/models/office_weather/all_weather_data.sql`)

11. Aktualisieren Sie all_weather_data.sql (`/models/office_weather/all_weather_data.sql`), um aus dem Modell stg_weather_data zu nutzen

    ```sql
    select *
    from {{ ref('stg_weather_data') }}
    ```

12. Konfigurieren Sie all_weather_data.sql (`/models/office_weather/all_weather_data.sql`) als inkrementelles Modell, indem Sie am Anfang der Datei einen config-Block hinzufügen. Geben Sie an, dass die Kombination von office und time Eindeutigkeit gewährleistet

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

13. Aktualisieren Sie all_weather_data.sql (`/models/office_weather/all_weather_data.sql`), um einen is_incremental-Block hinzuzufügen, der das time-Feld mit dem maximalen Wert des time-Feldes aus dessen vorherigem Zustand vergleicht

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

14. Führen Sie den Befehl in der Befehlszeile aus

    ```sh
    dbt build --select office_weather
    ```

    Um alle office_weather-Modelle zu erstellen. Sowohl stg_weather_data als auch all_weather_data sollten erstellt werden.

15. Führen Sie den Befehl in der Befehlszeile aus

    ```sh
    dbt build --select all_weather_data
    ```

    Um das all_weather_data-Modell im Data Warehouse zu erstellen. Sie sollten die inkrementelle Where-Klausel in den Logs sehen.

16. Führen Sie den Befehl in der Befehlszeile aus

    ```sh
    dbt build --select all_weather_data --full-refresh
    ```

    Um das all_weather_data-Modell im Data Warehouse zu erstellen. Jetzt sollte die inkrementelle Where-Klausel nicht mehr in den Logs sichtbar sein.

## [Zurück zum Inhaltsverzeichnis](readme.md)

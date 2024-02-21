# Transformando datos en información
En esta segunda parte del proyecto de análisis del Mercado Laboral de los Analistas de Datos en Latinoamérica y USA, usamos el dataset obtenido en el proyecto DA1: Linkedin Scraper. Las tareas en este caso consisten en hacer las transformaciones necesarias para facilitar su analisis y extraer información relevante.
En primer lugar utilizo un notebook de Jupiter. En estos primeros pasos que detallo a continuación limpio y preparo los datos para posteriormente importarlos ya preprocesados en PowerBI.
En segundo lugar, construyo un tablero en PowerBi que permite explorar los datos interactivamente.

> Al final del artículo comparto el link al repositorio con el código completo de la limpieza de los datos y el tablero de PowerBI.

## Herramientas 
Para la limpieza de los datos uso las librerías pandas y json. Esta última es porque hay datos que los tomamos como diccionarios así que me resulta conveniente trabajarlos como json.

## Pasos de la limpieza

### Traducción y creación de nuevas columnas
Si prestamos atención al dataset, vemos que la columna "criteria" es la que contiene los detalles de cada vacante. También vemos que algunas vacantes tienen sus datos en idioma Inglés y otras Portugués. Por esto, la primera transformación que hago es una traducción de esos casos al Español.

En el mismo paso creo nuevas columnas en el dataset, una para cada item del detalle de la vacante.

```python
# Extract from column "criteria" each value and take it to its own column
def extract_criteria(x, crit):
    trans = {
        "Seniority level": "Nivel de antigüedad",
        "Employment type": "Tipo de empleo",
        "Job function": "Función laboral",
        "Industries": "Sectores",
        "Nível de experiência": "Nivel de antigüedad",
        "Tipo de emprego": "Tipo de empleo",
        "Função": "Función laboral",
        "Setores": "Sectores"
        }

    for orig, dest in trans.items():
        x = x.replace(orig, dest)

    d = json.loads(x.replace("\'", "\""))

    exists = []
    for item in d:
        exists.append(crit in item in d)

    if bool(True in exists):
        pos = exists.index(True)
        return d[pos][crit]
    
jobs["seniority_level"] = jobs["criteria"].apply(extract_criteria, crit="Nivel de antigüedad")
jobs["employment_type"] = jobs["criteria"].apply(extract_criteria, crit="Tipo de empleo")
jobs["job_function"] = jobs["criteria"].apply(extract_criteria, crit="Función laboral")
jobs["industries"] = jobs["criteria"].apply(extract_criteria, crit="Sectores")

jobs.info()
```

### Tratamiento de nulos y duplicados
En este caso, como nos interesa indagar en los detalles de la vacante, aquellas que no tengan datos en las columnas relevantes directamente las excluyo.
Por otra parte, también elimino las publicaciones de vacantes duplicadas.

También eliminamos aquellas vacantes que por algún motivo se filtraron en nuestro dataset, pero que no corresponden a vacantes relevantes para el análisis. Para eso nos quedamos con las que en su título tienen las palabras Analista y Datos, en los 3 idiomas.

Siguiendo el notebook, vemos que de aproximadamente de 1.700 registros iniciales, a este punto hemos depurado los datos hasta quedarnos con alrededor de 1.100 vacantes relevantes para analizar. 

```python
# Remove records with missing and duplicates
jobs = jobs.drop(columns=["location", "criteria", "salary"])

jobs.drop_duplicates(subset=jobs.drop(columns="link").columns, inplace=True)

jobs["description"].replace("N/A", None, inplace=True)
jobs = jobs.dropna(axis=0, subset="description")

# Keep only relevant job titles
jobs = jobs.loc[jobs["title"].str.contains("Analyst|Analista|Data|Datos|Dados", case=False, regex=True)]

jobs.reset_index(drop=True, inplace=True)

jobs.info()
```

### Agrupación de datos con distinto nombre y Creación temporal de nuevas columnas
Analizando el dataset, se ve que las mismas tecnologías a veces aparcen escritas de diferente manera en distintas vacantes aunque se refieran a la misma, por ejemplo Power BI y PowerBI. Esto es algo que tenemos que solucionar, y por eso creo unos diccionarios en la que las claves son los nombres de las tecnologías y los valores son las distintas formas en que pueden aparecer en el dataset. Luego creo una columna para cada tecnología e indico si aparece nombrada en la descripción de la vacante.
Por último, doy el formato definitivo a la tabla que contiene los skills con los que venimos trabajando.

```python
# Define skills to be analyzed, and how can they be found in text
tech_stack_dicts = {
    "Visualization": {
        "Power BI": ["Power BI","PowerBI"],
        "Microstrategy": "Microstrategy",
        "Tableau": "Tableau",
        "Qlik Sense": ["Qlik Sense", "Qliksense", "Qlikview", "Qlik View"],
        "Klipfolio": "Klipfolio",
        "Looker": "Looker",
        "Zoho": "Zoho",
        "Domo": "Domo"
        },
    "ETL" : {
        "Excel": ["Excel ", "Excel,"],
        "Python": "Python",
        "Bash": "Bash",
        "Perl": "Perl",
        "PySpark": "PySpark"
        },
    "Data Base": {
        "MySQL": "MySQL",
        "SQL": ["SQL, ", " SQL "],
        "SQLite": "SQLite",
        "PostgreSQL": "Postgre",
        "MariaDB": "MariaDB",
        "Redis": "Redis",
        "MongoDB": "MongoDB",
        "Oracle": "Oracle",
        "Firebase": "Firebase",
        "Elasticsearch": "Elasticsearch",
        "Cassandra": "Cassandra",
        "DynamoDB": "DynamoDB",
        "IBM Db2": "Db2"
        },
    "Data Warehouse": {
        "Snowflake": "Snowflake",
        "AWS": ["Redshift", "AWS", "Amazon Web Services", "S3"],
        "Synapse": "Synapse",
        "Firebolt": "Firebolt",
        "IBM Netezza":"Netezza",
        "Vertica": " Vertica ",
        "Databricks": "Databricks",
        "Oracle Exadata": "Exadata",
        "Google Cloud": ["Google Cloud", "BigQuery"]
    }
}

skills = pd.DataFrame.from_dict(tech_stack_dicts, orient="index").stack().to_frame()

# Check if the stack is in the description
def includes_stack(x, to_check):
    if x is not None:
        return (str.casefold(to_check) in str.casefold(x))

# Variation when stack has several ways to be written
def includes_stack_list(x, to_check):
    if x is not None:
        result = []
        for item in to_check:
            result.append((str.casefold(item) in str.casefold(x)))
        return any(result)

# Add a column for each stack, filling with True if the stack is included in the job description. False if not.
for skill_list, skill in skills[0].items():
    if not isinstance(skill, list):
        jobs[skill_list[1]] = jobs["description"].apply(includes_stack, to_check=skill)
    else:
        jobs[skill_list[1]] = jobs["description"].apply(includes_stack_list, to_check=skill)

# Format skills df
skills.reset_index(inplace=True)
skills.rename(columns={"level_0": "type", "level_1": "skill", 0: "writing"}, inplace=True)
```

### Exportación de los datos transformados para su uso en Power BI
El último paso de estas trasnformaciónes es generar los archivos .csv que luego se utilizan en Power BI para armar el tablero que nos permita analizar interactivamente la información obtenida. Para eso, genero las tablas correspondientes una para cada "entidad": compañías, países, sectores, presencialidad, habilidades, y por supuesto las vacantes.

```python
# Export dfs for use in Power BI

# Columns to extract as separate table
entities = ["country", "company", "industries"]

# Extract columns and export to csv 
def replace_entities(x, df):
    if x is not None:
        return df[df == x].index[0]

for entity in entities:
    df = jobs[entity]
    df.drop_duplicates(inplace=True)
    df = df.reset_index(drop=True)
    df.to_csv(r"Datasets\{}.csv".format(entity), index_label="{}_id".format(entity))
    jobs[entity] = jobs[entity].apply(replace_entities, df=df)


skills.to_csv(r"Datasets\skill_type.csv", columns=["type" ,"skill"], sep=",", index_label="skill_id")

# Separate skills and jobs into different tables
jobs_skills = jobs.iloc[:,11:]
jobs_skills.to_csv(r"Datasets\skills.csv", sep=",", index_label="job_id")

jobs = jobs.iloc[:,:11]
jobs.to_csv(r"Datasets\jobs.csv", sep=",", index_label="job_id")
```


## Resultado Final
Con el tablero terminado, podemos indagar sobre las búsquedas de Analista de Datos.

<img src="images\Tablero_general.PNG" alt="Tablero - Solapa General" style="height: 570px; width:1000px;"/>
<br>

<img src="images\Tablero_skills.PNG" alt="Tablero - Solapa Skills" style="height: 570px; width:1000px;"/>

[**Link al repositorio**](https://github.com/JuanPabloVeliz/LinkedIn_DA)
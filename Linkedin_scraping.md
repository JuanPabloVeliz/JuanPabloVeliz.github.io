# Web Scraping en LinkedIn

Este artículo describe las primeras tareas de un proyecto más grande. En esta primera parte usamos un scraper escrito en Python aprovechando la librería BeautifulSoup para extraer las búsquedas activas en LinkedIn para el puesto de Analista de Datos en varios países de Latam + USA. Esto nos permite en poco tiempo obtener un dataset con los datos precisos más de 1.000 vacantes relevantes. 
A continuación explico las secciones más interesantes que aplican particularmente al scraping en LinkedIn. Te recomiendo que sigas esta lectura con el script completo a mano, ya que en este artículo sólo cito algunos bloques.

>Al final del artículo comparto el link al repositorio del proyecto que incluye este scraper.


## Qué necesitamos saber para hacer un buen scraping
Antes de empezar creo importante aclarar que este documento asume que el lector tiene nociones básicas de HTML y sabe cómo acceder a inspeccionar el código de un sitio mediante un navegador.
Obviamente, también son necesarias nociones de programación y particularmente conocimiento del lenguaje Python.
Independientemente de lo anterior, recomiendo revisar la documentación de BeautifulSoup, que es muy completa y explicativa.

## Analizando la URL de la búsqueda de empleos de LinkedIn
El primer paso para hacer el web scraping es asegurarnos de que estamos haciendo el request a la URL correcta.
La URL de la búsqueda de empleos de LinkedIn se ve de la siguiente forma:

https://www.linkedin.com/jobs/search/?currentJobId=3416472661&f_WT=1&geoId=100446943&keywords=data%20analyst&sortBy=R&start=

Mirando cuidadosamente, encontramos varios parámetros y jugando un poco con la página, podemos confirmar claramente lo siguiente:

* currentJobId=341647266
  
Se genera automáticamente cada vez que hacemos una búsqueda en LinkedIn, no nos interesa.

* f_WT=1
  
Indica el tipo de presencialidad. 
1 = Presencial
2 = Remoto
3 = Híbrido

* geoId=100446943

Indica el país donde se está buscando. Otra vez, jugando con la página podemos ver que a cada país le corresponde un número distinto, y así podemos obtener el o los que nos interesen.

* keywords=data%20analyst

Indica el texto que se está buscando.

* sortBy=R

Indica cómo se ordenan los resultados de la búsqueda. Por defecto es "R", que indica "Relevancia". Está ok para mi gusto.

* start=

Habrás notado que los resultados se presentan en páginas generalmente de 25 items cada una. Este parámetro indica el número de orden general del primer resultado que se muestra en la página actual y cambia cada vez que cambiamos de página, incrementándose en 25.

Identificar estos parámetros es muy importante porque nos permite crear un código reutilizable y personalizar las búsquedas a incluir en el scraping mediante la creación de URLs desde nuestro código. Y de eso se tratan las primera líneas en las que defino la posición a buscar (parámetro "keywords") y creo unos diccionarios que contienen los valores que me interesan para los países (parámetro "geoId") y el tipo de presencialidad (parámetro f_WT).

```python

## Job position to search
position = "Data Analyst"

countries_id = {
    "Argentina": "100446943", 
    "Brazil": "106057199",
    "Colombia": "100876405",
    "México": "103323778",
    "Chile": "104621616",
    "USA": "103644278",
    "Uruguay": "100867946",
    "Paraguay": "104065273",
    "Bolivia": "104379274",
    "Ecuador": "106373116",
    "Perú": "102927786"
    }

onsite_remote = {
    "Onsite": "1",
    "Remote": "2",
    "Hibrid": "3"
    }
```

Así, con unos loops anidados podemos generar las URL para las vacantes de cada país en cada modalidad de presencialidad. Esta URL es a la cual vamos a hacer los request y trabajar con BeautifulSoup.

```python
for country_name, country_id in countries_id.items():
    local = country_id
    
    for wfh_type, wfh_id in onsite_remote.items():
        wfh = wfh_id

        raw_link = f"https://www.linkedin.com/jobs/search/?f_WT={wfh}&geoId={local}&keywords={position}&refresh=true&sortBy=R&start="
        
```

## Analizando la estructura de la página de resultados de búsqueda y extrayendo los datos generales
Al hacer una búsqueda de empleos en LinkedIn e inspeccionar la página de resultado, vemos que las vacantes aparecen en una lista en la parte izquierda de la pantalla. Podemos extraerlas utilizando el método `find_all()`:

```python
jobs = soup_jobs_list.find_all(
    "div", 
    class_="base-card relative w-full hover:no-underline focus:no-underline base-card--link base-search-card base-search-card--link job-search-card"
    )
```

A cada vacante le corresponde una tarjeta que contiene los datos que nos interesan. Por ejemplo, el link a la página individual de la vacante, el título de la posición, la empresa y otros, que podemos extraer utilizando `find()`:

```python
job_link = job.find("a", class_="base-card__full-link")["href"]
job_title = job.find("h3", class_="base-search-card__title").text.strip()
job_company = job.find("h4", class_="base-search-card__subtitle").text.strip()
```

Hay más información para extrer, como se ve en el script completo, pero creo que con estos ejemplos se entiende la idea.

## Analizando la estructura de la página individual de la vacante
El link de la vacante que obtuvimos en el paso anterior podemos utilizarlo para hacer un nuevo request y profundizar el scraping.

```python
response_job = requests.get(job_link)
soup_job = BeautifulSoup(response_job.content, "html.parser")
```

Inspeccionando esta nueva página, vemos que podemos extraer la descripción completa del puesto y otra información adicional:

```python
job_description = soup_job.find("div", class_="show-more-less-html__markup show-more-less-html__markup--clamp-after-5").text.strip() if soup_job.find("div", class_="show-more-less-html__markup show-more-less-html__markup--clamp-after-5") is not None else "N/A"

job_criteria = []

criteria_items = soup_job.find_all("li", class_="description__job-criteria-item")

for item in criteria_items:
    item_name = item.find("h3", class_="description__job-criteria-subheader").text.strip()
    item_def = item.find("span", class_="description__job-criteria-text description__job-criteria-text--criteria").text.strip()
    job_criteria.append({item_name:item_def})
 ```                       

Gracias a los loops, este proceso se va a repetir y extraer todos los datos que las empresas hayan proporcionado sobre las vacantes. Sólo falta definir cómo la vamos a almacenar.

## Guardando los datos en un archivo .csv
El resto del código que aún no comenté está relacionado con la creación del .csv donde guardamos la información. 

Esta pieza de código está casi al principio del script e implica la creación del archivo y la escritura del encabezado:

```python
with open("Datasets/linkedin-jobs.csv", mode="w", encoding="UTF-8") as file:
    writer = csv.writer(file)
    
    writer.writerow(["country", "title", "company", "location", "onsite_remote", "salary",
                     "description", "criteria", "posted_date","link"])
```

Y esta al final es simplemente para escribir la información de cada variables en la columna correspondiente:

```python
writer.writerow([country_name, job_title, job_company, job_location, wfh, job_salary,
                job_description, job_criteria, job_date, job_link])
```

## Próximos pasos
Para mí, el próximo paso es modelar el dataset que acabamos de obtener en vistas de crear un tablero en Power BI que permita visualizar de forma integral y relevante todos estos datos.
Mientras tanto, te animo a que utilices esta base para hacer el scraping que se ajuste a tus necesidades modificando los filtros a tu gusto. Además, hay que tener en cuenta que LinkedIn puede ofrecer opciones para las búsquedas que yo no exploré y pueden resultar útiles en otras circunstancias.
Por último, aclaro que este script fue escrito y probado en Enero del año 2023. Si el sitio de LinkedIn cambiara con posterioridad, será necesario actualizarlo para ajustarse al nuevo modelo.


[**Link al repositorio**](https://github.com/JuanPabloVeliz/LinkedIn_DA)


Steps for fine tuning gpt-3.5:
In order to fine tune we have to have our custom dataset. Since the model will be used in querying sql database, each row of training dataset will have the structure similar to this:
```
{ “message” : [{ "role": "system", "content": “custom info 1 here ”
        { "role": "user", "content": “custom info 2 here ”
                   { "role": "assistant", "content": “custom info 3 here ” ]}
```
We have to structure our data in this format in a python dictionary to be able to generate the desired and required jsonl format.

Understanding the contents of the training dataset: 

We need three things to create the dataset in the above mentioned format:
1. System prompt : contains information about database + initial instruction
2. User prompt : contains the natural language query user is supposed to enter
3. Assistant prompt : contains the guided thought process (tables and columns associated with the query) + valid SQL query for the  specific prompt that the user entered

System prompt is very specific in our case. The initial instruction will start like:
```
You are an assistant that is an expert in generating sqlite SQL queries.
Having the access to database content, generate a correct sqlite SQL query for the given question.
### Database content ###
```
Given the database file, firstly we are to analyze all the tables and the relationships between them. After that we will have to include the information about the schema of the database we are working with in a specific format like:
```
CREATE TABLE management (
	“Column_name” datatype_of_column, 
	“Column_name” datatype_of_column,
	PRIMARY KEY ("Column_name", "Column_name"), 
	FOREIGN KEY("Column_name") REFERENCES head ("Column_name"), 
	FOREIGN KEY("Column_name") REFERENCES department ("Column_name")
)
/*
Columns in management and all categories for low cardinality columns :
department_ID : 7, 15, 2, 11
head_ID : 5, 4, 6, 3, 10
temporary_acting : Yes, No
*/
```
Inside the /* */ section after the schema info, we have to give three valid sample values from the table. We have to provide values for both low and high cardinality columns for each of the columns of each of the tables. High Cardinality refers to the situation when a column has too many unique values. On the other hand, low cardinality means a column having only some specific unique values ( generally if the number of unique value for a certain column is < 20 we consider it a low cardinality column, high otherwise )
For example :
 ```
CREATE TABLE category (
	“category_id” INTEGER NOT NULL, 
	“name” VARCHAR(25) NOT NULL, 
       “active” bool NOT NULL,
	“last_update” TIMESTAMP NOT NULL, 
	PRIMARY KEY (“category_id”)
)
/*
Columns in category and 3 examples in each column for high cardinality columns :
category_id : 1, 16, 13
name : Family, Sci-Fi, Action
*/
/*
Columns in category and all categories for low cardinality columns :
Active: true, false
last_update : 2006-02-15 04:46:27
*/
```
An example of the content of an entire system prompt should look like:
```
You are an assistant that is an expert in generating sqlite SQL queries.
Having the access to database content, generate a correct sqlite SQL query for the given question.
### Database content ###

CREATE TABLE management (
	"department_ID" INTEGER, 
	"head_ID" INTEGER, 
	temporary_acting TEXT, 
	PRIMARY KEY ("department_ID", "head_ID"), 
	FOREIGN KEY("head_ID") REFERENCES head ("head_ID"), 
	FOREIGN KEY("department_ID") REFERENCES department ("Department_ID")
)
/*
Columns in management and all categories for low cardinality columns :
department_ID : 7, 15, 2, 11
head_ID : 5, 4, 6, 3, 10
temporary_acting : Yes, No
*/
 
CREATE TABLE category (
	category_id INTEGER NOT NULL, 
	name VARCHAR(25) NOT NULL, 
	last_update TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL, 
	PRIMARY KEY (category_id)
)
/*
Columns in category and 3 examples in each column for high cardinality columns :
category_id : 1, 16, 13
name : Family, Sci-Fi, Action
*/
/*
Columns in category and all categories for low cardinality columns :
last_update : 2006-02-15 04:46:27
*/

CREATE TABLE chip_model (
	"Model_name" TEXT, 
	"Launch_year" REAL, 
	"RAM_MiB" REAL, 
	"ROM_MiB" REAL, 
	"Slots" TEXT, 
	"WiFi" TEXT, 
	"Bluetooth" TEXT, 
	PRIMARY KEY ("Model_name")
)
/*
Columns in chip_model and 3 examples in each column for high cardinality columns :
Model_name : X30 mid-range, X50 Advanced, X51 mid-range
*/
/*
Columns in chip_model and all categories for low cardinality columns :
Launch_year : 2002.0, 2005.0, 2004.0, 2003.0
RAM_MiB : 32.0, 64.0
ROM_MiB : 48.0, 256.0, 128.0, 32.0, 64.0
Slots : 1CFII,1SD, 1SD
WiFi : 802.11b, No
Bluetooth : 1.2, Yes, No, 1.1
*/
 
CREATE TABLE head (
	"head_ID" INTEGER, 
	name TEXT, 
	born_state TEXT, 
	age REAL, 
	PRIMARY KEY ("head_ID")
)
/*
Columns in head and all categories for low cardinality columns :
head_ID : 1, 2, 5, 7, 8, 4, 6, 3, 10, 9
name : Jeff Maggert, Pádraig Harrington, Billy Mayfair, K. J. Choi, Dudley Hart, Sergio García, Stewart Cink, Tiger Woods, Nick Faldo, Franklin Langham
born_state : Delaware, Connecticut, Alabama, California, Florida
age : 69.0, 67.0, 68.0, 53.0, 56.0, 52.0, 50.0, 43.0
*/
 
CREATE TABLE mountain (
	"Mountain_ID" INTEGER, 
	"Name" TEXT, 
	"Height" REAL, 
	"Prominence" REAL, 
	"Range" TEXT, 
	"Country" TEXT, 
	PRIMARY KEY ("Mountain_ID")
)
/*
Columns in mountain and all categories for low cardinality columns :
Mountain_ID : 1, 2, 5, 7, 4, 6, 3
Name : Ngaliema / Mt Stanley (Margherita Pk), Mount Kenya (Lenana), Kibo (Uhuru Pk), Ngaliema / Mt Stanley (Savoia Pk), Mount Kenya (Batian), Duwoni / Mt Speke (Vittorio Emanuele Pk), Mawenzi (Hans Meyer Pk)
Height : 5109.0, 5199.0, 5895.0, 4890.0, 4985.0, 4977.0, 5148.0
Prominence : 720.0, 850.0, 3951.0, 3825.0, 130.0, 5885.0, 110.0
Range : Kilimanjaro, Mount Kenya, Rwenzori
Country : DR Congo Uganda, Uganda, Tanzania, Kenya
*/
 
CREATE TABLE journalist (
	"journalist_ID" INTEGER, 
	"Name" TEXT, 
	"Nationality" TEXT, 
	"Age" TEXT, 
	"Years_working" INTEGER, 
	PRIMARY KEY ("journalist_ID")
)
/*
Columns in journalist and 3 examples in each column for high cardinality columns :
journalist_ID : 1, 11, 2
Name : Jack Meaney, Herbert Swindells, Jackie Waring
*/
/*
Columns in journalist and all categories for low cardinality columns :
Nationality : Northern Ireland, Wales, England
Age : 37, 28, 25, 33, 34, 43, 27, 29
Years_working : 1, 5, 7, 8, 21, 6, 3, 12, 10, 9
*/
 
CREATE TABLE list (
	"LastName" TEXT, 
	"FirstName" TEXT, 
	"Grade" INTEGER, 
	"Classroom" INTEGER, 
	PRIMARY KEY ("LastName", "FirstName")
)
/*
Columns in list and 3 examples in each column for high cardinality columns :
LastName : HOUTCHENS, GELL, FLACHS
FirstName : RAY, EMILE, PATRINA
Classroom : 109, 110, 106
*/
/*
Columns in list and all categories for low cardinality columns :
Grade : 1, 2, 5, 4, 6, 3, 0
*/

User prompt  is easy to understand. Let's say it is:
Find the films with a rental rate greater than $5.
For this input, the Assistant Prompt will be:
To construct the query, I'll be working with the following tables: film.
From these tables, I'll be using the following columns: rental_rate.
The SQL query I'll be generating is:
SELECT * FROM film WHERE rental_rate > 5.0;
```
Therefore one entire sample of the data will look like:
```
{
    "messages": [
        { "role": "^system^", "content":"^You are an assistant that is an expert in generating sqlite SQL queries.
Having the access to database content, generate a correct sqlite SQL query for the given question.
### Database content ###
 
CREATE TABLE public.actor
(
    "actor_id" INTEGER NOT NULL,
    "first_name" TEXT NOT NULL, - first name of actor
    "last_name" TEXT NOT NULL, - last name of actor
    "last_update" timestamp NOT NULL, - when this table public.actor was last updated 
    PRIMARY KEY ("actor_id")
);
/*
Columns in actor and 3 examples in each column for the high cardinality columns :
actor_id: 1 , 2 , 3
first_name : "Penelope", "Nick", "Ed"
last_name : "Guiness", "Wahlberg", "Chase"
*/
/*
Columns in actor and all categories for the low cardinality columns :
last_update : "2013-05-26 14:47:57.62"
*/

CREATE TABLE public.city
(
    "city_id" INTEGER NOT NULL,
    "city" TEXT NOT NULL, - name of city
    "country_id" smallint NOT NULL,
    "last_update" timestamp NOT NULL, - when this table public.city was last updated
    PRIMARY KEY ("city_id"),
    FOREIGN KEY ("country_id") REFERENCES public.country ("country_id")
);
/*
Columns in city and 3 examples in each column for the high cardinality columns :
city_id: 2 , 3, 4
city : "Abha", "Abu Dhabi", "Acua"
country_id: 82 , 101, 60
*/
/*
Columns in city and all categories for the low cardinality columns :
last_update: "2006-02-15 09:45:25"
*/

CREATE TABLE public.film
(
    "film_id" INTEGER NOT NULL,
    "title" TEXT NOT NULL, - title of the film
    "description" TEXT, - description of the film
    "release_year" TEXT, - release year of the film
    "language_id" smallint NOT NULL,
    "rental_duration" smallint NOT NULL, - rental duration of the film
    "rental_rate" numeric(4, 2) NOT NULL DEFAULT 4.99, rental rate of the film
    "length" smallint, - length of the film
    "replacement_cost" numeric(5, 2) NOT NULL, - replacement cost of the film
    "rating" mpaa_rating, - rating of the film
    "last_update" timestamp NOT NULL, - when this table public.film was last updated
    "special_features" TEXT[], - special features of the film
    "fulltext" TEXT NOT NULL, - fulltext of the film as keywords
    PRIMARY KEY ("film_id"),
    FOREIGN KEY ("language_id") REFERENCES public.language ("language_id")
);
/*
Columns in film and 3 examples in each column for the high cardinality columns :
customer_id: 133, 384, 8
title: "Chamber Italian", "Grosse Wonderful", "Airport Pollock"
length: 117, 49, 54
fulltext: "'chamber':1 'fate':4 'husband':11 'italian':2 'monkey':16 'moos':8 'must':13 'nigeria':18 'overcom':14 'reflect':5", 
          "'australia':18 'cat':8 'drama':5 'epic':4 'explor':11 'gross':1 'moos':16 'must':13 'redeem':14 'wonder':2" ,
          "'airport':1 'ancient':18 'confront':14 'epic':4 'girl':11 'india':19 'monkey':16 'moos':8 'must':13 'pollock':2 'tale':5"
*/
/*
Columns in film and all categories for the low cardinality columns :
rental_rate: 2.99, 4.99, 0.99
release_year: 2006
rating: "PG", "R", "NC-17", "PG-13", "G"
last_update: "2013-05-26 14:50:58.951"
special_features :
"{Trailers,Commentaries,""Deleted Scenes""}",
"{Trailers,""Deleted Scenes""}",
"{Trailers,Commentaries,""Behind the Scenes""}",
"{""Deleted Scenes""}",
"{Commentaries}",
"{Trailers,Commentaries,""Deleted Scenes"",""Behind the Scenes""}",
"{Trailers,""Behind the Scenes""}",
"{Commentaries,""Behind the Scenes""}",
"{""Behind the Scenes""}",
"{Trailers,Commentaries}",
"{Commentaries,""Deleted Scenes""}",
"{Trailers}",
"{Commentaries,""Deleted Scenes"",""Behind the Scenes""}",
"{Trailers,""Deleted Scenes"",""Behind the Scenes""}",
"{""Deleted Scenes"",""Behind the Scenes""}"
*/

CREATE TABLE public.inventory
(
    "inventory_id" INTEGER NOT NULL,
    "film_id" smallint NOT NULL,
    "store_id" smallint NOT NULL,
    "last_update" timestamp NOT NULL, - when this table public.inventory was last updated
    PRIMARY KEY ("inventory_id"),
    FOREIGN KEY ("film_id") REFERENCES public.film ("film_id") 
);
/*
Columns in inventory and 3 examples in each column for the high cardinality columns :
inventory_id: 3 , 5, 6
film_id: 1, 1, 1
store_id: 1, 2, 2
*/
/*
Columns in inventory and all categories for the low cardinality columns :
last_update: "2006-02-15 10:09:17"
*/

CREATE TABLE public.store
(
    "store_id" INTEGER NOT NULL,
    "manager_staff_id" smallint NOT NULL,
    "address_id" smallint NOT NULL,
    "last_update" timestamp NOT NULL, - when this table public.store was last updated
    PRIMARY KEY ("store_id"),
    FOREIGN KEY ("address_id") REFERENCES public.address ("address_id"),
    FOREIGN KEY ("manager_staff_id") REFERENCES public.staff ("staff_id")
);
/*
Columns in store and 2 examples in each column for the high cardinality columns :
store_id: 1, 2
manager_staff_id: 1, 2
*/
/*
Columns in store and all categories for the low cardinality columns :
last_update: "2006-02-15 09:57:12"
*/
^"
}, { "role": "^user^", "content": "^Find the films with a rental rate greater than $5.^" }, { "role": "^assistant^", "content": "^To
construct the query, I'll be working with the following tables: film.
From these tables, I'll be using the following columns: all columns.
The SQL query I'll be generating is:
SELECT * FROM film WHERE rental_rate > 5.0;^" }
    ]
} 
{ “message” : [ {“role” : “^system^” , “content” :  ... This is the second sample . . . and so on . . . 
```
We will keep all our data in this above-mentioned structure in a text file. Carefully note that we are using “^” symbol which encloses the items inside the “content” for each of the “role”s. We do this in order to extract these from text using regex and later on prepare the jsonl. 

Lets say we have all the dataset ready in this manner in preprocess.txt and we run this to create the “messages” which is a list of dictionaries. Each dictionary inside messages have one sample of the dataset.
```
import re
import json

file_path = './preprocess.txt'

# Open the file, store its contents as a string
with open(file_path, 'r') as file:
    file_contents = file.read()

# extract all roles and contents within "^"
pattern = r'{\s*"role":\s*"\^(.*?)\^",\s*"content":\s*"\^(.*?)\^"[,\s*}]'

# find all matches
matches = re.findall(pattern, file_contents, re.DOTALL)

# Create a list to store matched info in dictionaries 
data = []

for match in matches:
    role = match[0]
    content = match[1]
    data.append({"role": role, "content": content.strip()})

j = 0
temp_list = []
messages = [] # this is the accumulated that we need finally
for item in data:
    if j%3 ==0:
        # system
        temp_list.append(item) 
    if j%3 ==1:
        # user 
        temp_list.append(item) 
    if j%3 ==2:
        # assistant 
        temp_list.append(item) 
        message = { "message" : temp_list }
        messages.append(message)
        temp_list = []
    j=j+1
# we can print to check the structure of the data converting it to json
# json_data = json.dumps(messages, indent=4)
# print(json_data)


# we convert the ‘message’ object into jsonl
import jsonlines


with jsonlines.open('ready-for-training.jsonl', 'w') as writer:
    writer.write_all(messages)

```

Now we start the fine tuning job.

![ft](https://github.com/raiyan22/chatSQL/assets/58294098/66fd30ce-9bdf-409e-96da-f78a6bb8e843)

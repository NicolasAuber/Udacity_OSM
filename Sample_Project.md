# Project - OSM Reims Data Analysis
Author: Nicolas Auber  
Date: 24 November 2020

## Table of Contents
- Introduction
- Input File
- Data Wrangling
- Data Analysis
- Conclusion

## Introduction
This project is part of Data Analyst Nanodegree Progam. It will deal with the analysis of OpenStreetMap data focusing on Reims city in France. The reason is simple: Reims is my hometown and I am still curious about this city.  
Reims, France  
https://www.openstreetmap.org/relation/36458

## Input Files
The full map of Reims area is downloaded on [OpenStreetMap](https://www.openstreetmap.org). A sample is extracted from the full map to run the Python script against it.  
The size of both OSM files is given below.
```
Size of the OSM files:
Full file:  Reims.osm  >  136.7 Mo
Sample file: ReimsSample.osm  >  5.5 Mo
```

## Data Wrangling
The most important part of the data analyst job is to audit and clean the dataset based on several criteria: validity of the data, accuracy, completeness, consistency, uniformity, ... After the data cleaning, we will be able to explore the data on Reims area. 

### Data Auditing
A few statistics are calculated in order to know the data set better and maybe identify any problems in the dataset.
- Count of the unique users
- Count of the different types of tag
- Count of the different types of attributes
- Count of the 4 pre-defined types of value for the attribute k
- Count of the different types of value for the attribute k
- Count of the different values of postcode

e.g. count of the different values of postcode
```python 
def count_postcodes(filename):
    postcodes = {}
    for event, elem in ET.iterparse(filename, events=('start', 'end')):
        if event == 'end':
            key = elem.attrib.get('k')
            if key == 'addr:postcode':
                postcode = elem.attrib.get('v')
                if postcode not in postcodes:
                    postcodes[postcode] = 1
                else:
                    postcodes[postcode] += 1
    return postcodes

postcodes = count_postcodes(MY_FILE)
print("Number of occurrences:", sum(postcodes.values()))
print("Number of occurrences per postcode:")
sorted_dict = [(k, v) for (v, k) in sorted([(value, key) for (key, value) in postcodes.items()], reverse=True)]
pprint.pprint(sorted_dict)
```

Number of occurrences: 2971  
Number of occurrences per postcode:  
51100, 1954  
51430, 434  
51370, 302  
51450, 188  
51350, 78  
51109, 7  
51688, 3  
51092, 2  
51721, 1  
51687, 1  
51 100, 1  

### Data Problems
After the data audit, the dataset will be cleaned regarding:
- The postal codes
- The city names
- The street names

It will correct the following problems:
- Postal code validity i.e. with the expected 5 digits pattern
- Postal code accuracy i.e. meets the official postal code for cities (verified against https://www.laposte.fr/particulier/outils/trouver-un-code-postal)
- City name validity/accuracy i.e. with the expected official city name
- City name uniformity i.e. with the right text format

### Other Ideas of Improvement
Other problems, which could modify our data analysis, will not be dealt with in that project:
- The postal codes and city names completeness. Indeed, during the audit, it has been observed some missing data. 
- The validity of all the tags in accordance with https://wiki.openstreetmap.org/wiki/Map_Features  
Other data will not be cleaned as they are not analyzed. For instance phone numbers.

### Data Cleaning
The dataset is cleaned against the problems highlighted above. For instance, the following code is used to process the street names.
```python 
mapping_street = {"rue": "Rue",
					"bis rue": "Rue",
					"impasse": "Impasse",
					"route": "Route",
					"avenue": "Avenue",
					"vernouillet": "Rue Vernouillet"
					}
```
```python 
def update_streetname(value, mapping):
	after = []
	for part in value.split(" "):
		if part in mapping.keys():
			part = mapping[part]
		after.append(part)
	better_name = " ".join(after)
	return better_name
```

After the data is cleaned, the dataset is assumed to be reliable.

Therefore, the CSV files are generated applying the data cleaning code above. But the CSV files are also processed in order to remove all the information not related to Reims i.e. with a postal code not equal to 51100.

```python 
'''
The function drop_df_rows removes the rows of the CSV file containing data in the neighbouring cities.
This is done in 2 steps:
- First, any id with an associated postcode not equal to '51100' is added to a list
- Second, all the rows whose id is in that list are removed
Indeed, the idea is not to remove only the row with the other postcode but to remove all the data related to the 
other postcodes.
The number of rows removed is printed as well.
'''
def drop_df_rows(filename):
	df = pd.read_csv(filename, encoding = 'utf-8')
	before = len(df)
	df_list = df[(df.type == 'addr') & (df.key == 'postcode') & (df.value != '51100')]
	ids = []
	ids = df_list['id'].tolist()
	df = df.query("id not in @ids")
	after = len(df)
	print("{} rows removed in {} file.".format((before - after), filename))
	df.to_csv(filename, index = False, encoding = 'utf-8')

drop_df_rows('nodes_tags.csv')
drop_df_rows('ways_tags.csv')
```

## Data Analysis
First, a SQL databse named Reims.db is created. Many queries are made within the full analysis (refer to Jupyter Notebook file). A few queries on that database are detailed below.

### Number of Nodes and Ways
```sql
c.execute('SELECT COUNT(*) FROM nodes')
results = c.fetchall()
print('Number of nodes in the database:', results[0][0])

c.execute('SELECT COUNT(*) FROM ways')
results = c.fetchall()
print('Number of ways in the database:',results[0][0])
```
Number of nodes in the database: 521617  
Number of ways in the database: 93338

### Top 10 contributors
```sql
QUERY = '''
SELECT DISTINCT user, COUNT(*)
FROM (SELECT id, user, uid FROM nodes UNION ALL SELECT id, user, uid FROM ways) element
GROUP BY user
ORDER BY COUNT(*) DESC
LIMIT 10;
'''
c.execute(QUERY)
results = c.fetchall()
print("Top 10 contributors:")
pprint.pprint(results)
```
Top 10 contributors (edited):  
MXS, 181226  
MM51, 120721  
Ours51, 87495  
Denis_Helfer, 66151  
Reims Moderne, 46160  
botdidier2020, 26082  
Super-Map, 14509  
AmiFritz, 10762  
Hugo-D, 8343  
meihou, 4703  

### Top 10 types of cuisine in restaurants
```sql
QUERY = '''
SELECT value, COUNT(*) as count
FROM (SELECT * FROM nodes_tags UNION ALL SELECT * FROM ways_tags) tags
WHERE key = 'cuisine'
GROUP BY value
ORDER BY count DESC
LIMIT 10;
'''
c.execute(QUERY)
results = c.fetchall()
print("Top 10 types of cuisine:")
pprint.pprint(results)
```
Top 10 types of cuisine (edited):  
french, 22  
burger, 17  
pizza, 14  
italian, 9  
brasserie, 6  
asian, 5  
chinese, 5  
kebab, 4  
mexican, 4  
japanese, 3  

### Top of the brands of fast food chains
```sql
QUERY = '''
SELECT value, COUNT(*) as count
FROM (SELECT * FROM nodes_tags UNION ALL SELECT * FROM ways_tags) tags
JOIN (SELECT DISTINCT(id) FROM (SELECT * FROM nodes_tags UNION ALL SELECT * FROM ways_tags) tags WHERE value = 'fast_food') as sub
ON tags.id = sub.id
WHERE key = 'brand'
GROUP BY value
ORDER BY Count DESC;
'''
c.execute(QUERY)
results = c.fetchall()
print("Top brands of fast food restaurants:")
pprint.pprint(results)
```
Top brands of fast food restaurants (edited):  
McDonald's, 6  
Burger King, 2  
Quick, 2  
231 East Street, 1  
Bagelstein, 1  
Domino's, 1  
Domino's Pizza, 1  
KFC, 1  
O'Tacos, 1  
Pita Pit, 1  
Subway, 1  

## Conclusion
After this review, we can conclude that the OpenStreetMap data is quite complete and cleaned for Reims city. I was curious about Reims city and I found out a few interesting results.  
However, the data can always be improved. The following ideas could be put in place:
- Make the OSM community more active in Reims. Maybe we could contact the top contributors to see what could be done.  
- Like Udacity, Reims university could propose a course on OpenStreetMap inviting students to work on an OSM project asking them to improve the data
- Like other cities, Reims Métropole could make data on Reims available to everyone. This will be a win-win opportunity: OSM contributors could incporate these data into OSM and Reims would be able to use them for its statistics.
- Using INSEE (Instit National de la Statistique et des Etudes Economiques) data to verify the OSM data against the SIRENE database or the BAN database (recently used by OSM map community). These data are available here: (https://www.data.gouv.fr/fr/territories/commune/51454@1970-12-30/Reims/)

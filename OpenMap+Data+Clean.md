
##Map Area

San Jose, CA, USA

https://mapzen.com/data/metro-extracts/metro/san-jose_california/

I just relocated to San Jose and I am very interested in getting to know more about this area. Besides, I also want to contribute to its improvement.

#look at a smaller sample of the original data.


```python
import xml.etree.ElementTree as ET  # Use cElementTree or lxml if too slow

OSM_FILE = "san-jose_california.osm"  # Replace this with your osm file
SAMPLE_FILE = "sample.osm"

k = 10 # Parameter: take every k-th top level element

def get_element(osm_file, tags=('node', 'way', 'relation')):
    """Yield element if it is the right type of tag

    Reference:
    http://stackoverflow.com/questions/3095434/inserting-newlines-in-xml-file-generated-via-xml-etree-elementtree-in-python
    """
    context = iter(ET.iterparse(osm_file, events=('start', 'end')))
    _, root = next(context)
    for event, elem in context:
        if event == 'end' and elem.tag in tags:
            yield elem
            root.clear()


with open(SAMPLE_FILE, 'wb') as output:
    output.write('<?xml version="1.0" encoding="UTF-8"?>\n')
    output.write('<osm>\n  ')

    # Write every kth top level element
    for i, element in enumerate(get_element(OSM_FILE)):
        if i % k == 0:
            output.write(ET.tostring(element, encoding='utf-8'))

    output.write('</osm>')
```


```python
my_filename = "san-jose_california.osm"


def count_tags(filename):
    # YOUR CODE HERE
    tags = {}
    for event, element in ET.iterparse(filename):
        if element.tag not in tags.keys():
            tags[element.tag] = 1
        else:
            tags[element.tag] += 1
    return tags

tags = count_tags(my_filename)

print tags

##update the street name titles:

import xml.etree.cElementTree as ET
from collections import defaultdict
import re
import pprint

street_type_re = re.compile(r'\b\S+\.?$', re.IGNORECASE)


expected = ["Street", "Avenue", "Boulevard", "Drive", "Court", "Place", "Square", "Lane", "Road",
            "Trail", "Parkway", "Commons"]




def audit_street_type(street_types, street_name):
    m = street_type_re.search(street_name)
    if m:
        street_type = m.group()
        if street_type not in expected:
            street_types[street_type].add(street_name)


def is_street_name(elem):
    return (elem.attrib['k'] == "addr:street")


def is_postal_code(elem):
    return (elem.attrib['k'] == "addr:postcode")

def audit_street_types(osmfile):
    osm_file = open(osmfile, "r")
    street_types = defaultdict(set)
    for event, elem in ET.iterparse(osm_file, events=("start",)):

        if elem.tag == "node" or elem.tag == "way":
            for tag in elem.iter("tag"):
                if is_street_name(tag):
                    audit_street_type(street_types, tag.attrib['v'])
    osm_file.close()
    return street_types

def audit_postal_code(osmfile):
    osm_file=open(osmfile, "r")
    postal_code=set()
    for event, elem in ET.iterparse(osmfile, events=("start",)):
        if elem.tag=="node" or elem.tag=="way":
            for tag in elem.iter("tag"):
                if is_postal_code(tag):
                    postal_code.add(tag.attrib['v'])
    osm_file.close()
    return postal_code


```

    {'node': 1694589, 'nd': 1984372, 'bounds': 1, 'member': 19907, 'tag': 724176, 'relation': 2056, 'way': 232385, 'osm': 1}


A sample node:
        <node changeset="15964174" id="2243704240" lat="37.375257" lon="-122.0373861" timestamp="2013-05-03T18:47:56Z" uid="318696" user="n76" version="2">
                <tag k="addr:city" v="Sunnyvale" />
                <tag k="addr:state" v="CA" />
                <tag k="addr:street" v="Florence Street" />
                <tag k="source:addr" v="survey" />
                <tag k="addr:country" v="US" />
                <tag k="addr:postcode" v="94086" />
                <tag k="addr:housenumber" v="325" />
        </node>

##Problems Encountered in the Map

1.Street abbreviations: abbreviations are used, such as 'Rd': set(['Mt Hamilton Rd']), 'Dr': set(['Fountain Oaks Dr', '1490 S Park Victoria Dr']).

2. Unfamiliar street types: Unknown type exists, such as '0.1': set(['Ala 680 PM 0.1']), 'Alameda': set(['The Alameda']),  'Oaks': set(['North Fair Oaks']), 'Real': set(['El Camino Real', 'West El Camino Real', 'East El Camino Real']), 'East': set(['Vanderbilt Court East', 'Park Circle East', 'Rio Robles East']),'Barcelona': set(['Calle de Barcelona']),Madrid': set(['Corte de Madrid']), 'Seville': set(['Corte de Seville']).


3.Wrongly identified street type: 'CA': set(['Zanker Rd., San Jose, CA']). 

4. Postal codes: state name(s) are included, such as 'CA 94035'. Most postal code are in the form of "95014-3422" but some contain only the part before dash, such as '95133', '95131', '95136', '95135', '95134', '95138'.


#Abbreviations
Once the data was imported to SQL, there would be some name inconsistencies. To deal with correcting street names, I used a regular expression to match and then correct them using the following function:




```python

mapping = { "St": "Street",
            "St.": "Street",
            "Rd.": "Road",
            "Ave": "Avenue",
            "Dr":"Drive",
           "Blvd": "boulevard"    
            }

def update_name(name, mapping):

    # YOUR CODE HERE
    m = street_type_re.search(name)
    better_name=name
    if m:
        if m.group() in mapping.keys():
            better_street_type=mapping[m.group()]
            better_name=street_type_re.sub(better_street_type, name)

    return better_name

osm_file = open("san-jose_california.osm", "r")
for event, elem in ET.iterparse(osm_file, events=("start",)):
    if elem.tag == "node" or elem.tag == "way":
        for tag in elem.iter("tag"):
            if is_street_name(tag):
                update_name(tag.attrib['v'], mapping)
osm_file.close()


```

#Unknow street types

1. Real: The road that linked the missions, pueblos, and presidios in early California was called El Camino Real. Therefore, even though it is rare, it is a type of street. 
2. Ala 680 PM 0.1 is the Digital Archaeological Record. Users will not be ablt to read it. So it should be removed from the record. 
3. The Alameda is a historic street in San Jose and Santa Clara, California. It is weired to call it a "street type" in most areas but it is a feature in San Jose. So I will keep it. 
4. North Fair Oaks is a San Mateo County in California. It is 30 minute-drive away from San Jose. Probably this metro extract is about “San Jose Metropolitan Area” for its inclusion of surrounding cities.  
5. East is the extention of a road to the east. Many "East Road" exist in California so it can be kept as a street type or include it in the street name. 
6. 'Barcelona': set(['Calle de Barcelona']),Madrid': set(['Corte de Madrid']), 'Seville': set(['Corte de Seville']) are all Spanish street names because there are lots of Mexican people in California and many Spanish street names. Calle means street, corte mean court. Their types are not Barcelona, Madrid, or Seville. Yet they are valid street names. 

#Wrongly identified street type

I revised the address and remove the redundant information.



```python
osm_file = open("san-jose_california.osm", "r")
for event, elem in ET.iterparse(osm_file, events=("start",)):
    if elem.tag == "node" or elem.tag == "way":
        for tag in elem.iter("tag"):
            if is_street_name(tag) and tag.attrib['v']=='Zanker Rd., San Jose, CA':
                tag.attrib['v']='Zanker Road'
osm_file.close()

```

#Postal code
In order to keep postal code consistency and keep as much information as possible, I revised the postal code with state names but kept 5 digit postal code as well as 9 digit postal code with a dash.



```python
osm_file = open("san-jose_california.osm", "r")
for event, elem in ET.iterparse(osm_file, events=("start",)):
    if elem.tag == "node" or elem.tag == "way":
        for tag in elem.iter("tag"):
            if tag.attrib['v']=='CA 94035':
                tag.attrib['v']='94035'
osm_file.close()
    
```


```python
##Write parsed data into cvs files

import csv
import codecs
import pprint
import re
import xml.etree.cElementTree as ET

import cerberus

import schema

OSM_PATH = "san-jose_california.osm"

NODES_PATH = "nodes.csv"
NODE_TAGS_PATH = "nodes_tags.csv"
WAYS_PATH = "ways.csv"
WAY_NODES_PATH = "ways_nodes.csv"
WAY_TAGS_PATH = "ways_tags.csv"

LOWER_COLON = re.compile(r'^([a-z]|_)+:([a-z]|_)+')
PROBLEMCHARS = re.compile(r'[=\+/&<>;\'"\?%#$@\,\. \t\r\n]')

SCHEMA = schema.schema

# Make sure the fields order in the csvs matches the column order in the sql table schema
NODE_FIELDS = ['id', 'lat', 'lon', 'user', 'uid', 'version', 'changeset', 'timestamp']
NODE_TAGS_FIELDS = ['id', 'key', 'value', 'type']
WAY_FIELDS = ['id', 'user', 'uid', 'version', 'changeset', 'timestamp']
WAY_TAGS_FIELDS = ['id', 'key', 'value', 'type']
WAY_NODES_FIELDS = ['id', 'node_id', 'position']


def shape_element(element, node_attr_fields=NODE_FIELDS, way_attr_fields=WAY_FIELDS,
                  problem_chars=PROBLEMCHARS, default_tag_type='regular'):
    """Clean and shape node or way XML element to Python dict"""

    node_attribs = {}
    way_attribs = {}
    way_nodes = []
    tags = []  # Handle secondary tags the same way for both node and way elements
    tags2 = []

    position=0
    # YOUR CODE HERE
    if element.tag == 'node':
        for attr in element.attrib:
            #If attributes are in FIELDS
            if attr in NODE_FIELDS:
                node_attribs[attr] = element.attrib[attr]
        #Iterate through all sub-tags for element
        for sub in element:
            if sub.tag == 'tag':
                dicc={}
                for i in NODE_TAGS_FIELDS:
                    if i=='id':
                        dicc[i]=node_attribs['id']
                    elif i=='key':
                        if len(sub.attrib['k'].split(":"))==1:
                            dicc[i]=sub.attrib['k']
                        elif len(sub.attrib['k'].split(":"))==2:
                            dicc[i]=sub.attrib['k'].split(":")[1]
                        elif len(sub.attrib['k'].split(":"))==3:
                            dicc[i]=sub.attrib['k'].split(":")[1]+sub.attrib['k'].split(":")[2]

                    elif i=='value':
                        dicc[i]=sub.attrib['v']
                    else:
                        if len(sub.attrib['k'].split(":"))==1:
                            dicc[i]="regular"
                        elif len(sub.attrib['k'].split(":"))==2:
                            dicc[i]=sub.attrib['k'].split(":")[0]
                        elif len(sub.attrib['k'].split(":"))==3:
                            dicc[i]=sub.attrib['k'].split(":")[0]
                tags.append(dicc)
                #print tags
        #print {'node': node_attribs, 'node_tags': tags}
        return {'node': node_attribs, 'node_tags': tags}
        
    elif element.tag == 'way':
        for attr in element.attrib:
            #If attributes are in FIELDS
            if attr in WAY_FIELDS:
                way_attribs[attr] = element.attrib[attr]
        #Iterate through all sub-tags for element
        for sub in element:
            if sub.tag == 'tag':
                dicc2={}
                for i in WAY_TAGS_FIELDS:
                    if i=='id':
                        dicc2[i]=way_attribs['id']
                    elif i=='key':
                        if len(sub.attrib['k'].split(":"))==1:
                            dicc2[i]=sub.attrib['k']
                        elif len(sub.attrib['k'].split(":"))==2:
                            dicc2[i]=sub.attrib['k'].split(":")[1]
                        elif len(sub.attrib['k'].split(":"))==3:
                             dicc2[i]=sub.attrib['k'].split(":",1)[1]##+sub.attrib['k'].split(":")[2]
                    elif i=='value':
                        dicc2[i]=sub.attrib['v']
                    else:
                        if len(sub.attrib['k'].split(":"))==1:
                            dicc2[i]="regular"
                        elif len(sub.attrib['k'].split(":"))==2:
                            dicc2[i]=sub.attrib['k'].split(":")[0]
                        elif len(sub.attrib['k'].split(":"))==3:
                            dicc2[i]=sub.attrib['k'].split(":")[0]
                tags2.append(dicc2)

 
            elif sub.tag=='nd':
                dicc3={}
                dicc3['id']=way_attribs['id']
                dicc3['node_id']=sub.attrib['ref']
                dicc3['position']=position
                way_nodes.append(dicc3)
                position+=1
                
    #print way_nodes


        return {'way': way_attribs, 'way_nodes': way_nodes, 'way_tags': tags2}
        



# ================================================== #
#               Helper Functions                     #
# ================================================== #
def get_element(osm_file, tags=('node', 'way', 'relation')):
    """Yield element if it is the right type of tag"""

    context = ET.iterparse(osm_file, events=('start', 'end'))
    _, root = next(context)
    for event, elem in context:
        if event == 'end' and elem.tag in tags:
            yield elem
            root.clear()


def validate_element(element, validator, schema=SCHEMA):
    """Raise ValidationError if element does not match schema"""
    if validator.validate(element, schema) is not True:
        field, errors = next(validator.errors.iteritems())
        message_string = "\nElement of type '{0}' has the following errors:\n{1}"
        error_string = pprint.pformat(errors)
        
        raise Exception(message_string.format(field, error_string))


class UnicodeDictWriter(csv.DictWriter, object):
    """Extend csv.DictWriter to handle Unicode input"""

    def writerow(self, row):
        super(UnicodeDictWriter, self).writerow({
            k: (v.encode('utf-8') if isinstance(v, unicode) else v) for k, v in row.iteritems()
        })

    def writerows(self, rows):
        for row in rows:
            self.writerow(row)


# ================================================== #
#               Main Function                        #
# ================================================== #
def process_map(file_in, validate):
    """Iteratively process each XML element and write to csv(s)"""

    with codecs.open(NODES_PATH, 'w') as nodes_file, \
         codecs.open(NODE_TAGS_PATH, 'w') as nodes_tags_file, \
         codecs.open(WAYS_PATH, 'w') as ways_file, \
         codecs.open(WAY_NODES_PATH, 'w') as way_nodes_file, \
         codecs.open(WAY_TAGS_PATH, 'w') as way_tags_file:

        nodes_writer = UnicodeDictWriter(nodes_file, NODE_FIELDS)
        node_tags_writer = UnicodeDictWriter(nodes_tags_file, NODE_TAGS_FIELDS)
        ways_writer = UnicodeDictWriter(ways_file, WAY_FIELDS)
        way_nodes_writer = UnicodeDictWriter(way_nodes_file, WAY_NODES_FIELDS)
        way_tags_writer = UnicodeDictWriter(way_tags_file, WAY_TAGS_FIELDS)

        nodes_writer.writeheader()
        node_tags_writer.writeheader()
        ways_writer.writeheader()
        way_nodes_writer.writeheader()
        way_tags_writer.writeheader()

        validator = cerberus.Validator()

        for element in get_element(file_in, tags=('node', 'way')):
            el = shape_element(element)
            if el:
                if validate is True:
                    validate_element(el, validator)

                if element.tag == 'node':
                    nodes_writer.writerow(el['node'])
                    node_tags_writer.writerows(el['node_tags'])
                elif element.tag == 'way':
                    ways_writer.writerow(el['way'])
                    way_nodes_writer.writerows(el['way_nodes'])
                    way_tags_writer.writerows(el['way_tags'])


if __name__ == '__main__':
    # Note: Validation is ~ 10X slower. For the project consider using a small
    # sample of the map when validating.
    process_map(OSM_PATH, validate=False)
```


```python
##import c
import sqlite3
import csv
from pprint import pprint
```


```python
sqlite_file = 'mydb.db'    # name of the sqlite database file

# Connect to the database
conn = sqlite3.connect(sqlite_file)
cur = conn.cursor()
cur.execute('''DROP TABLE IF EXISTS nodes_tags''')
conn.commit()
cur.execute('''
    CREATE TABLE nodes_tags(id INTEGER, key TEXT, value TEXT,type TEXT)
''')
# commit the changes
conn.commit()
with open('nodes_tags.csv','rb') as fin:
    dr = csv.DictReader(fin) # comma is default delimiter
    to_db = [(i['id'], i['key'],i['value'].decode("utf-8"), i['type']) for i in dr]
    
# insert the formatted data
cur.executemany("INSERT INTO nodes_tags(id, key, value,type) VALUES (?, ?, ?, ?);", to_db)
# commit the changes
conn.commit()

# cur.execute('SELECT * FROM nodes_tags')
# all_rows = cur.fetchall()
# print('1):')
# pprint(all_rows)
conn.close()

```


```python
##nodes:
conn = sqlite3.connect(sqlite_file)
cur = conn.cursor()
cur.execute('''DROP TABLE IF EXISTS nodes''')
conn.commit()
cur.execute('''
    CREATE TABLE nodes (
    id INTEGER PRIMARY KEY NOT NULL,
    lat REAL,
    lon REAL,
    user TEXT,
    uid INTEGER,
    version INTEGER,
    changeset INTEGER,
    timestamp TEXT
)
''')
# commit the changes
conn.commit()
with open('nodes.csv','rb') as fin:
    dr = csv.DictReader(fin) # comma is default delimiter
    to_db = [(i['id'], i['lat'],i['lon'], i['user'].decode("utf-8"), i['uid'], i['version'], i['changeset'], i['timestamp'].decode("utf-8")) for i in dr]
    
# insert the formatted data
cur.executemany("INSERT INTO nodes(id, lat, lon, user, uid, version, changeset, timestamp) VALUES (?, ?, ?, ?, ?, ?, ?, ?);", to_db)
# commit the changes
conn.commit()

# cur.execute('SELECT * FROM nodes')
# all_rows = cur.fetchall()
# print('2):')
# pprint(all_rows)
conn.close()

```


```python
##ways:
conn = sqlite3.connect(sqlite_file)
cur = conn.cursor()
cur.execute('''DROP TABLE IF EXISTS ways''')
conn.commit()
cur.execute('''
CREATE TABLE ways (
    id INTEGER PRIMARY KEY NOT NULL,
    user TEXT,
    uid INTEGER,
    version TEXT,
    changeset INTEGER,
    timestamp TEXT
)
''')
# commit the changes
conn.commit()
with open('ways.csv','rb') as fin:
    dr = csv.DictReader(fin) # comma is default delimiter
    to_db = [(i['id'], i['user'].decode("utf-8"), i['uid'], i['version'], i['changeset'], i['timestamp'].decode("utf-8")) for i in dr]
    
# insert the formatted data
cur.executemany("INSERT INTO ways(id, user, uid, version, changeset, timestamp) VALUES (?, ?, ?, ?, ?, ?);", to_db)
# commit the changes
conn.commit()

# cur.execute('SELECT * FROM ways')
# all_rows = cur.fetchall()
# print('3):')
# pprint(all_rows)
conn.close()

```


```python
##ways_tags

conn = sqlite3.connect(sqlite_file)
cur = conn.cursor()
cur.execute('''DROP TABLE IF EXISTS ways_tags''')
conn.commit()
cur.execute('''
CREATE TABLE ways_tags (
    id INTEGER NOT NULL,
    key TEXT NOT NULL,
    value TEXT NOT NULL,
    type TEXT,
    FOREIGN KEY (id) REFERENCES ways(id)
)
''')
# commit the changes
conn.commit()
with open('ways_tags.csv','rb') as fin:
    dr = csv.DictReader(fin) # comma is default delimiter
    to_db = [(i['id'], i['key'].decode("utf-8"), i['value'].decode("utf-8"), i['type'].decode("utf-8")) for i in dr]
    
# insert the formatted data
cur.executemany("INSERT INTO ways_tags(id, key, value, type) VALUES (?, ?, ?, ?);", to_db)
# commit the changes
conn.commit()

# cur.execute('SELECT * FROM ways_tags')
# all_rows = cur.fetchall()
# print('4):')
# pprint(all_rows)
conn.close()
```


```python
##ways_nodes
conn = sqlite3.connect(sqlite_file)
cur = conn.cursor()
cur.execute('''DROP TABLE IF EXISTS ways_nodes''')
conn.commit()
cur.execute('''
CREATE TABLE ways_nodes (
    id INTEGER NOT NULL,
    node_id INTEGER NOT NULL,
    position INTEGER NOT NULL,
    FOREIGN KEY (id) REFERENCES ways(id),
    FOREIGN KEY (node_id) REFERENCES nodes(id)
)
''')
# commit the changes
conn.commit()
with open('ways_nodes.csv','rb') as fin:
    dr = csv.DictReader(fin) # comma is default delimiter
    to_db = [(i['id'], i['node_id'], i['position']) for i in dr]
    
# insert the formatted data
cur.executemany("INSERT INTO ways_nodes(id, node_id, position) VALUES (?, ?, ?);", to_db)
# commit the changes
conn.commit()

# cur.execute('SELECT * FROM ways_nodes')
# all_rows = cur.fetchall()
# print('5):')
# pprint(all_rows)
conn.close()
```

##Data Overview and Additional Ideas

This section contains information regarding the data in database.


#File sizes



```python
from pprint import pprint
import os
from hurry.filesize import size

dirpath = '/Users/wenjia.ma/Udacity/data-wrangling/openMap/csv'

files_list = []
for path, dirs, files in os.walk(dirpath):
    files_list.extend([(filename, size(os.path.getsize(os.path.join(path, filename)))) for filename in files])
files_list.sort(key = lambda letter: ''.join([i for i in letter[1] if not i.isdigit()]), reverse=True)
files_list.sort(key=lambda size: int(size[1].translate(None, "MKB")), reverse=True)


for filename, size in files_list:
    print '{:.<40s}: {:5s}'.format(filename,size)
```

    nodes.csv...............................: 134M 
    ways_nodes.csv..........................: 45M  
    ways_tags.csv...........................: 20M  
    ways.csv................................: 13M  
    .DS_Store...............................: 6K   
    nodes_tags.csv..........................: 2M   


#Number of nodes



```python
import sqlite3
conn = sqlite3.connect(sqlite_file)
c = conn.cursor()

query = '''SELECT COUNT(*) as count FROM nodes '''
# execute the query
nodes = c.execute(query) 

# print the results
print list(nodes)
```

    [(1694589,)]


#Number of ways


```python
query = '''SELECT COUNT(*) as count FROM ways '''
ways=c.execute(query)
print list(ways)
```

    [(232385,)]


#Number of unique users




```python
query='''SELECT COUNT(DISTINCT(e.uid))          
FROM (SELECT uid FROM nodes UNION ALL SELECT uid FROM ways) e '''
uids=c.execute(query)
print list(uids)
```

    [(1382,)]


#Top 10 contributing users




```python
query='''SELECT e.user, COUNT(*) as num
FROM (SELECT user FROM nodes UNION ALL SELECT user FROM ways) e
GROUP BY e.user
ORDER BY num DESC
LIMIT 10'''
top10users=c.execute(query)
print list(top10users)
```

    [(u'andygol', 295540), (u'nmixter', 284607), (u'mk408', 146712), (u'Bike Mapper', 91296), (u'samely', 81069), (u'RichRico', 76147), (u'dannykath', 74371), (u'MustangBuyer', 65006), (u'karitotp', 63396), (u'Minh Nguyen', 52894)]


#Number of users who contributes only once


```python
query='''SELECT COUNT(*) 
FROM
    (SELECT e.user, COUNT(*) as num
     FROM (SELECT user FROM nodes UNION ALL SELECT user FROM ways) e
     GROUP BY e.user
     HAVING num=1)  u'''
onceusers=c.execute(query)
print list(onceusers)
```

    [(302,)]


##Additional Data Exploration



#Top 10 appearing amenities




```python
query='''SELECT value, COUNT(*) as num
FROM nodes_tags
WHERE key='amenity'
GROUP BY value
ORDER BY num DESC
LIMIT 10'''
popamenity=c.execute(query)
print list (popamenity)
```

    [(u'restaurant', 868), (u'fast_food', 420), (u'bench', 312), (u'cafe', 256), (u'bicycle_parking', 200), (u'place_of_worship', 168), (u'toilets', 159), (u'school', 143), (u'bank', 126), (u'fuel', 124)]


#Popular restaurant cuisines


```python
query='''SELECT nodes_tags.value, COUNT(*) as num
FROM nodes_tags 
    JOIN (SELECT DISTINCT(id) FROM nodes_tags WHERE value='restaurant') i
    ON nodes_tags.id=i.id
WHERE nodes_tags.key='cuisine'
GROUP BY nodes_tags.value
ORDER BY num DESC
LIMIT 5'''

poprestaurant=c.execute(query)
print list(poprestaurant)

```

    [(u'vietnamese', 78), (u'chinese', 67), (u'mexican', 60), (u'pizza', 56), (u'japanese', 44)]



```python
query='''SELECT tags.value, COUNT(*) as count 
FROM (SELECT * FROM nodes_tags UNION ALL 
      SELECT * FROM ways_tags) tags
WHERE tags.key LIKE '%city'
GROUP BY tags.value
ORDER BY count DESC
LIMIT 10'''

cities=c.execute(query)
print list(cities)
```

    [(u'Sunnyvale', 3421), (u'San Jose', 1034), (u'Morgan Hill', 397), (u'Santa Clara', 323), (u'Saratoga', 233), (u'San Jos\xe9', 168), (u'Los Gatos', 133), (u'Milpitas', 105), (u'Campbell', 76), (u'1', 75)]


The results prove that this map of San Jose covers larger metro san jose area. It includes many cities in addition to the city san jose. One more thing to notice is that there are 168 data showing "San Jos\xe9", which may be the Spanish spelling of san jose. 

##Conclusion

The data cleaning process helps maintain the consistency in the street types and postal codes through replacement. Some street types are unique in this San Jose area, for example, Spanish names or historical streets. I prefer to keep them as it is instead of changing them into standard format because it maintains local features. This brings problem for future improvement as well. For example, when I query the tags for city names, I found "San Jose" and "San Jos\xe9" are listed seperately. In this case, the second type can be merged into the first one. 


```python

```

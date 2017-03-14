# Wrangle OpenStreetMap Data: Zurich, Switzerland

Note: complete data files (both source `.osm` and sanitized `.json`) have been added to .gitignore to prevent slowness and bloating on GitHub. Sample source and sanitized data can be found in the `extracts` folder of this repository.

## Source data

<img src="images/zurich_map.png" width="300">

The OpenStreetMap (OSM) extract for Zurich, Switzerland contains covers the city center and surrounding suburbs, and contains data on establishments, transportation systems, points of interest and more for the area. The **[OSM XML dataset from MapZen](https://mapzen.com/data/metro-extracts/metro/zurich_switzerland/) is 614.7 MB**.

In accordance with the OSM XML API, the data consists of three main types of elements:

- Nodes
- Ways
- Relations

In the sanitization process, all three types of elements are accounted for.

### XML parsing challenges

#### Domain & language knowledge

I chose to parse Zurich data because of an interest in the city, not a personal connection. Having never visited or lived the city and not knowing German, I was initially lacking knowledge to audit tag values.

Because of special characters in German (e.g. umlaut), I needed to familiarize myself with [Python's handling of unicode](https://docs.python.org/2/howto/unicode.html).


#### City names

Top 20 matches to Zurich:

[('Zurich', 100), ('Zuerich', 92), ('zuerich', 92), ('Zürich', 91), ('zürich', 91), ('Egg bei Zürich', 82), ('Zürich-Flughafen', 78), ('Zürich 50 Oerlikon', 78), ('Zürich-Altstetten', 78), ('Zürich Gockhausen', 78), ('Muri', 68), ('Höri', 60), ('Forch', 55), ('Buchs', 55), ('Embrach', 46), ('Zumikon', 46), ('Zufikon', 46), ('Neerach', 46), ('Uerikon', 46), ('Seuzach', 46)]

#### Tag key separators

OSM records level of specificity in the key values in tags. In the json conversion process, it made the most sense to nest these attributes together, but the approach had to differ based on the tag. For the purposes of this exercise, I only processed two-tiered keys (one separator), and disregarded three-tiered keys.

Two-tier key types & conversions:
```xml
<tag k="addr:street" v="Bülach" />
<tag k="addr:street" v="Spitalstrasse" />

# converted to:
"addr": {
  "city": "Bülach",
  "street": "Spitalstrasse"
}
```
```xml
<tag k="wheelchair" v="limited"/>
<tag k="wheelchair:description" v="rund 50% der Fahrzeuge verkehren mit Niederflur-Einstieg"/>

# converted to:
"wheelchair": {
  "description": "rund 50% der Fahrzeuge verkehren mit Niederflur-Einstieg",
  "base_value": "limited"
}
```
```xml
<tag k="maxspeed" v="100"/>
<tag k="source:maxspeed" v="sign"/>

# converted to:
"maxspeed": {
  "source": "sign",
  "base_value": "100"
}
```

Also, some separators departed from the standard `:` separator, and had `.`, like `surface.material`.  

#### Tag value separators

I had to conduct research on the data to ensure that I was handling the data correctly. Take the following XML:

```xml
<tag k="destination" v="Bern;Chur;Luzern;Flughafen;Nordring-Zürich"/>
<tag k="source:maxspeed" v="sign"/>
<tag k="destination:symbol" v="airport"/>
...
<tag k="name" v="Salomon-Bleuler-Weg"/>
<tag k="highway" v="residential"/>
<tag k="maxspeed" v="30"/>
...
<tag k="bus:lanes" v="no|designated" />
```

`;`, `-`, and `|` all potentially act as separators.

The first portion with `"Bern;Chur;Luzern;Flughafen;Nordring-Zürich"` is meant to be separated, as all of those are towns/cities in Switzerland. The third is also a clear separator. Values are converted to `["Bern","Chur","Luzern","Flughafen","Nordring-Zürich"]` and `["no", "designated"]`.

Searches for `"Salomon-Bleuler-Weg"` on [other maps](https://www.google.com/maps/place/Salomon-Bleuler-Weg,+8400+Winterthur,+Switzerland/@47.4916183,8.7383565,18z/data=!4m5!3m4!1s0x479a9997ae1936e5:0xb2db2c6fd90d87eb!8m2!3d47.4915712!4d8.7395501), however, confirm that the value is indeed the full name of the street.

<img src="images/sbw.png" width="200">


## Data overview

Data filtered down to just Zurich.

### File attributes

File | Type | Size
--- | --- | ---
`zurich_switzerland.osm` | Source | 614.7 MB
`just_zurich.json` | Sanitized | 702.2 MB


### Data counts

 | Count | Query
--- | --- | ---
All | 3146959 | `db.just_zurich.count()`
Nodes | 2706650 | `db.just_zurich.find({"type":"node"}).count()`
Ways  | 432670| `db.just_zurich.find({"type":"way"}).count()`
Relations | 134 | `db.just_zurich.find({"type":"relation"}).count()`

#### Users

##### Unique users

\> `db.just_zurich.distinct("created.uid").length`

```javascript
2642
```

##### Top user

\> ` db.just_zurich.aggregate([{$group: {_id:{"user": "$created.user","uid": "$created.uid"},count:{$sum:1}}},{"$sort": {"count": -1}},{"$limit": 1}])`

```javascript
{ "_id" : { "user" : "mdk", "uid" : "178186" },
  "count" : 566235 }
```

##### Top five users vs the rest

\> ` db.just_zurich.aggregate([{$group: {_id:{"user": "$created.user","uid": "$created.uid"},count:{$sum:1}}},{"$sort": {"count": -1}},{"$limit": 5}])`

```javascript
{ "_id" : { "user" : "mdk", "uid" : "178186" }, "count" : 566235 }
{ "_id" : { "user" : "SimonPoole", "uid" : "92387" }, "count" : 334879 }
{ "_id" : { "user" : "Sarob", "uid" : "1218134" }, "count" : 146217 }
{ "_id" : { "user" : "hecktor", "uid" : "465052" }, "count" : 117316 }
{ "_id" : { "user" : "feuerstein", "uid" : "194843" }, "count" : 102162 }
```

<img src="images/users.png" width="400">


#### Dates

Type | Result | Query
--- | --- | ---
Oldest record | "2006-05-05T16:19:04Z" | `db.just_zurich.find().sort({"created.timestamp": 1}).limit(1)`
Newest record | "2017-03-11T13:48:07Z" | `db.just_zurich.find().sort({"created.timestamp": -1}).limit(1)`

### Zurich exploration

For the purposes of this exercise, I'm only considering tags that explicitly list Zurich as a city as within Zurich.

##### Number of records
\> `db.just_zurich.aggregate([{"$match": {"addr.city": {"$exists": 1}}}, {$group:{_id:null,count:{$sum:1}}}])`

```javscript
22849
```

##### What are the top 5 amenities?

\> `db.just_zurich.aggregate([
  {"$match": {$and: [{"addr.city": {"$exists": 1}}, {"amenity": {"$exists": 1}}]}},
  {$group: {_id: "$amenity",count:{$sum:1}}},
  {"$sort": {"count": -1}},
  {"$limit": 5}])`

```javascript
{ "_id" : "restaurant", "count" : 327 }
{ "_id" : "car_sharing", "count" : 212 }
{ "_id" : "school", "count" : 67 }
{ "_id" : "place_of_worship", "count" : 64 }
{ "_id" : "cafe", "count" : 52 }
```
##### Which areas of the city (by postcode) have the most diverse cuisine?

\> `db.just_zurich.aggregate([
  {$match:{$and: [{"addr.city": {"$exists": 1}},{"amenity":"restaurant"}]}},
  {$group:{_id: {postcode: "$addr.postcode"}, cuisines:{"$addToSet":"$cuisine"}}},
  { $unwind: "$cuisines" },
  { $unwind: "$cuisines" },
  {$group:{_id: "$_id.postcode", cuisines:{"$addToSet":"$cuisines"}, num_cuisines:{"$sum": 1}}},
  {$sort: {num_cuisines: -1}},
  {$limit: 5}])`


```javascript
{ "_id" : "8004", "cuisines" :
  [ "international", "chinese", "tapas", "lebanese", "italian", "indian", "kebab", "spanish", "kosher", "coffee_shop", "tea", "burger", "cake", "regional", "vegan", "japanese", "american", "Bier, Bar", "vegetarian", "asian", "vietnamese" ],
  "num_cuisines" : 22 }

{ "_id" : "8005", "cuisines" :
  [ "american", "vegetarian", "asian", "italian", "pizza", "thai", "international", "regional", "indian", "turkish", "burger", "greek", "lebanese" ], "num_cuisines" : 13 }

{ "_id" : "8001", "cuisines" :
  [ "pizza", "italian", "asian", "bistro", "indian", "brazilian", "spanish", "fish", "regional", "thai", "greek", "sushi", "japanese" ],
  "num_cuisines" : 13 }

{ "_id" : "8006", "cuisines" :
  [ "pizza", "asian", "swiss", "indian", "coffee_shop", "italian", "lebanese", "African", "thai", "international" ],
  "num_cuisines" : 10 }

{ "_id" : "8050", "cuisines" :
  [ "regional", "thai", "kebab", "indian", "italian", "steak_house", "japanese", "asian", "Schnitzel", "bagels" ],
  "num_cuisines" : 10 }
```

Note: the double "unwind" was necessary to extract values from restaurants that had more than one associated cuisine.


## Looking ahead

### Payments

Currently, the payments schema looks like the following, where keys are descriptive and values are standardized:

```javascript
"payment" : {
		"notes" : "yes",
		"coins" : "yes",
		"cash" : "yes",
		"american_express" : "yes",
		"visa_electron" : "yes",
		"visa" : "yes",
		"mastercard" : "yes",
		"maestro" : "yes"
  }
```

With this schema, it is difficult to perform aggregations in the MongoDB framework. One can really only query the payments sub-document for counts, rather than form more complex queries.

> \> `db.just_zurich.aggregate([{"$match":{$and: [{"addr.city": {"$exists": 1}},{"payment": {"$exists": 1} } ] } }, {$group:{_id: null, "num_payments": {"$sum": 1} } }, {"$sort": {num_payments: -1}}, {"$limit": 10}])`

> Returns `{ "_id" : null, "num_payments" : 25 }`

Whereas:

> \> `db.just_zurich.aggregate([{"$match":{$and: [{"addr.city": {"$exists": 1}},{"payment": {"$exists": 1} } ] } }, {$group:{_id: "$payment", "num_payments": {"$sum": 1} } }, {"$sort": {num_cuisines: -1}}, {"$limit": 10}])`

> Returns `"_id" : { "coins" : "yes", "bitcoin" : "yes" }`, rather than payment types as intended.

In the future, I'd want to sanitize the payments field so that the structure for a given document follows this schema: `{"payments": ["notes", "visa", "bitcoin"]}`.

### Limits of "addr:city"

My method for extracting Zurich-only data needs improvement. At the moment, the only filter I apply is through the "addr:city" key -- if the value doesn't fuzzy match to "Zurich", then I skip the record.

I loaded a non-filtered version of the dataset into MongoDB, and compared the counts of both collections:

Dataset | Count | Query
--- | --- | ---
Filtered | 3146959 | `db.just_zurich.count()`
Unfiltered | 6084959 | `db.all_zurich.count()`

Despite culling around 3 million records, I don't do any checks for location other than "addr:city". In the future, I may consider using records' `lat` and `lon` keys to determine whether a document falls within what I consider to be the city boundaries.

## Resources

- [OSM XML API Wiki](http://wiki.openstreetmap.org/wiki/OSM_XML#Contents)
- [Elementtree API](https://docs.python.org/2/library/xml.etree.elementtree.html)
- [Unicode & Python](https://docs.python.org/2/howto/unicode.html)
- [Permutations for "street" in German](http://www.acronymfinder.com/Stra%C3%9Fe-\(German%3A-Street\)-\(STR\).html)
- [District validator](https://www.inyourpocket.com/zurich/Zurichs-districts_71823f?&page=2)
- [Fuzzywuzzy](https://pypi.python.org/pypi/fuzzywuzzy)  string match
- [Donut chart](http://stackoverflow.com/questions/36296101/donut-chart-python)

+++
title = 'The Web of Wealth'
date = 2024-04-30T06:30:59+08:00
draft = false
math = false
+++

## Unveiling High-Net-Worth Networks with Neo4j

![Donald Trump](https://notalvin.github.io/posts/images/Donald_Trump_Graph.png)

### Introduction to Neo4j

Neo4j, a leading graph database system, can be used to efficiently maps and analyze complex data networks using nodes, relationships, and properties.

### Importance of Network Analysis

For high-net-worth individuals, understanding their intricate web of connections is paramount, offering insights crucial for business, security, and personal matters.

### Potential Use Cases

1. **Due Diligence and Risk Assessment**
   - Regulatory and reputational risks can be identified through by Neo4j's ability to uncover hidden connections, forewarning and thus safeguarding financial institutions.

2. **Personalized Marketing and Relationship Management**
   - Companies can leverage Neo4j to craft personalized offers based on high-net-worth individuals' preferences and affiliations, enhancing marketing effectiveness.

3. **Security and Privacy Enhancements**
   - Neo4j can assist in threat assessment and privacy protection by analyzing associate networks, crucial for individuals valuing discretion.

4. **Business Networking Optimization**
   - Neo4j can enable high-net-worth individuals to identify key connections for expanding business opportunities and fostering strategic partnerships.

5. **Family Office Management**
   - Family offices could utilize Neo4j to streamline operations and enhance decision-making by mapping out family members' roles, investments, and interdependencies.

## Utilizing Wealth-X data to build such a graph

Let's begin by getting the data required to build such a graph from Wealth-X, a company providing verified data on high-net-worth individuals. The choice of Wealth-X for data retrieval stems from its unparalleled collection of records on wealthy individuals.

By leveraging their clean and trustable data, we can tap into a wealth of information crucial for informed decision-making, strategic planning, and tailored engagement strategies with high-net-worth individuals. You will need a Wealth-X account with an associated username, password and API key.

```python
import pandas as pd
import requests
import json
from tqdm import tqdm

'''
Testing API Call
'''
def get_api_key():
    with open('utils/documents/config.json') as f:
        config = json.load(f)
    return config['username'], config['password'], config['api_key']

username, password, api_key = get_api_key()

headers = {'username': username,
           'password': password,
           'apikey': api_key,
           'accept': "application/json"
           }


'''
Method to GET from wealthx API and store successfully gotten and failed dossiers
'''
def call_wealthx(dossier_list):
    storage = {}
    failure = {}
    print("***** Begin extracting dossier information *****")

    # for dossier_id in dossier_list:
    for dossier_id in tqdm(dossier_list, desc="Processing Dossiers"):
        template_url = f"https://connect.wealthx.com/rest/v1/dossiers/{dossier_id}"
        try:
            raw = requests.get(template_url, headers = headers)
            dossier_json = json.loads(raw.text)
            storage[dossier_id] = dossier_json['profilebasic']
        except:
            dossier_json = {"Message":f"Failed to get Dossier ID: {dossier_id}"}
            failure[dossier_id] = dossier_json
    
    df = pd.DataFrame.from_dict(storage, orient='index')
    print("***** Done extracting dossier information *****")
    if failure:
        for key, value in failure.items():
            print(value['Message'])
    else:
        print("All dossiers successfully extracted")
    return df, failure
```

Now that we have extracted dossier data for the list of parties we're interested in, we need to write some code to import these individuals to Neo4J as nodes, and the relationships between them as edges. The following code snippets are crucial for interacting with the Neo4j database, managing node and relationship creation, and adding data from the DataFrame to the graph:

1. Function to Get Neo4j Credentials: This function retrieves the credentials (URI, username, and password) required to connect to the Neo4j database from a JSON configuration file.

#### Function that gets neo4j credentials for where the graph is stored

```python
# Function that gets neo4j credentials for where the graph is stored
def get_neo4j_credentials():
    with open('utils/documents/config.json') as f:
        config = json.load(f)
    return config['uri'], config['neo4j_username'], config['neo4j_password']
```

2. Function to Delete All Nodes and Relationships: This function refreshes the existing graph by deleting all nodes and relationships. This is important to prevent accidental duplication during the importing process.

#### Function that deletes all the nodes and edges in existing graph (Refreshes the graph before attempting to add)

```python
def delete_all_nodes_and_relationships(driver):
    # Delete all nodes and relationships
    with driver.session() as session:
        session.run("MATCH (n) DETACH DELETE n")
```

3. Function to Create Nodes in the Graph: This function iterates through the DataFrame and creates nodes in the Neo4j graph for each individual, with attributes such as name, total assets, source of wealth, etc.

#### Function that adds the individuals in the dataframe as nodes in the graph

```python
def create_graph(df, driver):
    # Create nodes with names and IDs
    with driver.session() as session:
        for index, row in df.iterrows():
            # Process dictionary columns stored as strings
            try:
                entities = literal_eval(row['positions'])
                for entity in entities:
                    if entity['isPrimary']:
                        entity_name = entity['entityName']
            except Exception:
                entity_name = "Unknown"

            try:
                total_assets = literal_eval(row['netWorth'])['netWorthLower'] #TODO Other columns might be appropriate - Check coverage
            except Exception:
                total_assets = "Unknown"

            try:
                source_of_wealth = literal_eval(row['wealthSources'])[0]
            except Exception:
                source_of_wealth = "Unknown"

            try:
                business_country = literal_eval(row['businessContact'])['country']
            except Exception:
                business_country = "Unknown"

            # Create person node with attributes
            session.run(
                "CREATE (p:Person {id: $id, name: $name, dossierCategory: $dossierCategory, entityName: $entityName, totalAssets: $totalAssets, sourceOfWealth: $sourceOfWealth, businessCountry: $businessCountry})",
                id=row['ID'],
                name=str(row['firstName']) + ' ' + str(row['middleName']) + ' ' + str(row['lastName']),
                dossierCategory=row['dossierCategory'] or "Unknown",
                entityName=entity_name,
                totalAssets=total_assets,
                sourceOfWealth=source_of_wealth,
                businessCountry=business_country
            )
```

4. Function to Create Relationships Between Nodes: This function creates relationships between nodes in the graph based on specified criteria, such as known associates, family members, or service providers. Currently we have chosen 

#### Function that creates an edge between 2 nodes in the graph, using their unique IDs as keys

```python
def create_relationship(session, from_id, to_id, relationship):
    # Check if the relationship already exists
    result = session.run(
        "MATCH (from)-[r:`" + relationship + "`]-(to) "
        "WHERE from.id = $from_id AND to.id = $to_id "
        "RETURN count(r) AS count",
        from_id=from_id,
        to_id=to_id
    )
    record = result.single()
    count = record["count"] if record else 0
    
    # If the relationship does not exist, create it from from_id to to_id
    if count == 0:
        session.run(
            "MATCH (from), (to) "
            "WHERE from.id = $from_id AND to.id = $to_id "
            f"CREATE (from)-[:{relationship}]->(to)",
            from_id=from_id,
            to_id=to_id,
        )
```

5. Functions for Node Creation and Relationship Processing: These functions handle the creation or update of nodes, as well as the processing of relationship columns from the DataFrame and adding connections to the graph.

#### Function that creates or updates a node in the graph

```python
def create_or_update_node(session, node_id, node_name, node_type, attributes=None):
    # Check if the node already exists
    result = session.run(
        "MATCH (n) WHERE n.id = $id RETURN n",
        id=node_id
    )
    if not result.single():
        # Node does not exist, create a new node
        query = "CREATE (n:" + node_type + " {id: $id, name: $name"
        if attributes is not None:
            for key, value in attributes.items():
                query += ", " + key + ": $" + key
        query += "})"
        parameters = {"id": node_id, "name": node_name}
        if attributes is not None:
            parameters.update(attributes)
            #print(query)
        session.run(query, **parameters)
```

#### Function that processes relationship columns and adds connections to the graph

```python
def process_relationship_column(session, person_id, column_name, column_data, relationship_col):
    try:
        relationships = literal_eval(column_data)
    except Exception as e:
        print(f"Error with {column_name} for person {person_id}: {column_data}")
    else:
        if isinstance(relationships, list):
            for relationship in relationships:
                try:
                    relationship_id = relationship.get('dossierID') or relationship.get('entityID')
                    relationship_name = relationship.get('dossierName') or relationship.get('entityName')
                    relationship_type = relationship.get(relationship_col) or "Unknown"
                    attributes = {
                        'dossierCategory': relationship.get('dossierCategory') or "Unknown",
                        'totalAssets': relationship.get('totalAssetFound') or "Unknown",
                    }
                    node_type = 'Person' if relationship.get('dossierID') else 'Entity'
                    create_or_update_node(session, relationship_id, relationship_name, node_type, attributes=attributes)
                    create_relationship(session, person_id, relationship_id, replace_with_single_underscore(relationship_type))
                except Exception as e:
                    print(f"Error adding {column_name}: {e}")
```

#### Function that adds connections to the graph based on DataFrame columns

```python
def add_connections_to_graph(df, driver):
    with driver.session() as session:
        for index, row in tqdm(df.iterrows(), total=df.shape[0]):
            person_id = row['ID']
            for column_name in ['knownAssociates', 'families', 'serviceProviders']:
                mapping = {'knownAssociates': "relationship", 'families': "relationship", 'serviceProviders': "position"}
                process_relationship_column(session, person_id, column_name, row[column_name], mapping[column_name])
        print('All added')
```

### Conclusion

Now that we have defined all the necessary functions to both get the raw data and import it as nodes and edges into Neo4J, let us put it all together and see what the resulting graph looks like:

```python
dossier_list = pd.read_csv('utils/documents/eu_wealth_x.csv')['Dossier ID'] # List of dossier IDs
df, failure = call_wealthx(dossier_list)
df.to_csv('utils/documents/new_full_client_data.csv') # Save for future use

# Connect to Neo4j
uri, username, password = get_neo4j_credentials()
driver = GraphDatabase.driver(uri, auth=(username, password))

# Perform the operations
delete_all_nodes_and_relationships(driver)
print('***** Graph refreshed *****')
create_graph(df, driver)
print('***** Master nodes added *****')
add_connections_to_graph(df, driver)
print('***** Connections loaded *****')
```

Now that all the nodes and edges have been loaded into Neo4j, this web of wealth can offer invaluable insights into the world of high-net-worth individuals, such as the aforementioned use cases spanning financial management, security, personalized services, and strategic planning. For example, we can see the associates and connections of 2 publicly available individuals ([Thierry Henri Stern](https://www.bloomberg.com/profile/person/16837526) from Patek Philippe and [Yeung Sau Shing](https://en.wikipedia.org/wiki/Albert_Yeung) from the Emperor Group) in our graph below, being able to tell at a glance from their attributes some noteworthy information.

![Connected Rich Folk](https://notalvin.github.io/posts/images/Thierry_Henri_Graph.png)

| Attribute           | Value               |
|---------------------|---------------------|
| dossierName         | Thierry Henri Stern |
| entityName          | Patek Philippe      |
| netWorthLower       | 660,000,000         |
| liquidLower         | 35,000,000          |
| householdNetWorth   | 690,000,000         |
| householdWealth     | 665,000,000         |
| householdLiquidAsset| 665,000,000         |
| businessCity        | Geneva              |
| businessState       | Geneva              |
| businessCountry     | Switzerland         |
| dossierCategory     | CUHNW               |
| positionHeldName    | President           |

You can tell from the connecting edge that he is related through business to Yeung Sau Shing, and upon looking at his node's attributes one is able to discover:

| Attribute           | Value               |
|---------------------|---------------------|
| dossierName         | Yeung Sau Shing     |
| dateOfBirth         | 1943-03-03          |
| dossierState        | active              |
| maritalStatus       | Married             |
| gender              | Male                |
| primaryCompany      | Emperor Group       |
| position            | Chairman            |
| netWorthLower       | 1,400,000,000       |
| liquidLower         | 930,000,000         |
| householdNetWorth   | 1,600,000,000       |
| householdWealth     | 1,600,000,000       |
| householdLiquidAsset| 930,000,000         |


Queries to filter and find linkages between individuals can be written in Cypher, a SQL-like language allowing users to unlock the full potential of property graph database. [(Documentation)](https://neo4j.com/docs/cypher-manual/current/introduction/)

Some sample queries that I used can be found below, but the possibilities are endless!
| Example  | Description  | Query                                                                                   |
|-------|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|
|a| Original graph with Bernard Ecclestone and his connections that are people (Main node's id = 46278)  | MATCH p=(a:Person)-[]-(b:Person)-[]-(c:Person) where a.id=46278 RETURN p LIMIT 1000; |
|b| Selecting individuals who were created and added manually, along with their network | MATCH p=(n:Created_Person)-[]-()-[]-()-[]-() RETURN p LIMIT 1000;                       |
|c| Filtering for connections that raise a red flag such as Prosecutors| MATCH p=()-[]-()-[:Prosecutor]->() RETURN p LIMIT 25;                                   |

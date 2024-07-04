+++
title = 'Inferring Country from Contact Information'
date = 2024-07-03T00:42:24+08:00
draft = false
math = false
+++

## Introduction
Recently I was working on the task of scraping news sites for articles and the linked companies. As part of the dashboard building process, in order to make it easier for users to filter down the displayed articles, I wanted to enhance the company data with their country of origin.

Whilst the simpler (and smarter) solution would be to use the Google Maps API in conjunction with company information to label the HQ location, due to budgetary requirements we will have to come up with some creative solutions. I will be presenting 3 of the solutions along with how they eventually got integrated into a pipeline to be applied on input scraped information.

### Data we have to work with
From the company page we have scraped, what immediately stands out as relevant information from which to derive the country of origin would be the contact information of the company, pictured below:

![Tesla Contact Information](https://notalvin.github.io/posts/images/Tesla_contact_info.png)

From this we have 5 pieces of information to work with, that being:
1. Company name
2. Address Line 1
3. Address Line 2
4. Phone number
5. Website

Within our dataframe we will store this as a dictionary, with the code below being used to extract this information from the HTML for that company page.

```python
def get_contact_information(html_content):
    result = {}
    soup = BeautifulSoup(html_content, 'html.parser')
    # Find the div with class 'card-content'
    contact_info_div = soup.find_all('div', class_ = 'card mb-15')
    for div in contact_info_div:
        try:
        # Extract the paragraphs within this div
            contact_info_paragraphs = div.find_all('p', class_='m-0')
            if contact_info_paragraphs:
                
                company_name = contact_info_paragraphs[0].text
                address_line_1 = contact_info_paragraphs[1].text
                address_line_2 = contact_info_paragraphs[2].text
                phone_number = contact_info_paragraphs[3].text

                # Extract the website URL
                website = div.find('a', class_='m-0')['href']

            # Print the extracted information
            result['Company Name'] = company_name
            result['Address Line 1'] = address_line_1
            result['Address Line 2'] = address_line_2
            result['Phone Number'] = phone_number
            result['Website'] = website
        except:
            pass
    return result
```

### Solution 1 - Using the company phone number
The first solution attempted was to the the phone number, specifically the country code at the front to label the country. Country codes are defined by the International Telecommunication Union (ITU) in ITU-T standards E.123 and E.164. The prefixes enable international direct dialing (IDD). [Wikipedia](https://en.wikipedia.org/wiki/List_of_country_calling_codes)

To implement this the first step is to find a way to map country code to the country the number belongs to, and luckily this information is widely available on the internet.

Here is the code I used to get such a mapping, provided by [https://www.countrycode.org/](https://www.countrycode.org/):

```python
def get_phone_mapping():
    url = "https://www.countrycode.org"
    response = requests.get(url)

    # Parse the HTML content
    soup = BeautifulSoup(response.text, 'html.parser')

    # Find the table
    table = soup.find('table')

    # Extract the table headers
    headers = []
    for th in table.find_all('th'):
        headers.append(th.text.strip())

    # Extract the table rows
    rows = []
    for tr in table.find_all('tr'):
        cells = tr.find_all('td')
        row = [cell.text.strip() for cell in cells]
        if row:
            rows.append(row)

    # Create a DataFrame
    df = pd.DataFrame(rows, columns=headers)[['COUNTRY', 'COUNTRY CODE']]

    storage = {}
    for i, row in df.iterrows():
        if row['COUNTRY CODE'] not in storage.keys():
            storage[row['COUNTRY CODE']] = {row['COUNTRY']}
        else:
            storage[row['COUNTRY CODE']].add(row['COUNTRY'])
    return storage
```

This gives us a dictionary of country codes mapped to countries, which means that after a bit of processing to extract the Country code from the contact number string should mean the problem is solved right? Well some issues still remain...

As you can see from the code, some codes map to multiple countries (e.g. Kazakhstan and Russia both share +7). Furthermore, the phone numbers scraped do not uniformly follow the same format, with many North American (USA/Canada) number using their state codes as their leading digits instead of the country code. Some state codes overlap with existing country codes (E.g. +212 being the country code for Morocco whilst being the telephone area code in the United States that covers the central part of Manhattan in New York City) making it difficult to simple rely on the mapping data gotten.

### Solution 2 - Using the cities in the addresses
Given the issues with the above attempt, how can we do better? Another promising angle based on the information we have is to try and use the cities mentioned in the addresses as a clue to derive what country the address comes from.
To implement this the first step is to find a way to map city names to the country the city belongs to, and luckily this information is also available on the internet. [Generously provided for free by simplemaps.com](https://simplemaps.com/data/world-cities)

Upon downloading the worldcities.csv, we can create a mapping dictionary and use that to tag the companies based on the city mentioned in their address. I used regex to find the city names since the addresses follow a common format.

```python
def get_city_mapping():
    cities = pd.read_csv('utils/data/Scrape/worldcities.csv')
    city_storage = {}
    city_storage['St. Peter Port'] = {'Guernsey'}
    for i, row in cities.iterrows():
        city = remove_diacritics(row['city'])
        if city not in city_storage.keys():
            city_storage[city] = {row['country']}
        else:
            city_storage[city].add(row['country'])
    return city_storage

def extract_city_names(lst):
    for i in range(len(lst) - 1, -1, -1):
        if re.search(r'\d', lst[i]):
            if i + 1 < len(lst):
                return ' '.join(lst[i+1:]).split(',')[-1].strip()

def label_country_by_city(contact_information, city_storage):
    country = {'Unknown'}
    try:
        temp = contact_information['Address Line 2'].split(' ')
        city = extract_city_names(temp)
        country = city_storage[city]
    except:
        country = {'Unknown'}
    return country
```

Unfortunately as above, this approach is not perfect either. Many city names are common amongst multiple countries (E.g. Paris, France & Paris, Texas), and this means that we cannot tell exactly an address comes from by just using the city name.

What can we do to solve this problem? Fret not, as our previous efforts were not in vain as the work done allows us to generate a set of candidate countries from which to draw from, narrowing down the possible countries from which we can select the hopefully correct answer using the third method.

### Solution 3 - Similarity scoring using word embeddings

Embeddings work effectively for identifying the country of an address because they transform textual data into a numerical form that captures semantic meaning and relationships. This approach works because:

1. Semantic Capture: Embeddings, created by models like BERT or Word2Vec, encode words or phrases into dense vectors. These vectors capture not just the individual meaning of words but also their contextual relationships. For addresses, this means that embeddings can understand geographic terms, city names, and country-specific postal codes within the context they appear.

2. Dimensionality Reduction: Embeddings reduce high-dimensional textual data into a manageable, dense vector space. This allows for efficient computation and comparison of textual data, such as addresses.

3. Similarity Representation: In the embedding space, similar addresses are placed closer to each other. For example, addresses from the same country will have similar place names, postal codes, and formatting, leading their embeddings to cluster together in the vector space.

4. Generalization and Context: Embeddings can generalize across different but semantically related terms. For example, different cities within the same country might have distinct names but will share linguistic and contextual patterns that embeddings can capture.

When an address is converted into an embedding, its vector representation inherently carries information about the geographic and linguistic context. By comparing these embeddings using cosine similarity, we can effectively measure how close or similar an input address is to reference embeddings of known countries. This closeness in the embedding space directly correlates with the likelihood that the address belongs to a particular country.

Whilst the solution working as per the theory above would be ideal, in the real world unfortunately it is not so simple, as the noise from the street names and numbers etc. in the address meant that quite frequently the semantically closest country (as determined by the cosine similarity) would unfortunately not be quite correct.

Additionally there is the noise that comes from similar countries, where even for a human we feel that countries like Canada and the United Kingdom would have many similar looking addresses.

```python
model_name = 'nomic-ai/nomic-embed-text-v1'
tokenizer = AutoTokenizer.from_pretrained(model_name, trust_remote_code=True)
model = AutoModel.from_pretrained(model_name, trust_remote_code=True)

# Function to get embeddings
def get_embeddings(texts, tokenizer, model):
    inputs = tokenizer(texts, padding=True, truncation=True, return_tensors="pt")
    with torch.no_grad():
        outputs = model(**inputs)
    return outputs.last_hidden_state.mean(dim=1)

# Function to find the most similar country
def find_country(address, countries, tokenizer, model):
    country_embeddings = get_embeddings(countries, tokenizer, model)
    address_embedding = get_embeddings([address], tokenizer, model)
    address_embedding = address_embedding.flatten() # Convert to 1-D array
    similarities = [1 - cosine(address_embedding, country_embedding) for country_embedding in country_embeddings]
    most_similar_country = countries[similarities.index(max(similarities))]
    return most_similar_country
```

As per the code above, we try to solve this information by creating a set of 'Country candidates', which uses the first 2 methods to generate some likely countries based on the phone number and city. Following the creation of that set, we then compare the city name against the country candidate list to get the closest match, choosing that as the selected country for the given contact information.

## Putting it all together
Now that we have described the process above, let's put it all together in a pipeline that can be run on our dataframe.

```python
import pandas as pd
import re
import unicodedata
from transformers import AutoTokenizer, AutoModel
import torch
from scipy.spatial.distance import cosine

df['Country_phone'] = df['Contact Information'].apply(lambda x: label_country_by_phone(x, phone_storage))
df['Country_city'] = df['Contact Information'].apply(lambda x: label_country_by_city(x, city_storage))
df['Country_candidates'] = df.apply(lambda x: get_intersection(x.Country_phone, x.Country_city, tokenizer, model), axis = 1)
df['Country'] = df.apply(lambda x: final_country(x['Contact Information'], x.Country_candidates), axis = 1)
```

Now we have successfully labelled our contact information with what is as good a guess as any as to what country it belongs to!

### Future work
Ultimately I chose to stop here as performance was not a key metric and majority of my dataset was successfully labelled. However, fome further improvements can also be made, as I also managed to scrape some other useful information such as the company description from the source of data, which can contain keywords like the country of origin or which stock exchange it belongs to. All this information could be used to better pinpoint where the company is from and can be used as part of the embedded string for a similarity search.
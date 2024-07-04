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


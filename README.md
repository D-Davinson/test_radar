📡 RADAR - Researcher Automatic Detection & Address Retrieval
🚀 Objective
RADAR automates the search for researchers based on a research topic or a specific researcher's name.
It utilizes Google Scholar (via SerpAPI) and Perplexity AI to retrieve:

Researcher profiles
H-index values
Affiliations
Institution addresses
Geolocation mapping of researchers
The system also supports:

✔ Filtering (H-index, country, location-based search)
✔ Automated researcher name validation via Perplexity AI
✔ Visualization on an interactive world map

📦 Libraries Used
🔧 Dependencies
Type	Library
Standard	os, re, time, random, json
PyPi	streamlit, scholarly, serpapi, folium, geopy, pandas, pycountry, openai
🔧 Installation
To install all dependencies, run:

bash
Copier
Modifier
pip install streamlit scholarly serpapi folium geopy pandas pycountry openai
💡 Project Structure
📂 Main Components
Component	Description
search_scholars_from_theme()	Extracts authors and publications related to a research theme.
get_scholar_names_perplexity()	Retrieves full researcher names while verifying publication authorship.
find_scholar_profile_serpapi()	Finds a researcher's Google Scholar profile via SerpAPI.
get_scholar_profile_serpapi()	Extracts affiliation and H-index from the profile.
get_affiliation_address_serpapi()	Retrieves the address and country of institutions via Google Search.
get_affiliation_address_perplexity()	Uses Perplexity AI to refine and validate institution addresses.
display_researcher_map()	Displays researchers on an interactive world map.
🔍 Implementation Details
Step 1: Searching for Researchers by Theme
Extracts authors and publications related to a given research theme.

python
Copier
Modifier
def search_scholars_from_theme(theme, max_results=35):
    """Search Google Scholar for authors based on a research theme."""
    try:
        search_query = scholarly.search_pubs(theme)
        authors_list, publications_list = set(), []

        for _ in range(max_results):
            try:
                publication = next(search_query)
                title = publication['bib'].get('title', "Unknown title")
                authors = publication['bib'].get('author', [])

                if isinstance(authors, str):
                    authors = authors.split(", ")

                for author in authors:
                    authors_list.add(author.strip())

                publications_list.append(title)
            except StopIteration:
                break
        
        return list(authors_list), publications_list

    except Exception as e:
        print(f"Error searching Google Scholar: {e}")
        return [], []
Step 2: Retrieving the Researcher’s Full Name via Perplexity AI
Ensures that the researcher’s name matches the publications found.

python
Copier
Modifier
def get_scholar_names_perplexity(authors, publications):
    """Retrieve full names of researchers and verify authorship using Perplexity AI."""
    if not authors or not publications:
        return None

    messages = [
        {
            "role": "system",
            "content": (
                "You are an expert in academic research. "
                "Your task is to retrieve the **full names** of the listed researchers "
                "and **verify that they are indeed the authors** of the publications found on Google Scholar."
            ),
        },
        {   
            "role": "user",
            "content": (
                "Perform a **search on Google Scholar** to retrieve the **full names** "
                "of the researchers listed below, based on their work.\n\n"
                "**⚠️ Ensure that these researchers are indeed the authors of the publications below.**\n"
                "**If a found name is not linked to the publications, DO NOT INCLUDE IT.**\n\n"
                f"📚 **Found publications:** {', '.join(publications)}\n\n"
                f"👨‍🔬 **List of researchers extracted from Google Scholar:** {', '.join(authors)}\n\n"
                "**Return only the list of valid full names, one per line.**"
            ),
        },
    ]

    response = client.chat.completions.create(
        model="sonar-pro",
        messages=messages,
    )

    return response.choices[0].message.content if response else None
Step 3: Finding the Researcher’s Profile on Google Scholar
Using SerpAPI, the system finds the researcher's Google Scholar profile.

python
Copier
Modifier
def find_scholar_profile_serpapi(full_name):
    """Uses SerpAPI to find a researcher’s Google Scholar profile."""
    params = {
        "engine": "google_scholar_profiles",
        "q": full_name,
        "api_key": os.getenv("SERPAPI_KEY"),
    }

    search = GoogleSearch(params)
    results = search.get_dict()

    if "profiles" in results:
        first_profile = results["profiles"][0]  # Take the first found profile
        scholar_id = first_profile.get("author_id")
        if scholar_id:
            return f"https://scholar.google.com/citations?user={scholar_id}"

    return None
Step 4: Extracting Affiliation & H-index
python
Copier
Modifier
def get_scholar_profile_serpapi(scholar_url):
    """Fetch researcher details from Google Scholar via SerpAPI."""
    match = re.search(r"user=([a-zA-Z0-9_-]+)", scholar_url)
    if not match:
        return {"Name": "Error", "Affiliation": "Error", "H-index": "Error"}

    scholar_id = match.group(1)
    params = {
        "engine": "google_scholar_author",
        "author_id": scholar_id,
        "api_key": os.getenv("SERPAPI_KEY"),
    }

    search = GoogleSearch(params)
    results = search.get_dict()

    if "author" in results:
        profile = results["author"]
        full_name = profile.get("name", "Unknown name")
        affiliation = profile.get("affiliations", "Unknown affiliation")
        h_index = results.get("cited_by", {}).get("table", [{}])[1].get("h_index", {}).get("all", "Not available")

        return {"Name": full_name, "Affiliation": affiliation, "H-index": h_index}

    return {"Name": "Error", "Affiliation": "Error", "H-index": "Error"}
Step 5: Retrieving the Institution Address
python
Copier
Modifier
def get_affiliation_address_serpapi(affiliations):
    """Uses SerpAPI to search for institution addresses and countries."""
    if not affiliations:
        return {}

    affiliation_data = {}

    for affiliation in affiliations:
        params = {
            "engine": "google_search",
            "q": f"{affiliation} address",
            "api_key": os.getenv("SERPAPI_KEY"),
        }

        search = GoogleSearch(params)
        results = search.get_dict()

        if "organic_results" in results and results["organic_results"]:
            first_result = results["organic_results"][0]
            address = first_result.get("snippet", "Address not found")
            country = "Unknown"

            for word in address.split():
                if word in pycountry.countries:
                    country = pycountry.countries.lookup(word).name
                    break

            affiliation_data[affiliation] = (address, country)

    return affiliation_data
🧑🏽‍💻 Authors
@Davinson DOGLAS PRINCE
@Exaucé MAKELE

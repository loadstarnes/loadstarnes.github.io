## International Music Charts
### An analysis of the favor of music artists globally, by country, and with respect to their origin countries

Austin Starnes

#

### Inspiration
Music is a cultural phenomenon. Nearly everyone loves music. It becomes a tool to focus, relax, or party.
As such, music is everywhere. 

I've noticed lately - call me crazy - that k-pop is on the rise in the west. 

I was interested, then, how one artist can fare internationally in another country. How much does a country favor their own musicians?
What if you account for country's language? What countries have the most internationally popular music? Are some countries outperforming others? 
Does Hong Kong have different music preferences than China? What about Taiwan? 
Does geopolitical tension - such as that between US and Russia, or US and China - cause a country's musical exports to be hamstrung?

#

### Required Tools
Through the process, I added tools as I needed them. I ended up needing these python modules:

```markdown
# Austin Starnes - Final Tutorial

# Install modules not included with docker image
!pip install graphviz
!pip install lxml
!pip install html5lib
!pip install romkan

# File IO, System, Date and Time
import sys, os 
import time, math
from datetime import timedelta, date, datetime

# Network Requests, Parsing
import requests, lxml, html5lib
from bs4 import BeautifulSoup as BeSoup
import re

# Data Handling & Visualization
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from pandas.plotting import register_matplotlib_converters
from graphviz import Digraph, Graph
import romkan 
from scipy.stats import gaussian_kde as kde
from matplotlib.colors import Normalize
from matplotlib import cm
from matplotlib import animation
register_matplotlib_converters()


```

#

### Data - kworb.net's Apple Music Charts

Our most important data is from [kworb.net](https://kworb.net/apple_songs/). It has daily-archived tables of music popularity from the Apple Music app.
I know this means our data is skewed towards iPhone users' musical preferences, so it will be acknowledged when interpreting our data.

Downloading this data takes maybe 10 minutes, so I'm including a saved version of the table. If you allow the saved version to be used, it won't collect recent dates' charts, but it loads really quickly.

```
# Get today's date
today = datetime.now()
date = datetime(year=today.year, month=today.month, day=today.day)

df = None

# If the data is archived (I hope it is), just load that
if os.path.isfile('fresh_apple_df.csv'):
    df = pd.read_csv('fresh_apple_df.csv')
    print('Found archived dataframe! Loading that.')
    print('New dates will not be fetched.')
# Otherwise... you'll have to download the whole dataset... day by day.
# It's a little over a year, so that's > 365 tables. No concurrency, sorry.
else:
    # Apple Music Chart for given day
    url = 'https://kworb.net/apple_songs/archive/'
    filename = str(date.strftime('%Y%m%d')) + ".html"
    print('Fetching Apple Music chart for ' + filename)
    # Get request
    r = requests.get(url + filename)
    chart_by_date = {}
    today = date
    
    # In case they haven't archived today's chart yet, go back one day
    if (r.status_code != 200):
        print(' - may not be archived for today yet. Attempting previous:')
        date = date - timedelta(days=1)
        filename = str(date.strftime('%Y%m%d')) + ".html"
        print('Fetching Apple Music chart for ' + filename)
        r = requests.get(url + filename)
        today = date
    second_try = False
    dfs = []
    days_todo = (today - datetime(year=2017, month=12, day=20)).days
    sys.stdout.write('Archived version not found :( starting download of ~1000 tables\n\n')
    sys.stdout.write('|____/~   Overall Progress   ~\____|        |____/~     Current File     ~\____|\n')
    start = time.time()
    
    # Retrieve each day's charts, trying twice, and parse into a dataframe.
    # Included nice i/o loading bar!
    while r.status_code == 200 or second_try == False:
        if r.status_code != 200:
            second_try = True
            r = requests.get(url + filename)
        else:
            soup = BeSoup(r.content,'lxml')
            table = soup.find_all('thead')[0].parent
            df = pd.read_html(str(table))[0]
            df = df.assign(date=date)
            dfs.append(df)
            date = date - timedelta(days=1)
            filename = str(date.strftime('%Y%m%d')) + ".html"
            days_done = (today-date).days
            ratio = days_done/days_todo
            p_str = ("\r [{}{}]  ".format('=' * math.ceil(ratio*32), ' ' * math.floor((1 - ratio)*32)) + '{:05.2f}'.format(float(ratio*100)) + '%')
            p_str += '   ' + filename + '   ETA: ' + str(int((1-ratio)/(ratio/(time.time() - start)))) + 's remain'
            sys.stdout.write(p_str)
            sys.stdout.flush()
            r = requests.get(url + filename)
    df = pd.concat(dfs)
    
    # SAVE IT. Nobody likes waiting so long.
    print('\nSaving that so you don\'t have to endure the wait again if you restart the kernal or something.')
    df.to_csv('fresh_apple_df.csv', index=False)
df
```

So, looking at this data, we can see we have artist and title in one column, concatenated with a ' - ' delimiter.
Artists are written in many different ways when they collaborate on one release. Some artists appear under two names.

So, first I'll split the Artist and Title column into two:
We also have... a lot of countries. I'm going to choose a subset to analyze.
```
# Standardize column names. Nobody likes spaces, and I don't want to remember uppercases.
df.columns = df.columns.to_series().apply(lambda x: x.lower().replace(' ', '_'))

# Data cleaning! Separate the Artist and Title column into two.
artist_re = re.compile("(?P<artist_str>^(?:(?! - ).)+) - (?P<title>.+)$")
def process_artist_title(x):
    m = artist_re.match(x)
    if m:
        return m
    else:
        return print(x)
    
# Create Artist Column
df['artist_str'] = df['artist_and_title'].apply(lambda s: process_artist_title(s)['artist_str'])
# Create Title Column
df['title'] = df['artist_and_title'].apply(lambda s: process_artist_title(s)['title'])
# Drop the crap we don't need.
# We are only keeping some countries to speed up analysis.
# Those are, United States, South Korea, Japan, China, Canada, Germany, India,
#  United Kingdom, Taiwan, France, Thailand, Australia, Russia, Italy,
#  and Hong Kong
df = df[['pos', 'title', 'artist_str', 'artist_and_title', 'days',
         'pk', 'pts', 'tpts', 'us', 'kr', 'jp', 'cn', 'ca', 'de', 'in', 'uk',
         'tw', 'fr', 'th', 'au', 'ru', 'it', 'hk', 'date']]
df
```

Next, I'll use regular expressions to parse out the artists into an array of individual artists.
Certain assumptions are made, and I tried to whitelist certain artists (Tyler, The Creator; Mumford & Sons) that read as if they were collaborations. However, I most likely missed some artists. I have no automated way of knowing which artists are collaboration titles  and which are actual musical group titles. (Selena Gomez & The Scene)

However, I think I did a pretty good job.

```
# More data cleaning!

# Artist column... has multiple artists, in many rows.
# Written a complex regular expression to parse it out.
r = re.compile('^((?:[^,\n])+?)(?: Featuring |, )([^,\n]+)(?:, ([^,\n]+))?(?:, ([^,\n]+))?(?:, ([^,\n]+))?(?:, ([^,\n]+))?(?:, ([^,\n]+))?(?: &| And|,) ([^,]+)$')

# WHY do artists need commas or ampersands in their names?
# I know Selena is also a solo artist, but since The Scene doesn't exist on its own,
# I'm choosing to keep it like that...
# There's probably more I didn't notice. I checked columns that contained &, while also appearing
# in many rows. If an artist appears with 'another' that many times, they're likely just a band name
whitelist = ['Tyler, The Creator', 'Selena Gomez & The Scene', 'Mumford & Sons']
splartist = re.compile('^,|^ |^&|^x |^Featuring |^Feat\. |^feat\. |^Ft\. |^ft\. |&$| $|,$|Feat.$|feat\.|Ft\.$|ft\.$|Featuring$| x$')

def split_artist(s):
    orig = s
    c = []
    for item in whitelist:
        if item in s:
            c.append(item)
    for item in c:
        s = s.replace(item, '')
        old = ''
        while old != s:
            old = s
            s = re.sub(rem, '', s)
    test_split = re.split(' & | Featuring | x |, | Feat\. | feat\. | Ft\. | ft\. ',s)
    p = list(filter(bool, test_split)) + c
    return p

df['artists'] = df['artist_str'].apply(split_artist)
df['artist_num'] = df['artists'].apply(lambda a: len(a))

# So now we have an array of the artists of a song entry, as well as a column for
# number of collaborating artists.
```

Now, I noticed some artists have special characters in them, so I filtered them to look for that. Actually, while googling some artists, I found that they were actually western artists with their names written in a foreign language.

So then, I found a module that would 'romanize' the Japanese into kind-of phonetic, roman alphabet versions.
If it changed the text, I knew it contained Japanese, and so I printed it to check them manually (only four western artists were encoded in Japanese script)
```
# This hunk of code told me if an artist contained a special character,
# or if this japanese-to-romanized converter changed anything.
# It's how I found Carey, Bieber, et al
for art in artists:
    if romkan.to_roma(art) != art:
        print(art + " => " + romkan.to_roma(art))
    if spec_char.search(art):
        print(art)
```

And, replace the obvious mistakes...
```
# More... data cleaning.
# This time, I'm looking for a few problems I noticed.
# Such as, Japanese phonetic spelling of a western artist.
# Who knew - Mariah Carey, Justin Bieber, Avicii, and Maroon 5 are
# rephonetized (?) in Japanese releases.

spec_char = re.compile('([^a-zA-Z0-9_ \-!$.])')

# For fixing the obvious problems!
repl = { 'BTS (防弾少年団)': 'BTS',
        'アヴィーチー': 'Avicii',
        'マライア・キャリ': 'Mariah Carey',
        'ジャスティン・ビーバー': 'Justin Bieber',
        'マルーン5': 'Maroon 5',
        'Sofia Reyes': 'Sofía Reyes'}

df.artists = df.artists.apply(lambda l: [repl[x] if x in repl.keys() else x for x in l])
for rep in repl.keys():
    df.artist_str = df.artist_str.apply(lambda s: s.replace(rep, repl[rep]))
df
```

#
### Artists, Origin Countries!

Having a complete list of artists is nice. Looking at the number of times they appear in these charts is also nice.
Let's flatten the artists column into one array, and then check the number of times each artist appears:
```
# Get value counts for each artist, should have reduced the
# number of unique artists.
vc = [col for col in df['artists']]
artists = pd.Series([item for sublist in vc for item in sublist])
artists_vc = artists.value_counts()
artists = artists_vc.keys()

# This will hold the artists' origin country.
artists_country = {}
```
Now, let's make a dataframe for each artist's origin country. It takes a long time to load the api for artist's country, as it throttles requests at 1 per second. It should use the csv file if it's available.

```
artists = list(artists)
count = 0
co_orig_df = None
if os.path.isfile('artists_orig_country.csv'):
    print("Found archived version! Using it.")
    co_orig_df = pd.read_csv('artists_orig_country.csv')
else:
    # API for finding country of origin! They're nice, no dev token needed
    url = 'https://musicbrainz.org/ws/2/artist'
    payload = { 'fmt':'json', 'query': artists[0]}
    headers = {'User-Agent': 'DataScience-School-Proj/0.1 ( AustinStarnes136@gmail.com )'}
    r = requests.get(url=url, headers=headers, params=payload)
    x = 0
    l = len(artists)
    unresolved = {}
    # THIS TAKES LIKE 16 MINUTES.
    # The api throttles you by IP and by user-agent if you exceed a single
    # request per second...
    # Having nearly 800 unique artists, it will take some time.
    # If you rerun this block without the one above, it will not look for
    # artists already in the dictionary.
    start_time = time.time()
    while x < l:
        json = r.json()
        keys = json.keys()
        art = artists[x]
        # Only processes the artist if we haven't found their origin yet.
        # This will only matter if you run this block more than once - it can
        # retry for any artist it didn't find before. Not that it would help,
        # unless you changed the way it calculates the best match.
        if art not in artists_country.keys() or artists_country[art] == None:
            percent = (x+1)*100/l
            curr_time = time.time()
            elapsed = curr_time - start_time if curr_time != start_time else 0.000001
            remaining_secs = int((100-percent)/(percent/elapsed))
            time_left = str(int(remaining_secs/60)) + 'm' + str(int(remaining_secs%60))
            while r.status_code != 200 or 'error' in keys:
                print(json)
                payload = { 'fmt':'json', 'query': art }
                r = requests.get(url=url, headers=headers, params=payload)
                count = 0
                # Slowwwwwly
                time.sleep(1)
            aliases = []
            country = None
            best = None
            for i in json['artists']:
                if i['score'] == 100:
                    best = i
                    if best['name'] == art or best['sort-name'] == art:
                        break
                    break
                elif i['score'] > 90 and i['name'] == art or i['sort-name'] == art:
                    best = i
                    break
                else:
                    print(i['score'])

            # Do your best to find the match. Lil Wayne does not have an origin country in this data...
            country = best['country'] if best is not None and 'country' in best.keys() else None
            artists_country[art] = country 
            if artists_country[art] is not None:
                sys.stdout.write('\r' + '{:05.2f}%   '.format(percent) + str(art) + " :: " + artists_country[art])
            else:
                sys.stdout.write('\r' + 'couldn\'t find ' + art + ' easily')
                unresolved[art] = json
            sys.stdout.write('\t\t' + str(time_left) + "sec left                        ")
            sys.stdout.flush()
        count += 1
        x += 1
        if x < l:
            art = artists[x]
            if art not in artists_country.keys() or artists_country.keys() == None:
                time.sleep(0.99)
                payload = { 'fmt':'json', 'query': art }
                r = requests.get(url=url, headers=headers, params=payload)

    # make it a DataFrame. Best data structure.
    co_orig_df = pd.DataFrame.from_dict(artists_country, orient='index',
                                        columns = ['origin'])

# SAVE IT SAVE IT
co_orig_df.to_csv('artists_orig_country.csv', index=False)
co_orig_df.columns = ['artist', 'origin']
co_orig_df = co_orig_df.set_index(['artist'])
co_orig_df
```

Now, we can gather some data about each country...

```
# Our country data!
# There's only one table. IDK why they gave it in text...
r = requests.get('http://download.geonames.org/export/dump/countryInfo.txt')

# Read line by line
lines = pd.Series(r.text.split('\n'))
# We're gonna ignore the comment lines (starts with #)
del_re = re.compile('^#[ \t].*$')
lines = lines[(~lines.str.match(del_re))].reset_index(drop=True)

# Normalize column names.
col = re.sub(r' |-|\(|\)', string=lines[0][1:].lower(), repl='_').split('\t')
# The first line is the column names, so drop that...
lines = lines[1:]

# It's tab-deliminated.
c_df = lines.str.split('\t', expand=True)
c_df.columns = col

# Index on the two-character ISO code for the country. It's unique, and short.
c_df = c_df.set_index('iso')

# GIVE ME LIBERTY OR GIVE ME DEATH
c_df['neighbors'] = c_df['neighbours']

# We only care about a few of these. I'm not going to need things like...
# phone-country-code...
c_df = c_df.drop(['iso3', 'iso_numeric', 'fips',
           'capital', 'area_in_sq_km_', 'tld',
           'phone', 'postal_code_format', 'postal_code_regex',
           'geonameid', 'equivalentfipscode', 'neighbours'], axis=1)
# Not sure where it picked up a blank row, but it's there. So let's drop it.
c_df = c_df.drop('')

# And... let's get the primary language.
# only 0-2 because en-US and en-UK are compatible, for example.
c_df['p_lang'] = c_df['languages'].str[0:2]
c_df
```
And make a column that contains the set of countries a song's artists are from:
```
df['countries'] = df['artists'].apply(lambda l: {co_orig_df.loc[a].origin for a in l if str(co_orig_df.loc[a].origin) != 'nan'})
df
```


With this data, I made an [interactive visualization](https://terpconnect.umd.edu/~astarnes/csterpconnect/countries.html) of each country's neighbors. We'll use it later for testing of how well music taste flows over borders.
```
dot = Digraph(comment="Countries")
dot.attr(bgcolor='#1f1f1f')
dot.engine='fdp'
dot.format = 'svg'
color = {'AF':'#EBA6BC', # pink
         'AN':'#C7A6D2', # mauve
         'AS':'#F8EF95', # lemon
         'EU':'#F3CA9C', # apricot
         'NA':'#A2DEBF', # azure
         'OC':'#B8E1B1', # mint
         'SA':'#E2E680'  # chartruese
        } #https://eleanormaclure.files.wordpress.com/2011/03/colour-coding.pdf; page 7 figure 4
processed = []
has_curr = []
for i, row in c_df.iterrows():
    processed.append(i)
#     if len(c_df[c_df['p_lang'] == row.p_lang]) > 1:
    for s in row['neighbors'].split(','):
        if s == '':
            continue
        dot.node(row['country'],
                 style='filled',
                 color=color[row['continent']])
        o_row = c_df.loc[s]
        if s not in processed:
            dot.edge(row['country'], o_row['country'],
                     dir='none', penwidth='3', color='#ffffff')
    processed.append(i)
        
    

dot.render('countries.geo.fdp', view=True)  
```


We can also start to plot data! Let's compare Chinese versus Hong Kong music taste:
```
plt.figure(figsize=(10,8))
#fromUS = df['countries'].apply(lambda c: 'US' in c)
data = df#[fromUS]
plt.xlabel('Chinese Score')
plt.ylabel('Hong Kong Score')
plt.title('Chinese Score vs. Hong Kong Score')
plt.scatter('cn_pts', 'hk_pts', color="#3335FF20", s=5, data=data, marker='o')
plt.show()
```
You can see that Chinese and Hong-Kong-Chinese people have similar tastes in music.

Compare to the Chinese vs US:
```
plt.figure(figsize=(10,8))

data = df
plt.xlabel('Chinese Score')
plt.ylabel('United States Score')
plt.title('Chinese Score vs. United States Score')
plt.scatter('cn_pts', 'us_pts', color="#3335FF20", s=5, data=data, marker='o')
plt.show()

```
The data has not accumulated along the x=y line, so the music taste is less compatible.



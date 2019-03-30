## International Music Charts
### An analysis of the favor of music artists globally, by country, and with respect to their origin countries

Austin Starnes

[editor on GitHub](https://github.com/loadstarnes/loadstarnes.github.io/edit/master/index.md)
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
import matplotlib.pyplot as plt
from pandas.plotting import register_matplotlib_converters
from graphviz import Digraph, Graph
import romkan 
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
# Those are, United States, South Korea, Japan, China, Germany, India,
#  Taiwan, France, Thailand, Australia, Russia, Italy,
#  and Hong Kong
df = df[['pos', 'title', 'artist_str', 'artist_and_title', 'days',
         'pk', 'pts', 'tpts', 'us', 'kr', 'jp', 'cn', 'de', 'in',
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



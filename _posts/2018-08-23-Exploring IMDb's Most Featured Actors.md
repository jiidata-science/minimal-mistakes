---

layout: single
date: 2018-08-23T11:48:41-04:00
header:
  overlay_color: "#000"
  overlay_filter: "0.6"
  overlay_image: /assets/images/site/DataBlog_titleImage.png
excerpt: "Scraping, exploring & visualising most featured actors in IMDb's Top 250 movies, with BeautifulSoup, Pandas & Plotly"
author_profile: true
author: Jez Irving
classes: wide

---

# ============================ WORK-IN-PROGRESS


# INTRO

So here it is...my first data science article (hopefully not my last). I decided to kick-off my series with at IMDb's top rated films to identify actors/actresses that feature across multiple top films - the 'real movie stars'. To be brutally honest about why I've dived into this relatively random topic; primarily I wanted to use web scraping technology (i.e. Python with beautifulSoup in this case) to collect data that isn't otherwise readily available in the desired format. The secondary reason was merely because I enjoy watching movies - hardly a unique interest.

Whilst IMDb's does readily offer APIs for accessing movie information (seems a little suprising to me) they do offer a number of [static datasets] (https://datasets.imdbws.com/). I chose not use these datasets and scraped required data directly from the IMDb.com.

In this blog:

  * We start with scraping IMDb film, actors/actress data (using BeautifulSoup)
  * We process and clean the captured data (using Pandas)
  * Then (more interestingly) we start to pull explore the data by:
    * Looking at the distribution of Top 250 film ratings
    * Understanding which film genres are more likely to have higher ratings, and
    * Identifying which actors/actresses appear in the most top rated films

Ultimately, this blog culminates in identifying "Which actors/actresses feature in the most Top 250 films?". I began this project under the naive assumption that these actors/actresses would be popular household names; the likes of Katharine Hepburn, Robert De Niro and Jack Nicholson. IMDb themselves provide a ranking of ['100 greatest actors & actresses'] (https://www.imdb.com/list/ls053085147/) BUT This analysis actually provides some suprising outcomes...enjoy!


``` python

# Import required Python libraries and setup workspace

import certifi
import urllib3
http = urllib3.PoolManager(cert_reqs='CERT_REQUIRED', ca_certs=certifi.where())
from bs4 import BeautifulSoup
import pandas as pd
import numpy as np
import csv

from scipy import polyfit, polyval
from scipy.interpolate import CubicSpline

import plotly.graph_objs as go
from plotly import __version__
from plotly.offline import download_plotlyjs, init_notebook_mode, plot, iplot
init_notebook_mode(connected=True)
%matplotlib inline

```

&nbsp;
# DATA COLLECTION (Part One)
## Scraping Top 250 rated movies
*Data captured on 27th July 2018*

We start by using urlib3 and beautifulSoup libraries to pull the Top 250 movies from IMDb's [Top 250 web page] (https://www.imdb.com/chart/top). We capture each movie title, along with it's official IMDb ranking and rating. In the code cell below, we create a list of lists, stored in the table_data variable.

``` python
table_data = [] # empty list. We'll be adding results to this

# IMDb page url for all top 250 rated films
url_imdbTop250 = 'https://www.imdb.com/chart/top'

# run web page request
page_imdbTop250 = http.request('GET', url_imdbTop250)

# allow for page exploration using BeautifulSoup (i.e. soup-ify returned webpage)
soup_Top250 = BeautifulSoup(page_imdbTop250.data, "lxml")

# create tabulated data with top 250 film information
table = soup_Top250.find('table', attrs={'class':'chart full-width'})# select the <table ...> that contains the ranked movies
table_body = table.find('tbody') # further subset the page. select only the table body
rows = table_body.find_all('tr') # find all rows within top 250 rated movies table

# for each row in table extract the: rank, movie name, year published and Imdb rating
for row in rows:
    cols = row.find_all('td')
    cols = [ele.text.strip() for ele in cols]
    movie_name_string = cols[1]
    movie_rank = movie_name_string[:movie_name_string.index('.')] 
    movie_year = int(movie_name_string[-5:-1])
    movie_name = movie_name_string[movie_name_string.index('.')+1:movie_name_string.index('(')].strip()
    movie_rating = cols[2]
    table_data.append([movie_rank, movie_name, movie_year, movie_rating])

table_data[:5] # print the top 5 results

```
  
    #[Output:]
    #Top250Rank | MovieName | Published | Rating
    #---------- | --------- | --------- | ------
    #1|'The Shawshank Redemption'|1994|9.2
    #2|'The Godfather'|1972|9.2
    #3|'The Godfather: Part II'|1974|9.0
    #4|'The Dark Knight'|2008|9.0
    #5|'12 Angry Men'|1957|8.9

&nbsp;

# DATA COLLECTION (Part Two)
## Scrape movie Genre & Cast/Crew

Each movie has it's own title landing page, covering a summary of information and a separate page for viewing the full cast/crew list per feature. Providing that you use the Imdb movie Id (i.e. an ID specific to Imdb) it's very simple to manipulate standard URLs and pull the information need:

 - https://www.imdb.com/title/{film_id} *: used to retrieve film genre*
 - https://www.imdb.com/title/{film_id}/fullcredits *: used to retrieve full cast / crew*

The film IDs are first retrieved from the initial page scrape, as they are provided in the html in the **'wlb_ribbon'** class. With these 250 x film IDs we proceed to scrape the information we require. I have clearly commented the code, below, to ensure that it can be intuitively understood.

``` python
# return all of the imdb film ids for crawling film casts
film_ids = []
base_castAndCrew = []

links_class  = soup_Top250.findAll("div", {"class":"wlb_ribbon"}) # table/html tag containing each IMDb movie ID

# for each movie, using the film ID:
for idx, link in enumerate(links_class):    
    
    counter = idx + 1
    if counter % 25 == 0: # print to log every n iterations
        print("[INFO] Scraping film no. {0} of {1}".format(counter,len(links_class)))
        
    # Part 1 ----------------------------------------
    # FOR EACH MOVIE, USE MOVIE ID TO CONSTRUCT MOVIE PAGE URL AND RETURN FILM GENRE
    
    filmID = link.attrs['data-tconst']
    filmID_castURL = "https://www.imdb.com/title/{0}/fullcredits".format(filmID) # used to retrieve full cast
    filmID_homePageURL = "https://www.imdb.com/title/{0}".format(filmID) # used to retrieve genre
    
    # request movie home page to return genre
    html_filmHomePage = http.request('GET', filmID_homePageURL)
    soup_filmHomePage = BeautifulSoup(html_filmHomePage.data, "lxml")
    soup_getGenre = soup_filmHomePage.find('div', {'itemprop':'genre'})
    genre_clean = str(soup_getGenre.text).replace('\n' , '').replace('Genres: ', '')
    
    # append derived data to output list
    film_ids.append([filmID , filmID_castURL , filmID_homePageURL, genre_clean])
    
    # Part 2 ----------------------------------------
    # FOR EACH MOVIE, PULL FULL CAST FROM MOVIE CAST/CREW PAGE URL
    
    # query the website and return the html to the variable ‘page’
    html_castAndCrew = http.request('GET', filmID_castURL)
    soup_castAndCrew = BeautifulSoup(html_castAndCrew.data, "lxml")
    table_castAndCrew = soup_castAndCrew.find('table', {'class':'cast_list'})
    
    # select cast list
    list_castAndCrew = []
    for cast in table_castAndCrew.findAll('span', {'class':"itemprop"}):
        list_castAndCrew.append(cast.text)
    base_castAndCrew.append(list_castAndCrew)  
    
    del filmID, filmID_castURL, filmID_homePageURL, genre_clean, html_filmHomePage, soup_filmHomePage, soup_getGenre, \
    list_castAndCrew, html_castAndCrew, soup_castAndCrew, table_castAndCrew
``` 

    #[INFO] Scraping film no. 25 of 250
    #[INFO] Scraping film no. 50 of 250
    #[INFO] Scraping film no. 75 of 250
    #[INFO] Scraping film no. 100 of 250
    #[INFO] Scraping film no. 125 of 250
    #[INFO] Scraping film no. 150 of 250
    #[INFO] Scraping film no. 175 of 250
    #[INFO] Scraping film no. 200 of 250
    #[INFO] Scraping film no. 225 of 250
    #[INFO] Scraping film no. 250 of 250

&nbsp;
# EXPLORATORY ANALYSIS
## Part One : Distribution of Top 250 movie ratings

Looking at the two ratings charts, we deduce that across the Top 250 movies:
  - the median movie rating is 8.2
  - The box plot identifies a number of outliers (note: Plotly atypically considers outliers as data points that are 1.5 x Interquartile range from the first (Q1) and third (Q3) quartiles. These seven outliers represent relatively (very) highly rated movies compares with others. With no suprise, these top rated films include some of my (and everyone elses) favouries
[{'1':'The Shawshank Redemption', '2': 'The Godfather','3': 'The Godfather: Part II','4': 'The Dark Knight', '5': '12 Angry Men'}]
-   extending on the above point, ratings do not have a purely (inverse) linear relationship with movie ranking. 95% of movies have a rating between 8.2 - 8.7 (range: 0.5), whilst the remaining 5% have much higher scores between 8.7 to 9.2 (range: 0.5).
  - The boxplot illustrates movie rating distribution characteristics, of the Top 250 rated films, split by film genre
  - I've ordered the x-axis from highest to lowest median rating value, by genre.
  - Perhaps, suprisingly, Music and Horror movies have the highest median rankings; but unsuprisgly each of these genres have just five contributing movies
  - X% of top movies are either dramas, x or y
  
## Part Two : Who really are the best actors?

  - So this was the bit I was most interested in. Here we take a look at which actors and actresses appear (in the cast & crew listings) across all 250 movies, how many movies they appeared in and what those films were?
  - in the code snippet below we create an interactive plotly chart that allows the user to select the top N actors/actresses with the most film features. The film start with the highest number of features appear on the far left of the chart and appears in descending order.
  - Perhaps I'm not quite the film buff I first thought but, to my suprise, the first actor I had heard of was in position five - Robert De Niro - who has appeared in eight of the top 250 films and is undoubtably a household name.
  - Interestingly, John Ratzenberger has 'appeared' in more of the Top 250 movies than any other actor - a whopping 12 x movies. However, 10 x of these were animation films - so he didn't even 'appear' in them at all. Bess Flowers, in position two, featured in 10 of the Top 250 movies but all before the 1970s.
  - In third position, Joseph Oliveira, was an quirky find - whilst he's featured in 9 of the Top 250 movies, he's only played 'supporting' or uncredited roles in each. For example: Dark Knight - a walk on officer, https://www.imdb.com/title/tt0468569/fullcredits?ref_=tt_cl_sm#cast uncredited role as 'Marciano' in Goodfellas https://www.imdb.com/title/tt0099685/fullcredits?ref_=tt_cl_sm#cast The Departed, again an uncredited Officer Court Room Attendant (uncredited) in Wolf of Wall Street https://www.imdb.com/title/tt0993846/fullcredits
  
# SUMMARY & FINAL THOUGHTS  
  
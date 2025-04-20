# Report

Jonatan Alcala, Bryan Gutierrez, Alex Castronovo

## Introduction

HPV (Human Pappapillomavirus) is one of if not the worlds most commonly transmitted STI costing the world’s population billions in healthcare costs each year. It most commonly presents as genital warts, and most cases are able to clear on their own but in certain cases HPV can lead to cancer of the cervix or anus. The vaccine designed to treat this disease is ninety percent effective at preventing these cancers, so it is critical that as many people become vaccinated against it as possible especially since there is no consensus to its herd immunity levels.

Arkansas ranks forty third in vaccination rate against HPV barely crossing over fifty percent. This statistic is shocking as Arkansas is one of the nations poorer states ranking thirty fifth in GDP per capita with a lot of that wealth being skewed to areas like Bentonville. As mentioned earlier, HPV treatment is costly and can lead to cancer, which in the long run is even more expensive. When nearly ten percent of the state is uninsured these health care costs can be devastating to individuals. To combat this, we as a team have decided to build a maximum coverage optimization model to cover as much of the population as possible. We decided to use already existing vaccination sites as they would be easy to provide HPV vaccines to as they already have infrastructure in place to handle other vaccines as well as trained staff.

## Data

To populate the model we needed many different kinds of data specifically, population data, county borders, and vaccine clinic locations. We decided to look at the county level as it was relatively granular but not so much so that it would overwhelm the model, and it would be manageable in the time we had to work on this project. To gather the population of each county we found an excel file on arkansas-demographics.com. This data was accurate as of 2023 and was the mot recent we could find from a reputable source. We then converted the data to a dictionary with the county name as the key and the population as the value. Next, we needed to gather information about current vaccination sites. We found a reputable source of current covid 19 vaccination clinics maintained by the state of Arkansas, but the only problem was that it was not downloadable as a csv. To circumvent this we used the python package selenium to web scrape the vaadian grid hidden in a shadow-dom that the information was stored in. With the help of some generative AI we were able to create custom code that could not only find and extract data from the table but scroll through it as well to make sure we got all the sites. In doing this we were able to extract all 1600+ sites. The only problem was that this was only the addresses of each site and to be useful in our model they needed to be converted into latitude and longitude. To do this we fed the csv created from web scraping into numerous geocoding packages including the google maps api, geopy, mapbox and opencage. We fed them through in order and saved two csv files for each package the ones successfully geocoded and the ones it failed on. We then took the ones it failed on and fed it to the next package until all addresses were geocoded and their latitudes and longitudes made sense. We lastly needed the Arkansas county borders for our model to work, luckily we had the .shp file of Arkansas county and state borders from a previous class we took so we just used that as we already had the necessary files. 

## Maximum Coverage Location Problem Mathematical Formulation

In this project, we formulated the Maximal Coverage Location Problem (MCLP) to determine the optimal placement of vaccination sites within Arkansas to maximize the population served within a 15-mile service radius. Our model components are defined as follows:

**Decision Variables**

Let \(x_j\) be a binary variable equal to 1 if candidate site \(j\) is selected, and 0 otherwise.Let \(y_i\) be a binary variable equal to 1 if county i is covered by at least one selected site, and 0 otherwise.

**Parameters**

- \(I\): the set of demand nodes (Arkansas counties), indexed by \(i = 1,...,m\).
- \(J\): the set of candidate facility locations, indexed by \(j = 1,...,n\).
- \(a_i\): the population of county \(i\).
- \(d_ij\): the distance in miles from the centroid of county \(i\) to site \(j\), computed via the haversine function.
- \(S = 15\): the maximum service distance in miles that defines coverage.
- \(P\): the exact number of sites to select.

**Objective Function**

We seek to maximize the total covered population across all counties:

\(maximize Z = sum_{i in I} a_i y_i\).

**Constraints**

Coverage constraints enforce that a county can only be covered if at least one selected site lies within \(S\) miles:

\(for each i in I\):

\(sum_{j: d_ij <= S} x_j >= y_i\).

The facility-limit constraint requires exactly \(P\) sites to be opened:

\(sum_{j in J} x_j = P\).

Binary restrictions apply to both decision variables:

\(x_j in {0,1}, for all j in J\);

\(y_i in {0,1}, for all i in I\).

This formulation captures the relationship between site selection and population coverage under the specified service threshold.

## Model Formulation

To apply this mathematical model to a code a computer can understand we created many functions that when all called together calculate our optimal site location and number. To start we needed a way to be able to tell how far apart two points were from one another so we created a function that calculates the haversine distance between two points. This is necessary as this will allow us to create the necessary radius. Speaking of radius we found an article titled " Spatial and Temporal Trends in Travel for COVID-19 Vaccinations”, we used this as a guide to determine our radius of 15 miles as we had to assume individuals would be willing to travel as far on average. Then we had to load our shapefile of Arkansas along with the population data, we then combined them together so we could reference them together. We then calculated the centroid point of each county which is what we will use to help us determine if we covered a county. We then created a function that would create a geopandas dataframe of the latitude and longitude of the vaccination sites we collected earlier. The geopandas dataframe from this step along with the geodataframe of the county borders and centroids were then passed into another function that along with our service radius of 15 miles calculated the coverage matrix. Our coverage matrix was binary i.e. 1 for the county Is covered or 0 for the county is not covered. We determined that a county was covered if its centroid fell within the service radius of the vaccination site or for very large counties it was within county borders. We then used a greedy algorithm to try and cover as much of the state as possible with using as few sites as possible. To figure out the correct number of sites we used a marginal gain function to calculate the percentage of the population we gained by adding a new site arbitrarily we said that if the next new site does not cover 2 percent of the population that we had reached the threshold. We then created a function that would map not only the state of Arkansas with county borders colored depending on county population but also all the possible sites in grey with the selected sites as red stars with their service radius drawn. We then combined all these functions into one main function that only required the file paths of the .shp file and the location of the csv storing the site information. With all of these functions we were able to determine we can cover over ninety-nine percent of the state with as few as 50 sites. 

**Haversine Distance Function**: 

This function computes the great-circle distance \(d_ij\) between county centroids and candidate sites, populating the distance parameter used in the coverage constraints. Counties whose distance to a site falls within the service radius \(S\) are marked as potentially covered.

 ```{python}
# 1. Compute Haversine Distance
from math import radians, sin, cos, sqrt, atan2

def haversine(lon1, lat1, lon2, lat2):
    """Returns distance in miles between two coordinate pairs."""
    lon1, lat1, lon2, lat2 = map(radians, [lon1, lat1, lon2, lat2])
    dlon, dlat = lon2 - lon1, lat2 - lat1
    a = sin(dlat/2)**2 + cos(lat1) * cos(lat2) * sin(dlon/2)**2
    c = 2 * atan2(sqrt(a), sqrt(1 - a))
    return 3958.8 * c  # Earth radius in miles
 ```

**Coverage Matrix Builder**:

The ```{python}build_coverage_matrix``` function iterates through counties and sites, using the haversine function to assign a 1 in matrix M[i, j] when county \(i\) lies within 15 miles of site \(j\). This binary matrix directly implements the coverage constraints of the MCLP.

```{python}
# 2. Build Coverage Matrix
import geopandas as gpd
import pandas as pd

def build_coverage_matrix(county_gdf, sites_gdf, radius=15):
    """
    Returns binary matrix M where M[i, j] = 1 if site j covers county i.
    """
    n_counties = len(county_gdf)
    n_sites   = len(sites_gdf)
    M = pd.DataFrame(0, index=county_gdf.index, columns=sites_gdf.index)

    for i, county in county_gdf.iterrows():
        cx, cy = county.geometry.centroid.x, county.geometry.centroid.y
        for j, site in sites_gdf.iterrows():
            sx, sy = site.longitude, site.latitude
            if haversine(cx, cy, sx, sy) <= radius:
                M.at[i, j] = 1
    return M
```

**Greedy Heuristic**:

The ```{python}greedy_cover``` procedure selects P sites by evaluating marginal gains in covered population at each iteration. It approximates the MCLP objective of maximizing the sum of populations covered (\(sum_i a_i y_i\)) under the constraint of opening exactly \(P\) sites, updating uncovered demand after each selection.

```{python}
# 3. Greedy Adding Heuristic for MCLP

def greedy_cover(M, populations, P):
    """
    Select P sites to maximize covered population.
    Returns selected site indices.
    """
    uncovered = populations.copy()
    selected = []

    for _ in range(P):
        # compute marginal gain for each candidate
        gains = {j: (M[j] * (uncovered > 0)).dot(populations)
                 for j in M.columns.difference(selected)}
        best = max(gains, key=gains.get)
        selected.append(best)
        # update uncovered
        covered_by_best = M[best] == 1
        uncovered[covered_by_best] = 0
    return selected
```

**Visualization Function**:

Finally, ```{python}plot_solution``` produces a map showing county populations, all candidate sites in gray, and the chosen sites highlighted with red stars and service-radius circles. This visual output validates and communicates the model’s coverage results.

```{python}
# 4. Visualize Solution

def plot_solution(county_gdf, sites_gdf, selected, radius=15):
    ax = county_gdf.plot(figsize=(10, 8), column='population', legend=True)
    sites_gdf.plot(ax=ax, color='gray', markersize=5)
    selected_sites = sites_gdf.loc[selected]
    selected_sites.plot(ax=ax, color='red', marker='*', markersize=100)

    for _, site in selected_sites.iterrows():
        circle = site.geometry.buffer(radius / 69)  # approx degree buffer
        gpd.GeoSeries(circle).plot(ax=ax, facecolor='none', edgecolor='red')
    ax.set_title(f"MCLP Solution: {len(selected)} Sites, {coverage_pct}% Covered")
    return ax
```

These functions combine to translate our mathematical formulation—distance calculations, binary coverage relationships, objective-driven site selection, and stakeholder-ready visualization—into an end-to-end executable pipeline for optimal site placement.

## Model Assumptions

For our model to work, we made some assumptions. First, we had to assume the population is equally distributed among the counties. Distributing the population equally allows us to use the centroid to measure the county. Essentially, if the county’s centroid is covered, then the county is 100 percent covered. This assumption was created because county-level data was utilized, and the exact location of each household was unknown. The model also needed the current location of vaccination sites, and the team decided to use the current COVID-19 vaccination sites since the transition to administering the HPV vaccine would be simpler both physically and economically. Since only Arkansas vaccination sites are being utilized in the model, this caused the exclusion of vaccination clinics in neighboring states. Since our model only covers clinics within the Arkansas borders, this means that if an individual who lives on the Arkansas and Texas border is closer to the clinic in Texas, this would not be taken into account, and the model will only take the closest Arkansas clinic. For the model to function, the measurement must be defined. The team decided to minimize the distance between the clinic and not the time. This was selected because due to how it simplified the computations, due to the variability of transportation.

## Further Model Improvements

Although our base model effectively identifies clinics that maximize coverage under simplified assumptions, real-world planning demands more depth. Incorporating cost and capacity constraints would allow us to limit total budget and ensure each site can vaccinate a realistic number of individuals per day. By assigning both fixed opening costs and per-dose operational costs to each location, the model could balance coverage goals against financial and logistical limitations. Additionally, modeling a maximum throughput for each clinic would prevent overloading any single facility and improve service reliability.

Switching from a uniform 15-mile travel radius to travel-time isochrones computed from actual road networks would better reflect real accessibility differences across urban and rural areas. This approach could account for variations in transportation infrastructure and travel speeds, yielding more accurate coverage estimates. At the same time, introducing equity weights into the objective function would enable prioritizing underserved or high-risk counties, aligning site selection with public health equity mandates rather than purely population-based criteria.

Further enhancements include allowing out-of-state clinics near the Arkansas border to be candidates, preventing border counties from being disadvantaged by intrastate limits. To address uncertainty in demand, such as fluctuating vaccine acceptance, a two-stage optimization or stochastic programming framework could generate location plans robust to variable uptake scenarios. Finally, embedding supply-chain constraints—such as vaccine delivery schedules and staffing availability—as linear constraints would tie the model to operational realities. Collectively, these improvements would elevate the MCLP from a theoretical exercise to a comprehensive decision-support tool for state-wide vaccination planning.

## Key Takeaways

First of all, Arkansas ranks 43rd in the United States in HPV vaccination rates, so clearly there is room for improvement. Second, our model shows that if the state strategically places 50 facilities with a 15-mile service radius, then there is a near-complete coverage of the population. A caveat is that once more than 50 clinics are added to the state, this will lead to diminishing returns; additional facilities only marginally increase coverage. Finally, it's important to note that this model doesn't consider real-world constraints like staffing, cost, or supply chain logistics—just population and geography. 

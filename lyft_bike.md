# Bay Wheels and Lyft Bike Graph Analysis
##Utilizing PySpark for Business Insights

What started as a project for MSBX 5420:Unstructured Distributed Data Modeling/Analysis turned into my passion project! The data used was provided by Bay Wheels (first regional and large-scale bicycle sharing system deployed in California) and can be found [here](https://www.lyft.com/bikes/bay-wheels/system-data). I leveraged software virtualization using docker to avoid downloading Spark on my local machine. Try this analysis yourself or take a look at the full code by following the instructions on my [Github](https://github.com/DylanBolt/lyft-bike_analysis)!

## Background & Motivation

Bay Wheels is a partnership between Lyft, the Bay Area Air Quality Management District, and the Metropolitan Transportation Commission. The program operates within the Bay Area, and specifically it serves individuals in Berkeley, Emeryville, Oakland, San Jose, and San Francisco. The program began with a pilot program in 2013 with 700 bikes and 70 stations servicing 5 cities within the bay area. Since its inception, the Bay Wheels program has grown significantly and in 2019, partnered with Lyft to increase its capacity to over 7,000 bikes at 550 stations. At these stations, there are both electric and regular bikes offered for users to rent with different costs associated with each.

Currently, there are three separate pricing plans: one for members, one for casual riders, and one for members of a program called “Bike Share for All.” These plans differ in subscription cost, included minutes, and additional fees. Members pay $169 per year or $29 per month as a subscription fee to receive unlimited 45 minute rides and pay an additional $0.20 per minute after this. Casual members pay a $3.49 fee every time they ride to unlock a bike and receive 30 minutes of ride time with an additional $0.30 per minute for any time over this. The Bike Share for All program, which is excluded from our analysis, allows low-income community members to pay $5 for the first year with an additional $5 per month after this to receive unlimited 60 minute rides with an additional cost of $0.13 per minute for any time over this. For riders using electric bikes, there are also additional fees associated with ride time of $0.20 per minute and $0.30 per minute for members and casual riders respectively.

Along with my interest in the system as a whole, I has an interest in trends associated with the COVID-19 pandemic. I'm are curious about the change in user type, whether members--who are more likely to be regular users and commuters--or casual riders were more likely to use the system before versus during the pandemic. Because of this, we used two separate months and conducted analyses on both of them. Later, I used `PySpark`'s `GraphFrames` package to create a network consisting of source and destination stations. This allowed me to look at features such as PageRank. Playing the role of a data analyst at Lyft, I focused the capstone of my analysis on creating a query to locate popular and profitable routes (combination of starting and ending stations) that Lyft can recommend users via their mobile app.

## Feature Creation

The data Bay Wheels provided included features such as starting and ending stations, type of bike, longitude and latitude, whether the rider was a subscriber, and the duration of the ride.

### Price Feature

We have information on the type of bike, the membership of the rider, and the duration of the trip. Combining this information in a pricing function allows us to see how much a each trip earned for Lyft.

```python
def price_ebikes(minutes, user_type, bike_type):
    '''
    Pass this function duration of a ride, user_type, and the type of bike and it returns the price.
    Duration must be in minutes. If the user isn't subscribed, pass the function
    'Customer'. If the user is a subscriber pass any string. Same deal for 'bike_type',
    but enter 'electric_bike' if the bike is electric and any string for traditional.
    '''
    unlock_fee = 3.49
    included_time_customer = 30
    included_time_sub = 45
    price_per_min_customer = 0.30
    price_per_min_sub = 0.20
    if bike_type == 'electric_bike':
        if user_type == 'casual':
                return round(unlock_fee + (minutes * price_per_min_customer),2)
        else:
            return (minutes * price_per_min_sub)
    else:
        if user_type == 'casual':
            if minutes > included_time_customer:
                return round(unlock_fee + ((minutes - included_time_customer) * price_per_min_customer),2)
            else:
                return unlock_fee
        else:
            if minutes > included_time_sub:
                return round(((minutes - included_time_sub) * price_per_min_sub),2)
            else:
                return 0
```
### Duration Feature

Our during-COVID dataset was lacking a duration column, and the start/endings times were of `string` type. The following code fixes these issues.

```python
#Changing start and end times to datetime for post set
#Then we can calculate the duration of each ride


post = post.withColumn("started_at", post['started_at'].cast('Timestamp')).withColumn("ended_at", post['ended_at'].cast('Timestamp'))\
            .withColumn('duration_min', (fn.unix_timestamp("ended_at") - fn.unix_timestamp("started_at"))/60)
post = post.withColumn('duration_min', fn.col('duration_min').cast('float'))
```

## Analysis and Results

The following analysis looks into the difference in average earnings for electric vs. standards bikes as well as subscribers vs. casual users. After this I explore the functionality of `GraphFrames` such as the PageRank feature before performing queries on the edges of the graph to show popular and profitable routes.

### Electric Bikes VS. Standard Bikes

After making our calculations for the amount of earnings from each ride, we decided to take a closer look at the average earnings of rides from electric bikes compared to standard bikes. This was done by first omitting rides that were outliers by omitting all rides with earnings of more than $100. This roughly equates to a 5 hour ride which we deemed to be at the upper end of what is realistic for a bike rental. That table was then referenced to show the average earnings of rides for both electric and normal bikes.

<img src="images/lyft_bike/electric_vs_standard_earnings.jpg?raw=true"/>

The average earnings per ride is higher for electric bikes. This is likely intended to be the case as electric bikes are more expensive than standard bikes. The pricing structure of having a fee per minute from the beginning of the ride is likely the reason for the discrepancy in average earnings. The difference in average earnings is smaller than expected.

### Member Program

Bay Wheel’s pricing structure was a major point of emphasis of this project and further analysis of their tiered pricing system led to some interesting insights. Grouping rides by members or casual riders, and then looking at the average duration of each shows that members ride for markedly less time on average. This comparison is shown below with the outliers removed in a similar fashion to the electric vs. normal bike comparison.

<img src="images/lyft_bike/casual_vs_member_duration.jpg?raw=true"/>

Though casual riders do ride for longer on average, it is telling that both of the average ride durations were shorter than the allotted time before the per minute fee on normal bikes. These shorter ride times are likely another reason that the electric bike profits are higher than standard bikes. Longer duration rides for casual users likely point towards the nature of the rides taken. Casual riders are more likely to go for scenic routes for enjoyment and members are more likely to use the bikes for commuting.

### Graph Analysis: PageRank

After making a graph to depict the paths between each Lyft bike station (vertices), I was able to calculate the pagerank of each station. Pagerank was created by Google to determine the most important sites based on the number of links that lead to the site. In this case, the websites are stations, and the links leading to the site are customers finishing their trip at that station. The stations at Grand Ave, 5th St, and Market St are considered the most important by this algorithm. This means that these stations are popular destinations for riders.

<img src="images/lyft_bike/pagerank.jpg?raw=true"/>

### Graph Analysis: InDegree and OutDegree

Using our created graph, we calculated the InDegree and OutDegree of each station. As seen in the tables below, the same stations tend to have a high InDegree and OutDegree. Due to these stations having the most trips finishing and beginning at them, we are led to believe that these are the most important stations.

<img src="images/lyft_bike/InDegreeOutDegree.jpg?raw=true"/>


### Popular and Profitable Route Query

I envisioned a query that could help Lyft identify routes that would be interesting for their riders and at the same time result in the bikes being rented for longer periods of time. In turn this would help Lyft earn more from these trips. More on the proposed **Recommend Routes** in the following section.

To accomplish this, I start by grouping by source and destination stations to create a list of possible routes. After this I create the features “path_count” and “avg_earnings”; taking into account the number of times each path was taken and the average earnings of each path, respectively. This way, we can capture routes that are both popular and profitable. Other issues to iron out with this query included filtering out paths that have too high of average earnings and making sure the source and destination stations are different for each path. I found ludacris outliers in the earnings and duration columns undoubtedly due to users abandoning their bikes, so these are excluded from analysis here. The most popular destination station for any given station was itself, so I had to make sure the query would show us the next most popular destination to make the routes. Lastly I sorted by “path_count”, located the paths from this list with the greatest earnings, and quickly checked that they were interesting routes.

```python
edges_count = edges.groupBy('src','dst').agg(fn.count('src').alias('path_count'))

edge_earnings = edges.groupBy('src','dst').agg(fn.avg('earnings').alias('avg_earnings'))\
.orderBy(fn.avg('earnings'), ascending = False).filter(fn.col('avg_earnings') < 100)

edges_joined = edges_count.join(edge_earnings,['src','dst'])
edges_joined.orderBy('path_count', ascending = False).filter(~(fn.col('src') == fn.col('dst'))).show()
```

<img src="images/lyft_bike/recommended_route_query.jpg?raw=true"/>

Stay tuned for a closer look at the highlighted routes!

## Graph Visualizations

The first network graph created uses networkx and matplotlib to show a rough outline of what the created graph looks like. As seen below, there are three distint clusters in this graph. Considering the data comes from Bay Wheels which operates in three Bay Area cities: San Francisco, Oakland, and San Jose, this is not surprising

```python
import networkx as nx

#turn the large network into a smaller one and create network from pandas
vertice = bike_graph.vertices.toPandas()
edges = bike_graph.edges.groupBy('src','dst').agg(fn.count('*').alias('trips'), fn.avg('earnings').alias('avg_earnings')).toPandas()
ranks = pageranks.vertices.toPandas()
labels = clusters.toPandas()
#connected = components.toPandas()

vertice.index = vertice['id']
ranks.index = ranks['id']
labels.index = labels['id']
#connected.index = connected['id']

ranks['pagerank'] = ranks['pagerank'] * 100
edges['trips'] = edges['trips'] / 100

graph = nx.from_pandas_edgelist(edges, 'src',  'dst', ['trips', 'avg_earnings'])
nx.set_node_attributes(graph, pd.Series(vertice.id, index=vertice.id).to_dict(), 'label')
nx.set_node_attributes(graph, pd.Series(ranks.pagerank, index=ranks.id).to_dict(), 'size')
nx.set_node_attributes(graph, pd.Series(labels.label, index=labels.id).to_dict(), 'group')
#nx.set_node_attributes(graph, pd.Series(connected.component, index=connected.id).to_dict(), 'component')
```

<img src="images/lyft_bike/networkx_graph.jpg?raw=true"/>

Next, I visualized the graph using the pyvis network package. This graph is interactive and allows zooming in and panning around the graph. Try it out yourself through my [Github](https://github.com/DylanBolt/lyft-bike_analysis).

<img src="images/lyft_bike/pyvis_far.jpg?raw=true"/>

<img src="images/lyft_bike/pyvis_close.jpg?raw=true"/>


## Business Implications

### Important Stations

Using the PageRank and InDegree/OutDegree results, I can determine the most important stations. PageRank determines important stations based on how important they are to the connection of the graph. InDegree and OutDegree determine important stations based on their degree of centrality in the graph. After looking at the location of these stations, it seems that they are located near other means of transportation such as bus stops or train stations. Lyft can use these insights to determine which stations are the most critical in the larger transportation network they established with BayWheels. If Lyft is short on bike-collecting employees, they should prioritize getting bikes to these stations. Similarly, if Lyft needs to unveil a new feature at some select stations, choosing these important stations will result in the quickest results.

### Route Recommendations

We were interested in	locating routes that offer customers enjoyable rides with interesting attractions or shops to ride past. In the same vein, we want to maximize Lyft’s profits through encouraging users to rent out their bikes for longer periods of time–hopefully past their included time. These goals can both be met by recommending destination stations at select starting points via a QR code attached to the station kiosk!

<img src="images/lyft_bike/qr_code.jpg?raw=true"/>

#### Popular & Profitable Rides

Below we will showcase a couple examples of what our query returned as the most popular and profitable routes.

1. Golden Gate Park to Ocean Beach
  -	The route from Stanyan & Fell St. Station to 48th Ave. & Cabrillo St. Station had 38 record trips with average earnings of $24.39. After further inspection, this route starts on the east side of Golden Gate Park, goes through the park, past the San Francisco Botanical Gardens, and ends near Ocean Beach. We believe this route generates high earnings due to riders stopping and taking pictures or sightseeing in the park.

<img src="images/lyft_bike/route_1.jpg?raw=true"/>

2. Ferry Building Station @ Fishermans Wharf
  - The route from the Ferry Building Station to Buchanan St. & North Point St. Station has 20 recorded trips with average earnings of $14.42. The route goes along the famous San Francisco piers and through Fisherman’s Wharf. We expect this route has good earnings due to riders stopping to shop or take pictures along the piers.

<img src="images/lyft_bike/route_2.jpg?raw=true"/>

## Conclusion & Cloud Deployment Usage

This project demonstrates how I analyzed Lyft bike data to uncover business insights. While this data is currently applied to only a couple months of bike sharing data. It is easily extendable to larger datasets on a distributed filesystem such as HDFS for cluster deployment. In addition, Lyft has the option to apply this analysis to a real-time stream of data in order to provide constantly improving route recommendations, among other things. 

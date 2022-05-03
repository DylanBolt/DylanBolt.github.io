# Bay Wheels and Lyft Bike Graph Analysis

What started as a project for MSBX 5420:Unstructured Distributed Data Modeling/Analysis turned into my passion project! The data used was provided by Bay Wheels (first regional and large-scale bicycle sharing system deployed in California) and can be found [here](https://www.lyft.com/bikes/bay-wheels/system-data). I leveraged a software virtualization using docker to avoid downloading Spark on my local machine. Try this analysis yourself by following the instructions on my [Github](https://github.com/DylanBolt/lyft-bike_analysis)!

## Background & Motivation

Bay Wheels is a partnership between Lyft, the Bay Area Air Quality Management District, and the Metropolitan Transportation Commission. The program operates within the Bay Area, and specifically it serves individuals in Berkeley, Emeryville, Oakland, San Jose, and San Francisco. The program began with a pilot program in 2013 with 700 bikes and 70 stations servicing 5 cities within the bay area. Since its inception, the Bay Wheels program has grown significantly and in 2019, partnered with Lyft to increase its capacity to over 7,000 bikes at 550 stations. At these stations, there are both electric and regular bikes offered for users to rent with different costs associated with each.

Currently, there are three separate pricing plans: one for members, one for casual riders, and one for members of a program called “Bike Share for All.” These plans differ in subscription cost, included minutes, and additional fees. Members pay $169 per year or $29 per month as a subscription fee to receive unlimited 45 minute rides and pay an additional $0.20 per minute after this. Casual members pay a $3.49 fee every time they ride to unlock a bike and receive 30 minutes of ride time with an additional $0.30 per minute for any time over this. The Bike Share for All program, which is excluded from our analysis, allows low-income community members to pay $5 for the first year with an additional $5 per month after this to receive unlimited 60 minute rides with an additional cost of $0.13 per minute for any time over this. For riders using electric bikes, there are also additional fees associated with ride time of $0.20 per minute and $0.30 per minute for members and casual riders respectively.

Along with my interest in the system as a whole, I has an interest in trends associated with the COVID-19 pandemic. I'm are curious about the change in user type, whether members--who are more likely to be regular users and commuters--or casual riders were more likely to use the system before versus during the pandemic. Because of this, we used two separate months and conducted analyses on both of them. Later, I used PySpark's `GraphFrames` package to create a network consisting of source and destination stations. This allowed me to look at features such as PageRank and station clusters. Playing the role of a data analyst at Lyft, I focused the climax of my analysis on creating a query to locate popular and profitable routes (combination of starting and ending stations) that Lyft can recommend users via their mobile app.

## Feature Creation

The data Bay Wheels provided included features such as starting and ending stations, type of bike, longitude and latitude, whether the rider was a subscriber, and the duration of the ride.

### Pricing Function

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

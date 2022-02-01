# Group Project for Advanced Data Analytics with Professor Dan Zhang

**Project description:** I worked alongside Joey Athanasiou, Talia Zalesne, and Rebecca Bracuti for this project. It was a pleasure to work with them and they were integral to putting this together. In our project scenario we played the role of a data-based consulting company focused on streaming platforms. Our goal is to use IMDb data on movies to create an accurate model that can predict the interactions of users with the movie after release. We can accomplish this through metrics such as box office sales and ratings.

## Data Understanding

Our dataset was created with IMDb data that included movie titles, year produced, director, writer, production company, budget, income, and viewer rating. No movie with fewer than 100 vote instances was kept in this dataset. Because predicting high volumes of user interactions is our main goal, determining the reasoning behind a viewer’s vote is crucial. With this in mind, we invested time into understanding what contributes to a highly rated movie. To test this, we brought in data to observe whether any given film had a popular director, production company, actor, or writer involved. We obtained this data through IMDb in the format of the top 100 directors, actors, production companies, and writers with the largest following of all time, with our earliest movie created in 1921. Our data ranged throughout numerous decades, so, in order to make any historical revenue values meaningful to our analysis, we recomputed the revenue data in accordance with the current inflation rate.

## Data Preparation

Because our clients are American companies, we chose to only analyze USA movies to ensure our results were congruent with the domestic movie market specifically. We removed movies that had null values for budget and income, as we noticed a good portion of these movies had not been released yet. We then scaled the values of duration, budget, and income after accounting for inflation on monetary features. We created a feature called “vote_class” which changes our predictor variable “avg_vote” to a categorical variable with bins including “Low” (rated 1-4.99), “Medium” (rated 5-6.99), and “High” (rated 7-10) for any classification models we ran. We created these bins with the context of what we considered to be low, medium, highly rated values along with ensuring we had enough data points in each bin.

'''
rm(list = ls())
imdb <- read.csv("movie_project.csv")

#Separate USA Movies and moves without NULL income and budget values
USAMovies <- imdb[imdb$country == 'USA', ]
USAMovies <- USAMovies[USAMovies$usa_gross_income  != "" & USAMovies$budget != "", ]
str(USAMovies)

#Deal with year and month columns
USAMovies$year <- as.numeric(USAMovies$year)

USAMovies$date_published <- as.Date(USAMovies$date_published, format = "%Y-%m-%d")

USAMovies$month <- as.numeric(format(USAMovies$date_published, '%m'))

USAMovies$month <- as.factor(USAMovies$month)

#Create a number of languages column
num_languages <- rep(NA, times = nrow(USAMovies))

for (row in 1:nrow(USAMovies)){
  comma.count <- 0
  if (USAMovies$language[row] == 'None' | USAMovies$language[row] == ''){
    num_languages[row] <- 0
  }
  else {
    list <- USAMovies$language[row]
    for (i in 1:nchar(list)){
      if (substr(list, i, i) == ','){
        comma.count <- comma.count + 1
      }
    }
    num_languages[row] <- comma.count + 1
  }
}

USAMovies$languageCount <- num_languages



#Get rid of unimportant character columns
USAMovies$description <- NULL
USAMovies$country <- NULL
USAMovies$title <- NULL
USAMovies$imdb_title_id <- NULL
USAMovies$language <- NULL

'''
###Including popular actors, directors, writers, and production companies:
We decided that including popular actors, directors, writers, and production companies would be a strong predictor of success, especially when conducting analysis from a business perspective. To include these factors as numeric variables, we downloaded CSV files (from IMDb), containing the top 100 actors, top 100 directors, and top 10 production companies. To incorporate the top production staff into our dataset, we created binary dummy variables, indicating whether the director, writer, or production company of any given movie is also included within their respective top lists (1 = yes, 0 = no). To include top actors, we created a feature called numTopActors that includes the number of top actors. In order to account for the timeline spread of our data, the CSV files we downloaded were the “top of all time”, which allowed us to identify the top actors/directors/writers of each decade.  By having these variables present, we can perform analysis to determine the impact of top level production staff on ratings. We considered the impacts of top actors by determining how many top 100 actors appeared in any given movie. In doing this, we can determine the implications of having one or more top 100 actors in our analysis.

'''

#Create a new column that says whether the director is in the top100 of all time directors list
top100directors <- read.csv("directors.csv")
directors <- top100directors$Name
top100directors <- NULL

USAMovies$topdirector <- ifelse(USAMovies$director %in% directors, 1,0)


#Create a column that says whether movie was produced by a top production company:
topProducers <- 	c('Warner Bros.', 'Sony Pictures', 'Walt Disney', 'Universal Pictures', '20th Century Fox', 'Paramount Pictures', 'Lionsgate Films', 'The Weinstein Company', 'Metro-Goldwyn-Mayer Studios', 'DreamWorks Pictures')

USAMovies$topproduction <- rep(0, times = nrow(USAMovies))

for (i in 1:nrow(USAMovies)){
	for (j in 1:length(topProducers)){
		if (grepl(topProducers[j], USAMovies$production_company[i])){
			USAMovies$topproduction[i] <- 1
		}
	}
}
table(USAMovies$topproduction)

#Create a new column that counts how many actors are in the top100 of all time actors list
actors <- read.csv("Actors.csv")
actors <- actors$Name

USAMovies$numTopActors <- NULL

for (i in 1:nrow(USAMovies)){
  count <- 0
  for (j in 1:length(actors)){
   if (grepl(actors[j], USAMovies$actors[i])){
     count <- count + 1
   }
  }
  USAMovies$numTopActors[i] <- count
}  
'''

###Continueing with the cleaning...

'''
table(USAMovies$numTopActors)  

#Turn budgets into numeric column
budgets <- substr(USAMovies$budget, 2,100)

budgets1 <- NULL

for (i in 1:nrow(USAMovies)){
  budgets1[i] <- substr(budgets[i],1,nchar(budgets[i])-1)
}
budgets2 <- as.numeric(gsub(",", "",budgets1))
budgets[838]

sum(is.na(budgets2))
USAMovies$budget <- budgets2

na.omit(USAMovies$budget)



#Turn USA Gross into numeric column
USAGross <- substr(USAMovies$usa_gross_income, 2,100)

USAGross1 <- NULL

for (i in 1:nrow(USAMovies)){
  USAGross1[i] <- substr(USAGross[i],1,nchar(USAGross[i])-1)
}

USAGross2 <- as.numeric(gsub(",", "",USAGross1))  
USAGross2  

USAMovies$usa_gross_income <- USAGross2  


#Turn World Wide Gross into numeric column
WWGross <- substr(USAMovies$worlwide_gross_income, 2,100)

WWGross1 <- NULL

for (i in 1:nrow(USAMovies)){
  WWGross1[i] <- substr(WWGross[i],1,nchar(WWGross[i])-1)
}

WWGross2 <- as.numeric(gsub(",", "",WWGross1))  
WWGross2  

USAMovies$worldwide_gross_income <- WWGross2  
USAMovies$worlwide_gross_income <- NULL



#Create column that says whether the movie was released worldwide or not
USAMovies$international <- as.factor(ifelse(USAMovies$usa_gross_income == USAMovies$worldwide_gross_income,0,1))
USAMovies$topdirector <- as.factor(USAMovies$topdirector)

'''

###Inflation:
We realized that any revenue analysis conducted over time would be upward trending if we do not account for inflation (since the release date of our movies range from 1921-2020). Because of this, including values for budget and income for movies from 1921 and 2020 would prove inaccurate to their relative popularity and scale from the time they were released. To remedy this issue, we imported a csv (https://www.officialdata.org/us/inflation/1800?amount=) containing a dollar's worth for each year starting with 1921’s value of a dollar. We then applied this data to our data on budget and income for each year to create new features that account for inflation based on the year of the movie’s release.

'''
#Read in inflation data:
inflation <- read.csv('inflation_data.csv')
inflation
inflation <- inflation[1:nrow(inflation)-1,]
inflation$dollar_worth <- inflation$amount[nrow(inflation)]/inflation$amount

USAMovies$budgetWithInflation <- NA
for (i in 1:nrow(USAMovies)){
	USAMovies$budgetWithInflation[i] <- USAMovies$budget[i] * inflation[inflation$year == USAMovies$year[i], 'dollar_worth']
}

USAMovies$grossWithInflation <- NA
for (i in 1:nrow(USAMovies)){
	USAMovies$grossWithInflation[i] <- USAMovies$usa_gross_income[i] * inflation[inflation$year == USAMovies$year[i], 'dollar_worth']
}
'''

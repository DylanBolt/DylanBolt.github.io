# Group Project for Advanced Data Analytics with Professor Dan Zhang

**Project description:** I worked alongside Joey Athanasiou, Talia Zalesne, and Rebecca Bracuti for this project. It was a pleasure to work with them and they were integral to putting this together. In our project scenario we played the role of a data-based consulting company focused on streaming platforms. Our goal is to use IMDb data on movies to create an accurate model that can predict the interactions of users with the movie after release. We can accomplish this through metrics such as box office sales and ratings.

## Data Understanding

Our dataset was created with IMDb data that included movie titles, year produced, director, writer, production company, budget, income, and viewer rating. No movie with fewer than 100 vote instances was kept in this dataset. Because predicting high volumes of user interactions is our main goal, determining the reasoning behind a viewer’s vote is crucial. With this in mind, we invested time into understanding what contributes to a highly rated movie. To test this, we brought in data to observe whether any given film had a popular director, production company, actor, or writer involved. We obtained this data through IMDb in the format of the top 100 directors, actors, production companies, and writers with the largest following of all time, with our earliest movie created in 1921. Our data ranged throughout numerous decades, so, in order to make any historical revenue values meaningful to our analysis, we recomputed the revenue data in accordance with the current inflation rate.

## Data Preparation

Because our clients are American companies, we chose to only analyze USA movies to ensure our results were congruent with the domestic movie market specifically. We removed movies that had null values for budget and income, as we noticed a good portion of these movies had not been released yet. We then scaled the values of duration, budget, and income after accounting for inflation on monetary features. We created a feature called “vote_class” which changes our predictor variable “avg_vote” to a categorical variable with bins including “Low” (rated 1-4.99), “Medium” (rated 5-6.99), and “High” (rated 7-10) for any classification models we ran. We created these bins with the context of what we considered to be low, medium, highly rated values along with ensuring we had enough data points in each bin.

```
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

```

### Including popular actors, directors, writers, and production companies:

We decided that including popular actors, directors, writers, and production companies would be a strong predictor of success, especially when conducting analysis from a business perspective. To include these factors as numeric variables, we downloaded CSV files (from IMDb), containing the top 100 actors, top 100 directors, and top 10 production companies. To incorporate the top production staff into our dataset, we created binary dummy variables, indicating whether the director, writer, or production company of any given movie is also included within their respective top lists (1 = yes, 0 = no). To include top actors, we created a feature called numTopActors that includes the number of top actors. In order to account for the timeline spread of our data, the CSV files we downloaded were the “top of all time”, which allowed us to identify the top actors/directors/writers of each decade.  By having these variables present, we can perform analysis to determine the impact of top level production staff on ratings. We considered the impacts of top actors by determining how many top 100 actors appeared in any given movie. In doing this, we can determine the implications of having one or more top 100 actors in our analysis.

```

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
```

### More cleaning...

```
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

```

### Inflation:

We realized that any revenue analysis conducted over time would be upward trending if we do not account for inflation (since the release date of our movies range from 1921-2020). Because of this, including values for budget and income for movies from 1921 and 2020 would prove inaccurate to their relative popularity and scale from the time they were released. To remedy this issue, we imported a csv (https://www.officialdata.org/us/inflation/1800?amount=) containing a dollar's worth for each year starting with 1921’s value of a dollar. We then applied this data to our data on budget and income for each year to create new features that account for inflation based on the year of the movie’s release.

```
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
```

### Creating a streamlined dataframe for analysis

This process includes paring off extraneous columns, scaling numeric columns, and creating columns for each genre using R's stringr package.

```
clean_movies <- USAMovies[,c("original_title", "year", "month", "duration","avg_vote","budget","usa_gross_income","languageCount","topdirector","numTopActors", "international", "budgetWithInflation", "topwriter", "topproduction", "grossWithInflation")]


#Add columns for each genre

str(USAMovies$genre)
library(stringr)
genres <- unique(unlist(str_split(USAMovies$genre, ", ")))

for (genre in genres) {
	USAMovies[[genre]] <- 0
}

for (i in 1:nrow(USAMovies)) {
	for (genre in genres) {
		if (genre %in% str_split(USAMovies$genre, ", ")[[i]]) {

			USAMovies[[genre]][i] <- 1
		}
	}
}

for (genre in genres){
	clean_movies[genre] <- USAMovies[,genre]
}

str(clean_movies)

clean_movies$budget <- NULL
clean_movies$usa_gross_income <- NULL

clean_movies$budgetScaled <- scale(clean_movies$budgetWithInflation)
clean_movies$income <- USAMovies$grossWithInflation
clean_movies$durationScaled <- scale(clean_movies$duration)

clean_movies$budgetWithInflation <- NULL
clean_movies$grossWithInflation <- NULL
clean_movies$duration <- NULL

clean_movies$languageCount <- USAMovies$languageCount

colnames(clean_movies)[29] <- 'Sci_Fi'
colnames(clean_movies)[23] <- 'Film_Noir'
```
### Data Exploration

```
str(clean_movies)

#Correlation matrix between numeric variables:
cor(na.omit(clean_movies[, unlist(lapply(clean_movies, is.numeric))]))
#Unsurprisingly, budget and gross income are pretty highly correlated


#Tree model to see which variables are most important:
library(tree)
tree_mod <- tree(income ~. , data = cm)
summary(tree_mod)
#duration, usa_gross_income, and topdirector used



#Make some plots:
plot(clean_movies$duration, clean_movies$avg_vote)
plot(clean_movies$budget, clean_movies$avg_vote)
plot(clean_movies$budgetWithInflation, clean_movies$avg_vote)
plot(clean_movies$languageCount, clean_movies$avg_vote)
plot(clean_movies$topdirector, clean_movies$avg_vote)
plot(clean_movies$topproduction, clean_movies$avg_vote)
plot(clean_movies$topwriter, clean_movies$avg_vote)
plot(clean_movies$numTopActors, clean_movies$avg_vote)
plot(clean_movies$international, clean_movies$avg_vote)
```

<img src="images/movies_monthly.jpg?raw=true"/>



### Tree Models: Random forest, boosting

```
library(tree)
set.seed(300)
n <- nrow(clean_movies)
train <- sample(n, n*.6)

tree_mod <- tree(income ~. - original_title - incomeScaled - budget - profit, data = clean_movies,
                 subset = train)
summary(tree_mod)
plot(tree_mod)
text(tree_mod, pretty = 0)

pred <- predict(tree_mod, newdata = clean_movies[-train, ])
plot(pred, clean_movies[-train, "income"])

abline(0, 1)
mean((pred - clean_movies[-train, "income"])^2)
#2,434,526,928

#try turning this into a classification tree
#group movies into low, medium, high income bins
#low income < 3.751e+06
# 3.751e+06
hist(clean_movies$income)
hist(clean_movies$avg_vote)

#Bagging Regression

library(randomForest)

?randomForest

set.seed(300)
n <- nrow(clean_movies)
train <- sample(n, n*.6)
bag.movies <- randomForest(avg_vote ~ . - original_title - income - budget - profit,
                          data = clean_movies, subset = train, mtry = 33)
pred.bag <- predict(bag.movies, newdata = clean_movies[-train,])


#Scores
plot(pred.bag, clean_movies[-train, "avg_vote"])
abline(0, 1)
mean((pred.bag - clean_movies[-train, "avg_vote"])^2)
#MSE: 0.6733205


#Random forest Regression
#mtry = P/3 = 11
set.seed(300)
rf.movies <- randomForest(avg_vote ~ . - original_title - income - budget - profit - vote_class,
                          data = clean_movies, subset = train, mtry = 11)
pred.rf <- predict(rf.movies, newdata = clean_movies[-train,])

#Scores
plot(pred.rf, clean_movies[-train, "avg_vote"])
abline(0, 1)
mean(sum((pred.rf - clean_movies[-train, "avg_vote"])^2))
plot(rf.movies)
summary(rf.movies)
#MSE: 1130.408


#Random forest regression 500 trees

set.seed(300)
rf.movies500 <- randomForest(avg_vote ~ . - original_title - income - budget - profit - vote_class,
                            data = clean_movies, subset = train, ntree = 500, mtry = 11)
pred.rf500 <- predict(rf.movies500, newdata = clean_movies[-train,])


#Scores
plot(pred.rf500, clean_movies[-train, "avg_vote"])
abline(0, 1)
mean(sum((pred.rf500 - clean_movies[-train, "avg_vote"])^2))
plot(rf.movies500)
summary(rf.movies500)
#MSE: 1130.408


#Lets try classification
#movies rated from 1-4.99 are low, 4.99-7 are medium, and 7-10 are high



clean_movies$vote_class <- NULL

for (i in 1:nrow(clean_movies)){
  if (clean_movies$avg_vote[i] < 4.99 ){
    clean_movies$vote_class[i] <- "Low"
  } else if (clean_movies$avg_vote[i] > 4.99 & clean_movies$avg_vote[i] < 7){
    clean_movies$vote_class[i] <- "Medium"
  }
    else {
      clean_movies$vote_class[i] <- "High"
    }
}
table(clean_movies[-train, "vote_classes"])
clean_movies$vote_class <- as.factor(clean_movies$vote_class)


#Random forest Classification
#mtry = sqrt(p) = 6
set.seed(300)
rf.class.movies <- randomForest(vote_class ~ . - original_title - income - budget - profit - avg_vote,
                          data = clean_movies, subset = train, mtry = 6)
pred.rf.class <- predict(rf.class.movies, newdata = clean_movies[-train,])

#Scores
table(clean_movies[-train, "vote_class"], pred.rf.class)
summary(rf.class.movies)
#Accuracy = (1080 + 14 + 182)/1749 = 0.7295597
#Highly rated movie precision = 182/250 = 0.728
#Medium rated movie precision = 1080/1479 = 0.73022312
#Lowly rated movie precision = 14/20 = 0.7
print(rf.class.movies)


length(clean_movies[-train, "vote_class"])
library(ROCR)
auc <- performance(pred.rf.class, "auc")

varImpPlot(rf.class.movies)
plot(importance(rf.class.movies), )

# make dataframe from importance() output
library(ggplot2)
feat_imp_df <- importance(rf.class.movies) %>%
  data.frame() %>%
  mutate(feature = row.names(.))

# plot dataframe
ggplot(feat_imp_df, aes(x = reorder(feature, MeanDecreaseGini),
                        y = MeanDecreaseGini)) +
  geom_bar(stat='identity') +
  coord_flip() +
  theme_classic() +
  labs(
    x     = "Feature",
    y     = "Importance",
    title = "Feature Importance: <Model>"
  )


# boosting
library(gbm)
set.seed(300)
?gbm
clean_movies[,"Documentary" == 1] <- NULL
boost.movies <- gbm(avg_vote ~ . - original_title - income - budget - profit - vote_class,
                    data = clean_movies[train,], distribution = "gaussian", n.trees = 1000, cv.folds = 5)
pred.boost <- predict(boost.movies, n.trees = 1000, type = "response")
mean(sum((pred.boost - clean_movies[-train, "avg_vote"])^2))
#MSE: 1006.475


#boosting with classification
set.seed(300)
?gbm
boost.class <- gbm(vote_class ~ . - original_title - income - budget - profit - avg_vote - Documentary,
                   data = clean_movies[train,], distribution = "gaussian", n.trees = 1000, cv.folds = 5)

pred.boost.class <- predict(boost.class,newdata = clean_movies[-train,], n.trees = 1000, type = "response")

table(clean_movies[-train, "vote_class"], pred.boost.class)
pred.boost.class
table(clean_movies$vote_class)


best.iter <- gbm.perf(boost.movies, method = "cv")
plot(pred.boost, clean_movies[-train, "avg_vote"],
     xlab = "Predicted Rating", ylab = "Actual Rating")
abline(0, 1)

```
### Trying Linear Models:

```
cm <- clean_movies

cm$original_title <- NULL
cm <- na.omit(cm)

trainingRowIndex <- sample(1:nrow(cm), 0.8*nrow(cm))
trainingData <- cm[trainingRowIndex, ]
testData  <- cm[-trainingRowIndex, ]
trueIncome <- cm[-trainingRowIndex, 'income']

#Model with everything:
genMod <- lm(income ~., data = trainingData)
summary(genMod)
pred <- predict(genMod, testData)
mean(sum((pred - trueIncome)^2))


#Model based on what tree said was most important:
lm1 <- lm(income ~ year + budgetScaled + international + topwriter + avg_vote, data = trainingData)
summary(lm1)
pred <- predict(lm1, testData)
mean(sum((pred - trueIncome)^2))


#Start removing columns...
mod <- lm(income ~.-month, data = trainingData)
pred <- predict(mod, testData)
mean(sum((pred - trueIncome)^2))
summary(mod)


mod <- lm(income ~.-month-languageCount, data = trainingData)
pred <- predict(mod, testData)
mean(sum((pred - trueIncome)^2))
summary(mod)


mod <- lm(income ~.-month-languageCount-topproduction, data = trainingData)
pred <- predict(mod, testData)
mean(sum((pred - trueIncome)^2))
summary(mod)


mod <- lm(income ~.-languageCount, data = trainingData)
pred <- predict(mod, testData)
mean(sum((pred - trueIncome)^2))
summary(mod)
```
<img src="images/movies_boosting_results.jpg?raw=true"/>

<img src="images/movies_gini.jpg?raw=true"/>


### Feature Selection with Boruta package

```
#Try feature selection:
#install.packages('Boruta')
library(Boruta)

#For Income:
trainingRowIndex <- sample(1:nrow(cm), 0.8*nrow(cm))
trainingData <- cm[trainingRowIndex, ]
testData  <- cm[-trainingRowIndex, ]
trueIncome <- cm[-trainingRowIndex, 'income']

#Tentative = True
boruta_output <- Boruta(income ~ ., data=trainingData, doTrace=0)
boruta_signif_incomeT <- getSelectedAttributes(boruta_output, withTentative = TRUE)
#Tentative = False
boruta_signif_incomeF <- getSelectedAttributes(boruta_output, withTentative = FALSE)


#For AvgVote:
trainingRowIndex <- sample(1:nrow(vdf), 0.5*nrow(vdf))
trainingData <- vdf[trainingRowIndex, ]
testData  <- vdf[-trainingRowIndex, ]

#Tentative = True
boruta_output <- Boruta(AvgVote ~ .-FemVote-MaleVote, data=trainingData, doTrace=0)
boruta_signif_votesT <- getSelectedAttributes(boruta_output, withTentative = TRUE)
#Tentative = False
boruta_signif_votesF <- getSelectedAttributes(boruta_output, withTentative = FALSE)

#For Males only:
#Tentative = True
boruta_output <- Boruta(MaleVote~.-FemVote-AvgVote, data=trainingData, doTrace=0)
boruta_signif_MvotesT <- getSelectedAttributes(boruta_output, withTentative = TRUE)
#Tentative = False
boruta_signif_MvotesF <- getSelectedAttributes(boruta_output, withTentative = FALSE)

#For Females only:
#Tentative = True
boruta_output <- Boruta(FemVote ~.-AvgVote-MaleVote, data=trainingData, doTrace=0)
boruta_signif_FvotesT <- getSelectedAttributes(boruta_output, withTentative = TRUE)
#Tentative = False
boruta_signif_FvotesF <- getSelectedAttributes(boruta_output, withTentative = FALSE)
```
<img src="images/movies_variable_importance.jpg?raw=true"/>


### Deployment

Model implementation will allow us to work hand-in-hand with our clientele and discover plausible solutions to their creative project concerns. By allowing access to data on what their project entails: producer, director, starring roles and actors, production company, genre of film, will it be expanded globally and so on, we can conduct experimental models to determine the likelihood of viewership and user interactions on their site. Furthermore, recommending changes to increase probabilities of success for clients. For example, if Netflix is creating a ‘Netflix Original’ with an unknown director and Tom Hanks as the lead role, our models can reveal insights into the breakdown of situational changes. If the director is reassigned to Woody Allen, the projected average rating could increase twofold.
Although any model we create can have superior accuracy than a competitor, one factor to consider when using data to predict results is bias. Influence from uncontrolled factors such as scandal or sudden complexities within a movie’s project can result in failures of hitting the ideal mark. Data predictions are useful tools but can only predict within the realms of the data at hand. That being said, with every project we consult with, our database will expand, creating an updated and unique database for each client's required solutions. Newly discovered stars, immaculate production companies, new-age directors, and so on can also impact the sought out popularity of your film. Adding this data to our database will quickly improve the accuracy of our recommendation.

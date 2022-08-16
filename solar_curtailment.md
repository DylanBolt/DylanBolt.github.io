# EDA and Predicting Solar Curtailment

This notebook project was created as a case study for a Solar Energy Firm. I cannot provide the data used for privacy reasons, but I will include the most interesting insights here and provide a link to the [full code on my Github](https://github.com/DylanBolt/solar_curtailment_prediction). The data was consists of actual 5-minute resolution price and production data from the Electric Reliability Council of Texas (ERCOT) for Q2 2019.

## Objective

Solar curtailment primarily occurs when there is an oversupply in generation at any point in time. Electricity market operators need to ensure that power supplied closely matches power demanded to avoid damaging equipment. When there is an oversupply, the market operator ERCOT will use negative pricing to incentivize generators to reduce their power output. For the purposes of this analysis, I will assume that solar curtailment would occur when the system-wide energy price (SYSTEM_LAMBDA) drops below $0/MWh. After performing some exploratory data analysis I will create a model with Logistic Regression to predict whether solar curtailment is likely to occur for any 5-minute increment.

### Clean Data

First, let's change the column headers

```python
ercot = ercot.rename(columns = {
    "Date/Time":"DATETIME",
    "ERCOT (RT_ORD_PRADDER) Average":"RT_OR_PRADDER",
    "ERCOT (RT_ON_CAP) Average":"RT_ON_CAP",
    "ERCOT (HASL) Average":"HASL",
    "ERCOT (RTLOAD) Average":"RTLOAD",
    "ERCOT (GENERATION_SOLAR_RT) Average":"PVACTUAL",
    "ERCOT (SOLAR_PVGRPP_BIDCLOSE) Average":"PVFORECASTED",
    "ERCOT (WINDDATA) Average":"WINDACTUAL",
    "ERCOT (WIND_COPHSL_BIDCLOSE) Average":"WINDFORECASTED",
    "ERCOT (SYSTEM_LAMBDA) Average":"ERCOTRTPRICE",
    "HB_WEST (RTLMP) Average":"HBWESTRTPRICE"
                                })
```

The “Date/Time” column was of object type, so I parsed it to a datetime type. Besides this column, the others were read in correctly as float type.

```python
ercot['DATETIME_PARSED'] = pd.to_datetime(ercot['DATETIME'], format = '%m/%d/%Y %H:%M')
```

Next, I will check how many NA values are present and drop rows with NAs. The dataset lost 1,988 rows after removing NAs. Luckily, this is a clean dataset and only 3.75% of the rows had NAs.

```python
ercot_cleaned = ercot.dropna()

print(f'Rows in original dataset: {ercot.shape[0]}')
print(f'Rows with NAs dropped: {ercot_cleaned.shape[0]}')
print(f'Number of rows dropped: {(ercot.shape[0] - ercot_cleaned.shape[0])}')
print(f'Percentage of rows dropped: {round((ercot.shape[0] - ercot_cleaned.shape[0])/(ercot.shape[0]) * 100,2)}%')
```

### Exploratory Data Analysis with Data Visualizations

I chose to construct a histogram of the “ERCOTRTPRICE” or “SYSTEM-LAMBDA” data due to being interested in the distribution of our target variable. The majority of the time energy prices are between 15 and 30 $/MWh, and there are very few data points below 0 $/MWh (solar curtailment).

<img src="images/solar_curtailment/ERCOTRTPRICE_hist.jpg?raw=true"/>


Next, I chose to represent the relationship between “PVACTUAL” and “PVFORECASTED” to get an idea of how accurate the predictions for MW production are. The variables exhibit a strong relationship due to the values being concentrated around an imaginary line with slope of 1, as the values for each column are similar to each other most of the time.

<img src="images/solar_curtailment/PVACTUAL_vs_PVFORECASTED.jpg.jpg?raw=true"/>

### How does Wind Energy Production Impact ERCOT Prices?

I will create a new dataframe that only has values of 'WINDACTUAL' at or above the 3rd quartile value

```python
ercot_summary = ercot_cleaned.describe()

#Access the the 3rd quartile value (75%) for WINDACTUAL

windactual_3rdQ = ercot_summary.loc['75%','WINDACTUAL']

print(f"Third quartile of 'WINDACTUAL': {windactual_3rdQ} MW")

#Use 3rd quartile value to create new dataframe
#where all instances have a 'WINDACTUAL' value at or above the 3rd quartile

ercot_windacutal_3rdQ = ercot_cleaned.loc[ercot_cleaned['WINDACTUAL'] >= windactual_3rdQ, ]

#Check that the resulting dataframe has correct values for 'WINDACTUAL'
ercot_windacutal_3rdQ['WINDACTUAL'].min()
```
Now, I will do some light analytics to compare this new dataframe with the full data. I will focus on the ERCOT price.

```python
#Compare ERCOTRTPRICE means across both dataframes

original_mean = ercot_cleaned['ERCOTRTPRICE'].mean()
upper_windactual_mean = ercot_windacutal_3rdQ['ERCOTRTPRICE'].mean()
mean_diff = abs(ercot_cleaned['ERCOTRTPRICE'].mean() - ercot_windacutal_3rdQ['ERCOTRTPRICE'].mean())
percent_diff = round((ercot_windacutal_3rdQ['ERCOTRTPRICE'].mean()/ercot_cleaned['ERCOTRTPRICE'].mean())*100,2)

print(f"'ERCOTRTPRICE' mean of original dataframe: {original_mean} $/MWh")
print(f"'ERCOTRTPRICE' mean of new dataframe: {upper_windactual_mean} $/MWh")
print(f" Difference in means: {mean_diff} $/MWh")
print(f" Percent decrease in means: {percent_diff}%")
```
'ERCOTRTPRICE' mean of original dataframe: 33.8057095534911 $/MWh
'ERCOTRTPRICE' mean of new dataframe: 18.228114464915684 $/MWh
 **Difference in means: 15.577595088575414 $/MWh**
 **Percent decrease in means: 53.92%**

## Model Preparation and Feature Creation

First, I check correlations using a heatmap from the Seaborn package

<img src="images/solar_curtailment/correlation_heatmap.jpg?raw=true"/>

I can see that “HBWESTRTPRICE” virtually equals “ERCOTRTPRICE” and will be excluded from the prediction. I assume this information would not be available at the time of prediction.


Next, I create numerical features to represent the date/time column in a way that our model can interpret and use for prediction. I pulled information from week, day, weekday, hour, and month.

```python
#Create numerical time and date features from the parsed 'DATETIME' column


ercot_cleaned['week'] = ercot_cleaned['DATETIME_PARSED'].dt.week
ercot_cleaned['day'] = ercot_cleaned['DATETIME_PARSED'].dt.day
ercot_cleaned['weekday'] = ercot_cleaned['DATETIME_PARSED'].dt.weekday
ercot_cleaned['hour'] = ercot_cleaned['DATETIME_PARSED'].dt.hour
ercot_cleaned['month'] = ercot_cleaned['DATETIME_PARSED'].dt.month
```

Barplot to compare how much power is used in each month. As expected, less energy is used in the winter months in Texas. (Likely due to less need for AC)

<img src="images/solar_curtailment/month_barplot.jpg?raw=true"/>

After this, I will create a discrete target variable which tells our model whether an instance encountered solar curtailment. This column will take the value 1 if solar curtailment occurs, and 0 otherwise. Just as a reminder, solar curtailment occurs in this dataset when 'ERCOTRTPRICE' is less than $0/MWh.

```python
ercot_cleaned['SOLAR_CURTAILMENT'] = np.where(ercot_cleaned['ERCOTRTPRICE'] < 0, 1,0)

solar_curtailment_values = ercot_cleaned['SOLAR_CURTAILMENT'].value_counts()

print(f'Percent of instances with solar curtailment: \
{round((solar_curtailment_values[1]/solar_curtailment_values[0]) * 100,3)} %')
```
**Percent of instances with solar curtailment: 0.401 %**

As the last step of model prep, I split the dataframe into X (predictor variables) and y (target variable). I'm choosing not to include "HBWESTRTPRICE" due to target leakage as we saw in the correlation heatmap it almost perfectly predicts 'ERCOTRTPRICE' and I assume this information wouldn't be available at the time of prediction.

```python
included_variables = ['week',
                   'day',
                   'weekday',
                   'month',
                   'hour',
                   'RT_OR_PRADDER',
                   'RT_ON_CAP',
                   'HASL',
                   'RTLOAD',
                   'PVACTUAL',
                   'PVFORECASTED',
                   'WINDACTUAL',
                   'WINDFORECASTED']

X = ercot_cleaned[included_variables]

y = ercot_cleaned['SOLAR_CURTAILMENT']
```

## Modeling with Logistic Regression

First I'll do a train-test split and make sure to stratify the target to ensure both sets have a proportional amount of positive instances.

```python
X_train, X_test, y_train, y_test = train_test_split(X,y, train_size = 0.7, stratify = y, random_state = 0)
```

Next, I will create the model, fit to the training set, and predict the test set.

```python
#create object for regression

logiReg = LogisticRegression()

#train the data set

logiReg.fit(X_train, y_train)  

#predict the test set

prediction = logiReg.predict(X_test)
```

Let's take a look at the model's performance using two different thresholds.

<img src="images/solar_curtailment/model_eval.jpg?raw=true"/>

Above are classification reports for two thresholds: 0.5 and 0.3. Due to the nature of this prediction task, accuracy is misleading as a performance metric. We could simply classify each instance as negative (no solar curtailment) and achieve accuracy of about 99.6%, but that wouldn’t work for detecting solar curtailment. Instead, I will measure performance based on **precision and recall for the positive class.** Basically, **precision measures how accurate the model performs when it predicts solar curtailment** while **recall measures how well the model detects solar curtailment from the entire dataset.**

I will also create a Precision-Recall Curve for my model.

<img src="images/solar_curtailment/P-R_curve.jpg?raw=true"/>

This plot shows the relationship between precision (correctly classified positive instances/all classified positive instances) and recall (correctly classified positive instances/all positive instances in the dataset) for the positive class with ***varying thresholds.*** The plot shows precision and recall values as the threshold goes from 1 to 0 (left to right). This makes intuitive sense because if the threshold is 1, the model won’t classify any instances as positive and thus won’t incorrectly classify anything as positive (100% precision). If the threshold is 0, the model predicts every instance as positive and thus won’t miss any of the positive instances present in the dataset (100% recall). **The key is to find the appropriate tradeoff between these metrics.**

### Discussion of Results

#### Why I chose Logisitic Regression for this task

I chose logistic regression because it meets the task of classifying instances between two classes while remaining highly interpretable. With machine learning explainability packages such as SHAP, I can convey what went into each prediction the model makes. I can also view the importance of each predictor variable using permutation importance.

#### Alternative Modeling Approaches to Consider.

I would also consider boosted random trees due to their superior computational power and ability to expand a dataset with bootstrap sampling. This is especially important in an unbalanced dataset such as this one. However, I chose not to use this model due to it being more complicated to explain the results of the prediction. Trusting important business/investment decisions to a “black box” model can prove risky.

#### How I Could Improve Model Accuracy

The biggest issue with this data set would be its size. If I had data on a couple years worth of energy production, I could see how energy demand varies across each month (instead of just July through December). In addition, I could split the training and testing data based on year instead of a random split as I did here. Splitting the data based on time so that we train on earlier data and test on the following data is more realistic in a production setting where the model would have to forecast future trends. Another action I would advise to improve performance would be to up sample the positive instances. This means I would artificially increase the number of positive instances in the dataset so that the model can train more thoroughly on detecting them. Again, more data would eliminate the need for up sampling.

Another factor that is more of a clarifying issue instead of an issue with the data would be a better understanding of when these variables are made available to the model. I assumed that “HBWESTRTPRICE” would not be available, but technically any of the real-time variables provided may not be available at the time of prediction. Verifying which of these variables would not be available and removing them could improve performance in a production setting by reducing target leakage.

#### How I Would Use this Model/Analysis to Inform Investment or Operational Decisions

If I was a solar power plant investor, I would apply a curtailment forecasting tool such as this model to forecast how often I would expect curtailment for a range of power plants. Considering curtailment creates negative pricing and inefficiencies, I would invest in plants with the lowest possibility for curtailment in the future.

If I was a solar power plant operator on the other hand, I would use this model to limit future curtailment by forecasting likelihood for curtailment in 5-minute increments. If the probability of curtailment exceeds a set threshold, I could limit the power generated in an attempt to avoid the curtailment. I would use real-time information on the local market operator’s negative pricing model to determine the importance of limiting curtailment. If the negative pricing is severe, I would lower my threshold and limit power generated more often.

### The implications and high level conclusions from this analysis related to market design, price signals, and grid planning

In terms of solar market design, my analysis reveals how connected different renewable energy markets are. For example, when the wind energy generation was above the 3rd quartile, solar energy prices and demand fell by around 50%. This leads me to the conclusion that observing local renewable energy markets before investing in a solar power plant would be advisable.

Price signals play an important role in determining solar power plant investment. The severity of negative pricing for solar curtailment determines how important/unimportant it is to correctly predict and prevent solar curtailment. I could see from my analysis that high solar energy generation is a result of higher real-time prices. Monitoring price signals can thus inform us of energy demand in a prospective market.

From this analysis and task of identifying solar curtailment, I can see that a vital component of grid planning is balancing energy generation to consumption. Forecasting energy demand can drive decisions such as location and size for potential future solar arrays.

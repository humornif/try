# Movie Recommendation - Matrix Factorization problem sample

| ML.NET version | API type          | Status                        | App Type    | Data type | Scenario            | ML Task                   | Algorithms                  |
|----------------|-------------------|-------------------------------|-------------|-----------|---------------------|---------------------------|-----------------------------|
| v0.9   | Dynamic API | Updated to v0.9 | Console app | .csv files | Recommendation | Matrix Factorization | MatrixFactorizationTrainer|

In this sample, you can see how to use ML.NET to build a movie recommendation engine. 


## Problem
For this tutorial we will use the MovieLens dataset which comes with movie ratings, titles, genres and more.  In terms of an approach for building our movie recommendation engine we will use Factorization Machines which uses a collaborative filtering approach. 

‘Collaborative filtering’ operates under the underlying assumption that if a person A has the same opinion as a person B on an issue, A is more likely to have B’s opinion on a different issue than that of a randomly chosen person. 

## DataSet
The original data comes from MovieLens Dataset:
http://files.grouplens.org/datasets/movielens/ml-latest-small.zip

## ML task - [Matrix Factorization (Recommendation)](https://docs.microsoft.com/en-us/dotnet/machine-learning/resources/tasks#recommendation)

The ML Task for this sample is Matrix Factorization, which is a supervised machine learning task performing collaborative filtering. 

## Solution

To solve this problem, you build and train an ML model on existing training data, evaluate how good it is (analyzing the obtained metrics), and lastly you can consume/test the model to predict the demand given input data variables.

### 1. Build model

Building a model includes: 

* Define the data's schema mapped to the datasets to read (`recommendation-ratings-train.csv` and `recommendation-ratings-test.csv`) with a DataReader

* Matrix Factorization requires the two features userId, movieId to be encoded

* Matrix Factorization trainer then takes these two encoded features (userId, movieId) as input 

Here's the code which will be used to build the model:
```csharp --project ./MovieRecommendation/MovieRecommendation/MovieRecommendation.csproj --session "Recommend!" --source-file ./MovieRecommendation/MovieRecommendation/Program.cs --region build_model
var reader = mlcontext.Data.CreateTextReader(new TextLoader.Arguments()
{
    Separator = ",",
    HasHeader = true,
    Column = new[]
                {
                    new TextLoader.Column("userId", DataKind.R4, 0),
                    new TextLoader.Column("movieId", DataKind.R4, 1),
                    new TextLoader.Column("Label", DataKind.R4, 2)
                }
});

var trainingDataView = reader.Read(TrainingDataLocation);

var pipeline = mlcontext.Transforms.Conversion.MapValueToKey("userId", "userIdEncoded")
               .Append(mlcontext.Transforms.Conversion.MapValueToKey("movieId", "movieIdEncoded"))
               .Append(mlcontext.Recommendation().Trainers.MatrixFactorization("userIdEncoded", "movieIdEncoded", "Label", advancedSettings: s => { s.NumIterations = 20; s.K = 100; }));
```


### 2. Train model
Training the model is a process of running the chosen algorithm on a training data (with known movie and user ratings) to tune the parameters of the model. It is implemented in the `Fit()` method from the Estimator object. 

To perform training you need to call the `Fit()` method while providing the training dataset (`recommendation-ratings-train.csv` file) in a DataView object.

```csharp --project ./MovieRecommendation/MovieRecommendation/MovieRecommendation.csproj --session "Recommend!" --source-file ./MovieRecommendation/MovieRecommendation/Program.cs --region train_model
var model = pipeline.Fit(trainingDataView);
```
Note that ML.NET works with data with a lazy-load approach, so in reality no data is really loaded in memory until you actually call the method .Fit().

### 3. Evaluate model
We need this step to conclude how accurate our model operates on new data. To do so, the model from the previous step is run against another dataset that was not used in training (`recommendation-ratings-test.csv`). 

`Evaluate()` compares the predicted values for the test dataset and produces various metrics, such as accuracy, you can explore.

```csharp --project ./MovieRecommendation/MovieRecommendation/MovieRecommendation.csproj --session "Recommend!" --source-file ./MovieRecommendation/MovieRecommendation/Program.cs --region evaluate_model
var testDataView = reader.Read(TestDataLocation);
var prediction = model.Transform(testDataView);
var metrics = mlcontext.Regression.Evaluate(prediction, label: "Label", score: "Score");
Console.WriteLine($"The model evaluation metrics rms: {Math.Round(metrics.Rms, 1)}");
```

### 4. Consume model
After the model is trained, you can use the `Predict()` API to predict the rating for a particular movie/user combination. 
```csharp --project ./MovieRecommendation/MovieRecommendation/MovieRecommendation.csproj --session "Recommend!" --source-file ./MovieRecommendation/MovieRecommendation/Program.cs --region prediction       
var userId = 6;
var movieId = 10;

var predictionengine = model.CreatePredictionEngine<MovieRating, MovieRatingPrediction>(mlcontext);
/* Make a single movie rating prediction, the scores are for a particular user and will range from 1 - 5. 
   The higher the score the higher the likelihood of a user liking a particular movie.
   You can recommend a movie to a user if say rating > 3.5.*/
var movieratingprediction = predictionengine.Predict(
    new MovieRating()
    {
                    //Example rating prediction for userId = 6, movieId = 10 (GoldenEye)
                    userId = userId,
        movieId = movieId
    }
);

var movieService = new Movie();
Console.WriteLine($"For userId: {userId} movie rating prediction (1 - 5 stars) for movie: {movieService.Get(movieId).movieTitle} is: {Math.Round(movieratingprediction.Score, 0, MidpointRounding.ToEven)}");
```
Please note this is one approach for performing movie recommendations with Matrix Factorization. There are other scenarios for recommendation as well which we will build samples for as well. 


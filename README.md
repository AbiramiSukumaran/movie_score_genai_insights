# movie_score_genai_insights
Build a Movie Score prediction and prescription insights with BigQuery and Vertex AI PaLM API.

**Dataset**
In this use case, we will use the [movies_data]([url](https://github.com/AbiramiSukumaran/movie_score_genai_insights/blob/main/movies_data.csv)) dataset derived from [movielens]([url](https://grouplens.org/datasets/movielens/1m/)) source.

**Data to ML**
1. Movies dataset stored in BigQuery
2. Movie Score / User Rating Prediction Model based on GENRE and RUNTIME with BQML trained
3. SQL-only prediction of user rating and result analysis
4. Deploying your model in Vertex AI for a REST endpoint
5. Storing prediction result in Big Query

**STEPS**
Follow this codelab:
https://codelabs.developers.google.com/moviescore-prediction-bqmlsql

_Except in step 6, for model creation, use the following query instead:_

CREATE OR REPLACE MODEL
  `movies.movies_score_model`
OPTIONS
  ( model_type='LOGISTIC_REG',
    auto_class_weights=TRUE,
   data_split_method='NO_SPLIT',
    input_label_cols=['score']
  ) AS
SELECT name, genre,runtime, score
FROM
  movies.movies_score
WHERE
  data_cat = 'TRAIN';

_And in the same step, instead of the PREDICT query in the lab, use the below:_

SELECT
  *
FROM
  ML.PREDICT (MODEL movies.movies_score_model,
    (
    SELECT
      *
    FROM
      movies.movies_rating
    WHERE
      data_cat= 'TEST'
     )
  );


_To write the predicted result into another BigQuery table, use the following query:_

CREATE TABLE movies.predicted_movies_score as (
SELECT
  *
FROM
  ML.PREDICT (MODEL movies.movies_score_model,
    (
    SELECT
      *
    FROM
      movies.movies_rating
    WHERE
      data_cat= 'TEST'
     )
  )
);

**Data to Generative AI**
1. Analyze other factors besides genre and runtime that are influencing the predicted movie rating and summarize it with Generative AI using text-bison (latest) model using only sql queries 
     a. The table with the predicted results from the Data to ML step is the input for this
     b. BigQuery GENERATE_TEXT construct will be used to invoke the PaLM API remotely from Vertex AI
     c. External Connection will be created to establish the access between BigQuery ML and Vertex services.
   
**STEPS**
Follow this codelab starting **from step 6** (because we already have our dataset ready):
https://codelabs.developers.google.com/llm-codesummarizer-sqlonly

_Except for step 8: Instead of the query given there, use this:_


CREATE OR REPLACE MODEL
  movies.llm_model REMOTE
WITH CONNECTION `us-central1.bq_llm_connection` OPTIONS (remote_service_type = 'CLOUD_AI_LARGE_LANGUAGE_MODEL_V1');


_SKIP step 9.

Except for step 10: Instead of the query given there, to use the PaLM API to perform LLM on the dataset use the following query:_

SELECT
  *
FROM
  ML.GENERATE_TEXT( MODEL movies.llm_model,
    (
    SELECT
      CONCAT('FROM THE FOLLOWING TEXT ABOUT MOVIES, WHAT DO YOU THINK ARE THE FACTORS INFLUENCING A MOVIE SCORE TO BE GREATER THAN 5?: ', movie_data) AS prompt
    FROM (
      SELECT
        REPLACE(STRING_AGG( CONCAT('A movie named ',name, ' from the country ', country, ' with a censor rating of ',rating, ' and a budget of ', budget, ' produced by ', company, ' with a runtime of about ', runtime, ' and in the genre ', genre, ' starring ', star, ' has had a success score of ', score, '') ), ',','. ') AS movie_data
      FROM (
        SELECT
          *
        FROM
          `abis-345004.movies.movies_rating`
        WHERE
          CAST(SCORE AS INT64) > 5
        LIMIT
          50) ) AS MOVIES),
    STRUCT( 0.2 AS temperature,
      100 AS max_output_tokens,
      TRUE AS flatten_json_output));


  This is what your result should look like:

   Based on the given information, the following factors seem to influence a movie score to be greater than 5:

- **Genre**: All the movies with a success score greater than 5 belong to the crime genre

- **Censor Rating**: All the movies with a success score greater than 5 have a censor rating of R... 
  

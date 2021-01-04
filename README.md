# CA4022_Recommender_System
Recommender System using Apache Spark on Hadoop Cluster for CA4022


Please find my assignment here - https://weldonj.github.io/CA4022_Recommender_System/Intro_and_Desc_of_Data.html

{
 "cells": [
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# Collaborative Filtering Recommender System"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## Import Modules, create Spark session and load in the Dataset"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 285,
   "metadata": {},
   "outputs": [],
   "source": [
    "import pandas as pd\n",
    "import pyspark.sql.functions as func\n",
    "from pyspark.sql.types import DateType, IntegerType,NumericType\n",
    "from pyspark.sql.functions import min, max, col\n",
    "from pyspark.sql import SparkSession\n",
    "from pyspark.sql.types import DoubleType\n",
    "from pyspark.sql.functions import *\n",
    "from pyspark.sql.window import Window\n",
    "from pyspark.ml.feature import QuantileDiscretizer\n",
    "from pyspark.ml.feature import Bucketizer\n",
    "from pyspark.ml.evaluation import RegressionEvaluator\n",
    "from pyspark.ml import Pipeline\n",
    "from pyspark.ml.classification import LogisticRegression\n",
    "from pyspark.ml.evaluation import BinaryClassificationEvaluator\n",
    "from pyspark.ml.feature import HashingTF, Tokenizer\n",
    "from pyspark.ml.tuning import CrossValidator, ParamGridBuilder,CrossValidatorModel\n",
    "\n",
    "# Create our Spark Session\n",
    "spark = SparkSession.builder.appName('recnn').getOrCreate()"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "Once the spark session was created, it was time to read in the dataset from the HDFS as below"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 286,
   "metadata": {},
   "outputs": [],
   "source": [
    "df = spark.read.csv(\"hdfs://localhost:9000/john/hadoop/input/steam-200k.csv\")"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## Quick look at the Data and tidying\n",
    "\n",
    "Now I wanted to take a quick look at the dataset and what the rows looked like"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 287,
   "metadata": {},
   "outputs": [
    {
     "data": {
      "text/plain": [
       "[Row(_c0='151603712', _c1='The Elder Scrolls V Skyrim', _c2='purchase', _c3='1.0', _c4='0'),\n",
       " Row(_c0='151603712', _c1='The Elder Scrolls V Skyrim', _c2='play', _c3='273.0', _c4='0'),\n",
       " Row(_c0='151603712', _c1='Fallout 4', _c2='purchase', _c3='1.0', _c4='0'),\n",
       " Row(_c0='151603712', _c1='Fallout 4', _c2='play', _c3='87.0', _c4='0'),\n",
       " Row(_c0='151603712', _c1='Spore', _c2='purchase', _c3='1.0', _c4='0'),\n",
       " Row(_c0='151603712', _c1='Spore', _c2='play', _c3='14.9', _c4='0'),\n",
       " Row(_c0='151603712', _c1='Fallout New Vegas', _c2='purchase', _c3='1.0', _c4='0'),\n",
       " Row(_c0='151603712', _c1='Fallout New Vegas', _c2='play', _c3='12.1', _c4='0'),\n",
       " Row(_c0='151603712', _c1='Left 4 Dead 2', _c2='purchase', _c3='1.0', _c4='0'),\n",
       " Row(_c0='151603712', _c1='Left 4 Dead 2', _c2='play', _c3='8.9', _c4='0'),\n",
       " Row(_c0='151603712', _c1='HuniePop', _c2='purchase', _c3='1.0', _c4='0'),\n",
       " Row(_c0='151603712', _c1='HuniePop', _c2='play', _c3='8.5', _c4='0'),\n",
       " Row(_c0='151603712', _c1='Path of Exile', _c2='purchase', _c3='1.0', _c4='0'),\n",
       " Row(_c0='151603712', _c1='Path of Exile', _c2='play', _c3='8.1', _c4='0'),\n",
       " Row(_c0='151603712', _c1='Poly Bridge', _c2='purchase', _c3='1.0', _c4='0')]"
      ]
     },
     "execution_count": 287,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "df.head(15)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "And now a look at some basic stats, including the count of how many entries we have in the dataset (200,000) the maximum play time of any game by any user (999 hours) and the minimum play time of any game by any user (0.1 hours)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 288,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+-------+-------------------+----------------+--------+------------------+------+\n",
      "|summary|                _c0|             _c1|     _c2|               _c3|   _c4|\n",
      "+-------+-------------------+----------------+--------+------------------+------+\n",
      "|  count|             200000|          200000|  200000|            200000|200000|\n",
      "|   mean|  1.0365586594664E8|           140.0|    null|17.874383999999942|   0.0|\n",
      "| stddev|7.208073512913969E7|             0.0|    null|138.05695165086743|   0.0|\n",
      "|    min|          100012061|     007 Legends|    play|               0.1|     0|\n",
      "|    max|           99992274|theHunter Primal|purchase|             999.0|     0|\n",
      "+-------+-------------------+----------------+--------+------------------+------+\n",
      "\n"
     ]
    }
   ],
   "source": [
    "df.describe().show()"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "I now wanted to drop column 'c4' as it is not clear what it actually represents, and I also don't need whatever it may be for collaborative filtering"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 290,
   "metadata": {},
   "outputs": [],
   "source": [
    "df = df[('_c0', '_c1', '_c2', '_c3')]"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "It was necessary to remove the rows that are for purchases instead of play times. These rows also weren't needed for my collaborative filtering, I just needed to know how much time each user spent playing each game."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 292,
   "metadata": {},
   "outputs": [],
   "source": [
    "df = df[df['_c2'] != 'purchase']"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "So I was now left with just our user ID, game title and how many hours they have played the game for"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 293,
   "metadata": {},
   "outputs": [
    {
     "data": {
      "text/plain": [
       "Row(_c0='151603712', _c1='The Elder Scrolls V Skyrim', _c2='play', _c3='273.0')"
      ]
     },
     "execution_count": 293,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "df.head()"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "I now created a 'rating' column that was initially just a duplicate of the play duration column, and also a 'rating_double' column which contained a double version of this instead of string"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 297,
   "metadata": {},
   "outputs": [],
   "source": [
    "df = df.withColumn('rating', df._c3)\n",
    "changedTypedf = df.withColumn(\"rating_double\", df[\"rating\"].cast(DoubleType()))"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "Looking at the Schema at this point, I now had a column containing the play times as doubles"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 298,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "root\n",
      " |-- _c0: string (nullable = true)\n",
      " |-- _c1: string (nullable = true)\n",
      " |-- _c2: string (nullable = true)\n",
      " |-- _c3: string (nullable = true)\n",
      " |-- rating: string (nullable = true)\n",
      " |-- rating_double: double (nullable = true)\n",
      "\n"
     ]
    }
   ],
   "source": [
    "changedTypedf.printSchema()"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## Create a derived Ratings column\n",
    "\n",
    "The next six cells created a dataframe that gave me the total hours played by any single gamer (gamer_total_hours_played), and what percentage of that total is made up by each game they played (perc_of_gamer_total_hours)."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 300,
   "metadata": {},
   "outputs": [],
   "source": [
    "hours_by_game = changedTypedf.groupBy(\"_c0\", \"_c1\").sum(\"rating_double\")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 301,
   "metadata": {},
   "outputs": [],
   "source": [
    "hours_by_gamer = changedTypedf.groupBy(\"_c0\").sum(\"rating_double\")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 302,
   "metadata": {},
   "outputs": [],
   "source": [
    "hours_by_gamer = hours_by_gamer.withColumnRenamed(\"sum(rating_double)\", \"gamer_total_hours_played\")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 303,
   "metadata": {},
   "outputs": [],
   "source": [
    "hours_by_gamer = hours_by_gamer.withColumnRenamed(\"_c0\", \"gamer\")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 304,
   "metadata": {},
   "outputs": [],
   "source": [
    "hours_by_gamer_by_game = hours_by_game.alias(\"a\").join(hours_by_gamer\\\n",
    "                     .alias(\"b\"),hours_by_game['_c0'] == hours_by_gamer['gamer'],how='left')"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 305,
   "metadata": {},
   "outputs": [],
   "source": [
    "hours_by_gamer_by_game = hours_by_gamer_by_game.withColumn(\"perc_of_gamer_total_hours\", \n",
    "                                                         hours_by_gamer_by_game[\"sum(rating_double)\"]/ hours_by_gamer_by_game[\"gamer_total_hours_played\"] )\n"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 306,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+---------+--------------------+------------------+---------+------------------------+-------------------------+\n",
      "|      _c0|                 _c1|sum(rating_double)|    gamer|gamer_total_hours_played|perc_of_gamer_total_hours|\n",
      "+---------+--------------------+------------------+---------+------------------------+-------------------------+\n",
      "| 26122540|Call of Duty Blac...|              22.0| 26122540|                   182.3|      0.12068019747668678|\n",
      "|126340495|     Team Fortress 2|              16.1|126340495|      2770.8999999999996|     0.005810386517016133|\n",
      "| 97298878|            Mafia II|              15.9| 97298878|                   220.6|      0.07207615593834996|\n",
      "| 11373749|   Fate of the World|               2.3| 11373749|      3943.2000000000016|     5.832826131061064E-4|\n",
      "| 46759859|       Day of Defeat|               5.8| 46759859|                   812.8|     0.007135826771653544|\n",
      "| 32592631|  Duke Nukem Forever|               8.8| 32592631|                   286.7|     0.030694105336588774|\n",
      "| 57103808|   Company of Heroes|              89.0| 57103808|      1276.7000000000005|      0.06971097360382232|\n",
      "| 94088853|       ORION Prelude|              13.5| 94088853|       920.4000000000001|      0.01466753585397653|\n",
      "|  9823354|Europa Universali...|               2.1|  9823354|       2738.899999999997|     7.667311694475893E-4|\n",
      "|249165768|Battlefield Bad C...|               4.0|249165768|                   695.9|     0.005747952291995977|\n",
      "+---------+--------------------+------------------+---------+------------------------+-------------------------+\n",
      "only showing top 10 rows\n",
      "\n"
     ]
    }
   ],
   "source": [
    "hours_by_gamer_by_game.show(10)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "Because I now had this information, I was then able to add a 'rank' column which will order, from 1 to n, the games played by each gamer in terms of how much time they spent playing them. "
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 307,
   "metadata": {},
   "outputs": [],
   "source": [
    "ranked =  hours_by_gamer_by_game.withColumn(\"rank\", dense_rank().over(Window.partitionBy(\"gamer\").orderBy(desc(\"perc_of_gamer_total_hours\"))))"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 308,
   "metadata": {},
   "outputs": [],
   "source": [
    "ranked = ranked.withColumn(\"gamer_total_hours_played\", func.round(ranked[\"gamer_total_hours_played\"], 2))\n",
    "ranked = ranked.withColumn(\"perc_of_gamer_total_hours\", func.round(ranked[\"perc_of_gamer_total_hours\"], 2))"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 309,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+---------+--------------------+------------------+---------+------------------------+-------------------------+----+\n",
      "|      _c0|                 _c1|sum(rating_double)|    gamer|gamer_total_hours_played|perc_of_gamer_total_hours|rank|\n",
      "+---------+--------------------+------------------+---------+------------------------+-------------------------+----+\n",
      "|108750879|       Borderlands 2|             758.0|108750879|                  1807.5|                     0.42|   1|\n",
      "|108750879|The Elder Scrolls...|             511.0|108750879|                  1807.5|                     0.28|   2|\n",
      "|108750879|Call of Duty Mode...|             149.0|108750879|                  1807.5|                     0.08|   3|\n",
      "|108750879|       Saints Row IV|             117.0|108750879|                  1807.5|                     0.06|   4|\n",
      "|108750879|           Fallout 4|              77.0|108750879|                  1807.5|                     0.04|   5|\n",
      "|108750879|   Fallout New Vegas|              71.0|108750879|                  1807.5|                     0.04|   6|\n",
      "|108750879|Saints Row The Third|              33.0|108750879|                  1807.5|                     0.02|   7|\n",
      "|108750879|    Metro Last Light|              26.0|108750879|                  1807.5|                     0.01|   8|\n",
      "|108750879|              Dota 2|              25.0|108750879|                  1807.5|                     0.01|   9|\n",
      "|108750879|          F.E.A.R. 3|              17.2|108750879|                  1807.5|                     0.01|  10|\n",
      "+---------+--------------------+------------------+---------+------------------------+-------------------------+----+\n",
      "only showing top 10 rows\n",
      "\n"
     ]
    }
   ],
   "source": [
    "ranked.show(10)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "Above we can now the top 10 played games for gamer '108750879'"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 310,
   "metadata": {},
   "outputs": [],
   "source": [
    "new_ranked = ranked.drop(\"sum(rating_double)\")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 311,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+---------+--------------------+---------+------------------------+-------------------------+----+\n",
      "|      _c0|                 _c1|    gamer|gamer_total_hours_played|perc_of_gamer_total_hours|rank|\n",
      "+---------+--------------------+---------+------------------------+-------------------------+----+\n",
      "|108750879|       Borderlands 2|108750879|                  1807.5|                     0.42|   1|\n",
      "|108750879|The Elder Scrolls...|108750879|                  1807.5|                     0.28|   2|\n",
      "|108750879|Call of Duty Mode...|108750879|                  1807.5|                     0.08|   3|\n",
      "|108750879|       Saints Row IV|108750879|                  1807.5|                     0.06|   4|\n",
      "|108750879|           Fallout 4|108750879|                  1807.5|                     0.04|   5|\n",
      "|108750879|   Fallout New Vegas|108750879|                  1807.5|                     0.04|   6|\n",
      "|108750879|Saints Row The Third|108750879|                  1807.5|                     0.02|   7|\n",
      "|108750879|    Metro Last Light|108750879|                  1807.5|                     0.01|   8|\n",
      "|108750879|              Dota 2|108750879|                  1807.5|                     0.01|   9|\n",
      "|108750879|          F.E.A.R. 3|108750879|                  1807.5|                     0.01|  10|\n",
      "|108750879|Call of Duty Mode...|108750879|                  1807.5|                     0.01|  11|\n",
      "|108750879|            Warframe|108750879|                  1807.5|                      0.0|  12|\n",
      "|108750879|     Team Fortress 2|108750879|                  1807.5|                      0.0|  13|\n",
      "|108750879|Blacklight Tango ...|108750879|                  1807.5|                      0.0|  14|\n",
      "|108750879|             Nosgoth|108750879|                  1807.5|                      0.0|  14|\n",
      "|109278470|Counter-Strike Gl...|109278470|                    76.0|                     0.99|   1|\n",
      "|109278470|Half-Life 2 Episo...|109278470|                    76.0|                     0.01|   2|\n",
      "|109278470|     Team Fortress 2|109278470|                    76.0|                     0.01|   2|\n",
      "|110840027|     Team Fortress 2|110840027|                     0.2|                      1.0|   1|\n",
      "|121154091|     Team Fortress 2|121154091|                     3.0|                      1.0|   1|\n",
      "+---------+--------------------+---------+------------------------+-------------------------+----+\n",
      "only showing top 20 rows\n",
      "\n"
     ]
    }
   ],
   "source": [
    "new_ranked.show(20)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "The Spark ML function, 'QuantileDiscretizer' allowed me to assign a rating (0-9) from each gamer for each game, by separating the games they played into buckets based on the time spent playing. \n",
    "\n",
    "The game that they spent the most time playing would receive a 9 for example."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 312,
   "metadata": {},
   "outputs": [],
   "source": [
    "game_rated = QuantileDiscretizer(numBuckets=10, inputCol=\"perc_of_gamer_total_hours\",outputCol=\"rating\")\\\n",
    ".fit(hours_by_gamer_by_game).transform(hours_by_gamer_by_game)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 313,
   "metadata": {},
   "outputs": [],
   "source": [
    "ratings = game_rated.select(\"gamer\",\"_c1\", \"rating\",\"perc_of_gamer_total_hours\")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 314,
   "metadata": {},
   "outputs": [],
   "source": [
    "ratings = ratings.withColumnRenamed(\"_c1\", \"game\")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 315,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+---------+--------------------+------+-------------------------+\n",
      "|    gamer|                game|rating|perc_of_gamer_total_hours|\n",
      "+---------+--------------------+------+-------------------------+\n",
      "| 26122540|Call of Duty Blac...|   7.0|      0.12068019747668678|\n",
      "|126340495|     Team Fortress 2|   4.0|     0.005810386517016133|\n",
      "| 97298878|            Mafia II|   7.0|      0.07207615593834996|\n",
      "| 11373749|   Fate of the World|   1.0|     5.832826131061064E-4|\n",
      "| 46759859|       Day of Defeat|   4.0|     0.007135826771653544|\n",
      "| 32592631|  Duke Nukem Forever|   6.0|     0.030694105336588774|\n",
      "| 57103808|   Company of Heroes|   7.0|      0.06971097360382232|\n",
      "| 94088853|       ORION Prelude|   5.0|      0.01466753585397653|\n",
      "|  9823354|Europa Universali...|   1.0|     7.667311694475893E-4|\n",
      "|249165768|Battlefield Bad C...|   4.0|     0.005747952291995977|\n",
      "|130280718|              Dota 2|   7.0|      0.05423353624792473|\n",
      "| 25096601|              Dota 2|   8.0|      0.25513254933227025|\n",
      "| 25096601|Chivalry Medieval...|   2.0|     0.001195933824995...|\n",
      "| 82235763|           Homefront|   9.0|                      1.0|\n",
      "| 92914917|     Sniper Elite V2|   1.0|     4.014362496933477E-4|\n",
      "| 92914917|           Sanctum 2|   1.0|     3.345302080777897...|\n",
      "| 72842694|   Talisman Prologue|   3.0|     0.002284968173657...|\n",
      "| 72842694|             Strider|   2.0|     0.001958544148849358|\n",
      "|  4834220|           Fallout 4|   5.0|     0.009562445667922338|\n",
      "| 65958466|       Stronghold HD|   3.0|     0.004033699259637477|\n",
      "+---------+--------------------+------+-------------------------+\n",
      "only showing top 20 rows\n",
      "\n"
     ]
    }
   ],
   "source": [
    "ratings.show(20)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 316,
   "metadata": {},
   "outputs": [],
   "source": [
    "ratings = ratings.withColumn(\"gamer\", ratings[\"gamer\"].cast(IntegerType()))"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 317,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+---------+--------------------+------+-------------------------+\n",
      "|    gamer|                game|rating|perc_of_gamer_total_hours|\n",
      "+---------+--------------------+------+-------------------------+\n",
      "| 26122540|Call of Duty Blac...|   7.0|      0.12068019747668678|\n",
      "|126340495|     Team Fortress 2|   4.0|     0.005810386517016133|\n",
      "| 97298878|            Mafia II|   7.0|      0.07207615593834996|\n",
      "| 11373749|   Fate of the World|   1.0|     5.832826131061064E-4|\n",
      "| 46759859|       Day of Defeat|   4.0|     0.007135826771653544|\n",
      "| 32592631|  Duke Nukem Forever|   6.0|     0.030694105336588774|\n",
      "| 57103808|   Company of Heroes|   7.0|      0.06971097360382232|\n",
      "| 94088853|       ORION Prelude|   5.0|      0.01466753585397653|\n",
      "|  9823354|Europa Universali...|   1.0|     7.667311694475893E-4|\n",
      "|249165768|Battlefield Bad C...|   4.0|     0.005747952291995977|\n",
      "+---------+--------------------+------+-------------------------+\n",
      "only showing top 10 rows\n",
      "\n"
     ]
    }
   ],
   "source": [
    "ratings.show(10)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "It can be seen above that I now had the rating from each gamer for each game. I needed a game_id column for the model as it doesn't take in strings and so this next cell achieved this. It gave a unique game_id from 1 to n (n being the total number of games) to each game in the list"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 318,
   "metadata": {},
   "outputs": [],
   "source": [
    "ratings = ratings.withColumn(\"game_id\", func.dense_rank().over(Window.orderBy(ratings.game)))"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## Build and train the Model\n",
    "\n",
    "I was now able to split these ratings into training and test sets"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 319,
   "metadata": {},
   "outputs": [],
   "source": [
    "training, test = ratings.randomSplit([0.8,0.2], seed = 36)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "Next I defined the alternating least squares model for collaborative filtering"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 320,
   "metadata": {},
   "outputs": [],
   "source": [
    "from pyspark.ml.recommendation import ALS\n",
    "\n",
    "\n",
    "als = ALS(maxIter=10, regParam=0.01, userCol=\"gamer\", itemCol=\"game_id\", ratingCol=\"rating\",\n",
    "          coldStartStrategy=\"drop\", nonnegative = True)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "Fit the model to the training set"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 321,
   "metadata": {},
   "outputs": [],
   "source": [
    "model = als.fit(training)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "Use the model to make predictions on the test set"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 322,
   "metadata": {},
   "outputs": [],
   "source": [
    "predictions = model.transform(test)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 323,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+---------+--------------------+------+-------------------------+-------+----------+\n",
      "|    gamer|                game|rating|perc_of_gamer_total_hours|game_id|prediction|\n",
      "+---------+--------------------+------+-------------------------+-------+----------+\n",
      "|298890193|                  C9|   5.0|     0.008174386920980927|    463| 10.399544|\n",
      "|141773234|                  C9|   0.0|     5.068166844052507E-5|    463| 2.2671351|\n",
      "| 65229865|Europa Universali...|   8.0|       0.5202312138728324|   1088|  9.895632|\n",
      "| 58217836|Europa Universali...|   4.0|      0.00728108833109791|   1088| 3.2841067|\n",
      "| 75368808|Europa Universali...|   8.0|      0.22816053345276305|   1088|   6.60545|\n",
      "| 65958466|Europa Universali...|   4.0|     0.007658922644881285|   1088| 2.6655507|\n",
      "| 42695514|Football Manager ...|   5.0|     0.008433107756376887|   1238|  6.052161|\n",
      "| 40080833|Football Manager ...|   6.0|     0.037704231252618355|   1238| 5.8518333|\n",
      "|113260911|Football Manager ...|   7.0|      0.06747638326585695|   1238| 7.8499417|\n",
      "| 84925352|Football Manager ...|   7.0|      0.13810198300283288|   1238| 14.121485|\n",
      "| 81054122|Football Manager ...|   5.0|     0.015455950540958269|   1238| 12.906618|\n",
      "| 10302000|Football Manager ...|   7.0|      0.06990411382616765|   1238| 4.2817283|\n",
      "|131785763|Infinity Wars - A...|   1.0|     4.404316229905308E-4|   1580| 1.1740297|\n",
      "|120514307|Infinity Wars - A...|   8.0|       0.2422480620155039|   1580|  9.999819|\n",
      "| 14544587|Infinity Wars - A...|   0.0|     2.212090302441902...|   1580| 5.1255918|\n",
      "|131876863|Infinity Wars - A...|   0.0|     3.132832080200502E-4|   1580| 14.497024|\n",
      "| 69953893|Infinity Wars - A...|   0.0|     2.871912693854107E-4|   1580| 2.9354901|\n",
      "| 30519870|               Mafia|   1.0|       7.4162448005675E-4|   1829| 3.8883429|\n",
      "| 58345543|Steel Storm Burni...|   3.0|     0.003200232744199...|   2866|0.63559055|\n",
      "| 98561727|         Audiosurf 2|   6.0|     0.023094688221709007|    243|  16.11771|\n",
      "+---------+--------------------+------+-------------------------+-------+----------+\n",
      "only showing top 20 rows\n",
      "\n"
     ]
    }
   ],
   "source": [
    "predictions.show(20)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## Evaluate the Model"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 325,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "Root-mean-square error = 3.1969283493060523\n"
     ]
    }
   ],
   "source": [
    "evaluator = RegressionEvaluator(metricName='rmse', labelCol='rating')\n",
    "rmse = evaluator.evaluate(predictions)\n",
    "print(\"Root-mean-square error = \" + str(rmse))"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "Having evaluated the performance using root mean square error, I could see it was around 3. This isn't amazing for a range of 0 to 9 but it would still likely produce useful recommendations"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "## Recommendations"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "It was now time to get some actual recommendations based on the model. First I wanted to get the top 20 gamers in terms of hours played which I did in the next cell"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 333,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+---------+------------------+\n",
      "|      _c0|          sum(_c3)|\n",
      "+---------+------------------+\n",
      "| 73017395|           11754.0|\n",
      "| 10599862|11651.699999999999|\n",
      "|100630947|           10853.2|\n",
      "| 26762388|10470.100000000004|\n",
      "|153382649|            9640.0|\n",
      "| 43684632| 9546.299999999988|\n",
      "| 48798067| 9427.199999999997|\n",
      "| 52731290|            9417.6|\n",
      "| 42935819|            8172.9|\n",
      "| 14544587|8137.0999999999985|\n",
      "| 52567955|            7836.7|\n",
      "|130882834|            7801.9|\n",
      "| 57433226| 7741.499999999999|\n",
      "| 63615483| 7190.400000000001|\n",
      "| 47063596| 7161.000000000001|\n",
      "| 67694595| 7152.100000000001|\n",
      "| 17017968|            7108.5|\n",
      "| 50818751| 6929.899999999999|\n",
      "| 49893565|            6891.9|\n",
      "| 24721232|            6887.0|\n",
      "+---------+------------------+\n",
      "\n"
     ]
    }
   ],
   "source": [
    "top_gamers = changedTypedf.groupBy(changedTypedf['_c0']).agg({'_c3':\"sum\"}).sort(\"sum(_c3)\", ascending=False).dropna().limit(20)\n",
    "top_gamers.show(20)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "I placed these gamers into a list that I could then use"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 335,
   "metadata": {},
   "outputs": [
    {
     "data": {
      "text/plain": [
       "['73017395',\n",
       " '10599862',\n",
       " '100630947',\n",
       " '26762388',\n",
       " '153382649',\n",
       " '43684632',\n",
       " '48798067',\n",
       " '52731290',\n",
       " '42935819',\n",
       " '14544587',\n",
       " '52567955',\n",
       " '130882834',\n",
       " '57433226',\n",
       " '63615483',\n",
       " '47063596',\n",
       " '67694595',\n",
       " '17017968',\n",
       " '50818751',\n",
       " '49893565',\n",
       " '24721232']"
      ]
     },
     "execution_count": 335,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "top20_gamer_list = [row._c0 for row in top_gamers.select(\"_c0\").collect()]\n",
    "top20_gamer_list"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "Using this list and the model, I created a dictionary containing all 20 top gamers and the top 10 game recommendations for each"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 337,
   "metadata": {
    "scrolled": true
   },
   "outputs": [
    {
     "data": {
      "text/plain": [
       "{'73017395': [2521, 2074, 3007, 1184, 1127, 944, 871, 2151, 84, 2513],\n",
       " '10599862': [1999, 3404, 2797, 2301, 3406, 944, 916, 871, 3448, 650],\n",
       " '100630947': [966, 1440, 2797, 2513, 1705, 1788, 1092, 3118, 108, 2699],\n",
       " '26762388': [84, 2521, 108, 3404, 779, 1815, 1940, 1617, 2547, 3524],\n",
       " '153382649': [2074, 3404, 2634, 1999, 1113, 2521, 1184, 1790, 2301, 3406],\n",
       " '43684632': [2301, 3406, 84, 3404, 1292, 1127, 1113, 3054, 650, 2634],\n",
       " '48798067': [2151, 944, 650, 3406, 1127, 916, 2301, 1184, 871, 2634],\n",
       " '52731290': [2301, 3406, 650, 1292, 1127, 1113, 1999, 916, 2634, 608],\n",
       " '42935819': [2797, 3448, 2707, 2852, 2221, 966, 1440, 1492, 754, 2513],\n",
       " '14544587': [2797, 3448, 1999, 966, 2221, 2707, 2852, 1440, 3406, 3373],\n",
       " '52567955': [2699, 966, 668, 1857, 2377, 2797, 1440, 3391, 2828, 2353],\n",
       " '130882834': [1999, 2797, 3404, 2301, 3448, 3406, 966, 108, 84, 2513],\n",
       " '57433226': [779, 2828, 84, 3404, 2521, 1673, 108, 668, 2547, 1940],\n",
       " '63615483': [3007, 1999, 2074, 1609, 2134, 1806, 1087, 1869, 2491, 2297],\n",
       " '47063596': [3406, 2301, 1292, 1113, 2634, 1999, 608, 1672, 3054, 3384],\n",
       " '67694595': [3406, 2301, 3404, 1113, 1292, 84, 2634, 3524, 608, 1231],\n",
       " '17017968': [3007, 1999, 2074, 1609, 2134, 1869, 1806, 2297, 1087, 2772],\n",
       " '50818751': [3406, 2634, 1999, 2301, 650, 1790, 1113, 1292, 916, 3007],\n",
       " '49893565': [1184, 2074, 2521, 1424, 3007, 3021, 754, 944, 2634, 1816],\n",
       " '24721232': [1184, 2634, 3021, 2074, 2521, 754, 1492, 1424, 1790, 1816]}"
      ]
     },
     "execution_count": 337,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "gamer_recs = model.recommendForAllUsers(10)\n",
    "\n",
    "rec_game_top20_gamers = {}\n",
    "\n",
    "for gamer in top20_gamer_list:\n",
    "    rec_game = gamer_recs.where(gamer_recs.gamer == gamer).select(\"recommendations\").collect()\n",
    "    rec_game_top20_gamers[gamer] = [i.game_id for i in rec_game[0][\"recommendations\"]]\n",
    "\n",
    "rec_game_top20_gamers"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "I wanted a list of game titles and their game ids so that I could see what the actual game recommendations were"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 331,
   "metadata": {
    "scrolled": false
   },
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+-------+--------------------+\n",
      "|game_id|                game|\n",
      "+-------+--------------------+\n",
      "|      1|         007 Legends|\n",
      "|      2|           0RBITALIS|\n",
      "|      3|1... 2... 3... KI...|\n",
      "|      4|     10 Second Ninja|\n",
      "|      5|          10,000,000|\n",
      "|      6|   100% Orange Juice|\n",
      "|      7|           1000 Amps|\n",
      "|      8|12 Labours of Her...|\n",
      "|      9|12 Labours of Her...|\n",
      "|     10|12 Labours of Her...|\n",
      "|     11|                 140|\n",
      "|     12|             15 Days|\n",
      "|     13|        16bit Trader|\n",
      "|     14|1701 A.D. Sunken ...|\n",
      "|     15|18 Wheels of Stee...|\n",
      "|     16|1953 NATO vs Wars...|\n",
      "|     17|              1Quest|\n",
      "|     18|  3 Stars of Destiny|\n",
      "|     19|3089 -- Futuristi...|\n",
      "|     20|        3D Mini Golf|\n",
      "+-------+--------------------+\n",
      "only showing top 20 rows\n",
      "\n"
     ]
    }
   ],
   "source": [
    "game_list = ratings.select('game_id', 'game').dropDuplicates()\n",
    "game_list.show(20)"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "I took one of the gamers as user1, (gamer id '10599862') and had a look at the recommendations and also the actual games they played the most of"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 339,
   "metadata": {
    "scrolled": true
   },
   "outputs": [
    {
     "data": {
      "text/plain": [
       "DataFrame[game_id: int, game: string]"
      ]
     },
     "execution_count": 339,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "user1_recommend = game_list.filter(game_list[\"game_id\"].isin(rec_game_top20_gamers['10599862']))\n",
    "user1_recommend"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 340,
   "metadata": {
    "scrolled": true
   },
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+-------+--------------------+\n",
      "|game_id|                game|\n",
      "+-------+--------------------+\n",
      "|    650|Construction-Simu...|\n",
      "|    871|         Diaper Dash|\n",
      "|    916|Don Bradman Crick...|\n",
      "|    944|Dragon's Prophet ...|\n",
      "|   1999|             NBA 2K9|\n",
      "|   2301|Professional Farm...|\n",
      "|   2797|Speedball 2 Tourn...|\n",
      "|   3404|               WRC 5|\n",
      "|   3406|WTFast Gamers Pri...|\n",
      "|   3448|Warrior Kings Bat...|\n",
      "+-------+--------------------+\n",
      "\n"
     ]
    }
   ],
   "source": [
    "user1_recommend.show()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 352,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+--------+--------------------+------+-------------------------+-------+\n",
      "|   gamer|                game|rating|perc_of_gamer_total_hours|game_id|\n",
      "+--------+--------------------+------+-------------------------+-------+\n",
      "|10599862|   Crusader Kings II|   7.0|      0.05767398748680451|    701|\n",
      "|10599862|     Elite Dangerous|   7.0|      0.06402499206124429|   1033|\n",
      "|10599862|Football Manager ...|   7.0|      0.04986396834796639|   1236|\n",
      "|10599862|Mount & Blade War...|   7.0|      0.07552545980414876|   1969|\n",
      "|10599862|       Path of Exile|   7.0|      0.06737214312074634|   2198|\n",
      "|10599862|Total War ROME II...|   7.0|      0.09105967369568391|   3266|\n",
      "|10599862|       DARK SOULS II|   6.0|     0.017593999158921018|    727|\n",
      "|10599862|The Elder Scrolls...|   6.0|     0.018108945475767486|   3068|\n",
      "|10599862|    Total War ATTILA|   6.0|      0.03587459340697066|   3263|\n",
      "|10599862|The Witcher 3 Wil...|   6.0|     0.017336526000497784|   3180|\n",
      "+--------+--------------------+------+-------------------------+-------+\n",
      "\n"
     ]
    }
   ],
   "source": [
    "ratings.filter(ratings.gamer==\"10599862\").sort('rating', ascending=False).limit(10).show()"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "I could see that the recommendations passed a basic sanity test at least, Sports games and Fantasy games were being recommended to user1 and I could see that similar games appear in their top played games"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 342,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+-------+--------------------+\n",
      "|game_id|                game|\n",
      "+-------+--------------------+\n",
      "|    108|Air Conflicts Pac...|\n",
      "|    966| Duke Nukem Forever |\n",
      "|   1092|        EverQuest II|\n",
      "|   1440|Hacker Evolution ...|\n",
      "|   1705|LEGO Batman The V...|\n",
      "|   1788|Lost Planet Extre...|\n",
      "|   2513|Rugby League Team...|\n",
      "|   2699|Silent Hill Homec...|\n",
      "|   2797|Speedball 2 Tourn...|\n",
      "|   3118|             The Maw|\n",
      "+-------+--------------------+\n",
      "\n"
     ]
    }
   ],
   "source": [
    "user2_recommend = game_list.filter(game_list[\"game_id\"].isin(rec_game_top20_gamers['100630947']))\n",
    "user2_recommend.show()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 353,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+---------+--------------------+------+-------------------------+-------+\n",
      "|    gamer|                game|rating|perc_of_gamer_total_hours|game_id|\n",
      "+---------+--------------------+------+-------------------------+-------+\n",
      "|100630947|              Dota 2|   8.0|       0.9621125566653153|    923|\n",
      "|100630947|Counter-Strike Gl...|   6.0|      0.03731618324549441|    674|\n",
      "|100630947|        Aura Kingdom|   0.0|     1.842774481258983...|    247|\n",
      "|100630947|FreeStyle2 Street...|   0.0|     2.303468101573729...|   1264|\n",
      "|100630947|       Left 4 Dead 2|   0.0|     9.213872406294918E-6|   1734|\n",
      "|100630947|                Rust|   0.0|     3.132716618140272E-4|   2524|\n",
      "+---------+--------------------+------+-------------------------+-------+\n",
      "\n"
     ]
    }
   ],
   "source": [
    "ratings.filter(ratings.gamer==\"100630947\").sort('rating', ascending=False).limit(10).show()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 349,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+-------+--------------------+\n",
      "|game_id|                game|\n",
      "+-------+--------------------+\n",
      "|     84|Afterfall InSanit...|\n",
      "|    108|Air Conflicts Pac...|\n",
      "|    779|            Daylight|\n",
      "|   1617|Jagged Alliance F...|\n",
      "|   1815|            MLB 2K12|\n",
      "|   1940| Missing Translation|\n",
      "|   2521|    Runestone Keeper|\n",
      "|   2547|STORM Frontline N...|\n",
      "|   3404|               WRC 5|\n",
      "|   3524|X-Plane 10 Global...|\n",
      "+-------+--------------------+\n",
      "\n"
     ]
    }
   ],
   "source": [
    "user3_recommend = game_list.filter(game_list[\"game_id\"].isin(rec_game_top20_gamers['26762388']))\n",
    "user3_recommend.show()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 351,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "+--------+--------------------+------+-------------------------+-------+\n",
      "|   gamer|                game|rating|perc_of_gamer_total_hours|game_id|\n",
      "+--------+--------------------+------+-------------------------+-------+\n",
      "|26762388|Mount & Blade War...|   7.0|      0.07879580901806092|   1969|\n",
      "|26762388|The Elder Scrolls...|   7.0|      0.10410597797537746|   3068|\n",
      "|26762388|Chivalry Medieval...|   6.0|     0.032473424322594806|    560|\n",
      "|26762388|           Far Cry 3|   6.0|      0.02234935673966819|   1171|\n",
      "|26762388|Counter-Strike So...|   6.0|      0.03018118260570576|    676|\n",
      "|26762388|Age of Empires II...|   6.0|     0.021680786238908886|     93|\n",
      "|26762388| Assassin's Creed II|   6.0|     0.017382833019741924|    226|\n",
      "|26762388|Assassin's Creed ...|   6.0|     0.017669363234353055|    228|\n",
      "|26762388|              Dota 2|   6.0|     0.022444866811205232|    923|\n",
      "|26762388|  Dragon Age Origins|   6.0|     0.038681578972502635|    934|\n",
      "+--------+--------------------+------+-------------------------+-------+\n",
      "\n"
     ]
    }
   ],
   "source": [
    "ratings.filter(ratings.gamer==\"26762388\").sort('rating', ascending=False).limit(10).show()"
   ]
  },
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "I did the same for two more users and again I was happy enough with the reccomendations. At this point I was fairly confident that the system was doing its job and likely making some useful recommendations"
   ]
  }
 ],
 "metadata": {
  "kernelspec": {
   "display_name": "Python 3",
   "language": "python",
   "name": "python3"
  },
  "language_info": {
   "codemirror_mode": {
    "name": "ipython",
    "version": 3
   },
   "file_extension": ".py",
   "mimetype": "text/x-python",
   "name": "python",
   "nbconvert_exporter": "python",
   "pygments_lexer": "ipython3",
   "version": "3.6.10"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 4
}

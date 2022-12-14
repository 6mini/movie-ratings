# 프로젝트 소개
- 평점이 가장 높은 영화 30개를 추출하는 맵리듀스 프로그램 실습

## 데이터셋

### movies.csv

![image](https://user-images.githubusercontent.com/79494088/203214065-5d2800cf-a366-41d7-bbb9-51c4b60ee9fb.png)

### ratings.csv

![image](https://user-images.githubusercontent.com/79494088/203214133-91430ac3-196d-4c5e-821c-db688aacbb52.png)

## 출력 목표
- 평점 평균이 가장 높은 영화가 차례대로 TOP 30 전시

# JOIN & SORT

```java
// MovieAverageRateTopK.java
package com.fastcampus.hadoop;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.conf.Configured;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.MultipleInputs;
import org.apache.hadoop.mapreduce.lib.input.TextInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.Tool;
import org.apache.hadoop.util.ToolRunner;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.TreeMap;

public class MovieAverageRateTopK extends Configured implements Tool {
    enum Count {
        MAP_MOVIE_COUNT,
        REDUCE_MOVIE_COUNT
    }
    private final static int K = 30;

    public static class MovieMapper extends Mapper<Object, Text, Text, Text> {
        private Text movieId = new Text();
        private Text outValue = new Text();

        @Override
        protected void map(Object key, Text value, Context context) throws IOException, InterruptedException {
            // movieId,title,genres
            String[] columns = value.toString().split(",");
            if (columns[0].equals("movieId")) {
                return;
            }
            movieId.set(columns[0]);
            outValue.set("M" + columns[1]);
            context.write(movieId, outValue);
        }
    }

    public static class RatingMapper extends Mapper<Object, Text, Text, Text> {
        private Text movieId = new Text();
        private Text outValue = new Text();

        @Override
        protected void map(Object key, Text value, Context context) throws IOException, InterruptedException {
            // userId,movieId,rating,timestamp
            String[] columns = value.toString().split(",");
            if (columns[0].equals("userId")) {
                return;
            }
            movieId.set(columns[1]);
            outValue.set("R" + columns[2]);
            context.write(movieId, outValue);
        }
    }

    public static class MovieRatingJoinReducer extends Reducer<Text, Text, Text, Text> {
        private List<String> ratingList = new ArrayList<>();
        private Text movieName = new Text();
        private Text outValue = new Text();

        @Override
        protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
            ratingList.clear();

            for (Text value : values) {
                if (value.charAt(0) == 'M') {
                    movieName.set(value.toString().substring(1));
                } else if (value.charAt(0) == 'R') {
                    ratingList.add(value.toString().substring(1));
                }
            }

            double average = ratingList.stream().mapToDouble(Double::parseDouble).average().orElse(0.0);
            outValue.set(String.valueOf(average));
            context.write(movieName, outValue);
        }
    }

    public static class TopKMapper extends Mapper<Object, Text, Text, Text> {

        private TreeMap<Double, Text> topKMap = new TreeMap<>();

        @Override
        protected void map(Object key, Text value, Context context) throws IOException, InterruptedException {
            String[] columns = value.toString().split("\t");
            topKMap.put(Double.parseDouble(columns[1]), new Text(columns[0]));

            if (topKMap.size() > K) {
                topKMap.remove(topKMap.firstKey());
            }
        }

        @Override
        protected void cleanup(Context context) throws IOException, InterruptedException {
            for (Double k : topKMap.keySet()) {
                context.write(new Text(k.toString()), topKMap.get(k));
            }
        }
    }

    public static class TopKReducer extends Reducer<Text, Text, Text, Text> {

        private TreeMap<Double, Text> topKMap = new TreeMap<>();

        @Override
        protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
            for (Text value : values) {
                topKMap.put(Double.parseDouble(key.toString()), new Text(value));
                if (topKMap.size() > K) {
                    topKMap.remove(topKMap.firstKey());
                }
            }
        }

        @Override
        protected void cleanup(Context context) throws IOException, InterruptedException {
            for (Double k : topKMap.descendingKeySet()) {
                context.write(topKMap.get(k), new Text(k.toString()));
            }
        }
    }

    @Override
    public int run(String[] args) throws Exception {
        Configuration conf = getConf();
        Job job = Job.getInstance(conf, "MovieAverageRateTopN First");
        job.setJarByClass(MovieAverageRateTopK.class);
        job.setReducerClass(MovieRatingJoinReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(Text.class);

        MultipleInputs.addInputPath(job, new Path(args[0]), TextInputFormat.class, MovieMapper.class);
        MultipleInputs.addInputPath(job, new Path(args[1]), TextInputFormat.class, RatingMapper.class);

        FileOutputFormat.setOutputPath(job, new Path(args[2]));

        int returnCode = job.waitForCompletion(true) ? 0 : 1;
        if (returnCode == 0) {
            Job job2 = Job.getInstance(conf, "MovieAverageRateTopN First");
            job2.setJarByClass(MovieAverageRateTopK.class);
            job2.setMapperClass(TopKMapper.class);
            job2.setReducerClass(TopKReducer.class);
            job2.setNumReduceTasks(1);
            job2.setOutputKeyClass(Text.class);
            job2.setOutputValueClass(Text.class);

            FileInputFormat.addInputPath(job2, new Path(args[2]));
            FileOutputFormat.setOutputPath(job2, new Path(args[3]));
            return job2.waitForCompletion(true) ? 0 : 1;
        }

        return 1;
    }

    public static void main(String[] args) throws Exception {
        int exitCode = ToolRunner.run(new MovieAverageRateTopK(), args);
        System.exit(exitCode);
    }
}

```

## 실행

```s
$ jps
"""
52130 ResourceManager
19876 
51782 DataNode
51672 NameNode
52617 Jps
51929 SecondaryNameNode
52234 NodeManager
"""

$ hadoop fs -ls /user/fastcampus/
"""
-rw-r--r--   1 6mini supergroup      15217 2022-10-19 11:53 /user/fastcampus/LICENSE.txt
drwxr-xr-x   - 6mini supergroup          0 2022-11-01 12:26 /user/fastcampus/join
drwxr-xr-x   - 6mini supergroup          0 2022-11-07 11:29 /user/fastcampus/joinoutput1
drwxr-xr-x   - 6mini supergroup          0 2022-11-07 11:36 /user/fastcampus/joinoutput2
drwxr-xr-x   - 6mini supergroup          0 2022-10-19 11:58 /user/fastcampus/output
drwxr-xr-x   - 6mini supergroup          0 2022-11-07 11:17 /user/fastcampus/output2
drwxr-xr-x   - 6mini supergroup          0 2022-11-07 11:19 /user/fastcampus/sortoutput
"""

$ hadoop fs -mkdir -p /user/fastcampus/movie/input

$ hadoop fs -put dataset/MovieLens/movies.csv dataset/MovieLens/ratings.csv 
/user/fastcampus/movie/input

$ hadoop fs -ls /user/fastcampus/movie/input
"""
-rw-r--r--   1 6mini supergroup     484688 2022-11-23 12:10 /user/fastcampus/movie/input/movies.csv
-rw-r--r--   1 6mini supergroup    2382886 2022-11-23 12:10 /user/fastcampus/movie/input/ratings.csv
"""

$ hadoop jar target/mapreduce-example-1.0.0.jar com.fastcampus.hadoop.MovieAverageRateTopK /user/fastcampus/movie/input/movies.csv /user/fastcampus/movie/input/ratings.csv /user/fastcampus/output/first /user/fastcampus/output/second 
```

# 결과

```s
$ hadoop fs -ls /user/fastcampus/output/first
"""
-rw-r--r--   1 6mini supergroup          0 2022-11-23 12:13 /user/fastcampus/output/first/_SUCCESS
-rw-r--r--   1 6mini supergroup     312956 2022-11-23 12:13 /user/fastcampus/output/first/part-r-00000
"""

$ hadoop fs -ls /user/fastcampus/output/second
"""
-rw-r--r--   1 6mini supergroup          0 2022-11-23 12:14 /user/fastcampus/output/second/_SUCCESS
-rw-r--r--   1 6mini supergroup       1119 2022-11-23 12:14 /user/fastcampus/output/second/part-r-00000
"""

$ hadoop fs -head /user/fastcampus/output/first/part-r-00000
'''
Toy Story (1995)        3.9209302325581397
GoldenEye (1995)        3.496212121212121
City Hall (1996)        2.7857142857142856
Human Planet (2011)     4.0
Comme un chef (2012)    3.5
Movie 43 (2013) 3.5
"Pervert"s Guide to Ideology    3.5
Sightseers (2012)       4.5
Hansel & Gretel: Witch Hunters (2013)   2.9
Jim Jefferies: Fully Functional (EPIX) (2012)   4.5
Why Stop Now (2012)     1.5
Tabu (2012)     4.0
Extreme Measures (1996) 2.5
Upside Down (2012)      3.0
"Liability      3.0
Angst  (1983)   3.5
Stand Up Guys (2012)    2.5
Side Effects (2013)     3.9166666666666665
Identity Thief (2013)   2.875
"ABCs of Death  3.5
"Glimmer Man    3.0
Beautiful Creatures (2013)      2.0
"Good Day to Die Hard   2.0833333333333335
D3: The Mighty Ducks (1996)     2.1875
21 and Over (2013)      3.625
Safe Haven (2013)       4.0
Frozen Planet (2011)    4.5
"Act of Killing 5.0
Universal Soldier: Day of Reckoning (2012)      4.0
"Chamber        3.5
Escape from Planet Earth (2013) 4.0
"Apple Dumpling Gang    2.6
Before Midnight (2013)  3.5
Snitch (2013)   3.5
"Davy Crockett  3.0
Dark Skies (2013)       3.0
Oh Boy (A Coffee in Berlin) (2012)      3.5
'''

$ hadoop fs -head /user/fastcampus/output/second/part-r-00000 
'''
English Vinglish (2012) 5.0
"Trial  4.9
Bad Boy Bubby (1993)    4.833333333333333
"Imposter       4.75
Memories of Murder (Salinui chueok) (2003)      4.7
"Chorus Line    4.666666666666667
Tekkonkinkreet (Tekkon kinkurîto) (2006)        4.625
Yi Yi (2000)    4.6
Secrets & Lies (1996)   4.590909090909091
"Day of the Doctor      4.571428571428571
Guess Who"s Coming to Dinner (1967)     4.545454545454546
Paths of Glory (1957)   4.541666666666667
Redline (2009)  4.5
"Streetcar Named Desire 4.475
"Celebration    4.458333333333333
Ran (1985)      4.433333333333334
"Shawshank Redemption   4.429022082018927
Kolya (Kolja) (1996)    4.428571428571429
Central Station (Central do Brasil) (1998)      4.416666666666667
Incendies (2010)        4.4
His Girl Friday (1940)  4.392857142857143
Witness for the Prosecution (1957)      4.388888888888889
Paperman (2012) 4.375
Elite Squad: The Enemy Within (Tropa de Elite 2 - O Inimigo Agora É Outro) (2010)       4.357142857142857
All Quiet on the Western Front (1930)   4.35
Laura (1944)    4.333333333333333
Double Indemnity (1944) 4.323529411764706
'''
```

# AI-Based Recommendation System

A web app that suggests items (movies) to each user based on their ratings and preferences, using a
**collaborative filtering** algorithm (cosine similarity on centred ratings) written from scratch in Java.

## Features
- **Admin:** system settings (neighbours `k`, minimum similarity, minimum common items), recommendation
  effectiveness analysis (MAE, click-through rate, charts, improvement suggestions), user management.
- **User:** sign up / login, rate items 1-5, manage category preferences, personalized recommendation list with reasons.

## Requirements
- JDK 17 or newer (`javac -version`)
- A browser
- Optional (for permanent storage): MySQL 8 and the MySQL Connector/J jar (put it in a `lib/` folder)

## Run (quick start, in-memory demo data)
```bash
git clone https://github.com/lakshyakashyap571-rgb/java-review-1.git
cd java-review-1
javac -d out src/com/recsys/*.java
java -cp out com.recsys.App          # then open http://localhost:8080
```

## Run with MySQL (JDBC)
1. Create the tables: `mysql -u root -p < db/schema.sql`
2. Copy `db.properties.example` to `db.properties` and put your MySQL user and password in it.
3. Download MySQL Connector/J (https://dev.mysql.com/downloads/connector/j/) into `lib/`.
4. Run with the jar on the classpath (use `;` instead of `:` on Windows):
```bash
java -cp "out:lib/mysql-connector-j-9.0.0.jar" com.recsys.App
```
The console prints `Storage: MySQL (JDBC)`. On the first run an empty database is filled with demo data;
after that every rating, preference, user and setting change is saved, and restarts keep the data.
If the database cannot be reached the app prints the reason and continues in memory.

Demo logins: `admin@demo.com / admin123` (Admin), `aman@demo.com / user123` (User).
Console-only demo of the algorithm: `java -cp out com.recsys.Main`.

## Project structure
```
src/com/recsys/RecommendationEngine.java   core algorithm (similarity, recommend, MAE)
src/com/recsys/App.java                    HTTP server + REST API + demo data
src/com/recsys/DataStore.java              persistence interface
src/com/recsys/JdbcStore.java              MySQL implementation (JDBC)
src/com/recsys/MemoryStore.java            in-memory fallback
src/com/recsys/DataAccessException.java    unchecked SQL error wrapper
src/com/recsys/Recommendation.java         result record
src/com/recsys/Main.java                   console demo
web/index.html                             Admin and User dashboards
db/schema.sql                              MySQL schema
db.properties.example                      database connection template
```

## How the algorithm works
1. Centre ratings on 3 (liked = positive, disliked = negative).
2. Similarity of two users = cosine of their centred ratings on co-rated items (needs >= 2 common items).
3. Keep the `k` most similar users; predicted score = similarity-weighted average of their ratings.
4. Show unseen items predicted >= 3.5; preferred categories get a +0.5 boost. New users see popular items.
5. Accuracy is measured by Mean Absolute Error (hide a rating, predict it, compare).

## Limitations / future work
Passwords are hashed with SHA-256 (add a salt or use BCrypt for production). The REST layer uses the JDK HttpServer; a Servlet/Tomcat version is a possible extension.

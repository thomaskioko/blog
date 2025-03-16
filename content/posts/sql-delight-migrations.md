---
title: "SQLDelight Migrations"
date: "2025-03-16"
draft: false
hideToc: true
tags: ["SQLDelight",  "Performance", "Baseline-Profiles",]
series: "Tv Maniac Journey"
---

# Intro

SQLDelight is a powerful tool for managing database schemas in Kotlin Multiplatform projects. In this blog post, I'll walk through how I setup SQLDelight migrations for TvManiac.

### SQLDelight Migrations
SQLDelight migrations allow you to evolve your database schema over time while preserving existing data. Migration files in SQLDelight follow a simple naming convention: `<version>.sqm`. These files are stored in the same directory as your `.sq` files, typically under your sqldelight source directory.

``` text
src
└─ commonMain
   └─ sqldelight
      ├─ com/thomaskioko/tvmaniac/db
      |  ├─ Cast.sq
      |  └─ CastAppearance.sq
      └─ migrations
         └─ 1.sqm
```  

### TV Show Cast Migration
In my TV show tracking app, I initially modeled cast information with a simple table that included both cast details and their relationship to shows. 

``` sql
CREATE TABLE casts (
    id INTEGER NOT NULL PRIMARY KEY,
    name TEXT NOT NULL,
    character_name TEXT NOT NULL,
    profile_path TEXT,
    popularity REAL,
    tmdb_id INTEGER DEFAULT NULL,
    season_id INTEGER DEFAULT NULL,
    FOREIGN KEY(tmdb_id) REFERENCES tvshow(id) ON DELETE CASCADE,
    FOREIGN KEY(season_id) REFERENCES season(id) ON DELETE CASCADE
);
```

This issue with this design is that the cast table contained both cast member information and their relationship to shows and seasons. The relationship between cast members and shows was embedded in the cast table. It was difficult to represent cast members who appeared in multiple seasons. To fix this, I created a new table `cast_apprearance` to model the many-to-many relationship. And now, the casts table to only contain cast member information. Below is the  new structure


``` sql
CREATE TABLE casts (
    id INTEGER AS Id<CastId> NOT NULL PRIMARY KEY,
    name TEXT NOT NULL,
    character_name TEXT NOT NULL,
    profile_path TEXT,
    popularity REAL
);
```
*Cast Table*


```sql
CREATE TABLE cast_appearance (
    cast_id INTEGER AS Id<CastId> NOT NULL PRIMARY KEY,
    show_id INTEGER AS Id<TmdbId> NOT NULL,
    season_id INTEGER AS Id<SeasonId>, -- NULL if appears in whole show
    FOREIGN KEY(cast_id) REFERENCES casts(id) ON DELETE CASCADE,
    FOREIGN KEY(show_id) REFERENCES tvshow(id) ON DELETE CASCADE,
    FOREIGN KEY(season_id) REFERENCES season(id) ON DELETE CASCADE,
    UNIQUE(cast_id, show_id, season_id) -- Prevent duplicate entries
);
```
*Cast Appearance Table*

## The Migration Process

With this context, we can now create our migration. 

##### 1. Configure SQLDelight for Migrations
In the `build.gradle.kts` file, I have set up SQLDelight to handle migrations. I enabled set the migrationOutput directory and enabled migration verification. This will allow us to verify that the migration works correctly.:

``` kotlin
sqldelight {
  databases {
    create("TvManiacDatabase") {
      packageName.set("com.thomaskioko.tvmaniac.db")
      schemaOutputDirectory.set(file("src/commonMain/sqldelight/com/thomaskioko/tvmaniac/schemas"))
      migrationOutputDirectory.set(file("src/commonMain/sqldelight/com/thomaskioko/tvmaniac/migrations"))
      verifyMigrations.set(true)
    }
  }
}
```

##### 2. Create the migration script
SQLDelight makes migrations straightforward with its .sqm migration files. We will create a file named `1.sqm` and set up the migration.

##### 3. Create the new tables & Migrate existing data
To properly migrate existing data to the new tables, we will create temporary tables in our script so we can transfer the data and delete them once we are done. 

``` sql
-- Create a new casts table without the removed columns
CREATE TABLE casts_new (
    id INTEGER AS Id<CastId> NOT NULL PRIMARY KEY,
    name TEXT NOT NULL,
    character_name TEXT NOT NULL,
    profile_path TEXT,
    popularity REAL
);

-- Copy data from old table to new table
INSERT INTO casts_new (id, name, character_name, profile_path, popularity)
SELECT id, name, character_name, profile_path, popularity
FROM casts;

-- Drop the old table and rename the new one
DROP TABLE casts;
ALTER TABLE casts_new RENAME TO casts;

-- Create the cast_appearance table
CREATE TABLE IF NOT EXISTS cast_appearance (
    cast_id INTEGER AS Id<CastId> NOT NULL,
    show_id INTEGER AS Id<TmdbId> NOT NULL,
    season_id INTEGER AS Id<SeasonId>,
    PRIMARY KEY (cast_id, show_id, season_id),
    FOREIGN KEY(cast_id) REFERENCES casts(id) ON DELETE CASCADE,
    FOREIGN KEY(show_id) REFERENCES tvshow(id) ON DELETE CASCADE,
    FOREIGN KEY(season_id) REFERENCES season(id) ON DELETE CASCADE
);
```

##### 4. Verifying Migrations
SQLDelight provides a Gradle task called `verifySqlDelightMigration` that validates our migrations. This will fail if there are any issues with the migration file.

![AndroidTabBar](https://github.com/user-attachments/assets/c006b2d1-2bfa-4e2a-b6e3-ccb20aa1cb77) 


### Conclusion

SQLDelight's migration support makes it straightforward to evolve your database schema over time. By following the patterns demonstrated in our cast appearance example, you can confidently manage database changes in your Kotlin Multiplatform applications.

The key takeaways are:
- Use the create-copy-drop-rename pattern for table modifications
- Wrap migrations in transactions for atomicity. i.e `BEGIN/END TRANSACTION;`
- Migrations are not difficult.

Until we meet again, folks. Happy coding! ✌️

#### References

- [SQLDelight Migrations Documentation](https://sqldelight.github.io/sqldelight/2.0.2/multiplatform_sqlite/migrations/)
[SQLDelight Migration Verification](https://sqldelight.github.io/sqldelight/2.0.2/multiplatform_sqlite/migrations/#verifying-migrations)




---
title: "Similar string search with the Levenshtein distance on SQL Server"
slug: "similar-string-search-with-the-levenshtein-distance-on-sql-server"
date: "2021-02-01"
summary: "Recently I have been looking for more flexible ways to search through text within a SQL database, and I stumbled upon a suggestion which indicated to use the so called Levenshtein distance."
---

#software-development #sql #data-storage

Recently I have been looking for more flexible ways to search through text within a SQL database, and I stumbled upon a suggestion which indicated to use the so called Levenshtein distance. This parameter is a value which indicates the number of changes to a string required in order to match the searched value. In a certain way it is possible to regard this Levenshtein distance as being a similarity rating between two strings, whereas the lower the value, the more similar it is.


One thing I did not like about this answer was that it suggested building a table within the database itself to search through strings. Doing so would require an insane amount of overhead, for something which I'd only sparsely implement. Most of the time a literal match, or a `LIKE` is more than acceptable. I let the idea slide for a while.

That is, until the moment came where I needed such metric once again quite a specific scenario came up I could not solve using a `LIKE` clause. I needed to match a value to another, but there is a good change this value is not exactly the same. Casing may be different, the value may or may not include a dash and so on. Instead of guessing based on a predetermined set of rules I believe using the similarity metric is a much better alternative, and as such I want to look for the closest related match, if any single one exists.

## A T-SQL implementation of the [Levenshtein distance](https://en.wikipedia.org/wiki/Levenshtein_distance)
Sometimes I can simply appreciate the boldness of questions on StackOverflow, such as this one.

> ["I am interested in algorithm in T-SQL calculating Levenshtein distance."](https://stackoverflow.com/a/27734606/1720761)

An the answer is exactly that, and runs performant. The great thing is that it does not require butchering the data schema in order to implement such similarity search, but offers it in a convenient function, ready to be executed whenever you need.

## Finding the best match
Any value returned from this function does not immediately mean that this is the correct, or best match, and a certain amount of caution needs to be taken to ensure this function is used for the right scenario. As such, the things I'm doing;

1. Only query anything with a distance less than three
2. Grab the results with the lowest distance
3. Check if there is only a single value said distance away from the search
4. Grab said record from the db

The code to do so, as per my original implementation searching for a best matching aircraft based on the registration;

```sql
SELECT * 
FROM Aircraft 
WHERE Id = 
(
	SELECT X.Id
	FROM (
		SELECT TOP 1
			MIN(LD) AS LD,
			COUNT(*) AS Count,
			MIN(Id) AS Id
		FROM
		(
			SELECT 
        Id,
        dbo.Levenshtein(Registration, 'SearchTerm', 2) AS LD
			FROM Aircraft
		) AS X
		WHERE X.LD < 3
		GROUP BY X.LD
		ORDER BY X.LD
	) AS Y 
	WHERE Y.Count = 1
)
```

The neat thing with this query is that it does not return a result if it is unable to find an unique and definitive best match. In my case signalling more user interaction is required to finish the task at hand.


# Json transformations

Using Postgres as a JSON service seems to be faster than using a dedicated JSON query java library. The following compares execution times for a 350Kb JSON document transformed using Postgresql SQL (version 15.3), Josson and JSLT. The postgres DB runs on a docker on the same laptop, macbook pro 16 M1 Max.

Here's a screenshot of these 3 options:

![JSON transformation comparisson](json-transformation-pgsql-josson-jslt.png)
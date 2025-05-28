# Famous-Paintings-SQL-CaseStudy
## Overview
A comprehensive SQL case study exploring the world of famous paintings, artists, museums, and elated metadata. This project involves data ingestion, schema design, and analytical querying using PostgreSQL, providing insights into the art domain through structured data analysis.

## Objective
- Import and structure data from multiple CSV files into a PostgreSQL database.
- Understand and establish relationships between various entities.
-Execute a series of SQL queries to extract meaningful insights.
-Answer specific business questions related to the dataset.

## Dataset
The data for this project is sourced from the Kaggle dataset:

- **Dataset Link:** [Famous Paintings] (https://www.kaggle.com/datasets/mexwell/famous-paintings)
The dataset comprises the following CSV files:

| File Name          | Description                                 |
| ------------------ | ------------------------------------------- |
| `artist.csv`       | Information about artists                   |
| `canvas_size.csv`  | Details of canvas dimensions                |
| `image_link.csv`   | URLs linking to images of the paintings     |
| `museum.csv`       | Data about museums housing the artworks     |
| `museum_hours.csv` | Operating hours of the museums              |
| `product_size.csv` | Mapping between products and their sizes    |
| `subject.csv`      | Subjects or themes depicted in the artworks |
| `work.csv`         | Core information about the artworks         |

## SQL Queries

### 1. Fetch all the paintings which are not displayed on any museums
```sql
select *
from work w
where w.museum_id is null;
```
### 2. Fetch museums without any paintings
```sql
select * from museum m
where not exists (select 1 from work w
					        where w.museum_id=m.museum_id);
```

 ### 3. Fetch the paintings have asking price of more than their regular price
```sql
select p.work_id
from product_size p
where p.sale_price > p.regular_price;
```
### 4. Identify the paintings whose asking price is less than 50% of its regular price
```sql
select *
from product_size p
where p.regular_price > 2 * p.sale_price;
```
### 5. Fetch the canva size which cost the most
```sql
select p.size_id, c.label, p.sale_price
from product_size p
join canvas_size c
on p.size_id = cast(c.size_id as text)
order by p.sale_price desc
limit 1;
```
### 6. Delete dupliicate records from work, product_size, subject and image_link tables
```sql
delete from work
where ctid not in (select min(ctid) 
				           from work 
				           group by work_id);

delete from product_size
where ctid not in (select min(ctid)
				           from product_size
                   group by work_id);

delete from subject
where ctid not in (select min(ctid)
				           from subject
				           group by work_id);

delete from image_link 
where ctid not in (select min(ctid)
				           from image_link
				           group by work_id);
```
### 7. Identify the museums with invalid city information in the given dataset
```sql
select * from museum 
where city ~ '^[0-9]'; -- ~ is Regex 
```
### 8. Museum_Hours table has 1 invalid entry. Identify it and remove it
```sql
delete from museum_hours 
where ctid not in (select min(ctid)
				           from museum_hours
				           group by museum_id, day );
```
### 9. Fetch the top 10 most famous painting subject
```sql
select * 
from (
		select s.subject,count(1) as no_of_paintings,
		rank() over(order by count(1) desc) as ranking
		from work w
		join subject s on s.work_id=w.work_id
		group by s.subject ) x
where ranking <= 10;
-- OR
select s.subject, count(1)
from work w
join subject s 
on w.work_id=s.work_id
group by s.subject
order by count(1) desc
limit 10;
```
### 10. Identify the museums which are open on both Sunday and Monday. Display museum name, city
```sql
select m.name as museum_name, m.city 
from museum_hours mh1
join museum m
on mh1.museum_id = m.museum_id
where day='Sunday'
and exists (select 1 from museum_hours mh2
			      where mh1.museum_id=mh2.museum_id
			      and mh2.day='Monday');
```
### 11. Fetch the number of museums that are open every single day
```sql
select count(1)
from (select museum_id, count(1) as frequency
	    from museum_hours
	    group by museum_id) k
where k.frequency>=7;
```
### 12. List the top 5 most popular museum (Popularity is defined based on most no of paintings in a museum)
```sql
select m.name as museum, m.city, m.country, x.no_of_paintings
from (select m.museum_id, count(1) as no_of_paintings,
	    rank() over(order by count(1) desc) as rnk
	    from work w
	    join museum m on m.museum_id=w.museum_id
	    group by m.museum_id) x
join museum m
on m.museum_id=x.museum_id
where x.rnk<=5; 
```
### 13. List the top 5 most popular artist (Popularity is defined based on most no of paintings done by an artist)
```sql
select a.full_name as artist, a.nationality, x.number
from (select w.artist_id, count(1) as number,
	    rank() over(order by count(1) desc) as rnk
	    from work w
	    join artist a
	    on w.artist_id = a.artist_id
	    group by w.artist_id) x
join artist a
on x.artist_id=a.artist_id
where x.rnk<=5;
```
### 14. Display the 3 least popular canva sizes
```sql
select label,ranking,no_of_paintings
from (
		select cs.size_id,cs.label,count(1) as no_of_paintings,
		dense_rank() over(order by count(1) ) as ranking
		from work w
		join product_size ps on ps.work_id=w.work_id
		join canvas_size cs on cast(cs.size_id as text) = ps.size_id
		group by cs.size_id,cs.label) x
where x.ranking<=3;
```
### 15. Mmuseum which is open for the  longest duration during a day. Display museum name, state and hours open and which day
```sql
select m.name, m.state, TO_TIMESTAMP(close, 'HH:MI PM') - TO_TIMESTAMP(open, 'HH:MI AM') AS duration, mh.day
from museum_hours mh
join museum m
on mh.museum_id=m.museum_id
order by duration desc
limit 1;
```
### 16. Museum which has the most no of most popular painting style
```sql
with pop_style as 
			(select style,
			 rank() over(order by count(1) desc) as rnk
			 from work
			 group by style),
		cte as
			(select w.museum_id,m.name as museum_name,ps.style, count(1) as no_of_paintings,
			 rank() over(order by count(1) desc) as rnk
			 from work w
			 join museum m on m.museum_id=w.museum_id
			 join pop_style ps on ps.style = w.style
			 where w.museum_id is not null
			 and ps.rnk=1
			 group by w.museum_id, m.name,ps.style)
select museum_name,style,no_of_paintings
from cte 
where rnk=1;
```
### 17. Identify the artists whose paintings are displayed in multiple countries
```sql
with cte as
		(select distinct a.full_name as artist,
		 w.name as painting, m.name as museum,
		 m.country
		 from work w
		 join artist a on a.artist_id=w.artist_id
		 join museum m on m.museum_id=w.museum_id)
select artist,count(1) as no_of_countries
from cte
group by artist
having count(1)>1
order by 2 desc; -- 2 means second column that is no_of_countries
```
### 18. Display the country and the city with most of no of museums. Ouput 2 separate column to mention the city and the country. IF there are mutiple value, separate them with comma.
```sql
-- String_agg is used for separating them with commas in a single row
with cte_country as
		(select country, count(1),
		rank() over(order by count(1) desc) as rnk
		from museum m
		group by m.country),
	 cte_city as
	    (select city, count(1),
		rank() over(order by count(1) desc) as rnk
		from museum m
		group by m.city)
select string_agg( distinct country, ', ') as country, string_agg(city, ', ') as city
from cte_country
cross join cte_city
where cte_country.rnk=1
and cte_city.rnk=1;
```
### 19. Identify the artist and the museum where the most expensive and least expensive painting is placed. Display the artist name, sale_price, painting name, museum name, museum city and canvas label
```sql
with cte as 
		(select *,
		 rank() over(order by sale_price desc) as rnk,
		 rank() over(order by sale_price ) as rnk_asc
		 from product_size )
select w.name as painting,
cte.sale_price,
a.full_name as artist,
m.name as museum, m.city,
cz.label as canvas
from cte
join work w on w.work_id=cte.work_id
join museum m on m.museum_id=w.museum_id
join artist a on a.artist_id=w.artist_id
join canvas_size cz on cz.size_id = cte.size_id::NUMERIC
where rnk=1 or rnk_asc=1;
```
### 20. Country which has the 5th highest no of paintings
```sql
select x.country
from (select m.country, count(1),
	    rank() over(order by count(1) desc) as rnk
	    from work w
	    join museum m
	    on m.museum_id = w.museum_id
	    group by m.country) x
where x.rnk=5;
```
### 21. LIst 3 most popular and 3 least popular painting styles
```sql
select x.style, case when x.rnk<=3 then 'Most Popular' else 'Least Popular' end as remarks
from (select w.style, count(*) as no_of_paintings,
	    rank() over(order by count(*)) rnk,
	    count(1) over() as no_of_records
	    from work w
	    where style is not null
	    group by style) x
where x.rnk<=3 or no_of_records-3<rnk;
```
### 22. Artist which has the most no of Portraits paintings outside USA. Display artist name, no of paintings and the artist nationality.
```sql
select full_name as artist_name, nationality, no_of_paintings
from (
	select a.full_name, a.nationality,
	count(1) as no_of_paintings,
	rank() over(order by count(1) desc) as rnk
	from work w
	join artist a on a.artist_id=w.artist_id
	join subject s on s.work_id=w.work_id
	join museum m on m.museum_id=w.museum_id
	where s.subject='Portraits'
	and m.country != 'USA'
	group by a.full_name, a.nationality) x
where rnk=1;
```

## Conclusion
- Duplicate records and invalid entries (like incorrect city names and repeated museum hours) were successfully identified and removed using SQL.
- We found the most and least popular painting styles, canvas sizes, and artists, along with museums hosting the largest collections.
- Analysis revealed paintings priced above regular rates, discounted works, and the most/least expensive art pieces by artist and museum.
- Identified countries and cities with the most museums, artists with global reach, and painting styles common across regions.

This case study shows how SQL can be used to clean, explore, and draw valuable insights from real-world datasets in a structured and meaningful way.

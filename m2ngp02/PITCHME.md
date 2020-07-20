--- ?image=assets/img/bg-light.jpg&size=cover&opacity=80


### Common Backend Performance Issues
#### and ways to prevent them

---

@title[Performance issue price in Project Timeline]


![Performance issue price in Project Timeline](m2ngp02/Project-Timeline.png)

---

### Design phase

---

Every efficient solution starts from architecture

---

Wrong decision at the project start can lead to hundreds of hours of data fixes post launch

---

### Example with URL rewrites

--- 

### Customer wants URL slugs for layered navigation 

How would you implement it?

---

### Obvious Solution
 
Create url rewrite for every filter and category page combination

---

500 x 10 x 20 x 30 

![Progression of index size](m2ngp02/Index-Size.png)

~ 3,000,000

---

### Can we do better?

---

![SEO Friendly URL](m2ngp02/Url.png)

---

![Explained logical parts in URL](m2ngp02/Url-Parsed.png)

---

### Better Solution
@ul
- Add a plugin for URL rewrite processing
- Find category part from URL
- Treat rest of the URL as slugs
@ulend

---

### Example with store views

---

### Customer sells internationally 
@ul
- Each country has at least 2 languages (Local + English)
- There are 10 target currencies
@ulend

---

### Obvious Solution

Create a website for each country and store view within a website for each language

---

10k SKU x 80+ stores x 500 categories

![Progression of index size](m2ngp02/Index-Size-Product.png)

Category Index ~ 400,000,000    
Price Index ~ 8,000,000

---

![Product to Language and Price Relation](m2ngp02/Product-Price-Language.png)

---

### Better Solution
@ul
- Create store view for unique languages only
- Create country to currency and language map
- Write simplified price storage for merchant
- Detect at runtime price scope and language to apply based on domain/URL path.  
@ulend

--- 

### Development phase

---

### Rule #1
Minimize amount of I/O operations to the bare minimum

--- 

---

### Example from Magento 2.3

---

![Number of I/O operations in MSI](m2ngp02/MSI-Issue.png)

--- 

![MSI Issue Fix](m2ngp02/MSI-Issue-Fix.png)

---

### Rule #2

Make your I/O operations as lightweight as possible

---
#### SELECT rules
@ul
- Separate data retrieval from filtering 
- Avoid JOINs unless you need to filter or multiply result set
- Use iterators to fetch data from query
@ulend
---

### Almost final example

--- 

A friend once asked me to help with the query for finding open github issues per day for the last year 

---
This query was taking couple of hours to complete
```sql
SELECT 
  DATE(a.created_at) as day, 
  SUM(CASE WHEN a.created_at > b.created_at AND b.closed_at IS NULL 
      THEN 1 ELSE 0 END) AS open_issues
  FROM issues a
  LEFT JOIN issues b ON a.id > b.id
  WHERE a.created_at > ? 
  GROUP BY a.id, a.created_at
  ORDER BY a.id
```

---

```php
$openIssuesPerRepository = [];
$mutation = [];
$datePeriod = new DateTime('-1 year');
$incrementInterval = DateInterval::createFromDateString('1 day');
$startDate = $datePeriod->format('Y-m-d');
$statement = $connection->prepare(
    'SELECT COUNT(id) as open_issues, repo_id
        FROM issue
        WHERE
          created_at < ?
          AND closed_at > ? OR closed_at IS NULL
        GROUP BY repo_id'
);
$statement->execute([$startDate, $startDate]);
while ($row = $statement->fetch(PDO::FETCH_ASSOC)) {
    $openIssuesPerRepository[$row['repo_id']] = $row['open_issues'];
}
```
@[1-3](Initialize aggregates)
@[4-5](Condidtion and timespan)
@[6-13](Retrieve open issues on the start of period)
@[13-17](Set initial value from query into array)
--- 
```php
$mutationQueries = [
    'SELECT COUNT(id)*-1 as mutation, repo_id, 
            DATE(closed_at) as mutation_date
        FROM issue
        WHERE closed_at >= ?
        GROUP BY repo_id, DATE(closed_at)', 
    'SELECT COUNT(id) as mutation, repo_id, 
            DATE(created_at) as mutation_date
        FROM issue
        WHERE created_at >= ?
        GROUP BY repo_id, DATE(created_at)'
];
foreach ($mutationQueries as $query) {
    $statement = $connection->prepare($query);
    $statement->execute([$startDate]);
    while ($row = $statement->fetch(PDO::FETCH_ASSOC)) {
        $date = $row['mutation_date'];
        $repoId = $row['repo_id'];
        $previousValue = $mutation[$repoId][$date] ?? 0;
        $mutation[$repoId][$date] = $previousValue + $row['mutation'];
    }
}
```
@[2-6](Query for retrieving all closed issues per day)
@[7-11](Query for retrieving all open issues per day)
@[13-22](Execute queries and collect mutations)
---

```php
$now = new DateTime();
$days = [];
foreach ($openIssuesPerRepository as $repositoryId => $totalCount) {
    $dateIterable = clone $datePeriod;
    while ($now > $dateIterable) {
        $dayId = $dateIterable->format('Y-m-d');
        $totalCount += $mutation[$repositoryId][$dayId] ?? 0;
        $days[$repositoryId][$dayId] = $totalCount;
        $dateIterable->add($incrementInterval);
    }
}
```
@[1-2](Prepare results)
@[3,11](Start from collected initial values)
@[4-5,10](Iterate while date not reached today)
@[6](Create day group key)
@[7](Apply date mutation if available)
@[8](Add open issues count to result array)
@[9](Increase date by defined period)
---

The total time to generate report is now 100ms instead of an hour   

---
### Balance is important
Combining PHP code with multiple database queries usually much faster than single complex query

---

### Rule #3

Do not call third-party APIs/send email in within open database transaction

---

### Rule #4

Avoid updates to shared entities from concurrent connections

---

### The final example

---

During BlackFriday a customer of mine started to receive a lot of lock wait timeouts.

@css[fragment]( The issue was in site-wide free shipping sales rule )

---

### Sales rule lock of doom

@ul
- Sales rule counters get incremented with each order place  
- MySQL locks sales rule record till order place transaction gets completed
- As a result order throughput is 1 concurrent order
@ulend

---

### Solution

@ul
- Create sales rule mutation table
- Insert record on each rule counter increment/decrement
- Update counters by cronjob/queue consumer
@ulend

---

# Questions

Twitter: [@IvanChepurnyi](https://twitter.com/IvanChepurnyi)

Email: [ivan@ecom.dev](mailto:ivan@ecom.dev)

[IvanChepurnyi.github.io](https://ivanchepurnyi.github.io/)

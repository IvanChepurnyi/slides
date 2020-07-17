--- ?image=assets/img/bg-light.jpg&size=cover&opacity=80


### How to import 1 million SKUs
#### in under 10 minutes

--- 

#### Building an import correctly is hard

---

## 2008

it all started with

### DataFlow 

---?color=linear-gradient(100deg, white 50%, #567AD2 50%)

@snap[north-west span-50]
### DataFlow

@snapend

@snap[west span-40]
@ul
 - Stores CSV records into database
 - Imports product by product via AJAX call
@ulend
@snapend

@snap[east text-white span-40 text-10]
@ul
 - Uses product model to save data
 - Closing the browser window stops the whole process
@ulend
@snapend

---

### Speed

# 🐌

2-3 products per second 

@css[fragment](~ 20 minutes for 5k products)

--- 


## 2011

Import/Export saved us all


---?color=linear-gradient(100deg, white 50%, #567AD2 50%)

@snap[north-west span-50]
### ImportExport

@snapend

@snap[west span-40]
@ul
 - Stored batches of data into the database during validation
 - Processes stored data in one HTTP request
@ulend
@snapend

@snap[east text-white span-40 text-10]
@ul
 - Validates product data without using product model
 - Uses multi-row inserts to populate tables
 - Does not run indexers
@ulend
@snapend

---

### Speed

# 🐇

41 product per second

@css[fragment](~ 2 minutes for 5k products)

---

@snap[north span-80]
### But there are some drawbacks
@snapend

@ul
- High memory usage on large datasets
- Slow in generating primary keys for new products
@ulend

--- 

@snap[north]

# 2015
### Magento 2.x Import Export
@snapend

--- 
@snap[north span-60]
### ImportExport M2

@snapend

@ul
- Same base functionality as in M1
- More complex file format to edit and parse
- Slower on complex product data
- Adds additional single statement inserts
@ulend

--- 

# 2019

I got an idea and a project to implement it on

--- 

### Separate Feeds

@ul
- Main entity (sku, type, set)
- Attributes (sku, attribute, store, value)
- Category (sku, category slug, position)
- Configurable Options (sku, attribute, label)
- Images (sku, image)
- ...
@ulend

---

### Lazy Entity Resolving

@ul
- Reduce memory requirements of the import
- Cleaner and more readable feed processing
- Possibility of acquiring entity ids in batches automatically

@ulend

---

@snap[north-east span-100 text-06 text-gray]
Lazy Entity Resolving
@snapend


```php
$resolver = $this->resolverFactory->createSingleValueResolver(
    'catalog_product_entity', 'sku', 'entity_id'
);

$insert = InsertOnDuplicate::create(
    'catalog_product_entity_varchar', 
    ['entity_id', 'attribute_id', 'store_id', 'value']
)->withResolver($resolver);
    
$insert 
    ->withRow($resolver->unresolved('sku1'), 1, 0, 'some value')
    ->withRow($resolver->unresolved('sku2'), 1, 0, 'some value1')
    ->withRow($resolver->unresolved('sku3'), 1, 0, 'some value2');
```

@[1-3](Configure table lookup information)
@[5-8](Pass resolver into insert builder)
@[11-14](Use resolver to create identifier containers)
@[1-14](Insert on duplicate will skip any unresolved entries)
--- 

---

@snap[north-east span-100 text-06 text-gray]
Batch auto-increment generation
@snapend


```sql
START TRANSACTION;

INSERT INTO catalog_product_entity (sku)
    VALUES 
           ('sku1'), 
           ('sku2'), 
           ('sku3'), 
           ('sku4');

SELECT entity_id, sku
    FROM catalog_product_entity
    WHERE 
        sku IN ('sku1', 'sku2', 'sku3', 'sku4');

ROLLBACK;
```

@[1](Start a transaction)
@[2-9](Populate table with un-resolved keys)
@[10-14](Retrieve new identifiers)
@[15](Rollback transaction)
--- 

@snap[north span-100]
### Prepared Statements 🔥🔥🔥
@snapend
    
    

@ul
- Compile query for constant batch size
- Send only data instead of generating new queries
- Reduces query processing on MySQL side by half
@ulend

--- 

### Speed

# 🚀

450 products per second

@css[fragment](~ 11 seconds for 5k products)

--- 

### But it was still not good enough

@css[fragment](45 minutes to import 1 million SKUs 😥)

---

Sure, because it's a sequential process...

---

So I made it asynchronous as PoC 🧨

--- 

### Under the hood

@ul 
- Each **target table** receives a separate connection to MySQL
- Identity resolver is attached to the source table connection
- Each feed is processed concurrently by using **round robin** strategy
- During MySQL query execution PHP prepares the next batch 
@ulend


---

### Speed

# ⚡

1,850 products per second

@css[fragment](~ 9 minutes for 1m products)

---

It is coming this fall as an open source tool for everyone! 


---

# Questions

ivan@ecomdev.org

<a href="https://ivanchepurnyi.github.io/">IvanChepurnyi.GitHub.io<a>

---
theme: dracula
canvasWidth: 700
layout: cover
text-align: center
title: Better indexation without performance penalty
---

# Better Indexation without Performance Penalty


---
layout: two-cols
---

# Charities helping

- Ukrainian Foundation "Come Back Alive"

  https://savelife.in.ua/en/


- Dutch Foundatation "Sails of Freedom" 

  https://zeilenvanvrijheid.nl/
  
::right::

# Ukraine 🇺🇦

<img src="/qrcode-come-back-alive.png" class="w-20 ml-10 mt-8  shadow"  /> 

<img src="/qrcode-zeilen.png"  class="w-20 ml-10 mt-8 shadow"  />


---

# About me
    


<v-clicks>

🫶 Passionate about open source projects

🚀 Expert in optimizing web appplications

👴🏻 One of grand-parents of Magento

🧑‍🚒 Experienced in putting out fires on production systems

</v-clicks>


---

# Why is indexation important?


<v-clicks>

🐌 Normalized database structure is slower to query

💶 Expensive calculations on listing pages

🔍 Text based search tokenization

</v-clicks>

---

# What is offered by default? 


<v-clicks>

💾 Update on Save 

* Does not take into account changes from import/export
* Prone to wait timeouts and deadlocks

🗓️ Update by Schedule
    
* Named mview, but not a real materialized view
* Observes data modifications with triggers
* Litters up your database with **{mview_name}_cl** tables

</v-clicks>


---

# Writign Good Triggers is Hard

<v-clicks>

💥 2.2.x - Blows up large databases because core team did not understand how *<=>* operator works 

* [#23077](https://github.com/magento/magento2/issues/23077) Triggers created by MView are triggered all the time

🤦‍♂️ 2.4.x - Introduced better batching but made things worse 

* [#30012](https://github.com/magento/magento2/issues/30012) Asynchronous indexing can break large servers
* [#37367](https://github.com/magento/magento2/issues/37367) Schedule index - entities processed multiple times

</v-clicks>

---

# Not that Obvious Issues

<v-clicks>

🔒 Every database write produces INSERT into the same set of tables within write transaction

🐌 Catalog Staging introduces even more complex lookup logic on each write

🔄 Dangerous when not configured properly in multi-origin replication

</v-clicks>

---
layout: cover
---

# Can We do Better?

---

# Binary Log to the Resque

<v-clicks>

🧰 Enabled by default on AWS RDS

💾 Row based format is default since MySQL 8.x

🧑‍💻 Experienced DevOps already use it for replication

📜 Gives complete access to data change history

</v-clicks>

---

# Binary Log Configuration


📄 /etc/mysql/conf.d/mysql-x-enable-binlog.cnf

```ini {2-3|4-5|6-7|8-10|all}
[mysqld] 
# Enables binary loggin with default file paths
log_bin 
# Make sure you row based replication is used
binlog-format = row
# Reduce size of binary log to not include text and binary fields if they are not changed
binlog-row-image=noblob 
# Nice to have automatic clean up of binlog files
expire_logs_days = 5 
max_binlog_size = 100M
```

---

# User Permissions 

Make sure you have the following permission for your MySQL user
```sql
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO '[USER]'@'%';
```

Not needed if you have all **GRANT ALL**, but it is good practice to create separate user with only this permissions.


---

# Mage-OS Database Changelog 

<v-clicks>

🦀 Written in Rust as static binary

🛡️ No root permissions needed

⚡ Blazingly fast

</v-clicks>


---

# How it Works

<v-clicks>

🔌 Connects to MySQL as **Replication Client**

🕵️ Analyzes **Binary Log** since last run

📩 Transforms **Binary Events** into **Row Changes**

📋 Maps **Row Changes** into **Domain Updates**

🧮 Aggregate reduces **Domain Update** into **Log Output**

⚙️ Magento processes **Log Output** and updates **MView**
</v-clicks>


---

# Easy to Customize

<v-clicks>

📚 Available also as Cargo library

⚒️ Easy to create own Mappers

⚙️ Sample Application Skeleton (coming)

</v-clicks>

---

# Example of Custom Mapper

```rust {all|1|3|4-5|7-8|9-10|11|6,12}
pub struct MyAwesomeTableMapper;

impl ChangeLogMapper<ProductChange> for MyAwesomeTableMapper {
    fn map_event(&self, event: &Event, schema: &impl TableSchema) 
        -> Result<Option<ProductChange>, Error> {
        Ok(match event {
            Event::UpdateRow(row) => FieldUpdate
                ::new(row.parse("product_id", schema)?)
                    .process_field_update("not_awesome_field", row, schema)
                    .process_field_update("super_awesome_field", row, schema)
                    .into_change_log()
            _ => None,
        })
    }
}
```

---

# Adding Table Name Mapper

```rust {all|4-5|6-8|6,9}
pub struct MyAwesomeTableNameMatchMapper;

impl ChangeLogMapper<ItemChange> for MyAwesomeTableNameMatchMapper {
    fn map_event(&self, event: &Event, schema: &impl TableSchema) 
        -> Result<Option<ItemChange>, Error> {
         Ok(match schema.table_name() {
            "my_awesome_table_for_product" => MyAwesomeTableMapper
                .map_event(event, schema)?.map(ItemChange::ProductChange),
            _ => None
        })
    }
}
```


---

# Add to App Skeleton

```rust 
Application::parse()?
    .with_mapper(MyAwesomeTableNameMatchMapper)
    .run().await?;
```

---
layout: cover
---

# 📊 Eager for Nmbrs?  

---


# Updating Product Data with Triggers

```bash {all|1|4|5|6}
    checks.........................: 100.00% ✓ 1512     ✗ 0   
    data_received..................: 11 MB   19 kB/s
    data_sent......................: 1.5 MB  2.4 kB/s
    dropped_iterations.............: 28      0.046447/s
    http_req_waiting...............: avg=352.83ms min=101.32ms med=383.35ms
    http_reqs......................: 1512    2.508118/s

running (10m02.8s), 0/1 VUs, 72 complete and 0 interrupted iterations
default ✗ [=====>-----------] 1 VUs  10m02.8s/10m0s  072/100 shared iters
```

```
Product Category MView Time: 30.49039s
```

--- 

# Updating Product Data with Changelog

```bash {all|1|4|5|6}
    checks.........................: 100.00% ✓ 1680     ✗ 0   
    data_received..................: 12 MB   21 kB/s
    data_sent......................: 1.6 MB  2.7 kB/s
    dropped_iterations.............: 20      0.033187/s
    http_req_waiting...............: avg=316.38ms min=107.47ms med=335.25ms
    http_reqs......................: 1680    2.787712/s

running (10m02.6s), 0/1 VUs, 80 complete and 0 interrupted iterations
default ✗ [=======>--------] 1 VUs  10m02.6s/10m0s  080/100 shared iters

```

```
Product Category MView Time: 7.19157s
```

--- 

# Output Example from Load Test

```json {all|2|3-6|9-13}
{
    "attributes":{
        "106":[ 31, 14, 27, 58 ],
        "124":[ 16, 76, 10, 2 ],
        "97":[ 646, 403, 773, 41553, 306, 756 ],
        "99":[ 712, 45, 334, 690, 426 ]
    },
    "entity":"product",
    "metadata":{
        "file":"47de4c575982-bin.000001",
        "position":3086928,
        "timestamp":1683532855
    }
}
```

---

# But there is more

```rust {all|2-3|4-5|6-7|8-9|10|11-13}
enum AggregateKey {
    Created,
    Deleted,
    Field(&'static str),
    Attribute(usize),
    WebsiteAll,
    WebsiteSpecific(usize),
    CategoryAll,
    CategorySpecific(usize),
    Link(usize),
    Composite,
    MediaGallery,
    TierPrice,
}

```

---

# Roadmap 

<v-clicks>

📁 Support for Category Entity

🪄 Support for Staging in Adobe Commerce

🛠️ Support for Custom Tables via Configuration File

🧡 Support for Magento 1.x

🔌 Plug-and-Play Composer based Module with Binary

</v-clicks>

---

# Get Invovled 

<v-clicks>

🧡 Join Mage-OS Community [mage-os.org](https://mage-os.org/)

🧑‍💻 Follow on GitHub [EcomDev/mage-os-database-changelog](https://github.com/EcomDev/mage-os-database-changelog)

</v-clicks>


---

# Bright Future Ahead

<v-clicks>

🔒 Security monitoring tool based on database changelog

⚡ Blazingly fast data streams for even better indexation and data export

</v-clicks>

---

# Thank You

🙋 Want to learn more about performance? 

🌴 Join performance training this July in Barcelona, Spain

<img src="/qrcode-training.png"  class="w-20 center mt-8 shadow"  />

[bit.ly/mage-performance-training](https://bit.ly/mage-performance-training)
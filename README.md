# GORM

Yet Another ORM library for Go, aims for developer friendly

## Overview

* Chainable API
* Relations
* Callbacks (before/after create/save/update/delete)
* Soft Delete
* Auto Migration
* Transaction
* Logger Support
* Bind struct with tag
* Every feature comes with tests
* Convention Over Configuration
* Developer Friendly

## Conventions

```go
type User struct {         // TableName: `users`, gorm will pluralize struct's name as table name
	Id			 int64	   // Id: Database Primary key
	Birthday	 time.Time
	Age			 int64
	Name		 string  `sql:"size:255"` // set this field's length and as not null with tag
	CreatedAt	 time.Time // Time of record is created, will be insert automatically
	UpdatedAt	 time.Time // Time of record is updated, will be updated automatically
	DeletedAt	 time.Time // Time of record is deleted, refer `Soft Delete` for more

	Emails            []Email         // Embedded structs
	BillingAddress    Address         // Embedded struct
	BillingAddressId  sql.NullInt64   // Embedded struct BillingAddress's foreign key
	ShippingAddress   Address         // Embedded struct
	ShippingAddressId int64           // Embedded struct ShippingAddress's foreign key
	IgnoreMe          int64 `sql:"-"` // Ignore this field with tag
}

type Email struct {    // TableName: `emails`
	Id         int64
	UserId     int64   // Foreign key for above embedded structs
	Email      string  `sql:"type:varchar(100);"` // Set column type directly with tag
	Subscribed bool
}

type Address struct {  // TableName: `addresses`
	Id       int64
	Address1 string         `sql:"not null;unique"` // Set column as unique with tag
	Address2 string         `sql:"type:varchar(100);unique"`
	Post     sql.NullString `sql:not null`
    // Be careful: "NOT NULL" will only works for NullXXX scanner, because golang will initalize a default value for most type...
}
```

## Opening a Database

```go
import "github.com/jinzhu/gorm"
import _ "github.com/lib/pq"
// import _ "github.com/go-sql-driver/mysql"
// import _ "github.com/mattn/go-sqlite3"

db, err := Open("postgres", "user=gorm dbname=gorm sslmode=disable")
// db, err = Open("mysql", "gorm:gorm@/gorm?charset=utf8&parseTime=True")
// db, err = Open("sqlite3", "/tmp/gorm.db")


// Set the maximum idle database connections
db.SetPool(100)


// By default, table name is plural of struct type, if you like singular table name
db.SingularTable(true)


// Gorm is goroutines friendly, so you can create a global variable to keep the connection and use it everywhere like this

var DB gorm.DB

func init() {
    DB, err = gorm.Open("postgres", "user=gorm dbname=gorm sslmode=disable")
    if err != nil {
        panic(fmt.Sprintf("Got error when connect database, the error is '%v'", err))
    }
}
```

## Struct & Database Mapping

```go
// Create table from struct
db.CreateTable(User{})

// Drop table
db.DropTable(User{})
```

### Automating Migrations

Feel Free to update your struct, AutoMigrate will keep your database update to date.

FYI, AutoMigrate will only add new columns, won't change column's type or delete unused columns, to make sure gorm won't harm your data.

If table doesn't exist when AutoMigrate, it will run create table automatically.

(only postgres and mysql supported)

```go
db.AutoMigrate(User{})
```

## Create

```go
user := User{Name: "jinzhu", Age: 18, Birthday: time.Now()}
db.Save(&user)
```

### Create With SubStruct

Refer [Query With Related](#query-with-related) to find how to find associations

```go
user := User{
		Name:            "jinzhu",
		BillingAddress:  Address{Address1: "Billing Address - Address 1"},
		ShippingAddress: Address{Address1: "Shipping Address - Address 1"},
		Emails:          []Email{{Email: "jinzhu@example.com"}, {Email: "jinzhu-2@example@example.com"}},
}

db.Save(&user)
//// INSERT INTO "addresses" (address1) VALUES ("Billing Address - Address 1");
//// INSERT INTO "addresses" (address1) VALUES ("Shipping Address - Address 1");
//// INSERT INTO "users" (name,billing_address_id,shipping_address_id) VALUES ("jinzhu", 1, 2);
//// INSERT INTO "emails" (user_id,email) VALUES (111, "jinzhu@example.com");
//// INSERT INTO "emails" (user_id,email) VALUES (111, "jinzhu-2@example.com");
```

## Query

```go
// Get the first record
db.First(&user)
//// SELECT * FROM users ORDER BY id LIMIT 1;
// Search table `users` are guessed from the out struct type.
// You are possible to specify the table name with Model() if no out struct for some methods like Pluck()
// Or set table name with Table(), if so, it will ignore the out struct even have it. more details following.

// Get the last record
db.Last(&user)
//// SELECT * FROM users ORDER BY id DESC LIMIT 1;

// Get a record without order by primary key
db.Find(&user)
//// SELECT * FROM users LIMIT 1;

// Get first record as map
db.First(&users)
//// SELECT * FROM users LIMIT 1;

// Get All records
db.Find(&users)
//// SELECT * FROM users;

// Using a Primary Key
db.First(&user, 10)
//// SELECT * FROM users WHERE id = 10;
```

### Query With Where (SQL like condition)

```go
// Get the first matched record
db.Where("name = ?", "jinzhu").First(&user)
//// SELECT * FROM users WHERE name = 'jinzhu' limit 1;

// Get all matched records
db.Where("name = ?", "jinzhu").Find(&users)
//// SELECT * FROM users WHERE name = 'jinzhu';

db.Where("name <> ?", "jinzhu").Find(&users)
//// SELECT * FROM users WHERE name <> 'jinzhu';

// IN
db.Where("name in (?)", []string{"jinzhu", "jinzhu 2"}).Find(&users)
//// SELECT * FROM users WHERE name IN ('jinzhu', 'jinzhu 2');

// LIKE
db.Where("name LIKE ?", "%jin%").Find(&users)
//// SELECT * FROM users WHERE name LIKE "%jin%";

// Multiple Conditions
db.Where("name = ? and age >= ?", "jinzhu", "22").Find(&users)
//// SELECT * FROM users WHERE name = 'jinzhu' AND age >= 22;
```

### Query With Where (Struct & Map)

```go
// Search with struct
db.Where(&User{Name: "jinzhu", Age: 20}).First(&user)
//// SELECT * FROM users WHERE name = "jinzhu" AND age = 20 LIMIT 1;

// Search with map
db.Where(map[string]interface{}{"name": "jinzhu", "age": 20}).Find(&users)
//// SELECT * FROM users WHERE name = "jinzhu" AND age = 20;

// IN For Primary Key
db.Where([]int64{20, 21, 22}).Find(&users)
//// SELECT * FROM users WHERE id IN (20, 21, 22);
```

### Query With Not

```go
// Attribute Not Equal
db.Not("name", "jinzhu").First(&user)
//// SELECT * FROM users WHERE name <> "jinzhu" LIMIT 1;

// Not In
db.Not("name", []string{"jinzhu", "jinzhu 2"}).Find(&users)
//// SELECT * FROM users WHERE name NOT IN ("jinzhu", "jinzhu 2");

// Not In for Primary Key
db.Not([]int64{1,2,3}).First(&user)
//// SELECT * FROM users WHERE id NOT IN (1,2,3);

db.Not([]int64{}).First(&user)
//// SELECT * FROM users;

// Normal SQL
db.Not("name = ?", "jinzhu").First(&user)
//// SELECT * FROM users WHERE NOT(name = "jinzhu");

// Not With Struct
db.Not(User{Name: "jinzhu"}).First(&user)
//// SELECT * FROM users WHERE name <> "jinzhu";
```

### Query With Inline Condition

```go
// Find with primary key
db.First(&user, 23)
//// SELECT * FROM users WHERE id = 23 LIMIT 1;

// Normal SQL
db.Find(&user, "name = ?", "jinzhu")
//// SELECT * FROM users WHERE name = "jinzhu";

// Multiple Conditions
db.Find(&users, "name <> ? and age > ?", "jinzhu", 20)
//// SELECT * FROM users WHERE name <> "jinzhu" AND age > 20;

// Inline Search With Struct
db.Find(&users, User{Age: 20})
//// SELECT * FROM users WHERE age = 20;

// Inline Search With Map
db.Find(&users, map[string]interface{}{"age": 20})
//// SELECT * FROM users WHERE age = 20;
```

### Query With Or

```go
db.Where("role = ?", "admin").Or("role = ?", "super_admin").Find(&users)
//// SELECT * FROM users WHERE role = 'admin' OR role = 'super_admin';

// Or With Struct
db.Where("name = 'jinzhu'").Or(User{Name: "jinzhu 2"}).Find(&users)
//// SELECT * FROM users WHERE name = 'jinzhu' OR name = 'jinzhu 2';

// Or With Map
db.Where("name = 'jinzhu'").Or(map[string]interface{}{"name": "jinzhu 2"}).Find(&users)
```

### Query With Related

```go
// Find emails from user with guessed foreign key
db.Model(&user).Related(&emails)
//// SELECT * FROM emails WHERE user_id = 111;

// Find address from user with specified foreign key 'BillingAddressId'
db.Model(&user).Related(&address1, "BillingAddressId")
//// SELECT * FROM addresses WHERE id = 123; // 123 is the value of user's BillingAddressId

// Find user from email with guessed primary key value from emails
db.Model(&email).Related(&user)
//// SELECT * FROM users WHERE id = 111; // 111 is the value of email's UserId
```

### Query Chains

Gorm has a chainable API, so you could query like this

```go
db.Where("name <> ?","jinzhu").Where("age >= ? and role <> ?",20,"admin").Find(&users)
//// SELECT * FROM users WHERE name <> 'jinzhu' AND age >= 20 AND role <> 'admin';

db.Where("role = ?", "admin").Or("role = ?", "super_admin").Not("name = ?", "jinzhu").Find(&users)
```

## Update

### Update an existing struct

```go
user.Name = "jinzhu 2"
user.Age = 100
db.Save(&user)
//// UPDATE users SET name='jinzhu 2', age=100 WHERE id=111;
```

### Update one attribute with `Update`

```go
// Update an existing struct's name if name is different
db.Model(&user).Update("name", "hello")
//// UPDATE users SET name='hello' WHERE id=111;

// Find out a struct, and update it if name is different
db.First(&user, 111).Update("name", "hello")
//// SELECT * FROM users LIMIT 1;
//// UPDATE users SET name='hello' WHERE id=111;

// Specify table name with where search
db.Table("users").Where(10).Update("name", "hello")
//// UPDATE users SET name='hello' WHERE id = 10;
```

### Update multiple attributes with `Updates`

```go
// Update an existing record if have any changed values
db.Model(&user).Updates(User{Name: "hello", Age: 18})
//// UPDATE users SET name='hello', age=18 WHERE id = 111;

// Updates with Map
db.Table("users").Where(10).Updates(map[string]interface{}{"name": "hello", "age": 18})
//// UPDATE users SET name='hello', age=18 WHERE id = 10;

// Updates with Struct
db.Model(User{}).Updates(User{Name: "hello", Age: 18})
//// UPDATE users SET name='hello', age=18;
```

## Delete

### Delete an existing struct

```go
db.Delete(&email)
// DELETE from emails where id=10;
```

### Batch Delete with search

```go
db.Where("email LIKE ?", "%jinzhu%").Delete(Email{})
// DELETE from emails where email LIKE "%jinhu%";
```

### Soft Delete

If a struct has DeletedAt field, it will get soft delete ability automatically!

For those don't have the filed, will be deleted from database permanently

```go
db.Delete(&user)
//// UPDATE users SET deleted_at="2013-10-29 10:23" WHERE id = 111;

// Batch delete when search
db.Where("age = ?", 20).Delete(&User{})
//// UPDATE users SET deleted_at="2013-10-29 10:23" WHERE age = 20;

// For structs have DeletedAt field, when do query, will ignore deleted records by default
db.Where("age = 20").Find(&user)
//// SELECT * FROM users WHERE age = 100 AND (deleted_at IS NULL AND deleted_at <= '0001-01-02');

// Find out all records including those deleted with Unscoped
db.Unscoped().Where("age = 20").Find(&users)
//// SELECT * FROM users WHERE age = 20;

// Permanently delete a record with Unscoped
db.Unscoped().Delete(&order)
// DELETE FROM orders WHERE id=10;
```

## FirstOrInit

Try to load the first record, if fails, initialize struct with search conditions.

(only support map or struct conditions, SQL like conditions are not supported)

```go
db.FirstOrInit(&user, User{Name: "non_existing"})
//// User{Name: "non_existing"}

db.Where(User{Name: "Jinzhu"}).FirstOrInit(&user)
//// User{Id: 111, Name: "Jinzhu", Age: 20}

db.FirstOrInit(&user, map[string]interface{}{"name": "jinzhu"})
//// User{Id: 111, Name: "Jinzhu", Age: 20}
```

### FirstOrInit With Attrs

Attr's arguments would be used to initialize struct if no record found, but won't be used for search

```go
db.Where(User{Name: "non_existing"}).Attrs(User{Age: 20}).FirstOrInit(&user)
//// SELECT * FROM USERS WHERE name = 'non_existing';
//// User{Name: "non_existing", Age: 20}

// Above code could be simplified if has only one attribute
db.Where(User{Name: "noexisting_user"}).Attrs("age", 20).FirstOrInit(&user)

// If a record found, Attrs would be just ignored
db.Where(User{Name: "Jinzhu"}).Attrs(User{Age: 30}).FirstOrInit(&user)
//// SELECT * FROM USERS WHERE name = jinzhu';
//// User{Id: 111, Name: "Jinzhu", Age: 20}
```

### FirstOrInit With Assign

Assign's arguments would be used to set the struct even a record found, but won't be used for search

```go
db.Where(User{Name: "non_existing"}).Assign(User{Age: 20}).FirstOrInit(&user)
//// User{Name: "non_existing", Age: 20}

db.Where(User{Name: "Jinzhu"}).Assign(User{Age: 30}).FirstOrInit(&user)
//// User{Id: 111, Name: "Jinzhu", Age: 30}
```

## FirstOrCreate

Try to load the first record, if fails, initialize struct with search conditions and save it

```go
db.FirstOrCreate(&user, User{Name: "non_existing"})
//// User{Id: 112, Name: "non_existing"}

db.Where(User{Name: "Jinzhu"}).FirstOrCreate(&user)
//// User{Id: 111, Name: "Jinzhu"}

db.FirstOrCreate(&user, map[string]interface{}{"name": "jinzhu", "age": 30})
//// user -> User{Id: 111, Name: "jinzhu", Age: 20}
```

### FirstOrCreate With Attrs

Attr's arguments would be used to initialize struct if no record found, but won't be used for search

```go
db.Where(User{Name: "non_existing"}).Attrs(User{Age: 20}).FirstOrCreate(&user)
//// SELECT * FROM users WHERE name = 'non_existing';
//// User{Id: 112, Name: "non_existing", Age: 20}

db.Where(User{Name: "jinzhu"}).Attrs(User{Age: 30}).FirstOrCreate(&user)
//// User{Id: 111, Name: "jinzhu", Age: 20}
```

### FirstOrCreate With Assign

Assign's arguments would be used to initialize the struct if not record found,

If any record found, will assign those values to the record, and save it back to database.

```go
db.Where(User{Name: "non_existing"}).Assign(User{Age: 20}).FirstOrCreate(&user)
//// user -> User{Id: 112, Name: "non_existing", Age: 20}

db.Where(User{Name: "jinzhu"}).Assign(User{Age: 30}).FirstOrCreate(&user)
//// SELECT * FROM users WHERE name = 'jinzhu';
//// UPDATE users SET age=30 WHERE id = 111;
//// User{Id: 111, Name: "jinzhu", Age: 30}
```

## Select

```go
db.Select("name, age").Find(&users)
//// SELECT name, age FROM users;
```

## Order

```go
db.Order("age desc, name").Find(&users)
//// SELECT * FROM users ORDER BY age desc, name;

db.Order("age desc").Order("name").Find(&users)
//// SELECT * FROM users ORDER BY age desc, name;

// ReOrder
db.Order("age desc").Find(&users1).Order("age", true).Find(&users2)
//// SELECT * FROM users ORDER BY age desc; (users1)
//// SELECT * FROM users ORDER BY age; (users2)
```

## Limit

```go
db.Limit(3).Find(&users)
//// SELECT * FROM users LIMIT 3;

// Cleanup limit with -1
db.Limit(10).Find(&users1).Limit(-1).Find(&users2)
//// SELECT * FROM users LIMIT 10; (users1)
//// SELECT * FROM users; (users2)
```

## Offset

```go
db.Offset(3).Find(&users)
//// SELECT * FROM users OFFSET 3;

// Cleanup offset with -1
db.Offset(10).Find(&users1).Offset(-1).Find(&users2)
//// SELECT * FROM users OFFSET 10; (users1)
//// SELECT * FROM users; (users2)
```

## Count

```go
db.Where("name = ?", "jinzhu").Or("name = ?", "jinzhu 2").Find(&users).Count(&count)
//// SELECT * from USERS WHERE name = 'jinzhu' OR name = 'jinzhu 2'; (users)
//// SELECT count(*) FROM users WHERE name = 'jinzhu' OR name = 'jinzhu 2'; (count)

// Set table name with Model
db.Model(User{}).Where("name = ?", "jinzhu").Count(&count)
//// SELECT count(*) FROM users WHERE name = 'jinzhu' OR name = 'jinzhu 2'; (count)

// Set table name with Table
db.Table("deleted_users").Count(&count)
//// SELECT count(*) FROM deleted_users;
```

## Pluck

Get struct's attribute as map

```go
var ages []int64
db.Find(&users).Pluck("age", &ages)

// Set Table With Model
var names []string
db.Model(&User{}).Pluck("name", &names)
//// SELECT name FROM users;

// Set Table With Table
db.Table("deleted_users").Pluck("name", &names)
//// SELECT name FROM deleted_users;

// Pluck more than one column? Do it like this
db.Select("name, age").Find(&users)
```

## Callbacks

Callback is a function defined to a struct, the function would be run when reflect a struct to database.
If a function return error, gorm will prevent future operations and do rollback

Those callbacks are defined now:

`BeforeCreate`, `AfterCreate`
`BeforeUpdate`, `AfterUpdate`
`BeforeSave`, `AfterSave`
`BeforeDelete`, `AfterDelete`

```go
// Won't update readonly user
func (u *User) BeforeUpdate() (err error) {
    if u.readonly() {
        err = errors.New("Read Only User")
    }
    return
}

// If have more than 1000 users, will rollback the insertion
func (u *User) AfterCreate() (err error) {
    if (u.Id > 1000) { // just an example, don't use Id to count users
        err = errors.New("Only 1000 users allowed")
    }
    return
}
```

## Specify Table Name

```go
// When Create Table from struct
db.Table("deleted_users").CreateTable(&User{})

// When Pluck
db.Table("users").Pluck("age", &ages)
//// SELECT age FROM users;

// When Query
var deleted_users []User
db.Table("deleted_users").Find(&deleted_users)
//// SELECT * FROM deleted_users;

// When Delete
db.Table("deleted_users").Where("name = ?", "jinzhu").Delete()
//// DELETE FROM deleted_users WHERE name = 'jinzhu';
```

### Specify Table Name for Struct

You are possible to specify table name for a struct by defining method TableName

```go
type Cart struct {
}

func (c Cart) TableName() string {
	return "shopping_cart"
}

func (u User) TableName() string {
    if u.Role == "admin" {
        return "admin_users"
    } else {
        return "users"
    }
}
```

## Transaction

```go
tx := db.Begin()

user := User{Name: "transcation"}

tx.Save(&u)
tx.Update("age": 90)
// do whatever

// rollback
tx.Rollback()

// commit
tx.Commit()
```

## Logger

Grom has builtin logger support, enable it with:

```go
db.LogMode(true)
```

![logger](https://raw.github.com/jinzhu/gorm/master/images/logger.png)

```go
// Use your own logger
// Checkout gorm's default logger for how to format messages: https://github.com/jinzhu/gorm/blob/master/logger.go#files
db.SetLogger(log.New(os.Stdout, "\r\n", 0))

// Disable log
db.LogMode(false)

// Enable log for a single operation, make debug easy
db.Debug().Where("name = ?", "jinzhu").First(&User{})
```

## Run Raw SQl

```go
db.Exec("drop table users;")
```

## Error Handling

```go
query := db.Where("name = ?", "jinzhu").First(&user)
query := db.First(&user).Limit(10).Find(&users)
//// query.Error keep the latest error happened
//// query.Errors keep all errors happened
//// If an error happened, gorm will stop do query, insert, update, delete

// I often use below code to do error handling in real applicatoins
err = db.Where("name = ?", "jinzhu").First(&user).Error
```

## Advanced Usage With Query Chain

Already excited about above usage? Let's see some magic!

```go
db.First(&first_article).Count(&total_count).Limit(10).Find(&first_page_articles).Offset(10).Find(&second_page_articles)
//// SELECT * FROM articles LIMIT 1; (first_article)
//// SELECT count(*) FROM articles; (count)
//// SELECT * FROM articles LIMIT 10; (first_page_articles)
//// SELECT * FROM articles LIMIT 10 OFFSET 10; (second_page_articles)

db.Where("created_at > ?", "2013-10-10").Find(&cancelled_orders, "state = ?", "cancelled").Find(&shipped_orders, "state = ?", "shipped")
//// SELECT * FROM orders WHERE created_at > '2013/10/10' AND state = 'cancelled'; (cancelled_orders)
//// SELECT * FROM orders WHERE created_at > '2013/10/10' AND state = 'shipped'; (shipped_orders)

db.Where("product_name = ?", "fancy_product").Find(&orders).Find(&shopping_carts)
//// SELECT * FROM orders WHERE product_name = 'fancy_product'; (orders)
//// SELECT * FROM carts WHERE product_name = 'fancy_product'; (shopping_carts)
// Do you noticed the table is different?

db.Where("mail_type = ?", "TEXT").Find(&users1).Table("deleted_users").First(&user2)
//// SELECT * FROM users WHERE mail_type = 'TEXT'; (users1)
//// SELECT * FROM deleted_users WHERE mail_type = 'TEXT'; (users2)

db.Where("email = ?", "x@example.org").Attrs(User{FromIp: "111.111.111.111"}).FirstOrCreate(&user)
//// SELECT * FROM users WHERE email = 'x@example.org';
//// INSERT INTO "users" (email,from_ip) VALUES ("x@example.org", "111.111.111.111") (if no record found)

// Open your mind, add more cool examples
```

## TODO
* Join, Having, Group, Includes
* Scopes, Valiations
* AlertColumn, DropColumn, AddIndex, RemoveIndex

# Author

**jinzhu**

* <http://github.com/jinzhu>
* <wosmvp@gmail.com>
* <http://twitter.com/zhangjinzhu>

## License

Released under the [MIT License](http://www.opensource.org/licenses/MIT).

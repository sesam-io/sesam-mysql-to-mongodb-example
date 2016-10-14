#MySQL to MongoDB with Sesam

In this tutorial we will explore some basic functionality of the Sesam data hub
and use it to export data from a relational MySQL database, add some spice and
push the enriched data to a NoSQL document MongoDB database.

Before reading on you should have the following requirements:
* A running Sesam appliance with a valid license installed.
* [Docker](https://docs.docker.com/engine/installation/)
* [Sesam client](https://docs.sesam.io/commandlineclient.html?highlight=sesamclient#installing-the-client)
* [MySQL client](https://dev.mysql.com/doc/refman/5.7/en/mysql.html)
* [MongoDB client](https://docs.mongodb.com/getting-started/shell/client/)

First we need to set up the databases that Sesam will be connected to. We will
spin up two docker containers with MySQL and MongoDB and one microservice
container which we will look into more closely a bit later on.

Before starting the containers we will create a docker network where the
containers will be added to make it easier to communicate with each other.

    $ docker network create sesam-mongodb

Now we are ready to spin the databases. From the root folder of this repository
run the following commands:

    $ docker run -d \
      --name mysql \
      --network=sesam-mongodb \
      -p 3306:3306 \
      -e MYSQL_ROOT_PASSWORD=root-password \
      -e MYSQL_DATABASE=northwind \
      -e MYSQL_USER=sesam \
      -e MYSQL_PASSWORD=sesam-password \
      -v ${PWD}/northwind:/docker-entrypoint-initdb.d \
      mysql:latest

    $ docker run -d \
      --name=mongodb \
      --network=sesam-mongodb \
      -p 27017:27017 \
      -p 28017:28017 \
      -e MONGODB_USER=sesam \
      -e MONGODB_PASS="password" \
      -e MONGODB_DATABASE="northwind" \
      tutum/mongodb

The `-v ${PWD}/northwind:/docker-entrypoint-initdb.d` part will add the
contents of the `northwind` folder of the repository to the
`/docker-entrypoint-initdb.d` folder inside the mysql container and the
contents of the that folder will run as mysql initiates. In that way we will
create the schema of the northwind database in MySQL and add some basic data.

We give some time for the containers to initiate. We can check the progress of
the initiation by running:

    $ docker logs mysql
    $ docker logs mongodb

We are ready when you can see the following messages:

    #mysql
    2016-10-13T07:58:57.663318Z 0 [Note] mysqld: ready for connections.
    Version: '5.7.15'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server (GPL)

    #mongodb
    => Done!
    ========================================================================
    You can now connect to this MongoDB server using:

    mongo northwind -u sesam -p password --host <host> --port <port>

    Please remember to change the above password as soon as possible!
    ========================================================================


Since Sesam does not have a native way to talk to MongoDB we have created a
microservice MongoDB sink that accepts JSON entities on an HTTP endpoint and
adds them to MongoDB. The code of that service can be found here:
[mongodb-sink](https://github.com/giskou/mongodb-sink)

    $ docker run -d \
      --name=mongodb-sink \
      --network=sesam-mongodb \
      -p 5001:5001 \
      -e MONGODB_HOST=mongodb \
      -e MONGODB_USERNAME=sesam \
      -e MONGODB_PASSWORD=password \
      -e MONGODB_DATABASE=northwind \
      giskou/mongodb-sink

Now we will start interacting with the Sesam data hub.
Since we are using the Sesam VirtualBox appliance we need to find the IP of the
host machine that the VM can use to contact the databases we have there.

    $ curl http://localhost:8042

You should get something like this:

    {"id": "...", "host_ip": "10.0.2.2"}

Edit the file `sesam/secrets.json` and replace the IP in `mysql-host` with the
one you got for from the previous command.
Do the same with file `sesam/system-mongodb-orders.json` and `mongodb-url`.

We can now add the secrets in the Sesam secret storage.

    $ sesam put-secrets sesam/secrets.json

Now lets create the mysql and mongodb systems in Sesam.

    $ sesam add-systems "[$(cat sesam/system-mysql.json),$(cat sesam/system-mongodb-orders.json)]"

We will now add two pipes to pull the `Orders` and `Order Details` tables from
MySQL into Sesam.

    $ sesam add-pipes "[$(cat sesam/pipe-orders.json),$(cat sesam/pipe-order-details.json)]"

Sesam has now created two datasets named `orders` and `order-details` that
contain the same data as the MySQL tables.

We will now combine the those two datasets to create a new dataset that has
the list of all the order details for every order using the foreign key of the
order-details table.

We want to keep the element id to be the same in mongodb so we can update
elements with changed values.  We also want to add some aggregated vales to the
new elements.  We will add the total number of order-details for every order,
the total number of items for every order and the total price of the order
after calculating any possible discount.

The next pipe we are going to add looks like this:

    {
        "_id": "orders-with-details",
        "type": "pipe",
        "source": {
            "type": "dataset",
            "dataset": "orders"
        },
        "sink": {
            "type": "dataset",
            "dataset": "orders-with-details"
        },
        "transform": {
            "type": "dtl",
            "rules": {
                "default": [
                    ["copy", ["list", "*", "_id"], "_*"],
                    ["add", "Items",
                        ["apply-hops", "order", {
                            "datasets": ["order-details o"],
                            "where": [
                                ["eq", "_S.OrderID", "o.OrderID"]
                            ]
                        }]
                    ],
                    ["add", "ItemsCount", ["count", "_T.Items"]],
                    ["add", "ProductsCount",
                        ["sum", ["map", "_.Quantity", "_T.Items"]]
                    ],
                    ["add", "TotalPrice",
                        ["sum", ["map", "_.TotalPrice", "_T.Items"]]
                    ]
                ],
                "order": [
                    ["copy",
                        ["list", "*", "_id"],
                        ["list", "_*", "OrderID"]
                    ],
                    ["add", "TotalPrice",
                        ["multiply",
                            ["multiply", "_S.Quantity", "_S.UnitPrice"],
                            ["minus", "_S.Discount", 1]
                        ]
                    ]
                ]
            }
        },
        "pump": {
            "schedule_interval": 10,
            "run_at_startup": true
        }
    }

We will now look into the transform part in more detail.

The first copy DTL function is copying all the values of the source element
plus the `_id` and skips all the other internal elements that start with `_`.

The next add command is creating an Items attribute that contains all the
elements that meet the `where` requirements of the `order-details` dataset
after they are passed through the `order` transformation.

In the order transformation we keep all the attributes except the internal `_`
and the `OrderID` which was serving as a foreign key to the `Orders` table.

We then calculate the total price of the sub-order into the `TotalPrice`
attribute by multiplying the number of the items to their value and subtracting
the discount.

Now based on the new computed values we can add some more attributes to the
order elements.

`ItemsCount` contains the number of all order-details for the order.

`ProductsCount` contains the amount of all the products that are contained in
the order.

`TotalPrice` contains the sum of all the `TotalPrice` attributed of all the
order-details of the specific order.

The original elements looked like this:

    $ sesam get-dataset-entity orders 10250
    {
        "CustomerID": "HANAR",
        "EmployeeID": 4,
        "Freight": "~f65.8300",
        "OrderDate": "~t1996-07-08T00:00:00Z",
        "OrderID": 10250,
        "RequiredDate": "~t1996-08-05T00:00:00Z",
        "ShipAddress": "Rua do Pao, 67",
        "ShipCity": "Rio de Janeiro",
        "ShipCountry": "Brazil",
        "ShipName": "Hanari Carnes",
        "ShippedDate": "~t1996-07-12T00:00:00Z",
        "ShipPostalCode": "05454-876",
        "ShipRegion": "RJ",
        "ShipVia": 2,
        "_deleted": false,
        "_hash": "73e309bf01d9bd12d0a54a55c2bf801d",
        "_id": "10250",
        "_previous": null,
        "_ts": 1476367265444553,
        "_updated": 2
    }

    $ sesam get-dataset-entity order-details 10250_41
    {
        "Discount": "~f0.00",
        "OrderID": 10250,
        "ProductID": 41,
        "Quantity": 10,
        "UnitPrice": "~f7.7000",
        "_deleted": false,
        "_hash": "46a21eb0ad33917d54e99190008a20a0",
        "_id": "10250_41",
        "_previous": null,
        "_ts": 1476367265542975,
        "_updated": 6
    }

    $ sesam get-dataset-entity order-details 10250_51
    {
        "Discount": "~f0.15",
        "OrderID": 10250,
        "ProductID": 51,
        "Quantity": 35,
        "UnitPrice": "~f42.4000",
        "_deleted": false,
        "_hash": "cebe9003710f6ce37455a74c1b6793eb",
        "_id": "10250_51",
        "_previous": null,
        "_ts": 1476367265542994,
        "_updated": 7
    }

    $ sesam get-dataset-entity order-details 10250_65
    {
        "Discount": "~f0.15",
        "OrderID": 10250,
        "ProductID": 65,
        "Quantity": 15,
        "UnitPrice": "~f16.8000",
        "_deleted": false,
        "_hash": "ff62988d82659ccae03b4f2187947994",
        "_id": "10250_65",
        "_previous": null,
        "_ts": 1476367265543012,
        "_updated": 8
    }

Lets add that last pipe and see the new elements:

    $ sesam add-pipes "[$(cat sesam/pipe-orders-with-details.json)]"

The new elements look like this:

    $ sesam get-dataset-entity orders-with-details 10250
    {
        "CustomerID": "HANAR",
        "EmployeeID": 4,
        "Freight": "~f65.8300",
        "Items": [
            {
                "Quantity": 10,
                "Discount": "~f0.00",
                "ProductID": 41,
                "UnitPrice": "~f7.7000",
                "TotalPrice": "~f77.000000"
            },
            {
                "Quantity": 15,
                "Discount": "~f0.15",
                "ProductID": 65,
                "UnitPrice": "~f16.8000",
                "TotalPrice": "~f214.200000"
            },
            {
                "Quantity": 35,
                "Discount": "~f0.15",
                "ProductID": 51,
                "UnitPrice": "~f42.4000",
                "TotalPrice": "~f1261.400000"
            }
        ],
        "ItemsCount": 3,
        "OrderDate": "~t1996-07-08T00:00:00Z",
        "OrderID": 10250,
        "ProductsCount": 60,
        "RequiredDate": "~t1996-08-05T00:00:00Z",
        "ShipAddress": "Rua do Pao, 67",
        "ShipCity": "Rio de Janeiro",
        "ShipCountry": "Brazil",
        "ShipName": "Hanari Carnes",
        "ShippedDate": "~t1996-07-12T00:00:00Z",
        "ShipPostalCode": "05454-876",
        "ShipRegion": "RJ",
        "ShipVia": 2,
        "TotalPrice": "~f1552.600000",
        "_deleted": false,
        "_hash": "6be097297c65686c2608a7c03689d07e",
        "_id": "10250",
        "_previous": null,
        "_ts": 1476371716090715,
        "_updated": 2
    }

Now we want to add that data to the MongoDB store, so we will add one last pipe
to push that data to the mongodb container we have running, through the
microservice we created before.

    $ sesam add-pipes "[$(cat sesam/pipe-mongodb.json)]"

We can now check MongoDB and see the new data:

    $ mongo localhost
    > use northwind
    > db.auth('sesam', 'password')
    > db.orders.find({"OrderID":10250}).pretty()
    {
        "CustomerID" : "HANAR",
        "EmployeeID" : 4,
        "Freight" : "~f65.8300",
        "Items" : [
            {
                "UnitPrice" : "~f7.7000",
                "TotalPrice" : "~f77.000000",
                "Discount" : "~f0.00",
                "Quantity" : 10,
                "ProductID" : 41
            },
            {
                "UnitPrice" : "~f16.8000",
                "TotalPrice" : "~f214.200000",
                "Discount" : "~f0.15",
                "Quantity" : 15,
                "ProductID" : 65
            },
            {
                "UnitPrice" : "~f42.4000",
                "TotalPrice" : "~f1261.400000",
                "Discount" : "~f0.15",
                "Quantity" : 35,
                "ProductID" : 51
            }
        ],
        "ItemsCount" : 3,
        "OrderDate" : "~t1996-07-08T00:00:00Z",
        "OrderID" : 10250,
        "ProductsCount" : 60,
        "RequiredDate" : "~t1996-08-05T00:00:00Z",
        "ShipAddress" : "Rua do Pao, 67",
        "ShipCity" : "Rio de Janeiro"
        "ShipCountry" : "Brazil",
        "ShipName" : "Hanari Carnes",
        "ShippedDate" : "~t1996-07-12T00:00:00Z",
        "ShipPostalCode" : "05454-876",
        "ShipRegion" : "RJ",
        "ShipVia" : 2,
        "TotalPrice" : "~f1552.600000",
        "_deleted" : false,
        "_hash" : "6be097297c65686c2608a7c03689d07e",
        "_id" : "10250",
        "_last_modified" : ISODate("2016-10-13T15:22:25.876Z"),
        "_previous" : null,
        "_ts" : NumberLong("1476371716090715"),
        "_updated" : 2,
    }

The microservice is also adding a `_last_modified` attribute since MongoDB does
not track this information.

Now lets add a new order detail and see the magic happen!

    $ mysql -u sesam -h 127.0.0.1 -p
    mysql> USE northwind;
    mysql> SELECT * FROM `Order Details` WHERE `OrderID`=10250;
    mysql> INSERT INTO `Order Details` (OrderID, ProductID, UnitPrice, Quantity, Discount) \
                  VALUES (10250, 11, 14.0000, 12, 0.10);
    mysql> exit

    $ sesam start-pump order-details
    $ sesam start-pump orders-with-details
    $ sesam start-pump orders-to-mongodb

    #MongoDB
    $ mongo localhost
    > use northwind
    > db.auth('sesam', 'password')
    > db.orders.find({"OrderID":10250}).pretty()
    {
        "CustomerID" : "HANAR",
        "EmployeeID" : 4,
        "Freight" : "~f65.8300",
        "Items" : [
            {
                "UnitPrice" : "~f7.7000",
                "TotalPrice" : "~f77.000000",
                "Discount" : "~f0.00",
                "Quantity" : 10,
                "ProductID" : 41
            },
            {
                "UnitPrice" : "~f16.8000",
                "TotalPrice" : "~f214.200000",
                "Discount" : "~f0.15",
                "Quantity" : 15,
                "ProductID" : 65
            },
            {
                "UnitPrice" : "~f42.4000",
                "TotalPrice" : "~f1261.400000",
                "Discount" : "~f0.15",
                "Quantity" : 35,
                "ProductID" : 51
            },
            {
                "UnitPrice" : "~f14.0000",
                "TotalPrice" : "~f151.200000",
                "Discount" : "~f0.10",
                "Quantity" : 12,
                "ProductID" : 11
            }
        ],
        "ItemsCount" : 4,
        "OrderDate" : "~t1996-07-08T00:00:00Z",
        "OrderID" : 10250,
        "ProductsCount" : 72,
        "RequiredDate" : "~t1996-08-05T00:00:00Z",
        "ShipAddress" : "Rua do Pao, 67",
        "ShipCity" : "Rio de Janeiro",
        "ShipCountry" : "Brazil",
        "ShipName" : "Hanari Carnes",
        "ShipPostalCode" : "05454-876",
        "ShipRegion" : "RJ",
        "ShipVia" : 2,
        "ShippedDate" : "~t1996-07-12T00:00:00Z",
        "TotalPrice" : "~f1703.800000",
        "_deleted" : false,
        "_hash" : "c6607670d2b4c57bfd3ec12706fa76ff",
        "_id" : "10250",
        "_last_modified" : ISODate("2016-10-13T15:28:28.871Z"),
        "_previous" : 2,
        "_ts" : NumberLong("1476372500925052"),
        "_updated" : 830
    }

The new order detail is added to the Items and all the other values have been
updated!

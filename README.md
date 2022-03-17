# PostgREST Setup

##### Reference: https://postgrest.org/en/stable/tutorials/tut0.html

1. Install and run PostgreSQL Docker Container

```
sudo docker run --name postgres -p 5433:5432 \
                -e POSTGRES_PASSWORD=password \
                -d postgres
```

2. Obtain the PostgREST binary from: https://github.com/PostgREST/postgrest/releases/tag/v9.0.0 and unpack it to your local machine. Test that it works by running the binary file. 

*If you encounter an error like `error while loading shared libraries: libpq.so.5, see image:*

![libpq error](./images/libpq_error.jpg)

3. Access the Postgres container:

```
sudo docker exec -it postgres psql -U postgres
```

Now it is time to configure the database and PostgREST

# Configuring the Database

1. Create a schema: 
```
postgres=# create schema api;
```
2. Create a table for your data:
```
postgres=# create table api.products (
    id serial primary key,
    name VARCHAR ( 50 ) UNIQUE NOT NULL,
    price VARCHAR ( 20 ) NOT NULL,
    quantity INT NOT NULL
);
```

3. Create some entries to the table:
```
postgres=# insert into api.products (name, price, quantity)
    values('Banana', '0.79', 88);

postgres=# insert into api.products (name, price, quantity)
    values('Apple', '1.29', 26);

postgres=# insert into api.products (name, price, quantity)
    values('Pineapple', '6.29', 18);
```

4. Configure a role to use for anonymous web requests (PostgREST will use this role for web requests from anonymous sources)
```
postgres=# create role web_anon nologin;

postgres=# grant usage on schema api to web_anon;
postgres=# grant select on api.products to web_anon;
```

It is also useful to have an authenticator role without admin privileges for connecting to the database, and also allow to switch to the web_anon role:
```
postgres=# create role authenticator noinherit login password 'password';

postgres=# grant web_anon to authenticator;

postgres=# grant usage on schema api to web_anon;
postgres=# grant all on api.products to web_anon;
postgres=# grant usage, select on sequence api.products_id_seq to web_anon;

```

5. Create a Secret Key
In order to successfully authenticate the client and allow them special privileges to the database, we need to create JSON Web Tokens for the requests. JSON Web tokens are cryptographically signed with a secret password that only the server and admin knows. Following best security practices, the password should be a minimum of 32 characters. We need to add the following line to the configuration file.

```
jwt-secret = "<the password you made>"
```

7. Sign a Token. A signed authentication token is needed to properly make requests to the server. We can use jwt.io to sign a token. 

jwtImage.PNG
![Signing a Token](images/jwtImage.PNG)


8. Adding an expiration time to the token for security purposes and to ensure the authentication token becomes invalid after a certain period of time. Run the following command in psql to add an expiration of 10 minutes. 

```
select extract(epoch from now() + '10 minutes'::interval) :: integer;
```

Now, change the payload of the web token at jwt.io to add the new epoch value
```
{
  "role": "todo_user",
  "exp": 123456789
}

```
NOTE: Your epoch value should be different than the example one used above. 

Now, copy and save the new token as a new environment variable
```
export NEW_TOKEN="<paste new token>"
```

After adding this, the requests will be invalid after the expiration time


9. With the configuration finished, exit psql:
```
postgres=# \q
```

# Configuring and starting PostgREST

1. Create a configuration file for PostgREST. The configuration file for this tutorial will be called postgrest.conf

*postgrest.conf*
```
db-uri = "postgres://authenticator:password@localhost:5433/postgres"
db-schemas = "api"
db-anon-role = "web_anon"
```

2. Run the PostgREST server
```
$ ./postgrest postgrest.conf
```

3. Call the API at localhost:3000. Use curl, Postman, or whatever other tool you would like.

![output from postman](images/PostmanResult.png)

4. Create a new product with a call to the API using your selected method.

5. Update an existing product with a call to the API and a specific ID using your selected method. 

6. Delete an existing product by identifying the unique ID of the element. 

7. Read the list of products to display the updated list. 


We have now successfully create a RESTful API out-of-the-box using PostgREST.

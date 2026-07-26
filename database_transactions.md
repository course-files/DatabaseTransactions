# Lab Manual on Database Transactions

## Learning Outcomes

By the end of this lab, you will be able to:

1. **Explain and control** commit behaviour in PostgreSQL (and why it differs conceptually from MySQL's server-side `autocommit` variable)
2. **Set** the isolation level of a transaction
3. **Start** a transaction
4. **Create** a **SAVEPOINT** in a transaction
5. **Roll back** a transaction to a named **SAVEPOINT**
6. **Commit** a transaction
7. **Terminate** a stuck or long-running transaction/session
8. **Embed** a database transaction written in SQL inside Python
9. **Edit** the PostgreSQL configuration file to support database transactions using the following parameters: `default_transaction_isolation`,`statement_timeout`, and `idle_in_transaction_session_timeout`.

## Required Software

| # | Software                                       | Notes                                                                |
|---|------------------------------------------------|----------------------------------------------------------------------|
| 1 | PostgreSQL Server (==18)                       | Installed and running in your Ubuntu Server Virtual Machine (VM)     |
| 2 | Secure Shell (SSH)                             | SSH access is required for connecting to the VM                      |
| 3 | `psql`                                         | A PostgreSQL client that ships with the PostgreSQL server by default |
| 4 | Python 3 and `psycopg2` (or `psycopg2-binary`) | Required for embedding SQL in Python                                 |
| 5 | Code Editors and IDEs: VS Code and PyCharm     | We will use both VS Code and PyCharm                                 |
| 6 | PGAdmin4                                       | If you choose to connect from your laptop over the network           |

## Prerequisites

1. PostgreSQL is installed in your Ubuntu Server VM, and it is running as a service. Refer to the following lab manual for instructions on how to install PostgreSQL in your Ubuntu Server VM if you had not already done so in a previous lab: [https://github.com/course-files/RelationalAlgebra/blob/main/part_1_install_postgresql_in_ubuntu_server.md](https://github.com/course-files/RelationalAlgebra/blob/main/part_1_install_postgresql_in_ubuntu_server.md)
2. You have SSH access to the VM. Refer to the following lab manual for instructions on how to set up SSH: [https://github.com/course-files/RelationalAlgebra/blob/main/part_1_install_postgresql_in_ubuntu_server.md](https://github.com/course-files/RelationalAlgebra/blob/main/part_1_install_postgresql_in_ubuntu_server.md)
3. You are comfortable with basic SQL (`SELECT`, `INSERT`, `UPDATE`, `DELETE`). If not, please review the following tutorial before proceeding: [https://www.postgresql.org/docs/current/tutorial-sql.html](https://www.postgresql.org/docs/current/tutorial-sql.html)

## Approximate Time Required

4 hours

---

## Narrative

A transaction is **a sequential group of operations, executed as a single unit of work, to update a database to reflect the occurrence of an event**. A transaction includes one or more of the following operations:

(i) **Create:** To insert new data  
(ii) **Read:** To retrieve existing data  
(iii) **Update:** To change the value of existing data  
(iv) **Delete:** To remove existing data  

The DBMS is responsible for ensuring that either all the operations in a transaction complete successfully and their effect is recorded (the transaction is **"committed"**), or that the transaction has no effect whatsoever on the database, and the database is restored to its state prior to the transaction’s execution (the transaction is **"aborted"**).

The context of the data in the database is a restaurant called "Siwaka Dishes". It models a multi-branch Kenyan restaurant serving African food. The database includes tables for branches, employees, products, customers, customer orders, order details, payments, and customer feedback.

The database transaction simulates the following algorithm in a Point-of-Sale (POS) Information System:

* A customer places an order for one or more products.  
* The restaurant's cashier creates an order in the database.  
* A **SAVEPOINT** is created before each product is added to the order. This allows the cashier to roll back the transaction to a specific point if a product is not added successfully.  
* The cashier adds the product the customer ordered to the order in the database. This includes the `product_id`, `quantity_ordered`, and `price_each`.  
* The `quantity_in_stock` for each product ordered is updated to reflect the sale.  
* If the product is added successfully, then the cashier adds the next product to the order. If the product is not added successfully, then the cashier rolls back the transaction to the **SAVEPOINT** created before the product was added, and then adds the next product to the order.  
* Once all products have been added to the order, the cashier requests payment for the order. This is received and the `payment` table is updated to reflect the payment.  
* The transaction is **committed** to the database, and a receipt is printed for the customer.  

---

## Step 1: SSH into the VM from Your Laptop

Execute the following in the Ubuntu Server VM's terminal to confirm the IP address of the host-only network interface:

```bash
ip addr show
```

Note the IP address assigned to the `enp0s8` interface. `enp0s8` stands for Ethernet adapter located on PCI bus 0, slot 8.

Below is an example of the output you should see:

```text
student@classlab:~$ ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8a:c1:6f brd ff:ff:ff:ff:ff:ff
    altname enx0800278ac16f
    inet 10.0.2.15/24 metric 100 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 81403sec preferred_lft 81403sec
    inet6 fd17:625c:f037:2:a00:27ff:fe8a:c16f/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 86112sec preferred_lft 14112sec
    inet6 fe80::a00:27ff:fe8a:c16f/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:b8:27:f7 brd ff:ff:ff:ff:ff:ff
    altname enx080027b827f7
    inet 192.168.56.103/24 metric 100 brd 192.168.56.255 scope global dynamic enp0s8
       valid_lft 489sec preferred_lft 489sec
    inet6 fe80::a00:27ff:feb8:27f7/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
```

In this example, the IP address of the VM is `192.168.56.103`. **Replace this with the actual IP address of your VM when executing the following commands.**

Ping the VM **from your host machine** to confirm connectivity:

```bash
ping 192.168.56.103
```

This is the **Host-Only Adapter** which is used by the host to access the VM.

Leave the VM window and control the server entirely from your laptop's terminal or an application like PuTTY ([https://putty.org/index.html](https://putty.org/index.html)) or Termius ([https://termius.com/download/](https://termius.com/download/)).

If you are using your laptop's terminal, then use the **Git Bash** terminal if you are on Windows (**NOT** **PowerShell** or any other terminal) or the default terminal if you are on Linux or macOS. This is so that we have a consistent experience across all platforms. The lab manual assumes you are using the Git Bash terminal on Windows, and the default terminal on Linux or macOS.

Use the VM's Host-Only IP address directly from the host machine. For example, assuming the VM's Host-Only IP address is `192.168.56.103`:

```bash
ssh -p 22 student@192.168.56.103
```

Replace `192.168.56.103` with the actual IP address of your VM in the host-only network.

You will be prompted to enter the password for the `student` user, which you set during the Ubuntu Server installation.

You will see a message like this if it is the first time you are connecting to the VM via SSH:

```text
The authenticity of host '192.168.56.103 (192.168.56.103)' can't be established.

ED25519 key fingerprint is: SHA256:...

This key is not known by any other names.

Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Type `yes` and press Enter. This is normal on the first connection — SSH is recording the server's identity so it can detect if it changes in the future.

Enter your password when prompted.

You should now see:

```bash
student@classlab:~$
```

## Step 2: Setup the Database

Refer to the instructions available here to create the database and load the synthetic data into the database if you had not done so already in a previous lab: [https://github.com/course-files/RelationalAlgebra/blob/main/part_3_synthetic_data.md](https://github.com/course-files/RelationalAlgebra/blob/main/part_3_synthetic_data.md)

## Step 3: Connect to the PostgreSQL server

Confirm the server is running by executing the following in the Ubuntu Server's terminal via SSH:

```bash
sudo systemctl status postgresql
```

Connect using the `postgres` account (the default superuser account created during PostgreSQL installation) to the `siwaka_dishes` database:

```bash
psql -U postgres -h localhost -W -d siwaka_dishes
```

You should land on a `siwaka_dishes=#` prompt.

## Step 4: Understand commit behavior in PostgreSQL

* In **MySQL**, every statement outside an explicit `START TRANSACTION` is committed immediately by the **server**, and `autocommit` toggles that server-side behavior for the session.
* In **PostgreSQL**, the same effect (each standalone statement behaving as its own transaction) happens automatically whenever a statement is issued **outside** an explicit `BEGIN ... COMMIT` block. There is nothing to switch off at the server level — you get single-statement, transaction-safe behavior by default, and multi-statement transactional behavior the moment you issue `BEGIN`.
* The `psql` client offers a **client-side** convenience that mirrors the MySQL toggle, for situations where you want every single statement (even ones not wrapped in `BEGIN`) to require an explicit `COMMIT`:

```sql
\set AUTOCOMMIT off
\echo :AUTOCOMMIT
```

For the rest of this lab, you do not need `\set AUTOCOMMIT off` — every step from Step 5 onward is wrapped inside an explicit `BEGIN ... COMMIT` block, which is the idiomatic PostgreSQL approach and works identically regardless of the client's autocommit setting.

## Step 5: Set the isolation level

The isolation property of transactions ensures that the execution of a transaction is not interfered with by other transactions executing concurrently. Interference from other transactions can present as any of the following problems:

(i) Lost update problem  
(ii) Dirty read problem  
(iii) Incorrect summary problem  
(iv) Unrepeatable read problem  
(v) Phantom problem  

![concurrency_problems](/assets/images/concurrency_problems.png)

PostgreSQL's **concurrency control subsystem** uses **Multi-Version Concurrency Control (MVCC)** rather than a **lock-based approach**. The table below reflects PostgreSQL's behavior:

| Isolation Level              | Dirty Read | Unrepeatable Read | Lost Update      | Incorrect Summary | Phantom Read    |
|------------------------------|------------|-------------------|------------------|-------------------|-----------------|
| **READ UNCOMMITTED**         | Prevented¹ | Can still occur   | Can still occur  | Can still occur   | Can still occur |
| **READ COMMITTED** (default) | Prevented  | Can still occur   | Can still occur² | Can still occur   | Can still occur |
| **REPEATABLE READ**          | Prevented  | Prevented         | Prevented³       | Prevented         | Prevented³      |
| **SERIALIZABLE**             | Prevented  | Prevented         | Prevented        | Prevented         | Prevented       |

¹ Unlike MySQL, PostgreSQL never allows dirty reads at any isolation level — `READ UNCOMMITTED` is silently treated as `READ COMMITTED`.  
² A "read-then-write" lost update (read a value, compute a new value in application code, then write it back — exactly the pattern used later in this lab for `quantity in stock`) **can** happen at `READ COMMITTED` if two sessions interleave. However, explicit locking such as `SELECT FOR UPDATE` or **Optimistic Concurrency Control (OCC)** can be used to prevent the lost update problem at a **READ COMMITTED** isolation level.  
³ PostgreSQL's `REPEATABLE READ` is implemented as true snapshot isolation, which is stricter than the SQL standard requires and also prevents phantom reads — unlike MySQL, where `REPEATABLE READ` still allows the phantom problem.

Confirm the current default isolation level:

```sql
SHOW default_transaction_isolation;
```

Execute the following to set the isolation level of the *next* transaction to `SERIALIZABLE`, the strictest level. Note that in PostgreSQL, this command must be the **first** statement issued after `BEGIN`:

```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SHOW TRANSACTION ISOLATION LEVEL;
ROLLBACK;
```

The commands for each isolation level in PostgreSQL are:

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

The setting applies only to the current transaction if issued after `BEGIN`, or can be set as a session default. Perform the steps below for a more permanent, cluster-wide default:

**Step 1:** Find the configuration files.

We can confirm the path to the configuration file by asking PostgreSQL directly through `psql`:

```sql
SHOW config_file;
```

This returns the absolute path, typically similar to:

`/etc/postgresql/18/main/postgresql.conf`

Alternatively, you can execute the following command in the shell to locate the configuration file:

```bash
ls -al /etc/postgresql/*/main/
```

or

```bash
sudo find /etc/postgresql -name postgresql.conf
```

Copy the path to the configuration file for use in the next step.

**Step 2:** Edit `postgresql.conf`.

```bash
sudo vim /etc/postgresql/*/main/postgresql.conf
```

Use the vim search function to jump directly to the line rather than scrolling manually. In normal mode (i.e., before pressing `i`), type:

```text
/isolation
```

Press `Enter` then type `i` to enter insert mode. Remove the `#` and change the value:

from:

`#default_transaction_isolation = 'read committed'`

to:

`default_transaction_isolation = 'serializable'`

Press `Esc` to exit insert mode, then type `:wq` and press `Enter` to save and quit.

**Step 3:** Reload the service: `sudo systemctl reload postgresql`.

Confirm the server is running:

```bash
sudo systemctl status postgresql
```

**Step 4:** Confirm the new default isolation level:

```sql
psql -U siwaka_dishes_app_runtime -h localhost -W -d siwaka_dishes
```

```sql
SHOW default_transaction_isolation;
```

## Step 6: Start the transaction

```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

**Note:** `BEGIN` and `START TRANSACTION` are interchangeable in PostgreSQL.

## Step 7: Calculate the latest order number

PostgreSQL has no direct equivalent of MySQL's user-defined session variables (`SET @var = ...`). The idiomatic `psql` equivalent is the **`\gset`** meta-command, which stores the result of a query into a `psql` client-side variable that can then be referenced with a leading colon (`:variable_name`) in later statements.

```sql
SELECT MAX(order_number) + 1 AS order_number FROM customer_order; \gset
\echo Next order number is: :order_number
```

This evaluates to customer order number **2501**.

## Step 8: Confirm the timezone

To check the timezone of a Linux server (Ubuntu):

```shell
timedatectl
```

To check the timezone of the PostgreSQL server:

```sql
SHOW timezone;
```

## Step 9: Create a new order

The table `customer_order` was created as:

```sql
CREATE TABLE customer_order (
    order_number INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_date TIMESTAMP NOT NULL,
    required_date TIMESTAMP NOT NULL,
    dispatch_date TIMESTAMP,

    order_status_id INT NOT NULL,
    customer_number INT NOT NULL,
    branch_code INT NOT NULL,

    CONSTRAINT fk_1_customer_to_m_customer_order
    FOREIGN KEY (customer_number)
    REFERENCES customer(customer_number)
    ON DELETE CASCADE,

    CONSTRAINT fk_1_order_status_to_m_customer_order
    FOREIGN KEY (order_status_id)
    REFERENCES order_status(order_status_id),

    CONSTRAINT fk_1_branch_to_m_customer_order
    FOREIGN KEY (branch_code)
    REFERENCES branch(branch_code)
);
```

`order_number` is defined as an **identity column** (PostgreSQL's equivalent of MySQL's `AUTO_INCREMENT`), so it is automatically generated if not provided. However, the lab exercise explicitly calculates the next order number to demonstrate how to use a calculated value in a transaction.

A **GENERATED ALWAYS** identity column normally does not allow you to supply your own value. PostgreSQL generates it automatically. We therefore execute the following to create a new customer order while allowing PostgreSQL to generate the `order_number` automatically:

```sql
INSERT INTO
    public.customer_order(
        -- order_number,
        order_date,
        required_date,
        dispatch_date,
        order_status_id,
        customer_number,
        branch_code
    )
VALUES
    (
        -- :order_number,
        CURRENT_TIMESTAMP,
        CURRENT_TIMESTAMP + INTERVAL '1 hour',
        CURRENT_TIMESTAMP + INTERVAL '45 minutes',
        3,
        264,
        16
    )

RETURNING order_number \gset
\echo Inserted order number: :order_number
```

An alternative (which we **DO NOT** use in this lab) is to override the system-generated value by using the `OVERRIDING SYSTEM VALUE` clause:

```sql
INSERT INTO
	public.customer_order
    OVERRIDING SYSTEM VALUE
    (
		order_number,
		order_date,
		required_date,
		dispatch_date,
		order_status_id,
		customer_number,
		branch_code
	)
VALUES
	(
		:order_number,
		CURRENT_TIMESTAMP,
		CURRENT_TIMESTAMP + INTERVAL '1 hour',
		CURRENT_TIMESTAMP + INTERVAL '45 minutes',
		3,
		264,
		16
	);
```

## Step 10: Create the first SAVEPOINT before product 1 is inserted

A `SAVEPOINT` divides a transaction into multiple units of work so that you can roll back to a specific point instead of undoing the entire transaction. The command to roll back the **entire** transaction is (**DO NOT** execute this (the `ROLLBACK` command) now — it would undo everything): `ROLLBACK;`

Execute the following to create the savepoint before the first product is inserted:

```sql
SAVEPOINT before_product_1;
```

The following command destroys a named `SAVEPOINT` without undoing the effects of statements executed after it was created (**DO NOT** execute this (the `RELEASE SAVEPOINT` command) now): `RELEASE SAVEPOINT savepoint_name;`

## Step 11: Insert the first product ordered and update its "quantity in stock"

**Part A:** Get the product's selling price:

```sql
SELECT
    selling_price
FROM
    public.product
WHERE
    product_code = 'P018'; \gset

\echo The selling price is: :selling_price
```

**Part B:** Insert the product as part of the customer's order.

```sql
INSERT INTO
    public.order_detail(
        -- order_detail_number,
        order_number,
        product_code,
        quantity_ordered,
        price_each
    )
VALUES
    (
        -- ?,
        :order_number,
        'P018',
        5,
        :selling_price
    );
```

**Part C:** Get the product's quantity in stock:

```sql
SELECT
    quantity_in_stock
FROM
    public.product
WHERE
    product_code = 'P018'; \gset

\echo The quantity in stock is: :quantity_in_stock
```

**Part D:** Update the quantity in stock to reflect the sale:

```sql
UPDATE
    public.product
SET
    quantity_in_stock = :quantity_in_stock - 5
WHERE
    product_code = 'P018';
```

**Part E:** Confirm that the quantity in stock has been updated:

```sql
SELECT
    quantity_in_stock
FROM
    public.product
WHERE
    product_code = 'P018'; \gset

\echo The updated quantity in stock (before committing) is: :quantity_in_stock
```

## Step 12: Create the second SAVEPOINT before product 2 is inserted

```sql
SAVEPOINT before_product_2;
```

## Step 13: Insert the second product ordered and update its "quantity in stock"

**Part A:** Get the product's selling price:

```sql
SELECT
    selling_price
FROM
    public.product
WHERE
    product_code = 'P008'; \gset

\echo The selling price is: :selling_price
```

**Part B:** Insert the product as part of the customer's order.

```sql
INSERT INTO
    public.order_detail(
        -- order_detail_number,
        order_number,
        product_code,
        quantity_ordered,
        price_each
    )
VALUES
    (
        -- ?,
        :order_number,
        'P008',
        100,
        :selling_price
    );
```

**Part C:** Get the product's quantity in stock:

```sql
SELECT
    quantity_in_stock
FROM
    public.product
WHERE
    product_code = 'P008'; \gset

\echo The quantity in stock is: :quantity_in_stock
```

**Part D:** Update the quantity in stock to reflect the sale:

```sql
UPDATE
    public.product
SET
    quantity_in_stock = :quantity_in_stock - 100
WHERE
    product_code = 'P008';
```

**Part E:** Confirm that the quantity in stock has been updated:

```sql
SELECT
    quantity_in_stock
FROM
    public.product
WHERE
    product_code = 'P008'; \gset

\echo The updated quantity in stock (before committing) is: :quantity_in_stock
```

## Step 14: Roll back the transaction to a SAVEPOINT

Suppose a teller makes a data-entry mistake on the second product and needs to undo just that entry before continuing.

Some Point of Sale (POS) systems require the teller's immediate supervisor to approve the rollback of a transaction. This prevents a possible loophole where a corrupt teller scans a product, then rolls it back but still goes ahead to receive the payment for the product. In such a case, if the calculation of the total amount is not rolled back, then the payment goes to the teller's pocket instead of going to the business's account. This is why a supervisor's approval is required to roll back a transaction.

Roll back to the savepoint created before the second product was inserted, so it is as though the second product (Product Code **P008**) was never ordered:

```sql
ROLLBACK TO SAVEPOINT before_product_2;
```

Had you executed plain `ROLLBACK;` instead, the entire transaction would have been undone back to Step 5.

## Step 15: Create the third SAVEPOINT and insert the third product

```sql
SAVEPOINT before_product_3;
```

**Part A:** Get the product's selling price:

```sql
SELECT
    selling_price
FROM
    public.product
WHERE
    product_code = 'P002'; \gset

\echo The selling price is: :selling_price
```

**Part B:** Insert the product as part of the customer's order.

```sql
INSERT INTO
    public.order_detail(
        -- order_detail_number,
        order_number,
        product_code,
        quantity_ordered,
        price_each
    )
VALUES
    (
        -- ?,
        :order_number,
        'P002',
        1,
        :selling_price
    );
```

**Part C:** Get the product's quantity in stock:

```sql
SELECT
    quantity_in_stock
FROM
    public.product
WHERE
    product_code = 'P002'; \gset

\echo The quantity in stock is: :quantity_in_stock
```

**Part D:** Update the quantity in stock to reflect the sale:

```sql
UPDATE
    public.product
SET
    quantity_in_stock = :quantity_in_stock - 1
WHERE
    product_code = 'P002';
```

**Part E:** Confirm that the quantity in stock has been updated:

```sql
SELECT
    quantity_in_stock
FROM
    public.product
WHERE
    product_code = 'P002'; \gset

\echo The updated quantity in stock (before committing) is: :quantity_in_stock
```

## Step 16: Request for payment for the order

```sql
SELECT
    SUM(quantity_ordered * price_each) AS total
FROM
    public.order_detail
WHERE
    order_number = :order_number; \gset

\echo The total amount to be paid is: :total
```

## Step 17: Receive the payment for the order

```sql
INSERT INTO
    public.payment(
        -- payment_number,
        order_number,
        payment_date,
        amount,
        payment_method_id
    )
VALUES
    (
        -- ?,
        :order_number,
        CURRENT_TIMESTAMP,
        :total,
        1);
```

## Step 18: COMMIT the transaction

```sql
COMMIT;
```

This concludes the transaction and ensures the changes are recorded in the database — the **durability** property of transactions.

## Step 19: Print the receipt for the customer

```sql
SELECT
    public.order_detail.order_number,
    public.order_detail.product_code,
    public.product.product_name,
    public.order_detail.quantity_ordered,
    public.order_detail.price_each,
    (public.order_detail.quantity_ordered * public.order_detail.price_each) AS line_total
FROM
    public.order_detail INNER JOIN public.product ON public.order_detail.product_code = public.product.product_code
WHERE
    public.order_detail.order_number = :order_number
ORDER BY
    public.order_detail.product_code;
```

Verified result:

```text
 order_number | product_code | product_name | quantity_ordered | price_each | line_total 
--------------+--------------+--------------+------------------+------------+------------
         2509 | P002         | Sukuma Wiki  |                1 |      20.00 |      20.00
         2509 | P018         | Mishkaki     |                5 |     200.00 |    1000.00
(2 rows)
```

100 items of product `P008` (`Kachumbari`) does not appear in the receipt because it was rolled back before the transaction was committed. Its original stock level of 100 is also retained, confirming that both its `INSERT INTO order_detail...` and `UPDATE product SET quantity_in_stock...` were undone. Only items 1 and 3 appear; exactly as intended.

A Point-of-Sale (POS) system can be used to format the receipt appropriately. Typical sections of a receipt include:

![receipt sections](/assets/images/receipt_sections.png)

*Source: Microsoft Copilot*

---

## Step 20: Open a session in `psql` using the `postgres` account

```shell
psql -U postgres -h localhost -W -d siwaka_dishes
```

Confirm you are logged in as the correct user and connected to the correct database:

```sql
SELECT current_user, current_database();
```

This session must remain open throughout the remaining steps; do not close it.

## Step 21: Establish a second SSH session and PostgreSQL session using the `siwaka_dishes_analytics` account

Open a new, separate terminal window (do not close or reuse the first):

>Replace `192.168.33.3` with the actual IP address of your VM in the host-only network.

```shell
ssh -p 22 student@192.168.33.3

psql -U siwaka_dishes_analytics -h localhost -W -d siwaka_dishes
```

Confirm you are logged in as the correct user and connected to the correct database:

```sql
SELECT current_user, current_database();
```

## Step 22: Start a long-running transaction in the second session

`pg_sleep(120)` pauses the backend process executing it for 120 seconds, then returns with no result rows of consequence (a `void` return type). It does not consume CPU while sleeping; the backend simply waits.

```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT pg_sleep(120);
```

**This session will now appear "frozen". This is expected.** Do not type anything further into this window. Immediately switch to the `siwaka_dishes_db_admin` session and proceed to the next step. You have **120 seconds** before the transaction closes on its own.

## Step 23: View the running transaction in the first session

Switch to the `postgres` session, which is not locked, and run:

```sql
SELECT
    pid,
    usename,
    datname,
    state,
    now() - xact_start AS transaction_duration,
    now() - query_start AS query_duration,
    query
FROM
    pg_stat_activity
WHERE
    xact_start IS NOT NULL
    AND datname = 'siwaka_dishes'
    AND pid <> pg_backend_pid()
ORDER BY
    xact_start;
```

You should see one row, for the `siwaka_dishes_analytics` session, with `state = active` and a growing `transaction_duration` and `query_duration` each time you re-execute the query. If the result set is empty, more than 120 seconds have elapsed since the transaction using `siwaka_dishes_analytics` started; return to Step 20 and repeat, switching to this session more quickly.

Record the `pid` value from this result; it is required for Step 22.

## Step 24: Terminate the stuck or long-running transaction, from the first session

Before proceeding, confirm the role used in the first session has the necessary privilege to terminate a session belonging to a different user:

```sql
SELECT rolsuper,
       pg_has_role(current_user, 'pg_signal_backend', 'member') AS can_signal_others
FROM pg_roles
WHERE rolname = current_user;
```

If both values return `false`, `pg_cancel_backend` and `pg_terminate_backend` will return `false` without effect in the next step, and you must reconnect using a superuser role or a role granted `pg_signal_backend` membership before continuing.

There are two options, using the `pid` recorded in the previous step.

**1. Cancel the currently running query, leaving the session alive:**

```sql
SELECT pg_cancel_backend(<pid>);
```

If the transaction is idle (no statement running), this has no effect.

**2. Terminate the entire backend session** (rolls back the transaction and closes the connection). This is a last resort, used when `pg_cancel_backend` does not resolve the issue, or when the session is idle in transaction and holding locks:

```sql
SELECT pg_terminate_backend(<pid>);
```

Confirm termination succeeded by re-running the query used to view running transactions; the row should no longer appear.

Both functions require the calling role to either own the target session or have appropriate privileges (superuser, or membership in pg_signal_backend from PostgreSQL 14 onward for terminating other users' non-superuser backends).

---

## Step 25: Set a permanent idle transaction timeout in postgresql.conf

### Step 25.1: Locate the configuration file

We can confirm the path to the configuration file by asking PostgreSQL directly through `psql`:

```sql
SHOW config_file;
```

This returns the absolute path, typically similar to:

`/etc/postgresql/18/main/postgresql.conf`

Alternatively, you can execute the following command in the shell to locate the configuration file:

```bash
ls -al /etc/postgresql/*/main/
```

or

```bash
sudo find /etc/postgresql -name postgresql.conf
```

Copy the path to the configuration file for use in the next step.

### Step 25.2: Open the file on the server

From your SSH session (not psql):

```shell
sudo vim /etc/postgresql/18/main/postgresql.conf
```

Substitute the path with the actual path returned in the previous step. It can vary by installation method and operating system.

### Step 25.3: Locate or add the setting

Use the vim search function to jump directly to the line rather than scrolling manually. In normal mode (i.e., before pressing i), type:

```text
/idle_in_transaction_session_timeout
```

Press `Enter` then type `i` to enter insert mode. Remove the `#` and change the value:

`idle_in_transaction_session_timeout = 180000`

Press `Esc` to exit insert mode, then type `:wq` and press `Enter` to save and quit.

The unit here is milliseconds when set as a bare number in the configuration file, unlike the SQL SET command, which also accepts unit-suffixed strings. 180000 corresponds to 3 minutes. To avoid unit-conversion mistakes, you can alternatively use an explicit suffix instead, which `postgresql.conf` also accepts:

`idle_in_transaction_session_timeout = '3min'`

Press `Esc` to exit insert mode, then type `:wq` and press `Enter` to save (**w**rite) and **q**uit.

### Step 25.4: Reload the configuration

This setting is not one of the parameters requiring a full server restart; it can be applied with a reload, which does not interrupt existing connections.

```shell
sudo systemctl reload postgresql
```

Or, from within `psql` as a superuser, without needing shell access to the server:

```sql
SELECT pg_reload_conf();
```

### Step 25.5: Confirm the new value is active

Reconnect (a reload does not change already-open sessions' inherited value, only new connections and future evaluations) and check:

```sql
SHOW idle_in_transaction_session_timeout;
```

Expected result:

```text
 idle_in_transaction_session_timeout 
-------------------------------------
 3min
(1 row)
```
You can also confirm the source of the parameter value. This is useful for distinguishing a config-file setting from a role-level or database-level overrides applied on top of it:

```sql
SELECT name, setting, source, sourcefile, sourceline
FROM pg_settings
WHERE name = 'idle_in_transaction_session_timeout';
```

`source` will be specified as `configuration file` if it is coming from `postgresql.conf` as set here, versus `session` or `database` if something else has overridden it since.

### Step 25.6: Verify with a real transaction

We then repeat the two-session exercise from the previous steps, but this time, we use an idle transaction rather than `pg_sleep` for a long-running query.:

```sql
BEGIN;
SELECT 1;
```

View the ongoing transaction in the other session, then wait for **3 minutes**. The idle transaction should be automatically terminated by PostgreSQL once the configured duration elapses, without any manual intervention that uses `pg_cancel_backend` or `pg_terminate_backend`

```sql
SELECT
    pid,
    usename,
    datname,
    state,
    now() - xact_start AS transaction_duration,
    now() - query_start AS query_duration,
    query
FROM
    pg_stat_activity
WHERE
    xact_start IS NOT NULL
    AND datname = 'siwaka_dishes'
    AND pid <> pg_backend_pid()
ORDER BY
    xact_start;
```

Confirm the session terminates automatically once the configured duration elapses, without any manual `pg_cancel_backend` or `pg_terminate_backend` call.

## Step 26: Set a Permanent Statement Timeout in postgresql.conf

`idle_in_transaction_session_timeout` protects against a transaction left open with nothing executing. `statement_timeout` protects against a single statement that runs for too long while actively executing — the exact scenario simulated by `pg_sleep(120)` earlier in this lab. A **Database Administrator** who only configures `idle_in_transaction_session_timeout` may conclude, incorrectly, that they have covered "long-running transactions" in general; they have not covered this case. Both are needed for complete protection, and they operate independently.

### Step 26.1: Locate the configuration file

We can confirm the path to the configuration file by asking PostgreSQL directly through `psql`:

```sql
SHOW config_file;
```

This returns the absolute path, typically similar to:

`/etc/postgresql/18/main/postgresql.conf`

Alternatively, you can execute the following command in the shell to locate the configuration file:

```bash
ls -al /etc/postgresql/*/main/
```

or

```bash
sudo find /etc/postgresql -name postgresql.conf
```

Copy the path to the configuration file for use in the next step.

### Step 26.2: Open the file on the server

From your SSH session (not psql):

```shell
sudo vim /etc/postgresql/18/main/postgresql.conf
```

Substitute the path with the actual path returned in the previous step. It can vary by installation method and operating system.

### Step 26.3: Locate or add the setting

Search for `statement_timeout`. In a default installation it appears commented out, near `idle_in_transaction_session_timeout` in the same section:

`#statement_timeout = 0`

Uncomment and set a value of 1 minute (60,000 milliseconds). A longer value may be appropriate in production, but a short value is deliberately chosen for a learning environment.

`statement_timeout = '60000'`

### Step 26.4: Reload the configuration

```shell
sudo systemctl reload postgresql
```

Or, without shell access:

```sql
SELECT pg_reload_conf();
```

### Step 26.5: Confirm the new value is active

Reconnect, then check:

```sql
SHOW statement_timeout;
```

```sql
SELECT name, setting, source, sourcefile, sourceline
FROM pg_settings
WHERE name = 'statement_timeout';
```

### Step 26.6: Verify with a real, actively-running statement

This test must use an actively executing statement, not an idle transaction, or it will not exercise this setting at all. Reuse `pg_sleep`, now set to a duration longer than the configured timeout.

Since `statement_timeout` was set to 1 minute in the previous step, this statement will be forcibly canceled by the server after 1 minute, before the full sleep completes. The terminal will display:

`ERROR:  canceling statement due to statement timeout`

Unlike `idle_in_transaction_session_timeout`, which terminates the entire session (`FATAL`, connection closed), `statement_timeout` only cancels the offending statement and returns an ERROR; the session and connection remain open, and the user can immediately issue a new query in the same session.

You can confirm this again by running:

```sql
BEGIN;
SELECT pg_sleep(120);
```

Then, after the query is cancelled, run a simple query to confirm the session is still alive:

```sql
ROLLBACK; -- to clean the error from the previous statement
SELECT 1; -- this should succeed, confirming the session is still alive
```

---

## Step 27: Embedding the transaction in a Python backend

Refer to the files in the `0_admin_instructions` folder followed by the [pos_transaction_demo.py](pos_transaction_demo.py) file for a demonstration of how to embed the transaction in a Python backend. The code is annotated with comments to explain most of the steps.

---

## Step 28: Simulate a lost update at `READ COMMITTED` and verify that it is prevented at `REPEATABLE READ`

Create two sessions via `psql`, both using the `siwaka_dishes_app_runtime` account:

```shell
psql -U siwaka_dishes_app_runtime -h localhost -W -d siwaka_dishes
```

The first session will be used to simulate a long-running transaction that reads a value, sleeps for a few seconds, and then writes a new value based on the original read. The second session will read the same value and write a new value before the first session commits.

The PostgreSQL isolation table presented earlier claimed that the `quantityinstock` **read-then-write** pattern used throughout this lab is vulnerable to a lost update at `READ COMMITTED`, and protected at `REPEATABLE READ`. This was verified directly rather than only asserted. To reproduce it, run these two sessions with roughly a one-second gap between starting them:

**Session A**:

```sql
-- Session A
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT quantity_in_stock FROM product WHERE product_code='P007'; \gset  

SELECT pg_sleep(15);

-- Start executing Session B after the previous command
UPDATE product SET quantity_in_stock = :quantity_in_stock - 5 WHERE product_code='P007';
COMMIT;
-- Go back to Session B after the previous command
```

**Session B**, started about one second after Session A:

```sql
-- Session B, started ~1 second after Session A
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT quantity_in_stock FROM product WHERE product_code='P007'; \gset  

-- Go back to Session A after the previous command
UPDATE product SET quantity_in_stock = :quantity_in_stock - 10 WHERE product_code='P007';
COMMIT;
```

Results:

```sql
SELECT quantity_in_stock FROM product WHERE product_code='P007';
```

**Observed result:** Under `READ COMMITTED`, Session A commits normally, and Session B's `UPDATE` also succeeds — but because Session B's `UPDATE` uses the stock value it read a few seconds earlier, its write silently overwrites Session A's committed decrement. The final `quantityinstock` reflects only Session B's `-10`; Session A's `-5` is lost, **with no error and no warning.** 

Re-run the same pair of sessions with `REPEATABLE READ` instead, and the outcome reverses: Session B's `UPDATE` now fails with `ERROR: could not serialize access due to concurrent update`, and its transaction rolls back. PostgreSQL detects that the row changed underneath Session B's snapshot and refuses to let it silently overwrite Session A's change, **forcing the application to retry rather than lose data**.

**Session A**:

```sql
-- Session A
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT quantity_in_stock FROM product WHERE product_code='P007'; \gset  

SELECT pg_sleep(15);

-- Start executing Session B after the previous command
UPDATE product SET quantity_in_stock = :quantity_in_stock - 5 WHERE product_code='P007';
COMMIT;
-- Go back to Session B after the previous command
```

**Session B**, started about one second after Session A:

```sql
-- Session B, started ~1 second after Session A
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT quantity_in_stock FROM product WHERE product_code='P007'; \gset  

-- Go back to Session A after the previous command
UPDATE product SET quantity_in_stock = :quantity_in_stock - 10 WHERE product_code='P007';
COMMIT;
```

Results:

```sql
SELECT quantity_in_stock FROM product WHERE product_code='P007';
```

>**Class Discussion:** Discuss why the isolation level chosen for a transaction is a design decision, not boilerplate (i.e., it is not always best to use the strictest isolation level). Consider the trade-offs between performance and data integrity, and how different isolation levels can affect concurrent transactions in a multi-user environment.

>**Trade-Off:** Stricter isolation levels prevent more anomalies but do so by increasing the rate of blocking, retries, and serialization failures under concurrent load.

---

# Lab Submission Requirements

In addition to the completed practical steps above:

(i) Create a flowchart that can be used to understand the general database transaction.

(ii) Write pseudocode that can be used to understand the general database transaction.

(iii) Suppose the sales department accepts payments in installments but needs a report showing which orders have not been paid in full, and the balance remaining. This is so that it can follow up with clients who still owe the business money. Write an SQL query that can be used to generate this report. *Hint:* Make use of both an INNER JOIN and a LEFT OUTER JOIN in your query.

Submit your answer according to the submission instructions.

---

# References

Elmasri, R., & Navathe, S., B. (2016). Chapter 20: Database Transactions. In *Fundamentals of Database Systems* (7th ed., pp. 745–779). Pearson Education, Inc.

PostgreSQL Global Development Group. *PostgreSQL 18 Documentation*, Chapter 13: Concurrency Control. <https://www.postgresql.org/docs/current/mvcc.html>

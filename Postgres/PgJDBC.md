Ever wondered how pgjdbc communicates with postgres using custom Postgres protocol.
I tried to explain it with a simple query example and wireshark's packet capture.

#### Create a simple table with 2 records in it.

```sql 
gauravsingh=# \d+ foo;
                                           Table "public.foo"
 Column |  Type  | Collation | Nullable | Default | Storage  | Compression | Stats target | Description 
--------+--------+-----------+----------+---------+----------+-------------+--------------+-------------
 bar    | bigint |           |          |         | plain    |             |              | 
 baz    | text   |           |          |         | extended |             |              | 
Access method: heap
Options: autovacuum_enabled=false, toast.autovacuum_enabled=false

gauravsingh=# select * from foo;
 bar |   baz    
-----+----------
   1 | Test
   2 | New Test
(2 rows)
``` 

#### Write a Java class 
Write a Java class to connect to locally running postgres db, this class uses pgjdbc to connect and run a simple select query.
```java
public class ConnectToPostgres {
    public static void main(String[] args) {
        String url = "jdbc:postgresql://localhost:5432/gauravsingh";
        String user = "username";
        String password = "password";
        Connection conn = null;
        Statement stmt = null;
        ResultSet rs = null;
        try {
            conn = DriverManager.getConnection(url, user, password);
            stmt = conn.createStatement();
            String sql = "SELECT bar, baz FROM foo";
            rs = stmt.executeQuery(sql);
            while (rs.next()) {
                int id = rs.getInt("bar");
                String name = rs.getString("baz");
                System.out.println("bar: " + id + ", baz: " + name);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                if (rs != null) rs.close();
                if (stmt != null) stmt.close();
                if (conn != null) conn.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

#### Install Wireshark
Install Wireshark , Select Loopback: lo0 network interface for capturing packets, your postgres instance is running locally on port `5432`, In the capture window use `tcp.port == 5432` filter 
to capture all frame on `5432` port. It should look like below
![wireshark screen](./images/wireshark%20screen.png)   

#### Capture packets
1. Now execute above Java code from class `ConnectToPostgres`
2. Wireshark will capture all the network frames and you can individually inspect them. 
3. Here is one frame which shows result of query `SELECT bar, baz FROM foo`. This is hexadecimal representation. In the next step, I will decipher these.
```
0000   02 00 00 00 45 00 00 b1 00 00 40 00 40 06 00 00   		    ....E.....@.@...
0010   7f 00 00 01 7f 00 00 01 15 38 da d8 df 69 c2 f1   			.........8...i..
0020   47 1b 5a 0d 80 18 18 e6 fe a5 00 00 01 01 08 0a   			G.Z.............
0030   25 4b 86 cb 3d e2 59 98 31 00 00 00 04 32 00 00  			%K..=.Y.1....2..
0040   00 04 54 00 00 00 32 00 02 62 61 72 00 00 00 40   			..T...2..bar...@
0050   06 00 01 00 00 00 14 00 08 ff ff ff ff 00 00 62   				...............b
0060   61 7a 00 00 00 40 06 00 02 00 00 00 19 ff ff ff   			az...@..........
0070   ff ff ff 00 00 44 00 00 00 13 00 02 00 00 00 01   			.....D..........
0080   31 00 00 00 04 54 65 73 74 44 00 00 00 17 00 02   			1....TestD......
0090   00 00 00 01 32 00 00 00 08 4e 65 77 20 54 65 73   			....2....New Tes
00a0   74 43 00 00 00 0d 53 45 4c 45 43 54 20 32 00 5a   			tC....SELECT 2.Z
00b0   00 00 00 05 49                                    						....I
```

Before deciphering this frame, below are the characters from postgres message format involved, you can read [full message format here](https://www.postgresql.org/docs/8.1/protocol-message-formats.html)
| Character     | Meaning |
|----------|-----|
| 1    | Parse Completion  |
| 2      | Bind Completion  |
| T  | Row description  |
| D  | Data row  |
| C  | Command Completion  |
| I  | Idle  | 

##### 1st 4 bytes packet `02 00 00 00`
first 4 bytes `02 00 00 00` are for family of protocol which is `IP(2)`.

![Family](./images/1st%204%20bytes.png)

##### 20 bytes packet `45 00 00 b1 ..... 7f 00 00 01`
This is for IP version 4 details.

![IP Version 4](./images/IP%20Version%204%20packet.png)

##### 32 bytes packet `15 38 da d8 ..... 3d e2 59 98`
This is for TCP details including source port, destination port, sequence number etc.

![tcp src dst](./images/tcp%20src%20dst.png)

##### 5 bytes packet `31 00 00 00 04`
Hex 31, which is 49 which is `1` is ASCII. In postgres protocol message format `1` is indicative of 
`Parse Completion`. `00 00 00 04` which are for length 4.

##### 5 bytes packet `32 00 00 00 04`
`32` which `2` in ASCII which is `Bind Completion`. Subsequent bytes are `00 00 00 04` which are for length 4. 

##### 51 bytes packet `54 00 00 ..... ff ff 00 00`
`54` which is 84 in ASCII which is `T`. `T` is `Row description` as per
postgres message format. Subsequent bytes are `00 00 00 32` which are for length 50. Next `00 02` is for field count 2. Here we receive a description of all the 
columns returned. Next 4 bytes are `62 61 72 00` which are `bar`,next `00 00 40 06` which is postgres table id (OID) 16390. Next `00 01` is for column index, which is 1.
Next `00 00 00 14` is for column type id 20. Next `00 08` is for column length 8. `ff ff ff ff` is -1 which is for type modifier.`00 00` is for format which is Text(0).
`62 61 7a 00` is for `baz`, `00 00 40 06` is for table id (OID) 16390
![row description packet](./images/row%20description.png)

##### 20 bytes packet `44 00 00 ..... 04 54 65 73 74`
This packet is for 1st data row. 

![1st data row](./images/data%20row.png)

##### 24 bytes packet `44 00 00 ..... 20 54 65 73 74`
This packet is for 2nd data row.

![2nd data row](./images/2nd%20data%20row.png)

Above 2 packets (44 bytes) are the hex representation of following.
```sql 
   1 | Test
   2 | New Test
(2 rows)
```

##### 14 bytes packet `43 00 00 ..... 20 32 00`
This is for marking command completion.

![command completion](./images/command%20completion.png)

##### 6 bytes packet `5a 00 ..... 05 49`
This is for indicating state of readiness.

![ready for query](./images/ready%20for%20query.png)



#### PgJDBC Code
Much of pgjdbc code which interprets and translates these messages from postgres are in following class.
[QueryExecutorImpl](https://github.com/pgjdbc/pgjdbc/blob/master/pgjdbc/src/main/java/org/postgresql/core/v3/QueryExecutorImpl.java)







   



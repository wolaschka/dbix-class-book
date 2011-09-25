Chapter 4 - Creating, Reading, Updating and Deleting
====================================================

Chapter summary
---------------

In this chapter we will show how to do basic database operations using your DBIx::Class classes. We are using the MyBlog schema described in [chapter 3]()

Pre-requisites
--------------

We will be giving code examples and comparing them to the SQL statements that they produce, you should have basic SQL knowledge to understand this chapter. The database we are using is provided as an SQL file to import into an [SQLite database](http://search.cpan.org/dist/DBD-SQLite) to get started. You should also have basic knowledge of object-oriented code and Perl classes.

[Download url]() / preparation?

Introduction
------------

The DBIx::Class classes (also called your DBIC schema) contain all the data needed to produce and execute SQL commands on the database. To run commands we just manipulate the objects representing the data.

## Create a Schema object using a database connection

All the database manipulation with DBIx::Class is done via one central Schema object, which maintains the connection to the database via a [storage object](## storage link). To create a schema object, call `connect` on your DBIx::Class::Schema subclass, passing it a [Data Source Name][^dsn].

    my $schema = MyBlog::Schema->connect("dbi:SQLite:myblog.db");
    
Keep the `$schema` object in scope, if it disappears, other DBIx::Class objects you have floating about will stop working. 

To pass a username and password for the database, just add the strings as extra arguments to `connect`, for example when using MySQL:

    my $schema = MyBlog::Schema->connect("dbi:mysql:dbname=myblog", "myuser", "mypassword");

You can also pass various [DBI](http://search.cpan.org/dist/DBI) connection parameters by passing a fourth argument containing a hashref. This is also used by DBIx::Class to set options such as the correct type of quote to use when quoting table names, eg:

    my $schema = MyBlog::Schema->connect("dbi:mysql:dbname=myblog", "myuser", "mypassword", { quote_char => "`'", quote_sep => '.' });

For more detailed information about all the available connection arguments, see the [connect_info documentation](http://search.cpan.org/perldoc?DBIx::Class::Storage::DBI)

## Accessing data, the empty query aka ResultSet

To manipulate any data in your database, you first need to create a **ResultSet** object. A ResultSet is an object representing a potential query, it is used to store the conditions and joins needed to produce the SQL statement.

ResultSets can be fetched using the **Result class** names, for example the users table is in `User.pm`, to fetch its ResultSet, using the `resultset` method:

    my $users_rs = $schema->resultset('User');

Now we can move on to some actual database operations ... 

## Creating users

Now that we have a ResultSet, we can start adding some data. To create one user, we can collect all the relevant data, and initiate and insert the **Row** all at once, by calling the `create` method:

    my $schema = MyBlog::Schema->connect("dbi:mysql:dbname=myblog", "myuser", "mypassword");
    my $users_rs = $schema->resultset('User');
    my $fred = $users_rs->create({
      realname => 'Fred Bloggs',
      username => 'fred',
      password => 'mypass',
      email => 'fred@bloggs.com',
    });
    
`create` is the equivalent of calling the `new_result` method, which returns a **Row** object, and then calling the `insert` method on it, so you can also do this:

    my $schema = MyBlog::Schema->connect("dbi:mysql:dbname=myblog", "myuser", "mypassword");
    my $users_rs = $schema->resultset('User');
    my $fred = $users_rs->new_result();
    $fred->realname('Fred Bloggs');
    $fred->username('fred');
    $fred->password('mypass');
    $fred->email('fred@bloggs.com');
    $fred->insert();

Note how all the columns described in the `User.pm` class using `add_columns` appear on the **Row object** as accessor methods.

To see what's going on, set the shell environment variable [`DBIC_TRACE`](## appendix?) to a true value, and DBIx::Class will display the SQL statement for either of these code samples on STDOUT:

    INSERT INTO users (realname, username, password, email) VALUES (?, ?, ?, ?): 'Fred Bloggs', 'fred', 'mypass', 'fred@bloggs.com'

NB: The `?` symbols are placeholders, the actual values will be quoted according to your database rules, and passed in.

As the `id` column was defined as being `is_auto_increment` we haven't
supplied that value at all, the database will fill it in, and the
`insert` call will fetch the value and store it in our `$fred`
object. It will also do this for other database-supplied fields if
defined as `retrieve_on_insert` in `add_columns`.

### Your turn, create a User and verify with a test

Now that's all hopefully made sense, time for a bit of Test-Driven-Development. 

This is a short Perl test that will check that a user, and only one user, with the `email` of **alice@bloggs.com** exists in the database. You can type it up into a file named **check-alice-exists.t** in t/ directory, or unpack it from the provided tarball.

Note, there are tests for a couple of other things too, happy coding!

    #!/usr/bin/env perl
    use strict;
    use warnings;
    
    use Test::More;
    use_ok('MyBlog::Schema');

    unlink 't/var/myblog.db';
    my $schema = MyBlog::Schema->connect('dbi:SQLite:t/var/myblog.db');
    $schema->deploy();
    ## Your code goes here!
    
    
    ## Tests:   
    my $users_rs = $schema->resultset('User')->search({ email => 'alice@bloggs.com' });
    is($users_rs->count, 1, 'Found exactly one alice user');

    my $alice = $users_rs->next();
    is($alice->id, 1, 'Magically discovered Alice's PK value');
    is($alice->username, 'alice', 'Alice has boring ole username of "alice"');
    is($alice->password, 'aliceandfred', "Guessed Alice's password, woot!');
    like($alice->realname, qr/^Alice/, 'Yup, Alice is named Alice');
    
    done_testing;

Finished? If you get stuck, solutions are included with the downloadable code, and in the Appendix.

## Importing multiple rows at once

Creating users one at a time when they register is all very useful,
but sometimes we want to import a whole bunch of data at once. We can
do this using the `populate` method on **ResultSet**. 

Populate can be called with either an arrayref of hashrefs, one for
each row, using the column names as keys; or an arrayref of arrayrefs,
with the first arrayref containing the column names, and the rest
containing the values in the same order.

Here's an example that will add Fred and Alice at the same time.

    my $schema = MyBlog::Schema->connect("dbi:mysql:dbname=myblog", "myuser", "mypassword");
    my $users_rs = $schema->resultset('User');

    $users_rs->populate([
      [qw/realname username password email/],
      ['Fred Bloggs', 'fred', 'mypass', 'fred@bloggs.com'],
      ['Alice Bloggs, 'alice', 'aliceandfred', 'alice@bloggs.com']
    ]);
    
Populate is most useful in _void context_, that is without requesting
a return value from the call. In this case it will use DBI's
`execute_array` method to insert multiple sets of row data. In list
context `populate` will call `create` repeatedly and return a list of
**Row** objects.

This code will do the same work as the above example, but return
DBIx::Class **Row** objects for later use:

    my $schema = MyBlog::Schema->connect("dbi:mysql:dbname=myblog", "myuser", "mypassword");
    my $users_rs = $schema->resultset('User');

    my @users = $users_rs->populate([
    {
      realname => 'Fred Bloggs',
      username => 'fred',
      password => 'mypass',
      email    => 'fred@bloggs.com',
    },
    {
      realname => 'Alice Bloggs, 
      username => 'alice', 
      password => 'aliceandfred', 
      email    => 'alice@bloggs.com',
    }
    ]);

## Your turn, import some users from a CSV file and verify

The downloadable content for this chapter contains a file named
_multiple-users.csv_ containing several user's data in
comma-separated-values format. To read the lines from the file you can
parse it using a module like
[Text::xSV](https://metacpan.org/module/Text::xSV). The test file can also be found in the Appendix if you don't have the downloadable content.

Data file:

    "realname", "username", "password", "email"
    "Janet Bloggs", "janet", "fredsdaughter", "janet@bloggs.com"
    "Dan Bloggs", "dan", "sillypassword", "dan@bloggs.com"

Add your import code to this Perl test, then run to see how you did:

    #!/usr/bin/env perl
    use strict;
    use warnings;
    
    use Text::xSV;
    
    use Test::More;
    use_ok('MyBlog::Schema');

    unlink 't/var/myblog.db';
    my $schema = MyBlog::Schema->connect('dbi:SQLite:t/var/myblog.db');
    $schema->deploy();
    
    my $csv = Text::xSV->new();
    $csv->load_file('t/data/multiple-users.csv');
    $csv->read_header();
    
    while ($csv->get_row()) {
      my $row = $csv->extract_hash();

      ## Your code goes here!


    }

    ## Tests:
    
    is($schema->resultset('User')->count, 2, 'Two users exist in the database'));
    my $janet = $schema->resultset('User')->find({ username => 'janet' });
    ok($janet, 'Found Janet');
    is($janet->email, 'janet@bloggs.com', 'Janet has the correct email address');
    my $dan = $schema->resultset('User')->find({ username => 'dan' });
    ok($dan, 'Found Dan');
    is($dan->password, 'sillypassword', "Got Dan's password right");

Lookup the solution if you get stuck.

## Finding and changing a User's data later on

We've entered several users into our database, now it would be useful
to be able to find them again, and update their data. If you've been
paying close attention to the tests we've used to check your progress,
you'll notice the `find` ResultSet method.

`find` can be used to find a single database row, using either its primary key or a known unique key

## Create a Post entry for the user

## Update many rows at once

## Deleting a row or rows

## Advanced create/update/delete